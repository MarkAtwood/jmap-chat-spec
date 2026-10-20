---
title: JMAP Chat Push Notifications
abbrev: JMAP Chat Push
docname: draft-atwood-jmap-chat-push-00
category: std
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
  RFC8620:
  RFC8030:
  RFC8291:
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026

informative:
  JMAP-EMAILPUSH:
    title: "JSON Meta Application Protocol (JMAP) Email Delivery Push Notifications"
    author:
      - fullname: Neil Jenkins
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-emailpush-03
    date: 2026-05-03
  JMAP-CHAT-FED:
    title: "JMAP Chat Federation"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026
  RFC9420:

--- abstract

This document defines JMAP Chat Push Notifications, a companion specification to JMAP Chat ({{JMAP-CHAT}}). It extends the RFC 8620 PushSubscription mechanism to deliver inline message content in push notification payloads, enabling mobile and background clients to display a notification without a follow-up server request. The extension defines a per-account push configuration, a notification payload object carrying message metadata and an optional body snippet, per-chat and per-kind filtering, mention-aware urgency, and behavior for end-to-end encrypted deployments where body content must not leave the server.

--- middle

# Introduction

The RFC 8620 push model was designed as an edge trigger: the server delivers a small `StateChange` event carrying an opaque state token, and the client calls `/changes` to discover what is new. This "edge trigger then resync" pattern works well for email, where the client application is typically already running and maintains a persistent connection or periodic sync interval.

Chat notifications have a different operational constraint. Every chat message is time-sensitive, and mobile platforms (iOS, Android) penalize applications that receive a push, wake, perform network requests, and fail to display a notification promptly. A `StateChange` push requires the device to wake, establish a TLS connection, authenticate, issue `Message/changes`, issue `Message/get`, parse the response, and render — a sequence that routinely exceeds mobile platform budgets for background execution time. The result is missed or delayed notifications, OS-imposed throttling, and degraded user experience.

{{JMAP-EMAILPUSH}} recognized this same constraint for email delivery notifications and introduced a pattern in which the server includes message content directly in the push payload, eliminating the follow-up round-trip. This document applies that pattern to chat: it extends `PushSubscription` with a `chatPush` property that instructs the server to deliver a `ChatMessagePush` object — a standalone push payload carrying message metadata and an optional body snippet — directly to the push endpoint. A mobile client receiving this payload can render a complete notification immediately, without any follow-up request. The underlying push infrastructure ({{RFC8620}}, {{RFC8030}}) is unchanged; the extension is purely in what the payload contains.

## Design Principles

- The extension is additive to the RFC 8620 `PushSubscription`. Servers that do not support this specification deliver unmodified `StateChange` payloads; clients are expected to handle those regardless.
- Per-account and per-chat filtering allows clients to limit push volume to the conversations that matter.
- Mention-aware urgency allows a single configuration to request high-urgency delivery for mentions and normal-urgency delivery for everything else.
- In end-to-end encrypted deployments, the server cannot access plaintext body content. The `encrypted` flag and metadata-only mode provide enough context to display a useful notification without exposing ciphertext. Deployments where the push platform operator must not learn who messaged whom omit `chatPush` and rely on the RFC 8620 `StateChange`, which carries no message metadata.
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

If a `PushSubscription/set` create or update includes a `chatPush` entry for an account whose `accountCapabilities` does not include `urn:ietf:params:jmap:chat:push`, the server MUST return an `invalidProperties` SetError whose `properties` names the offending entry (e.g., `chatPush/<id>`) and MUST NOT store the subscription change.

## ChatPushConfig {#chat-push-config}

A `ChatPushConfig` specifies which messages trigger inline push and what content is included. It has the following fields:

`kinds` (String[], optional):
: The Chat kinds for which push is enabled. Each entry is one of `"direct"`, `"group"`, or `"channel"`. If absent or null, push is enabled for all kinds.

`chatIds` (String[], optional):
: An explicit list of Chat ids for which push is enabled. If absent or null, push is enabled for all Chats of the matching kinds that the account is a member of. If both `kinds` and `chatIds` are present, a Chat matches only if its id appears in `chatIds` and its kind appears in `kinds`. A Chat whose id is in `chatIds` but whose kind is not in `kinds` receives no push; clients SHOULD ensure the two fields are consistent to avoid unexpected gaps in coverage.

