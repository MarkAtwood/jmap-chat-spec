---
title: JMAP Chat over WebSocket
abbrev: JMAP Chat WebSocket
docname: draft-atwood-jmap-chat-wss-00
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
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026

informative:
  RFC9325:
  JMAP-VTC:
    title: JMAP for Video/Voice Teleconferencing
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  JMAP-VTC-WSS:
    title: JMAP VTC over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-wss-00
    date: 2026
  JMAP-SCENE:
    title: JMAP Scene
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026
  JMAP-SCENE-WSS:
    title: JMAP Scene over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-wss-00
    date: 2026

--- abstract

This document defines how JMAP Chat ({{JMAP-CHAT}}) operates over the WebSocket transport binding defined in {{RFC8887}}. It specifies the `urn:ietf:params:jmap:chat:websocket` capability, the interaction between JMAP Chat's authentication model and the WebSocket handshake, and the server-to-client message types for delivering ephemeral chat events — typing indicators and presence updates — over a WebSocket connection.

--- middle

# Introduction

{{RFC8887}} defines a WebSocket subprotocol for JMAP that provides bidirectional, low-latency transport for JMAP API requests and push notifications. {{JMAP-CHAT}} defines a push model based on the RFC 8620 EventSource (SSE) mechanism and, optionally, RFC 8620 push subscriptions. SSE provides a one-way server-to-client channel; clients must issue separate HTTP requests to send JMAP method calls.

This document extends that model in two ways. First, it makes explicit that all JMAP Chat API methods operate without modification over the RFC 8887 WebSocket transport. Second, it defines a subscription mechanism and two new server-to-client message types — `ChatTypingEvent` and `ChatPresenceEvent` — that carry ephemeral chat events over the WebSocket connection alongside the RFC 8887 `StateChange` push notifications.

No new JMAP methods are defined. The protocol changes are limited to: a new capability advertisement, two new client-to-server control messages for managing ephemeral subscriptions, and two new server-to-client event message types.

## Motivation

JMAP Chat has two categories of real-time server-push events with different characteristics:

**State-change events** (new messages, chat updates, contact changes):
Carried as `StateChange` objects ({{RFC8620}} Section 7.1). These reference state tokens and allow the client to issue `/changes` calls to retrieve the updated objects. {{RFC8887}} handles these without modification via `WebSocketPushEnable`.

**Ephemeral events** (typing indicators, real-time presence updates):
These do not correspond to persistent state changes and carry no state token. A typing indicator is useful for seconds; a presence transition is useful for tens of seconds. Routing either through the `StateChange`-then-`/changes`-then-`/get` round-trip would impose connection setup, authentication, and method-call latency on signals that are valuable only if delivered immediately and silently discarded otherwise. There is nothing to persist, nothing to sync on reconnection, and no state to catch up on after a disconnect — missed typing events are simply irrelevant by the time the client reconnects. {{RFC8887}}'s push mechanism addresses only the state-change category; this document fills the gap for the ephemeral category.

## Relationship to RFC 8887

This document is strictly additive to {{RFC8887}}. Implementations MUST support {{RFC8887}} as a prerequisite. All RFC 8887 requirements for authentication, handshake, message framing, out-of-order request handling, push notification enabling and disabling, and error handling apply unchanged.

{{RFC8887}} Section 4.2 states that "other message types MUST NOT be transmitted over this connection," enumerating exactly the message types it defines. This document defines additional message types ({{chat-stream-enable}}, {{chat-stream-disable}}, {{chat-typing-event}}, {{chat-presence-event}}) that are transmitted over the same WebSocket connection. Advertising the `urn:ietf:params:jmap:chat:websocket` capability in the JMAP Session object ({{RFC8620}} Section 2) constitutes the negotiation event that permits these additional message types. A client that has read the Session object and observed this capability has pre-agreed to their use; the RFC 8887 Section 4.2 restriction applies only within the scope of the base RFC 8887 protocol and does not constrain extensions that are explicitly negotiated via the Session capabilities mechanism.

## Relationship to JMAP Chat

This document is a companion to {{JMAP-CHAT}} and is intended to be read alongside it. The data types, methods, push event payloads, and security model defined in {{JMAP-CHAT}} are used throughout without re-definition.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{RFC8887}}, and {{JMAP-CHAT}} is used throughout.

