# JMAP Chat WebSocket — Implementer's Guide

For client and server implementers. Covers connection lifecycle, startup sequencing,
credential expiry, event handling, and fan-out architecture for
`draft-atwood-jmap-chat-wss-00`.

Read `draft-atwood-jmap-chat-wss-00` and RFC 8887 first. This guide does not re-explain
the message formats or protocol requirements — it covers the implementation decisions the
spec intentionally leaves open.

---

## The two event tracks

A WebSocket connection carries two independent streams of server-to-client events that
require completely different handling:

**State-change events** (`StateChange` frames, via `WebSocketPushEnable`): reference
state tokens; require the client to follow up with `/changes` calls to retrieve updated
objects; survive reconnection via `pushState` catch-up; and when the server retains
history, allow missed notifications to be replayed on reconnect.

The state token + `/changes` + `cannotCalculateChanges` path is a verified client-server
consistency protocol: a client that follows it correctly is guaranteed to reach a
consistent view of server state. This is what JMAP WebSocket provides on the sync side —
efficient, verified client-server synchronization. How a server keeps data consistent
across nodes or between federation peers is implementation-specific and out of scope for
this spec.

**Ephemeral events** (`ChatTypingEvent`, `ChatPresenceEvent`, via `ChatStreamEnable`):
carry no state token; require no follow-up API call; are not durable — the server does
not buffer them for disconnected clients. On reconnect, missed ephemeral events are gone.

Receiving a `ChatTypingEvent` MUST NOT trigger a `ChatContact/changes` call. Receiving a
`ChatPresenceEvent` MUST NOT trigger a `ChatContact/changes` call. Ephemeral events are
complete as received; their only job is to update transient UI state.

---

## When the server does not deliver events

The server silently drops events for a recipient under certain conditions. Clients MUST
account for this when interpreting the absence of an expected event.

### ChatTypingEvent suppression

The server silently drops a `ChatTypingEvent` for a recipient when:

- `Chat.receiveTypingIndicators` is `false` for the recipient account. This applies to
  direct and group chats only; channel chats are exempt.
- The recipient is no longer a member of the chat at delivery time.
- The sender is blocked by the recipient.

### ChatPresenceEvent suppression

The server silently drops a `ChatPresenceEvent` for a recipient when:

- The contact is blocked by the recipient.
- The recipient no longer has visibility of the contact.

### Implication for clients

Clients MUST NOT infer from the absence of events that the underlying conditions have
changed. The absence of a `ChatTypingEvent` means the user stopped typing OR the server
suppressed it — clients cannot distinguish these cases. Apply the decay timer described
in "Handling typing indicators" regardless; do not assume a missing event implies anything
about the user's typing state or membership status.

---

## Client implementation

### Checking capabilities

Before opening a WebSocket connection, fetch the JMAP Session object and verify both
capabilities are present:

- `urn:ietf:params:jmap:websocket` — the WebSocket transport (RFC 8887); provides the
  `url` field containing the `wss://` endpoint
- `urn:ietf:params:jmap:chat:websocket` — ephemeral event support

If the first is absent, WebSocket is unavailable. Fall back to EventSource (SSE) for
server-to-client state-change events, and `PushSubscription`-based push for background
and mobile delivery. HTTP polling is a last resort if neither SSE nor push is available.
If the second is absent but the first is present, the WebSocket transport works but
ephemeral events are not supported; do not send `ChatStreamEnable`.

### Startup sequence

After the WebSocket handshake completes, send these messages in order:

1. **`WebSocketPushEnable`** with the `dataTypes` your UI needs. Include `pushState`
   if you have a stored token from a previous session; omit it entirely on first connect:

   ```json
   {
     "@type": "WebSocketPushEnable",
     "dataTypes": ["Message", "Chat", "ChatContact", "ReadPosition"]
   }
   ```

   With a stored token:

   ```json
   {
     "@type": "WebSocketPushEnable",
     "dataTypes": ["Message", "Chat", "ChatContact", "ReadPosition"],
     "pushState": "{last_stored_pushState}"
   }
   ```

   The full set of subscribable data types is: `Message`, `Chat`, `ChatContact`,
   `ReadPosition`, `PresenceStatus`, `Space`, `CustomEmoji`, `SpaceBan`, `SpaceInvite`.
   Subscribe to whichever your client uses; you do not need to subscribe to all of them.

