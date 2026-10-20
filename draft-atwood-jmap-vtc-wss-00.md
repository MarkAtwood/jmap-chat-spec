---
title: JMAP VTC over WebSocket
abbrev: JMAP VTC WebSocket
docname: draft-atwood-jmap-vtc-wss-00
category: std
updates: 8887
stream: IETF

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
  JMAP-VTC:
    title: JMAP for Video/Voice Teleconferencing
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
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

--- abstract

This document defines a WebSocket binding for the ephemeral in-call
events specified by JMAP for Video/Voice Teleconferencing
({{JMAP-VTC}}).  It registers the
`urn:ietf:params:jmap:vtc:websocket` capability and introduces
`VTCStreamEnable` and `VTCStreamDisable` control messages that give
clients call-scoped, event-type-scoped subscription over a JMAP
WebSocket connection ({{RFC8887}}).

--- middle

# Introduction

{{JMAP-VTC}} defines eight ephemeral event types for real-time
call state: `VTCRingEvent`, `VTCCallEndEvent`, `VTCGatewaySignal`,
`VTCParticipantEvent`, `VTCMediaStateEvent`, `VTCActiveSpeakerEvent`,
`VTCUnmuteRequestEvent`, and `VTCRecordingStateEvent`.  These events
are fire-and-forget signals — not persisted as JMAP objects — that
drive in-call user interfaces.

When the server advertises both `urn:ietf:params:jmap:chat:websocket`
({{JMAP-CHAT-WSS}}) and the capability defined here, clients can
subscribe to VTC events by including `"vtc"` in the `dataTypes`
array of a `ChatStreamEnable` message.  However, a deployment can offer JMAP
VTC without JMAP Chat.  This document provides a standalone
subscription mechanism for that case and a finer-grained,
call-scoped alternative for deployments that have both.

## Update to RFC 8887 {#rfc8887-update}

{{RFC8887}} Section 4.2 states that "other message types MUST NOT
be transmitted over this connection," and Section 4.3 limits the
message types each side may send.  This document updates
{{RFC8887}} to relax both restrictions as follows:

- A JMAP capability advertised in the Session object ({{RFC8620}}
  Section 2) MAY define additional WebSocket message types, each
  identified by its `@type` value.

- A server MUST NOT send an additional server-to-client message
  type unless the client has enabled it using a control message
  defined by the same capability.

This is the same rule stated in {{JMAP-CHAT-WSS}}; it is restated
here so that this document stands alone.  Under it, the
`urn:ietf:params:jmap:vtc:websocket` capability defines
`VTCStreamEnable`, `VTCStreamDisable`, and the WebSocket delivery of
the event types listed in {{event-types}}.  Except as updated here,
all {{RFC8887}} requirements apply unchanged.

# Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Capability {#capability}

## urn:ietf:params:jmap:vtc:websocket

Servers supporting this specification MUST add a property named
`urn:ietf:params:jmap:vtc:websocket` to the JMAP Session object
capabilities.  The value of this property is an empty JSON object
`{}`.

Presence of this capability indicates that the server supports
ephemeral VTC event push: it delivers the in-call events defined
by {{JMAP-VTC}} over the WebSocket connection and accepts
`VTCStreamEnable` and `VTCStreamDisable` control messages from
the client.

This capability is distinct from `urn:ietf:params:jmap:websocket`
({{RFC8887}}), which advertises the WebSocket transport itself, and
from `urn:ietf:params:jmap:vtc` ({{JMAP-VTC}}), which advertises
the VTC data model and methods.

### Session Object Example

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:websocket": {
      "url": "wss://vtc.example.com/jmap/ws/",
      "supportsPush": true
    },
    "urn:ietf:params:jmap:vtc": {},
    "urn:ietf:params:jmap:vtc:websocket": {}
  }
}
~~~

# VTC Stream Control Messages {#stream-control}

## VTCStreamEnable {#vtc-stream-enable}

A client sends a `VTCStreamEnable` message to start or replace its
ephemeral VTC event subscription on the current WebSocket connection.

### Properties

- **`@type`** (String): MUST be `"VTCStreamEnable"`.

- **`callIds`** (String[]|null): An explicit list of VTCCall ids
  for which events are requested, or `null` to receive events for
  all calls in which the authenticated user is a current participant.
  Default: `null`.