# Capability {#capability}

Servers supporting this specification MUST add a property named `urn:ietf:params:jmap:chat:websocket` to the JMAP Session object capabilities. The value of this property is an empty JSON object `{}`. Presence of this capability indicates that the server supports ephemeral push: it delivers `ChatTypingEvent` and `ChatPresenceEvent` messages over the WebSocket connection and accepts `ChatStreamEnable` and `ChatStreamDisable` control messages from the client. Servers that do not support ephemeral push MUST NOT advertise this capability.

This capability is distinct from `urn:ietf:params:jmap:websocket` ({{RFC8887}}), which advertises the WebSocket transport itself. A server supporting both this specification and {{RFC8887}} will advertise both capabilities in the Session object.

Example Session object fragment:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:websocket": {
      "url": "wss://chat.example.com/jmap/ws/",
      "supportsPush": true
    },
    "urn:ietf:params:jmap:chat:websocket": {}
  }
}
~~~

# Authentication and Identity {#authentication}

{{JMAP-CHAT}} requires the transport layer to supply a stable, opaque user identity string for each connection. When JMAP Chat is used over a WebSocket connection, this identity is established at the WebSocket handshake per {{RFC8887}} Section 4.2. The identity string supplied by the server's authentication layer for the HTTP request that initiates the handshake becomes the `ownerUserId` for the duration of the WebSocket connection.

Servers MUST NOT accept JMAP Chat requests over a WebSocket connection that was not successfully authenticated. If the authentication credentials expire during a WebSocket session, the server MUST either treat subsequent requests as unauthenticated or close the WebSocket connection with a Close frame status code of 1008 (Policy Violation). This document intentionally strengthens the "MAY" of {{RFC8887}} Section 4.2 to "MUST": servers MUST take one of these two actions and MUST use status code 1008 if closing, so that clients can reliably detect and recover from credential expiry on a long-lived chat connection.

Because a single authenticated WebSocket connection authorizes a continuous stream of JMAP Chat operations, credential theft has broader impact than in the per-request HTTP model. Implementations SHOULD use short-lived credentials (e.g., OAuth 2.0 Bearer tokens with short expiry) and SHOULD support credential refresh by closing and re-establishing the connection with fresh credentials.

# JMAP Chat API over WebSocket {#api}

All JMAP Chat owner-facing methods defined in {{JMAP-CHAT}} — including all `/get`, `/set`, `/query`, `/changes`, `/queryChanges`, `Chat/typing`, and `Space/join` methods — operate over the {{RFC8887}} WebSocket transport without modification. Clients include `"urn:ietf:params:jmap:chat"` in the `using` array of each JMAP Request object.

The `maxConcurrentRequests` limit in the Session capabilities applies to requests made on the WebSocket connection. When using the WebSocket subprotocol over a binding of HTTP that supports request multiplexing (e.g., HTTP/2), this limit applies to the sum of requests made on both the JMAP API endpoint and the WebSocket connection, per {{RFC8887}} Section 4.3.2.

Blob upload and download use separate HTTP connections per {{RFC8620}} Section 6. The WebSocket connection does not carry binary blob data.

# Push Notifications for Chat State Changes {#push-state}

State-change push notifications for JMAP Chat data types work without modification using the {{RFC8887}} push mechanism. When push is enabled via `WebSocketPushEnable`, clients MAY include any of the following JMAP Chat data type names in the `dataTypes` filter:

- `"Message"` — new or updated Message objects
- `"Chat"` — new or updated Chat objects
- `"ChatContact"` — new or updated ChatContact objects
- `"ReadPosition"` — read position changes
- `"PresenceStatus"` — changes to the owner's own PresenceStatus record
- `"Space"` — new or updated Space objects
- `"CustomEmoji"` — new or updated CustomEmoji objects
- `"SpaceBan"` — new or updated SpaceBan objects
- `"SpaceInvite"` — new or updated SpaceInvite objects

Upon receiving a `StateChange` for a Chat data type, clients SHOULD issue the corresponding `/changes` call to retrieve updated objects. On `cannotCalculateChanges`, clients MUST fall back to `/get`.

