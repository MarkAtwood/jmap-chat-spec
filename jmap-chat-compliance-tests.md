# JMAP Chat Compliance Test Suite

A structured checklist and test harness specification for verifying any JMAP Chat implementation against the spec suite. Derived from 20 audit rounds against a reference implementation that found and fixed ~190 gaps.

Each test is tagged with the spec, section, severity (MUST/SHOULD/minor), and the round in which the gap was first discovered.

---

## How to use this document

**As a review checklist:** Walk through each section and verify the behavior against your implementation. Each test describes the expected behavior and the most common failure mode.

**As an automated test suite:** Each test specifies exact JMAP requests and expected responses. Implement them as integration tests against a running server.

**As an audit prompt:** Feed individual sections to an AI auditor with your source code to check specific areas.

---

## 1. Data Types — Field Names and Types

### 1.1 ChatContact identity model
**Spec:** draft-atwood-jmap-chat-00 Section 4.7, lines 399-402
**Severity:** MUST | **Round:** 1

> ChatContact.id MUST be the userId from the authentication layer. Servers MUST NOT assign a different value.

**Test:** Call `ChatContact/get` with `ids: null`. Verify that each returned object's `id` field equals the userId of the contact (not a server-generated ULID or other surrogate).

**Common failure:** Implementation uses a separate server-generated id and exposes the real userId as a different field (e.g., `contactUserId`).

### 1.2 ChatContact required fields
**Spec:** draft-atwood-jmap-chat-00 Section 4.7
**Severity:** MUST | **Round:** 1

> ChatContact MUST include: id, login, displayName, blocked, firstSeenAt, lastSeenAt, presence, lastActiveAt, statusText, statusEmoji, endpoints.

**Test:** Call `ChatContact/get` with `ids: null`. Verify every returned object contains all listed fields (some may be null).

**Common failure:** Missing `login`, `firstSeenAt`, `lastSeenAt`, `presence`, `lastActiveAt`, `statusText`, `statusEmoji`, `endpoints`.

### 1.3 ChatContact federation fields
**Spec:** draft-atwood-jmap-chat-federation-00 Section 5
**Severity:** MUST (serverUrl), SHOULD (directAddress) | **Round:** 1

**Test:** After a federated message delivery, call `ChatContact/get` for the sender. Verify `serverUrl` is populated with the peer's base URL. Verify `directAddress` is populated if the peer advertised `ownerDirectAddress`.

### 1.4 Message.deletedAt is UTCDate, not Boolean
**Spec:** draft-atwood-jmap-chat-00 Section 4.11, line 649
**Severity:** MUST | **Round:** 1

**Test:** Delete a message via `Message/set` destroy. Call `Message/get` for that id. Verify `deletedAt` is a UTCDate string (not a boolean `true`), and `body` is empty string.

**Common failure:** Field named `deleted` with boolean type instead of `deletedAt` with UTCDate.

### 1.5 ReadPosition has server-assigned id
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, RFC 8620 Section 5.1
**Severity:** MUST | **Round:** 1

**Test:** Set a read position via `ReadPosition/set`. Call `ReadPosition/get` with `ids: null`. Verify each object has an `id` field (server-assigned, distinct from chatId).

**Common failure:** ReadPosition has no `id` field; uses chatId as identity.

### 1.6 ReadPosition field name is lastReadAt
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 874
**Severity:** MUST | **Round:** 1

**Test:** Call `ReadPosition/get`. Verify the timestamp field is named `lastReadAt` (not `updatedAt`).

### 1.7 ChatMember.invitedBy field
**Spec:** draft-atwood-jmap-chat-00 Section 4.8, lines 451-452
**Severity:** MUST | **Round:** 1

**Test:** Create a group chat with `Chat/set`, then call `Chat/addMembers` (or patch key). Call `Chat/get` and inspect member objects. Verify `invitedBy` is set to the adder's id for added members, null for the creator.

### 1.8 Message.receivedAt is required
**Spec:** draft-atwood-jmap-chat-00 Section 4.11, lines 615-616
**Severity:** MUST | **Round:** 1

**Test:** Create a message. Call `Message/get`. Verify `receivedAt` is always present (not null/absent).

### 1.9 ChatMember role values: admin|member only
**Spec:** draft-atwood-jmap-chat-00 Section 4.8, line 446
**Severity:** MUST | **Round:** 18

