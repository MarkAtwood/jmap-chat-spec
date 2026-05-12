# JMAP Push — Platform Delivery Guide

For server implementers. Covers authentication, request format, urgency mapping, and error
handling for delivering JMAP push payloads to each supported platform. Applies to any JMAP
extension that uses the RFC 8620 `PushSubscription` mechanism. For payload construction
specific to a particular extension, see the supplement for that extension (e.g.,
`jmap-chat-push-platform-guide.md` for `draft-atwood-jmap-chat-push-00`).

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The relevant drafts (`draft-atwood-jmap-chat-*.md`, RFC 8620, RFC 8030, RFC 8291) are
  the normative source of truth. Where this guide describes a spec requirement using a
  keyword, the keyword reflects the spec's normativity; if guide and spec disagree, the
  spec wins.
- Where this guide uses a keyword for an operational practice, platform-handling
  convention, or deployment choice (e.g., "servers SHOULD cache OAuth tokens"), the
  keyword reflects implementer best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## Platform overview

| Platform | Registration | Auth | Body | Size limit | E2E encrypted |
|---|---|---|---|---|---|
| Web Push | URL (native) | VAPID JWT | Encrypted binary | 3993 B plaintext | Yes |
| WNS | URL (native) | OAuth 2.0 | Raw JSON | 5000 B | No |
| FCM | Out-of-band token | OAuth 2.0 | JSON | 4096 B | No |
| ADM | Out-of-band token | OAuth 2.0 | JSON | 6000 B | No |
| HPK | Out-of-band token | OAuth 2.0 | JSON | 4096 B | No |
| APNs | Out-of-band token | JWT (ES256) | JSON | 4096 B | No |
| MiPush | Out-of-band token | Static key | Form-encoded | 4096 B | No |

**URL-based platforms** (Web Push, WNS) fit `PushSubscription.url` natively — the client
provides the URL and the server POSTs to it directly. No out-of-band registration is
needed.

**Token-based platforms** (all others) require an extra step: the client receives an
opaque token from the platform SDK and registers it with the JMAP server via a
server-specific API outside JMAP. The server maintains a mapping from JMAP subscription
to platform token and routes delivery internally.

Only Web Push (with RFC 8291 encryption) is end-to-end encrypted — only the client device
can decrypt the payload. All other platforms relay traffic through vendor infrastructure
that can read the payload in transit.

---

## Common patterns

### Always use data-only messages

Every platform supports two message types: **display messages**, where the OS or vendor
infrastructure constructs and shows a notification, and **data messages**, where the
payload is delivered to the app silently. Servers MUST use data messages, not display
messages. With a display message, the OS generates a notification before the app runs —
the implementation cannot apply JMAP-specific logic (mention highlighting, E2E
decryption, deduplication) to a notification that has already been shown.

| Platform | How to send data-only |
|---|---|
| FCM | Omit `"notification"` key; use `"data"` only |
| ADM | `"data"` map only; no notification key |
| HPK | Omit `"notification"` key; use `"data"` only |
| APNs | `apns-push-type: background` + `"content-available": 1`, no `"alert"` key — or alert push with a Notification Service Extension to rewrite before display |
| MiPush | `pass_through=1` |
| WNS | `X-WNS-Type: wns/raw` |

### Credential errors are not subscription errors

On OAuth token refresh failure or APNs JWT signing failure, servers SHOULD retry with
exponential backoff. Servers MUST NOT destroy the `PushSubscription` in this case — a
credential problem means the server cannot reach the platform right now; it says
nothing about whether the device registration is still valid. Servers MUST only destroy
the subscription when the platform explicitly tells the server the token or endpoint is
gone (e.g. FCM `UNREGISTERED`, APNs `Unregistered`, HTTP `404` or `410`).

---

## Google FCM (Firebase Cloud Messaging)

Delivers to Android globally and to iOS via APNs proxy. The standard push path for
Android devices with Google Play Services.