The `pushState` token mechanism defined in {{RFC8887}} Section 4.3.5.1 applies to Chat data types and enables clients to catch up on missed changes after reconnection without issuing a full `/changes` sweep.

# Ephemeral Chat Events {#ephemeral}

Ephemeral chat events — typing indicators and real-time presence updates from other contacts — do not correspond to persistent state changes and carry no state token. They are delivered as additional server-to-client text frames over the WebSocket connection when ephemeral push has been enabled by the client.

Ephemeral event frames are interleaved with `StateChange` and `Response` frames on the same WebSocket connection and SHOULD be processed independently. Receiving a `ChatTypingEvent` or `ChatPresenceEvent` SHOULD NOT cause the client to update any locally cached state token or issue any JMAP method call.

## Enabling Ephemeral Push {#chat-stream-enable}

A client enables ephemeral event delivery by sending a `ChatStreamEnable` object to the server.

`@type` (String):
: MUST be `"ChatStreamEnable"`.

`dataTypes` (String[]):
: A non-empty list of ephemeral event categories the client wishes to receive. This specification defines `"typing"` and `"presence"`; companion specifications MAY register additional values. Currently registered companion values: `"vtc"` for in-call events defined by {{JMAP-VTC-WSS}}, and `"scene"` for spatial events defined by {{JMAP-SCENE-WSS}}.

> **Implementation Note:** The `"vtc"` dataType in ChatStreamEnable is a convenience alias. When present, the Chat WSS server delegates VTC event delivery to the VTC WSS event pipeline. The Chat WSS implementation SHOULD treat this as an opaque delegation — it activates VTC event delivery for the connection without the Chat WSS layer needing to understand VTC event semantics. When `urn:ietf:params:jmap:vtc:websocket` is not present in the server's capabilities, the server MUST silently ignore `"vtc"` in `dataTypes` (per the general unrecognized-value rule).

`chatIds` (String[]|null):
: Applicable when `"typing"` is in `dataTypes`. An explicit list of Chat ids for which typing events are requested, or `null` to receive typing events for all Chats of which the owner is a current member. Ignored if `"typing"` is not in `dataTypes`.

`contactIds` (String[]|null):
: Applicable when `"presence"` is in `dataTypes`. An explicit list of ChatContact ids for which presence events are requested, or `null` to receive presence events for all known ChatContacts. Ignored if `"presence"` is not in `dataTypes`.

A subsequent `ChatStreamEnable` message replaces the previous ephemeral subscription in its entirety. There is no partial-update mechanism; the client re-sends the full desired subscription state.

If `dataTypes` is empty, or contains only unrecognized values (i.e., none of `"typing"` or `"presence"`), the server MUST respond with a `RequestError` frame ({{RFC8887}} Section 4.3.4) with a `type` of `"urn:ietf:params:jmap:error:invalidArguments"` and MUST NOT update the current ephemeral subscription state. Unrecognized values in `dataTypes` that appear alongside recognized values MUST be silently ignored; only the recognized values take effect.

> **Note:** When a companion capability (e.g., `urn:ietf:params:jmap:vtc:websocket`) is not present, its corresponding `dataTypes` value (e.g., `"vtc"`) is treated as unrecognized and silently ignored. Clients SHOULD check the server's capability list before relying on companion dataTypes, as no error feedback is provided for absent capabilities.

~~~json
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence"],
  "chatIds": null,
  "contactIds": null
}
~~~

## Disabling Ephemeral Push {#chat-stream-disable}

A client cancels all ephemeral event delivery by sending a `ChatStreamDisable` object:

`@type` (String):
: MUST be `"ChatStreamDisable"`.

The server MUST stop delivering ephemeral events after processing this message. The server MUST silently succeed if no ephemeral subscription is currently active. The WebSocket connection and any active `WebSocketPushEnable` state-change subscription remain unaffected.

~~~json
{
  "@type": "ChatStreamDisable"
}
~~~

## ChatTypingEvent {#chat-typing-event}

Delivered to the client when another participant begins or stops typing in a Chat for which the client has an active typing subscription.

`@type` (String):
: MUST be `"ChatTypingEvent"`.