- **`eventTypes`** (String[]|null): An explicit list of event type
  names the client wishes to receive, or `null` to receive all
  event types.  Recognized values are: `"VTCRingEvent"`,
  `"VTCCallEndEvent"`, `"VTCGatewaySignal"`,
  `"VTCParticipantEvent"`, `"VTCMediaStateEvent"`,
  `"VTCActiveSpeakerEvent"`, `"VTCUnmuteRequestEvent"`, and
  `"VTCRecordingStateEvent"`.  Default: `null`.
  `VTCRecordingStateEvent` is the recording consent signal
  ({{JMAP-VTC}}), so the server delivers it for every call the
  subscription covers whether or not `eventTypes` lists it.

- **`id`** (String, optional): A client-specified identifier for
  this message.  A `RequestError` caused by this message carries
  this value as its `requestId` ({{RFC8887}} Section 4.3.4).

### Example

~~~json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB0"],
  "eventTypes": ["VTCParticipantEvent", "VTCMediaStateEvent",
                  "VTCActiveSpeakerEvent"]
}
~~~

### Semantics

A subsequent `VTCStreamEnable` message replaces the previous
ephemeral subscription in its entirety.  There is no partial-update
mechanism; the client re-sends the full desired subscription state.

In this document, a user is a current participant of a call when
the user's VTCParticipant record in that call has a non-null
`joinedAt` and a null `leftAt` ({{JMAP-VTC}}).  A callee who has been
rung but has not joined is not a current participant, and neither
is a participant waiting in the lobby, whose `joinedAt` stays
`null` until a moderator admits them.

If a `callIds` entry refers to a call in which the authenticated
user is not a current participant, that entry produces no events
but is retained in the subscription record.  If the user later
joins (or rejoins) that call, per-call events begin without a new
`VTCStreamEnable`, so a client may subscribe before joining without
missing the first in-call events.  If no entry refers to a call in
which the user is a current participant (or `callIds` is an empty
array), no per-call events are delivered; per-user events are
unaffected ({{event-types}}).

VTCCall ids are unique only within an account.  An id listed in
`callIds` matches that id in every account the authenticated user
can access; clients use the `accountId` of each event to tell the
accounts apart.

The server does not acknowledge `VTCStreamEnable`.  As with
`WebSocketPushEnable` ({{RFC8887}}), success is silent and failure
produces a `RequestError`.  A client that sends several control
messages in succession SHOULD give each an `id` so that it can
attribute any `RequestError` to the message that caused it.

Unrecognized values in `eventTypes` that appear alongside recognized
values MUST be silently ignored; only the recognized values take
effect.  If `eventTypes` is an empty array or contains only
unrecognized values, the server MUST respond with a `RequestError`
frame ({{RFC8887}} Section 4.3.4) with a `type` of
`"urn:ietf:params:jmap:error:invalidArguments"` and MUST NOT update
the current ephemeral subscription state.

### Subscription Scope

When `callIds` is `null`, the subscription is dynamic: as the user
joins new calls or leaves existing calls, the set of calls producing
events adjusts automatically.

When `callIds` is an explicit list, the subscription is static: only
events from the listed calls are delivered.  If the user leaves a
listed call, events for that call stop.  If the user later rejoins
the same call, events resume without a new `VTCStreamEnable`.

## VTCStreamDisable {#vtc-stream-disable}

A client sends a `VTCStreamDisable` message to cancel all ephemeral
VTC event delivery on the current connection.

### Properties

- **`@type`** (String): MUST be `"VTCStreamDisable"`.

### Example

~~~json
{
  "@type": "VTCStreamDisable"
}
~~~

### Semantics

The server MUST stop delivering VTC ephemeral events after processing
this message.  The server MUST silently succeed if no ephemeral VTC
subscription is currently active.  The WebSocket connection and any
active `WebSocketPushEnable` state-change subscription remain
unaffected.

# Event Delivery Semantics {#event-delivery}

## General Principles

VTC ephemeral events do not correspond to persistent state changes
and carry no state token.  They are delivered as server-to-client
text frames over the WebSocket connection when an ephemeral VTC
subscription is active.

Ephemeral event frames are interleaved with `StateChange` and
`Response` frames on the same WebSocket connection and SHOULD be
processed independently.  Receiving a VTC ephemeral event SHOULD NOT
cause the client to update any locally cached state token or issue
any JMAP method call.

## Event Types {#event-types}

This specification governs the WebSocket delivery of all eight
ephemeral event types defined by {{JMAP-VTC}}:

| Event Type              | Scope        | Description                          |
|-------------------------|--------------|--------------------------------------|
| `VTCRingEvent`          | per-user     | Incoming ring-call notification      |
| `VTCCallEndEvent`       | per-user     | Ring stopped or call ended           |
| `VTCGatewaySignal`      | per-call     | Opaque gateway protocol pass-through |
| `VTCParticipantEvent`   | per-call     | Participant joined, left, or kicked  |
| `VTCMediaStateEvent`    | per-call     | Participant media state change       |
| `VTCActiveSpeakerEvent` | per-call     | Active speaker changed               |
| `VTCUnmuteRequestEvent` | per-call     | Moderator requests participant unmute|
| `VTCRecordingStateEvent`| per-call     | Recording started, paused, or stopped|

When delivered over the WebSocket, every event in this table
carries an `accountId` (String) property in addition to the
properties defined in {{JMAP-VTC}}: the id of the call's hosting
account, the account that contains the VTCCall and its
VTCParticipant records ({{JMAP-VTC}}).  This is usually not the
recipient's personal account.  This applies whether the event is delivered
through `VTCStreamEnable` or through `ChatStreamEnable`
({{chat-wss-interop}}).

Per-user events (`VTCRingEvent`, `VTCCallEndEvent`) are delivered
based on the authenticated user's identity and are not filtered by
`callIds`.  While a VTC subscription is active, a per-user event is
delivered if `eventTypes` is `null` or includes that event type,
including when `callIds` is an empty array or names only calls in
which the user is not a current participant.

Per-call events are delivered only for calls that match the active
subscription's `callIds` filter (or all calls if `callIds` is
`null`) and for which the user is a current participant.

## Rate Limiting {#rate-limiting}

Servers SHOULD enforce the following rate limits to prevent resource
exhaustion:

- **`VTCMediaStateEvent`**: no more than one event per participant
  per call per second.  Events above this rate MAY be silently
  dropped; the most recent state SHOULD be delivered when the rate
  window reopens.

- **`VTCActiveSpeakerEvent`**: no more than two events per call per
  second.  Events above this rate MAY be silently dropped.

- **`VTCRingEvent`**: subject to the per-caller and per-callee ring
  rate limits defined in {{JMAP-VTC}} Security Considerations.

Servers MAY apply additional rate limits to other event types.

## Event Ordering

Events are delivered in best-effort order.  The server SHOULD
deliver events in the order they were generated, but ordering is
not guaranteed.  Each event is self-contained; clients
SHOULD use the event's `callId` and `participantId` fields to
correlate events with local state.

## Blocked-Sender Suppression {#blocked-sender}

The server SHOULD NOT deliver a `VTCRingEvent` or `VTCCallEndEvent`
(with `endReason` of `"cancelled"` or `"timeout"`) if the
`initiatorId` corresponds to a contact whose `blocked` field is
`true` on the recipient's contact list (when {{JMAP-CHAT}} is
present) or an equivalent deployment-defined block list.  The
initiator is not informed.

This parallels the blocked-sender suppression rule for
`ChatTypingEvent` in {{JMAP-CHAT-WSS}} and the ring push
suppression rule in {{JMAP-VTC}}.

### In-Call Event Suppression

The server SHOULD extend blocked-sender suppression to in-call
events.  If a per-call event originates from a participant whose
corresponding contact has `blocked` set to `true` on the
recipient's contact list (or an equivalent deployment-defined block
list), the server SHOULD silently drop the event for that recipient.
The blocked participant is not informed of the suppression.

The following event types are subject to this rule:

- `VTCParticipantEvent`: join, leave, or kick notifications for a
  blocked participant.

- `VTCMediaStateEvent`: media-state updates originating from a
  blocked participant.

- `VTCActiveSpeakerEvent`: active-speaker indications naming a
  blocked participant.

- `VTCUnmuteRequestEvent`: unmute requests sent by a blocked
  participant acting as moderator.

This rule does not apply to `VTCRecordingStateEvent`, the consent
signal that {{JMAP-VTC}} delivers to all participants regardless
of block status.  It also does not apply to
`VTCGatewaySignal`, which is already restricted to moderators per
{{security}}.

## Subscription Lifecycle

The ephemeral VTC subscription is bound to the WebSocket connection.
When the connection closes — whether by graceful close handshake,
network failure, or server-initiated close — the subscription is
immediately cancelled.  No events are buffered for later delivery.