**Test:** Via `Peer/groupUpdate` create, send a member with `role: "owner"`. Verify the server either rejects it or normalizes to `"member"`. The only valid values are `"admin"` and `"member"`.

**Common failure:** Implementation defines "owner" and "guest" roles that don't exist in the spec.

### 1.10 ReadPosition.lastReadMessageId is optional
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 871
**Severity:** MUST | **Round:** 3

**Test:** Verify a ReadPosition can exist with `lastReadMessageId: null` (unread chat).

---

## 2. Wire Format

### 2.1 Message body and bodyType are separate top-level fields
**Spec:** draft-atwood-jmap-chat-00 Section 4.11
**Severity:** MUST | **Round:** 3

**Test:** Call `Message/set` create with:
```json
{
  "chatId": "...",
  "body": "Hello world",
  "bodyType": "text/plain"
}
```
Verify this succeeds. If the server expects `body` to be a nested object `{"content": "...", "mediaType": "..."}`, it violates the spec.

### 2.2 senderId is "self" for owner-composed messages
**Spec:** draft-atwood-jmap-chat-00 Section 4.11, line 577
**Severity:** MUST | **Round:** 8

**Test:** Create a message. Call `Message/get` as the sender. Verify `senderId` is the string `"self"`, not the account's actual id.

**Common failure:** Returns the raw account id instead of "self".

### 2.3 Chat/set create direct uses contactId
**Spec:** draft-atwood-jmap-chat-00 Section 4.10, line 957
**Severity:** MUST | **Round:** 3

**Test:** Create a direct chat:
```json
{"kind": "direct", "contactId": "<other-user-id>"}
```
Verify this works. If the server expects `memberIds` for direct chats, it violates the spec.

### 2.4 Chat/set create duplicate direct returns alreadyExists
**Spec:** draft-atwood-jmap-chat-00 Section 4.10
**Severity:** MUST | **Round:** 3

**Test:** Create a direct chat with contactId X. Try to create another with the same contactId. Verify the error is `alreadyExists` (not `invalidProperties`) and includes `existingId`.

### 2.5 chatName absent for direct chats in push
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.2
**Severity:** MUST | **Round:** 3

**Test:** Generate a push payload for a direct chat message. Verify `chatName` key is absent from the JSON (not present as `null`).

---

## 3. Session and Capabilities

### 3.1 Session-level chat capability is empty object
**Spec:** draft-atwood-jmap-chat-00 line 197
**Severity:** MUST | **Round:** 15

**Test:** Fetch `/.well-known/jmap`. Verify `capabilities["urn:ietf:params:jmap:chat"]` is `{}`. Chat properties (maxBodyBytes, supportedBodyTypes, etc.) belong ONLY in account-level `accountCapabilities`.

**Common failure:** Session-level capability contains the properties that should only be at account level.

### 3.2 Session-level push capability is empty object
**Spec:** draft-atwood-jmap-chat-push-00 Section 2
**Severity:** MUST | **Round:** 3

**Test:** Verify `capabilities["urn:ietf:params:jmap:chat:push"]` at session level is `{}`. Push properties (maxSnippetBytes, supportedUrgencyValues, maxMessagesPerPush) belong at account level.

### 3.3 Account-level push capability populated
**Spec:** draft-atwood-jmap-chat-push-00 Section 3.2
**Severity:** MUST | **Round:** 3

**Test:** Verify `accountCapabilities["urn:ietf:params:jmap:chat:push"]` contains `maxSnippetBytes` (UnsignedInt), `supportedUrgencyValues` (String[]), and optionally `maxMessagesPerPush`.

### 3.4 RFC 8887 WebSocket capability present
**Spec:** draft-atwood-jmap-chat-wss-00 Section 3
**Severity:** MUST | **Round:** 3

**Test:** Verify `capabilities["urn:ietf:params:jmap:websocket"]` exists with `url` (String) and `supportsPush` (Boolean). This is SEPARATE from `urn:ietf:params:jmap:chat:websocket` (which must be `{}`).

### 3.5 Account-level role field
**Spec:** draft-atwood-jmap-chat-federation-00 Section 4
**Severity:** MUST | **Round:** 2

**Test:** Verify `accountCapabilities["urn:ietf:params:jmap:chat"]` contains `role` field. Value is `"owner"` for normal accounts, `"peer"` for peer-provisioned accounts.

