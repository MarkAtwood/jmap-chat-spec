# JMAP Scene WebSocket — Implementer's Guide

For client and server implementers. Covers WebSocket connection setup, capability
negotiation, subscription control, ephemeral event handling, rate limiting, visibility
filtering, and interop with Chat and VTC WebSocket subscriptions for
`draft-atwood-jmap-scene-wss-00`.

Read `draft-atwood-jmap-scene-wss-00`, `draft-atwood-jmap-scene-00`, and RFC 8887 first.
This guide does not re-explain the message formats or protocol requirements — it covers
the implementation decisions the spec intentionally leaves open.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-scene-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or deployment
  choice (e.g., "servers SHOULD throttle object-update fan-out"), the keyword reflects
  implementer best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## Scene ephemeral events vs. other event tracks

A JMAP WebSocket connection can carry three independent categories of server-to-client
data. Understanding where Scene ephemeral events fit is essential before implementing
anything.

**State-change events** (`StateChange` frames, via `WebSocketPushEnable`): carry state
tokens for the four Scene data types (SceneRegion, SceneObject, SceneAvatar, SceneAsset);
require the client to follow up with `/changes` calls; survive reconnection via
`pushState` catch-up. These track persistent state — objects created, avatars that
joined, regions modified.

**Scene ephemeral events** (`SceneAvatarEvent`, `SceneObjectEvent`,
`SceneInteractionEvent`, via `SceneStreamEnable`): carry no state token; require no
follow-up API call; are not durable — the server does not buffer them for disconnected
clients. On reconnect, missed ephemeral events are gone. These are fire-and-forget
signals that drive spatial UI updates: an avatar appeared, an object was placed, a user
clicked something.

**High-frequency simulation data** (avatar position at 10+ Hz, physics updates): does NOT
travel over the JMAP WebSocket. That data belongs on the simulation layer behind
`simulationUri`. Scene ephemeral events are discrete, low-frequency signals that
complement the simulation layer — they are not a replacement for it.

Receiving a `SceneAvatarEvent` with `event: "entered"` MUST NOT trigger a
`SceneAvatar/changes` call. The event itself contains enough information (avatar id,
display name, region) for the UI update. If the client needs the full SceneAvatar record
(visual representation, custom properties), it MAY issue a `SceneAvatar/get`, but this is
optional and should not be the default path.

Similarly, receiving a `SceneObjectEvent` with `event: "created"` or `"updated"` MAY
trigger a `SceneObject/get` if the client needs the full object state, but the event
provides enough information (id, position, event type) for many UI updates without a
round-trip.

---

## When the server does not deliver events

The server silently drops ephemeral events under specific conditions. Clients MUST
account for this when interpreting the absence of an expected event.

### SceneAvatarEvent suppression

The server silently drops a `SceneAvatarEvent` for a recipient when:

- The recipient does not have an active avatar in the event's region at delivery time.
- The recipient's subscription does not include the event's region (explicit `regionIds`
  that do not contain this region).
- The recipient's subscription does not include `"SceneAvatarEvent"` in `eventTypes`.

### SceneObjectEvent suppression

The server silently drops a `SceneObjectEvent` for a recipient when:

- The object would not be returned in a `SceneObject/get` response for this user (the
  visibility contract).
- Server-side visibility filtering (radius-based, occlusion-based) determines the object
  is outside the recipient's visibility scope.
- The event exceeds the rate limit (2 updates per object per second).

### SceneInteractionEvent suppression

The server silently drops a `SceneInteractionEvent` for a recipient when:

- The interacted object is not visible to the recipient per the visibility contract.
- The event exceeds the rate limit (5 per user per region per second).

### Implication for clients

Clients MUST NOT infer from the absence of events that the underlying conditions have
changed. A missing `SceneAvatarEvent` could mean the avatar entered outside subscription
scope, or the server suppressed it for visibility reasons. Clients that need authoritative
state SHOULD fall back to `SceneAvatar/get` or `SceneObject/get`.

---

## Client implementation

### Checking capabilities

Before opening a WebSocket connection, clients MUST fetch the JMAP Session object and
verify the required capabilities are present:

- `urn:ietf:params:jmap:websocket` — the WebSocket transport (RFC 8887); provides the
  `url` field containing the `wss://` endpoint
