# JMAP Chat Compliance Test Suite

A structured checklist and test harness specification for verifying any JMAP Chat implementation against the spec suite. Derived from 23 audit rounds plus 3 code reviews against a reference implementation that found and fixed ~260 gaps.

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

## 13. Late-Discovery Edge Cases (Rounds 15-22)

These tests catch subtle issues that survived 14+ audit rounds. They represent the deepest compliance gaps — the kind that only surface under adversarial review or cross-spec interaction analysis.

### 13.1 Session-level chat capability must be empty `{}`
**Spec:** draft-atwood-jmap-chat-00 line 197
**Severity:** MUST | **Round:** 15

**Test:** Fetch `/.well-known/jmap`. Verify `capabilities["urn:ietf:params:jmap:chat"]` is exactly `{}`. The chat properties (maxBodyBytes, supportedBodyTypes, etc.) must ONLY appear in per-account `accountCapabilities["urn:ietf:params:jmap:chat"]`.

**Common failure:** Copying the properties into both session-level and account-level. Session-level must be empty.

### 13.2 Direct chat contactId accepts remote/federated users
**Spec:** draft-atwood-jmap-chat-00 Section 4.10
**Severity:** MUST | **Round:** 16

**Test:** Call `Chat/set` create with `{"kind": "direct", "contactId": "remote.peer.example"}` where that userId has no local account. Verify success — the server should auto-create a placeholder account for the remote user.

**Common failure:** Server validates contactId against local accounts table and rejects with "account does not exist".

### 13.3 CustomEmoji name uniqueness within scope
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 791
**Severity:** MUST | **Round:** 17

**Test:** Create two CustomEmojis with identical `name` in the same Space. Verify the second returns `alreadyExists` error. Also test across scopes: same name in different Spaces should succeed.

**Common failure:** No UNIQUE constraint or application-level check on (name, space_id).

### 13.4 ChatMember role restricted to admin|member
**Spec:** draft-atwood-jmap-chat-00 Section 4.8, line 446
**Severity:** MUST | **Round:** 18

**Test:** Via `Peer/groupUpdate` create, send members with `role: "owner"` or `role: "guest"`. Verify the server either rejects or normalizes to "member". Only `"admin"` and `"member"` are valid.

**Test:** Verify the admin-verification check for group updates only accepts `role == "admin"` (not "owner").

**Common failure:** Implementation defines "owner"/"guest" roles from internal design that don't exist in the spec's wire protocol.

### 13.5 SpaceBan silent-drop response has String receivedMsgId
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.1.1, line 296
**Severity:** MUST | **Round:** 19

**Test:** Deliver a message from a space-banned sender via `Peer/deliver`. Verify the `receivedMsgId` in the response is a String (a ULID), not `null`.

**Common failure:** Returns `null` for silently-dropped messages, violating the String type constraint.

### 13.6 Peer/receipt preserves previously-stored fields
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.2
**Severity:** MUST | **Round:** 13 (rediscovered 19)

**Test:** Call `Peer/receipt` with only `deliveredAt`. Then call again with only `readAt` (no `deliveredAt`). Verify BOTH values are preserved in the delivery receipt — the second call must NOT null out the previously-stored `deliveredAt`.

**Common failure:** Using INSERT OR REPLACE which overwrites the entire row, nulling fields not provided in the current call.

### 13.7 deviceDeliveredAt suppressed unconditionally when receiptSharing=false
**Spec:** draft-atwood-jmap-chat-federation-00 Section 6.2, lines 354-362
**Severity:** MUST | **Round:** 10

**Test:** Set receiptSharing=false for a message sender. Call `Peer/receipt` with `deliveredAt` and `deviceDeliveredAt` (but NO `readAt`). Verify `deviceDeliveredAt` is NOT stored.

**Common failure:** Only suppressing `deviceDeliveredAt` when `readAt` is also present. The spec says: "The server MUST NOT record deviceDeliveredAt when the effective preference is false" — regardless of what other fields are present.

### 13.8 Rich body mention guard in update AND federation edit paths
**Spec:** draft-atwood-jmap-chat-00 Section 4.16, line 1390
**Severity:** MUST | **Round:** 9

**Test:** Send `Message/set` update with `bodyType: "application/jmap-chat-rich"` and non-empty `mentions`. Verify rejection. Send `Peer/deliver` edit with same. Verify rejection.

**Common failure:** Validation only on create path; update and federation edit paths allow mentions on rich bodies.

### 13.9 Rich body inline span parsing for push mentions
**Spec:** draft-atwood-jmap-chat-push-00 Section 4.2
**Severity:** MUST | **Round:** 12

**Test:** Send a message with `bodyType: "application/jmap-chat-rich"` and body containing `{"spans": [{"type": "mention", "userId": "<owner-id>"}]}`. Verify `hasMention: true` in the push notification.

**Test:** Same with `{"type": "broadcast", "scope": "everyone"}` span. Verify it appears in `mentionScopes`.

**Common failure:** Only checking `Message.mentions` array (mandated empty for rich bodies) and never parsing the body JSON for inline spans.