`properties` (String[], optional):
: The `ChatMessageEntry` fields to include in each payload entry. If absent, the server uses the default set: `messageId`, `chatId`, `chatKind`, `senderId`, `senderDisplayName`, `sentAt`, `hasMention`, `mentionScopes`, `encrypted`, and `bodySnippet`. The `properties` list is authoritative: the server returns exactly the requested fields. The server MUST support at minimum the following properties: `messageId`, `chatId`, `chatKind`, `chatName`, `spaceId`, `spaceName`, `senderId`, `senderDisplayName`, `sentAt`, `hasMention`, `mentionScopes`, `encrypted`, and `bodySnippet`.

`urgency` (String, optional):
: The Web Push urgency level for notifications generated from this configuration. MUST be one of the values in `supportedUrgencyValues`. Default is `"normal"`. If the client supplies a value not in `supportedUrgencyValues`, the server MUST return an `invalidProperties` SetError on `PushSubscription/set` whose `properties` names the offending path (e.g., `chatPush/<id>/urgency`) and MUST NOT store the configuration.

`mentionUrgency` (String, optional):
: When present, overrides `urgency` for payloads that contain a direct @user mention (`hasMention: true`) or a broadcast-scope mention (`mentionScopes` is non-empty). MUST be one of the values in `supportedUrgencyValues`. If absent, mention messages use the same urgency as all other messages. The same `invalidProperties` rule applies as for `urgency`, naming `chatPush/<id>/mentionUrgency`.

# Push Payload {#push-payload}

## ChatMessagePush {#chat-message-push}

A `ChatMessagePush` object is the push payload delivered directly to the registered push endpoint. It is a standalone object, distinct from the `StateChange` object defined in {{RFC8620}} Section 7.1.

`@type` (String):
: MUST be `"ChatMessagePush"`.

`accountId` (String):
: The account id for which this payload was generated.

`state` (String):
: The `Message` state string for this account after all messages in the `messages` array have been applied, equivalent to the value that would appear in `StateChange.changed[accountId]["Message"]` following those deliveries. The server MUST set this field to the post-delivery state. This state also covers Message changes that the payload does not carry, such as the owner's own messages and messages excluded by `ChatPushConfig`, mute, or block (see {{delivery}}), so clients SHOULD NOT replace their cached `Message` state with it. As with `StateChange` ({{RFC8620}} Section 7.1), state strings are opaque and are compared only for equality: if `state` differs from the client's cached `Message` state, the client calls `Message/changes` with its cached state as `sinceState`. This also handles pushes delivered out of order, whose relative age `state` cannot show.

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
: The display name of the Chat. Present for `"group"` and `"channel"` kinds. Absent for `"direct"` chats; clients can use `senderDisplayName` as the notification title instead.

`spaceId` (String, optional):
: For `"channel"` chats, the id of the containing Space. Absent for other kinds.

`spaceName` (String, optional):
: For `"channel"` chats, the display name of the containing Space. Absent for other kinds.

`senderId` (String):
: The accountId for owner-composed messages; otherwise the ChatContact.id of the message sender. This is the authoritative sender identity.

`senderDisplayName` (String, optional):
: The sender's display name at push-generation time. For `chatKind: "channel"`, derived from the sender's `SpaceMember.nick` in the containing Space if present, otherwise `ChatContact.displayName` if present, otherwise `ChatContact.login`, otherwise `ChatContact.identity`. For `"direct"` and `"group"` chats, derived from `ChatContact.displayName` if present, otherwise `ChatContact.login`, otherwise `ChatContact.identity`. The `identity` fallback occurs when the ChatContact record has not yet been fully resolved — for example, when a message arrives from a previously unknown peer in a federated deployment and the server holds only a placeholder ChatContact for that peer. This value is a snapshot and may become stale; clients SHOULD NOT treat it as an authoritative identity signal.

`sentAt` (UTCDate):
: The sender's claimed composition time, as stored in the message. Included for display purposes, such as showing the send time in the notification preview. This is a client-supplied value and, as in {{JMAP-CHAT}}, is not used for ordering.

`hasMention` (Boolean, optional):
: `true` if the account owner is directly @-mentioned in the message; `false` otherwise. MUST be absent when `encrypted` is `true` (see {{e2ee}}). The server MUST set this to `true` when the accountId, which identifies the account owner ({{JMAP-CHAT}}), appears as a Mention `id` in the message's `Message.mentions` array, for every `bodyType`. The server MUST NOT derive mentions from the message body, including a rich body's spans ({{JMAP-CHAT}}). Clients MAY use this to render direct-mention notifications with a distinct sound or badge style.

