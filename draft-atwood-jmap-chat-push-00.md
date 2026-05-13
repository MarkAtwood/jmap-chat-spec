---
title: JMAP Chat Push Notifications
abbrev: JMAP Chat Push
docname: draft-atwood-jmap-chat-push-00
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
  RFC8620:
  RFC8030:
  RFC8291:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026

informative:
  JMAP-EMAILPUSH:
    title: >
      JSON Meta Application Protocol (JMAP) Email Delivery Push
      Notifications
    author:
      fullname: Neil Jenkins
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-emailpush-02
    date: 2025
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  RFC9420:

--- abstract

This document defines JMAP Chat Push Notifications, a companion specification to JMAP Chat ({{JMAP-CHAT}}). It extends the RFC 8620 PushSubscription mechanism to deliver inline message content in push notification payloads, enabling mobile and background clients to display a notification without a follow-up server request. The extension defines a per-account push configuration, a notification payload object carrying message metadata and an optional body snippet, per-chat and per-kind filtering, mention-aware urgency, and behavior for end-to-end encrypted deployments where body content must not leave the server.

--- middle

# Introduction

{{JMAP-CHAT}} defines a push model based on the RFC 8620 PushSubscription mechanism. When a new message arrives, the server delivers a `StateChange` event carrying a state token for the `Message` data type. The client must then issue a `Message/changes` call to determine which messages are new, followed by a `Message/get` call to retrieve content. This round-trip is acceptable for desktop clients on persistent connections but is costly for mobile clients: each notification requires the device to wake, establish a connection, authenticate, and fetch before it can display any content.

This document follows the pattern established by {{JMAP-EMAILPUSH}} for JMAP Mail: it extends `PushSubscription` with a `chatPush` property that instructs the server to deliver a `ChatMessagePush` object — a standalone push payload carrying message metadata and an optional body snippet — directly to the push endpoint. A mobile client receiving this payload can render a complete notification immediately, without any follow-up request.

## Design Principles

- The extension is additive to the RFC 8620 `PushSubscription`. Servers that do not support this specification deliver unmodified `StateChange` payloads; clients are expected to handle those regardless.
- Per-account and per-chat filtering allows clients to limit push volume to the conversations that matter.
- Mention-aware urgency allows a single configuration to request high-urgency delivery for mentions and normal-urgency delivery for everything else.
- In end-to-end encrypted deployments, the server cannot access plaintext body content. The `encrypted` flag and metadata-only mode provide enough context to display a useful notification without exposing ciphertext.
- The push delivery infrastructure is entirely that of {{RFC8620}} and the underlying push channel {{RFC8030}}, unchanged.

## Relationship to JMAP Email Push

This specification is structurally parallel to {{JMAP-EMAILPUSH}} and is intended to coexist with it. The `chatPush` property defined here is distinct from the `emailPush` property defined in {{JMAP-EMAILPUSH}}; both MAY appear on the same `PushSubscription` object. Servers MAY implement either or both independently.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} and {{JMAP-CHAT}} is used throughout.

# Capability {#capability}

