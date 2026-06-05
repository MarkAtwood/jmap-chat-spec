# JMAP VTC — Ring Push Guide

For implementers handling ring-call push notifications with `draft-atwood-jmap-vtc-00`.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-vtc-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or deployment
  choice (e.g., "servers SHOULD log push failures"), the keyword reflects implementer
  best practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. Introduction

Ring calls are the most latency-sensitive path in the JMAP VTC specification. When a
caller presses "call", the callee's device MUST ring within roughly two seconds — a
missed-call starts at three. This is a hard constraint, not a soft goal: at five
seconds of delay, the caller has likely already assumed the call failed. The spec
explicitly identifies ring notification as "first-class" and designs the push integration
around this constraint.

Unlike chat message notifications, a ring push must:

- **Wake the device from sleep.** Low-power Doze mode (Android) or app suspension (iOS)
  will silently drop a normal notification. Ring pushes bypass these restrictions using
  platform-specific mechanisms.
- **Present a native incoming-call UI.** Users expect a full-screen call interface, not
  a banner. On iOS this requires CallKit. On Android this requires `ConnectionService` or
  `TelecomManager`. On web it is a partial approximation at best.
- **Carry enough data to render without a network round-trip.** The `VTCCallPush` payload
  is self-contained: it includes `initiatorDisplayName`, `mediaTypes`, `joinUri`, and
  `subject` so the device can display the call screen while the app connects to the JMAP
  server in the background.
- **Self-destruct promptly.** A ring push that arrives after the call has already been
  answered or declined is worse than no notification: it wakes the device unnecessarily
  and confuses the user. The spec's `TTL` guidance (30–60 seconds on Web Push) and
  `VTCCallEndEvent` delivery are the two mechanisms that suppress stale rings.

This guide covers the four delivery paths — APNs VoIP (iOS), FCM high-priority
(Android), Web Push (browser), and WSS (connected clients) — plus the multi-device
forking, timeout, and latency budget topics that cut across all platforms.

### The VTCCallPush payload

Every ring delivery starts with the server constructing a `VTCCallPush` object and
embedding it in the platform envelope. The canonical payload is:

```json
{
  "@type": "VTCCallPush",
  "accountId": "a1b2c3",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "callType": "ring",
  "initiatorId": "user:alice@example.com",
  "initiatorDisplayName": "Alice Chen",
  "subject": null,
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkzq",
  "chatId": null,
  "spaceId": null
}
```

`initiatorDisplayName` is a snapshot taken at push-generation time. It is not an
authoritative identity — a client that wants to verify the caller SHOULD fetch
`VTCCall/get` after the UI is displayed, not before.

`joinUri` is peer-supplied and MUST be treated as untrusted. The app MUST NOT connect
to it without explicit user action (the user tapping "answer"). Auto-joining exposes the
microphone and camera without consent.

Before dispatching any ring push, the server MUST check whether the initiator is blocked
by the target. If so, the push is silently dropped and the initiator is not informed.
Over WebSocket (per the VTC WSS spec), servers MUST also suppress `VTCCallEndEvent`
delivery with `endReason` of `"cancelled"` or `"timeout"` when the initiator is blocked,
to prevent leaking that a blocked contact attempted to call.

---

## 2. APNs VoIP push (iOS)

### Why VoIP push, not standard alert push

Apple provides two push mechanisms relevant to calling:

- **Standard APNs (alert push)**: The system displays a banner or lock-screen
  notification. The app is not launched in the foreground. On a locked device the user
  sees a notification, not a call screen. There is no CallKit integration.
- **PushKit VoIP push**: The system delivers the payload directly to the app's
  `PKPushRegistry` delegate, launches the app in the background if it is not running,
  and gives it CPU time to register the call with CallKit. The result is the native
  incoming-call screen: full-screen UI, "answer" / "decline" buttons, ringtone.

The spec is explicit: servers MUST use `apns-push-type: voip` for ring notifications on
iOS. Standard alert push is not an acceptable substitute for ring calls.

