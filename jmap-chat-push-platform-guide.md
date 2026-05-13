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

`data` map values MUST be strings. Servers MUST serialize `ChatMessagePush` as JSON and
place it under `"jmap-push"`:

```json
"data": {
  "jmap-push": "{ChatMessagePush serialized as a JSON string}"
}
```

### HPK

HPK's `data` field is a single string (not a map). Servers MUST serialize
`ChatMessagePush` as JSON and place the string there directly:

```json
"data": "{ChatMessagePush serialized as a JSON string}"
```

### MiPush

Servers MUST URL-encode the `ChatMessagePush` JSON and pass it as the `payload` form
field:

```
payload={url_encoded_ChatMessagePush_json}
```

### APNs

Servers MUST embed `ChatMessagePush` inline as a JSON object under `"jmap-push"`.

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

Servers MUST derive the platform urgency from `ChatPushConfig.urgency` using the
mapping tables in the platform delivery guide.

If `ChatPushConfig.mentionUrgency` is set and at least one entry in `messages` has
`hasMention: true` or a non-empty `mentionScopes` array, the server MUST use
`mentionUrgency` instead of `urgency` for the entire push. When a batch mixes mention
and non-mention messages, all messages in that payload MUST share the elevated urgency
— the mention notification MUST NOT be silently demoted.

`mentionScopes` is per-recipient: the receiving server lists only the broadcast scopes
for which the account owner was in the locally-resolved recipient set at delivery time
(per `draft-atwood-jmap-chat-push-00` §`mentionScopes` and the federation draft's
Broadcast Mention Resolution rules). A recipient who received a message tagged `@here`
but who did not satisfy the deployment's "active" predicate at delivery time gets
`mentionScopes: []` and therefore no urgency elevation from that message alone.

### Per-scope urgency recommendations

The wire contract collapses all three broadcast scopes into a single elevation gate
(non-empty `mentionScopes` triggers `mentionUrgency`). Deployments that want finer
control over per-scope urgency typically run multiple `PushSubscription` records with
different `mentionUrgency` values, gated by the scope set on the client side, or filter
in the client itself. There is no wire field for "elevate `@everyone` but not `@here`"
because the spec defers per-scope preference to deployment policy (see
`draft-atwood-jmap-chat-00` §Chat.muted and §Broadcast Mention Abuse).

When choosing the deployment default for `mentionUrgency`, consider:

| Scope | Sender intent | Typical default elevation |
|---|---|---|
| `@everyone` | Page everyone in the Chat | High urgency; bypass mute |
| `@here` | Page currently-active members | High urgency; bypass mute |
| `@admins` | Page administrative staff | High urgency for admins; non-admins are not in the recipient set so do not see this scope at all |
| `@user` (per-user `hasMention: true`) | Page a specific person | High urgency, equal to broadcast |

Because the spec uses one `mentionUrgency` value for all four cases, deployments that
want a softer @here than @everyone need to handle it at the application layer (separate
PushSubscriptions, client-side filtering, or per-account UI rules). The push wire format
itself is intentionally simple.

### Client UI interpretation

The `mentionScopes` array is a signal for the client UI as well as a server-side
elevation trigger. Clients SHOULD use the array to differentiate notification
appearance:

- Empty `mentionScopes`: regular message; standard notification appearance.
- `["everyone"]`: a `@everyone` broadcast targeted the owner. Render a distinctive
  banner or sound that conveys "the whole room was paged"; users have a low tolerance
  for misuse of this scope and want it visually obvious.
- `["here"]`: a `@here` broadcast targeted the owner. Render similarly to a `@user`
  mention; the owner satisfied the deployment's "active" predicate at delivery time so
  the elevation matches their direct-attention expectations.
- `["admins"]`: an `@admins` broadcast targeted the owner. Render with an
  administrative-attention banner if the deployment offers one; the owner has been
  paged in an admin capacity.
- Multiple scopes (for example, `["everyone","admins"]`): a message that tags more
  than one scope. Clients SHOULD pick the highest-priority appearance among them
  rather than rendering separate banners.

Clients MUST silently ignore unknown scope values in `mentionScopes` (forward
compatibility for future scope additions).

---

## The `state` field

`ChatMessagePush.state` is the `Message` state token for the account after all messages
in the payload have been applied. Clients SHOULD update their locally cached `Message`
state token on receipt. Using this value as `sinceState` in a subsequent `Message/changes`
call will not re-surface messages already present in the payload.

---

## Payload truncation

When a `ChatMessagePush` payload exceeds the platform size limit (see the platform
delivery guide for per-platform limits), servers SHOULD drop fields from
`ChatMessageEntry` objects in this order until the payload fits:

1. `bodySnippet` — the largest variable field; drop first
2. `spaceId` — opaque identifier with no display value
3. `sentAt` — display nicety; the recipient's local clock is close enough
4. `spaceName`
5. `chatName`
6. `senderDisplayName`

This truncation order applies when these fields are present in the payload (whether
delivered by default or because the client requested them via `properties`). If the
`ChatPushConfig` specifies a `properties` list that excludes some of these fields, they
will already be absent and SHOULD be skipped in the sequence.

Servers MUST NOT drop: `@type`, `accountId`, `state` (on the outer object), `messageId`,
`chatId`, `chatKind`, `senderId`, `hasMention`, `mentionScopes`, `encrypted`.

A client receiving a `ChatMessageEntry` with no `bodySnippet` SHOULD display a generic
"new message" notification and fetch full content via `Message/changes` on next
foreground wake — the same behavior as when `encrypted: true`.

If the payload still exceeds the limit after all droppable fields are removed from every
entry (typically due to a large batch), servers SHOULD drop entries from the end of the
`messages` array, keeping the first entry. If a single minimal entry still exceeds the
platform limit, servers MUST fall back to a plain `StateChange` for that subscription
rather than sending an invalid or oversized payload.

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