### 13.10 Peer/deliver mention offset/length must be stored
**Spec:** draft-atwood-jmap-chat-00 Section 4.4
**Severity:** MUST | **Round:** 11

**Test:** Deliver a message via `Peer/deliver` with `mentions: [{"id": "u1", "offset": 5, "length": 3}]`. Query the message back. Verify offset and length are preserved (not just the user id).

**Common failure:** INSERT INTO mentions only stores (message_id, account_id), dropping the offset/length columns.

### 13.11 Slow mode must use received_at, not sent_at
**Spec:** draft-atwood-jmap-chat-00 §4.11 lines 612-615, federation §6.1.2 step 6
**Severity:** MUST | **Round:** 21

**Test:** Send two messages via `Peer/deliver` to a channel with slowModeSeconds=60. The second message has `sentAt` backdated 120 seconds but arrives immediately. Verify the server rejects the second message (rate limited by actual arrival time, not the forged sentAt).

**Common failure:** Slow mode query uses `sent_at` column (untrusted, sender-supplied) instead of `received_at` (authoritative, server-assigned). Spec explicitly states: "sentAt MUST NOT be used for ordering."

---

## 14. Security and Robustness (Code Review Findings)

These are implementation-quality issues found via code review rather than spec audit. They don't violate spec text but represent security or correctness risks.

### 14.1 Peer authentication header bypass
**Severity:** P0 Security | **Round:** Code Review 1

**Test:** Send a `Peer/deliver` request without a TLS client certificate, but with an `X-Peer-Hostname: victim.example` header. Verify the server rejects with 403 (not proceeds as if authenticated).

**Common failure:** Accepting an HTTP header as peer identity without requiring it to be gated behind an explicit test-mode flag. Combined with CERT_OPTIONAL, this allows any HTTP client to impersonate any peer.

### 14.2 Transaction atomicity for multi-step writes
**Severity:** P1 Correctness | **Round:** Code Review 1

**Test:** During message creation (INSERT message + INSERT attachments + INSERT mentions + UPDATE chat.lastMessageAt + INSERT state_log), simulate a crash after the message INSERT but before the mentions INSERT. Verify the database is not left in an inconsistent state.

**Common failure:** Each `execute()` auto-commits. No transaction wrapping means partial writes survive crashes.

### 14.3 State counter atomicity
**Severity:** P1 Correctness | **Round:** Code Review 1

**Test:** Issue two concurrent `record_change()` calls for the same (account_id, data_type). Verify they produce distinct, monotonically increasing state tokens (no duplicates).

**Common failure:** Read-increment-write pattern across multiple awaits allows interleaving that produces duplicate tokens.

### 14.4 LIKE wildcard injection in Chat/query
**Severity:** P2 | **Round:** Code Review 2

**Test:** Call `Chat/query` with `filter: {name: "%"}`. Verify it does NOT return all chats — the `%` should be treated as a literal character, not a wildcard.

**Common failure:** Passing user-supplied filter value directly into a LIKE clause without escaping `%` and `_`.

### 14.5 Nested transaction safety
**Severity:** P1 | **Round:** Code Review 2

**Test:** If your implementation has a transaction context manager, verify that nested calls do not prematurely commit. The outer transaction must own the commit; inner calls should be no-ops.

**Common failure:** Boolean `_in_transaction` flag set to False on any exit, regardless of nesting depth.

### 14.6 Channel with NULL space_id bypasses enforcement
**Severity:** P1 | **Round:** Code Review 2

**Test:** Create a channel chat with `space_id = NULL` in the database. Send a message to it via `Peer/deliver`. Verify SpaceBan checks, send-permission checks, and slow-mode enforcement still run (or the server rejects the channel as invalid).

**Common failure:** All Space-related checks gated behind `if space_id:`, silently skipping enforcement for malformed channels.

### 14.7 burn_on_read column missing from base schema
**Severity:** P0 | **Round:** Code Review 3

**Test:** On a fresh database (no V2 migration applied), mark a message as read. Verify the server does NOT crash with a "no such column: burn_on_read" error.

**Common failure:** The `burn_on_read` column is added by a secondary migration but the code path that checks it after setting `readAt` has no column-existence guard.

### 14.8 Peer/groupUpdate partial write on validation failure
**Severity:** P1 | **Round:** Code Review 3

**Test:** Send `Peer/groupUpdate` create with 3 members: two valid, one with an invalid ULID-format ID that doesn't exist locally. Verify the chat row is NOT left in the database with only the first two members.

**Common failure:** The chat INSERT commits before member validation completes. A validation error mid-loop leaves an orphaned chat row with partial membership.

### 14.9 Permission check logic divergence between modules
**Severity:** P2 | **Round:** Code Review 3

**Test:** Store `role_ids` as a JSON array `["role-a", "role-b"]` (not CSV). Call a channel operation that checks permissions. Verify it works. Then call a different operation (e.g., create_channel vs. Message/set broadcast mention check). Verify both give the same result.

**Common failure:** Multiple copies of "check space permission" logic diverge over time. One handles JSON arrays, the other only handles CSV. The CSV-only version silently returns "no permission" for JSON-encoded role_ids.