- `urn:ietf:params:jmap:scene` — the Scene data model and methods
- `urn:ietf:params:jmap:scene:websocket` — ephemeral Scene event support

If `urn:ietf:params:jmap:websocket` is absent, WebSocket is unavailable. If
`urn:ietf:params:jmap:scene:websocket` is absent but the first two are present, the
WebSocket transport works and the Scene data types are available via standard JMAP
methods, but ephemeral Scene events are not supported; clients MUST NOT send
`SceneStreamEnable` in that case.

Example Session object:

```json
{
  "capabilities": {
    "urn:ietf:params:jmap:websocket": {
      "url": "wss://scene.example.com/jmap/ws/",
      "supportsPush": true
    },
    "urn:ietf:params:jmap:scene": {},
    "urn:ietf:params:jmap:scene:websocket": {}
  }
}
```

### Startup sequence

After the WebSocket handshake completes, send these messages in order:

1. **`WebSocketPushEnable`** with the Scene data types your UI needs. Include `pushState`
   if you have a stored token from a previous session:

   ```json
   {
     "@type": "WebSocketPushEnable",
     "dataTypes": ["SceneRegion", "SceneObject", "SceneAvatar"]
   }
   ```

2. **Enter a region** by calling `SceneAvatar/set` with a `create`:

   ```json
   [["SceneAvatar/set", {
     "accountId": "account-xyz",
     "create": {
       "av0": {
         "regionId": "01J5ABC0000000000000000001"
       }
     }
   }, "0"]]
   ```

3. **`SceneStreamEnable`** to subscribe to ephemeral events for the region:

   ```json
   {
     "@type": "SceneStreamEnable",
     "regionIds": ["01J5ABC0000000000000000001"],
     "eventTypes": null
   }
   ```

The ordering matters. Send `WebSocketPushEnable` first so state-change notifications
begin flowing immediately. Enter the region before enabling the scene stream — the
server silently ignores `regionIds` entries where the user does not have an active
avatar.

### SceneStreamEnable: region-scoped filtering

`SceneStreamEnable` provides two filtering dimensions:

**Region filtering** via `regionIds`:

- `null` — dynamic subscription: events for all regions where the user has an active
  avatar. As the user enters or leaves regions, the event scope adjusts automatically.
- Explicit list — static subscription: only events from the listed regions. If the user
  leaves a listed region, events stop; if they re-enter, events resume without a new
  `SceneStreamEnable`.
- Empty array — no events delivered; effectively the same as `SceneStreamDisable` but
  the subscription remains "active" (a subsequent `SceneStreamEnable` replaces it).

**Event-type filtering** via `eventTypes`:

- `null` — all three event types.
- Explicit list — only the listed types. Recognized values: `"SceneAvatarEvent"`,
  `"SceneObjectEvent"`, `"SceneInteractionEvent"`.
- Empty array or all-unrecognized — the server responds with a `RequestError`
  (`invalidArguments`) and does not update the subscription.

A client that only cares about social presence (who is in the region) but not object
changes or interactions:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": null,
  "eventTypes": ["SceneAvatarEvent"]
}
```

A client monitoring a specific region for object changes (e.g., a collaborative editor):

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": ["SceneObjectEvent"]
}
```

### Replacing vs. disabling subscriptions

`SceneStreamEnable` is a full replacement — there is no partial-update mechanism. Every
`SceneStreamEnable` replaces the previous subscription entirely. Clients MUST re-send the
full desired subscription state whenever they change scope.

`SceneStreamDisable` cancels the Scene-specific subscription. It takes no parameters:

```json
{
  "@type": "SceneStreamDisable"
}
```

The server MUST silently succeed if no subscription is active. Other subscriptions
(Chat, VTC, `WebSocketPushEnable`) are unaffected.

### Connection lifecycle

**When to connect:** clients SHOULD open the WebSocket when the user is about to enter a
spatial environment. Unlike Chat, where presence and typing events are useful from the
moment the app launches, Scene events are only relevant when the user has an active
avatar in a region.

**Keepalives:** the WebSocket layer handles ping/pong at the transport level;
application-level heartbeats are not needed. If the platform's WebSocket implementation
does not send pings automatically, clients SHOULD send a WebSocket ping frame every
30-60 seconds on idle connections.

