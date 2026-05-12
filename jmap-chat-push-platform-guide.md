# JMAP Chat Push — Platform Supplement

Supplements the [JMAP Push Platform Delivery Guide](jmap-push-platform-guide.md) with
details specific to `draft-atwood-jmap-chat-push-00`. Read that guide first — it covers
authentication, request format, urgency mapping, error handling, and size limits for every
supported platform. This document covers only what changes when the payload is a
`ChatMessagePush` object: how to embed it in each platform's envelope, how to derive
urgency from `ChatPushConfig`, and how to truncate oversized payloads.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or deployment
  choice (e.g., "servers SHOULD log admin actions"), the keyword reflects implementer
  best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## Overview

`ChatMessagePush` is an extension to the RFC 8620 `PushSubscription` mechanism. When a
server supports this extension, it advertises the `urn:ietf:params:jmap:chat:push`
capability. Clients that want inline push payloads add a `chatPush` property to their
`PushSubscription` objects. The server then delivers `ChatMessagePush` payloads directly
to the push endpoint instead of (or in addition to) the standard `StateChange` object.

Before registering a `chatPush` configuration, clients SHOULD check the account-level
capability object for `urn:ietf:params:jmap:chat:push`. It contains:

- `maxSnippetBytes` — the maximum byte length the server will include in `bodySnippet`
- `supportedUrgencyValues` — the urgency values the server accepts in `ChatPushConfig`
- `maxMessagesPerPush` (optional) — the maximum number of entries in one `ChatMessagePush`

Clients MUST NOT request an `urgency` or `mentionUrgency` value not in
`supportedUrgencyValues`; the server will reject the `PushSubscription/set` call with
`invalidArguments`.

If `urn:ietf:params:jmap:chat:push` is absent from the account capabilities, the server
does not support inline `ChatMessagePush` payloads. Clients MUST fall back to a standard
RFC 8620 `PushSubscription` without a `chatPush` property. In that case the server
delivers plain `StateChange` events for the `Message` data type; clients handle them by
calling `Message/changes` to determine which messages are new — the same flow as the
rate-limit fallback described below. Clients MUST NOT include a `chatPush` property in
`PushSubscription/set` calls when the capability is absent; the server will reject the
call with `invalidArguments`.

### When the server does not send a push

The server suppresses `ChatMessagePush` delivery in these cases:

- The message was stored before the `PushSubscription` was created.
- The sender is the account owner (push is for inbound messages only).
- The account is no longer a member of the chat at delivery time.
- The chat is muted (`Chat.muted: true` or within an active `muteUntil` period).

When the delivery rate exceeds the server's rate limit, the server MAY fall back to a
plain `StateChange` for the `Message` data type. Clients receiving a `StateChange` on a
subscription with `chatPush` configured MUST handle it the same way as any other
`StateChange`: call `Message/changes` to find out which messages are new.

---

## Payload encoding

`ChatMessagePush` is the push payload. How it is embedded differs by platform:

| Platform | Where `ChatMessagePush` goes | Encoding |
|---|---|---|
| FCM | `message.data["jmap-push"]` | JSON-serialized string |
| ADM | `data["jmap-push"]` | JSON-serialized string |
| HPK | `message.data` | JSON-serialized string |
| MiPush | `payload` form field | URL-encoded JSON string |
| APNs | `payload["jmap-push"]` | Inline JSON object |
| Web Push | Entire encrypted body | Raw JSON bytes |
| WNS | Entire raw body | Raw JSON bytes |

### FCM and ADM

`data` map values must be strings. Serialize `ChatMessagePush` as JSON and place it under
`"jmap-push"`:

```json
"data": {
  "jmap-push": "{ChatMessagePush serialized as a JSON string}"
}
```

### HPK

HPK's `data` field is a single string (not a map). Serialize `ChatMessagePush` as JSON
and place the string there directly:

```json
"data": "{ChatMessagePush serialized as a JSON string}"
```