### The 2-second CallKit rule

Apple imposes a strict requirement: after receiving a VoIP push, the app MUST call
`CXProvider.reportNewIncomingCall(with:update:completion:)` within approximately two
seconds, or Apple will penalize the app — first by throttling VoIP push delivery, then
by revoking VoIP push entitlement entirely for repeated violations. The app cannot
defer this: it must synchronously report the incoming call before performing any
asynchronous network operation.

Implementation consequence: the `VTCCallPush` payload MUST contain enough data to
call `reportNewIncomingCall` without a network round-trip. The
`CXCallUpdate` object requires at minimum a caller handle and display name; the payload
provides both via `initiatorId` and `initiatorDisplayName`. The app calls
`VTCCall/get` to refresh state after reporting to CallKit, not before.

### APNs request format

Headers:

```
apns-push-type: voip
apns-topic: com.example.app.voip     (bundle ID with .voip suffix)
apns-expiration: <unix timestamp 30-60 seconds from now>
apns-priority: 10                    (immediate delivery; required for voip)
```

The `apns-topic` for VoIP push MUST use the `.voip` suffix. Using the bare bundle ID
sends a standard alert push instead, which will not invoke PushKit.

Body (the outer APNs payload; `VTCCallPush` is embedded in `aps` or a custom key):

```json
{
  "aps": {},
  "jmap-vtc-push": {
    "@type": "VTCCallPush",
    "accountId": "a1b2c3",
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
    "callType": "ring",
    "initiatorId": "user:alice@example.com",
    "initiatorDisplayName": "Alice Chen",
    "subject": null,
    "mediaTypes": ["audio", "video"],
    "joinUri": "https://meet.example.com/room/xkzq",
    "chatId": null,
    "spaceId": null
  }
}
```

The `aps` dictionary is left empty because CallKit — not the system notification
UI — handles the presentation. Including `alert` in `aps` for a VoIP push has no
effect.

### Authentication: token vs. certificate

APNs supports two authentication methods:

- **Token-based (recommended)**: A JWT signed with a private `.p8` key, presented in
  the `authorization` header. The JWT is valid for one hour; the server must rotate it.
  One key can serve multiple apps. This is the preferred method for new deployments.
- **Certificate-based**: A TLS client certificate specific to each app. Valid for one
  year. More operationally heavy; Apple is deprecating it for some use cases.

For VoIP push, the certificate (or token key) must be the VoIP-specific credential, not
the standard push credential. The two are separate entitlements.

### Background launch

When the app is not running and a VoIP push arrives, iOS launches the app in the
background. The app has roughly 30 seconds of background execution time. During this
window it should:

1. Report the incoming call to CallKit immediately (`reportNewIncomingCall`).
2. Authenticate to the JMAP server (token should be cached; do not prompt for
   credentials here).
3. Call `VTCCall/get` to refresh call state.
4. When the user answers, call `VTCParticipant/set` to set `joinedAt` and connect to
   `joinUri`.

If the app is killed by the user (swiped away from the app switcher), iOS will not
deliver VoIP pushes to it on some configurations. Apps SHOULD inform users not to force-
quit the app if they want to receive calls, or implement a server-side fallback (e.g.,
a standard alert push that prompts the user to open the app).

### Handling VTCCallEndEvent on iOS

When the call is answered on another device, declined, or times out, the server delivers
a `VTCCallEndEvent` with the appropriate `endReason`. If the app has a WebSocket
connection active, it receives this as an ephemeral event and MUST call
`CXProvider.reportCall(with:endedAt:reason:)` to dismiss the CallKit UI. If the app was
backgrounded by the VoIP push and has no WebSocket, it should poll `VTCCall/get` after a
short delay to detect the ended state.

---

## 3. FCM high-priority (Android)

### Data messages vs. notification messages

FCM offers two message types:

- **Notification messages**: FCM constructs and displays a system notification. The app
  is not launched. This is appropriate for chat message alerts but not for calls.