Servers supporting this specification advertise the `urn:ietf:params:jmap:chat:push` capability in the JMAP Session object. This capability is meaningful only when `urn:ietf:params:jmap:chat` is also present.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:push"]` at the session level is an empty object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:push"]` is a JSON object with the following fields:

`maxSnippetBytes` (UnsignedInt):
: Maximum byte length of the `bodySnippet` field in a `ChatMessageEntry`. Servers MUST truncate body snippets to this length before including them in the payload. Truncation MUST occur on a UTF-8 character boundary.

`supportedUrgencyValues` (String[]):
: The set of Web Push urgency values ({{RFC8030}} Section 5.3) the server supports. Valid values are `"very-low"`, `"low"`, `"normal"`, and `"high"`. Servers MUST support at least `"normal"` and `"high"`.

`maxMessagesPerPush` (UnsignedInt, optional):
: Maximum number of `ChatMessageEntry` objects the server will include in a single `ChatMessagePush` payload. If absent, the server does not advertise a bound. If present, the server MUST NOT include more than this many entries in the `messages` array of any `ChatMessagePush` it delivers.

# PushSubscription Extension {#push-subscription-extension}

This document extends the `PushSubscription` object defined in {{RFC8620}} Section 7.2 with one additional property:

`chatPush` (Id[ChatPushConfig], optional):
: A map of accountId to `ChatPushConfig` object. When present for an account, the server delivers `ChatMessagePush` payloads to the push endpoint for new inbound messages in that account that match the configuration. When absent or null for an account, the server delivers standard `StateChange` notifications for that account without inline message content.

Servers MUST NOT include `chatPush` entries for accounts whose `accountCapabilities` does not include `urn:ietf:params:jmap:chat:push`.

## ChatPushConfig {#chat-push-config}

A `ChatPushConfig` specifies which messages trigger inline push and what content is included. It has the following fields:

`kinds` (String[], optional):
: The Chat kinds for which push is enabled. Each entry is one of `"direct"`, `"group"`, or `"channel"`. If absent or null, push is enabled for all kinds.

`chatIds` (String[], optional):
: An explicit list of Chat ids for which push is enabled. If absent or null, push is enabled for all Chats of the matching kinds that the account is a member of. If both `kinds` and `chatIds` are present, a Chat matches only if its id appears in `chatIds` and its kind appears in `kinds`. A Chat whose id is in `chatIds` but whose kind is not in `kinds` receives no push; clients SHOULD ensure the two fields are consistent to avoid unexpected gaps in coverage.

`properties` (String[], optional):
: The `ChatMessageEntry` fields to include in each payload entry. If absent, the server uses the default set: `messageId`, `chatId`, `chatKind`, `senderId`, `senderDisplayName`, `sentAt`, `hasMention`, `mentionScopes`, `encrypted`, and `bodySnippet`. The `properties` list is authoritative: the server returns exactly the requested fields. The server MUST support at minimum the following properties: `messageId`, `chatId`, `chatKind`, `chatName`, `spaceId`, `spaceName`, `senderId`, `senderDisplayName`, `sentAt`, `hasMention`, `mentionScopes`, `encrypted`, and `bodySnippet`.

`urgency` (String, optional):
: The Web Push urgency level for notifications generated from this configuration. MUST be one of the values in `supportedUrgencyValues`. Default is `"normal"`. If the client supplies a value not in `supportedUrgencyValues`, the server MUST return an `invalidArguments` SetError on `PushSubscription/set` and MUST NOT store the configuration.

`mentionUrgency` (String, optional):
: When present, overrides `urgency` for payloads that contain a direct @user mention (`hasMention: true`) or a broadcast-scope mention (`mentionScopes` is non-empty). MUST be one of the values in `supportedUrgencyValues`. If absent, mention messages use the same urgency as all other messages. The same `invalidArguments` rule applies as for `urgency`.

# Push Payload {#push-payload}

## ChatMessagePush {#chat-message-push}

A `ChatMessagePush` object is the push payload delivered directly to the registered push endpoint. It is a standalone object, distinct from the `StateChange` object defined in {{RFC8620}} Section 7.1.

`@type` (String):
: MUST be `"ChatMessagePush"`.

`accountId` (String):
: The account id for which this payload was generated.

`state` (String):
: The `Message` state string for this account after all messages in the `messages` array have been applied, equivalent to the value that would appear in `StateChange.changed[accountId]["Message"]` following those deliveries. The server MUST set this field to the post-delivery state, so that a client using it as `sinceState` in a subsequent `Message/changes` call will not re-receive any message already present in this payload. Clients SHOULD update their locally cached `Message` state on receipt.

`messages` (ChatMessageEntry[]):
: MUST be a non-empty array of message entries that triggered this push. Contains exactly one entry for a single new message, or multiple entries when the server batches messages that arrived in quick succession (see {{delivery}}).

## ChatMessageEntry {#chat-message-entry}

A `ChatMessageEntry` carries the inline notification data for one message. The fields present in a given entry are governed by the `properties` list in the associated `ChatPushConfig` (see {{chat-push-config}}). The following fields are defined:

`messageId` (String):
: The id of the new message as assigned by the receiving server (`Message.id` in {{JMAP-CHAT}}). Clients MAY use this to deduplicate notifications.

`chatId` (String):
: The id of the Chat containing the message.

`chatKind` (String):
: The `kind` of the Chat: `"direct"`, `"group"`, or `"channel"`.

`chatName` (String, optional):
: The display name of the Chat. Present for `"group"` and `"channel"` kinds. Absent for `"direct"` chats; clients SHOULD use `senderDisplayName` as the notification title instead.

`spaceId` (String, optional):
: For `"channel"` chats, the id of the containing Space. Absent for other kinds.

`spaceName` (String, optional):
: For `"channel"` chats, the display name of the containing Space. Absent for other kinds.

`senderId` (String):
: The ChatContact.id of the message sender. This is the authoritative sender identity.

`senderDisplayName` (String, optional):
: The sender's display name at push-generation time. For `chatKind: "channel"`, derived from the sender's `SpaceMember.nick` in the containing Space if present, otherwise `ChatContact.displayName` if present, otherwise `ChatContact.login`, otherwise `senderId`. For `"direct"` and `"group"` chats, derived from `ChatContact.displayName` if present, otherwise `ChatContact.login`, otherwise `senderId`. The `senderId` fallback occurs when the ChatContact record has not yet been fully resolved — for example, when a message arrives from a previously unknown peer in a federated deployment. This value is a snapshot and may become stale; clients MUST NOT treat it as an authoritative identity signal.

`sentAt` (UTCDate):
: The sender's claimed composition time, as stored in the message. Included for display purposes, such as showing the send time in the notification preview. This is a client-supplied value and MUST NOT be used for ordering.

`hasMention` (Boolean):
: `true` if the account owner is directly @-mentioned in the message; `false` otherwise. The server MUST set this to `true` when the account owner's ChatContact.id appears in the message's `Message.mentions` array, OR when the message's `bodyType` is `"application/jmap-chat-rich"` and any span of `type: "mention"` carries the owner's ChatContact.id in its `userId` field. (For rich-body messages, `Message.mentions` is mandated empty by {{JMAP-CHAT}}; mention information is carried inline within the spans.) Clients MAY use this to render direct-mention notifications with a distinct sound or badge style.

`mentionScopes` (String[]):
: The broadcast-mention scopes by which the account owner was targeted in this message. Each element is one of `"everyone"`, `"here"`, or `"admins"` as defined by the `BroadcastMention` type in {{JMAP-CHAT}}. The server MUST compute the candidate scopes as the deduplicated union of (a) the `scope` values appearing in `Message.broadcastMentions[]` and (b) the `scope` values of any `"broadcast"` spans when `bodyType` is `"application/jmap-chat-rich"` (for rich-body messages, `Message.broadcastMentions` is mandated empty by {{JMAP-CHAT}}; broadcast-mention information is carried inline within the spans). The server MUST then filter this candidate set to those scopes for which the receiving account owner is in the locally-resolved recipient set: `"everyone"` is included whenever the owner is a member of the Chat; `"here"` is included only when the owner satisfies the deployment-defined "active" predicate at delivery time; `"admins"` is included only when the owner satisfies the deployment-defined "administrative authority" predicate at delivery time. The empty array means either that no broadcast scopes were used or that the owner was not targeted by any of them. A message MAY have both `hasMention: true` and a non-empty `mentionScopes`. Order within the array is not significant; clients SHOULD treat it as a set. Send-side authorization for any non-empty broadcast mention is gated by the `"mention_broadcast"` permission in {{JMAP-CHAT}}; per-recipient resolution is described in Broadcast Mention Resolution in {{JMAP-CHAT-FED}}.

`encrypted` (Boolean):
: `true` if the message body is end-to-end encrypted and the server cannot produce a plaintext snippet; `false` otherwise. See {{e2ee}} for how servers determine this value. When `true`, `bodySnippet` MUST be absent.

`bodySnippet` (String, optional):
: A truncated plaintext rendering of the message body, suitable for display in a notification preview. At most `maxSnippetBytes` bytes, truncated on a UTF-8 character boundary. MUST be absent when `encrypted` is `true`. When the message's `bodyType` is `"text/markdown"` or `"application/jmap-chat-rich"`, the server SHOULD generate a plaintext approximation rather than including raw markup syntax.

# End-to-End Encrypted Deployments {#e2ee}

In relay deployments using end-to-end encryption (see {{JMAP-CHAT}} Security Considerations and {{RFC9420}}), the message body is ciphertext and the server cannot generate a meaningful plaintext snippet.

The server determines the `encrypted` field value from the message's `bodyType`. For the plaintext body types defined by {{JMAP-CHAT}} — `"text/plain"`, `"text/markdown"`, and `"application/jmap-chat-rich"` — the server MUST set `encrypted` to `false`. For all other `bodyType` values, the server MUST set `encrypted` to `true` unless it has explicit knowledge that the type represents a plaintext format. The server MUST omit `bodySnippet` when `encrypted` is `true` and MUST NOT include ciphertext in `bodySnippet`.

The remaining fields — `messageId`, `chatId`, `chatKind`, `chatName`, `senderId`, `senderDisplayName`, `sentAt`, `hasMention`, `mentionScopes`, and `encrypted` — carry enough context for clients to display a useful notification such as "New message from Alice" or "@mention in #general" without exposing message content. Servers MUST NOT decrypt message content to generate a snippet.

# Delivery {#delivery}

When a new inbound message is stored that matches a `ChatPushConfig`, the server MUST construct a `ChatMessagePush` payload and deliver it to the registered push endpoint for that `PushSubscription`.

The server MUST NOT deliver a `ChatMessagePush` in any of the following cases:

- The message was stored before the `PushSubscription` was created.
- The sender is the account owner (push is for inbound messages from other participants only).
- At the time of delivery, the account is no longer a member of the Chat containing the message.
- The Chat is muted (`Chat.muted: true` or within an active `muteUntil` period).

## Urgency

The server uses `urgency` as the Web Push urgency for the delivery. If the `ChatPushConfig` specifies a `mentionUrgency` and at least one entry in `messages` has `hasMention: true` or a non-empty `mentionScopes`, the server MUST use `mentionUrgency` for the entire payload instead. When a batch contains both mention and non-mention messages, all messages share the elevated urgency to ensure the mention notification is not silently demoted.

## Batching and Retry

Servers SHOULD deliver at most one `ChatMessagePush` per message per `PushSubscription`. If delivery fails and the server retries, the payload MUST NOT be altered between attempts. When a single account has multiple `PushSubscription` objects with `chatPush` configurations, each subscription is evaluated and delivered to independently.

If multiple new messages arrive in quick succession, servers MAY include multiple `ChatMessageEntry` objects in the `messages` array of a single `ChatMessagePush` rather than delivering a separate payload per message. When `maxMessagesPerPush` is advertised, the server MUST NOT exceed it; if more matching messages are pending than the limit allows, the server MUST deliver additional `ChatMessagePush` payloads to cover the remainder.

## Rate Limiting and Fallback

Servers SHOULD rate-limit `ChatMessagePush` delivery per `PushSubscription`. When the delivery rate exceeds an implementation-defined threshold, the server MAY fall back to delivering a plain `StateChange` for the `Message` data type. Clients receiving a `StateChange` on a `PushSubscription` for which `chatPush` is configured MUST handle it identically to an unaugmented `StateChange`: by calling `Message/changes` to determine which messages are new.

## Example Payload

~~~json
{
  "@type": "ChatMessagePush",
  "accountId": "u1",
  "state": "d35ecb040aab",
  "messages": [
    {
      "messageId": "01J3YKZQP5MWVT8PPBEHTJ3HX",
      "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX",
      "chatKind": "channel",
      "chatName": "general",
      "spaceId": "01J2WKZQM3LVST7OOBDHSI2GW",
      "spaceName": "ACME Corp",
      "senderId": "user:alice@example.com",
      "senderDisplayName": "Alice",
      "sentAt": "2026-04-26T14:32:00Z",
      "hasMention": true,
      "mentionScopes": [],
      "encrypted": false,
      "bodySnippet": "Hey @bob, the deploy is ready for review"
    }
  ]
}
~~~

In this example the message body is plain text, so `hasMention` was computed from `Message.mentions` and `mentionScopes` is empty because the body contains no broadcast mention. When the underlying message uses `bodyType: "application/jmap-chat-rich"`, both `Message.mentions` and `Message.broadcastMentions` are mandated empty by {{JMAP-CHAT}}; the server computes `hasMention` from inline `"mention"` spans and `mentionScopes` from inline `"broadcast"` spans instead (see {{chat-message-entry}}). The resulting payload structure is otherwise identical.

The following variant shows the payload when the same channel message also tags `@here`:

~~~json
{
  "@type": "ChatMessagePush",
  "accountId": "u1",
  "state": "d35ecb040aab",
  "messages": [
    {
      "messageId": "01J3YKZQP5MWVT8PPBEHTJ3HX",
      "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX",
      "chatKind": "channel",
      "chatName": "general",
      "spaceId": "01J2WKZQM3LVST7OOBDHSI2GW",
      "spaceName": "ACME Corp",
      "senderId": "user:alice@example.com",
      "senderDisplayName": "Alice",
      "sentAt": "2026-04-26T14:32:00Z",
      "hasMention": true,
      "mentionScopes": ["here"],
      "encrypted": false,
      "bodySnippet": "Hey @bob and @here, the deploy is ready for review"
    }
  ]
}
~~~

The receiving server set `mentionScopes` to `["here"]` because the owner appeared in the `@here` recipient set under the deployment's `@here` predicate at delivery time (see Broadcast Mention Resolution in {{JMAP-CHAT-FED}}). The owner's id was also in `Message.mentions`, so `hasMention` remains `true`.

# Security Considerations {#security}

## Push Channel Encryption

`ChatMessagePush` payloads contain message metadata and, for unencrypted messages, body snippets. The push channel SHOULD be encrypted per the Web Push message encryption specification {{RFC8291}} to protect this content in transit. Servers SHOULD NOT deliver `bodySnippet` content over unencrypted push channels.

Note that push-channel encryption ({{RFC8291}}) and message body end-to-end encryption ({{RFC9420}}, {{e2ee}}) are orthogonal: a message body may be end-to-end encrypted regardless of whether the push channel is encrypted, and vice versa.

## Notification Content Sanitization

Body snippets are rendered by OS notification systems outside the application sandbox. Servers SHOULD strip or escape content that could be misinterpreted by the target platform's notification renderer — in particular, markup syntax, HTML tags, and control characters SHOULD be removed from `bodySnippet` before inclusion in the payload.

## Display Name Freshness

`senderDisplayName` is a snapshot taken at push-generation time and is for display purposes only. Clients MUST NOT use it as an authoritative identity signal. The authoritative sender identity is `senderId` (the verified ChatContact.id).

## Rate Limiting

Without rate limiting, a flood of incoming messages could drive unbounded push delivery and exhaust push infrastructure. The rate-limiting and fallback behavior described in {{delivery}} SHOULD be enforced server-side.

## Federation

Push payloads are generated and delivered by the receiving server. In federated deployments ({{JMAP-CHAT-FED}}), the receiving server generates the `ChatMessagePush` from locally stored message data. The originating server is not involved in push delivery and has no visibility into whether or how the notification was delivered.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:push`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

--- back

# Acknowledgements

The author thanks the authors of {{JMAP-EMAILPUSH}} for the push notification pattern this specification follows, and the JMAP working group for {{RFC8620}}.