### 3.6 Chat capability field names
**Spec:** draft-atwood-jmap-chat-00 Section capability
**Severity:** MUST | **Round:** 3

**Test:** Verify account-level chat capability uses exact field names: `maxBodyBytes` (not `maxMessageBodySize`), `supportedBodyTypes` (not `supportedMediaTypes`), `maxAttachmentBytes`, `maxAttachmentsPerMessage`, `supportsThreads`.

### 3.7 Push capability in primaryAccounts
**Spec:** draft-atwood-jmap-chat-push-00 Section 2
**Severity:** SHOULD | **Round:** 4

**Test:** Verify `primaryAccounts` contains `urn:ietf:params:jmap:chat:push` mapped to the primary account id.

---

## 4. Validation

### 4.1 maxBodyBytes enforcement
**Spec:** draft-atwood-jmap-chat-00 capability
**Severity:** MUST | **Round:** 3

**Test:** Send `Message/set` create with body exceeding `maxBodyBytes` (default 100,000). Verify server returns `tooLarge` SetError.

### 4.2 bodyType validated against supportedBodyTypes
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Send `Message/set` create with `bodyType: "video/mp4"`. Verify server rejects with `invalidProperties`.

### 4.3 Attachment filename path traversal rejection
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Send `Message/set` create with attachment `filename: "../../../etc/passwd"`. Verify rejection.

### 4.4 BroadcastMention scope validation
**Spec:** draft-atwood-jmap-chat-00 Section 4.5
**Severity:** MUST | **Round:** 3

**Test:** Send `Message/set` create with `broadcastMentions: [{"scope": "invalid"}]`. Verify rejection.

### 4.5 Mention offset+length bounds check
**Spec:** draft-atwood-jmap-chat-00 Section 4.4
**Severity:** MUST | **Round:** 3

**Test:** Send a 5-byte body with `mentions: [{"id": "u1", "offset": 10, "length": 5}]`. Verify rejection (offset+length > body bytes).

### 4.6 Rich body mentions/broadcastMentions must be empty
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 1390
**Severity:** MUST | **Round:** 8

**Test:** Send `Message/set` create with `bodyType: "application/jmap-chat-rich"` and non-empty `mentions` array. Verify rejection.

**Test:** Same for `Message/set` update and `Peer/deliver` edit paths. Verify all three paths reject.

**Common failure:** Validation only on create, missing on update and federation edit (Round 9).

### 4.7 Broadcast mention permission check
**Spec:** draft-atwood-jmap-chat-00 Section 4.5, lines 350-351
**Severity:** MUST | **Round:** 2

**Test:** In a channel chat, send a message with broadcastMentions from a user lacking `mention_broadcast` permission. Verify `forbidden` error.

### 4.8 Channel send permission check
**Spec:** draft-atwood-jmap-chat-00 Section 5.3
**Severity:** MUST | **Round:** 3

**Test:** In a channel chat, send a message from a user lacking `send` permission. Verify rejection.

### 4.9 Channel view permission check
**Spec:** draft-atwood-jmap-chat-00 Section 5.3
**Severity:** MUST | **Round:** 5

**Test:** Call `Message/get` for messages in a channel where the caller lacks `view` permission. Verify messages are not returned (treated as notFound).

### 4.10 Slow mode integration in Message/set
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Set slowModeSeconds on a channel. Send two messages in rapid succession from a non-admin. Verify the second is rejected with `rateLimited`.

### 4.11 CustomEmoji name uniqueness within scope
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 791
**Severity:** MUST | **Round:** 17

**Test:** Create two CustomEmojis with the same `name` in the same Space. Verify the second returns `alreadyExists`.

### 4.12 CustomEmoji name regex
**Spec:** draft-atwood-jmap-chat-00 Section 4.16
**Severity:** MUST | **Round:** 3

**Test:** Create CustomEmoji with `name: "invalid name!"`. Verify rejection.

### 4.13 Attachment contentType and sha256 validation
**Spec:** draft-atwood-jmap-chat-00
**Severity:** SHOULD | **Round:** 3

**Test:** Send attachment with `contentType: "not a mime type"`. Verify rejection. Send `sha256: "xyz"` (not 64 hex chars). Verify rejection.

---

## 5. Permissions and Access Control

### 5.1 Space/set destroy uses manage_space permission
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** A user with `manage_space` permission (but not role="owner") destroys a space. Verify success. A user without the permission is rejected.

**Common failure:** Implementation checks role name instead of resolved permission.