`chatId` (String):
: The Chat id in which the typing event occurred.

`senderId` (String):
: The ChatContact.id of the typing user. MUST NOT be `"self"`.

`typing` (Boolean):
: `true` if the user is typing; `false` if typing has stopped.

~~~json
{
  "@type": "ChatTypingEvent",
  "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX",
  "senderId": "user:alice@example.com",
  "typing": true
}
~~~

Servers MUST NOT persist `ChatTypingEvent` messages. Servers SHOULD rate-limit outbound typing events per sender per chat: no more than one `ChatTypingEvent` per sender per chat per 3 seconds SHOULD be delivered; events above this rate MAY be silently dropped. This rate limit is consistent with the `Chat/typing` inbound rate limit defined in {{JMAP-CHAT}} and the `Peer/typing` rate limit defined in {{JMAP-CHAT-FED}}.

For direct and group Chats, servers MUST check each recipient's `receiveTypingIndicators` field (defined in {{JMAP-CHAT}} (see Chat/typing Server Behavior)) before delivery. If `receiveTypingIndicators` is `false`, the server MUST silently drop the `ChatTypingEvent` and MUST NOT deliver it to any of that owner's WebSocket connections. The sender is not notified. This check does not apply to channel Chats.

Servers MUST verify at delivery time that the owner remains a current member of the identified Chat. If the owner is no longer a member, the event MUST be dropped silently.

Servers MUST NOT deliver a `ChatTypingEvent` whose `senderId` corresponds to a ChatContact that is `blocked` on the recipient's ChatContact record. The sender is not notified. This parallels the message-suppression rule for blocked contacts in {{JMAP-CHAT}} Security Considerations and prevents a blocked user from leaking presence or attention patterns to a recipient who has explicitly chosen to ignore them.

## ChatPresenceEvent {#chat-presence-event}

Delivered to the client when a ChatContact's presence state changes and that contact is within the scope of the active `contactIds` subscription. `ChatPresenceEvent` delivers real-time presence snapshots; it is distinct from `PresenceStatus` state-change notifications ({{push-state}}), which track changes to the owner's own PresenceStatus record.

`@type` (String):
: MUST be `"ChatPresenceEvent"`.

`contactId` (String):
: The ChatContact.id whose presence state changed.

`presence` (String):
: The updated presence state: `"online"`, `"away"`, `"busy"`, `"invisible"`, or `"offline"`.

`lastActiveAt` (UTCDate, optional):
: Updated last-active timestamp, if known.

`statusText` (String|null, optional):
: Updated custom status text. `null` explicitly clears the field. If absent, the client SHOULD treat the existing value as unchanged.

`statusEmoji` (String|null, optional):
: Updated status emoji. `null` explicitly clears the field. If absent, the client SHOULD treat the existing value as unchanged.

~~~json
{
  "@type": "ChatPresenceEvent",
  "contactId": "user:alice@example.com",
  "presence": "away",
  "lastActiveAt": "2026-04-26T12:00:00Z",
  "statusText": "In a meeting",
  "statusEmoji": null
}
~~~

Servers SHOULD rate-limit outbound `ChatPresenceEvent` messages per contact: no more than one `ChatPresenceEvent` per contact per 30 seconds SHOULD be delivered; events above this rate MAY be silently dropped. This rate limit is consistent with the `Peer/subscribePresence` delivery rate defined in {{JMAP-CHAT-FED}}.

Servers MUST NOT deliver `ChatPresenceEvent` for a contact whose `blocked` field is `true` on the owner's ChatContact record.

# Handling Unknown Message Types {#unknown-types}

A client receiving a server-to-client message with an unrecognized `@type` value SHOULD silently ignore the message and SHOULD NOT close the WebSocket connection. This preserves forward compatibility as new ephemeral message types are introduced in future documents.

A server receiving a client-to-server message with an unrecognized `@type` value SHOULD silently ignore the message and SHOULD NOT close the WebSocket connection.

# Example {#example}

The following example shows a WebSocket JMAP Chat session. TLS negotiation and initial JMAP Session fetch are assumed complete. The client enables both state-change push and ephemeral push, sends a chat message, and receives a typing event followed by a state-change notification.

~~~
[[ From Client ]]                 [[ From Server ]]