- **Data messages**: The payload is delivered directly to the app's
  `FirebaseMessagingService.onMessageReceived()`. The app handles display entirely. This
  is the correct type for incoming calls.

JMAP VTC ring pushes MUST be sent as FCM data messages, not notification messages. A
notification message cannot trigger `ConnectionService`; a data message can.

### priority: high and Doze mode

Android's Doze mode suspends app execution and network access when the device is idle.
Normal-priority FCM messages are held until the device leaves Doze. Setting
`"priority": "high"` (REST v1 API: `message.android.priority: "HIGH"`) causes FCM to
deliver the message immediately by temporarily waking the device. This is the Android
equivalent of APNs VoIP push.

Without `priority: high`, a ring notification may be delayed by minutes while the device
is in Doze — a missed call.

FCM REST v1 payload:

```json
{
  "message": {
    "token": "<device-fcm-token>",
    "android": {
      "priority": "HIGH",
      "ttl": "45s"
    },
    "data": {
      "jmap-vtc-push": "{\"@type\":\"VTCCallPush\",\"accountId\":\"a1b2c3\",\"callId\":\"01J4XKZQN4MWVT8PPBEHTJ3AB\",\"callType\":\"ring\",\"initiatorId\":\"user:alice@example.com\",\"initiatorDisplayName\":\"Alice Chen\",\"subject\":null,\"mediaTypes\":[\"audio\",\"video\"],\"joinUri\":\"https://meet.example.com/room/xkzq\",\"chatId\":null,\"spaceId\":null}"
    }
  }
}
```

FCM data values are strings; the `VTCCallPush` JSON object MUST be JSON-serialized as a
string in the `data` map. The receiving app deserializes it in `onMessageReceived`.

The `ttl` (time-to-live) for ring pushes SHOULD be 30–60 seconds. Beyond that window
the call has almost certainly ended; delivering a stale ring push is harmful.

### ConnectionService and TelecomManager

Android's `ConnectionService` API (and the higher-level `TelecomManager`) provides the
same native incoming-call UI as iOS CallKit: full-screen call screen, ringtone, hardware
button integration, and do-not-disturb bypass.

In `onMessageReceived`, the app should:

1. Deserialize the `VTCCallPush` payload from `data["jmap-vtc-push"]`.
2. Call `TelecomManager.addNewIncomingCall(phoneAccountHandle, extras)` to trigger the
   native call screen. The `extras` bundle should include the caller display name and
   a reference to the `callId` so the `Connection` subclass can answer and hang up
   correctly.
3. The `Connection.onAnswer()` callback fires when the user taps "answer". At that
   point, call `VTCParticipant/set` to set `joinedAt` and connect to `joinUri`.
4. The `Connection.onDisconnect()` callback fires when the user declines or the call
   ends. Send `VTCParticipant/set` with `callResponse: "declined"` if in state
   `"ringing"`.

`TelecomManager.addNewIncomingCall` requires the `MANAGE_OWN_CALLS` permission.
Apps targeting Android 9 (API 28) and later MUST declare and request this permission.

### Android 12+ foreground service restrictions

Android 12 (API 31) introduced restrictions on starting foreground services from the
background. A data message handler running in the background can no longer start a
foreground service arbitrarily. The exception for incoming calls: `ConnectionService` and
`TelecomManager.addNewIncomingCall` are explicitly exempt from this restriction and
remain callable from the FCM `onMessageReceived` handler.

Implementations that attempt to start a foreground service to handle the call outside of
`ConnectionService` will encounter `ForegroundServiceStartNotAllowedException` on
Android 12+ unless an appropriate foreground service type (`phoneCall`,
`mediaPlayback`, or `camera`) is declared in the manifest and the launch context is
permitted. Using `ConnectionService` avoids this complication entirely.

### Handling VTCCallEndEvent on Android