2. **`ChatStreamEnable`** if ephemeral events are supported:

   ```json
   {
     "@type": "ChatStreamEnable",
     "dataTypes": ["typing", "presence"],
     "chatIds": null,
     "contactIds": null
   }
   ```

Send both control messages before issuing JMAP API requests. The server begins delivering
events as soon as it processes each control message — if API requests arrive first,
`StateChange` frames may arrive while catch-up is still in progress. Queue any
`StateChange` frames that arrive before startup is complete and process them afterward.

`null` for `chatIds` and `contactIds` subscribes to all chats and contacts the owner
belongs to or knows about. Use explicit lists to scope delivery to a specific view (e.g.
a currently open chat window), but you will need to re-send `ChatStreamEnable` when the
user navigates to a different chat.

### Connection lifecycle

**When to connect:** open the WebSocket on app launch or when the user first needs
real-time data. Do not open it speculatively before the user has authenticated.

**Keepalives:** the WebSocket layer handles ping/pong at the transport level; application-level heartbeats are not needed. If your platform's WebSocket implementation does not
send pings automatically, send a WebSocket ping frame every 30–60 seconds on idle
connections to detect half-open TCP connections.

**Reconnection:** on any unclean close (non-1000 close code, network error, or timeout),
reconnect with exponential backoff starting at 1 second, capped at 60 seconds. Add
jitter to avoid thundering-herd on server restarts. On reconnect, repeat the full startup
sequence — `WebSocketPushEnable` with your stored `pushState`, then `ChatStreamEnable`.

**When to disconnect:** on logout, or when the OS is about to suspend the app. Send a
clean WebSocket close (code 1000) rather than dropping the TCP connection; this lets the
server release subscription state promptly. On mobile, the right moment depends on the
platform: iOS gives ~30 seconds of background execution before suspension; Android
foreground services can keep the connection alive indefinitely. Disconnecting too eagerly
on background will interrupt delivery for users who switch apps briefly.

### Credential expiry

The server closes the connection with code `1008` when authentication credentials expire
during a live session. On receiving `1008`:

1. Re-authenticate using your app's normal auth flow (refresh token, re-login, etc.)
2. Reconnect to the same WebSocket URL with the new credentials and repeat the startup
   sequence, using your stored `pushState`. The WebSocket URL does not change between
   auth cycles; only re-fetch the Session object if the reconnect itself fails.

Do not attempt to reconnect with expired credentials — the server will close again
immediately. Do not clear your stored `pushState` on a `1008` close; credential expiry
does not invalidate the state-change history you need to catch up on.

### pushState catch-up

`pushState` is how you avoid re-fetching everything after a reconnect. Every `StateChange`
frame from the server contains a `pushState` token. Store the most recent one durably
(localStorage, a database, wherever survives an app restart).

On reconnect, include it in `WebSocketPushEnable`. The server may replay `StateChange`
frames missed since that token, but the spec provides no explicit signal indicating
whether replay occurred. A server that cannot honor the stored `pushState` simply resumes
from the current state; the reconnecting client receives no indication that history was
skipped.

Because you cannot reliably tell whether replay occurred, treat `pushState` as an
optimization rather than a guarantee: after reconnecting, issue a proactive
`{Type}/changes` call for your most critical data types (at minimum, `Message`) using
your locally cached state token as `sinceState`. Handle the response normally; if
`StateChange` replay also arrived, the `/changes` response will be a no-op. On
`cannotCalculateChanges`, fall back to `{Type}/get`.

Clear your stored `pushState` on explicit logout. On fresh install or first login, omit
`pushState` from `WebSocketPushEnable` entirely.

### Handling state-change events

