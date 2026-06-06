---
title: JMAP Scene over WebSocket
abbrev: JMAP Scene WebSocket
docname: draft-atwood-jmap-scene-wss-00
category: std
stream: ietf

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC8174:
  RFC6455:
  RFC8620:
  RFC8887:
  JMAP-SCENE:
    title: JMAP for Scenes
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026

informative:
  RFC9325:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-WSS:
    title: JMAP Chat over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-VTC-WSS:
    title: JMAP VTC over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-wss-00
    date: 2026

--- abstract

This document defines a WebSocket binding for ephemeral spatial
events in JMAP for Scenes ({{JMAP-SCENE}}).  It registers the
`urn:ietf:params:jmap:scene:websocket` capability and introduces
`SceneStreamEnable` and `SceneStreamDisable` control messages that
give clients region-scoped, event-type-scoped subscription over a
JMAP WebSocket connection ({{RFC8887}}).

These ephemeral events supplement the real-time simulation layer
(which carries high-frequency position updates) and JMAP
`StateChange` notifications (which carry persistent state changes).
They provide low-latency notification of discrete spatial events —
avatar entry/exit, object placement/removal, and user interactions —
that benefit from immediate delivery but do not warrant a full
simulation-layer connection.

--- middle

# Introduction

{{JMAP-SCENE}} defines four JMAP data types for spatial state:
SceneRegion, SceneObject, SceneAvatar, and SceneAsset.  Changes to
these types produce standard JMAP `StateChange` notifications.
However, certain discrete spatial events benefit from immediate
delivery without the round-trip of a `StateChange` followed by a
`/get` call:

- An avatar entering or leaving a region (social presence signal).
- An object being placed, moved, or removed (collaborative editing).
- A user interacting with an object (click, grab, activate).

These events are fire-and-forget signals — ephemeral, not persisted
as JMAP objects — that drive spatial user interfaces.

High-frequency continuous state (avatar position at 10+ Hz, physics
simulation) is NOT carried over the JMAP WebSocket.  That data
travels through the simulation layer behind the SceneRegion's
`simulationUri`.  This document covers only the discrete,
low-frequency spatial events that complement the simulation layer
and `StateChange` notifications.

When {{JMAP-CHAT-WSS}} is present, clients can subscribe to Scene
events by including `"scene"` in the `dataTypes` array of a
`ChatStreamEnable` message.  However, a deployment MAY offer JMAP
Scene without JMAP Chat.  This document provides a standalone
subscription mechanism for that case and a finer-grained,
region-scoped alternative for deployments that have both.

# Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Capability {#capability}

## urn:ietf:params:jmap:scene:websocket

Servers supporting this specification MUST add a property named
`urn:ietf:params:jmap:scene:websocket` to the JMAP Session object
capabilities.  The value of this property is an empty JSON object
`{}`.

Presence of this capability indicates that the server supports
ephemeral Scene event push: it delivers the spatial events defined
by this document over the WebSocket connection and accepts
`SceneStreamEnable` and `SceneStreamDisable` control messages from
the client.

Servers that do not support ephemeral Scene event push MUST NOT
advertise this capability.

This capability is distinct from `urn:ietf:params:jmap:websocket`
({{RFC8887}}), which advertises the WebSocket transport itself, and
from `urn:ietf:params:jmap:scene` ({{JMAP-SCENE}}), which
advertises the Scene data model and methods.

### Session Object Example

~~~json
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
~~~

# Ephemeral Event Types {#event-types}

This specification defines three ephemeral event types for spatial
state.  These events are delivered as server-to-client text frames
over the WebSocket connection.  They are not persisted as JMAP
objects and carry no state token.

## SceneAvatarEvent {#scene-avatar-event}

Delivered when an avatar enters, leaves, or is ejected from a
region.

`@type` (String):
: `"SceneAvatarEvent"`.

`regionId` (String):
: The SceneRegion id.

`avatarId` (String):
: The SceneAvatar id.

`event` (String):
: `"entered"`, `"left"`, or `"ejected"`.

`userId` (String):
: The authenticated user identity.