When the server dispatches a `VTCCallEndEvent`, the app (if it has a WebSocket
connection) receives it as an ephemeral event and MUST call `Connection.setDisconnected`
on the active `Connection` object to dismiss the call UI. If the app has no active
WebSocket, it should monitor `VTCCall/get` after the FCM-driven `onMessageReceived`
returns, using a short polling interval during the ring window.

---

## 4. Web Push

### Overview

RFC 8030 Web Push delivers push messages to a browser's push service (e.g., Google's
FCM-backed service for Chrome, Mozilla's autopush for Firefox). The browser wakes the
registered service worker, which receives the push event and can display a notification
or take other action.

Web Push cannot present a native incoming-call screen. This is a fundamental platform
limitation: browsers do not expose `ConnectionService` or CallKit equivalents to web
applications. The web platform's best approximation is:

- A permission-granted system notification with "Answer" and "Decline" action buttons.
- When the user clicks "Answer", the service worker opens (or focuses) the app tab and
  passes the `VTCCallPush` payload for the tab to connect to `joinUri`.

This is a meaningful degradation compared to native. Users running JMAP VTC clients in
a browser SHOULD be informed that the native app provides a better incoming-call
experience.

### VAPID authentication

Web Push requires the server to authenticate push requests using VAPID (RFC 8292).
VAPID uses an EC key pair (P-256); the public key is shared with the push subscription
endpoint at subscription time. The server signs a JWT with the private key and includes
it in the `Authorization: vapid` header on every push request.

Key rotation: VAPID keys are long-lived (months to years). Rotating them requires
re-registering all push subscriptions. Plan key rotation carefully and keep the old key
valid until all active subscriptions have migrated.

### Web Push request

```
POST https://push.example.com/push/<subscription-id>
Content-Type: application/octet-stream
Content-Encoding: aes128gcm
Authorization: vapid t=<jwt>,k=<public-key-base64url>
TTL: 45
Urgency: high
```