**Reconnection:** on any unclean close (non-1000 close code, network error, or timeout),
clients SHOULD reconnect with exponential backoff starting at 1 second, capped at 60
seconds, with jitter. On reconnect, clients MUST repeat the full startup sequence:
`WebSocketPushEnable` (with stored `pushState`), re-enter the region if needed, then
`SceneStreamEnable`. Ephemeral events are not buffered — anything missed during the
disconnect is gone.

**When to disconnect:** clients SHOULD disconnect when the user leaves all spatial
environments or when the app is about to be suspended. Send a clean WebSocket close
(code 1000) so the server can release subscription state promptly.

### Credential expiry

The server closes the connection with code `1008` when authentication credentials expire.
The handling is identical to the Chat WSS case: re-authenticate, then reconnect with the
new credentials and repeat the startup sequence. Do not clear the stored `pushState` on
a `1008` close.

### Handling SceneAvatarEvent

`SceneAvatarEvent` signals avatar presence transitions:

```json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "entered",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
```

On `"entered"`: add the avatar to the region's presence list. The `displayName` is the
avatar's display name at event time — use it directly for UI. If the client needs the
full avatar record (visual representation, custom properties), issue a `SceneAvatar/get`.

On `"left"`: remove the avatar from the presence list.

On `"ejected"`: remove the avatar from the presence list. This is an administrative
action (the user was kicked). Clients MAY display a different visual treatment than
a voluntary departure.

Clients MUST NOT persist avatar presence across reconnects. On reconnect, clear the local
presence list and repopulate from `SceneAvatar/query` with `isActive: true` for the
current region; avatars will reappear via ephemeral events once the new subscription is
active.

### Handling SceneObjectEvent

`SceneObjectEvent` signals object lifecycle changes:

```json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "created",
  "updatedBy": "user:bob@example.com",
  "position": [8.0, 1.2, -3.5]
}
```

On `"created"`: the client knows a new object exists at a given position. To render it,
issue a `SceneObject/get` for the full record (visual reference, scale, orientation). For
simpler UIs (e.g., a 2D map), the position alone may suffice.

On `"updated"`: the object changed. The `position` field gives the new position; for
other property changes (visual, scale, custom properties), issue a `SceneObject/get`. Do
not issue `SceneObject/get` on every update — if the event rate is high, batch requests
or skip intermediate updates.

On `"destroyed"`: remove the object from the scene. `position` is `null`. No follow-up
call needed.

`updatedBy` is `null` for server-initiated changes (physics simulation, scripted
behavior). Clients MAY use this to distinguish user actions from system actions in UI
(e.g., showing "Bob placed an object" vs. silently updating).

### Handling SceneInteractionEvent