`mentionScopes` (String[], optional):
: The broadcast-mention scopes by which the account owner was targeted in this message. MUST be absent when `encrypted` is `true` (see {{e2ee}}). Each element is one of `"everyone"`, `"here"`, or `"admins"` as defined by the `BroadcastMention` type in {{JMAP-CHAT}}. The server MUST compute the candidate scopes as the deduplicated `scope` values appearing in `Message.broadcastMentions[]`, for every `bodyType`; a rich body's `"broadcast"` spans are display-only and are not consulted ({{JMAP-CHAT}}). The server MUST then filter this candidate set to those scopes for which the receiving account owner is in the locally-resolved recipient set: `"everyone"` is included whenever the owner is a member of the Chat; `"here"` is included only when the owner satisfies the deployment-defined "active" predicate at delivery time; `"admins"` is included only when the owner satisfies the deployment-defined "administrative authority" predicate at delivery time. The empty array means either that no broadcast scopes were used or that the owner was not targeted by any of them. A message MAY have both `hasMention: true` and a non-empty `mentionScopes`. Order within the array is not significant; clients SHOULD treat it as a set. Send-side authorization for any non-empty broadcast mention is gated by the `"mention_broadcast"` permission in {{JMAP-CHAT}}; per-recipient resolution is described in Broadcast Mention Resolution in {{JMAP-CHAT-FED}}.

`encrypted` (Boolean):
: `true` if the message body is end-to-end encrypted and the server cannot produce a plaintext snippet; `false` otherwise. See {{e2ee}} for how servers determine this value. When `true`, `bodySnippet` MUST be absent.

`bodySnippet` (String, optional):
: A truncated plaintext rendering of the message body, suitable for display in a notification preview. At most `maxSnippetBytes` bytes, truncated on a UTF-8 character boundary. When the message's `bodyType` is `"text/markdown"` or `"application/jmap-chat-rich"`, the server SHOULD generate a plaintext approximation rather than including raw markup syntax.

`unreadCount` (UnsignedInt|null, optional):
: The account owner's total unread message count across all non-muted Chats at the time this push was generated. Servers SHOULD include this for platforms that support atomic badge updates (e.g., APNs `aps.badge`). When absent, the client can derive the badge count via a subsequent `Chat/get` request.

# End-to-End Encrypted Deployments {#e2ee}

In relay deployments using end-to-end encryption (see {{JMAP-CHAT}} Security Considerations and {{RFC9420}}), the message body is ciphertext and the server cannot generate a meaningful plaintext snippet.

The server determines the `encrypted` field value from the message's `bodyType`. For the plaintext body types defined by {{JMAP-CHAT}} — `"text/plain"`, `"text/markdown"`, and `"application/jmap-chat-rich"` — the server MUST set `encrypted` to `false`. For all other `bodyType` values, the server MUST set `encrypted` to `true` unless it has explicit knowledge that the type represents a plaintext format. When `encrypted` is `true`, `bodySnippet` is absent ({{chat-message-entry}}).

For an encrypted message, mentions and broadcast mentions are carried inside the ciphertext (see Fields Inside the Ciphertext in {{JMAP-CHAT}}), so the server cannot compute `hasMention` or `mentionScopes`. Both fields are therefore absent when `encrypted` is `true` ({{chat-message-entry}}). Such a message never triggers `mentionUrgency` and never bypasses mute suppression (see {{delivery}}); clients determine mentions after decryption.

The remaining fields (`messageId`, `chatId`, `chatKind`, `chatName`, `senderId`, `senderDisplayName`, `sentAt`, and `encrypted`) carry enough context for clients to display a useful notification such as "New message from Alice" without exposing message content.

These fields still reveal who sent a message to whom, in which named Chat and Space, and when, to any party that can read the push payload. On push platforms other than Web Push with {{RFC8291}} encryption, that includes the platform operator (see {{platform-visibility}}).

Clients in end-to-end encrypted deployments SHOULD NOT set `chatPush` for an account on any `PushSubscription` delivered through a platform push service other than Web Push with {{RFC8291}} encryption. The server then delivers {{RFC8620}} `StateChange` notifications for that account, which carry no message metadata; after waking, the client fetches new messages with `Message/changes` and `Message/get` and decrypts them locally. Such a subscription loses per-Chat filtering, mute suppression, and mention urgency, because those are properties of `ChatPushConfig`.

# Delivery {#delivery}

When a new inbound message is stored that matches a `ChatPushConfig`, the server MUST construct a `ChatMessagePush` payload and deliver it to the registered push endpoint for that `PushSubscription`.

The server MUST NOT deliver a `ChatMessagePush` in any of the following cases:

- The message was stored before the `PushSubscription` was created.
- The sender is the account owner (push is for inbound messages from other participants only).
- At the time of delivery, the account is no longer a member of the Chat containing the message.
- The Chat is muted (`Chat.muted: true` or within an active `muteUntil` period), UNLESS the message contains a broadcast mention whose resolved recipient set includes the account owner (per the Chat `muted` field, {{JMAP-CHAT}} Section 4.11).
- The message sender's ChatContact record has `blocked: true` on the account owner's contact list.