### 5.2 validate_permissions ignores unknown perms
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Create a SpaceRole with `permissions: ["send", "view", "future_perm_xyz"]`. Verify no error — unknown permissions are silently filtered.

**Common failure:** Implementation raises an error for unrecognized permission strings.

### 5.3 Peer/* methods role-gated on /jmap endpoint
**Spec:** draft-atwood-jmap-chat-federation-00 Section 4
**Severity:** MUST | **Round:** 4

**Test:** Authenticate as a normal (owner) account. Call `Peer/deliver` via `/jmap`. Verify `forbidden` error. Peer methods must only be accessible to peer-role accounts.

### 5.4 Chat/set channel destroy forbidden
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Try to destroy a channel chat via `Chat/set` destroy. Verify `forbidden` — channels must be destroyed via `Space/set`.

### 5.5 Chat/set update: patch keys for member operations
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Call `Chat/set` update with `addMembers`, `removeMembers`, `updateMemberRoles` as patch keys. Verify they work.

---

## 6. Push Notifications

### 6.1 ChatPushConfig field names
**Spec:** draft-atwood-jmap-chat-push-00 Section 3.1
**Severity:** MUST | **Round:** 1

**Test:** Create a PushSubscription with `chatPush` containing `kinds` (not `chatKindFilter`), `chatIds` (not `chatIdFilter`), `urgency`, `properties`. Verify acceptance.

### 6.2 ChatMessagePush.accountId required
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.1
**Severity:** MUST | **Round:** 1

**Test:** Generate a push payload. Verify it contains `accountId` at top level.

### 6.3 ChatMessageEntry required fields
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.2
**Severity:** MUST | **Round:** 1

**Test:** Generate a push payload. Verify each entry contains `chatKind`, `sentAt`, `senderDisplayName`. For channel chats, verify `spaceId` and `spaceName` are present.

### 6.4 Properties filtering
**Spec:** draft-atwood-jmap-chat-push-00 Section 3.1
**Severity:** MUST | **Round:** 3

**Test:** Set `properties: ["messageId", "chatId"]` in ChatPushConfig. Generate push. Verify only those fields appear in entries (plus conditional fields like bodySnippet when not encrypted).

### 6.5 Urgency resolution uses base urgency
**Spec:** draft-atwood-jmap-chat-push-00 Section 6.1
**Severity:** MUST | **Round:** 1

**Test:** Set `urgency: "low"` in config. Send a message without mentions. Verify push urgency is `"low"` (not hardcoded `"normal"`).

### 6.6 Broadcast mention mute bypass includes admins
**Spec:** draft-atwood-jmap-chat-push-00
**Severity:** MUST | **Round:** 7

**Test:** Mute a chat. Send a message with `broadcastMentions: [{"scope": "admins"}]` where the recipient is an admin. Verify push is NOT suppressed.

**Common failure:** Only "everyone" and "here" bypass mute; "admins" incorrectly suppressed.

### 6.7 Rich body inline span mention detection
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.2
**Severity:** MUST | **Round:** 12

**Test:** Send a message with `bodyType: "application/jmap-chat-rich"` containing inline `{"type": "mention", "userId": "<owner-id>"}` span. Verify `hasMention: true` in push.

**Common failure:** Only checks `Message.mentions` array (which is mandated empty for rich bodies).

### 6.8 senderDisplayName resolution chain
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.2
**Severity:** MUST | **Round:** 1

**Test:** For a channel chat, verify resolution order: SpaceMember.nick -> ChatContact.displayName -> login -> senderId. For direct/group: ChatContact.displayName -> login -> senderId.

### 6.9 muteUntil expiry check
**Spec:** draft-atwood-jmap-chat-push-00
**Severity:** MUST | **Round:** 3

**Test:** Set `muteUntil` to a time in the past. Send a message. Verify push is NOT suppressed (mute has lapsed).

### 6.10 PushSubscription/set validates urgency
**Spec:** draft-atwood-jmap-chat-push-00 Section 3.1
**Severity:** MUST | **Round:** 3

**Test:** Call `PushSubscription/set` create with `chatPush.urgency: "invalid"`. Verify `invalidArguments` error (not `invalidProperties`).

---

## 7. Federation

### 7.1 Peer/retract sets deletedForAll
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.4, line 451
**Severity:** MUST | **Round:** 2