**Endpoint:** `https://fcm.googleapis.com/v1/projects/{project_id}/messages:send`

### Auth

FCM v1 uses OAuth 2.0 with a service account JSON key file:

```
POST https://oauth2.googleapis.com/token
grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&assertion={signed_jwt}
```

Scope: `https://www.googleapis.com/auth/firebase.messaging`. Tokens are valid for one
hour; servers SHOULD cache and refresh proactively.

### Request

```http
POST https://fcm.googleapis.com/v1/projects/{project_id}/messages:send
Authorization: Bearer {oauth_token}
Content-Type: application/json

{
  "message": {
    "token": "{device_token}",
    "android": {
      "priority": "{HIGH|NORMAL}"
    },
    "data": {
      "jmap-push": "{payload JSON string}"
    }
  }
}
```

`data` values MUST be strings; servers embed the payload JSON as an escaped string
under an application-defined key (`"jmap-push"` is a reasonable choice). The
`"notification"` key MUST be omitted (see "Always use data-only messages").

**iOS via FCM:** servers MUST replace the `"android"` block with an `"apns"` block. FCM
can only deliver to iOS if an APNs auth key (`.p8`) or certificate is configured in the
Firebase project — without it, FCM accepts the request but silently drops delivery.

### Urgency

| JMAP urgency | `android.priority` |
|---|---|
| `"high"` | `"HIGH"` |
| `"normal"` | `"NORMAL"` |
| `"low"` | `"NORMAL"` |
| `"very-low"` | `"NORMAL"` |

`"HIGH"` wakes devices in Doze mode. `"NORMAL"` is opportunistic. FCM has no lower
tier; servers MUST map `"low"` and `"very-low"` to `"NORMAL"`.

### Error responses

FCM returns `200 OK` even when delivery fails — servers MUST always check the response
body:

- `error.status: "UNREGISTERED"` or `"SENDER_ID_MISMATCH"` — servers MUST destroy the
  `PushSubscription`.
- `error.status: "QUOTA_EXCEEDED"` — servers SHOULD back off and retry.
- `error.status: "INVALID_ARGUMENT"` — payload problem; servers SHOULD log and fix, and
  MUST NOT retry.

---

## Amazon ADM (Amazon Device Messaging)

Delivers to Amazon Fire tablets and Fire TV. Fire OS is Android-derived; many apps
support both FCM and ADM.

**Endpoint:** `https://api.amazon.com/messaging/registrations/{registration_id}/messages`

### Auth

ADM uses OAuth 2.0 client credentials. Obtain client ID and secret from the Amazon
Developer Portal:

```
POST https://api.amazon.com/auth/O2/token
grant_type=client_credentials&scope=messaging:push&client_id={id}&client_secret={secret}
```

Tokens are valid for one hour; servers SHOULD cache and refresh proactively.

### Request

```http
POST https://api.amazon.com/messaging/registrations/{registration_id}/messages
Authorization: Bearer {oauth_token}
Content-Type: application/json
X-Amzn-Type-Version: com.amazon.device.messaging.ADMMessage@1.0
Accept: application/json
X-Amzn-Accept-Type: com.amazon.device.messaging.ADMSendResult@1.0

{
  "data": {
    "jmap-push": "{payload JSON string}"
  },
  "priority": "{high|normal}",
  "expiresAfter": {seconds}
}
```

`data` values MUST be strings. Servers SHOULD set `expiresAfter` to reflect how long
the notification is still useful — shorter values for higher urgency.

### Urgency

| JMAP urgency | `priority` |
|---|---|
| `"high"` | `"high"` |
| `"normal"` | `"normal"` |
| `"low"` | `"normal"` |
| `"very-low"` | `"normal"` |

`"high"` wakes devices immediately; `"normal"` is opportunistic. No lower tier exists.

### Error responses

ADM returns a JSON body with a `reason` field. Servers MUST parse it — HTTP status
alone is not sufficient:

- `InvalidRegistrationId` / `Unregistered` — servers MUST destroy the
  `PushSubscription`.
- `MaxRateExceeded` — servers SHOULD back off and retry.
- `MessageTooLarge` — servers SHOULD reduce payload and retry.

---

## Huawei Push Kit (HPK)

Delivers to HarmonyOS and EMUI devices. Structurally similar to FCM v1; the main
difference is that `"data"` is a single string rather than a map.

**Endpoint:** `https://push-api.cloud.huawei.com/v1/{app_id}/messages:send`

### Auth

HPK uses OAuth 2.0 client credentials. Obtain App ID and App Secret from Huawei
AppGallery Connect:

```
POST https://oauth-login.cloud.huawei.com/oauth2/v3/token
grant_type=client_credentials&client_id={app_id}&client_secret={app_secret}
```

Tokens are valid for one hour; servers SHOULD cache and refresh proactively.

### Request

```http
POST https://push-api.cloud.huawei.com/v1/{app_id}/messages:send
Authorization: Bearer {oauth_token}
Content-Type: application/json

{
  "message": {
    "token": ["{push_token}"],
    "android": {
      "urgency": "{HIGH|NORMAL}"
    },
    "data": "{payload JSON string}"
  }
}
```

`"data"` is a single string (not a map); servers MUST serialize the entire payload JSON
into it. The `"notification"` key MUST be omitted.

### Urgency

| JMAP urgency | `android.urgency` |
|---|---|
| `"high"` | `"HIGH"` |
| `"normal"` | `"NORMAL"` |
| `"low"` | `"NORMAL"` |
| `"very-low"` | `"NORMAL"` |

`"HIGH"` wakes devices immediately; `"NORMAL"` is opportunistic. No lower tier exists.
Servers MUST map `"low"` and `"very-low"` to `"NORMAL"`.

### Error responses

HPK returns a JSON body with a `code` field on every response regardless of HTTP
status. Success is `code: 80000000`. Servers MUST parse the `code` field to classify
errors — HTTP status alone is not sufficient.

---

## Xiaomi Push (MiPush)

Delivers to Xiaomi (MIUI/HyperOS) devices in mainland China, where Google Play Services
is unavailable. Key differences from FCM/HPK: authentication uses a static secret key
rather than OAuth, and the request body is form-encoded rather than JSON.

**Endpoint:** `https://api.xmpush.xiaomi.com/v3/message/regid`

### Auth

MiPush uses a static App Secret from the Xiaomi Developer Console. Servers MUST include
it on every request — no token exchange, no expiry:

```
Authorization: key={app_secret}
```

### Request

```http
POST https://api.xmpush.xiaomi.com/v3/message/regid
Authorization: key={app_secret}
Content-Type: application/x-www-form-urlencoded

registration_id={regid}&restricted_package_name={package}&payload={url_encoded_json}&pass_through=1&time_to_live={ttl_ms}
```

`payload` is a string; servers MUST URL-encode the JSON payload and place it there.
`pass_through=1` routes to the app's broadcast receiver silently. `time_to_live` is in
milliseconds.

### Urgency

MiPush pass-through delivery priority is governed by notification channel importance
levels configured in the app at channel registration time, not by a per-message field.
Implementers SHOULD check current Xiaomi developer documentation for per-message
priority fields that may be available in newer API versions.

### Error responses

MiPush returns a JSON body. `result: "ok"` on success; on failure, the `code` field
identifies the error. Implementers SHOULD consult current Xiaomi developer
documentation for the full code reference. Invalid or unregistered regIds indicate the
subscription MUST be destroyed; rate-limit errors call for backoff and retry.

---

## Other Chinese OEM push systems

OPPO (ColorOS Push) and Vivo ship their own push SDKs on mainland China devices without
Google Play Services. Both use the same structural pattern as MiPush: token-based
out-of-band registration, a vendor endpoint, and a pass-through (data-only) mode.
Authentication and request format differ by vendor; consult the respective developer
portals for specifics.

