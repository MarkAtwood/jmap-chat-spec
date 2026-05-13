---
title: JMAP for Chat
abbrev: JMAP Chat
docname: draft-atwood-jmap-chat-00
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
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

informative:
  RFC7763:
  RFC8621:
  RFC9404:
  RFC9420:
  RFC9425:
  MIMI-CONTENT:
    title: More Instant Messaging Interoperability (MIMI) Content Format
    target: https://datatracker.ietf.org/doc/draft-ietf-mimi-content/
  MIMI-PROTOCOL:
    title: More Instant Messaging Interoperability (MIMI) Protocol
    target: https://datatracker.ietf.org/doc/draft-ietf-mimi-protocol/
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  JMAP-CHAT-WSS:
    title: JMAP Chat over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-OBJ-HISTORY:
    title: JMAP Object History
    target: https://datatracker.ietf.org/doc/draft-gondwana-jmap-object-history/
  RFC9610:
  RFC9670:
  JMAP-FILENODE:
    title: JMAP Filenode
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/
  JMAP-WEBPUSH-VAPID:
    title: JMAP WebPush VAPID
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-webpush-vapid/
  JMAP-BLOBEXT:
    title: JMAP Blob Management
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-blobext/
  JMAP-CID:
    title: JMAP Blob Content Identifiers
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-cid-00
    date: 2026
  JMAP-METADATA:
    title: JMAP Metadata
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-metadata/
  JMAP-REFPLUS:
    title: JMAP Result References Plus
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-refplus/
  JMAP-CALENDARS:
    title: JMAP for Calendars
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-calendars/
  JMAP-TASKS:
    title: JMAP for Tasks
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-tasks/
  JMAP-CHAT-CALENDARS:
    title: JMAP Chat Calendars
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-calendars-00
    date: 2026
  JMAP-CHAT-TASKS:
    title: JMAP Chat Tasks
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-tasks-00
    date: 2026
  W3C-DID-CORE:
    title: Decentralized Identifiers (DIDs) v1.0
    target: https://www.w3.org/TR/did-core/

--- abstract

This document defines JMAP for Chat, a JMAP capability ({{RFC8620}}) for direct and group text messaging. It supports two deployment topologies: a mailbox-per-user model in which each user operates their own server, and a relay model in which a shared server routes end-to-end encrypted messages.

The specification defines the `urn:ietf:params:jmap:chat` capability; twenty-one data types (Attachment, Endpoint, MessageAction, Mention, BroadcastMention, MessageRevision, Reaction, ChatContact, ChatMember, Chat, Message, SpaceRole, SpaceMember, Category, ChannelPermission, Space, SpaceInvite, CustomEmoji, SpaceBan, ReadPosition, and PresenceStatus); and JMAP owner methods for each top-level type. Server-to-server federation methods are defined in {{JMAP-CHAT-FED}}.

The protocol covers the feature set common to contemporary messaging systems: group chat with membership roles, message reactions, editing, deletion, threading, @mentions, typing indicators, read receipts per participant, presence, pinned messages, per-chat notification settings, sender-controlled message expiry, and burn-on-read. Spaces provide a named multi-channel container with a role-based permission system, analogous to what other systems call a server, workspace, or team.


--- middle

# Introduction

JMAP {{RFC8620}} defines a JSON-based protocol for accessing and mutating application data. The core protocol is intentionally generic; application semantics are expressed through capability URIs declared in the JMAP Session object. {{RFC8621}} defines JMAP for Mail. This document defines an analogous capability for real-time chat.

## Deployment Topologies

This specification accommodates two primary deployment topologies.

In the **mailbox-per-user** model, each participant runs their own JMAP server storing only their own messages. Mailboxes exchange messages directly using the server-to-server methods defined in {{JMAP-CHAT-FED}}.