For clarity: a direct `@user` mention (`hasMention: true` with empty `mentionScopes`) in a muted Chat does NOT bypass the mute suppression. Only broadcast mentions with non-empty `mentionScopes` targeting the account owner bypass mute, per {{JMAP-CHAT}}.

## Urgency

The server uses `urgency` as the Web Push urgency for the delivery. If the `ChatPushConfig` specifies a `mentionUrgency` and at least one entry in `messages` has `hasMention: true` or a non-empty `mentionScopes`, the server MUST use `mentionUrgency` for the entire payload instead. When a batch contains both mention and non-mention messages, all messages share the elevated urgency to ensure the mention notification is not silently demoted.

## Batching and Retry

Servers SHOULD deliver at most one `ChatMessagePush` per message per `PushSubscription`. If delivery fails and the server retries, the payload SHOULD NOT be altered between attempts. When a single account has multiple `PushSubscription` objects with `chatPush` configurations, each subscription is evaluated and delivered to independently.

If multiple new messages arrive in quick succession, servers MAY include multiple `ChatMessageEntry` objects in the `messages` array of a single `ChatMessagePush` rather than delivering a separate payload per message. The `maxMessagesPerPush` bound ({{capability}}) applies; if more matching messages are pending than the limit allows, the server MUST deliver additional `ChatMessagePush` payloads to cover the remainder.

Servers keep each `ChatMessagePush` within the push platform's payload size limit, which is implementation-defined. If a single `ChatMessageEntry` does not fit, the server SHOULD shorten or omit that entry's `bodySnippet`; if the entry still does not fit, the server delivers a `StateChange` for the `Message` data type in place of a `ChatMessagePush` for that message.

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
      "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX0",
      "chatKind": "channel",
      "chatName": "general",
      "spaceId": "01J2WKZQM3LVST7OOBDHSI2GW",
      "spaceName": "ACME Corp",
      "senderId": "c8xq2m",
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

In this example the message body is plain text, so `hasMention` was computed from `Message.mentions` and `mentionScopes` is empty because the body contains no broadcast mention. The same computation applies when the underlying message uses `bodyType: "application/jmap-chat-rich"`: `hasMention` and `mentionScopes` come from `Message.mentions` and `Message.broadcastMentions`, never from the body's spans (see {{chat-message-entry}}).

The following variant shows the payload when the same channel message also tags `@here`:

~~~json
{
  "@type": "ChatMessagePush",
  "accountId": "u1",
  "state": "d35ecb040aab",
  "messages": [
    {
      "messageId": "01J3YKZQP5MWVT8PPBEHTJ3HX",
      "chatId": "01J3XKZQN4MWVT8PPBEHTJ3HX0",
      "chatKind": "channel",
      "chatName": "general",
      "spaceId": "01J2WKZQM3LVST7OOBDHSI2GW",
      "spaceName": "ACME Corp",
      "senderId": "c8xq2m",
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

## Platform Operator Visibility {#platform-visibility}

{{RFC8291}} encryption applies only to Web Push. Platform push services such as Apple Push Notification service (APNs) and Firebase Cloud Messaging (FCM) relay payloads through infrastructure run by the platform operator, and those payloads are visible to the operator unless the server encrypts them at the application layer. Without such encryption, a `ChatMessagePush` delivered over these platforms discloses its `ChatMessageEntry` fields, including `senderId`, `senderDisplayName`, `chatName`, and `spaceName`, even when `encrypted` is `true` and `bodySnippet` is absent.

A subscription without `chatPush` for the account ({{e2ee}}) receives only {{RFC8620}} `StateChange` payloads, which carry account ids, type names, and state strings. The operator can still observe the device token, delivery time and frequency, and payload size.

## Notification Content Sanitization

Body snippets are rendered by OS notification systems outside the application sandbox. Servers SHOULD strip or escape content that could be misinterpreted by the target platform's notification renderer — in particular, markup syntax, HTML tags, and control characters SHOULD be removed from `bodySnippet` before inclusion in the payload.

## Display Name Freshness

`senderDisplayName` is a snapshot taken at push-generation time and is for display purposes only and is not an authoritative identity signal ({{chat-message-entry}}). The authoritative sender identity is `senderId` (the accountId for owner-composed messages, otherwise the verified ChatContact.id of the sender).

## Rate Limiting

Without rate limiting, a flood of incoming messages could drive unbounded push delivery and exhaust push infrastructure. The rate-limiting and fallback behavior described in {{delivery}} addresses this.

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