`SceneInteractionEvent` signals user interactions with objects:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000001",
  "userId": "user:alice@example.com",
  "action": "click",
  "data": null
}
```

Standard actions: `"click"`, `"grab"`, `"release"`, `"activate"`. Deployment-specific
actions use reverse-domain notation (e.g., `"com.example.puzzle.solve"`). Clients MUST
ignore unrecognized action values — do not close the connection or log errors for unknown
actions; new actions may be introduced by deployments at any time.

The `data` field is opaque — the server relays it without interpretation. Clients that
understand the action type MAY use `data`; clients that do not MUST ignore it. Do not
trust `data` contents for security-critical decisions.

Interaction events are purely ephemeral. They are not persisted and carry no state.
Clients SHOULD use them to drive transient visual feedback (highlight the clicked object,
show a grab animation) rather than permanent state changes.

### Handling interleaved frames

Scene ephemeral events, `StateChange` frames, Chat ephemeral events, VTC ephemeral
events, and JMAP `Response` frames all arrive on the same WebSocket connection.
Distinguish them by the `@type` field:

| `@type` value              | Category           | Action                               |
|----------------------------|--------------------|--------------------------------------|
| `StateChange`              | State-change push  | Issue `/changes` calls               |
| `Response`                 | JMAP response      | Process as method response           |
| `SceneAvatarEvent`         | Scene ephemeral    | Update presence UI                   |
| `SceneObjectEvent`         | Scene ephemeral    | Update scene graph                   |
| `SceneInteractionEvent`    | Scene ephemeral    | Show interaction feedback            |
| `ChatTypingEvent`          | Chat ephemeral     | Show typing indicator                |
| `ChatPresenceEvent`        | Chat ephemeral     | Update contact presence              |
| `VTCParticipantEvent`      | VTC ephemeral      | Update participant list              |
| (unrecognized)             | Unknown            | Silently ignore                      |

Clients MUST silently ignore messages with unrecognized `@type` values. Do not close the
connection — future spec extensions will add new event types.

---

## Interop with Chat WSS and VTC WSS

### Convenience path: ChatStreamEnable with "scene"

When the server advertises both `urn:ietf:params:jmap:chat:websocket` and
`urn:ietf:params:jmap:scene:websocket`, clients can subscribe to Scene events through
`ChatStreamEnable` by including `"scene"` in `dataTypes`:

```json
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence", "scene"],
  "chatIds": null,
  "contactIds": null
}
```

This subscribes to all Scene event types for all regions where the user has an active
avatar. It is the coarse-grained path — no region or event-type filtering. For clients
that display a Chat UI with an embedded Scene view, this avoids sending a separate
`SceneStreamEnable`.

### Subscription union

If the client has both an active `ChatStreamEnable` (with `"scene"`) and an active
`SceneStreamEnable`, the server delivers the union. The server MUST NOT deliver
duplicates — if an event matches both subscriptions, it is delivered exactly once.

This means a client can use `ChatStreamEnable` for broad coverage and
`SceneStreamEnable` for fine-grained filtering on a specific region without worrying
about duplicate events.

### Subscription independence

Disabling one subscription does not affect the other:

- `SceneStreamDisable` cancels only the `SceneStreamEnable` subscription. Scene events
  still flow through an active `ChatStreamEnable` that includes `"scene"`.
- `ChatStreamDisable` cancels only the `ChatStreamEnable` subscription. Scene events
  still flow through an active `SceneStreamEnable`.

This independence is critical for UI transitions. A client that navigates from a Chat
view (with `ChatStreamEnable` including `"scene"`) to a dedicated Scene view can send
`SceneStreamEnable` with explicit `regionIds` and later `ChatStreamDisable` without
losing Scene events.

### VTC interop

Scene and VTC ephemeral subscriptions are fully independent. `SceneStreamEnable` /
`SceneStreamDisable` does not affect VTC events, and vice versa.

When a SceneRegion has an `activeCallId` binding to a VTCCall, a client typically
subscribes to both:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
```

```json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
  "eventTypes": null
}
```

Both deliver events on the same connection. The client distinguishes them by `@type`.

### Choosing a subscription strategy

| Scenario                                    | Recommended approach                           |
|---------------------------------------------|------------------------------------------------|
| Chat app with embedded 3D scene view        | `ChatStreamEnable` with `"scene"` in dataTypes |
| Dedicated 3D viewer, single region          | `SceneStreamEnable` with explicit `regionIds`  |
| Multi-region dashboard (monitoring)         | `SceneStreamEnable` with `regionIds: null`     |
| Scene + VTC in same region                  | Both `SceneStreamEnable` and `VTCStreamEnable` |
| Scene without Chat capability               | `SceneStreamEnable` (only option)              |

Clients displaying a single region SHOULD prefer `SceneStreamEnable` with an explicit
`regionIds` list. This minimizes server fan-out and network traffic.

---

## Server implementation

### Fan-out architecture

Delivering Scene ephemeral events efficiently requires a pub/sub layer between your JMAP
application logic and your WebSocket connection handlers.

A workable structure:

- Each WebSocket connection handler maintains its Scene ephemeral subscription state (the
  current `regionIds` and `eventTypes` filter from the last `SceneStreamEnable`).
- Publish avatar, object, and interaction events to topics keyed by `(accountId,
  regionId)` for all three event types.
- Each connection handler subscribes to the topics matching its active `regionIds` filter
  when `SceneStreamEnable` is received and unsubscribes when `SceneStreamDisable` is
  received or the connection closes.
- For `regionIds: null` subscriptions, the handler subscribes to all regions where the
  user currently has an active avatar. The server MUST dynamically update these
  subscriptions when the user enters or leaves a region.

The delivery path is: application logic publishes event to region topic, pub/sub layer
fans out to subscribed handlers, each handler checks (1) event-type filter, (2) avatar
presence, (3) visibility contract, and sends the frame if all checks pass.