A reconnecting client sends a new `VTCStreamEnable` to
re-establish its subscription.

# Interoperability with JMAP Chat WebSocket {#chat-wss-interop}

When a server advertises both `urn:ietf:params:jmap:chat:websocket`
({{JMAP-CHAT-WSS}}) and `urn:ietf:params:jmap:vtc:websocket`
(this document), two subscription paths exist for VTC ephemeral
events:

1. **`ChatStreamEnable` with `"vtc"` in `dataTypes`**:  Subscribes
   to all VTC event types: per-user events for the user, and
   per-call events for all calls in which the user is a current
   participant ({{event-types}}).  This is the coarse-grained path
   and requires no additional control messages.

2. **`VTCStreamEnable`**:  Provides call-scoped and event-type-scoped
   filtering.  This is the fine-grained path.

## Subscription Union

If the client has an active `ChatStreamEnable` subscription that
includes `"vtc"` AND an active `VTCStreamEnable` subscription, the
server MUST deliver the union of both subscriptions.  The server
MUST NOT deliver duplicate events; if an event matches both
subscriptions, it is delivered exactly once.

## Subscription Independence

`VTCStreamDisable` cancels only the VTC-specific subscription
established by `VTCStreamEnable`.  It does not affect VTC events
delivered through `ChatStreamEnable`.

`ChatStreamDisable` cancels only the Chat ephemeral subscription.
If `"vtc"` was included in the `ChatStreamEnable` `dataTypes`, VTC
events delivered through that path stop; events delivered through an
active `VTCStreamEnable` subscription are unaffected.

## Recommendation

Clients that need events from a single call (e.g., the call
currently displayed in the UI) SHOULD prefer `VTCStreamEnable` with
an explicit `callIds` list.  This minimizes server fan-out and
network traffic in multi-call scenarios.

Clients that need VTC events alongside Chat typing and presence
events and do not require call-scoped filtering MAY use
`ChatStreamEnable` with `dataTypes: ["typing", "presence", "vtc"]`
as a single subscription for all ephemeral event types.

# Forward Compatibility {#forward-compat}

A client receiving a server-to-client message with an unrecognized
`@type` value SHOULD silently ignore the message and SHOULD NOT
close the WebSocket connection.  This preserves forward
compatibility as new ephemeral event types are introduced.

A client-to-server message with an unrecognized `@type` value is a
request-level error under {{RFC8887}} Section 4.3.1, which requires
the server to return a `RequestError`;
`"urn:ietf:params:jmap:error:notRequest"` ({{RFC8620}} Section
3.6.1) is the appropriate `type`.  The server SHOULD NOT close the
WebSocket connection.

# Complete Example {#complete-example}

The following example illustrates a WebSocket session where a client
joins a call, subscribes to VTC events, receives several events, and
then leaves.

## WebSocket Handshake

~~~
GET /jmap/ws/ HTTP/1.1
Host: vtc.example.com
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer eyJ0eXAi...
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: jmap
Sec-WebSocket-Version: 13
~~~

## Enable State-Change Push

Client enables push for VTCCall and VTCParticipant state changes:

~~~json
{
  "@type": "WebSocketPushEnable",
  "dataTypes": ["VTCCall", "VTCParticipant"]
}
~~~

## Enable VTC Event Stream

Client subscribes to events for a specific call:

~~~json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB0"],
  "eventTypes": null
}
~~~

## Participant Joins

Server delivers a participant-join event:

~~~json
{
  "@type": "VTCParticipantEvent",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "participantId": "01J4XMVQK7NRYS9PPCEHTK4CD0",
  "event": "joined",
  "displayName": "Alice Chen",
  "role": "participant"
}
~~~

## Media State Update

A participant mutes their microphone:

~~~json
{
  "@type": "VTCMediaStateEvent",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "participantId": "01J4XMVQK7NRYS9PPCEHTK4CD0",
  "mediaState": {
    "audio": false,
    "video": true,
    "screen": false,
    "raisedHand": false
  }
}
~~~

## Active Speaker Change

The active speaker changes:

~~~json
{
  "@type": "VTCActiveSpeakerEvent",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "participantId": "01J4XNZRM8QWVT0RRDFJVS5EF0"
}
~~~

## Recording Starts

A moderator starts recording:

~~~json
{
  "@type": "VTCRecordingStateEvent",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "recordingId": "01J4XP2SN9RXVV1SSEGJVT6GH0",
  "state": "recording",
  "initiatedBy": "user:bob@example.com"
}
~~~

## State Change (Interleaved)

A JMAP state-change notification arrives on the same connection:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "bob-account": {
      "VTCParticipant": "t101"
    }
  }
}
~~~

## Disable VTC Stream

Client leaves the call and cancels its VTC subscription:

~~~json
{
  "@type": "VTCStreamDisable"
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
JMAP ({{RFC8620}} Section 8), and JMAP VTC ({{JMAP-VTC}} Security
Considerations) apply without modification.

The TLS requirements of {{RFC8887}} Section 5.1 apply.  BCP 195
{{?RFC9325}} gives current TLS recommendations.

## Event Authorization

Servers SHOULD verify authorization at the time of each ephemeral
event delivery, not only at the time `VTCStreamEnable` is received,
since participation, permissions, and block status can change while
the connection is open:

- Per-call events (`VTCParticipantEvent`, `VTCMediaStateEvent`,
  `VTCActiveSpeakerEvent`, `VTCUnmuteRequestEvent`,
  `VTCRecordingStateEvent`, `VTCGatewaySignal`) go only to current
  participants of the call at delivery time ({{event-types}}).

- `VTCRingEvent`, `VTCCallEndEvent`, and the in-call events listed
  in {{blocked-sender}} are suppressed when the originating user
  corresponds to a blocked contact at delivery time, as specified
  there.

- `VTCGatewaySignal` SHOULD only be delivered to participants with
  `moderator` role unless the deployment explicitly permits broader
  delivery.

## Credential Expiry

The credential-expiry rules of {{RFC8887}} Section 4.2 apply.  If
the server closes the connection when credentials expire, it SHOULD
use Close frame status code 1008 (Policy Violation).  Ephemeral
events are pushed, not requested, so treating subsequent requests
as unauthenticated does not by itself stop them: a server that does
not close the connection SHOULD NOT deliver further VTC ephemeral
events on that connection.

## Rate Limiting {#security-rate-limiting}

The rate limits defined in {{rate-limiting}} are enforced
server-side.  Without these limits, a high-frequency media-state or
active-speaker update stream could exhaust server fan-out resources
or flood client connections.

## Event Data Exposure

VTC ephemeral events reveal real-time call participation and media
state.  Servers SHOULD ensure that events for one user's
subscription are never delivered to another user's WebSocket
connection.  Each WebSocket connection has exactly one authenticated
identity.

`VTCMediaStateEvent` is client-reported and server-relayed.  Clients
SHOULD NOT rely on `VTCMediaStateEvent` for security-critical
decisions, as documented in {{JMAP-VTC}} Security Considerations.

## End-to-End Encrypted Calls {#e2ee}

Media end-to-end encryption (`e2eeEnabled: true` in {{JMAP-VTC}})
does not hide the events defined here from the server.  The server
generates or relays every `VTCParticipantEvent`, so it knows who is
in the call and when they join and leave.  `VTCActiveSpeakerEvent`
requires the server to map media streams to participants, which
reveals who speaks, when, and for how long.

For calls with `e2eeEnabled: true`, the server SHOULD NOT map media
streams to participant ids and SHOULD NOT deliver
`VTCActiveSpeakerEvent`, as specified in the End-to-End Encrypted
Calls section of {{JMAP-VTC}}.  Clients determine the active speaker
locally from decrypted audio levels instead.  The rate limit in
{{rate-limiting}} applies only when the server delivers the event.

A disclosed key-holding recorder, livestream bridge, or gateway in
an E2EE call appears in `VTCParticipantEvent` like any other
participant, and a recorder or bridge is announced with
`VTCRecordingStateEvent`.  The blocked-sender exemption for
`VTCRecordingStateEvent` ({{blocked-sender}}) applies to these
announcements.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP
Capabilities" registry defined in {{RFC8620}} Section 9.4:

- **Capability Name:** `urn:ietf:params:jmap:vtc:websocket`
- **Intended Use:** common
- **Change Controller:** IETF
- **Specification document:** This document.
- **Security and Privacy Considerations:** See {{security}} of this
  document.

--- back

# Acknowledgements

The design of this specification follows the patterns established by
{{JMAP-CHAT-WSS}} for JMAP Chat ephemeral events.