`displayName` (String):
: The avatar's display name at event time.

Example — avatar enters a region:

~~~json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:alice@example.com",
  "event": "entered",
  "userId": "user:alice@example.com",
  "displayName": "Alice Chen"
}
~~~

## SceneObjectEvent {#scene-object-event}

Delivered when an object is created, updated, or destroyed within
a subscribed region.  This provides low-latency notification of
object changes; the full object state is available via
`SceneObject/get`.

`@type` (String):
: `"SceneObjectEvent"`.

`regionId` (String):
: The SceneRegion id.

`objectId` (String):
: The SceneObject id.

`event` (String):
: `"created"`, `"updated"`, or `"destroyed"`.

`updatedBy` (String|null):
: The userId of the user who made the change.  `null` for
  server-initiated changes (e.g., physics simulation, scripted
  behavior).

`position` (Number[3]|null):
: The object's position after the change.  Present for `"created"`
  and `"updated"` events; `null` for `"destroyed"`.

Example — object placed:

~~~json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "created",
  "updatedBy": "user:bob@example.com",
  "position": [15.0, 0, -5.0]
}
~~~

Example — object destroyed:

~~~json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "destroyed",
  "updatedBy": "user:bob@example.com",
  "position": null
}
~~~

## SceneInteractionEvent {#scene-interaction-event}

Delivered when a user interacts with an interactable object.  This
is a purely ephemeral signal; interactions are not persisted.  The
specific interaction semantics (what happens when an object is
clicked or grabbed) are deployment-defined.

`@type` (String):
: `"SceneInteractionEvent"`.

`regionId` (String):
: The SceneRegion id.

`objectId` (String):
: The SceneObject id of the interacted object.

`userId` (String):
: The userId of the user who performed the interaction.

`action` (String):
: The interaction type.  This specification registers `"click"`,
  `"grab"`, `"release"`, and `"activate"` as short names.
  Deployment-specific actions SHOULD use reverse-domain notation
  (e.g., `"com.example.puzzle.solve"`).  Clients MUST ignore
  unrecognized values.

`data` (Object|null):
: Action-specific payload.  Opaque to this specification; the
  server relays without interpretation.  Clients that understand
  the action type MAY use it.  `null` when no additional data is
  needed.

Example — user clicks a sculpture:

~~~json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000001",
  "userId": "user:alice@example.com",
  "action": "click",
  "data": null
}
~~~

Example — deployment-specific puzzle interaction:

~~~json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000007",
  "userId": "user:alice@example.com",
  "action": "com.example.puzzle.solve",
  "data": {"pieceIndex": 4, "rotation": 90}
}
~~~

# Scene Stream Control Messages {#stream-control}

## SceneStreamEnable {#scene-stream-enable}

A client sends a `SceneStreamEnable` message to start or replace
its ephemeral Scene event subscription on the current WebSocket
connection.

### Properties

- **`@type`** (String): MUST be `"SceneStreamEnable"`.

- **`regionIds`** (String[]|null): An explicit list of SceneRegion
  ids for which events are requested, or `null` to receive events
  for all regions in which the authenticated user has an active
  avatar.  Default: `null`.

- **`eventTypes`** (String[]|null): An explicit list of event type
  names the client wishes to receive, or `null` to receive all
  event types.  Recognized values are: `"SceneAvatarEvent"`,
  `"SceneObjectEvent"`, and `"SceneInteractionEvent"`.
  Default: `null`.

### Example

~~~json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": ["SceneAvatarEvent", "SceneObjectEvent"]
}
~~~

### Semantics

A subsequent `SceneStreamEnable` message replaces the previous
ephemeral subscription in its entirety.  There is no partial-update
mechanism; the client re-sends the full desired subscription state.

If a `regionIds` entry refers to a region in which the authenticated
user does not have an active avatar, that entry is silently ignored.
If all entries are ignored (or `regionIds` is an empty array), the
subscription is effectively empty: no events will be delivered.