**Test:** Call `Peer/retract`. Verify the tombstoned message has `deletedForAll: true` (not just `deletedAt`).

### 7.2 Peer/deliver fires state-change events
**Spec:** draft-atwood-jmap-chat-federation-00
**Severity:** MUST | **Round:** 3

**Test:** Deliver a message via `Peer/deliver`. Verify a state change is recorded for `Message` data type so EventSource/WebSocket subscribers get notified.

### 7.3 Peer/deliver SpaceBan returns String receivedMsgId
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.1.1
**Severity:** MUST | **Round:** 19

**Test:** Deliver a message from a banned sender. Verify `receivedMsgId` in the response is a String (ULID), not `null`.

**Common failure:** Returns `null` for silently dropped messages.

### 7.4 Peer/deliver edit validates steps 3-7
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.1.3
**Severity:** MUST | **Round:** 3

**Test:** Send a `Peer/deliver` edit from a blocked sender. Verify silent drop. Send edit to a channel where sender lacks `send` permission. Verify rejection.

### 7.5 Peer/deliver stores mention offset/length
**Spec:** draft-atwood-jmap-chat-00 Section 4.4
**Severity:** MUST | **Round:** 11

**Test:** Deliver a message via `Peer/deliver` with `mentions: [{"id": "u1", "offset": 5, "length": 3}]`. Query the DB. Verify offset and length are stored (not just the user id).

### 7.6 Peer/receipt preserves existing fields
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.2
**Severity:** MUST | **Round:** 13

**Test:** Call `Peer/receipt` with only `deliveredAt`. Then call again with only `readAt`. Verify both values are preserved in the delivery receipt (the second call does NOT null out deliveredAt).

**Common failure:** INSERT OR REPLACE clobbers previously-stored fields.

### 7.7 deviceDeliveredAt suppressed when receiptSharing=false
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.2
**Severity:** MUST | **Round:** 10