GET /jmap/ws/ HTTP/1.1
Host: chat.example.com
Upgrade: websocket
Connection: Upgrade
Authorization: Bearer <token>
Origin: https://chat.example.com
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: jmap
Sec-WebSocket-Version: 13

                                  HTTP/1.1 101 Switching Protocols
                                  Upgrade: websocket
                                  Connection: Upgrade
                                  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
                                  Sec-WebSocket-Protocol: jmap

[WebSocket connection established]

WS_DATA
{
  "@type": "WebSocketPushEnable",
  "dataTypes": ["Message", "Chat",
                "ChatContact", "ReadPosition"],
  "pushState": "aaa"
}

WS_DATA
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence"],
  "chatIds": null,
  "contactIds": null
}

WS_DATA
{
  "@type": "Request",
  "id": "R1",
  "using": ["urn:ietf:params:jmap:core",
            "urn:ietf:params:jmap:chat"],
  "methodCalls": [["Message/set", {
    "accountId": "u1",
    "create": {
      "m1": {
        "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX",
        "body": "Hello!",
        "bodyType": "text/plain",
        "sentAt": "2026-04-26T12:00:00Z"
      }
    }
  }, "0"]]
}

                                  WS_DATA
                                  {
                                    "@type": "ChatTypingEvent",
                                    "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX",
                                    "senderId": "user:alice@example.com",
                                    "typing": true
                                  }

                                  WS_DATA
                                  {
                                    "@type": "Response",
                                    "requestId": "R1",
                                    "methodResponses": [
                                      ["Message/set", {
                                        "accountId": "u1",
                                        "created": {
                                          "m1": {
                                            "id": "01J3YKZQP..."
                                          }
                                        }
                                      }, "0"]
                                    ]
                                  }

                                  WS_DATA
                                  {
                                    "@type": "StateChange",
                                    "changed": {
                                      "u1": {
                                        "Message": "d35ecb040aab"
                                      }
                                    },
                                    "pushState": "bbb"
                                  }

WS_CLOSE

                                  WS_CLOSE

[WebSocket connection closed]
~~~

# Security Considerations {#security}

## Inherited Security Requirements

All security considerations for WebSocket ({{RFC6455}} Section 10), JMAP ({{RFC8620}} Section 8), and JMAP Chat ({{JMAP-CHAT}} Security Considerations) apply without modification.

The TLS requirements of {{RFC8887}} Section 5.1 apply: the WebSocket connection MUST use TLS 1.2 or later, following the recommendations in BCP 195 {{?RFC9325}}. Servers SHOULD support TLS 1.3 or later.

## Ephemeral Event Authorization

Servers MUST verify authorization at the time of each ephemeral event delivery, not only at the time `ChatStreamEnable` is received, since membership and block status can change while the connection is open:

- `ChatTypingEvent` MUST only be delivered for Chats in which the owner is a current member at delivery time.
- `ChatTypingEvent` MUST NOT be delivered to a recipient whose Chat record (for a direct or group Chat) has `receiveTypingIndicators: false` at delivery time.
- `ChatTypingEvent` MUST NOT be delivered when the sender (`senderId`) corresponds to a ChatContact whose `blocked` field is `true` on the recipient's ChatContact record at delivery time.
- `ChatPresenceEvent` MUST NOT be delivered for ChatContacts whose `blocked` field is `true` at delivery time.

## Rate Limiting

The per-sender, per-chat rate limit for `ChatTypingEvent` ({{chat-typing-event}}) and the per-contact rate limit for `ChatPresenceEvent` ({{chat-presence-event}}) SHOULD be enforced server-side. Without these limits, a malicious peer could send typing or presence updates at high frequency and exhaust server fan-out resources or flood client connections with ephemeral traffic.

## Ephemeral Event Scope Isolation

Servers MUST ensure that ephemeral events for one owner's subscription are never delivered to another owner's WebSocket connection. Each WebSocket connection has exactly one authenticated identity, and all ephemeral event filtering and delivery MUST be scoped to that identity.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:websocket`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.

--- back

# Acknowledgements

The author thanks Kenneth Murchison for {{RFC8887}} and the JMAP working group for {{RFC8620}}.