### 14.10 Migration error suppression too broad
**Severity:** P1 | **Round:** Code Review 1

**Test:** Introduce a real schema error (e.g., invalid SQL in a migration). Run the server. Verify the error is surfaced, not silently swallowed.

**Common failure:** `except Exception: pass` around migration statements masks disk-full errors, constraint violations, and typos — then bumps the schema version anyway, declaring the broken migration "complete."

### 14.11 Password hash timing oracle
**Severity:** P1 | **Round:** Code Review 1

**Test:** Time authentication attempts for: (a) a non-existent username, (b) an existing username with wrong password. Verify both take approximately the same time (within 10% variance).

**Common failure:** Non-existent user path runs a different code branch (e.g., dummy hash with a pre-generated salt) that takes measurably different time than the real hash-then-compare path.

### 14.12 scrypt work factor below OWASP minimum
**Severity:** P1 | **Round:** Code Review 1

**Test:** Check the scrypt N parameter. Verify N >= 2^17 (131072). Lower values (2^14 = 16384) make offline dictionary attacks against a stolen DB 8x cheaper.

**Common failure:** Using N=16384 which was adequate in 2015 but is below the 2024 OWASP minimum of 2^17.

### 14.13 Variable-time comparison for blob SHA256 verification
**Severity:** P2 | **Round:** Code Review 1

**Test:** Verify that blob hash comparison uses constant-time comparison (e.g., `hmac.compare_digest`), not `!=` or `==`.

**Common failure:** `if actual_sha256 != expected_sha256:` leaks information about how many bytes match via timing.

### 14.14 Content-Disposition header injection via filename
**Severity:** P2 | **Round:** Code Review 1

**Test:** Upload a blob with filename `"test.txt` (contains a double-quote). Download it. Verify the `Content-Disposition` header is correctly encoded (RFC 6266) and doesn't produce a malformed HTTP header.

**Common failure:** Filename interpolated directly into `attachment; filename="{name}"` without escaping quotes or non-ASCII.

### 14.15 Bare except swallowing real DB errors
**Severity:** P1 | **Round:** Code Review 2

**Test:** On a database with a V2-schema column present, introduce a constraint violation (e.g., NOT NULL on a column). Attempt the operation. Verify you get an error, not silent success via the fallback path.

**Common failure:** `except Exception:` around V2-column INSERT/UPDATE catches ALL errors (disk full, type mismatch, constraint violation), not just the intended "missing column" case. Should be `except sqlite3.OperationalError:` with message check.

### 14.16 update_message not atomic
**Severity:** P1 | **Round:** Code Review 2

**Test:** Simulate a crash after updating message body but before re-inserting mentions. Verify the database is consistent (either both body and mentions are updated, or neither is).

**Common failure:** Message UPDATE, mention DELETE, and mention INSERT are separate auto-committed statements with no transaction wrapper.

### 14.17 Per-request session creation in federation outbox
**Severity:** P2 | **Round:** Code Review 1

**Test:** Under load, verify the server does not create hundreds of TCP connections to the same peer. Each outbound delivery should reuse a connection pool.

**Common failure:** `aiohttp.ClientSession()` created per-request instead of once at startup, bypassing HTTP connection pooling.

---

## Summary Statistics

| Category | MUST tests | SHOULD tests | Total |
|----------|:----------:|:------------:|:-----:|
| Data types | 10 | 0 | 10 |
| Wire format | 5 | 0 | 5 |
| Session/capabilities | 7 | 1 | 8 |
| Validation | 13 | 1 | 14 |
| Permissions | 5 | 0 | 5 |
| Push | 12 | 0 | 12 |
| Federation | 15 | 0 | 15 |
| WebSocket | 4 | 1 | 5 |
| Scheduling | 2 | 2 | 4 |
| Space | 6 | 0 | 6 |
| Late-discovery edge cases | 11 | 0 | 11 |
| Security/robustness | 17 | 0 | 17 |
| Other | 2 | 0 | 2 |
| **Total** | **104** | **5** | **109** |

---

## Appendix: Discovery Order

The tests are ordered by the round in which the gap was first discovered, which corresponds roughly to subtlety:

- **Rounds 1-2 (Easy):** Missing fields, wrong types, missing methods
- **Rounds 3-4 (Medium):** Missing validations, wrong capabilities, permission model errors
- **Rounds 5-8 (Hard):** Permission gating, wire format semantics, rich body interactions
- **Rounds 9-13 (Subtle):** Update path omissions, privacy field leaks, delivery receipt semantics
- **Rounds 14-19 (Edge cases):** Capability placement, type constraints on error responses, role enum values, uniqueness constraints, federation contact lifecycle
- **Rounds 20-22 (Final):** Untrusted timestamp usage in rate limiting, session/account capability placement
- **Code Reviews (Security):** Auth bypass chains, transaction atomicity, state counter races, wildcard injection, nesting safety, burn_on_read schema guard, partial write atomicity, permission logic divergence, migration error masking, timing oracles, scrypt params, constant-time comparison, header injection, bare except scope