### MiPush

URL-encode the `ChatMessagePush` JSON and pass it as the `payload` form field:

```
payload={url_encoded_ChatMessagePush_json}
```

### APNs

Embed `ChatMessagePush` inline as a JSON object under `"jmap-push"`.

**Background push** (opportunistic delivery, no user-visible notification until the app
wakes):

```json
{
  "aps": {
    "content-available": 1
  },
  "jmap-push": { ...ChatMessagePush object... }
}
```

**Alert push** (immediate delivery; requires a Notification Service Extension to rewrite
the placeholder before the OS displays it):

```json
{
  "aps": {
    "alert": { "body": "New message" },
    "mutable-content": 1
  },
  "jmap-push": { ...ChatMessagePush object... }
}
```

See the platform delivery guide for when each mode is appropriate and how to set
`apns-push-type` and `apns-priority` for each.

### Web Push and WNS

`ChatMessagePush` JSON is the entire message body — no wrapper envelope is needed.

---

## Urgency

Derive the platform urgency from `ChatPushConfig.urgency` using the mapping tables in the
platform delivery guide.

If `ChatPushConfig.mentionUrgency` is set and at least one entry in `messages` has
`hasMention: true` or `hasMentionAll: true`, use `mentionUrgency` instead of `urgency`
for the entire push. When a batch mixes mention and non-mention messages, all messages
in that payload share the elevated urgency, because the mention notification must not be
silently demoted.

---

## The `state` field

`ChatMessagePush.state` is the `Message` state token for the account after all messages
in the payload have been applied. Clients SHOULD update their locally cached `Message`
state token on receipt. Using this value as `sinceState` in a subsequent `Message/changes`
call will not re-surface messages already present in the payload.

---

## Payload truncation

When a `ChatMessagePush` payload exceeds the platform size limit (see the platform
delivery guide for per-platform limits), drop fields from `ChatMessageEntry` objects in
this order until the payload fits:

1. `bodySnippet` — the largest variable field; drop first
2. `spaceId` — opaque identifier with no display value
3. `sentAt` — display nicety; the recipient's local clock is close enough
4. `spaceName`
5. `chatName`
6. `senderDisplayName`

This truncation order applies when these fields are present in the payload (whether
delivered by default or because the client requested them via `properties`). If the
`ChatPushConfig` specifies a `properties` list that excludes some of these fields, they
will already be absent and can be skipped in the sequence.

Never drop: `@type`, `accountId`, `state` (on the outer object), `messageId`, `chatId`,
`chatKind`, `senderId`, `hasMention`, `hasMentionAll`, `encrypted`.

A client receiving a `ChatMessageEntry` with no `bodySnippet` should display a generic
"new message" notification and fetch full content via `Message/changes` on next foreground
wake — the same behavior as when `encrypted: true`.

If the payload still exceeds the limit after all droppable fields are removed from every
entry (typically due to a large batch), drop entries from the end of the `messages` array,
keeping the first entry. If a single minimal entry still exceeds the platform limit, fall
back to a plain `StateChange` for that subscription rather than sending an invalid or
oversized payload.

---

## Security notes

### Push channel encryption

`ChatMessagePush` payloads contain message metadata and, for unencrypted messages, body
snippets. Web Push (RFC 8291) is end-to-end encrypted — only the client device can
decrypt the payload. All other platforms route traffic through vendor infrastructure that
can read the payload in transit. Servers SHOULD NOT include `bodySnippet` content over
push channels that are not RFC 8291 encrypted.

Push channel encryption and message body end-to-end encryption are independent. A
message body may be E2E encrypted regardless of whether the push channel is encrypted,
and vice versa.

### Sender display name freshness

`senderDisplayName` is a snapshot taken at push-generation time. It is for display
purposes only. Clients MUST NOT use it as an authoritative identity signal. The
authoritative sender identity is `senderId` (the verified `ChatContact.id`).