On receiving a `StateChange` frame:

1. Save the `pushState` token from the frame.
2. For each data type listed in `changed`, issue a `{Type}/changes` call using your
   locally cached state token as `sinceState`.
3. On `cannotCalculateChanges`, fall back to `{Type}/get`.
4. Update your local state tokens after each successful response.

`PresenceStatus` is a singleton — one record per account, tracking the owner's own
availability and custom status. A `StateChange` for this type means the owner's record
changed; the standard `/changes` to `/get` fallback applies.

Do not process `StateChange` frames while the connection is still in the startup sequence
(before you have sent both `WebSocketPushEnable` and `ChatStreamEnable`). Queue them and
process once startup is complete to avoid issuing `/changes` calls against a state you
are still catching up from.

### Handling typing indicators

On receiving `ChatTypingEvent` with `typing: true`:

- Show the typing indicator for `chatId` attributed to `senderId`
- Start or reset a decay timer for that (chatId, senderId) pair

On receiving `ChatTypingEvent` with `typing: false`:

- Hide the typing indicator immediately

Two roles are involved: the **sender** (the user who is typing, on their own client)
signals the server when they start and stop; the server delivers `ChatTypingEvent` frames
to **receivers** (other connected clients subscribed to that chat). The decay timer is on
the receiver side.

When the sender stops typing and their client signals the server, you receive
`ChatTypingEvent` with `typing: false` and can clear immediately. When the sender's
client crashes or loses connectivity, no stop signal reaches the server and no
`typing: false` arrives. The server rate-limits typing events to one per sender per chat
per 3 seconds; if you receive no further event for a (chatId, senderId) pair within
10 seconds, hide the indicator automatically.

To signal typing state to the server, a sender calls `Chat/typing` (defined in the main
JMAP Chat spec) with `chatId` and `typing: true` when the user begins typing, and
`typing: false` when they stop. The server then fans out the `ChatTypingEvent` to
participants and, for federated chats, forwards via `Peer/typing`.

`Chat.receiveTypingIndicators` is a server-side preference — when `false`, the server
drops typing events for that chat before delivering them to the recipient. Clients MUST
NOT gate calls to `Chat/typing` on the local value of this field; the server handles
suppression transparently and `Chat/typing` always succeeds from the sender's
perspective.

Do not persist typing state across reconnects. On reconnect, clear all visible typing
indicators; they will reappear if peers are still typing once the new ephemeral
subscription is active.

### Handling presence events

`ChatPresenceEvent` delivers partial updates — individual field changes when something
changes. It does not deliver a full presence snapshot on connection.

On first connect, populate your presence display from the `ChatContact` objects returned
by `ChatContact/get` — the `presence`, `lastActiveAt`, `statusText`, and `statusEmoji`
fields reflect the last-known state as of the JMAP response.

When a `ChatPresenceEvent` arrives, apply it as a delta to your local contact record:

- Always update `presence`
- Update `lastActiveAt` if present in the event
- Update `statusText` and `statusEmoji` only if the field is present in the event; a
  missing field means unchanged, `null` means explicitly cleared

On reconnect, re-fetch contact presence via `ChatContact/get` (or rely on the `pushState`
catch-up to deliver `ChatContact` state-change events for contacts whose presence changed
while you were disconnected). Do not rely on the ephemeral stream to restore presence
state after a reconnect — those events were not buffered.

---

## Server implementation

### Fan-out architecture

Delivering ephemeral events efficiently to many concurrent connections requires a pub/sub
layer between your JMAP application logic and your WebSocket connection handlers.

A workable structure:

- Each WebSocket connection handler maintains its ephemeral subscription state (the
  current `chatIds` and `contactIds` filter from the last `ChatStreamEnable`).
- Publish typing and presence state changes to topics keyed by (accountId, chatId) for
  typing and (accountId, contactId) for presence.
- Each connection handler subscribes to the topics matching its active filter when
  `ChatStreamEnable` is received and unsubscribes when `ChatStreamDisable` is received
  or the connection closes.