**Test:** Set receiptSharing=false for a sender. Call `Peer/receipt` with `deliveredAt` and `deviceDeliveredAt`. Verify `deviceDeliveredAt` is NOT stored (it's privacy-sensitive).

**Common failure:** Only suppresses read_at when receiptSharing=false; forgets deviceDeliveredAt.

### 7.8 Peer auth rejects on discovery failure
**Spec:** draft-atwood-jmap-chat-federation-00
**Severity:** MUST | **Round:** 3

**Test:** Send a `Peer/deliver` where the peer's `.well-known/jmap` is unreachable. Verify 403 rejection (not silent pass-through).

### 7.9 Direct chat contactId auto-creates placeholder
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 16

**Test:** Create a direct chat with `contactId` of a user that has no local account (federated user). Verify success (placeholder account auto-created).

**Common failure:** Rejects with "account does not exist".

### 7.10 Peer rate limiter integrated
**Spec:** draft-atwood-jmap-chat-federation-00
**Severity:** MUST | **Round:** 3

**Test:** Send many `Peer/deliver` calls in rapid succession from the same peer. Verify `rateLimited` error after exceeding the limit.

---

## 8. WebSocket

### 8.1 ChatPresenceEvent null vs absent semantics
**Spec:** draft-atwood-jmap-chat-wss-00 Section 6.4
**Severity:** MUST | **Round:** 3

**Test:** Send a presence update with `statusText: null` (explicit clear) vs omitting `statusText` (no change). Verify the server emits the key with `null` value in the first case and omits the key in the second.

### 8.2 Typing membership re-verified at delivery time
**Spec:** draft-atwood-jmap-chat-wss-00 Section 6.3, Section 8.2
**Severity:** MUST | **Round:** 4

**Test:** User A starts typing. Before the event is delivered, remove user B from the chat. Verify B does NOT receive the typing event.

### 8.3 receiveTypingIndicators re-checked at delivery time
**Spec:** draft-atwood-jmap-chat-wss-00 Section 6.3, line 414
**Severity:** MUST | **Round:** 4

**Test:** Same pattern: a user has `receiveTypingIndicators: false` at the moment of delivery (not just at subscription time). Verify they don't receive typing events.

### 8.4 Block status re-checked at delivery for typing and presence
**Spec:** draft-atwood-jmap-chat-wss-00 Section 8.2
**Severity:** SHOULD | **Round:** 4

**Test:** Block a contact AFTER a typing event is dispatched but BEFORE it reaches the connection. Verify the blocked contact's typing event is dropped.

### 8.5 ChatPresenceEvent includes statusEmoji and lastActiveAt
**Spec:** draft-atwood-jmap-chat-wss-00 Section 6.4
**Severity:** MUST | **Round:** 2

**Test:** Update presence with statusEmoji and lastActiveAt. Verify both fields appear in the ChatPresenceEvent delivered via WebSocket.

---

## 9. Scheduling and Lifecycle

### 9.1 burnOnRead immediate hard-delete
**Spec:** draft-atwood-jmap-chat-00 Section 4.11
**Severity:** MUST | **Round:** 6

**Test:** Create a message with `burnOnRead: true`. Mark it as read (set `readAt`). Immediately query `Message/get` for that id. Verify it is GONE (hard-deleted, not tombstoned).

**Common failure:** Defers to periodic sweep instead of immediate deletion.

### 9.2 SpaceBan expiry auto-restore
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Create a SpaceBan with `expiresAt` in the past. Run the expiry sweep. Verify the ban is removed.

### 9.3 Presence expiresAt auto-clear
**Spec:** draft-atwood-jmap-chat-00
**Severity:** SHOULD | **Round:** 3

**Test:** Set presence with `expiresAt` in the past. Run the expiry sweep. Verify presence resets to "offline".

### 9.4 Chat muteUntil auto-clear
**Spec:** draft-atwood-jmap-chat-00
**Severity:** SHOULD | **Round:** 3

**Test:** Set `muteUntil` in the past. Run the expiry sweep. Verify `muted` is false and `muteUntil` is null.

---

## 10. Space Operations

### 10.1 Space/join accepts spaceId for public spaces
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Create a public space. Call `Space/join` with `spaceId` (no invite code). Verify success.

### 10.2 Space/get restricted view for non-members
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Call `Space/get` for a public space the caller is NOT a member of. Verify a restricted property set is returned (name, description, memberCount — but NOT roles or members).

### 10.3 Space/query includes public spaces
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Call `Space/query` with `filter: {isPublic: true}`. Verify public spaces the caller is NOT a member of are included.

### 10.4 SpaceBan/set update supports reason and expiresAt
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Update an existing ban's `reason` and `expiresAt`. Verify success (not `forbidden`).

### 10.5 CustomEmoji/set update supports name and blobId
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Update an existing emoji's `name` and `blobId`. Verify success.

---

## 11. Chat.unreadCount

### 11.1 unreadCount computed from ReadPosition
**Spec:** draft-atwood-jmap-chat-00
**Severity:** MUST | **Round:** 3

**Test:** Send 5 messages. Set ReadPosition to message 3. Call `Chat/get`. Verify `unreadCount` is 2 (messages 4 and 5).

---

## 12. PushSubscription

### 12.1 PushSubscription/get, /set, /changes exist
**Spec:** draft-atwood-jmap-chat-push-00 Section 3.1, RFC 8620 Section 7.2
**Severity:** MUST | **Round:** 2

**Test:** Call `PushSubscription/set` create with `url` and `chatPush`. Verify success. Call `/get` and `/changes`.

---

## Summary Statistics

| Category | MUST tests | SHOULD tests | Total |
|----------|:----------:|:------------:|:-----:|
| Data types | 10 | 0 | 10 |
| Wire format | 5 | 0 | 5 |
| Session/capabilities | 6 | 1 | 7 |
| Validation | 13 | 1 | 14 |
| Permissions | 5 | 0 | 5 |
| Push | 10 | 0 | 10 |
| Federation | 10 | 0 | 10 |
| WebSocket | 4 | 1 | 5 |
| Scheduling | 2 | 2 | 4 |
| Space | 5 | 0 | 5 |
| Other | 2 | 0 | 2 |
| **Total** | **72** | **5** | **77** |

---

## Appendix: Discovery Order

The tests are ordered by the round in which the gap was first discovered, which corresponds roughly to subtlety:

- **Rounds 1-2 (Easy):** Missing fields, wrong types, missing methods
- **Rounds 3-4 (Medium):** Missing validations, wrong capabilities, permission model errors
- **Rounds 5-8 (Hard):** Permission gating, wire format semantics, rich body interactions
- **Rounds 9-13 (Subtle):** Update path omissions, privacy field leaks, delivery receipt semantics
- **Rounds 14-19 (Edge cases):** Capability placement, type constraints on error responses, role enum values, uniqueness constraints, federation contact lifecycle