The body is the encrypted `VTCCallPush` payload (RFC 8291, encrypted with the
subscription's public key). The `Urgency: high` header tells the push service to deliver
immediately rather than batching with lower-urgency messages. For ring calls, `high`
urgency MUST be used.

Decrypted payload (what the service worker receives):

```json
{
  "@type": "VTCCallPush",
  "accountId": "a1b2c3",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "callType": "ring",
  "initiatorId": "user:alice@example.com",
  "initiatorDisplayName": "Alice Chen",
  "subject": null,
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkzq",
  "chatId": null,
  "spaceId": null
}
```

### Service worker handling

```javascript
self.addEventListener('push', event => {
  const payload = event.data.json();
  if (payload['@type'] !== 'VTCCallPush') return;

  const options = {
    body: `Incoming call from ${payload.initiatorDisplayName}`,
    requireInteraction: true,          // keep notification visible until user acts
    tag: `vtc-ring-${payload.callId}`, // deduplicate if multiple pushes arrive
    data: payload,
    actions: [
      { action: 'answer', title: 'Answer' },
      { action: 'decline', title: 'Decline' }
    ]
  };

  event.waitUntil(
    self.registration.showNotification('Incoming call', options)
  );
});

self.addEventListener('notificationclick', event => {
  const payload = event.notification.data;
  event.notification.close();

  if (event.action === 'decline') {
    // POST VTCParticipant/set with callResponse: "declined"
    event.waitUntil(declineCall(payload));
    return;
  }

  // 'answer' or click on notification body: open/focus app tab
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(windowClients => {
      for (const client of windowClients) {
        if (client.url.includes(self.location.origin) && 'focus' in client) {
          client.postMessage({ type: 'vtc-incoming', payload });
          return client.focus();
        }
      }
      return clients.openWindow(`/?vtc-call=${payload.callId}`);
    })
  );
});
```

Key points:

- `requireInteraction: true` prevents the notification from auto-dismissing after a few
  seconds, which is critical for call notifications.
- `tag` deduplicates: if the server retries the push or both push and WSS arrive, the
  second notification replaces the first rather than stacking.
- The "Answer" action opens or focuses the app window and passes the payload via
  `postMessage`. The app tab connects to `joinUri` after the user confirms. The service
  worker MUST NOT connect to `joinUri` directly.

### Notification permission UX

Web Push requires explicit notification permission (`Notification.requestPermission()`).
Browsers suppress the permission prompt if it is requested too early (before any user
gesture) and may permanently block the origin. Best practice:

- Request notification permission only after the user has signed in and explicitly opted
  in to call notifications (e.g., a settings toggle labeled "Incoming call
  notifications").
- Explain why the permission is needed before requesting it.
- If permission is denied, surface a persistent in-app prompt explaining that the user
  will not receive ring notifications and how to re-enable.

### Trade-offs vs. native

| Capability | iOS (PushKit + CallKit) | Android (FCM + ConnectionService) | Web Push |
|---|---|---|---|
| Wakes device | Yes | Yes | Partial (tab must be open or browser running) |
| Native call UI | Yes | Yes | No (notification only) |
| Answer without opening app | Yes | Yes | No (must focus app tab) |
| Do-not-disturb bypass | Yes (CallKit) | Yes (ConnectionService) | No |
| TTL enforcement | Server-set `apns-expiration` | Server-set `ttl` | Server-set `TTL` header |

Web Push is suitable for desktop browser sessions where the user is actively working.
It is not a substitute for native push on mobile devices.

---

## 5. Multi-device forking and races

### Simultaneous delivery

A user typically has a phone, a tablet, and a desktop browser. All registered devices
for that user receive the ring push simultaneously. The server delivers a `VTCCallPush`
to every push endpoint and a `VTCRingEvent` to every active WebSocket connection. All
devices ring at once.

This is the correct behavior. Picking which device to ring requires the server to know
the user's current device — information it does not reliably have. Forking to all devices
ensures the call is received regardless of which device is in the user's hand.

### First device wins

The state machine resolves the race: the first `VTCParticipant/set` update that sets
`joinedAt` on the target's VTCParticipant record wins. The server accepts it, transitions
the call to `"active"`, and then:

1. Dispatches `VTCCallEndEvent` with `endReason: "answered_elsewhere"` to all other
   push endpoints and WebSocket connections of the same user.
2. Dispatches `VTCCallEndEvent` with `endReason: "answered"` to all other target
   participants in a multi-party ring.

### Client behavior on "answered_elsewhere"

Each device receiving `VTCCallEndEvent` with `endReason: "answered_elsewhere"` MUST
stop ringing immediately:

- **iOS**: Call `CXProvider.reportCall(with:endedAt:reason:)` with
  `CXCallEndedReason.answeredElsewhere`.
- **Android**: Call `Connection.setDisconnected` with a `DisconnectCause` of
  `ANSWERED_ELSEWHERE`.
- **Web**: Close the ring notification (`Notification.close()` or close via
  `tag`-deduplication by showing a replacement with `requireInteraction: false`).

Clients that miss the `VTCCallEndEvent` (e.g., a backgrounded iOS app with no WebSocket)
SHOULD poll `VTCCall/get` at a short interval (every 2–3 seconds) during the ring window
to detect state transitions. The maximum polling window is bounded by the ring timeout
(Section 6).

### Simultaneous answer race on the same device

In the unusual case where two `VTCParticipant/set` updates for the same participant
arrive concurrently (e.g., two rapid taps), the server processes them in order. The
second update arrives on a call already in state `"active"` with the participant already
joined; the server SHOULD return a no-op success rather than an error, since the
desired final state is already achieved.

### Device preference

The spec does not define a device priority mechanism — forking is uniform. Deployments
that want priority routing (e.g., "ring mobile first, then desktop after 5 seconds")
can implement this server-side by delaying push delivery to lower-priority endpoints.
This is a deployment policy, not a protocol behavior. If implemented, the delay MUST be
short (at most a few seconds) to remain within the overall latency budget.

---

## 6. Timeout and missed calls

### Ring timeout

The spec states that ring timeout is deployment-defined and recommends 30 seconds. A
range of 30–60 seconds is appropriate for most deployments:

- 30 seconds is the minimum before a caller loses patience.
- 60 seconds matches traditional cellular ring timeout behavior and is the APNs
  VoIP push `apns-expiration` ceiling recommended in this guide.
- Beyond 60 seconds, the APNs VoIP push may have already expired and the iOS call
  screen dismissed by the OS.

The timeout timer starts when the server transitions the call to `"ringing"` (i.e.,
after ring notifications have been dispatched). It is reset only by a participant
answering or declining. It is not reset by device-level acknowledgments.

### Timeout behavior

When the timeout fires with no answer:

1. The server transitions the VTCCall to `"ended"` with `endReason: "missed"`.
2. The server dispatches `VTCCallEndEvent` with `endReason: "timeout"` to all ringing
   devices and WebSocket connections.
3. Devices receiving the event dismiss the call UI via the platform-specific
   mechanism described in Sections 2–4.

On iOS, if the app has no active WebSocket (e.g., it was launched by the VoIP push and
has not yet established a connection), the timeout `VTCCallEndEvent` may not arrive in
time. The app SHOULD implement a local failsafe: if `VTCCall/get` returns `state:
"ended"` during the polling loop, dismiss the call UI and do not proceed to `joinUri`.

### Missed call notifications

After a ring times out (or all targets decline), the server SHOULD deliver a missed-call
notification. This is a normal-urgency notification, not a VoIP push:

- **iOS**: Standard APNs alert push with `apns-push-type: alert`.
- **Android**: Normal-priority FCM notification message.
- **Web**: Web Push with `Urgency: normal`.

The missed-call notification payload is not a `VTCCallPush`; it is a regular JMAP
`StateChange` or a deployment-defined notification indicating that the VTCCall's
`endReason` is `"missed"`. The client renders it as "Missed call from Alice Chen".

Missed-call notifications can be delivered with a longer TTL (several hours) since they
are informational, not time-critical.

### Voicemail

Voicemail is out of scope for `draft-atwood-jmap-vtc-00`. Deployments that support
voicemail can model it as a gateway-specific signal flow: when the ring times out, a
PSTN or SIP gateway records a message, and the server delivers a `VTCCallEndEvent`
followed by a deployment-defined notification linking to the voicemail recording. The
voicemail recording itself would be served as a separate media resource, not as a
VTCRecording object (which is for in-call recordings, not voicemail).

### Decline handling

When all ring targets decline before the timeout:

1. The server transitions the call to `"ended"` with `endReason: "declined"`.
2. `VTCCallEndEvent` with `endReason: "declined"` is dispatched to remaining ringing
   devices of the declining user.

A single decline (in a multi-target ring) ends ringing on all devices of that specific
user. The other target participants continue ringing until they answer, decline, or the
timeout fires.

### Retry behavior

The spec does not define call retry behavior. If the caller wants to retry after a
missed call or declined call, they create a new VTCCall (a new `VTCCall/set create`),
which generates new push notifications under the per-caller ring rate limits.

The per-caller rate limit (the spec recommends no more than 3 ring calls to the same
target within 60 seconds) prevents retry from becoming harassment. Servers MUST enforce
this limit. When a caller exceeds the limit, the server SHOULD reject the `VTCCall/set
create` with `tooManyRequests` rather than silently accepting it and dropping the
notification.

---

## 7. Latency budget breakdown

### The 2-second target

The spec states that ring notification is designed to deliver within two seconds of the
caller pressing "call". This is the end-to-end budget: from the server receiving the
`VTCCall/set create` to the callee's device ringing audibly.

The following breakdown is based on production calling system benchmarks (FaceTime,
WhatsApp, Signal) and reflects realistic expectations. Actual numbers vary by network
conditions, device state, and push service load.

| Stage | Typical | Worst case | Notes |
|---|---|---|---|
| Server creates VTCCall, dispatches push | 20–80 ms | 200 ms | Database write + push service HTTP request |
| Push service queues and delivers to device | 50–300 ms | 800 ms | APNs/FCM delivery; varies by load |
| Device wakes from sleep | 50–200 ms | 500 ms | Doze exit on Android; iOS resume latency |
| App receives push, reports to CallKit/ConnectionService | 50–200 ms | 500 ms | App execution startup if not running |
| OS presents call screen, ringtone starts | 10–50 ms | 100 ms | OS-controlled; typically fast |
| **Total** | **180–830 ms** | **2100 ms** | |

The 2-second budget is achievable under normal conditions. It fails under:

- FCM delivery delays (GCM/FCM is occasionally slow during high-load periods; median
  is fast but the 99th percentile is not).
- A cold-start Android app that performs unnecessary initialization before calling
  `TelecomManager.addNewIncomingCall`.
- Network conditions on the callee's device (cellular handoff, poor signal).

### Where time is most often lost

**Server-side (controllable):** The push dispatch should be the first thing the server
does after creating the VTCCall. Do not wait for the JMAP response to be serialized and
sent to the caller before dispatching push. These can happen in parallel.

**APNs/FCM delivery (partially controllable):** Use a connection pool to the push
service rather than establishing a new TLS connection per push. APNs HTTP/2 supports
multiplexing; maintain a persistent connection and send pushes without waiting for
acknowledgment of prior pushes.

**App startup (controllable):** On iOS, the VoIP push delegate callback
(`PKPushRegistry.pushRegistry(_:didReceiveIncomingPushWith:for:)`) MUST be synchronous
with respect to calling `reportNewIncomingCall`. Do not `async/await` before that call.
On Android, `onMessageReceived` runs on a background thread; call
`TelecomManager.addNewIncomingCall` immediately without posting to a Handler.

**Measurement:** Instrument with timestamps at each stage:

- `T0`: `VTCCall/set create` received by server.
- `T1`: Push request sent to APNs/FCM (server log).
- `T2`: Push received by device (client log in PushKit delegate or
  `onMessageReceived`).
- `T3`: `reportNewIncomingCall` / `addNewIncomingCall` called (client log).
- `T4`: Ringtone plays (OS callback confirming call screen presented).

Report `T2 - T1` as push delivery latency and `T4 - T0` as end-to-end ring latency.
Alert when p95 of `T4 - T0` exceeds 2 seconds.

### WSS path (connected clients)

For clients with an active WebSocket connection, the server delivers a `VTCRingEvent`
in addition to the push. The WSS path is typically faster than push (sub-100 ms for a
connected client on a good network) because it skips the push service entirely. Clients
that receive a `VTCRingEvent` before the push arrives SHOULD begin presenting the
incoming-call UI immediately and suppress the duplicate push-triggered UI when (if) the
push arrives later.

On iOS, a client with an active WebSocket (i.e., the app is in the foreground) receives
the `VTCRingEvent` over WSS and can present the in-app calling UI without going through
CallKit. However, CallKit SHOULD still be notified so that the call appears in the
system's recent calls list, integrates with Siri, and handles hardware button events
correctly.

The `VTCRingEvent` carries `callId`, `initiatorId`, `mediaTypes`, and `joinUri` — a
smaller field set than `VTCCallPush`, which additionally includes `accountId`,
`callType`, `initiatorDisplayName`, `subject`, `chatId`, and `spaceId`. The client has
all the information needed to render the call screen without a `VTCCall/get` round-trip.

### TTL and stale-push mitigation

The server sets a short TTL on ring pushes (30–60 seconds, matching the ring timeout).
If the device is offline for longer than the TTL, the push service discards the
notification and the device never rings — which is the correct behavior, since the
call has already ended by the time it would be delivered.

The `VTCCallEndEvent` provides the belt-and-suspenders: even if a push arrives after the
call ends (e.g., the push service was slow), the `VTCCallEndEvent` that arrived earlier
over WSS has already dismissed the call UI. The two mechanisms together ensure that
devices do not ring for calls that are already resolved.