### Authorization at delivery time

Servers MUST verify authorization at the time of each event delivery, not only when
`SceneStreamEnable` is received. Region access and avatar presence can change while the
connection is open:

- All Scene ephemeral events MUST only be delivered for regions where the user has an
  active avatar at delivery time. If the user's avatar was ejected between subscription
  and delivery, the event MUST be dropped.
- `SceneObjectEvent` MUST respect the visibility contract: if the server would not return
  this object in a `SceneObject/get` response to this user, it MUST NOT deliver the
  ephemeral event.
- `SceneInteractionEvent` for objects the user cannot see MUST NOT be delivered.

Perform these checks in the delivery path, not at subscribe time. This is simpler —
implementations need not track avatar presence changes and update subscriptions
reactively.

### Visibility filtering

The visibility contract applies equally to ephemeral events and to `SceneObject/get`
responses. The same filtering logic should be reused:

- A simple deployment that returns all objects in `SceneObject/get` delivers all
  `SceneObjectEvent` frames.
- A deployment with radius-based filtering suppresses `SceneObjectEvent` for objects
  outside the recipient avatar's visibility radius.
- A deployment with occlusion culling suppresses events for objects not in the avatar's
  potential visibility set.

The key implementation detail: the visibility check must use the avatar's position at
delivery time, which may be the last known position from the simulation layer
reconciliation (updated every 5-30 seconds). Servers SHOULD NOT attempt real-time
position lookups from the simulation layer for each event delivery — the periodic
reconciliation is sufficient for this purpose.

### Rate limiting

The spec defines two rate limits. Both are per outbound connection — each connected
client has its own independent budget.

**SceneObjectEvent** with `event: "updated"`: no more than 2 events per object per
second. Track the last delivery timestamp in a structure keyed by `(connectionId,
regionId, objectId)`. When an event would exceed the limit, drop it silently. When the
rate window reopens, deliver the most recent state — clients should see the latest
position, not an intermediate one. Object updates at higher frequency belong on the
simulation layer, not the JMAP WebSocket.

**SceneInteractionEvent**: no more than 5 events per user per region per second. Track
with a structure keyed by `(connectionId, regionId, userId)`. Drop silently when
exceeded.

These structures are per-connection and in-memory. When a connection closes, discard them.

Servers SHOULD also apply an inbound rate limit on `SceneStreamEnable` — a client that
sends it in a tight loop causes unnecessary churn on the pub/sub layer. A limit of a few
per second per connection is reasonable.

### State-change push

State-change delivery for Scene data types (SceneRegion, SceneObject, SceneAvatar,
SceneAsset) via `WebSocketPushEnable` reuses RFC 8887's existing mechanism. The concerns
are identical to Chat: assign a new `pushState` token with each delivery, retain enough
history for reconnect replay (RECOMMENDED: 10-15 minutes minimum).

Note that Scene ephemeral events and Scene state-change events can overlap in subject
matter. A `SceneObject/set` create will produce both a `StateChange` for SceneObject and
a `SceneObjectEvent` with `event: "created"`. These are independent signals — the
`StateChange` drives the sync protocol, the ephemeral event drives the real-time UI.
Clients MUST handle both arriving and MUST NOT treat them as duplicates.

### Handling ChatStreamEnable with "scene"

When the server advertises both Chat and Scene WebSocket capabilities, `ChatStreamEnable`
with `"scene"` in `dataTypes` activates a Scene ephemeral subscription with `regionIds:
null` and `eventTypes: null`. The server MUST compute the union with any active
`SceneStreamEnable` subscription and deduplicate events.

Implementation: maintain two subscription records per connection — one for the Chat path
and one for the Scene-specific path. At delivery time, check whether the event matches
either subscription. If it matches both, deliver once. If the client sends
`SceneStreamDisable`, clear only the Scene-specific record. If the client sends
`ChatStreamDisable`, clear only the Chat-path record (including the `"scene"` component).

---

## Security

### TLS

Clients MUST always connect to `wss://`, never `ws://`. The Session object's WebSocket
URL will always be `wss://`; clients MUST reject any URL that does not begin with
`wss://`.

### Event authorization at delivery time