Unrecognized values in `eventTypes` that appear alongside recognized
values MUST be silently ignored; only the recognized values take
effect.  If `eventTypes` is an empty array or contains only
unrecognized values, the server MUST respond with a `RequestError`
frame ({{RFC8887}} Section 4.3.4) with a `type` of
`"urn:ietf:params:jmap:error:invalidArguments"` and MUST NOT update
the current ephemeral subscription state.

### Subscription Scope

When `regionIds` is `null`, the subscription is dynamic: as the
user enters new regions or leaves existing regions, the set of
regions producing events adjusts automatically.

When `regionIds` is an explicit list, the subscription is static:
only events from the listed regions are delivered.  If the user
leaves a listed region, events for that region stop.  If the user
later re-enters the same region, events resume without a new
`SceneStreamEnable`.

## SceneStreamDisable {#scene-stream-disable}

A client sends a `SceneStreamDisable` message to cancel all
ephemeral Scene event delivery on the current connection.

### Properties

- **`@type`** (String): MUST be `"SceneStreamDisable"`.

### Example

~~~json
{
  "@type": "SceneStreamDisable"
}
~~~

### Semantics

The server MUST stop delivering Scene ephemeral events after
processing this message.  The server MUST silently succeed if no
ephemeral Scene subscription is currently active.  The WebSocket
connection and any active `WebSocketPushEnable` state-change
subscription or other ephemeral subscriptions (VTC, Chat) remain
unaffected.

# Event Delivery Semantics {#event-delivery}

## General Principles

Scene ephemeral events do not correspond to persistent state changes
and carry no state token.  They are delivered as server-to-client
text frames over the WebSocket connection when an ephemeral Scene
subscription is active.

Ephemeral event frames are interleaved with `StateChange`,
`Response`, and other ephemeral event frames (VTC, Chat) on the
same WebSocket connection and SHOULD be processed independently.

For `SceneObjectEvent` with `event: "created"` or `"updated"`, the
client MAY issue a `SceneObject/get` to retrieve the full object
state.  This is optional; the event provides enough information
(id, position, event type) for many UI updates without a round-trip.

## Event Summary

| Event Type              | Scope        | Description                    |
|-------------------------|--------------|--------------------------------|
| `SceneAvatarEvent`      | per-region   | Avatar entered, left, ejected  |
| `SceneObjectEvent`      | per-region   | Object created, updated, gone  |
| `SceneInteractionEvent` | per-region   | User interacted with object    |

All Scene ephemeral events are per-region: they are delivered only
for regions that match the active subscription's `regionIds` filter
(or all regions if `regionIds` is `null`) and for which the user has
an active avatar.

## Rate Limiting {#rate-limiting}

Servers SHOULD enforce the following rate limits to prevent resource
exhaustion:

- **`SceneObjectEvent`** with `event: "updated"`: no more than
  two events per object per second.  Events above this rate MAY be
  silently dropped; the most recent state SHOULD be delivered when
  the rate window reopens.  Object updates at higher frequency
  (e.g., physics simulation) belong on the simulation layer, not
  the JMAP WebSocket.

- **`SceneInteractionEvent`**: no more than five events per user
  per region per second.  Events above this rate MAY be silently
  dropped.

Servers MAY apply additional rate limits to other event types.

## Event Ordering

Events are delivered in best-effort order.  The server SHOULD
deliver events in the order they were generated, but clients MUST
NOT assume strict ordering.  Each event is self-contained; clients
SHOULD use the event's `regionId` and `objectId` or `avatarId`
fields to correlate events with local state.

## Visibility Filtering

The server MUST apply the same visibility contract defined in
{{JMAP-SCENE}} (Section "Visibility Contract") to ephemeral events.
The server MUST NOT deliver a `SceneObjectEvent` for an object the
client would not receive in a `SceneObject/get` response.

A deployment that performs server-side visibility filtering (e.g.,
radius-based or occlusion-based) SHOULD apply the same filter to
ephemeral events: a `SceneObjectEvent` for an object outside the
client's visibility scope SHOULD be suppressed.

## Subscription Lifecycle