In the **relay** model, a shared server routes messages between clients. The Peer/* methods are implemented by the relay rather than by individual user-controlled mailboxes. Relay deployments are designed to handle only opaque ciphertext; see {{e2ee}} for the normative requirements.

Both topologies are fully compatible with this specification. Transport, identity, and encryption choices are confined to the deployment layer.

## Authentication Model

Authentication is handled entirely at the transport layer. The protocol requires only that the authentication layer provide a stable, opaque user identity string for each connection. How that identity is established — overlay network membership, mutual TLS, bearer tokens, or any other mechanism — is outside the scope of this document. Authorization rules derived from this identity are defined in {{authorization}}.

In the mailbox-per-user deployment topology, the peer authentication model is defined in {{JMAP-CHAT-FED}}.

## Relationship to MIMI

The IETF MIMI (More Instant Messaging Interoperability) working group {{MIMI-PROTOCOL}} {{MIMI-CONTENT}} is developing a separate approach to messaging interoperability, primarily targeting provider-to-provider federation between large existing messaging platforms under regulatory interoperability mandates. MIMI's client-server API layer is intentionally outside the MIMI charter scope; this document fills that gap for JMAP-based deployments.

The two specifications are complementary rather than competing. MIMI defines `application/mimi-content` {{MIMI-CONTENT}}, a CBOR-encoded message body format designed to operate as an MLS PrivateMessage payload. A JMAP Chat server that also participates in a MIMI federation domain MAY include `"application/mimi-content"` in its `supportedBodyTypes` capability and accept MIMI-formatted message bodies in `Message/set` and `Peer/deliver`. This document does not require or preclude such interoperability.

The federation protocol defined in {{JMAP-CHAT-FED}} uses a mailbox-per-user architecture distinct from MIMI's hub-and-spoke room ownership model; see {{JMAP-CHAT-FED}} for discussion of that distinction.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout.

The following terms are specific to this document:

Mailbox:
: A JMAP server instance serving exactly one user.

Owner:
: The user whose data a mailbox stores and serves.

Peer:
: Another mailbox server communicating with this mailbox.

id / userId:
: A ChatContact's `id` is the stable, opaque identity string provided by the authentication layer for that user. These two terms are intentionally equivalent in this protocol: ChatContact.id IS the userId. There is no separate identity namespace. Servers MUST set ChatContact.id to the userId string obtained from the authentication layer. The specific form of the identifier (for example, a `user@host`-style string, a Decentralized Identifier URI {{?W3C-DID-CORE}}, or any other URI form) is not constrained by this specification; servers MUST treat it as opaque regardless of form.

# The urn:ietf:params:jmap:chat Capability {#capability}

The `urn:ietf:params:jmap:chat` capability is advertised in the JMAP Session object at both the top-level `capabilities` key and within each account's `accountCapabilities` map.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat"]` at the session level is an empty object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat"]` is a JSON object with the following fields:

`maxBodyBytes` (UnsignedInt):
: Maximum UTF-8 byte length of a Message `body`. Servers MUST reject messages exceeding this limit with `invalidArguments`.

`maxAttachmentBytes` (UnsignedInt):
: Maximum size in bytes of a single attachment blob.

`maxAttachmentsPerMessage` (UnsignedInt):
: Maximum number of attachments per message.

`supportedBodyTypes` (String[]):
: MIME types accepted in `bodyType`. MUST include `"text/plain"`. End-to-end encrypted deployments SHOULD also include an appropriate encrypted-content type such as `"application/mls-ciphertext"`. Servers SHOULD support `"text/markdown"` (CommonMark profile, as specified in {{RFC7763}}) and `"application/jmap-chat-rich"` (defined in {{rich-body}}). Servers participating in MIMI federation domains MAY also include `"application/mimi-content"` {{MIMI-CONTENT}}.

`supportsThreads` (Boolean):
: Whether this server supports the optional thread model defined in {{threads}}.

The `role` field used to distinguish owner and peer accounts in federation deployments is defined in {{JMAP-CHAT-FED}}.

Note: {{RFC9425}} defines a `Quota` data type for exposing account-level storage and object-count limits to clients. Servers SHOULD implement the JMAP Quota extension and include relevant chat data types — for example `"Message"` (storage), `"Chat"` (count of group and direct chats), or `"Space"` (count of Spaces) — in the `types` field of relevant Quota records. JMAP Quotas express account-level (or domain/global) limits; servers additionally enforce per-aggregate limits (members per group Chat, members/roles/channels/categories per Space) by returning an `overQuota` SetError ({{RFC8620}} §5.3) on the relevant `Chat/set` or `Space/set` entry at the time the limit would be exceeded.

## Session Object Extensions

When the `urn:ietf:params:jmap:chat` capability is present, servers MAY include the following additional fields in the Session object:

`ownerUserId` (String):
: The id of the mailbox owner (equals the owner's ChatContact.id on any peer server that has recorded this mailbox as a contact). Servers implementing {{JMAP-CHAT-FED}} MUST include this field; other servers MAY include it.

`ownerLogin` (String):
: A human-readable login name for the mailbox owner.

`ownerEndpoints` (Endpoint[], optional):
: The owner's advertised out-of-band capability endpoints (see {{endpoint}}). Peers that probe `/.well-known/jmap` SHOULD merge these into the `endpoints` field of the corresponding ChatContact record.

The `ownerDirectAddress` field used in federation deployments is defined in {{JMAP-CHAT-FED}}.

# Data Types

Data types are defined in dependency order: embedded sub-types precede the types that contain them.

## Attachment {#attachment}

An Attachment carries metadata for a file blob associated with a Message.

`blobId` (String):
: Opaque server-assigned blob identifier.

`filename` (String):
: Original filename. MUST NOT contain `/`, `\`, or null bytes.

`contentType` (String):
: Valid MIME type string.

`size` (UnsignedInt):
: Blob size in bytes. Servers MUST verify against actual content.

`sha256` (String):
: Lowercase hex SHA-256 of blob content. Servers SHOULD verify.

## Endpoint {#endpoint}

An Endpoint advertises an out-of-band capability reachable at a URI. Endpoints appear on ChatContact records and in Session objects as persistent capability advertisements. The `type` field uses an extensible URI namespace; clients MUST silently ignore Endpoint records whose `type` they do not recognize.

`type` (String):
: A URI identifying the capability type. Well-known values defined by this document:

  - `"urn:jmap:chat:cap:vtc"` — video/voice teleconference. `uri` is a signaling or room URL (e.g., a WebRTC signaling endpoint, a Jitsi room URL, a SIP URI).
  - `"urn:jmap:chat:cap:payment"` — payment receiving endpoint. `uri` is a payment URI (e.g., `lightning:...`, `zcash:...`, `monero:...`, `bitcoin:...`).
  - `"urn:jmap:chat:cap:blob"` — out-of-band file transfer endpoint. `uri` is a base URL for fetching or uploading blobs outside the JMAP blob mechanism.
  - `"urn:jmap:chat:cap:calendar-event"` — calendar event link or meeting invite. `uri` is a calendar event URL or a JMAP CalendarEvent identifier. See {{JMAP-CALENDARS}} for the underlying calendar data model and {{JMAP-CHAT-CALENDARS}} for the chat integration semantics, including RSVP handling and structured rendering hints.
  - `"urn:jmap:chat:cap:availability"` — invitation to perform a free/busy availability lookup. `uri` is a scheduling URL or JMAP Principal identifier. See {{JMAP-CALENDARS}} and {{JMAP-CHAT-CALENDARS}}.
  - `"urn:jmap:chat:cap:task"` — task or to-do item link. `uri` is a JMAP Task identifier or an external task tracking URL. See {{JMAP-TASKS}} for the underlying task data model and {{JMAP-CHAT-TASKS}} for the chat integration semantics, including task-chat back-references and structured rendering hints.
  - `"urn:jmap:chat:cap:filenode"` — file storage node link. `uri` is a JMAP FileNode URL or identifier in a Space-scoped file tree. See {{JMAP-FILENODE}}.

  Other type URIs MAY be defined by deployments or future documents. The `urn:jmap:chat:cap:` prefix is reserved for types defined in JMAP Chat specifications.

`uri` (String):
: The endpoint URI. Format is type-specific. Peer-supplied; MUST be treated as untrusted.

`label` (String, optional):
: Human-readable label for this endpoint (e.g., `"Personal Jitsi"`, `"Zcash address"`).

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys. Examples by type:

  - `vtc`: `{"protocol": "webrtc", "roomName": "...", "password": "..."}`
  - `payment`: `{"network": "lightning", "currency": "BTC"}`
  - `blob`: `{"maxBytes": 10485760}`
  - `calendar-event`: `{"title": "...", "startTime": "..."}`
  - `task`: `{"title": "...", "status": "..."}`
  - `filenode`: `{"name": "...", "mediaType": "..."}`

## MessageAction {#message-action}

A MessageAction is a per-message out-of-band action invitation carried in a Message and in `Peer/deliver`. It signals that a message is associated with an out-of-band interaction — a video call invitation, a payment request, a file available outside the blob channel, etc. Servers MUST store and forward MessageAction records without inspection or transformation. Clients MUST NOT act on a MessageAction automatically; all OOB actions require explicit user initiation.

`type` (String):
: Same URI namespace as Endpoint `type` (see {{endpoint}}). Clients MUST ignore actions whose `type` they do not recognize.

`uri` (String):
: The action URI. Peer-supplied; MUST be treated as untrusted.

`label` (String, optional):
: Human-readable label for the action (e.g., `"Join call"`, `"Pay $5"`, `"Download file"`).

`expiresAt` (UTCDate, optional):
: Time after which the action is no longer valid. Clients SHOULD visually indicate expired actions. Servers MUST NOT enforce expiry on stored actions; enforcement is the OOB system's responsibility.

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys.

## Mention {#mention}

A Mention identifies a user referenced within a message body.

`id` (String):
: The ChatContact.id (userId) of the mentioned participant.

`offset` (UnsignedInt):
: Byte offset into `body` where the mention text begins.

`length` (UnsignedInt):
: Byte length of the mention text. Servers MUST reject a mention where `offset + length` exceeds the byte length of `body`.

The `id` field carries the full identifier of the mentioned user as known to the authoritative server, regardless of textual form. Common composer-side textual forms include the Mastodon-style `@user@host` for federated users and DID URI forms (for example, `@did:web:alice.example` per {{?W3C-DID-CORE}}) when DID-based identity is in use; the wire format is unaffected by which textual form the sender's client recognized. Parsing the textual form into a candidate id, and resolving that candidate to a known ChatContact, are deployment-defined; server-side validation of the resulting Mention (offset/length checks above, ChatContact existence) applies unchanged. The same principle applies to the `userId` field on rich-body `"mention"` spans ({{rich-body}}).

The Mention type carries per-user mentions only. Broadcast-scope mentions (such as `@everyone`, `@here`, and `@admins`) are carried in the separate `broadcastMentions` array on Message; see {{broadcast-mention}}.

## BroadcastMention {#broadcast-mention}

A BroadcastMention identifies a broadcast-scope reference within a message body. Unlike Mention, a BroadcastMention does not name a specific participant; it names a member set whose composition is resolved at delivery time by each receiving server.

`scope` (String):
: One of:
  - `"everyone"` — all current members of the Space or Chat
  - `"here"` — currently-active members; the predicate that defines "active" (which `PresenceStatus.availability` states qualify, what activity window applies) is deployment-defined
  - `"admins"` — members holding administrative authority in the Space or Chat; the predicate that defines "admin" (which SpaceRole permissions qualify, whether controller principals from {{space-deployment}} are included) is deployment-defined

  Servers MUST reject a BroadcastMention with an unrecognized `scope` value with `invalidArguments`.

`offset` (UnsignedInt):
: Byte offset into `body` where the broadcast-mention text begins.

`length` (UnsignedInt):
: Byte length of the broadcast-mention text. Servers MUST reject a BroadcastMention where `offset + length` exceeds the byte length of `body`.

The canonical textual forms presented by composers are `@everyone`, `@here`, and `@admins` respectively. Deployments MAY recognize additional aliases (for example, `@all` as a synonym for `@everyone`) in their composer UI; alias-to-scope resolution is a client-side or deployment-side concern and is not part of the wire contract. The wire `scope` value is fixed to one of the three enumerations above.

The set of recipients targeted by a BroadcastMention is computed at delivery time on each receiving server, against that server's local view of membership, presence, and administrative permissions. The sending server MAY compute its own set at send time for sender-side UX (for example, displaying "you mentioned N people") but its send-time set is informational only; the receiving server's delivery-time set is authoritative for push urgency, notification routing, and badging. Where the sending and receiving servers reside on different deployments and disagree on which members qualify for `"here"` or `"admins"`, each receiving server's view governs delivery on its own owners' accounts.

Sending a Message with a non-empty `broadcastMentions` array, or with any rich-body `"broadcast"` span ({{rich-body}}), requires the `"mention_broadcast"` Space permission. Servers MUST reject `Message/set` create or update requests that would write a broadcast mention from a sender lacking this permission with `forbidden`.

## MessageRevision {#message-revision}

A MessageRevision records one historical version of a Message body.

`body` (String):
: The prior body text.

`bodyType` (String):
: The prior MIME type.

`editedAt` (UTCDate):
: The time this version was superseded by an edit.

## Reaction {#reaction}

A Reaction is an emoji response to a Message, stored as a value in the `reactions` map on a Message object. The map key is the `senderReactionId`.

`emoji` (String):
: A non-empty string identifying the reaction. Typically a Unicode emoji sequence or a deployment-defined token.

`customEmojiId` (String, optional):
: The id of a Space-scoped custom emoji. When present, `emoji` SHOULD contain a fallback representation (e.g., the emoji name) for clients that do not support custom emoji.

`senderId` (String):
: `"self"` for the owner's reaction, or a ChatContact.id.

`sentAt` (UTCDate):
: Time the reaction was added.

## ReadDisposition {#read-disposition-type}

A ReadDisposition is a String value that indicates *why* a recipient acknowledged a message. It is carried on `Message.readDisposition` and within `deliveryReceipts` entries. The following values are defined:

`"displayed"`:
: The message content was presented to the user's attention. This is the default when `readDisposition` is absent or unrecognized.

`"deleted"`:
: The message was removed from the recipient's mailbox without being displayed (e.g., bulk delete or swipe-to-dismiss without opening).

`"processed"`:
: The message was handled by an automated process (e.g., a bot or filter rule) without a human viewing it.

Implementations MUST NOT reject messages or receipts that carry an unrecognized `readDisposition` value. Unrecognized values MUST be stored as-is and SHOULD be treated as `"displayed"` for the purposes of unread-count computation and user-interface rendering. This allows deployments to define additional values (e.g., `"voice-listened"`, `"forwarded"`) without breaking older implementations.

## ChatContact {#chat-contact}

A ChatContact represents a remote user known to this mailbox. A ChatContact's `id` is exactly the userId provided by the authentication layer: it is the single, global identity key for that user within this deployment.

`id` (String, immutable, server-set):
: The userId provided by the authentication layer. Servers MUST set this to the verified identity string and MUST NOT assign a different value.

`login` (String, server-set):
: A non-empty human-readable identifier for this contact, suitable for display when `displayName` is absent. The format is deployment-specific but MUST be a valid UTF-8 string of at least one non-whitespace character. Clients SHOULD fall back to `login` when `displayName` is absent or empty, and MAY fall back to `id` when `login` is unavailable.

`displayName` (String, optional):
: Human-readable display name. MAY be absent or empty. Clients SHOULD fall back to `login`, then `id`.

`firstSeenAt` (UTCDate, server-set):
: Time this ChatContact was first recorded.

`lastSeenAt` (UTCDate, server-set):
: Time of most recent interaction with this ChatContact's mailbox.

`presence` (String, optional, server-set):
: Last known presence state: `"online"`, `"away"`, `"busy"`, `"invisible"`, or `"offline"`. Updated on a best-effort basis. If absent, the contact's presence state is unknown to this server.

`lastActiveAt` (UTCDate, optional, server-set):
: Time the ChatContact was last observed to be active.

`statusText` (String, optional, server-set):
: The contact's current custom status message. Mirrors `PresenceStatus.statusText` on the contact's own server; updated when a presence event is received from that server. Absent when the contact has no active custom status text.

`statusEmoji` (String, optional, server-set):
: The contact's current status emoji. Mirrors `PresenceStatus.statusEmoji` on the contact's own server; updated when a presence event is received from that server. Absent when the contact has no active status emoji.

`endpoints` (Endpoint[], optional):
: Out-of-band capability endpoints advertised by this ChatContact. Servers populate this field from the ChatContact's `ownerEndpoints` at `/.well-known/jmap`. Clients MAY use these to initiate video calls, send payments, or fetch files outside the JMAP message channel. Clients MUST NOT act on these values automatically without explicit user intent.

`blocked` (Boolean):
: When `true`, messages from this ChatContact are silently dropped by this mailbox, including messages arriving in group chats. Default: `false`.

Additional ChatContact fields used in federation deployments (`serverUrl`, `directAddress`) are defined in {{JMAP-CHAT-FED}}.

Note: {{RFC9610}} defines `ContactCard` (a JSContact record) for storing rich contact information in a JMAP address book. `ChatContact` and `ContactCard` serve different purposes and have distinct identity models: `ChatContact.id` is the authenticated userId assigned by the transport layer, while `ContactCard.id` is an opaque JMAP-assigned identifier. Implementations MAY surface `ChatContact` records as `ContactCard` objects in a user's address book as a display-layer mapping; this is not a protocol requirement.

## ChatMember {#chat-member}

A ChatMember describes one participant in a group Chat. The `id` field is the participant's ChatContact.id.

`id` (String):
: The participant's ChatContact.id / userId.

`role` (String):
: Either `"admin"` or `"member"`. Admins may add and remove members and update group chat metadata. The bootstrap-role assignment at chat creation — whether and how the creator initially receives the `"admin"` role — is server-defined.

`joinedAt` (UTCDate):
: Time this participant joined the chat.

`invitedBy` (String, optional):
: The ChatContact.id of the member who added this participant.

The `"admin" | "member"` role enum is the wire-observable representation of group-chat authority and is what remote peers see via {{JMAP-CHAT-FED}}. Servers MAY designate additional internal principals (server administrators, members of a dedicated moderator role, automated maintenance systems, or principals delegated by an external identity provider) as having admin-equivalent authority for the actions admins may perform; the means of designating such principals is deployment-defined. From a remote peer's perspective, all such administrative actions appear to originate from a member with the `"admin"` role on the originating server.

## Chat {#chat}

A Chat is a conversation between two or more participants. Three kinds are defined: `"direct"` (one-to-one), `"group"` (multi-party), and `"channel"` (a channel within a Space). Fields whose applicability is restricted to one or two kinds are labeled accordingly; unlabeled fields apply to all kinds.

### Chat ID Assignment {#chat-id}

All Chat IDs are ULIDs {{ULID}} assigned by the creating server at the moment the chat is created. IDs are opaque and stable for the lifetime of the chat.

For a **direct chat**, the creating server is the one whose owner sends the first message. Before assigning a new chatId, the server MUST check whether a direct chat with the relevant contactId already exists locally. If one exists, the server MUST use the existing chatId rather than creating a new one. When a `Peer/deliver` arrives for a direct chat with an unknown chatId, the receiving server creates a new Chat record with that chatId and sets `contactId` to the sender.

For a **group chat**, the creating server assigns the chatId and distributes it to all initial members via `Peer/groupUpdate` ({{peer-groupupdate}}) before any messages are sent.

**Channel** Chats are created as part of a Space via the `addChannels` patch key in `Space/set` ({{space-set}}). Their chatId is assigned by the server at that time.

### Chat Object Fields

`id` (String, immutable, server-set):
: A ULID assigned per {{chat-id}}.

`kind` (String, immutable):
: `"direct"`, `"group"`, or `"channel"`. Channel Chats do not carry a `members` field; membership and access control are determined by the containing Space.

`contactId` (String, immutable):
: **Direct chats only.** The ChatContact.id of the other participant.

`name` (String):
: **Group and channel Chats.** Display name. Required at creation for group chats; mutable by admins. For channel chats, mutable by members with `"manage_channels"` permission.

`description` (String, optional):
: **Group chats only.** Short description. Mutable by admins.

`avatarBlobId` (String, optional):
: **Group chats only.** blobId of the group avatar image. Mutable by admins.

`members` (ChatMember[]):
: **Group chats only.** Full membership list.

`spaceId` (String, immutable):
: **Channel Chats only.** The id of the containing Space.

`categoryId` (String, optional):
: **Channel Chats only.** The Category id within the Space. Absent if this channel is uncategorized.

`position` (UnsignedInt, optional):
: **Channel Chats only.** Sort order within the category (or among uncategorized channels). Lower values appear first.

`topic` (String, optional):
: **Channel Chats only.** Short description shown in the channel header. Mutable by members with `"manage_channels"` permission.

`slowModeSeconds` (UnsignedInt, optional):
: **Channel Chats only.** When non-zero, the server MUST reject messages from non-exempt members sent within this many seconds of their previous message in this channel, with `rateLimited`. Servers SHOULD exempt members holding the `"manage_channels"` permission, since channel managers are the principals who configure `slowModeSeconds` in the first place and should not be rate-limited by their own settings. Deployments MAY define additional exempt principals (for example, server administrators, members of dedicated moderator roles, or specific automated systems); the wire contract is only that the exempt decision is server-side and that non-exempt senders receive `rateLimited` when they exceed the rate. Clients SHOULD display a countdown to the next permitted send time. Clients MAY simply attempt the send and react to `rateLimited`; this is functionally equivalent and avoids the client needing to know its own exempt status. Default: `0`.

`permissionOverrides` (ChannelPermission[]):
: **Channel Chats only.** Per-channel permission overrides for specific roles or members. Evaluated after Space-level role permissions per {{space-permissions}}. Only members with `"manage_channels"` permission may modify this list. Empty by default.

`createdAt` (UTCDate, immutable, server-set):
: Time this chat was first recorded on this mailbox.

`lastMessageAt` (UTCDate, optional, server-set):
: Received time of the most recent message.

`unreadCount` (UnsignedInt, server-set):
: Count of Messages in this Chat whose `id` ULID is lexicographically greater than `ReadPosition.lastReadMessageId`. Derived server-side from the owner's ReadPosition record. If `lastReadMessageId` is absent, all messages are counted as unread.

`pinnedMessageIds` (String[]):
: Ordered list of pinned Message ids, most-recently-pinned first. For group chats, only admins may modify this list. For direct chats, the owner may modify it freely. For channel chats, only members with the `"pin"` Space permission may modify this list. Empty by default.

`muted` (Boolean):
: When `true`, push notifications for this chat are suppressed. Owner-side preference; not shared with peers. Default: `false`.

  Broadcast mentions ({{broadcast-mention}}) targeting the owner are an exception: when the owner is in the recipient set computed by the receiving server at delivery time, the server MUST elevate the push for that message regardless of `muted` or any active `muteUntil` window. Deployments MAY offer the owner a per-account or per-Chat preference to override this elevation (for example, "always honor mute"); the wire contract does not define a field for that preference, and any such preference is deployment-defined. See {{broadcast-mention-abuse}} for the security implications.

`muteUntil` (UTCDate, optional):
: Muting expires at this time. Servers SHOULD clear `muted` and `muteUntil` automatically when the time passes.

`receiveTypingIndicators` (Boolean):
: When `false`, the server MUST silently drop any typing push event (see {{push}}) for this Chat before delivering it to the owner. The sender is not informed; `Chat/typing` succeeds normally from the sender's perspective. Applies to direct and group chats only; for channel chats, servers MUST store the value if set via `Chat/set` but MUST NOT use it to suppress typing events. Default: `true`. Owner-side preference; not shared with peers.

`receiptSharing` (Boolean, optional):
: Per-Chat override of the account-level `PresenceStatus.receiptSharing` preference (see {{presence-status}}). When present, this value takes precedence over the account-level setting for this Chat only. When absent, the account-level preference applies. Owner-side preference; not shared with peers.

`messageExpirySeconds` (UnsignedInt, optional):
: A local expiry policy. When set and non-zero, messages in this chat older than this many seconds are deleted by this mailbox. Each mailbox enforces its own policy independently. This is a local setting, not a bilateral negotiated commitment.

Note: {{JMAP-METADATA}} defines a generic annotation layer that can be attached to any JMAP object. Implementations MAY support JMAP Metadata for private client-side annotations on `Chat` objects (using `objectType: "Chat"`), enabling use cases such as per-chat color coding or custom labels without modifying the shared chat record.

## Message {#message}

A Message is a single transmission within a Chat.

### Message IDs {#message-ids}

Message IDs are ULIDs {{ULID}}, assigned by the **receiving** mailbox at storage time. ULIDs are lexicographically ordered by time, enabling ordered retrieval without a separate sort field.

The **sender-assigned ULID** (`senderMsgId`) is set by the originating mailbox and carried in `Peer/deliver`. The receiving mailbox stores both its own `id` and the `senderMsgId`. Servers MUST maintain a durable index of `senderMsgId` values per chat to support idempotent delivery, `Peer/retract` lookup, and resolution of `replyTo` / `threadRootId` references. If a `senderMsgId` is seen again for the same chat, the server MAY silently discard the duplicate.

### Message Object Fields

`id` (String, immutable, server-set):
: Receiver-assigned ULID. Used in all client-facing references.

`senderMsgId` (String, immutable, server-set):
: The sender-assigned ULID carried in `Peer/deliver`. Equals `id` for messages composed by the owner.

`chatId` (String, immutable):
: ID of the containing Chat.

`senderId` (String, immutable, server-set):
: `"self"` for owner-composed messages; the sender's ChatContact.id for inbound messages, as verified by the authentication layer.

`body` (String):
: Message content. When `bodyType` is `"text/plain"` or another plaintext type, `body` MUST be valid UTF-8 text. When `bodyType` indicates an end-to-end encrypted payload (e.g., `"application/mls-ciphertext"`), `body` contains ciphertext encoded as a base64url string; servers MUST store and forward it without inspection or transformation. Cleared to empty string when the message is deleted.

`bodyType` (String):
: MIME type of `body`. MUST be in `supportedBodyTypes`.

`attachments` (Attachment[]):
: File attachments. Cleared to empty array when deleted.

`mentions` (Mention[]):
: Structured per-user @mention annotations. Empty by default.

`broadcastMentions` (BroadcastMention[]):
: Structured broadcast-scope mention annotations (`@everyone`, `@here`, `@admins`). Empty by default. See {{broadcast-mention}}. When `bodyType` is `"application/jmap-chat-rich"`, this array MUST be empty; broadcast-scope mentions are carried inline as `"broadcast"` spans ({{rich-body}}).

`actions` (MessageAction[]):
: Out-of-band action invitations associated with this message. Empty by default. Servers MUST store and forward these without inspection.

`reactions` (Id[Reaction]):
: Emoji reactions, keyed by `senderReactionId`. The `senderReactionId` is a client-assigned ULID for owner reactions, or a peer-supplied ULID for received reactions. Empty object by default.

`replyTo` (String, optional):
: The receiver-assigned `id` of the Message this replies to. Servers MUST validate that this ID exists in the same Chat before storing.

`threadRootId` (String, optional):
: The receiver-assigned `id` of the thread root message. Only meaningful when `supportsThreads` is `true`. See {{threads}}.

`replyCount` (UnsignedInt, server-set):
: Count of messages in this chat with `replyTo` equal to this message's `id`. Present only when `supportsThreads` is `true`.

`unreadReplyCount` (UnsignedInt, server-set, optional):
: Count of replies to this message received after the owner's `ReadPosition.lastReadMessageId` in this Chat. Present only when `supportsThreads` is `true` and this message is a thread root (`threadRootId` is absent). A value of zero means all replies have been read.

`sentAt` (UTCDate):
: Sender's claimed composition time. Peer-supplied; MUST be treated as untrusted and MUST NOT be used for ordering.

`receivedAt` (UTCDate, immutable, server-set):
: Time this mailbox stored the message. Authoritative for ordering and expiry calculations.

`senderExpiresAt` (UTCDate, optional, immutable):
: Sender-set hard-deletion deadline. When present, servers MUST permanently delete this message — removing the row entirely, not leaving a tombstone — at or before this time. A hard-deleted message appears in the `destroyed` list of subsequent `Message/changes` responses, not `updated`. Receiving servers MUST honor this field regardless of local `messageExpirySeconds` policy; whichever deadline arrives first takes effect. Servers MUST NOT use this field for ordering. Servers MUST reject a `senderExpiresAt` value that is already in the past at delivery time with `invalidArguments`. After hard deletion, stored attachment blobs referenced by the message SHOULD also be purged.

`burnOnRead` (Boolean, optional, immutable):
: When `true`, the receiving server MUST permanently hard-delete this message (row removal, not tombstone) immediately after setting `readAt`. Applies only to the receiving mailbox; the sender's own copy is not affected. In E2EE relay deployments, the relay cannot observe read events; the bridge or client layer MUST enforce this rule after receiving the read acknowledgement from the owner. When the recipient's effective `receiptSharing` preference is `false`, the receiver's server still sets `readAt` locally and the `burnOnRead` hard-delete fires as normal — the message IS deleted from the receiving mailbox. However, the sending server receives no `Peer/receipt` confirmation and cannot verify the deletion occurred. Senders MUST NOT rely on `Peer/receipt` delivery to confirm that a `burnOnRead` message was destroyed.

`deliveryState` (String, server-set):
: `"pending"`, `"delivered"`, `"failed"`, or `"received"`. For group chats, reflects aggregate state across all recipients; see `deliveryReceipts` for per-recipient detail.

`deliveryReceipts` (Object, optional, server-set):
: For group chats, a JSON object mapping each non-owner participant's ChatContact.id to `{"deliveredAt": <UTCDate-or-null>, "deviceDeliveredAt": <UTCDate-or-null>, "readAt": <UTCDate-or-null>, "readDisposition": <String-or-null>}`. Present only when `senderId` is `"self"`. `deviceDeliveredAt` is optional and MAY be absent if the recipient's platform does not support device-delivery confirmation (e.g., APNs content-available callbacks or FCM data delivery receipts); implementations that cannot obtain this signal MUST omit the field rather than approximate it with `deliveredAt`. `readDisposition` is absent when `readAt` is null. See {{read-disposition-type}}. Unlike the top-level `deliveredAt` and `readAt` fields, `deviceDeliveredAt` has no top-level parallel because device-delivery confirmation is always a per-recipient event with no aggregate owner-side equivalent.

`deliveredAt` (UTCDate, optional, server-set):
: Time the first outbound delivery was acknowledged.

`readAt` (UTCDate, optional, server-set):
: Time the owner acknowledged reading this message.

`readDisposition` (String, optional, server-set):
: The ReadDisposition value ({{read-disposition-type}}) recorded when `readAt` was last set. Absent when `readAt` is not set. When the server sets `readAt` in response to a `Message/set` update that omits `readDisposition`, the server MUST store `"displayed"`.

`editedAt` (UTCDate, optional, server-set):
: Time of the most recent edit.

`editHistory` (MessageRevision[], optional, server-set):
: Prior versions, oldest first. Servers MAY limit the number of retained revisions.

Note: {{JMAP-OBJ-HISTORY}} defines a general JMAP mechanism for retrieving historical object versions via `Foo/get` with `includeReplaced`. This specification instead embeds edit history inline within the Message object, trading on-demand retrieval for always-available history at the cost of response size. Servers implementing both specifications MAY register `"Message"` with `canRetrieveHistory: true` in the JMAP Data Types Registry, enabling `Message/get` with `includeReplaced` as an alternative retrieval path for clients that prefer on-demand history.

`deletedAt` (UTCDate, optional, server-set):
: Time the message was deleted. When set, `body` is empty and `attachments` is empty. The record is retained as a tombstone unless a hard-delete rule applies.

`deletedForAll` (Boolean, optional, server-set):
: `true` when deletion was propagated to all participants via `Peer/retract`.

Note: {{JMAP-METADATA}} defines a generic annotation layer that can be attached to any JMAP object. Implementations MAY support JMAP Metadata for private client-side annotations on `Message` objects (using `objectType: "Message"`), enabling use cases such as per-message bookmarks or personal labels without modifying the message record itself.

## SpaceRole {#space-role}

A SpaceRole is a named set of permissions within a Space. Roles are ordered by `position`; higher position values outrank lower ones. The implicit `@everyone` role (position 0) is held by all Space members and is not included in the `roles` array.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this role.

`name` (String):
: Display name of the role.

`color` (String, optional):
: Hex color string (e.g., `"#5865f2"`). Clients MAY use this to visually distinguish role holders.

`permissions` (String[]):
: Named permissions this role grants. Defined permission names:

  - `"view"` — see the channel
  - `"send"` — send messages
  - `"pin"` — pin messages
  - `"manage_channels"` — create, edit, delete, and reorder channels
  - `"manage_members"` — kick members, edit nicknames
  - `"manage_roles"` — create and edit roles below own highest role
  - `"manage_space"` — edit Space name, description, and icon
  - `"ban"` — ban and unban members
  - `"mention_broadcast"` — use broadcast-scope @mentions (`@everyone`, `@here`, `@admins`); see {{broadcast-mention}}

  Servers MUST ignore unrecognized permission names.

`position` (UnsignedInt):
: Role hierarchy position, sorted descending: higher `position` values outrank lower ones. Position `0` is reserved for the implicit `@everyone` role, which every member of a Space holds and which serves as the permission floor; defined SpaceRoles MUST have `position` > 0. No two roles in a Space SHOULD share the same value.

## SpaceMember {#space-member}

A SpaceMember describes one participant in a Space.

`id` (String):
: The participant's ChatContact.id.

`roleIds` (String[]):
: SpaceRole ids held by this member. Order is not significant. An empty list means the member holds only the `@everyone` role.

`nick` (String, optional):
: Space-specific display name override. MAY be absent; clients SHOULD fall back to ChatContact `displayName`, then `login`.

`joinedAt` (UTCDate):
: Time this member joined the Space.

## Category {#category}

A Category is a named grouping of channels within a Space.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this category.

`name` (String):
: Display name of the category.

`position` (UnsignedInt):
: Sort order among categories. Lower values appear first.

`channelIds` (String[]):
: Ordered list of channel Chat ids in this category.

## ChannelPermission {#channel-permission}

A ChannelPermission record overrides Space-level role permissions for a specific channel, for a specific role or member. The evaluation order for these overrides is defined in {{space-permissions}}.

`targetId` (String):
: A SpaceRole id or a SpaceMember ChatContact.id.

`targetType` (String):
: `"role"` or `"member"`.

`allow` (String[]):
: Permissions explicitly granted in this channel, overriding the Space-level role defaults.

`deny` (String[]):
: Permissions explicitly denied in this channel, overriding the Space-level role defaults.

## Space {#space}

A Space is a named container for channel Chats, members, roles, and categories. It corresponds to what other systems call a server, workspace, or team.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this Space.

`name` (String):
: Display name of the Space.

`description` (String, optional):
: Short description. Mutable by members with `"manage_space"` permission.

`iconBlobId` (String, optional):
: blobId of the Space icon. Mutable by members with `"manage_space"` permission.

`roles` (SpaceRole[]):
: Named roles defined for this Space, ordered by `position` descending. Does not include the implicit `@everyone` role.

`members` (SpaceMember[]):
: Full membership list.

`categories` (Category[]):
: Categories, ordered by `position`.

`uncategorizedChannelIds` (String[]):
: Ordered list of channel Chat ids not assigned to any category.

`createdAt` (UTCDate, immutable, server-set):
: Time this Space was created.

`isPublic` (Boolean):
: If `true`, any user may join this Space via `Space/join` without an invite code. Default is `false`. Mutable by members with `"manage_space"` permission.

`isPubliclyPreviewable` (Boolean):
: If `true`, users who are not members of this Space may discover its existence via `Space/query` and fetch a restricted view of it via `Space/get`. The restricted view is defined in the `Space/get` section. Default is `false`. Mutable by members with `"manage_space"` permission.

`memberCount` (UnsignedInt, server-set):
: Current number of members in this Space.

Note: {{RFC9670}} defines a general JMAP sharing framework (`shareWith`, `myRights`) for simple read/write/admin access control. The Space permission model (role hierarchy, named permissions, per-channel overrides) is intentionally richer than that framework and is not expressed using it. In deployments that also implement {{RFC9670}}, server implementations may choose to align `Principal.id` values with `ChatContact.id` values (both ultimately derived from the same authentication identity), but this alignment is implementation-defined and not required by either specification.

Note: {{JMAP-FILENODE}} defines a hierarchical file-storage extension for JMAP. A future companion specification could define Space-scoped file storage by associating a Filenode namespace with each Space, analogous to how server-to-server federation methods are defined in a separate companion draft.

Note: {{JMAP-METADATA}} defines a generic annotation layer that can be attached to any JMAP object. Implementations MAY support JMAP Metadata for private client-side annotations on `Space` objects (using `objectType: "Space"`), enabling use cases such as per-Space color tags or custom labels without server-side semantic significance.

## CustomEmoji {#custom-emoji}

A CustomEmoji is a server- or Space-scoped custom emoji image available for use in Reactions.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this emoji.

`name` (String):
: The shortcode name for this emoji, without colons (e.g., `catjam`). MUST be unique within its scope (Space or server-global). MUST contain only lowercase alphanumeric characters, hyphens, and underscores.

`blobId` (String):
: blobId of the emoji image. MUST be a valid image type (e.g., PNG, GIF, WebP).

`spaceId` (String, optional):
: The id of the Space this emoji belongs to. If absent, the emoji is server-global and available in all chats on this server.

`createdBy` (String, immutable, server-set):
: ChatContact.id of the user who created this emoji.

`createdAt` (UTCDate, immutable, server-set):
: Time this emoji was created.

## SpaceInvite {#space-invite}

A SpaceInvite grants a new member access to a Space via a shared invite code.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this invite.

`code` (String, immutable, server-set):
: The user-shareable invite code. This is the value passed as `inviteCode` to `Space/join`. Servers SHOULD generate short, URL-safe strings suitable for sharing.

`spaceId` (String, immutable):
: The Space this invite grants access to.

`defaultChannelId` (String, optional):
: Chat id of the channel to highlight when a new member arrives.

`createdBy` (String, immutable, server-set):
: ChatContact.id of the member who created this invite.

`expiresAt` (UTCDate, optional):
: Expiry time. Servers MUST reject redemption after this time.

`maxUses` (UnsignedInt, optional):
: Maximum redemption count. Servers MUST reject redemption when `uses` equals `maxUses`.

`uses` (UnsignedInt, server-set):
: Number of times this invite has been redeemed.

`createdAt` (UTCDate, immutable, server-set):
: Time this invite was created.

## SpaceBan {#space-ban}

A SpaceBan prevents a user from participating in a Space.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this ban.

`spaceId` (String, immutable):
: The id of the Space this ban applies to.

`userId` (String, immutable):
: The ChatContact.id of the banned user.

`bannedBy` (String, immutable, server-set):
: The ChatContact.id of the Space member who issued this ban.

`reason` (String, optional):
: Human-readable reason for the ban. Visible to the banned user.

`createdAt` (UTCDate, immutable, server-set):
: Time this ban was created.

`expiresAt` (UTCDate, optional):
: If present, the ban expires at this time and the server MUST restore the user's access. If absent, the ban is permanent until explicitly destroyed.

## ReadPosition {#read-position}

A ReadPosition tracks the owner's read state within a Chat. The server creates a ReadPosition record automatically when a Chat first becomes visible to the owner, and destroys it when the Chat is destroyed. For direct and group Chats, a ReadPosition is created when the first message is delivered. For channel Chats, a ReadPosition is created when the owner joins the containing Space. This cursor model serves the same local read-tracking purpose as the `$seen` keyword in {{RFC8621}} (JMAP Mail), but uses a per-chat high-water mark rather than per-message boolean flags, which is appropriate for the ordered-stream access pattern of chat.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this read position.

`chatId` (String, immutable):
: The id of the Chat this position tracks.

`lastReadMessageId` (String, optional):
: The `id` of the most recent Message the owner has read in this Chat. The server uses this to compute `Chat.unreadCount`. If absent, the owner has not read any messages in this Chat.

`lastReadAt` (UTCDate, server-set):
: Time the `lastReadMessageId` was last updated.

The receipt-sharing opt-out is bidirectional: when the effective `receiptSharing` preference is `false` (either `PresenceStatus.receiptSharing` at account level, or a `Chat.receiptSharing` override for the specific Chat), the server MUST NOT deliver inbound `Peer/receipt` events (carrying `readAt`) to the owner's clients, as defined in {{JMAP-CHAT-FED}}. An account that does not broadcast its own read times does not observe others' read times in the affected scope.

## PresenceStatus {#presence-status}

A PresenceStatus represents the owner's self-reported availability and custom status. There is exactly one PresenceStatus record per account; the server creates it automatically.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this record.

`presence` (String):
: The owner's self-reported availability. One of `"online"`, `"away"`, `"busy"`, `"invisible"`, or `"offline"`. Default is `"online"`.

`statusText` (String, optional):
: A short custom status message. If absent, no custom text is displayed.

`statusEmoji` (String, optional):
: A single emoji or emoji shortcode representing the owner's status. If absent, no status emoji is displayed.

`expiresAt` (UTCDate, optional):
: If set, the server SHOULD clear `statusText` and `statusEmoji` (setting them to null) at this time, and SHOULD reset `presence` to `"online"`.

`receiptSharing` (Boolean):
: When `false`, this account does not send read receipts to peers. The server MUST NOT invoke `Peer/receipt` with a `readAt` value on behalf of this account. The server MUST silently suppress such outbound calls; the recipient is not notified of the suppression. Default: `true`.

`updatedAt` (UTCDate, server-set):
: Time the owner last updated this record.

# Methods

Note: Implementations SHOULD support the `urn:ietf:params:jmap:refplus` capability {{JMAP-REFPLUS}} for ergonomic multi-step batch operations. RefPlus allows result references inside `/set` create/update bodies and `/query` filter conditions, enabling patterns such as creating a Message and immediately using its returned `id` in a `Message/query` within the same JMAP request array.

## ChatContact Methods

ChatContacts are created automatically when a peer delivers a message or a group update names a new participant. Owner clients may not create or destroy ChatContacts directly.

### ChatContact/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### ChatContact/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### ChatContact/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`create` and `destroy` are not supported; both MUST return `forbidden`.

`update` supports: `blocked`, `displayName`.

### ChatContact/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties: `blocked` (Boolean), `presence` (String).
Sort properties: `lastSeenAt`, `login`, `lastActiveAt`. When sorting by `lastActiveAt`, ChatContacts for which the field is absent sort last.

### ChatContact/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## Chat Methods

### Chat/get

Standard JMAP `/get`.

### Chat/changes

Standard JMAP `/changes`.

### Chat/set

Standard JMAP `/set`.

#### Creating a Direct Chat

`create` with `kind: "direct"` accepts:

`contactId` (String, required):
: ChatContact.id of the other participant. If a direct Chat with this contactId already exists, the server MUST return it in `updated` rather than creating a duplicate. Otherwise the server assigns a new ULID per {{chat-id}}.

#### Creating a Group Chat

`create` with `kind: "group"` accepts:

`name` (String, required):
: Display name of the group.

`memberIds` (String[], required):
: ChatContact.ids of additional initial members. If the resulting membership would exceed a server-defined limit on the number of members per group Chat, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

Optional at creation: `description` (String), `avatarBlobId` (String), `messageExpirySeconds` (UnsignedInt).

The server assigns the chatId (a ULID), adds the creator to `members` with at least one role granting permissions sufficient to administer the chat (the specific bootstrap-role configuration is server-defined), and MUST send `Peer/groupUpdate` to each initial member before any messages are sent.

#### Updating a Chat

`update` supports the following patch keys for all chat kinds: `muted`, `muteUntil`, `receiveTypingIndicators`, `pinnedMessageIds`, `messageExpirySeconds`, `receiptSharing`.

The `receiveTypingIndicators` field is described in {{chat}}; it applies to direct and group chats only.

For group chats, admins additionally may update: `name`, `description`, `avatarBlobId`.

Member list changes use the following patch keys (all require admin role):

`addMembers` (Object[]):
: Each entry: `id` (String, ChatContact.id) and optional `role` (String, default `"member"`). If the resulting membership would exceed a server-defined limit on the number of members per group Chat, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3) for that entry. On success, the server MUST send `Peer/groupUpdate` to all current members.

`removeMembers` (String[]):
: ChatContact.ids to remove. The server MUST send `Peer/groupUpdate` to all remaining members and to the removed members.

`updateMemberRoles` (Object[]):
: Each entry: `id` (String) and `role` (String). The server MUST send `Peer/groupUpdate` to all members.

### Chat/query

Standard JMAP `/query`.

Filter properties: `kind` (String), `muted` (Boolean).
Default sort: `lastMessageAt` descending; chats without messages sort last.

### Chat/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

### Chat/typing {#chat-typing}

Signals to the server that the authenticated user has started or stopped typing in a specific Chat. The event is ephemeral; it is never stored.

Method name: `Chat/typing`

#### Request Arguments

`accountId` (String):
: The account making the request.

`chatId` (String):
: The Chat ID of the conversation in which typing is occurring. The server MUST return a `notFound` method error if `chatId` does not exist or the account is not a participant in that Chat.

`typing` (Boolean):
: `true` if the user has started typing; `false` if typing has stopped.

#### Response

`accountId` (String):
: The account ID from the request.

#### Server Behavior

The server MUST NOT persist this event. Before delivering a typing push event to each potential recipient, the server MUST check that recipient's `receiveTypingIndicators` field for this Chat; if it is `false` and the Chat `kind` is `"direct"` or `"group"`, the server MUST silently drop the event and MUST NOT deliver it to any of that recipient's connected clients. For channel chats, `receiveTypingIndicators` has no effect. The server MUST also check whether the requesting account corresponds to a ChatContact whose `blocked` field is `true` on the recipient's contact list; if so, the server MUST silently drop the event for that recipient and MUST NOT deliver it to any of that recipient's connected clients. The sender is not informed. On success, the server SHOULD deliver a `typing` push event (see {{push}}) to all connected clients of each local participant in the Chat other than the requesting account, subject to the `receiveTypingIndicators` and blocked-sender checks above. For federated Chats, the server SHOULD invoke `Peer/typing` ({{JMAP-CHAT-FED}}) on each remote participant's home server.

Servers SHOULD rate-limit `Chat/typing` calls per account per chat. Servers SHOULD accept no more than one call per account per chat per 3 seconds; calls that exceed this rate MAY be silently discarded without error.

The recommended 3-second value pairs with the 10-second client-side decay timer described later in this section: with one accepted event per 3 seconds while a sender is actively typing, the receiver's decay timer sees at least three events before deciding the sender has stopped. Servers using a significantly lower rate impose unnecessary load on themselves and on federated peers; servers using a significantly higher rate risk receivers clearing the typing indicator while the sender is still active.

Changes to `receiveTypingIndicators` via `Chat/set` MUST take effect for all typing push events delivered after the `Chat/set` response is returned.

When `receiveTypingIndicators` transitions from `false` to `true`, the server does not synthesize `typing: false` events for previously suppressed senders. Clients SHOULD use a decay timer to recover from any stale typing state; if no typing event is received for a given `(chatId, senderId)` pair within approximately 10 seconds, the client SHOULD hide the typing indicator for that pair.

The recommended 10-second value is calibrated against the server-side rate limit specified earlier in this section ({{chat-typing}}): with one `Chat/typing` call accepted per 3 seconds while a sender is actively typing, 10 seconds represents three missed events and is a reliable signal that the sender has stopped. Clients that diverge significantly from this value will appear inconsistent to users running multiple clients in parallel (for example, a mobile client showing the typing indicator long after a desktop client has cleared it). Implementations targeting environments with high inbound latency MAY choose a slightly longer window; implementations targeting low-latency contexts MAY choose shorter.

Note: `Chat.receiveTypingIndicators` is a persistent per-chat preference and survives reconnects. It is distinct from the `chatIds` scope in `ChatStreamEnable` ({{JMAP-CHAT-WSS}}), which is a session-scoped view-management filter. Both may suppress delivery of a typing event for the same chat, but they serve different purposes and are evaluated independently.

Implementations SHOULD cache each participant's `receiveTypingIndicators` value in memory and invalidate the cache entry on `Chat/set` updates, rather than performing a database read per delivery.

The privacy requirement that senders cannot detect suppression means a server cannot signal opt-out status back to the caller. Servers MUST accept and process all conforming `Chat/typing` calls regardless of any recipient's `receiveTypingIndicators` value.

## Message Methods

### Message/get

Standard JMAP `/get`.

### Message/changes

Standard JMAP `/changes`.

### Message/set

Standard JMAP `/set`.

#### Creating a Message

`create` accepts:

`chatId` (String, required), `body` (String, required), `bodyType` (String, required), `sentAt` (UTCDate, required).

Optional: `attachments` (Attachment[]), `mentions` (Mention[]), `broadcastMentions` (BroadcastMention[]), `actions` (MessageAction[]), `replyTo` (String), `threadRootId` (String), `senderExpiresAt` (UTCDate), `burnOnRead` (Boolean).

The server sets `id`, `senderMsgId`, `senderId`, `receivedAt`, `deliveryState`, and delivery timestamp fields, then enqueues the message for outbound delivery.

A create request that includes any broadcast mention — a non-empty `broadcastMentions` array, or a rich-body `body` containing one or more `"broadcast"` spans ({{rich-body}}) — MUST be rejected with `forbidden` when the sender lacks the `"mention_broadcast"` permission in the target Space.

#### Editing a Message

`update` with changed `body`, `bodyType`, `mentions`, and/or `broadcastMentions`, on a message where `senderId` is `"self"` and `deletedAt` is absent. The same `"mention_broadcast"` permission check applied at create time applies to updates that add or retain a broadcast mention.

The server MUST:

1. If the server retains edit history, push a MessageRevision onto `editHistory` with the current `body`, `bodyType`, and current server time as `editedAt`.
2. Replace `body` and `bodyType` with the submitted values.
3. Set `editedAt` to the current server time.
4. Send `Peer/deliver` carrying an `edit` payload to all recipients (see {{peer-deliver}}).

#### Adding and Removing Reactions

Reactions are mutated via standard JSON Pointer patch keys on the `reactions` map.

To **add** a reaction, the client SHOULD supply a ULID as the `senderReactionId` and patch:

~~~
"reactions/<senderReactionId>": {
  "emoji": "<value>",
  "sentAt": "<UTCDate>"
}
~~~

The server MUST set `senderId` to `"self"` and MUST propagate the reaction to all recipients via `Peer/deliver` `reactionUpdate` payload, carrying the `senderReactionId` as the map key.

To **remove** a reaction, the client patches:

~~~
"reactions/<senderReactionId>": null
~~~

The server MUST remove the entry and MUST propagate the removal via `Peer/deliver` `reactionUpdate` payload using the same `senderReactionId`.

Clients SHOULD only add or remove reactions where `senderId` is `"self"`. Servers MUST reject attempts to add or remove reactions for other senders with `forbidden`.

#### Deleting a Message

`update` with `deletedAt: <timestamp>`.

- If `deletedForAll: true` is also set, the server MUST send `Peer/retract` to all participants before marking the local record. Servers MUST reject `deletedForAll: true` for messages where `senderId` is not `"self"`.
- Otherwise, deletion is local only: `body` and `attachments` are cleared on this mailbox with no peer notification.

#### Marking as Read

`update` with `readAt: <UTCDate>` and optionally `readDisposition: <String>`. If `readDisposition` is omitted, the server stores `"displayed"`. See {{read-disposition-type}} for defined values; unrecognized values are stored as-is.

### Message/query

Standard JMAP `/query`.

All requests MUST include a `chatId` filter, unless the request includes a `hasMention: true` filter condition. Servers MUST return an `unsupportedFilter` error for any request that omits `chatId` without also including `hasMention: true`.

When `chatId` is absent and `hasMention: true` is present, the query spans all Chats of which the owner is a member. Servers that do not support cross-chat mention queries MUST return `unsupportedFilter`; clients SHOULD handle this gracefully.

Filter properties:

`chatId` (String, optional):
: Restrict results to a single Chat. Required unless `hasMention: true` is also present.

`text` (String, optional):
: Full-text search over `body`. Servers that do not support full-text search MUST return `unsupportedFilter`.

`threadRootId` (String, optional):
: Return only messages in this thread. Valid only when `supportsThreads` is `true`; otherwise servers MUST return `unsupportedFilter`.

`hasAttachment` (Boolean, optional):
: Filter to messages with or without attachments.

`hasMention` (Boolean, optional):
: Filter to messages that mention the owner (owner's ChatContact.id appears in `mentions`).

Default sort: `receivedAt` ascending when `chatId` is present; `receivedAt` descending when `chatId` is absent.

### Message/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## Space Methods

### Space/get

Standard JMAP `/get`, with the following additional rules for non-member callers.

When the caller is not a member of a requested Space:

* If the Space has `isPubliclyPreviewable: true`, the server MUST return a restricted view of the Space containing only the fields `id`, `name`, `description`, `iconBlobId`, `memberCount`, `createdAt`, `isPublic`, and `isPubliclyPreviewable`. All other Space fields MUST be omitted from the returned object, even if explicitly requested via the `properties` argument. The server MUST NOT include the Space `id` in the `notFound` array.
* If the Space has `isPubliclyPreviewable: false` (or the Space does not exist), the server MUST include the requested Space `id` in the `notFound` array of the response and MUST NOT include the Space in the `list` array.

When the caller is a member of a requested Space, the full Space object is returned subject to the standard JMAP `/get` `properties` filter; the restricted view above does not apply.

### Space/changes

Standard JMAP `/changes`. The Space object is returned in full on each change; the sub-arrays `roles`, `members`, and `categories` are included in every Space object and reflect the current state.

### Space/set {#space-set}

Standard JMAP `/set`.

The `update` operation for Space uses semantic mutation keys (`addRoles`, `removeRoles`, `addMembers`, etc.) rather than JSON Pointer paths. This departure from the standard RFC 8620 PatchObject model is intentional: Space membership and role lists are ordered, server-enforced collections subject to permission checks and cascading side effects (e.g., peer notifications, role hierarchy enforcement) that cannot be expressed as simple pointer-path assignments. Each key names a discrete, permission-checked mutation operation rather than a direct property assignment.

#### Creating a Space

`create` accepts:

`name` (String, required).

Optional: `description` (String), `iconBlobId` (String).

The server assigns the Space's `id` and adds the caller to `members` with at least one role granting permissions sufficient to administer the Space. The specific bootstrap-role configuration is server-defined; servers MAY pre-create a high-position role for the caller, MAY assign every permission from {{space-permissions}}, or MAY use any other mechanism that ensures the newly-created Space is administrable. The server returns the new Space.

#### Updating a Space

`update` supports the following patch keys:

`name`, `description`, `iconBlobId`:
: Metadata fields. Require `"manage_space"` permission.

`addRoles` (Object[]):
: Each entry: `name` (String), `permissions` (String[]), `position` (UnsignedInt), and optionally `color` (String). Server assigns ULIDs. Requires `"manage_roles"`. Members may only add roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this. If the resulting role count would exceed a server-defined limit on the number of roles per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeRoles` (String[]):
: SpaceRole ids to remove. Members holding only removed roles are demoted to `@everyone`. Requires `"manage_roles"`.

`updateRoles` (Object[]):
: Each entry: `id` (String) and any of `name`, `color`, `permissions`, `position`. Requires `"manage_roles"`. Members may only modify roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this.

`addMembers` (Object[]):
: Each entry: `id` (ChatContact.id, String) and optional `roleIds` (String[]). Requires `"manage_members"`. If the resulting membership would exceed a server-defined limit on the number of members per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeMembers` (String[]):
: ChatContact.ids to remove. Requires `"manage_members"`. Servers SHOULD refuse to remove a member when doing so would leave the Space without any member holding `"manage_members"` or `"manage_space"`; the specific last-administrator protection policy is server-defined. Servers MUST return an `invalidProperties` SetError ({{RFC8620}} §5.3) naming `removeMembers` when refusing on this basis.

`updateMembers` (Object[]):
: Each entry: `id` (String) and any of `roleIds`, `nick`. Role changes require `"manage_roles"`.

`addChannels` (Object[]):
: Each entry: `name` (String, required), optional `categoryId` (String), `position` (UnsignedInt), and `topic` (String). The server creates a Chat record of `kind: "channel"` with `spaceId` set to this Space's id and assigns a ULID as the chatId. Requires `"manage_channels"`. If the resulting channel count would exceed a server-defined limit on the number of channels per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeChannels` (String[]):
: Channel Chat ids to remove. Cascades to all Messages in those channels. Requires `"manage_channels"`.

`updateChannels` (Object[]):
: Each entry: `id` (String, channel Chat id) and any of `name`, `topic`, `categoryId`, `position`, `slowModeSeconds`, `permissionOverrides`. Requires `"manage_channels"`.

`addCategories` (Object[]):
: Each entry: `name` (String), optional `position` (UnsignedInt) and `channelIds` (String[]). Server assigns ULIDs. Requires `"manage_channels"`. If the resulting category count would exceed a server-defined limit on the number of categories per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeCategories` (String[]):
: Category ids to remove. Channels in removed categories move to `uncategorizedChannelIds`. Requires `"manage_channels"`.

`updateCategories` (Object[]):
: Each entry: `id` (String) and any of `name`, `position`, `channelIds`. Requires `"manage_channels"`.

#### Destroying a Space

Cascades to all channel Chats and their Messages. Hard-deletes all records; no tombstones are retained. The caller MUST hold `"manage_space"`; servers MAY impose additional deployment-defined requirements (multi-administrator approval, dedicated controller principal, etc.) before honoring the destroy. Servers MUST return a `forbidden` SetError when the caller does not satisfy the deployment's destroy-authorization rules.

### Space/query

Standard JMAP `/query`.

Filter properties: `name` (String, substring match), `isPublic` (Boolean).
Default sort: `name` ascending.

`Space/query` follows the standard {{RFC8620}} Section 5.5 response shape and returns only the matching `ids`. Field-level visibility restrictions for non-member callers are applied by `Space/get` (see the `Space/get` section above).

When the request includes an `isPublic: true` filter condition, the server MUST include in the result `ids` array the identifiers of Spaces for which the caller is not a member but which have both `isPublic: true` and `isPubliclyPreviewable: true`. For Spaces where the caller is a member, identifiers are included irrespective of `isPubliclyPreviewable` per the standard `/query` semantics. Callers retrieve the (possibly restricted) Space objects by chaining a `Space/get` call against the returned `ids`, typically via a `#ids` ResultReference.

### Space/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

### Space/join

Allows a caller to join a Space either by redeeming an invite code or by directly requesting membership in a public Space.

Method name: `Space/join`

Request: `accountId` (String), and exactly one of `inviteCode` (String) or `spaceId` (String). Supplying both or neither MUST cause the server to return an `invalidArguments` method error.

**Joining via invite code:** The server MUST resolve `inviteCode` to a SpaceInvite record by matching against the `code` field (not the `id` field), verify the invite has not expired and has not reached `maxUses`, and then atomically increment `uses` and add the caller to the Space's member list. The `uses` increment and membership insertion MUST be performed within a single atomic operation so that concurrent redemptions cannot exceed `maxUses`. The caller is assigned no roles beyond `@everyone` unless the invite specifies otherwise. If the invite has expired (`expiresAt` is in the past) or has reached its redemption limit (`uses` equals `maxUses`), the server MUST return an `invalidArguments` method error. If the `inviteCode` does not correspond to any SpaceInvite record, the server MUST return an `invalidArguments` method error.

**Joining a public Space:** When `spaceId` is supplied, the server MUST verify that the identified Space has `isPublic: true`. If the Space does not exist or has `isPublic: false`, the server MUST return a `notPermitted` method error. On success, the server adds the caller to the Space's member list with no roles beyond `@everyone`.

Response: `accountId` (String), `spaceId` (String).

## ReadPosition Methods

### ReadPosition/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### ReadPosition/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### ReadPosition/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`update` supports: `lastReadMessageId`. The server sets `lastReadAt` to the current time and recomputes `Chat.unreadCount` for the affected Chat.

`create` and `destroy` are not supported; both MUST return a SetError of type `forbidden`. ReadPosition records are managed by the server.

## PresenceStatus Methods

### PresenceStatus/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1). Returns the singleton PresenceStatus record for the account.

### PresenceStatus/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### PresenceStatus/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`update` supports: `presence`, `statusText`, `statusEmoji`, `expiresAt`, `receiptSharing`.

`create` and `destroy` are not supported; both MUST return a SetError of type `forbidden`. The PresenceStatus record is managed by the server.

## CustomEmoji Methods

### CustomEmoji/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### CustomEmoji/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### CustomEmoji/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

Authorization for `CustomEmoji/set` (`create`, `update`, and `destroy`) is implementation-defined, for both Space-scoped and server-global emoji. Servers MUST return a `forbidden` SetError ({{RFC8620}} §5.3) when the caller is not authorized to act on the targeted emoji.

`create` accepts: `name` (String, required), `blobId` (String, required), `spaceId` (String, optional).

`update` supports: `name`, `blobId`.

`destroy` removes the emoji. Existing Reaction records referencing this emoji retain their `customEmojiId` value; clients SHOULD fall back to the `emoji` field when the referenced emoji cannot be resolved.

### CustomEmoji/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties: `spaceId` (String, optional — if absent returns all emoji accessible to the account including server-global).
Default sort: `name` ascending.

### CustomEmoji/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SpaceInvite Methods

### SpaceInvite/get

Standard JMAP `/get`. Only members of the Space may retrieve its invites. Members with `"manage_members"` permission see all invites; other members see only invites they created.

### SpaceInvite/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2). The same visibility rules as `SpaceInvite/get` apply: members with `"manage_members"` permission see all changes; other members see only changes to invites they created.

### SpaceInvite/set

Standard JMAP `/set`.

`create` accepts: `spaceId` (String, required), `defaultChannelId` (String, optional), `expiresAt` (UTCDate, optional), `maxUses` (UnsignedInt, optional). Requires `"manage_members"` permission. The server assigns both `id` and `code`; clients MUST NOT supply either.

`destroy` revokes an invite. Requires `"manage_members"` permission or ownership of the invite.

`update` is not supported; any attempt MUST return a SetError of type `forbidden`.

## SpaceBan Methods

### SpaceBan/get

Standard JMAP `/get`. Members with `"ban"` permission in the Space see all SpaceBan records for that Space. A banned user fetching their own account sees only SpaceBan records where `userId` matches their own identity.

### SpaceBan/changes

Standard JMAP `/changes`.

### SpaceBan/set

Standard JMAP `/set`.

`create` accepts: `spaceId` (String, required), `userId` (String, required), `reason` (String, optional), `expiresAt` (UTCDate, optional). Requires `"ban"` permission in the Space.

`update` supports: `reason`, `expiresAt`.

`destroy` lifts the ban. Requires `"ban"` permission in the Space.

# Rich Body Format {#rich-body}

This section defines the `application/jmap-chat-rich` body type for structured inline formatting.

When `bodyType` is `"application/jmap-chat-rich"`, `body` MUST contain a JSON-encoded string whose top-level parsed value is an object conforming to this section. Both the `mentions` and `broadcastMentions` arrays on the Message MUST be empty; all mention information (user and broadcast) is carried inline within the spans.

## Span Object

The body object contains a single field:

`spans` (Span[]):
: An ordered array of Span objects representing the message content from left to right.

Each Span has the following fields:

`type` (String):
: One of the defined span types below.

`text` (String):
: The plaintext content of this span. MUST be present on all span types. Clients that do not support a given span type SHOULD render this field as plaintext.

Additional fields are type-specific:

| Type | Additional fields | Meaning |
|---|---|---|
| `"text"` | none | Plain text |
| `"bold"` | none | Bold text |
| `"italic"` | none | Italic text |
| `"bold-italic"` | none | Bold and italic text |
| `"code"` | none | Inline code |
| `"codeblock"` | `lang` (String, optional) | Fenced code block; `lang` is a language hint for syntax highlighting |
| `"blockquote"` | none | Quoted text |
| `"mention"` | `userId` (String) | Per-user @mention; `userId` is the ChatContact.id of the mentioned user |
| `"broadcast"` | `scope` (String) | Broadcast-scope @mention; `scope` is one of `"everyone"`, `"here"`, or `"admins"` (see {{broadcast-mention}}). Servers MUST reject a `"broadcast"` span with an unrecognized `scope` value with `invalidArguments`. |
| `"link"` | `uri` (String) | Hyperlink; `uri` MUST be treated as untrusted |

Servers MUST reject messages containing unknown span types with `invalidArguments`. Clients MUST silently ignore unknown span types and render the `text` field as plaintext, preserving forward compatibility.

## Example

~~~json
{
  "spans": [
    {"type": "text", "text": "Hello "},
    {"type": "mention", "text": "@alice", "userId": "user:alice@example.com"},
    {"type": "text", "text": " and "},
    {"type": "broadcast", "text": "@here", "scope": "here"},
    {"type": "text", "text": ", see this code: "},
    {"type": "code", "text": "foo()"},
    {"type": "text", "text": ". Full example:"},
    {"type": "codeblock", "text": "fn main() {}", "lang": "rust"}
  ]
}
~~~

# Thread Model {#threads}

Servers advertising `supportsThreads: true` support structured conversation threads.

A thread is the set of Messages sharing a common `threadRootId`. The root message has `threadRootId` absent.

Thread root assignment rules:

- A message with no `replyTo` is a potential thread root. `threadRootId` MUST be absent.
- A message replying to a thread root (the referenced message has no `threadRootId`): set `threadRootId` to the value of `replyTo`.
- A message replying to a non-root message: set `threadRootId` to the referenced message's `threadRootId`.

Clients SHOULD follow these rules. Servers SHOULD validate them and MAY correct `threadRootId` if the client supplies an incorrect value. A client that sends an incorrect `threadRootId` — or a server that does not correct one — will produce a message that does not appear in `Message/query` results filtered by the intended thread root, and whose `replyCount` will not be reflected on the intended thread root message. Thread structure is a display feature; incorrect assignment causes broken threading in the UI but does not affect message delivery or data integrity.

`Message/query` with a `threadRootId` filter returns all messages in that thread. The `replyCount` field on each message gives the count of direct replies.

# Server-to-Server Methods

Server-to-server federation methods (`Peer/deliver`, `Peer/receipt`, `Peer/typing`, `Peer/retract`, `Peer/groupUpdate`) are defined in {{JMAP-CHAT-FED}}.

# Push Notifications {#push}

Servers MUST support the EventSource mechanism defined in {{RFC8620}} Section 7.3.

Servers SHOULD also support the push subscription mechanism defined in {{RFC8620}} Section 7.2 for deployments requiring offline and mobile push delivery.

Servers that deliver push subscriptions via Web Push SHOULD also advertise the `urn:ietf:params:jmap:webpush-vapid` capability {{JMAP-WEBPUSH-VAPID}} and authenticate their Web Push messages with a VAPID-signed JWT. VAPID authentication allows push services to verify the server's identity, which is required for reliable authenticated delivery to mobile clients. Servers that use only the EventSource mechanism or a non-Web-Push delivery channel are not subject to this recommendation.

## State-Change Events

~~~
event: state
data: {"@type":"StateChange","changed":{"<accountId>":{"Message":"<s>","Chat":"<s>","ChatContact":"<s>"}}}
~~~

Clients SHOULD call the corresponding `/changes` method upon receipt. On `cannotCalculateChanges`, fall back to `/get`.

## Typing Events

~~~
event: typing
data: {"chatId":"<id>","senderId":"<contact-id>","typing":<bool>}
~~~

Not stored; carries no state token. Before delivering this event to a recipient, the server MUST apply the same `receiveTypingIndicators` and blocked-sender checks described in the `Chat/typing` server behavior ({{chat-typing}}).

## Presence Events

~~~
event: presence
data: {"contactId":"<id>","presence":"<state>","lastActiveAt":"<ts>","statusText":"<string>|null","statusEmoji":"<string>|null"}
~~~

Before delivering this event, the server MUST silently drop it if the identified ChatContact (`contactId`) has `blocked: true` on the receiving owner's contact list. The remote contact is not informed.

The `statusText` and `statusEmoji` fields reflect the contact's current PresenceStatus values; they are `null` when the contact has no active custom status. Clients SHOULD update their displayed status immediately upon receiving this event without waiting for a `/changes` poll.

# Blob Storage {#blobs}

Standard JMAP blob upload and download per {{RFC8620}} Section 6, using the `uploadUrl` and `downloadUrl` Session templates.

Implementations requiring additional blob operations (Blob/get, Blob/copy) SHOULD refer to {{RFC9404}} (JMAP Blob Management Extensions). The more recent {{JMAP-BLOBEXT}} (`urn:ietf:params:jmap:blob2`) obsoletes RFC 9404 and adds substantial new capabilities; servers SHOULD advertise `urn:ietf:params:jmap:blob2` where available.

The upload response defined here includes the `sha256` field as specified
in {{JMAP-CID}}; see that document for the normative definition, client
verification guidance, and security considerations.

Servers SHOULD support `Blob/lookup` (defined in {{JMAP-BLOBEXT}}) and register `"Message"` as a participating data type, enabling clients to perform reverse lookups from a blobId to all Messages that reference it as an attachment.

Servers MAY support `Blob/convert` (defined in {{JMAP-BLOBEXT}}) for image media types, enabling server-side thumbnail generation for chat attachment previews.

## Upload

`POST <uploadUrl>` with the blob as the request body.

Response (HTTP 200) — the standard RFC 8620 upload response, extended per {{JMAP-CID}} with the `sha256` field:

~~~json
{
  "blobId":  "<id>",
  "type":    "<mime-type>",
  "size":    <bytes>,
  "sha256":  "<lowercase-hex>"
}
~~~

## Download

`GET <downloadUrl>` with placeholders percent-encoded.

# Outbox and Delivery {#outbox}

Outbound delivery mechanics for the mailbox-per-user federation topology are defined in {{JMAP-CHAT-FED}}.

# Authorization {#authorization}

Authentication is handled at the transport layer as described in {{introduction}}. The protocol derives access control from the stable, opaque id provided per connection:

- **Owner** (identity equals the mailbox owner's id): all methods.
- **Other**: HTTP 401.

Beyond this transport-level distinction, all Space-internal authorization — the "Requires `X`" clauses on `Space/set`, `Chat/set`, and related methods, and the "Mutable by members with `X` permission" clauses on object fields — is enforced server-side. Clients MUST NOT be trusted to enforce their own access; servers MUST evaluate authorization independently on every request and return `forbidden` to unauthorized callers (see {{space-permissions}}).

The permission vocabulary defined in {{space-role}} appears on the wire in `SpaceRole.permissions` arrays and `SpaceMember.roleIds` references. This exposure is for client-side presentation and pre-flight UX (rendering admin controls, displaying role membership, avoiding obviously-doomed requests), not as a client-enforced security layer. A spec-compliant client MAY ignore the vocabulary entirely and rely on `forbidden` responses; this is functionally equivalent but produces less responsive UI.

Authorization for peer server access in federation deployments is defined in {{JMAP-CHAT-FED}}.

# Space Permission Resolution {#space-permissions}

When determining whether a member may perform an action in a channel Chat, servers MUST evaluate permissions in this order:

1. Compute the union of `permissions` across all SpaceRoles held by the member, including the implicit `@everyone` role.
2. Apply `deny` entries from any ChannelPermission records matching the member's roles (`targetType: "role"`), in ascending position order.
3. Apply `allow` entries from the same role-targeted records, in ascending position order.
4. Apply `deny` entries from any ChannelPermission record matching the member directly (`targetType: "member"`).
5. Apply `allow` entries from the same member-targeted record.

Servers MAY designate one or more controller principals that bypass this resolution and have all permissions implicitly. The mechanism for designating such principals (and for transferring or revoking the designation) is deployment-defined and outside the scope of this specification. Implementations differ on this point; see {{space-deployment}}.

Servers MUST perform this resolution server-side. Clients MUST NOT be trusted to assert their own permissions.

Role hierarchy enforcement: members may only create or modify SpaceRoles whose `position` is strictly less than their own highest-position role. Servers MUST reject role management operations that violate this constraint.

The "Requires `X`" and "Mutable by members with `X` permission" clauses elsewhere in this specification describe the recommended mapping from the closed permission vocabulary defined in {{space-role}} to the actions those permissions gate. Deployments MAY substitute or augment these mappings with deployment-specific authorization policy — for example, gating an action on a different permission, requiring additional authorization beyond merely holding the named permission, or recognizing an internal capability not present in the closed vocabulary. The wire-level contract is that unauthorized callers MUST receive `forbidden`; the permission names defined in {{space-role}} retain their stated meanings when they appear in `SpaceRole.permissions` values returned to clients.

Some MUST constraints in this specification are not subject to this latitude because they define wire-level behaviour rather than server-internal authorization: the sender-only constraints on `Message/set update` and on Reaction creation and removal, the mailbox-owner constraint inherited from {{RFC8620}}, and all input validation requirements in {{security}}.

## Space Governance: Deployment Variation {#space-deployment}

This specification defines the membership, role, and permission mechanisms for Spaces but does not prescribe a single ownership model. Implementations vary:

- Some deployments designate a single irreducible controller per Space (the creator, in the style of Discord, Telegram groups, or Slack workspaces), enforced server-side so that the controller cannot be removed or demoted by other members.
- Other deployments operate without a designated controller, relying entirely on the role-and-permission graph; if administrative coverage is lost (no member holds `"manage_space"`), recovery is out-of-band.
- Enterprise deployments may delegate Space governance to an external identity provider or directory service, with the JMAP Chat server enforcing administrative actions on behalf of that external authority.

Federated deployments SHOULD coordinate their governance policies with peer servers when federating Spaces, since divergence can produce surprising interop behaviour during membership churn (for example, server A may refuse to remove its local controller while server B has no such restriction). This specification does not define a wire-level mechanism for advertising or negotiating a governance policy between peers; deployments that federate at scale are encouraged to publish their policy out-of-band.

Clients SHOULD NOT assume a specific governance model. Where the UI needs to surface "who runs this Space", clients SHOULD derive the answer from the role-and-permission graph (e.g., members holding `"manage_space"`) rather than from a single distinguished principal.

# Security Considerations {#security}

## Identity Verification

`senderUserId` in all Peer/* methods is caller-supplied and MUST be treated as untrusted. The server MUST obtain the verified identity from its own authentication layer independently and MUST compare before any storage or action. Verification MUST precede all effects.

## Input Validation

All peer-supplied fields are attacker-controlled. Servers MUST validate:

- `body`: validate UTF-8 for plaintext body types; enforce `maxBodyBytes`.
- `bodyType`: validate against `supportedBodyTypes`.
- `filename`: reject values containing `/`, `\`, or null bytes.
- `contentType`: reject syntactically invalid MIME values.
- `size`: verify against actual blob byte count after fetch.
- `sha256`: verify against actual blob content after fetch.
- `sentAt`, `editedAt`: store as-is; never use for ordering or expiry.
- `chatId`: verify per {{chat-id}} — confirm match with stored id or verify membership; reject mismatches.
- `emoji`: validate as a non-empty string; enforce an implementation-defined maximum byte length to prevent denial of service.
- `mentions`: reject any entry where `offset + length` exceeds body byte length.
- `broadcastMentions`: reject any entry whose `scope` is not one of `"everyone"`, `"here"`, `"admins"`, and any entry where `offset + length` exceeds body byte length. Additionally enforce the `"mention_broadcast"` permission check defined in {{broadcast-mention}}.

## Denial of Service

Enforce `maxBodyBytes` and `maxAttachmentBytes` at parse time, before any fetch or storage. Enforce `maxAttachmentsPerMessage` at creation and update time. Servers MUST also enforce implementation-defined per-aggregate caps on group Chat membership, Space membership, roles, channels, and categories at `Chat/set` and `Space/set` time, returning `overQuota` SetErrors ({{RFC8620}} §5.3) as specified in those methods. Rate-limit `Peer/typing` per peer.

## Chat ID Integrity

Chat IDs are server-assigned ULIDs. Security against cross-conversation injection relies on sender authentication and chat membership verification, not on ID derivation.

## Broadcast Mention Abuse {#broadcast-mention-abuse}

Broadcast mentions (`@everyone`, `@here`, `@admins`; see {{broadcast-mention}}) elevate push urgency for every member targeted by the receiving server's delivery-time set, and bypass each targeted owner's `Chat.muted` and `Chat.muteUntil` settings ({{chat}}). The blast radius scales with Space or Chat membership, and a single send can wake every active device in a large Space. This makes broadcast mentions a high-leverage target for both abuse and accidental noise.

The `"mention_broadcast"` permission is the spec's primary control: only members with the permission can send a broadcast mention, and servers MUST reject offending `Message/set` requests with `forbidden` ({{broadcast-mention}}). The permission MUST NOT be granted by default to the `@everyone` role; deployments SHOULD scope it to roles that already carry administrative or moderator responsibility, and SHOULD audit grant changes. Rate-limiting broadcast mentions per sender, per Space, and per scope is RECOMMENDED to bound damage when the permission is misused or a sender's credentials are compromised.

The `Chat.muted` bypass for targeted recipients is a deliberate UX choice for the common case (a moderator paging the on-call set during an incident). It is also the most abusable property of the feature. Deployments SHOULD provide owners a way to override the bypass — for example, a "respect mute regardless of broadcast scope" account preference, or a per-Chat preference that disables broadcast elevation for that Chat — and SHOULD make that preference discoverable to owners who report ongoing abuse. Such preferences are deployment-defined; this specification does not assign a wire field for them, and federated peers SHOULD NOT be told whether a remote owner has chosen to suppress broadcast elevation, since that information would let a sender confirm which recipients are silently filtering them.

The send-time recipient set computed by the sending server (the informational list described in {{broadcast-mention}}) MUST NOT be used by a receiving server for authorization or for deciding whether to elevate push: trusting it would let a hostile or buggy sending server force elevation against recipients who would not otherwise qualify under the receiving server's own predicate. Each receiving server resolves its own set and acts on its own resolution.

## Blocked Contacts

Messages from a ChatContact whose `blocked` field is `true` are silently dropped regardless of whether they arrive in a direct chat or a group chat context.

Typing and presence push events whose sending or referenced ChatContact has `blocked: true` on the receiving owner's contact list are similarly dropped server-side before delivery to any of the owner's clients (see the `Chat/typing` server behavior in {{chat-typing}}, the typing and presence event sections in {{push}}, and the analogous rules in {{JMAP-CHAT-WSS}} and {{JMAP-CHAT-FED}}). The sender is not informed. This prevents a blocked user from leaking presence or attention patterns on any transport even though their messages are dropped.

When `Chat.receiveTypingIndicators` is `false`, typing push events for that Chat are suppressed server-side (see `receiveTypingIndicators` in {{chat}} and the `Chat/typing` server behavior in {{chat-typing}}) before delivery to the owner. The sender is not informed; `Chat/typing` succeeds normally. This prevents a sender from inferring that typing indicators are being suppressed.

## Read Receipt Privacy

When the effective `receiptSharing` preference is `false`, the server suppresses outbound and inbound read-receipt events as specified in `PresenceStatus.receiptSharing` ({{presence-status}}), `Chat.receiptSharing` ({{chat}}), and the `Peer/receipt` Sender and Receiver Behavior in {{JMAP-CHAT-FED}}. This prevents remote peers from inferring when the owner reads messages.

Clients SHOULD NOT expose sub-minute precision for `readAt`, `lastReadAt`, or `deviceDeliveredAt` timestamps in the user interface. Displaying relative representations (such as "today", "yesterday", or hour-granularity) reduces the risk of exposing behavioral patterns through precise read timing. The underlying UTCDate values retain full precision for protocol purposes; coarsening is a presentation concern only.

## Out-of-Band Endpoints and Actions

`ChatContact.endpoints`, `Session.ownerEndpoints`, and `Message.actions` carry peer-supplied URIs and MUST be treated as untrusted at every level:

- Clients MUST NOT fetch or connect to any OOB URI automatically. All OOB interactions require explicit user initiation.
- Payment URIs (`urn:jmap:chat:cap:payment`) MUST be validated by the client wallet before any funds are transferred. Servers MUST NOT inspect or act on payment URI values.
- VTC URIs (`urn:jmap:chat:cap:vtc`) MUST NOT be opened without user consent; auto-joining a call is a privacy violation.
- Blob/file URIs (`urn:jmap:chat:cap:blob`) used for OOB fetch are an SSRF vector; servers that fetch from peer-supplied blob endpoints MUST restrict connections to the known peer address space.
- `metadata` values are peer-supplied and MUST be ignored if they do not conform to the expected shape for the known `type`.
- Unknown `type` URIs SHOULD be silently ignored by both clients and servers.

## End-to-End Encrypted Deployments {#e2ee}

In relay deployments, the relay routes Peer/* messages but MUST NOT have access to plaintext message content. Implementations MUST ensure:

- The `body` field carries ciphertext only; plaintext MUST never be transmitted to the relay in an encrypted deployment.
- The relay is architecturally excluded from the encryption key schedule (e.g., by using MLS {{RFC9420}} or a similar protocol that does not involve the relay in key agreement).
- Servers MUST NOT reject or transform `body` based on content when `bodyType` indicates an encrypted type.
- Metadata visible to the relay — sender id, recipient id, timestamp, and body size — remains an information-leakage surface. Deployments requiring metadata privacy SHOULD apply message padding and cover traffic at the transport layer; those techniques are outside the scope of this document.

## Federation Security

Security considerations for server-to-server federation are defined in {{JMAP-CHAT-FED}}.

# IANA Considerations

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.


## JMAP Data Types Registration

IANA is requested to register the following entries in the "JMAP Data Types" registry:

Type Name:
: ChatContact

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: Chat

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: Message

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: Space

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: SpaceInvite

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: CustomEmoji

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: SpaceBan

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: ReadPosition

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

Type Name:
: PresenceStatus

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat`

Reference:
: This document

--- back

# Acknowledgements

The author thanks the JMAP working group for {{RFC8620}}.