- The delivery path is: application logic publishes event → pub/sub layer fans out to
  subscribed handlers → each handler checks membership and block authorization at
  delivery time and sends the frame if authorized.

Check authorization at delivery time rather than at subscribe time. This is required by
the spec and also simpler — you do not need to track membership changes and update
subscriptions when a user leaves a chat or blocks a contact. Just check at the moment of
delivery and drop silently if unauthorized.

In addition to the membership check, each handler MUST check `Chat.receiveTypingIndicators`
for the recipient account before sending a `ChatTypingEvent` frame. If the field is
`false`, the handler MUST silently drop the event. This check applies to direct and group
chats only; channel chats are exempt. Perform this check in the delivery path alongside
the membership check, not at subscribe time.

### State-change push

State-change delivery via `WebSocketPushEnable` reuses RFC 8887's existing mechanism.
The main server-side concern is `pushState` management: assign a new `pushState` token
with each `StateChange` delivery and maintain enough history to replay missed frames for
any stored token a reconnecting client presents. How long to retain history is a policy
decision; a minimum of 10–15 minutes covers most transient disconnects.

When a client reconnects with a `pushState` you no longer have history for, resume
delivering from the current state without replaying history. The client has no way to
distinguish this from a successful replay — well-behaved clients issue a proactive
`/changes` call on reconnect regardless, so they will catch up either way.

### Rate limiting

The spec requires rate limiting typing events to one per sender per chat per 3 seconds
and presence events to one per contact per 30 seconds. These limits are per outbound
connection: each connected client has its own independent rate-limit budget, so two
simultaneously connected clients can each receive the same event at the full rate.

Track the last delivery timestamp in a structure keyed by `(connectionId, chatId, senderId)`
for typing and `(connectionId, contactId)` for presence, where `connectionId` is a
server-internal identifier for the WebSocket connection. Before sending a frame, check
the elapsed time; drop silently if within the rate window.

These structures are per-connection and in-memory — they do not need to be durable or
shared across nodes. When a connection closes, discard its rate-limit state.

An inbound rate limit on `ChatStreamEnable` is also advisable — a client that sends
`ChatStreamEnable` in a tight loop to change subscription scope repeatedly can cause
unnecessary churn on your pub/sub subscriptions. A limit of a few per second per
connection is reasonable.

### Credential expiry

When you detect that the credentials for an authenticated WebSocket connection have
expired (token validation failure, session revocation, etc.), the spec requires one of
two responses: return authentication error responses for subsequent requests without
closing the connection, or close the WebSocket connection with close code `1008` (Policy
Violation). Do not silently continue accepting requests as if they were authenticated.
If you close the connection, use `1008` — it is what well-behaved clients detect and
recover from.

If you support credential refresh without closing the connection (e.g. a token rotation
mechanism that the client can trigger mid-session), that is outside the scope of this
spec but not prohibited. The `1008` requirement applies only when credentials are
unrecoverably expired and the server cannot continue the session.

---

## Security

### TLS

Always connect to `wss://`, never `ws://`. The Session object's WebSocket URL will always
be `wss://`; reject any URL that does not begin with `wss://` rather than downgrading the
connection.

### Credential handling

Do not attempt to reconnect with expired credentials after a `1008` close — re-authenticate
first, then reconnect. Do not log session tokens or authentication credentials from
WebSocket handshake headers; treat them with the same care as passwords.

### Authorization at delivery time

The server checks membership and block status at the moment of frame delivery, not at
subscribe time. Clients receive only events they are authorized to see; no additional
client-side filtering is needed. This also means a client does not need to update its
subscription scope when a user leaves a chat or blocks a contact — the server handles
suppression at delivery.

### Behavioral metadata

Typing and presence signals are metadata about user behavior. Even without message
content, these signals can reveal whether a user is online and active, and with whom they
are communicating. Servers SHOULD honor `Chat.receiveTypingIndicators` and implement
presence visibility controls accordingly. Clients SHOULD make these controls visible and
accessible to users.