The ephemeral Scene subscription is bound to the WebSocket
connection.  When the connection closes — whether by graceful close
handshake, network failure, or server-initiated close — the
subscription is immediately cancelled.  No events are buffered for
later delivery.

On reconnect, the client MUST send a new `SceneStreamEnable` to
re-establish its subscription.

# Interoperability with Other WebSocket Subscriptions {#interop}

The JMAP WebSocket connection ({{RFC8887}}) carries multiple
independent subscription channels.  When a server advertises
multiple ephemeral-event capabilities, all subscriptions coexist
on the same connection.

## Interoperability with JMAP Chat WebSocket {#chat-wss-interop}

When a server advertises both `urn:ietf:params:jmap:chat:websocket`
({{JMAP-CHAT-WSS}}) and `urn:ietf:params:jmap:scene:websocket`
(this document), two subscription paths exist for Scene ephemeral
events:

1. **`ChatStreamEnable` with `"scene"` in `dataTypes`**:  Subscribes
   to all Scene event types for all regions in which the user has
   an active avatar.  This is the coarse-grained path and requires
   no additional control messages.

2. **`SceneStreamEnable`**:  Provides region-scoped and
   event-type-scoped filtering.  This is the fine-grained path.

### Subscription Union

If the client has an active `ChatStreamEnable` subscription that
includes `"scene"` AND an active `SceneStreamEnable` subscription,
the server MUST deliver the union of both subscriptions.  The
server MUST NOT deliver duplicate events; if an event matches both
subscriptions, it is delivered exactly once.

### Subscription Independence

`SceneStreamDisable` cancels only the Scene-specific subscription
established by `SceneStreamEnable`.  It does not affect Scene events
delivered through `ChatStreamEnable`.

`ChatStreamDisable` cancels only the Chat ephemeral subscription.
If `"scene"` was included in the `ChatStreamEnable` `dataTypes`,
Scene events delivered through that path stop; events delivered
through an active `SceneStreamEnable` subscription are unaffected.

## Interoperability with JMAP VTC WebSocket {#vtc-wss-interop}

Scene and VTC ephemeral subscriptions are fully independent.
`SceneStreamEnable` / `SceneStreamDisable` does not affect VTC
events, and `VTCStreamEnable` / `VTCStreamDisable` ({{JMAP-VTC-WSS}})
does not affect Scene events.

When a SceneRegion has an `activeCallId` binding to a VTCCall, a
client typically subscribes to both:

~~~json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
~~~

~~~json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
  "eventTypes": null
}
~~~

Both subscriptions deliver events on the same WebSocket connection.
The client distinguishes them by the `@type` field on each frame.

## Recommendation

Clients displaying a single region (e.g., a 3D viewport) SHOULD
prefer `SceneStreamEnable` with an explicit `regionIds` list.  This
minimizes server fan-out and network traffic.

Clients that need Scene events alongside Chat typing, presence, and
VTC events and do not require region-scoped filtering MAY use
`ChatStreamEnable` with
`dataTypes: ["typing", "presence", "vtc", "scene"]` as a single
subscription for all ephemeral event types across all capabilities.

# Forward Compatibility {#forward-compat}

A client receiving a server-to-client message with an unrecognized
`@type` value SHOULD silently ignore the message and SHOULD NOT
close the WebSocket connection.  This preserves forward
compatibility as new ephemeral event types are introduced.

A server receiving a client-to-server message with an unrecognized
`@type` value SHOULD silently ignore the message and SHOULD NOT
close the WebSocket connection.

# Complete Example {#example}

The following example illustrates a WebSocket session where a client
enters a region, subscribes to Scene events, receives several
events, and then leaves.

## WebSocket Handshake

~~~
GET /jmap/ws/ HTTP/1.1
Host: scene.example.com
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer eyJ0eXAi...
Sec-WebSocket-Protocol: jmap
~~~

## Enable State-Change Push

Client enables push for SceneObject and SceneAvatar state changes:

~~~json
{
  "@type": "WebSocketPushEnable",
  "dataTypes": ["SceneRegion", "SceneObject", "SceneAvatar"]
}
~~~

## Enter Region