The server checks avatar presence, region access, and object visibility at the moment of
frame delivery, not at subscribe time. Clients receive only events they are authorized to
see; no additional client-side filtering is needed.

This also means a client does not need to update its subscription scope when another
user's avatar is ejected or when an object's visibility changes — the server handles
suppression at delivery.

### Spatial presence privacy

`SceneAvatarEvent` reveals when users enter and leave regions. This is intentional — it
enables social presence features — but it is privacy-sensitive. The `displayName` field
exposes the avatar's name at event time.

Deployments that require avatar anonymity SHOULD consider:

- Suppressing `displayName` (sending `null` or a generic placeholder) in regions where
  anonymity is desired.
- Using pseudonymous `userId` values that do not map to real identities.
- Allowing users to opt out of `SceneAvatarEvent` delivery to other users (analogous to
  `Chat.receiveTypingIndicators`).

The server observes who enters which regions, when, and for how long. This metadata is
privacy-sensitive. Deployments requiring spatial privacy SHOULD minimize logging and
retention of avatar movement data.

### Interaction event data exposure

`SceneInteractionEvent` carries an opaque `data` field that the server relays without
interpretation. Deployments SHOULD validate that `data` payloads do not exceed a
deployment-defined size limit. Clients MUST NOT trust `data` contents for
security-critical decisions — a malicious user can send arbitrary payloads.

### Credential handling

Clients MUST NOT attempt to reconnect with expired credentials after a `1008` close.
Implementations MUST NOT log session tokens or authentication credentials from WebSocket
handshake headers.

---

## Example: full WebSocket session lifecycle

The following shows a complete session from connect to close, with all message types
interleaved as they would appear on the wire.

### 1. WebSocket handshake

```
GET /jmap/ws/ HTTP/1.1
Host: scene.example.com
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer eyJ0eXAi...
Sec-WebSocket-Protocol: jmap
```

### 2. Enable state-change push

Client enables push for Scene data types:

```json
{
  "@type": "WebSocketPushEnable",
  "dataTypes": ["SceneRegion", "SceneObject", "SceneAvatar"]
}
```

### 3. Enter a region

Client creates an avatar in the target region:

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "create": {
    "av0": {
      "regionId": "01J5ABC0000000000000000001"
    }
  }
}, "0"]]
```

Server responds with the created avatar:

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "created": {
    "av0": {
      "id": "user:alice@example.com",
      "regionId": "01J5ABC0000000000000000001",
      "userId": "user:alice@example.com",
      "displayName": "Alice Chen",
      "position": [0, 0, 10],
      "orientation": [0, 0, 0, 1],
      "joinedAt": "2026-06-05T14:35:00Z",
      "leftAt": null
    }
  }
}, "0"]]
```

### 4. Enable Scene event stream

Client subscribes to all event types for the region:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
```

### 5. Another avatar enters

Server delivers an avatar-enter event:

```json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "entered",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
```

### 6. Object placed

Bob places an object; server delivers a create event:

```json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "created",
  "updatedBy": "user:bob@example.com",
  "position": [8.0, 1.2, -3.5]
}
```

Client optionally fetches the full object:

```json
[["SceneObject/get", {
  "accountId": "account-xyz",
  "ids": ["01J5OBJ0000000000000000042"]
}, "1"]]
```

### 7. User interaction

Alice clicks an interactable object:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000001",
  "userId": "user:alice@example.com",
  "action": "click",
  "data": null
}
```

### 8. State change (interleaved)

A JMAP state-change notification arrives on the same connection:

```json
{
  "@type": "StateChange",
  "changed": {
    "account-xyz": {
      "SceneObject": "s301"
    }
  }
}
```

Client processes this via the normal `/changes` path.

### 9. Object updated (rate-limited)

Bob moves the object several times in quick succession. The server delivers at most 2
updates per second for this object, dropping intermediate positions:

```json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "updated",
  "updatedBy": "user:bob@example.com",
  "position": [12.0, 1.2, -4.0]
}
```

### 10. Bob leaves

Server delivers an avatar-leave event:

```json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "left",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
```

### 11. Disable Scene stream

Client leaves the region and cancels its Scene subscription:

```json
{
  "@type": "SceneStreamDisable"
}
```

### 12. WebSocket close

```
Client sends Close frame (1000, "leaving")
Server responds with Close frame (1000)
```