When targeting mainland China, implementations SHOULD support MiPush and HPK at
minimum; OPPO and Vivo MAY be added depending on the target market.

---

## Apple APNs (Apple Push Notification Service)

Delivers to iOS, macOS, and watchOS devices. **APNs requires HTTP/2.**

**Production:** `https://api.push.apple.com/3/device/{device_token}`
**Sandbox:**    `https://api.sandbox.push.apple.com/3/device/{device_token}`

### Auth

**Token-based (recommended):** Obtain a signing key (`.p8`) from Apple Developer and
sign a JWT for each request:

- Header: `alg: ES256`, `kid: {key_id}`
- Payload: `iss: {10-char team ID}`, `iat: {unix time}`

```
Authorization: bearer {jwt}
```

JWTs are valid for one hour; reuse within the window.

**Certificate-based (legacy):** Mutual TLS with an APNs certificate. Simpler to set up,
harder to rotate.

### Delivery modes

APNs has two delivery modes; servers SHOULD choose based on urgency requirements:

**Background push** (`apns-push-type: background`, `apns-priority: 5`,
`"content-available": 1`, no `"alert"` key): the OS wakes the app silently; the app
constructs and posts the notification. Delivered opportunistically — servers SHOULD
NOT use background push for high-urgency notifications.

**Alert push** (`apns-push-type: alert`, `apns-priority: 10`, with an `"alert"` key):
the OS can display a notification immediately. Servers SHOULD use a
[Notification Service Extension](https://developer.apple.com/documentation/usernotifications/modifying_content_in_newly_delivered_notifications)
to intercept and rewrite the placeholder before display: add
`"alert": {"body": "New message"}` and `"mutable-content": 1` to `"aps"`.

### Request

```http
POST https://api.push.apple.com/3/device/{device_token}
Authorization: bearer {jwt}
apns-topic: {bundle_id}
apns-push-type: {background|alert}
apns-priority: {5|10}
apns-expiration: {unix_timestamp}
Content-Type: application/json

{ ...see supplement for payload structure... }
```

Servers SHOULD set `apns-expiration` to limit how long APNs queues an undelivered
notification; omitting it queues indefinitely. Shorter expiries suit higher urgency
(e.g. 1 hour for `"high"`, 24 hours for `"normal"`).

### Urgency

| JMAP urgency | Background mode | Alert mode |
|---|---|---|
| `"high"` | priority `5` / `background` | priority `10` / `alert` |
| `"normal"` | priority `5` / `background` | priority `5` / `alert` |
| `"low"` | priority `5` / `background` | priority `5` / `background` |
| `"very-low"` | priority `5` / `background` | priority `5` / `background` |

In alert mode, `"high"` uses priority `10` (immediate) and `"normal"` uses priority `5`
(power-efficient). `"low"` and `"very-low"` remain background regardless of mode.

Background push cannot guarantee timely delivery regardless of urgency. For prompt
delivery of high-urgency notifications, servers MUST use alert mode.

### Error responses

APNs returns a JSON body with a `reason` field on 4xx/5xx responses. HTTP status alone
is not sufficient — servers MUST parse `reason`:

- `BadDeviceToken` / `DeviceTokenNotForTopic` — servers MUST destroy the
  `PushSubscription`.
- `Unregistered` / `ExpiredToken` — servers MUST destroy the `PushSubscription`.
- `TooManyRequests` — servers SHOULD back off and retry.
- `BadTopic` — wrong `apns-topic` value; configuration error, not a subscription
  error. Servers MUST NOT destroy the `PushSubscription` in this case.

---

## Web Push (RFC 8030 / RFC 8291)

The native JMAP push platform. Urgency values map directly with no translation, payloads
are end-to-end encrypted, and there is no vendor lock-in. Browsers, ntfy, and UnifiedPush
distributors all expose standard Web Push endpoints that fit `PushSubscription.url`
natively — no out-of-band registration step is needed.

### Registration

The client obtains a subscription object from the platform Push API (`PushManager`,
ntfy SDK, UnifiedPush library, etc.) containing:

- `endpoint` — the push URL; this becomes `PushSubscription.url`
- `keys.p256dh` — the client's ECDH public key (base64url)
- `keys.auth` — 16-byte authentication secret (base64url)

Clients MUST store all three. Servers need `p256dh` and `auth` to encrypt each push.

### Auth (VAPID)

Servers MUST generate a P-256 ECDSA key pair once at setup and persist the private key.
Servers MUST include the VAPID header on every request:

```
Authorization: vapid t={jwt},k={base64url_public_key}
```

JWT claims: `aud` (push service origin, e.g. `https://push.example.com`), `sub`
(operator contact URI, e.g. `mailto:admin@example.com`), `exp` (at most 24 hours from
now).

### Payload encryption

Servers MUST encrypt payload bytes per RFC 8291 (`aes128gcm`) using the client's
`p256dh` and `auth` values. Libraries: `web-push` (Node.js), `pywebpush` (Python),
`web-push` crate (Rust), `webpush-go` (Go).

### Request

```http
POST {endpoint}
Authorization: vapid t={jwt},k={public_key}
Content-Type: application/octet-stream
Content-Encoding: aes128gcm
Urgency: {urgency}
TTL: {ttl_seconds}

{encrypted payload bytes}
```

Servers SHOULD set `TTL` to reflect how stale a notification can be before it becomes
noise (e.g. 3600 for `"high"`, 86400 for `"normal"`).

### Urgency

Web Push urgency values are identical to JMAP's. Servers MUST pass them through
directly — no translation needed:

| JMAP urgency | `Urgency` header |
|---|---|
| `"high"` | `high` |
| `"normal"` | `normal` |
| `"low"` | `low` |
| `"very-low"` | `very-low` |

### Payload size

RFC 8291 `aes128gcm` adds 103 bytes of fixed overhead: an 86-byte content-coding header
(16-byte salt, 4-byte record size, 1-byte key length, 65-byte sender public key), a
16-byte AES-GCM authentication tag, and a 1-byte padding delimiter. Servers MUST keep
plaintext under **3993 bytes** to stay within the 4096-byte ciphertext limit.

### ntfy and UnifiedPush

ntfy exposes standard Web Push endpoints — servers MUST treat it identically to any
other Web Push delivery; no special casing is required.

UnifiedPush routes through user-chosen distributors (ntfy, Gotify, a self-hosted relay,
etc.). Servers MUST treat any UnifiedPush endpoint identically to Web Push. RFC 8291
encryption support varies by distributor. Servers SHOULD test with an encrypted payload
against each distributor they plan to support and MAY fall back to unencrypted delivery
for distributors that reject encrypted payloads. Operators MUST be aware that
unencrypted delivery exposes payload content to the distributor infrastructure.

---

## Microsoft WNS (Windows Push Notification Services)

Delivers to Windows desktop and UWP applications. WNS is URL-based: the client receives
a channel URI and the server POSTs to it, fitting `PushSubscription.url` natively — no
out-of-band registration needed.

Channel URIs expire after approximately 30 days. Clients MUST renew and update
`PushSubscription.url` before expiry.

### Auth

WNS uses OAuth 2.0 client credentials. Obtain Package SID and client secret from the
Microsoft Partner Center:

```
POST https://login.live.com/accesstoken.srf
grant_type=client_credentials&client_id={package_sid}&client_secret={secret}&scope=notify.windows.com
```

Implementers SHOULD verify this endpoint against current Microsoft WNS documentation —
Microsoft has been migrating authentication infrastructure. Tokens are valid for 24
hours; servers SHOULD cache and refresh proactively.

### Request

```http
POST {channel_uri}
Authorization: Bearer {oauth_token}
Content-Type: application/octet-stream
X-WNS-Type: wns/raw

{UTF-8 JSON bytes}
```

Servers MUST use `X-WNS-Type: wns/raw` with `Content-Type: application/octet-stream`.
This delivers raw bytes to the app's background task handler, which constructs and
posts the notification. The alternative `wns/toast` causes Windows to generate a
notification from an XML template before the app runs — servers MUST NOT use
`wns/toast` for JMAP push.

WNS has no delivery priority field for raw notifications; urgency is carried in the
payload and the app's background task handler MAY use it to influence local
notification priority after delivery.

### Error responses

WNS reports errors in response headers, not the response body. Servers MUST check on
every response:

- `X-WNS-NotificationStatus`: `received`, `dropped`, or `channelthrottled` — a `200 OK`
  with `dropped` means the notification was not delivered.
- `X-WNS-Error-Description`: human-readable detail.

Key status codes:

- `401 Unauthorized` — token expired; servers SHOULD refresh and retry.
- `404 Not Found` / `410 Gone` — channel URI invalid; servers MUST destroy the
  `PushSubscription`.
- `406 Not Acceptable` / `channelthrottled` — rate limited; servers SHOULD back off
  and retry.

---

## Delivery failures and subscription lifecycle

Success codes differ by platform: Web Push returns `201 Created`; all others return
`200 OK`. For FCM, a `200` does not guarantee delivery — servers MUST always check the
response body.

| HTTP status | Meaning | Action |
|---|---|---|
| `200 OK` | Accepted (non-Web Push) | For FCM: server MUST check `error.status` in body. For others: none. |
| `201 Created` | Accepted (Web Push) | None |
| `400 Bad Request` | Malformed payload | Server SHOULD log and fix; MUST NOT retry this payload |
| `401 Unauthorized` | Credential expired | Server MUST refresh credential and retry |
| `403 Forbidden` | App unregistered or token invalid | Server MUST destroy the `PushSubscription` |
| `404 Not Found` | Endpoint gone | Server MUST destroy the `PushSubscription` |
| `410 Gone` | Subscription explicitly expired | Server MUST destroy the `PushSubscription` |
| `413 Payload Too Large` | Over size limit | Server SHOULD truncate and retry |
| `429 Too Many Requests` | Rate limited | Server SHOULD back off per `Retry-After`; MAY fall back to `StateChange` |
| `5xx` | Push service error | Server SHOULD retry with exponential backoff |

When the server falls back from inline push to `StateChange` due to rate limiting,
clients MUST handle it identically to an unaugmented `StateChange`: by calling
`Message/changes` to determine which messages are new.

---

## Security

### Vendor-visible payloads

Only Web Push (RFC 8291) is end-to-end encrypted — only the client device can decrypt the
payload. All other platforms (FCM, ADM, HPK, APNs, MiPush, WNS) route traffic through
vendor infrastructure that can read the payload in transit. Servers SHOULD NOT include
sensitive content — such as message body snippets — in push payloads delivered over
non-Web-Push channels.

### Credential security

OAuth tokens (FCM, ADM, HPK, WNS) SHOULD be cached and refreshed proactively before
expiry, and MUST NOT be reused past their validity window. Static keys (MiPush App
Secret) MUST be stored as server-side secrets — they MUST NOT be logged and MUST NOT
be embedded in client code. APNs token-based signing keys (`.p8` files) cannot be
revoked through Apple's infrastructure; implementations MUST treat them as long-lived
secrets requiring equivalent protection to private CA keys.

### Push channel encryption and message E2E encryption are independent

Push channel encryption and message body end-to-end encryption operate at different
layers and are independent of each other. A message body may be E2E encrypted regardless
of whether the push channel is encrypted, and vice versa. The push payload (for example,
a state-change notification or a body snippet) is separate from the message body: securing
one does not secure the other.