Client enters a region via standard JMAP method call:

~~~json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "create": {
    "av0": {
      "regionId": "01J5ABC0000000000000000001"
    }
  }
}, "0"]]
~~~

## Enable Scene Event Stream

Client subscribes to events for the region it just entered:

~~~json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
~~~

## Another Avatar Enters

Server delivers an avatar-enter event:

~~~json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "entered",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
~~~

## Object Placed

Bob places an object; server delivers an object-create event:

~~~json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000042",
  "event": "created",
  "updatedBy": "user:bob@example.com",
  "position": [8.0, 1.2, -3.5]
}
~~~

The client may optionally fetch the full object:

~~~json
[["SceneObject/get", {
  "accountId": "account-xyz",
  "ids": ["01J5OBJ0000000000000000042"]
}, "1"]]
~~~

## User Interaction

Alice clicks an interactable object:

~~~json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000001",
  "userId": "user:alice@example.com",
  "action": "click",
  "data": null
}
~~~

## State Change (Interleaved)

A JMAP state-change notification arrives on the same connection:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "account-xyz": {
      "SceneObject": "s301"
    }
  }
}
~~~

## Bob Leaves

Server delivers an avatar-leave event:

~~~json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "left",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
~~~

## Disable Scene Stream

Client leaves the region and cancels its Scene subscription:

~~~json
{
  "@type": "SceneStreamDisable"
}
~~~

## WebSocket Close

~~~
Client sends Close frame (1000, "leaving")
Server responds with Close frame (1000)
~~~

# Security Considerations {#security}

## Inherited Security Requirements

All security considerations for WebSocket ({{RFC6455}} Section 10),
JMAP ({{RFC8620}} Section 8), and JMAP Scene ({{JMAP-SCENE}}
Security Considerations) apply without modification.

The TLS requirements of {{RFC8887}} Section 5.1 apply: the WebSocket
connection MUST use TLS 1.2 or later, following the recommendations
in BCP 195 {{?RFC9325}}.  Servers SHOULD support TLS 1.3 or later.

## Event Authorization

Servers MUST verify authorization at the time of each ephemeral
event delivery, not only at the time `SceneStreamEnable` is
received, since region access and avatar presence can change while
the connection is open:

- All Scene ephemeral events MUST only be delivered for regions in
  which the authenticated user has an active avatar at delivery
  time.

- `SceneObjectEvent` MUST respect the visibility contract defined
  in {{JMAP-SCENE}}: if the server would not return this object in
  a `SceneObject/get` response to this user, it MUST NOT deliver
  an ephemeral event for it.

- `SceneInteractionEvent` for objects the user cannot see (per the
  visibility contract) MUST NOT be delivered.

## Rate Limiting

The rate limits defined in {{rate-limiting}} SHOULD be enforced
server-side.  Without these limits, a high-frequency object-update
or interaction stream could exhaust server fan-out resources or
flood client connections.

## Interaction Event Data Exposure

`SceneInteractionEvent` carries an opaque `data` field.  The server
relays this without interpretation.  Deployments SHOULD validate
that `data` payloads do not exceed a deployment-defined size limit
to prevent abuse.  Clients MUST NOT trust `data` contents for
security-critical decisions.

## Spatial Presence Exposure

`SceneAvatarEvent` reveals when users enter and leave regions.  This
is intentional (it enables social presence features) but
privacy-sensitive.  Deployments that require avatar anonymity SHOULD
consider suppressing `displayName` or using pseudonymous identifiers
in regions where anonymity is desired.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP
Capabilities" registry defined in {{RFC8620}} Section 9.3:

- **Capability Name:** `urn:ietf:params:jmap:scene:websocket`
- **Intended Use:** common
- **Change Controller:** IETF
- **Specification document:** This document.
- **Security and Privacy Considerations:** See {{security}} of this
  document.

--- back

# Acknowledgements

The design of this specification follows the patterns established by
{{JMAP-CHAT-WSS}} for JMAP Chat ephemeral events and
{{JMAP-VTC-WSS}} for JMAP VTC ephemeral events.
