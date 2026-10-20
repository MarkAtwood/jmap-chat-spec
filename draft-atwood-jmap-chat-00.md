---
title: JMAP for Chat
abbrev: JMAP Chat
docname: draft-atwood-jmap-chat-00
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
  RFC5234:
  RFC3629:
  RFC4648:
  RFC8292:
  RFC9749:
  ULID:
    title: "Universally Unique Lexicographically Sortable Identifier"
    target: https://github.com/ulid/spec
  JMAP-CID:
    title: "JMAP Blob Content Identifiers"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-cid-00
    date: 2026

informative:
  RFC8141:
  RFC7763:
  RFC7764:
  RFC8621:
  RFC9404:
  RFC9420:
  RFC9425:
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) message content"
    author:
      - fullname: Rohan Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-content-09
    date: 2026-07-04
  MIMI-PROTOCOL:
    title: "More Instant Messaging Interoperability (MIMI) using HTTPS and MLS"
    author:
      - fullname: Richard Barnes
      - fullname: Matthew Hodgson
      - fullname: Konrad Kohbrok
      - fullname: Rohan Mahy
      - fullname: Travis Ralston
      - fullname: Raphael Robert
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-protocol-06
    date: 2026-04-25
  JMAP-CHAT-MIMI:
    title: "JMAP Chat MIMI Mapping"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-mimi-00
    date: 2026
  JMAP-CHAT-FED:
    title: "JMAP Chat Federation"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026
  JMAP-CHAT-WSS:
    title: "JMAP Chat over WebSocket"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-OBJ-HISTORY:
    title: "JMAP Object History"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-object-history-02
    date: 2026-10-16
  RFC9610:
  RFC9670:
  JMAP-FILENODE:
    title: "JMAP File Storage extension"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-filenode-17
    date: 2026-10-18
  JMAP-CHAT-FILENODE:
    title: "JMAP Chat File Storage"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-filenode-00
    date: 2026
  JMAP-BLOBEXT:
    title: "JMAP Blob Management"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-blobext-02
    date: 2026-10-14
  JMAP-VTC:
    title: "JMAP for Video/Voice Teleconferencing"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  JMAP-SCENE-WSS:
    title: "JMAP Scene over WebSocket"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-wss-00
    date: 2026
  JMAP-METADATA:
    title: "JMAP Object Metadata"
    author:
      - fullname: Mauro De Gennaro
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-metadata-04
    date: 2026-10-19
  JMAP-REFPLUS:
    title: "JMAP Enhanced Result References"
    author:
      - fullname: Mauro De Gennaro
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-refplus-02
    date: 2026-06-21
  JMAP-CALENDARS:
    title: "JSON Meta Application Protocol (JMAP) for Calendars"
    author:
      - fullname: Neil Jenkins
      - fullname: Michael Douglass
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-calendars-33
    date: 2026-10-13
  JMAP-TASKS:
    title: "JMAP for Tasks"
    author:
      - fullname: Joris Baum
      - fullname: Hans-Jörg Happel
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-tasks-06
    date: 2023-03-10
    annotation: "Work in progress; this draft expired on 2023-09-11 and has no successor."
  JMAP-CHAT-CALENDARS:
    title: "JMAP Chat Calendars"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-calendars-00
    date: 2026
  JMAP-CHAT-TASKS:
    title: "JMAP Chat Tasks"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-tasks-00
    date: 2026
  DID-CORE:
    title: "Decentralized Identifiers (DIDs) v1.0"
    author:
      - fullname: Manu Sporny
        role: editor
      - fullname: Amy Guy
        role: editor
      - fullname: Markus Sabadello
        role: editor
      - fullname: Drummond Reed
        role: editor
    seriesinfo:
      W3C: REC-did-core-20220719
    date: 2022-07-19
    target: https://www.w3.org/TR/2022/REC-did-core-20220719/
  DTCG-COLOR:
    title: "Design Tokens Color Module 2025.10"
    author:
      - fullname: Ayesha Mazrana (Mazumdar)
        ins: A. Mazrana (Mazumdar)
        role: editor
      - fullname: Kathleen McMahon
        role: editor
      - fullname: Adekunle Oduye
        role: editor
      - fullname: Matthew Ström-Awn
        role: editor
    date: 2025-10-28
    target: https://www.designtokens.org/TR/2025.10/color/
    annotation: "W3C Design Tokens Community Group Final Community Group Report."
  CSS-COLOR-4:
    title: "CSS Color Module Level 4"
    author:
      - fullname: Tab Atkins Jr.
        role: editor
      - fullname: Chris Lilley
        role: editor
      - fullname: Lea Verou
        role: editor
    seriesinfo:
      W3C: CRD-css-color-4-20261020
    date: 2026-10-20
    target: https://www.w3.org/TR/2026/CRD-css-color-4-20261020/
  CAM16:
    title: "Comprehensive color solutions: CAM16, CAT16, and CAM16-UCS"
    author:
      - fullname: Changjun Li
      - fullname: Zhiqiang Li
      - fullname: Zhifeng Wang
      - fullname: Yang Xu
      - fullname: Ming Ronnier Luo
      - fullname: Guihua Cui
      - fullname: Manuel Melgosa
      - fullname: Michael H. Brill
      - fullname: Michael Pointer
    seriesinfo:
      DOI: 10.1002/col.22131
    refcontent: "Color Research and Application, vol. 42, no. 6, pp. 703-718"
    date: 2017

--- abstract

This document defines JMAP for Chat, a JMAP capability ({{RFC8620}}) for direct and group text messaging. It supports two deployment topologies: a mailbox-per-user model in which each user operates their own server, and a relay model in which a shared server routes end-to-end encrypted messages.

The specification defines the `urn:jmap:chat` capability; twenty-one data types (Attachment, Endpoint, MessageAction, Mention, BroadcastMention, MessageRevision, Reaction, ChatContact, ChatMember, Chat, Message, SpaceRole, SpaceMember, Category, ChannelPermission, Space, SpaceInvite, CustomEmoji, SpaceBan, ReadPosition, and PresenceStatus); and JMAP owner methods for each top-level type. Server-to-server federation methods are defined in {{JMAP-CHAT-FED}}.

The protocol covers the feature set common to contemporary messaging systems: group chat with membership roles, message reactions, editing, deletion, threading, @mentions, typing indicators, read receipts per participant, presence, pinned messages, per-chat notification settings, sender-controlled message expiry, and burn-on-read. Spaces provide a named multi-channel container with a role-based permission system, analogous to what other systems call a server, workspace, or team.


--- middle

# Introduction

JMAP {{RFC8620}} defines a JSON-based protocol for accessing and mutating application data. The core protocol is intentionally generic; application semantics are expressed through capability URIs declared in the JMAP Session object. {{RFC8621}} defines JMAP for Mail. This document defines an analogous capability for real-time chat.

## Deployment Topologies

This specification accommodates two primary deployment topologies.

In the **mailbox-per-user** model, each participant runs their own JMAP server storing only their own messages. Mailboxes exchange messages directly using the server-to-server methods defined in {{JMAP-CHAT-FED}}.

In the **relay** model, a shared server routes messages between clients. The Peer/* methods are implemented by the relay rather than by individual user-controlled mailboxes. Relay deployments are designed to handle only opaque ciphertext plus the routing fields listed in {{e2ee}}; see {{e2ee}} for the normative requirements.

Both topologies are fully compatible with this specification. Transport, identity, and encryption choices are confined to the deployment layer.

## Authentication Model

Authentication is handled entirely at the transport layer. The protocol requires only that the authentication layer provide a stable, opaque user identity string for each connection. How that identity is established — overlay network membership, mutual TLS, bearer tokens, or any other mechanism — is outside the scope of this document. Authorization rules derived from this identity are defined in {{authorization}}.

In the mailbox-per-user deployment topology, the peer authentication model is defined in {{JMAP-CHAT-FED}}.

## Relationship to MIMI

The IETF MIMI (More Instant Messaging Interoperability) working group {{MIMI-PROTOCOL}} {{MIMI-CONTENT}} is developing a separate approach to messaging interoperability, primarily targeting provider-to-provider federation between large existing messaging platforms under regulatory interoperability mandates. MIMI's client-server API layer is intentionally outside the MIMI charter scope; this document fills that gap for JMAP-based deployments.

The two specifications are complementary rather than competing. MIMI defines `application/mimi-content` {{MIMI-CONTENT}}, a CBOR-encoded message body format designed to operate as an MLS PrivateMessage payload. A JMAP Chat server that also participates in a MIMI federation domain MAY include `"application/mimi-content"` in its `supportedBodyTypes` capability and accept MIMI-formatted message bodies in `Message/set` and `Peer/deliver`. This document does not require or preclude such interoperability. {{JMAP-CHAT-MIMI}} defines how a gateway converts MIMI messages to JMAP Chat Messages, including message identifiers, edits, topics, media reactions, and message statuses.

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

identity / id:
: A user's identity is the stable, opaque identity string the authentication layer provides for that user, recorded in the `identity` property of the user's ChatContact ({{chat-contact}}). Its form (for example, a `user@host`-style string, a Decentralized Identifier URI {{DID-CORE}}, or any other URI form) is not constrained by this specification; servers MUST treat it as opaque regardless of form, except that peer authentication in {{JMAP-CHAT-FED}} extracts the host of an identity to bind it to a verified peer domain. A ChatContact's `id` is an Id ({{RFC8620}} Section 1.2) that the server assigns; it is local to that server. Every reference to a user that a client sends or receives carries the `id` of the user's ChatContact on the client's server. Every reference to a user that passes between servers ({{JMAP-CHAT-FED}}) carries the user's identity; the receiving server maps it to its own ChatContact with that `identity`. The owner has no ChatContact: a client-facing reference to the owner carries the accountId of the owner's account ({{RFC8620}} Section 1.6.2), including where this document describes the reference as a ChatContact.id, and no ChatContact has that accountId as its `id`. Between servers, the owner's identity is the `ownerUserId` Session field; a server maps its `ownerUserId` to its accountId on receipt and its accountId to its `ownerUserId` when sending.

## Color Representation {#color-convention}

Where this specification or companion specifications define color-valued fields, implementations SHOULD represent colors using the W3C Design Tokens Community Group color token format {{DTCG-COLOR}} (a JSON object with `"$type": "color"` and a `"$value"` that is a Design Tokens color object with `colorSpace` and `components` members and optional `alpha` and `hex` members, not a CSS color string; `colorSpace` names a color space defined in CSS Color Level 4 {{CSS-COLOR-4}}). For example: `{"$type": "color", "$value": {"colorSpace": "srgb", "components": [0.345, 0.396, 0.949], "hex": "#5865f2"}}`. When the Design Tokens format is not practical, implementations SHOULD use CAM16 uniform color space coordinates {{CAM16}} (`[J, a, b]` or `[J, M, h]`). As a baseline fallback, implementations MAY use sRGB hex strings (e.g., `"#5865f2"`). Servers MUST accept all three representations in color-valued fields and MUST preserve the representation provided by the client. Clients MUST handle all three representations gracefully.

# The urn:jmap:chat Capability {#capability}

The `urn:jmap:chat` capability is advertised in the JMAP Session object at both the top-level `capabilities` key and within each account's `accountCapabilities` map.

## Session-Level Capability Object

The value of `capabilities["urn:jmap:chat"]` at the session level is an empty object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:jmap:chat"]` is a JSON object with the following fields:

`maxBodyBytes` (UnsignedInt):
: Maximum UTF-8 {{RFC3629}} byte length of a Message `body`. Servers MUST reject messages exceeding this limit; in `Message/set` the rejection is a `tooLarge` SetError.

`maxAttachmentBytes` (UnsignedInt):
: Maximum size in bytes of a single attachment blob.

`maxAttachmentsPerMessage` (UnsignedInt):
: Maximum number of attachments per message.

`supportedBodyTypes` (String[]):
: The MIME types a Message `bodyType` can take on this account. The server MUST accept in `Message/set` create every listed type except types a companion specification reserves for server-created Messages. MUST include `"text/plain"` unless the server accepts only end-to-end encrypted body types ({{e2ee-fields}}). End-to-end encrypted deployments SHOULD also include an appropriate encrypted-content type such as `"application/mls-ciphertext"`. Servers SHOULD support `"text/markdown"` (the CommonMark variant, {{RFC7763}} and {{RFC7764}} Section 3.5) and `"application/jmap-chat-rich"` (defined in {{rich-body}}). Servers participating in MIMI federation domains MAY also include `"application/mimi-content"` {{MIMI-CONTENT}}.

`supportsThreads` (Boolean):
: Whether this server supports the optional thread model defined in {{threads}}.

The `role` field used to distinguish owner and peer accounts in federation deployments is defined in {{JMAP-CHAT-FED}}.

Note: {{RFC9425}} defines a `Quota` data type for exposing account-level storage and object-count limits to clients. Servers SHOULD implement the JMAP Quota extension and include relevant chat data types — for example `"Message"` (storage), `"Chat"` (count of group and direct chats), or `"Space"` (count of Spaces) — in the `types` field of relevant Quota records. JMAP Quotas express account-level (or domain/global) limits; servers additionally enforce per-aggregate limits (members per group Chat, members/roles/channels/categories per Space) by returning an `overQuota` SetError ({{RFC8620}} §5.3) on the relevant `Chat/set` or `Space/set` entry at the time the limit would be exceeded.

## Session Object Extensions

When the `urn:jmap:chat` capability is present, servers MAY include the following additional fields in the Session object:

`ownerUserId` (String):
: The identity of the mailbox owner (equals the `identity` of the owner's ChatContact on any peer server that has recorded this mailbox as a contact). Servers implementing {{JMAP-CHAT-FED}} MUST include this field; other servers MAY include it.

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
: Original filename. SHOULD NOT contain `/`, `\`, or null bytes.

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
: The endpoint URI. Format is type-specific. Peer-supplied; SHOULD be treated as untrusted.

`label` (String, optional):
: Human-readable label for this endpoint (e.g., `"Personal Jitsi"`, `"Zcash address"`).

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys. Examples by type:

  - `vtc`: `{"protocol": "webrtc", "callId": "...", "joinPassword": "..."}`
  - `payment`: `{"network": "lightning", "currency": "BTC"}`
  - `blob`: `{"maxBytes": 10485760}`
  - `calendar-event`: `{"startsAt": "...", "endsAt": "..."}`
  - `task`: `{"title": "...", "progress": "..."}`
  - `filenode`: `{"name": "...", "mediaType": "..."}`

## MessageAction {#message-action}

A MessageAction is a per-message out-of-band action invitation carried in a Message and in `Peer/deliver`. It signals that a message is associated with an out-of-band interaction — a video call invitation, a payment request, a file available outside the blob channel, etc. Servers MUST store and forward MessageAction records without inspection or transformation, except that a server implementing {{JMAP-CHAT-FED}} maps any user reference that this or a companion specification defines in `metadata` as that document specifies. Clients SHOULD NOT act on a MessageAction automatically; all OOB actions require explicit user initiation.

`type` (String):
: Same URI namespace as Endpoint `type` (see {{endpoint}}). Clients MUST ignore actions whose `type` they do not recognize.

`uri` (String):
: The action URI. Peer-supplied; SHOULD be treated as untrusted.

`label` (String, optional):
: Human-readable label for the action (e.g., `"Join call"`, `"Pay $5"`, `"Download file"`).

`expiresAt` (UTCDate, optional):
: Time after which the action is no longer valid. Clients can visually indicate expired actions. Servers MUST NOT enforce expiry on stored actions; enforcement is the OOB system's responsibility.

`metadata` (Object, optional):
: Type-specific key-value pairs. Clients MUST ignore unknown keys.

## Mention {#mention}

A Mention names a user that a message mentions.

`id` (String):
: The `id` of the mentioned participant's ChatContact, or the accountId when the mentioned participant is the owner.

A Mention carries no position in `body`. Where the mention appears, and how it is rendered, is presentation: a rich body marks it with a `"mention"` span ({{rich-body}}), and for other body types clients locate the textual form themselves. The `mentions` array is authoritative for every `bodyType`: servers MUST take mentions only from it, for mention-based push, notification, and query behavior, and MUST NOT derive mentions from `body`.

The `id` field carries the mentioned user's ChatContact `id`, or the accountId for the owner, regardless of textual form; between servers, a Mention carries the mentioned user's identity instead ({{JMAP-CHAT-FED}}). Common composer-side textual forms include the Mastodon-style `@user@host` for federated users and DID URI forms (for example, `@did:web:alice.example` per {{DID-CORE}}) when DID-based identity is in use; the wire format is unaffected by which textual form the sender's client recognized. Parsing the textual form into a candidate identity, and resolving that candidate to a known ChatContact, are deployment-defined. This does not apply to the `userId` of a rich-body `"mention"` span, which is display data that servers never map ({{rich-body}}).

The Mention type carries per-user mentions only. Broadcast-scope mentions (such as `@everyone`, `@here`, and `@admins`) are carried in the separate `broadcastMentions` array on Message; see {{broadcast-mention}}.

## BroadcastMention {#broadcast-mention}

A BroadcastMention names a broadcast-scope mention in a message. Unlike Mention, a BroadcastMention does not name a specific participant; it names a member set whose composition is resolved at delivery time by each receiving server.

`scope` (String):
: One of:
  - `"everyone"` — all current members of the Space or Chat
  - `"here"` — currently-active members; the predicate that defines "active" (which `PresenceStatus.availability` states qualify, what activity window applies) is deployment-defined
  - `"admins"` — members holding administrative authority in the Space or Chat; the predicate that defines "admin" (which SpaceRole permissions qualify, whether controller principals from {{space-deployment}} are included) is deployment-defined

  Servers MUST reject a BroadcastMention with an unrecognized `scope` value; in `Message/set` the rejection is an `invalidProperties` SetError naming `broadcastMentions`.

Like Mention, a BroadcastMention carries no position in `body`. The `broadcastMentions` array is authoritative for every `bodyType`: servers MUST take broadcast mentions only from it and MUST NOT derive them from `body`.

The canonical textual forms presented by composers are `@everyone`, `@here`, and `@admins` respectively. Deployments MAY recognize additional aliases (for example, `@all` as a synonym for `@everyone`) in their composer UI; alias-to-scope resolution is a client-side or deployment-side concern and is not part of the wire contract. The wire `scope` value is fixed to one of the three enumerations above.

The set of recipients targeted by a BroadcastMention is computed at delivery time on each receiving server, against that server's local view of membership, presence, and administrative permissions. The sending server MAY compute its own set at send time for sender-side UX (for example, displaying "you mentioned N people") but its send-time set is informational only; the receiving server's delivery-time set is authoritative for push urgency, notification routing, and badging. Where the sending and receiving servers reside on different deployments and disagree on which members qualify for `"here"` or `"admins"`, each receiving server's view governs delivery on its own owners' accounts.

Sending a Message with a non-empty `broadcastMentions` array requires the `"mention_broadcast"` Space permission. Servers MUST reject `Message/set` create or update requests that would write a broadcast mention from a sender lacking this permission with `forbidden`. For group chats that are not part of any Space (and thus have no SpaceRole permission graph), broadcast mentions are unrestricted; the `"mention_broadcast"` permission check applies only to channel chats within a Space.

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
: The accountId for the owner's reaction, or the reacting user's ChatContact.id.

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

A ChatContact represents a remote user known to this mailbox.

`id` (Id, immutable, server-set):
: The server-assigned identifier of this ChatContact ({{RFC8620}} Section 1.2). It identifies the ChatContact only on this server; another server holds a different `id` for the same user. How the server chooses it is implementation-defined, except that the server MUST NOT assign the accountId of the owner's account.

`identity` (String, immutable):
: The identity string the authentication layer provides for this user, for example a `user@host`-style string, a Decentralized Identifier URI {{DID-CORE}}, or another URI. It is set by the server when the server records the contact automatically, or by the owner in `ChatContact/set` create. Servers MUST treat it as opaque and compare it octet for octet. The one exception is peer authentication in {{JMAP-CHAT-FED}}, which extracts the host of an identity to bind it to a verified peer domain; for every other purpose the identity stays opaque. Within a server, at most one ChatContact has a given `identity`.

`login` (String, server-set):
: A non-empty human-readable identifier for this contact, suitable for display when `displayName` is absent. The format is deployment-specific but MUST be a valid UTF-8 string of at least one non-whitespace character. Clients SHOULD fall back to `login` when `displayName` is absent or empty, and MAY fall back to `identity` when `login` is unavailable.

`displayName` (String, optional):
: Human-readable display name. MAY be absent or empty. Set by the owner via `ChatContact/set`. Owner-side preference; not shared with peers.

`firstSeenAt` (UTCDate, server-set):
: Time this ChatContact was first recorded.

`lastSeenAt` (UTCDate, server-set):
: Time of most recent interaction with this ChatContact's mailbox.

`presence` (String, optional, server-set):
: Last known presence state: `"online"`, `"away"`, `"busy"`, or `"offline"`. A contact whose own presence is `"invisible"` appears as `"offline"`. Updated on a best-effort basis. If absent, the contact's presence state is unknown to this server.

`lastActiveAt` (UTCDate, optional, server-set):
: Time the ChatContact was last observed to be active.

`statusText` (String, optional, server-set):
: The contact's current custom status message. Mirrors `PresenceStatus.statusText` on the contact's own server; updated when a presence event is received from that server. Absent when the contact has no active custom status text.

`statusEmoji` (String, optional, server-set):
: The contact's current status emoji. Mirrors `PresenceStatus.statusEmoji` on the contact's own server; updated when a presence event is received from that server. Absent when the contact has no active status emoji.

`endpoints` (Endpoint[], optional):
: Out-of-band capability endpoints advertised by this ChatContact. Servers populate this field from the ChatContact's `ownerEndpoints` at `/.well-known/jmap`. Clients MAY use these to initiate video calls, send payments, or fetch files outside the JMAP message channel. Clients SHOULD NOT act on these values automatically without explicit user intent.

`blocked` (Boolean):
: When `true`, messages from this ChatContact are silently dropped by this mailbox, including messages arriving in group chats. Default: `false`.

Additional ChatContact fields used in federation deployments (`serverUrl`, `directAddress`) are defined in {{JMAP-CHAT-FED}}.

Note: {{RFC9610}} defines `ContactCard` (a JSContact record) for storing rich contact information in a JMAP address book. `ChatContact` and `ContactCard` serve different purposes. Both `ChatContact.id` and `ContactCard.id` are server-assigned Ids, but only a ChatContact records the identity the authentication layer provides for its user (`ChatContact.identity`). Implementations MAY surface `ChatContact` records as `ContactCard` objects in a user's address book as a display-layer mapping; this is not a protocol requirement.

## ChatMember {#chat-member}

A ChatMember describes one participant in a group Chat. The `id` field is the participant's ChatContact.id, or the accountId for the owner.

`id` (String):
: The participant's ChatContact.id, or the accountId for the owner.

`role` (String):
: Either `"admin"` or `"member"`. Admins may add and remove members and update group chat metadata. The bootstrap-role assignment at chat creation — whether and how the creator initially receives the `"admin"` role — is server-defined.

`joinedAt` (UTCDate):
: Time this participant joined the chat.

`invitedBy` (String, optional):
: The ChatContact.id of the member who added this participant, or the accountId when the owner added them.

The `"admin" | "member"` role enum is the wire-observable representation of group-chat authority and is what remote peers see via {{JMAP-CHAT-FED}}. Servers MAY designate additional internal principals (server administrators, members of a dedicated moderator role, automated maintenance systems, or principals delegated by an external identity provider) as having admin-equivalent authority for the actions admins may perform; the means of designating such principals is deployment-defined. From a remote peer's perspective, all such administrative actions appear to originate from a member with the `"admin"` role on the originating server.

## Chat {#chat}

A Chat is a conversation between two or more participants. Three kinds are defined: `"direct"` (one-to-one), `"group"` (multi-party), and `"channel"` (a channel within a Space). Fields whose applicability is restricted to one or two kinds are labeled accordingly; unlabeled fields apply to all kinds.

### Chat ID Assignment {#chat-id}

All Chat IDs are ULIDs {{ULID}} assigned by the creating server at the moment the chat is created. IDs are opaque and stable for the lifetime of the chat.

For a **direct chat**, the creating server is the one whose owner sends the first message. Before assigning a new chatId, the server MUST check whether a direct chat with the relevant contactId already exists locally. If one exists, the server MUST use the existing chatId rather than creating a new one. When a `Peer/deliver` arrives for a direct chat with an unknown chatId, the receiving server creates a new Chat record with that chatId and sets `contactId` to the `id` of the sender's ChatContact.

For a **group chat**, the creating server assigns the chatId and distributes it to all initial members via `Peer/groupUpdate` ({{JMAP-CHAT-FED}}) before any messages are sent.

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

`closedAt` (UTCDate, optional, server-set):
: **Group chats only.** Time the group closed. Absent while the group is open. A group closes when its last admin leaves while other members remain (see `removeMembers` under Chat/set). Once set, `closedAt` MUST NOT be cleared. After it is set, the server MUST reject with a `forbidden` SetError every `Message/set` create in this Chat and every `Chat/set` update that changes membership (`addMembers`, `removeMembers`, `updateMemberRoles`) or shared group settings (`name`, `description`, `avatarBlobId`, `pinnedMessageIds`). Message history remains readable. Owner-side preferences (`muted`, `muteUntil`, `receiveTypingIndicators`, `messageExpirySeconds`, `receiptSharing`) remain updatable, and members MAY still destroy the Chat via `Chat/set` to remove it from their view.

`spaceId` (String, immutable):
: **Channel Chats only.** The id of the containing Space.

`categoryId` (String, optional):
: **Channel Chats only.** The Category id within the Space. Absent if this channel is uncategorized.

`position` (UnsignedInt, optional):
: **Channel Chats only.** Sort order within the category (or among uncategorized channels). Lower values appear first.

`topic` (String, optional):
: **Channel Chats only.** Short description shown in the channel header. Mutable by members with `"manage_channels"` permission.

`slowModeSeconds` (UnsignedInt):
: **Channel Chats only.** When non-zero, the server MUST reject messages from non-exempt members sent within this many seconds of their previous message in this channel, with a `rateLimit` SetError ({{RFC8620, Section 5.3}}) on the offending `Message/set` create. Servers SHOULD exempt members holding the `"manage_channels"` permission, since channel managers are the principals who configure `slowModeSeconds` in the first place and should not be rate-limited by their own settings. Deployments MAY define additional exempt principals (for example, server administrators, members of dedicated moderator roles, or specific automated systems); the wire contract is only that the exempt decision is server-side and that non-exempt senders receive `rateLimit` when they exceed the rate. Clients can display a countdown to the next permitted send time. Clients MAY simply attempt the send and react to `rateLimit`; this is functionally equivalent and avoids the client needing to know its own exempt status. Default: `0`.

`permissionOverrides` (ChannelPermission[]):
: **Channel Chats only.** Per-channel permission overrides for specific roles or members. Evaluated after Space-level role permissions per {{space-permissions}}. Only members with `"manage_channels"` permission may modify this list. Empty by default.

`createdAt` (UTCDate, immutable, server-set):
: Time this chat was first recorded on this mailbox.

`lastMessageAt` (UTCDate, optional, server-set):
: Received time of the most recent message.

`unreadCount` (UnsignedInt, server-set):
: Count of Messages in this Chat whose `id` ULID is lexicographically greater than `ReadPosition.lastReadMessageId`. Derived server-side from the owner's ReadPosition record. If `lastReadMessageId` is absent, all messages are counted as unread.

`pinnedMessageIds` (String[]):
: Ordered list of pinned Message ids, most-recently-pinned first. For group chats, only admins may modify this list. For direct chats, the owner may modify it freely. For channel chats, only members with the `"pin"` Space permission may modify this list. Each id MUST be the id of a Message in this Chat; the server MUST reject a `Chat/set` update that would add any other id with `invalidProperties`. When a pinned Message is hard-deleted, the server MUST remove its id from this list. Empty by default.

`muted` (Boolean):
: When `true`, push notifications for this chat are suppressed. Owner-side preference; not shared with peers. Default: `false`.

  Broadcast mentions ({{broadcast-mention}}) targeting the owner are an exception: when the owner is in the recipient set computed by the receiving server at delivery time, the server SHOULD elevate the push for that message regardless of `muted` or any active `muteUntil` window. Deployments MAY offer the owner a per-account or per-Chat preference to override this elevation (for example, "always honor mute"); the wire contract does not define a field for that preference, and any such preference is deployment-defined. See {{broadcast-mention-abuse}} for the security implications.

`muteUntil` (UTCDate, optional):
: Muting expires at this time. Servers SHOULD clear `muted` and `muteUntil` automatically when the time passes.

`receiveTypingIndicators` (Boolean):
: When `false`, the server MUST silently drop any typing push event (see {{push}}) for this Chat before delivering it to the owner. The sender is not informed; `Chat/typing` succeeds normally from the sender's perspective. Applies to direct and group chats only; for channel chats, servers MUST store the value if set via `Chat/set` but MUST NOT use it to suppress typing events. Default: `true`. Owner-side preference; not shared with peers.

`receiptSharing` (Boolean, optional):
: Per-Chat override of the account-level `PresenceStatus.receiptSharing` preference (see {{presence-status}}). When present, this value takes precedence over the account-level setting for this Chat only. When absent, the account-level preference applies. Owner-side preference; not shared with peers.

`activeCallId` (String|null, server-set):
: When `urn:jmap:vtc` ({{JMAP-VTC}}) is present: the id of a VTCCall that is currently active and bound to this Chat via the VTCCall's `chatId` field. The server sets this when a bound VTCCall enters the `"active"` state and clears it to `null` when the call ends. `null` when no call is active or VTC is not available. Clients MAY use this to display a "join call" banner.

  When a VTCCall with a `chatId` matching this Chat's `id` transitions to state `"active"`, the server MUST set this field to that VTCCall's `id` and emit a `StateChange` for the Chat type. When that VTCCall transitions to state `"ended"`, the server MUST clear this field to `null` and emit a `StateChange`. These updates are side effects of the VTCCall state transition and MUST be applied atomically with it from the client's perspective — a client polling `/changes` after receiving the VTCCall `StateChange` MUST see the corresponding Chat `activeCallId` update.

  If `activeCallId` references a VTCCall whose state is `"ended"`, the server SHOULD clear it to `null` on next read (lazy repair). Clients encountering an `activeCallId` that resolves to an ended or nonexistent VTCCall SHOULD treat it as `null` for UI purposes. Clients cannot clear the field themselves: `activeCallId` is not a `Chat/set` patch key.

  The server MUST NOT deliver a `StateChange` for the Chat type with an updated `activeCallId` until the corresponding VTCCall state transition has also been committed. If the VTCCall `StateChange` and Chat `StateChange` are delivered separately, the VTCCall `StateChange` MUST be delivered first or simultaneously.

  Servers SHOULD coalesce rapid `activeCallId` transitions within a short window (e.g., 1-2 seconds) before emitting a Chat `StateChange`. This is particularly important for high-membership Chats where the fan-out cost of each `StateChange` is proportional to the subscriber count.

  When a VTCCall ends, clients subscribed to both VTCCall and Chat state changes will receive two `StateChange` notifications for the same logical event (the call ending and `activeCallId` clearing). Clients SHOULD deduplicate by correlating the VTCCall id with the Chat's previous `activeCallId` value within a 5-second window.

`activeCallAccountId` (Id|null, server-set):
: The id of the account hosting the VTCCall named by `activeCallId`, so a client can join a call held in another account. The server sets and clears this field together with `activeCallId`, in the same state change: it is non-null exactly when `activeCallId` is non-null. Like `activeCallId`, it is not a `Chat/set` patch key.

`messageExpirySeconds` (UnsignedInt, optional):
: A local expiry policy. When set and non-zero, messages in this chat older than this many seconds are deleted by this mailbox. Each mailbox enforces its own policy independently. This is a local setting, not a bilateral negotiated commitment.

Note: {{JMAP-METADATA}} defines per-type `metadata` and `privateMetadata` properties for data types declared in the `urn:ietf:params:jmap:metadata` capability's `dataTypes` map. Implementations MAY include `Chat` in that map, enabling use cases such as per-chat color coding or custom labels readable and writable via `Chat/get` and `Chat/set` without modifying the shared chat record.

## Message {#message}

A Message is a single transmission within a Chat.

### Message IDs {#message-ids}

Message IDs are ULIDs {{ULID}}, assigned by the **receiving** mailbox at storage time. ULIDs are lexicographically ordered by time, enabling ordered retrieval without a separate sort field.

The **sender-assigned ULID** (`senderMsgId`) is set by the originating mailbox and carried in `Peer/deliver`. The receiving mailbox stores both its own `id` and the `senderMsgId`. Servers MUST be able to resolve any stored `senderMsgId` to its Message within the chat, for idempotent delivery, `Peer/retract` lookup, and resolution of `replyTo` / `threadRootId` references. If a `senderMsgId` is seen again for the same chat, the server MAY silently discard the duplicate.

`senderMsgId` is the cross-server reference for a message. Receiver-assigned `id` values differ between mailboxes, so on the federation wire `replyTo`, `threadRootId`, edits, retractions, and reactions identify their target message by `senderMsgId` ({{JMAP-CHAT-FED}}). A sending server MAY also include an optional causal hint, the `senderMsgId` of the latest message it had stored in the chat when the message was composed, which receivers MAY use for display ordering as defined in {{JMAP-CHAT-FED}}.

### Message Object Fields

`id` (String, immutable, server-set):
: Receiver-assigned ULID. Used in all client-facing references.

`senderMsgId` (String, immutable, server-set):
: The sender-assigned ULID carried in `Peer/deliver`. Equals `id` for messages composed by the owner.

`chatId` (String, immutable):
: ID of the containing Chat.

`senderId` (String, immutable, server-set):
: The accountId for owner-composed messages; for inbound messages, the `id` of the ChatContact whose `identity` is the sender identity verified by the authentication layer.

`body` (String):
: Message content. When `bodyType` is `"text/plain"` or another plaintext type, `body` MUST be valid UTF-8 text. When `bodyType` indicates an end-to-end encrypted payload (e.g., `"application/mls-ciphertext"`), `body` contains ciphertext encoded as a base64url string ({{RFC4648}} Section 5); servers MUST store and forward it without inspection or transformation. Cleared to empty string when the message is deleted.

`bodyType` (String):
: MIME type of `body`. MUST be in `supportedBodyTypes`.

`attachments` (Attachment[]):
: File attachments. Cleared to empty array when deleted.

`mentions` (Mention[]):
: Structured per-user @mention annotations. Empty by default. Authoritative for every `bodyType`; see {{mention}}.

`broadcastMentions` (BroadcastMention[]):
: Structured broadcast-scope mention annotations (`@everyone`, `@here`, `@admins`). Empty by default. Authoritative for every `bodyType`; see {{broadcast-mention}}.

`actions` (MessageAction[]):
: Out-of-band action invitations associated with this message. Empty by default. Servers store and forward these without inspection, as {{message-action}} requires.

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

`topicId` (String, optional, immutable):
: An opaque, sender-chosen identifier that groups this Message with other Messages in the same Chat into a named topic; Messages in a Chat with equal `topicId` values belong to the same topic. MUST be non-empty and MUST NOT exceed 255 bytes. A topic is independent of reply threads ({{threads}}): a reply inherits neither `topicId` nor `subject` from the Message it replies to, and servers MUST NOT set `topicId` on the sender's behalf. This document defines no topic object and no method for listing topics.

`subject` (String, optional, immutable):
: The human-readable name of the topic identified by `topicId`. MUST be valid UTF-8 text. MUST be absent when `topicId` is absent. Clients SHOULD display the `subject` of the most recent Message in a topic that carries one as that topic's name.

`sentAt` (UTCDate):
: Sender's claimed composition time. Peer-supplied; SHOULD be treated as untrusted and MUST NOT be used for ordering.

`receivedAt` (UTCDate, immutable, server-set):
: Time this mailbox stored the message. Authoritative for ordering and expiry calculations.

`senderExpiresAt` (UTCDate, optional, immutable):
: Sender-set hard-deletion deadline. When present, servers MUST permanently delete this message — removing the row entirely, not leaving a tombstone — at or before this time. A hard-deleted message appears in the `destroyed` list of subsequent `Message/changes` responses, not `updated`. Receiving servers MUST honor this field regardless of local `messageExpirySeconds` policy; whichever deadline arrives first takes effect. Servers MUST NOT use this field for ordering. Servers MUST reject a `senderExpiresAt` value that is already in the past at delivery time; in `Message/set` the rejection is an `invalidProperties` SetError with `properties: ["senderExpiresAt"]`. After hard deletion, stored attachment blobs referenced by the message SHOULD also be purged.

`burnOnRead` (Boolean, optional, immutable):
: When `true`, the receiving server MUST permanently hard-delete this message (row removal, not tombstone) immediately after setting `readAt`. Applies only to the receiving mailbox; the sender's own copy is not affected. In E2EE relay deployments, the relay cannot observe read events; the bridge or client layer MUST enforce this rule after receiving the read acknowledgement from the owner. When the recipient's effective `receiptSharing` preference is `false`, the receiver's server still sets `readAt` locally and the `burnOnRead` hard-delete fires as normal — the message IS deleted from the receiving mailbox. However, the sending server receives no `Peer/receipt` confirmation and cannot verify the deletion occurred. Senders cannot rely on `Peer/receipt` delivery to confirm that a `burnOnRead` message was destroyed.

`expiresAfterReadSeconds` (UnsignedInt, optional, immutable):
: When present, the receiving server MUST permanently hard-delete this message (row removal, not tombstone) this many seconds after setting `readAt`. Applies only to the receiving mailbox; the sender's own copy is not affected. If `burnOnRead` is also `true`, `burnOnRead` takes precedence and the message is hard-deleted immediately after `readAt` is set. In E2EE relay deployments, the relay cannot observe read events; the bridge or client layer MUST enforce this rule after receiving the read acknowledgement from the owner. When the recipient's effective `receiptSharing` preference is `false`, the receiver's server still sets `readAt` locally and the hard-delete still occurs, but the sending server receives no `Peer/receipt` confirmation and cannot verify the deletion occurred. Senders cannot rely on `Peer/receipt` delivery to confirm that an `expiresAfterReadSeconds` message was destroyed.

`deliveryState` (String, server-set):
: `"pending"`, `"delivered"`, `"failed"`, or `"received"`. For group chats, reflects aggregate state across all recipients; see `deliveryReceipts` for per-recipient detail.

  State transitions: `"pending"` → `"delivered"` (remote server accepted the message), `"pending"` → `"failed"` (delivery abandoned after retries exhausted), `"delivered"` → `"received"` (recipient returned a read receipt via `Peer/receipt`). The `"received"` state indicates the message was read, not merely delivered.

`deliveryReceipts` (Object, optional, server-set):
: For group chats, a JSON object mapping each non-owner participant's ChatContact.id to `{"deliveredAt": <UTCDate-or-null>, "deviceDeliveredAt": <UTCDate-or-null>, "readAt": <UTCDate-or-null>, "readDisposition": <String-or-null>}`. Present only when `senderId` is the accountId. `deviceDeliveredAt` is optional and MAY be absent if the recipient's platform does not support device-delivery confirmation (e.g., APNs content-available callbacks or FCM data delivery receipts); implementations that cannot obtain this signal MUST omit the field rather than approximate it with `deliveredAt`. `readDisposition` is absent when `readAt` is null. See {{read-disposition-type}}. Unlike the top-level `deliveredAt` and `readAt` fields, `deviceDeliveredAt` has no top-level parallel because device-delivery confirmation is always a per-recipient event with no aggregate owner-side equivalent.

`deliveredAt` (UTCDate, optional, server-set):
: Time the first outbound delivery was acknowledged.

`readAt` (UTCDate, optional):
: Time the owner acknowledged reading this message.

`readDisposition` (String, optional):
: The ReadDisposition value ({{read-disposition-type}}) recorded when `readAt` was last set. Absent when `readAt` is not set. A client supplies it only in the same `Message/set` update that sets `readAt` (see Marking as Read); the server MUST reject an update that sets `readDisposition` without `readAt` with an `invalidProperties` SetError naming `readDisposition`. When the server sets `readAt` in response to a `Message/set` update that omits `readDisposition`, the server MUST store `"displayed"`.

`editedAt` (UTCDate, optional, server-set):
: Time of the most recent edit.

`editHistory` (MessageRevision[], optional, server-set):
: Prior versions, oldest first. Servers MAY limit the number of retained revisions.

Note: {{JMAP-OBJ-HISTORY}} defines a general JMAP mechanism for retrieving historical object versions via `Foo/get` with `includeReplaced` and `includeDestroyed`. This specification instead embeds edit history inline within the Message object, trading on-demand retrieval for always-available history at the cost of response size. Servers implementing both specifications MAY list `"Message"` in the `supportedTypes` of the {{JMAP-OBJ-HISTORY}} account capability, enabling `Message/get` with `includeReplaced` as an alternative retrieval path for clients that prefer on-demand history. Such a server SHOULD NOT return any prior or destroyed version of a Message once that Message is deleted (`deletedAt` set) or hard-deleted, and SHOULD permanently remove those versions at the same time. Otherwise {{JMAP-OBJ-HISTORY}} would expose content that deletion, `senderExpiresAt`, `burnOnRead`, and `expiresAfterReadSeconds` are required to remove.

`deletedAt` (UTCDate, optional):
: Time the message was deleted. When set, `body` is empty, `attachments` is empty, and `editHistory` is absent. The record is retained as a tombstone unless a hard-delete rule applies.

`deletedForAll` (Boolean, optional):
: `true` when the message was deleted for all participants rather than only on this mailbox. A server that supports {{JMAP-CHAT-FED}} propagates such a deletion with `Peer/retract`.

Note: {{JMAP-METADATA}} defines per-type `metadata` and `privateMetadata` properties for data types declared in the `urn:ietf:params:jmap:metadata` capability's `dataTypes` map. Implementations MAY include `Message` in that map, enabling use cases such as per-message bookmarks or personal labels readable and writable via `Message/get` and `Message/set` without modifying the message record itself.

## SpaceRole {#space-role}

A SpaceRole is a named set of permissions within a Space. Roles are ordered by `position`; higher position values outrank lower ones. The implicit `@everyone` role (position 0) is held by all Space members and is not included in the `roles` array.

`id` (String, immutable, server-set):
: Opaque server-assigned JMAP identifier for this role.

`name` (String):
: Display name of the role.

`color` (String|Object|null, optional):
: Color for visually distinguishing role holders. Representations per {{color-convention}}: a Design Tokens color object, a CAM16 coordinate array, or an sRGB hex string (e.g., `"#5865f2"`). `null` means no color assigned.

`permissions` (String[]):
: Named permissions this role grants. Defined permission names:

  - `"view"` — see the channel
  - `"send"` — send messages
  - `"pin"` — pin messages
  - `"manage_channels"` — create, edit, delete, and reorder channels
  - `"manage_members"` — kick members, edit other members' nicknames
  - `"manage_roles"` — create and edit roles below own highest role
  - `"manage_space"` — edit Space name, description, and icon
  - `"ban"` — ban and unban members
  - `"mention_broadcast"` — use broadcast-scope @mentions (`@everyone`, `@here`, `@admins`); see {{broadcast-mention}}
  - `"start_call"` — initiate a voice or video call in a channel (see {{JMAP-VTC}} when present)

  Companion specifications MAY register additional permission names. Servers MUST ignore unrecognized permission names.

`position` (UnsignedInt):
: Role hierarchy position, sorted descending: higher `position` values outrank lower ones. Position `0` is reserved for the implicit `@everyone` role, which every member of a Space holds and which serves as the permission floor; defined SpaceRoles MUST have `position` > 0. No two roles in a Space SHOULD share the same value.

## SpaceMember {#space-member}

A SpaceMember describes one participant in a Space.

`id` (String):
: The participant's ChatContact.id, or the accountId for the owner.

`roleIds` (String[]):
: SpaceRole ids held by this member. Order is not significant. An empty list means the member holds only the `@everyone` role.

`nick` (String, optional):
: Space-specific display name override. MAY be absent; clients typically fall back to ChatContact `displayName`, then `login`.

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

`Category.channelIds`, `Space.uncategorizedChannelIds`, and each channel Chat's `categoryId` describe the same assignment. The server MUST keep them consistent: a `Space/set` operation that changes one MUST update the others in the same operation. Where `Chat.position` and the order of the containing list disagree, the order of the list governs display.

## ChannelPermission {#channel-permission}

A ChannelPermission record overrides Space-level role permissions for a specific channel, for a specific role or member. The evaluation order for these overrides is defined in {{space-permissions}}.

`targetId` (String):
: A SpaceRole id or a SpaceMember `id`.

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
: If `true`, users who are not members of this Space may fetch a restricted view of it via `Space/get`. When the Space also has `isPublic: true`, non-members may discover its existence via `Space/query` requests that include an `isPublic: true` filter condition. The restricted view is defined in the `Space/get` section. Default is `false`. Mutable by members with `"manage_space"` permission.

`memberCount` (UnsignedInt, server-set):
: Current number of members in this Space.

`closedAt` (UTCDate, optional, server-set):
: Time the Space closed. Absent while the Space is open. A Space closes when its last administrator, the last member holding `"manage_space"`, leaves while other members remain (see `removeMembers` under Space/set). Once set, `closedAt` MUST NOT be cleared. After it is set, the Space's channels are read-only: the server MUST reject every `Message/set` create in them with a `forbidden` SetError. The server MUST also reject `Space/join` for the Space with a `forbidden` method error, `SpaceInvite/set` create for the Space with a `forbidden` SetError, and every `Space/set` update to the Space with a `forbidden` SetError, except a member removing themselves via `removeMembers`. Message history remains readable.

Note: {{RFC9670}} defines a general JMAP sharing framework (`shareWith`, `myRights`) designed for binary resource sharing — "can this principal read/write/admin this object." That model maps a single principal to a flat rights set on a single resource. Space permissions are structurally different: a member holds multiple named roles, each role carries a set of named permissions, roles are ordered by hierarchy position, and individual channels within the Space can override the role-derived permissions on a per-role or per-member basis. This is a full role-based access control (RBAC) system, not a simple ACL, and it cannot be projected onto `shareWith` without losing the role hierarchy, multi-role membership, and per-channel override dimensions. The Space permission model is therefore not expressed using {{RFC9670}}. In deployments that also implement {{RFC9670}}, server implementations may choose to align `Principal.id` values with `ChatContact.id` values (for example, by assigning both for the same `identity`), but this alignment is implementation-defined and not required by either specification.

Note: {{JMAP-FILENODE}} defines a hierarchical file-storage extension for JMAP. {{JMAP-CHAT-FILENODE}} defines Space-scoped file storage by associating a FileNode namespace with each Space, analogous to how server-to-server federation methods are defined in {{JMAP-CHAT-FED}}.

Note: {{JMAP-METADATA}} defines per-type `metadata` and `privateMetadata` properties for data types declared in the `urn:ietf:params:jmap:metadata` capability's `dataTypes` map. Implementations MAY include `Space` in that map, enabling use cases such as per-Space color tags or custom labels readable and writable via `Space/get` and `Space/set` without server-side semantic significance.

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

The receipt-sharing opt-out is bidirectional: when the effective `receiptSharing` preference is `false` (either `PresenceStatus.receiptSharing` at account level, or a `Chat.receiptSharing` override for the specific Chat), the server does not deliver inbound `Peer/receipt` events (carrying `readAt`) to the owner's clients, as {{JMAP-CHAT-FED}} requires. An account that does not broadcast its own read times does not observe others' read times in the affected scope.

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
: When `false`, this account does not send read receipts to peers: the server silently suppresses outbound `Peer/receipt` calls carrying a `readAt` value on behalf of this account, as {{JMAP-CHAT-FED}} requires, and the recipient is not notified of the suppression. Default: `true`.

`updatedAt` (UTCDate, server-set):
: Time the owner last updated this record.

# Methods

Note: Implementations SHOULD support the `urn:ietf:params:jmap:refplus` capability {{JMAP-REFPLUS}} for ergonomic multi-step batch operations. RefPlus allows result references inside `/set` create/update bodies and `/query` filter conditions, enabling patterns such as creating a Message and immediately using its returned `id` in a `Message/query` within the same JMAP request array.

## ChatContact Methods

ChatContacts are created automatically when a peer delivers a message or a group update names a new participant, and by the owner through `ChatContact/set` create. Owner clients may not destroy ChatContacts.

### ChatContact/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### ChatContact/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### ChatContact/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`create` accepts only `identity`, which is required; the server assigns `id` and sets the other server-set properties. If a ChatContact with that `identity` already exists, the server MUST reject the create with a SetError of type `alreadyExists`, including the existing ChatContact's id in the `existingId` field ({{RFC8620}} Section 5.3). If `identity` equals the owner's identity (the `ownerUserId` Session field), the server MUST reject the create with an `invalidProperties` SetError naming `identity`: the owner has no ChatContact, and clients refer to the owner by the accountId.

`destroy` is not supported and MUST return `forbidden`.

`update` supports: `blocked`, `displayName`.

### ChatContact/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties: `blocked` (Boolean), `presence` (String), `identity` (String; matches the ChatContact whose `identity` equals the value octet for octet).
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
: ChatContact.id of the other participant. If a direct Chat with this contactId already exists, the server MUST reject the create with a SetError of type `alreadyExists`, including the existing Chat's id in the `existingId` field. Otherwise the server assigns a new ULID per {{chat-id}}.

#### Creating a Group Chat

`create` with `kind: "group"` accepts:

`name` (String, required):
: Display name of the group.

`memberIds` (String[], required):
: ChatContact.ids of additional initial members. If the resulting membership would exceed a server-defined limit on the number of members per group Chat, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

Optional at creation: `description` (String), `avatarBlobId` (String), `messageExpirySeconds` (UnsignedInt).

The server assigns the chatId (a ULID), adds the creator to `members` with at least one role granting permissions sufficient to administer the chat (the specific bootstrap-role configuration is server-defined), and MUST send `Peer/groupUpdate` to each initial member before any messages are sent.

Creating a Chat with `kind: "channel"` via `Chat/set` is not supported; the server MUST return a SetError of type `forbidden`. Channel Chats are created exclusively via the `addChannels` patch key in `Space/set`.

#### Updating a Chat

`update` supports the following patch keys for all chat kinds: `muted`, `muteUntil`, `receiveTypingIndicators`, `pinnedMessageIds`, `messageExpirySeconds`, `receiptSharing`. Changes to `pinnedMessageIds` are subject to the per-kind restrictions in {{chat}}; the server MUST reject an unauthorized change with a `forbidden` SetError.

The `receiveTypingIndicators` field is described in {{chat}}; it applies to direct and group chats only.

For group chats, admins additionally may update: `name`, `description`, `avatarBlobId`.

Member list changes use the following patch keys (all require admin role, except leaving the group as described under `removeMembers`):

`addMembers` (Object[]):
: Each entry: `id` (String, ChatContact.id) and optional `role` (String, default `"member"`). If the resulting membership would exceed a server-defined limit on the number of members per group Chat, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3) for that entry. On success, the server MUST send `Peer/groupUpdate` to all current members.

`removeMembers` (String[]):
: ChatContact.ids to remove. The server MUST send `Peer/groupUpdate` to all remaining members and to the removed members.

  The owner MAY leave the group by including only the accountId in `removeMembers`; the server MUST permit this whether or not the owner holds the admin role, and a server implementing {{JMAP-CHAT-FED}} MUST notify the other members with a `Peer/groupUpdate` of action `"leave"`. If the owner is the last admin and other members remain, leaving closes the group: the server MUST set `closedAt` ({{chat}}) in the same operation, and a server implementing {{JMAP-CHAT-FED}} MUST notify the other members with a `Peer/groupUpdate` of action `"close"` in place of `"leave"`. Clients can ask the user to confirm before the last admin leaves, stating that the group will close.

`updateMemberRoles` (Object[]):
: Each entry: `id` (String) and `role` (String). The server MUST send `Peer/groupUpdate` to all members.

The `role` value in `addMembers` and `updateMemberRoles` entries MUST be `"admin"` or `"member"` ({{chat-member}}); the server MUST reject any other value with an `invalidProperties` SetError.

### Destroying a Chat

For direct and group Chats, `Chat/set` destroy removes the Chat from the caller's view. The server MUST NOT delete the underlying message history for other participants. For channel Chats, destroy via `Chat/set` is not supported; the server MUST return a SetError of type `forbidden`. Channel Chats are removed exclusively via the `removeChannels` patch key in `Space/set`.

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
: The Chat ID of the conversation in which typing is occurring. The server MUST return an `invalidArguments` method error if `chatId` does not exist or the account is not a participant in that Chat, and SHOULD NOT distinguish the two cases.

`typing` (Boolean):
: `true` if the user has started typing; `false` if typing has stopped.

#### Response

`accountId` (String):
: The account ID from the request.

#### Server Behavior

Before delivering a typing push event to each potential recipient, the server applies that recipient's `receiveTypingIndicators` field for this Chat as {{chat}} specifies. For channel chats, `receiveTypingIndicators` has no effect. The server SHOULD also check whether the requesting account corresponds to a ChatContact whose `blocked` field is `true` on the recipient's contact list; if so, the server SHOULD silently drop the event for that recipient and not deliver it to any of that recipient's connected clients. The sender is not informed. On success, the server SHOULD deliver a `typing` push event (see {{push}}) to all connected clients of each local participant in the Chat other than the requesting account, subject to the `receiveTypingIndicators` and blocked-sender checks above. For federated Chats, the server SHOULD invoke `Peer/typing` ({{JMAP-CHAT-FED}}) on each remote participant's home server.

Servers SHOULD rate-limit `Chat/typing` calls per account per chat. Servers SHOULD accept no more than one call per account per chat per 3 seconds; calls that exceed this rate MAY be silently discarded without error.

The recommended 3-second value pairs with the 10-second client-side decay timer described later in this section: with one accepted event per 3 seconds while a sender is actively typing, the receiver's decay timer sees at least three events before deciding the sender has stopped. Servers using a significantly lower rate impose unnecessary load on themselves and on federated peers; servers using a significantly higher rate risk receivers clearing the typing indicator while the sender is still active.

Changes to `receiveTypingIndicators` via `Chat/set` MUST take effect for all typing push events delivered after the `Chat/set` response is returned.

When `receiveTypingIndicators` transitions from `false` to `true`, the server does not synthesize `typing: false` events for previously suppressed senders. Clients SHOULD use a decay timer to recover from any stale typing state; if no typing event is received for a given `(chatId, senderId)` pair within approximately 10 seconds, the client SHOULD hide the typing indicator for that pair.

The recommended 10-second value is calibrated against the server-side rate limit specified earlier in this section ({{chat-typing}}): with one `Chat/typing` call accepted per 3 seconds while a sender is actively typing, 10 seconds represents three missed events and is a reliable signal that the sender has stopped. Clients that diverge significantly from this value will appear inconsistent to users running multiple clients in parallel (for example, a mobile client showing the typing indicator long after a desktop client has cleared it). Implementations targeting environments with high inbound latency MAY choose a slightly longer window; implementations targeting low-latency contexts MAY choose shorter.

Note: `Chat.receiveTypingIndicators` is a persistent per-chat preference and survives reconnects. It is distinct from the `chatIds` scope in `ChatStreamEnable` ({{JMAP-CHAT-WSS}}), which is a session-scoped view-management filter. Both may suppress delivery of a typing event for the same chat, but they serve different purposes and are evaluated independently.

Implementations can cache each participant's `receiveTypingIndicators` value in memory and invalidate the cache entry on `Chat/set` updates, rather than performing a database read per delivery.

The privacy requirement that senders cannot detect suppression means a server cannot signal opt-out status back to the caller. Servers SHOULD accept and process all conforming `Chat/typing` calls regardless of any recipient's `receiveTypingIndicators` value.

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

Optional: `attachments` (Attachment[]), `mentions` (Mention[]), `broadcastMentions` (BroadcastMention[]), `actions` (MessageAction[]), `replyTo` (String), `threadRootId` (String), `topicId` (String), `subject` (String), `senderExpiresAt` (UTCDate), `burnOnRead` (Boolean), `expiresAfterReadSeconds` (UnsignedInt).

The server MUST verify the sender is a current member of the target Chat. For group chats, this means a ChatMember record exists; for channel chats, the sender must be a member of the containing Space. Non-members MUST receive a SetError of type `forbidden`.

For channel Chats, the server MUST verify the sender holds the `"send"` permission (resolved per {{space-permissions}}) before accepting the message; if absent, the server MUST return a SetError of type `forbidden`.

The server sets `id`, `senderMsgId`, `senderId`, `receivedAt`, `deliveryState`, and delivery timestamp fields, then enqueues the message for outbound delivery.

As {{broadcast-mention}} requires, a create request with a non-empty `broadcastMentions` array is rejected with `forbidden` when the target is a channel Chat and the sender lacks the `"mention_broadcast"` permission in its Space. Group chats outside any Space are not subject to this check ({{broadcast-mention}}).

#### Editing a Message

`update` with changed `body`, `bodyType`, `mentions`, and/or `broadcastMentions`, on a message where `senderId` is the accountId and `deletedAt` is absent. The server MUST reject an edit of any other message with a `forbidden` SetError. The same `"mention_broadcast"` permission check applied at create time applies to updates that add or retain a broadcast mention.

The server MUST:

1. If the server retains edit history, push a MessageRevision onto `editHistory` with the current `body`, `bodyType`, and current server time as `editedAt`.
2. Replace `body` and `bodyType` with the submitted values.
3. Set `editedAt` to the current server time.
4. Send `Peer/deliver` carrying an `edit` payload to all recipients (see {{JMAP-CHAT-FED}}).

#### Adding and Removing Reactions

Reactions are mutated via standard JSON Pointer patch keys on the `reactions` map.

To **add** a reaction, the client SHOULD supply a ULID as the `senderReactionId` and patch:

~~~
"reactions/<senderReactionId>": {
  "emoji": "<value>",
  "sentAt": "<UTCDate>"
}
~~~

The server MUST set `senderId` to the accountId and MUST propagate the reaction to all recipients via `Peer/deliver` `reactionUpdate` payload, carrying the `senderReactionId` as the map key.

To **remove** a reaction, the client patches:

~~~
"reactions/<senderReactionId>": null
~~~

The server MUST remove the entry and MUST propagate the removal via `Peer/deliver` `reactionUpdate` payload using the same `senderReactionId`.

Clients SHOULD only add or remove reactions where `senderId` is the accountId. Servers MUST reject attempts to add or remove reactions for other senders with `forbidden`.

#### Deleting a Message

`update` with `deletedAt: <timestamp>`.

- If `deletedForAll: true` is also set, the deletion applies to all participants. Servers MUST reject `deletedForAll: true` for messages where `senderId` is not the accountId with a `forbidden` SetError. A server that supports {{JMAP-CHAT-FED}} propagates the deletion to other servers as that document specifies.
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
: Full-text search over `body`. For a `bodyType` of `"application/jmap-chat-rich"`, the filter matches only the spans' `text` values. Servers that do not support full-text search MUST return `unsupportedFilter`.

`threadRootId` (String, optional):
: Return only messages in this thread. Valid only when `supportsThreads` is `true`; otherwise servers MUST return `unsupportedFilter`.

`topicId` (String, optional):
: Return only messages whose `topicId` equals this value. Messages with an encrypted `bodyType` carry no server-visible `topicId` ({{e2ee-fields}}) and never match this filter.

`hasAttachment` (Boolean, optional):
: Filter to messages with or without attachments.

`hasMention` (Boolean, optional):
: Filter to messages that mention the owner: the accountId appears in `mentions`, OR `broadcastMentions` contains a scope whose resolved recipient set includes the owner.

For channel Chats, the server MUST verify the caller holds the `"view"` permission before returning messages in `Message/get` or `Message/query` responses; messages in channels the caller cannot view MUST be omitted.

Default sort: `receivedAt` ascending when `chatId` is present; `receivedAt` descending when `chatId` is absent.

### Message/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## Space Methods

### Space/get

Standard JMAP `/get`, with the following additional rules for non-member callers.

When the caller is not a member of a requested Space:

* If the Space has `isPubliclyPreviewable: true`, the server MUST return a restricted view of the Space containing only the fields `id`, `name`, `description`, `iconBlobId`, `memberCount`, `createdAt`, `isPublic`, `isPubliclyPreviewable`, and `closedAt` (when set). All other Space fields MUST be omitted from the returned object, even if explicitly requested via the `properties` argument. The server MUST NOT include the Space `id` in the `notFound` array.
* If the Space has `isPubliclyPreviewable: false` (or the Space does not exist), the server MUST include the requested Space `id` in the `notFound` array of the response and MUST NOT include the Space in the `list` array.

When the caller is a member of a requested Space, the full Space object is returned subject to the standard JMAP `/get` `properties` filter; the restricted view above does not apply.

### Space/changes

Standard JMAP `/changes`. The Space object is returned in full on each change; the sub-arrays `roles`, `members`, and `categories` are included in every Space object and reflect the current state.

### Space/set {#space-set}

Standard JMAP `/set`.

The `update` operation for Space uses semantic mutation keys (`addRoles`, `removeRoles`, `addMembers`, etc.) rather than JSON Pointer paths. This departure from the standard RFC 8620 PatchObject model is intentional.

Other JMAP extensions — including those with server-enforced access control such as {{JMAP-FILENODE}} — mutate a single property map on a single object and can express that naturally as a PatchObject pointer path. Space mutations are structurally different: each operation is an atomic transaction across multiple concerns. Adding a member checks role-hierarchy constraints, enforces per-Space membership quotas, cascades default-channel subscriptions, and triggers outbound `Peer/groupUpdate` notifications in federated deployments. Removing a role demotes affected members, recalculates effective permissions across every channel's override table, and may trigger last-administrator protection. These cross-cutting effects cannot be reliably expressed as a bag of independent pointer-path assignments that the server reassembles into a coherent operation — the server needs to know the caller's intent, not just which bytes changed. Each semantic key names a discrete, permission-checked mutation operation rather than a direct property assignment.

#### Creating a Space

`create` accepts:

`name` (String, required).

Optional: `description` (String), `iconBlobId` (String).

The server assigns the Space's `id` and adds the caller to `members` with at least one role granting permissions sufficient to administer the Space. The specific bootstrap-role configuration is server-defined; servers MAY pre-create a high-position role for the caller, MAY assign every permission from {{space-permissions}}, or MAY use any other mechanism that ensures the newly-created Space is administrable. The server returns the new Space.

#### Updating a Space

`update` supports the following patch keys:

`name`, `description`, `iconBlobId`, `isPublic`, `isPubliclyPreviewable`:
: Metadata fields. Require `"manage_space"` permission.

`addRoles` (Object[]):
: Each entry: `name` (String), `permissions` (String[]), `position` (UnsignedInt), and optionally `color` (String\|Object\|null, optional) — per the color convention ({{color-convention}}). Server assigns ULIDs. Requires `"manage_roles"`. Members may only add roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this. If the resulting role count would exceed a server-defined limit on the number of roles per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeRoles` (String[]):
: SpaceRole ids to remove. Members holding only removed roles are demoted to `@everyone`. Requires `"manage_roles"`.

`updateRoles` (Object[]):
: Each entry: `id` (String) and any of `name`, `color`, `permissions`, `position`. Requires `"manage_roles"`. Members may only modify roles whose `position` is strictly less than their own highest-position role; servers MUST enforce this.

`addMembers` (Object[]):
: Each entry: `id` (ChatContact.id, String) and optional `roleIds` (String[]). Requires `"manage_members"`. If the resulting membership would exceed a server-defined limit on the number of members per Space, the server MUST return an `overQuota` SetError ({{RFC8620}} §5.3).

`removeMembers` (String[]):
: ChatContact.ids to remove. Requires `"manage_members"`. Servers SHOULD refuse to remove another member when doing so would leave the Space without any member holding `"manage_members"` or `"manage_space"`; the specific last-administrator protection policy is server-defined. Servers MUST return an `invalidProperties` SetError ({{RFC8620}} §5.3) naming `removeMembers` when refusing on this basis.

  A member who does not hold `"manage_members"` permission MAY remove themselves from the Space by including only the accountId in `removeMembers`. The server MUST permit self-removal regardless of the caller's permission set. Last-administrator protection does not apply to self-removal. If the member removing themselves is the last member holding `"manage_space"` and other members remain, leaving closes the Space: the server MUST set `closedAt` ({{space}}) in the same operation. Clients can ask the user to confirm before the last administrator leaves, stating that the Space will close.

`updateMembers` (Object[]):
: Each entry: `id` (String) and any of `roleIds`, `nick`. Role changes require `"manage_roles"`. Changing another member's `nick` requires `"manage_members"`. A member MAY set their own `nick` by submitting `updateMembers` with a single entry whose `id` is the accountId and which contains only `nick`; the server MUST permit this regardless of the caller's permission set.

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

**Joining via invite code:** `inviteCode` resolves only within the request's `accountId`, which names the account hosting the Space; invite links therefore carry both the `accountId` and the code. The server MUST resolve `inviteCode` to a SpaceInvite record in that account by matching against the `code` field (not the `id` field), verify the invite has not expired and has not reached `maxUses`, and then atomically increment `uses` and add the caller to the Space's member list. The `uses` increment and membership insertion MUST be performed within a single atomic operation so that concurrent redemptions cannot exceed `maxUses`. The caller is assigned no roles beyond `@everyone` unless the invite specifies otherwise. If the invite has expired (`expiresAt` is in the past) or has reached its redemption limit (`uses` equals `maxUses`), the server MUST return an `invalidArguments` method error. If the `inviteCode` does not correspond to any SpaceInvite record in that account, the server MUST return an `invalidArguments` method error.

**Joining a public Space:** When `spaceId` is supplied, the server MUST verify that the identified Space has `isPublic: true`. If the Space does not exist or has `isPublic: false`, the server MUST return a `notPermitted` method error. On success, the server adds the caller to the Space's member list with no roles beyond `@everyone`.

The server MUST verify that no active SpaceBan exists for the requesting account's owner in the target Space. If a ban exists, the server MUST reject the request with a `forbidden` method error.

If the target Space has `closedAt` set ({{space}}), the server MUST reject the request with a `forbidden` method error, whether the caller supplied `inviteCode` or `spaceId`.

The server evaluates these checks in order: the argument check (`invalidArguments`), then resolution of the target Space from `inviteCode` (`invalidArguments`) or `spaceId` (`notPermitted`), then the ban check and the closed-Space check (`forbidden`). It returns the error for the first check that fails. A caller can therefore learn of a ban only for a Space it could otherwise join.

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

`destroy` removes the emoji. Existing Reaction records referencing this emoji retain their `customEmojiId` value; clients can fall back to the `emoji` field when the referenced emoji cannot be resolved.

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

Standard JMAP `/get`. Members with `"ban"` permission in the Space see all SpaceBan records for that Space. A banned user fetching their own account sees only SpaceBan records where `userId` is the accountId.

### SpaceBan/changes

Standard JMAP `/changes`.

### SpaceBan/set

Standard JMAP `/set`.

`create` accepts: `spaceId` (String, required), `userId` (String, required), `reason` (String, optional), `expiresAt` (UTCDate, optional). Requires `"ban"` permission in the Space.

`update` supports: `reason`, `expiresAt`. Requires `"ban"` permission in the Space.

`destroy` lifts the ban. Requires `"ban"` permission in the Space.

# Rich Body Format {#rich-body}

This section defines the `application/jmap-chat-rich` body type for structured inline formatting.

When `bodyType` is `"application/jmap-chat-rich"`, `body` MUST contain a JSON-encoded string whose top-level parsed value is an object conforming to this section. Spans are presentation only: servers MUST NOT map, validate, or rewrite a span's `userId` or any other part of `body`, including when a Message passes between servers ({{JMAP-CHAT-FED}}), and MUST NOT make any authorization, notification, push, query, or delivery decision from span content. A client composing a rich body MUST list each user it marks with a `"mention"` span in `mentions` ({{mention}}) and each scope it marks with a `"broadcast"` span in `broadcastMentions` ({{broadcast-mention}}); only those arrays take effect.

## Span Object

The body object contains a single field:

`spans` (Span[]):
: An ordered array of Span objects representing the message content from left to right.

Each Span has the following fields:

`type` (String):
: One of the defined span types below.

`text` (String):
: The plaintext content of this span. MUST be present on all span types.

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
| `"mention"` | `userId` (String) | Per-user @mention, for display; `userId` is display data written by the composing client, with no server meaning, and need not be an id on the receiving server |
| `"broadcast"` | `scope` (String) | Broadcast-scope @mention, for display; `scope` is one of `"everyone"`, `"here"`, or `"admins"` (see {{broadcast-mention}}) |
| `"link"` | `uri` (String) | Hyperlink; `uri` SHOULD be treated as untrusted |

Servers MUST NOT reject messages solely because they contain unrecognized span types; this allows protocol evolution. Clients MUST silently ignore unknown span types and render the `text` field as plaintext, preserving forward compatibility.

## Example

~~~json
{
  "spans": [
    {"type": "text", "text": "Hello "},
    {"type": "mention", "text": "@alice", "userId": "contact-alice"},
    {"type": "text", "text": " and "},
    {"type": "broadcast", "text": "@here", "scope": "here"},
    {"type": "text", "text": ", see this code: "},
    {"type": "code", "text": "foo()"},
    {"type": "text", "text": ". Full example:"},
    {"type": "codeblock", "text": "fn main() {}", "lang": "rust"}
  ]
}
~~~

A Message carrying this body lists `{"id": "contact-alice"}` in `mentions` and `{"scope": "here"}` in `broadcastMentions`.

# Thread Model {#threads}

Servers advertising `supportsThreads: true` support structured conversation threads.

A thread is the set of Messages sharing a common `threadRootId`. The root message has `threadRootId` absent.

Thread root assignment rules:

- A message with no `replyTo` is a potential thread root: leave `threadRootId` absent.
- A message replying to a thread root (the referenced message has no `threadRootId`): set `threadRootId` to the value of `replyTo`.
- A message replying to a non-root message: set `threadRootId` to the referenced message's `threadRootId`.

Clients SHOULD follow these rules. Servers SHOULD validate them and MAY correct `threadRootId` if the client supplies an incorrect value. A client that sends an incorrect `threadRootId` — or a server that does not correct one — will produce a message that does not appear in `Message/query` results filtered by the intended thread root, and whose `replyCount` will not be reflected on the intended thread root message. Thread structure is a display feature; incorrect assignment causes broken threading in the UI but does not affect message delivery or data integrity.

`Message/query` with a `threadRootId` filter returns all messages in that thread. The `replyCount` field on each message gives the count of direct replies.

# Server-to-Server Methods

Server-to-server federation methods (`Peer/deliver`, `Peer/receipt`, `Peer/typing`, `Peer/retract`, `Peer/groupUpdate`) are defined in {{JMAP-CHAT-FED}}.

# Push Notifications {#push}

Servers MUST support the EventSource mechanism defined in {{RFC8620}} Section 7.3.

Servers SHOULD also support the push subscription mechanism defined in {{RFC8620}} Section 7.2 for deployments requiring offline and mobile push delivery.

Servers that deliver push subscriptions via Web Push SHOULD also advertise the `urn:ietf:params:jmap:webpush-vapid` capability {{RFC9749}} and authenticate their Web Push messages with a VAPID-signed JWT {{RFC8292}}. VAPID authentication allows push services to verify the server's identity, which is required for reliable authenticated delivery to mobile clients. Servers that use only the EventSource mechanism or a non-Web-Push delivery channel are not subject to this recommendation.

## State-Change Events

~~~
event: state
data: {"@type":"StateChange","changed":{"<accountId>":{"Message":"<s>","Chat":"<s>","ChatContact":"<s>"}}}
~~~

The example shows three types. The `changed` object can name any data type registered by this document (`ChatContact`, `Chat`, `Message`, `Space`, `SpaceInvite`, `CustomEmoji`, `SpaceBan`, `ReadPosition`, `PresenceStatus`) and any type defined by a companion capability the server supports.

Clients SHOULD call the corresponding `/changes` method upon receipt. On `cannotCalculateChanges`, fall back to `/get`.

## Typing Events

~~~
event: typing
data: {"chatId":"<id>","senderId":"<contact-id>","typing":<bool>}
~~~

Not stored; carries no state token. Before delivering this event to a recipient, the server applies the same `receiveTypingIndicators` and blocked-sender checks, at the same strength, described in the `Chat/typing` server behavior ({{chat-typing}}).

## Presence Events

~~~
event: presence
data: {"contactId":"<id>","presence":"<state>","lastActiveAt":"<ts>","statusText":"<string>|null","statusEmoji":"<string>|null"}
~~~

Before delivering this event, the server SHOULD silently drop it if the identified ChatContact (`contactId`) has `blocked: true` on the receiving owner's contact list. The remote contact is not informed.

The `statusText` and `statusEmoji` fields reflect the contact's current PresenceStatus values; they are `null` when the contact has no active custom status. Clients can update their displayed status immediately upon receiving this event without waiting for a `/changes` poll.

## Why Ephemeral Events Are Not State Changes

The RFC 8620 push model delivers a `StateChange` containing an opaque state token; the client then calls `/changes` to discover what changed and `/get` to fetch the updated objects. This model is well-suited to durable data: a new email, a modified calendar event, an updated contact. Chat typing indicators and real-time presence transitions are not durable data. A typing indicator has sub-second relevance; by the time a client receives a `StateChange`, issues a `/changes` call, and fetches the result, the signal is stale — the user may have already stopped typing or sent the message. There is nothing to persist, nothing to sync, and nothing to catch up on after reconnection. Routing these signals through the state/changes machinery would impose a full round-trip on information that is useful only if delivered immediately and silently discarded if not. The ephemeral event types defined here and in {{JMAP-CHAT-WSS}} are therefore delivered inline as fire-and-forget signals with no state token, no `/changes` participation, and no persistence requirement.

# Blob Storage {#blobs}

Standard JMAP blob upload and download per {{RFC8620}} Section 6, using the `uploadUrl` and `downloadUrl` Session templates.

`Blob/copy` is defined in {{RFC8620}} Section 6.3. Implementations requiring additional blob operations (such as `Blob/get`) SHOULD refer to {{JMAP-BLOBEXT}} (`urn:ietf:params:jmap:blob2`), which obsoletes {{RFC9404}} (`urn:ietf:params:jmap:blob`) and adds substantial new capabilities; servers SHOULD advertise `urn:ietf:params:jmap:blob2` where available. A server MAY advertise both capabilities, but a client uses only one of them in a request, per {{JMAP-BLOBEXT}}.

The upload response defined here includes the `sha256` field as specified
in {{JMAP-CID}}; see that document for the normative definition, client
verification guidance, and security considerations.

Servers SHOULD support `Blob/lookup` (defined in {{JMAP-BLOBEXT}}) and register `"Message"` as a participating data type, enabling clients to perform reverse lookups from a blobId to all Messages that reference it as an attachment.

Servers MAY support `Blob/convert` (defined in {{JMAP-BLOBEXT}}) for image media types, enabling server-side thumbnail generation for chat attachment previews.

Chat file-sharing workflows typically upload a blob and reference it in a `Message/set` within seconds. The one-hour minimum retention for unreferenced blobs required by {{RFC8620}} Section 6 covers this window.

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

Authentication is handled at the transport layer as described in {{introduction}}. The protocol derives access control from the stable, opaque identity provided per connection:

- **Owner** (identity equals the mailbox owner's identity): all methods.
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

Role hierarchy enforcement: members may only create or modify SpaceRoles whose `position` is strictly less than their own highest-position role. Servers enforce this on `addRoles` and `updateRoles` ({{space-set}}).

The "Requires `X`" and "Mutable by members with `X` permission" clauses elsewhere in this specification describe the recommended mapping from the closed permission vocabulary defined in {{space-role}} to the actions those permissions gate. Deployments MAY substitute or augment these mappings with deployment-specific authorization policy — for example, gating an action on a different permission, requiring additional authorization beyond merely holding the named permission, or recognizing an internal capability not present in the closed vocabulary. The wire-level contract is that unauthorized callers MUST receive `forbidden`; the permission names defined in {{space-role}} retain their stated meanings when they appear in `SpaceRole.permissions` values returned to clients.

Some MUST constraints in this specification are not subject to this latitude because they define wire-level behaviour rather than server-internal authorization: the sender-only constraints on `Message/set update` and on Reaction creation and removal, the mailbox-owner constraint inherited from {{RFC8620}}, and all input validation requirements in {{security}}.

## Space Governance: Deployment Variation {#space-deployment}

This specification defines the membership, role, and permission mechanisms for Spaces but does not prescribe a single ownership model. Implementations vary:

- Some deployments designate a single irreducible controller per Space (the creator, in the style of Discord, Telegram groups, or Slack workspaces), enforced server-side so that the controller cannot be removed or demoted by other members.
- Other deployments operate without a designated controller, relying entirely on the role-and-permission graph; if administrative coverage is lost (no member holds `"manage_space"`), recovery is out-of-band.
- Enterprise deployments may delegate Space governance to an external identity provider or directory service, with the JMAP Chat server enforcing administrative actions on behalf of that external authority.

Federated deployments SHOULD coordinate their governance policies with peer servers when federating Spaces, since divergence can produce surprising interop behaviour during membership churn (for example, server A may refuse to remove its local controller while server B has no such restriction). This specification does not define a wire-level mechanism for advertising or negotiating a governance policy between peers; deployments that federate at scale are encouraged to publish their policy out-of-band.

Clients cannot assume a specific governance model. Where the UI needs to surface "who runs this Space", clients can derive the answer from the role-and-permission graph (e.g., members holding `"manage_space"`) rather than from a single distinguished principal.

# Security Considerations {#security}

## Identity Verification

`senderUserId` in all Peer/* methods is caller-supplied and untrusted. {{JMAP-CHAT-FED}} requires the server to compare it with the identity from its own authentication layer before any storage or action.

## Input Validation

All peer-supplied fields are attacker-controlled. Servers MUST validate:

- `body`: validate UTF-8 for plaintext body types; enforce `maxBodyBytes`.
- `bodyType`: reject a value not listed in `supportedBodyTypes` ({{capability}}); in `Message/set` create, also reject a type that a companion specification reserves for server-created Messages.
- `contentType`: reject syntactically invalid MIME values.
- `size`: verify against actual blob byte count after fetch.
- `sentAt`, `editedAt`: store as-is; never use for ordering or expiry.
- `chatId`: verify per {{chat-id}} — confirm match with stored id or verify membership; reject mismatches.
- `emoji`: validate as a non-empty string; enforce an implementation-defined maximum byte length to prevent denial of service.
- `topicId`: reject an empty value or a value exceeding 255 bytes.
- `subject`: validate UTF-8; enforce an implementation-defined maximum byte length; reject when `topicId` is absent.
- `broadcastMentions`: reject any entry whose `scope` is not one of `"everyone"`, `"here"`, `"admins"`. Additionally enforce the `"mention_broadcast"` permission check defined in {{broadcast-mention}}.

Servers SHOULD also reject a `filename` containing `/`, `\`, or null bytes, and SHOULD verify `sha256` against actual blob content after fetch ({{attachment}}).

When a `Message/set` create or update fails one of these checks and this document names no other error for that check, the server MUST reject the entry with an `invalidProperties` SetError ({{RFC8620}} §5.3) whose `properties` names the offending property.

## Denial of Service

Enforce `maxBodyBytes` and `maxAttachmentBytes` at parse time, before any fetch or storage. Enforce `maxAttachmentsPerMessage` at creation and update time. Servers SHOULD also enforce implementation-defined per-aggregate caps on group Chat membership, Space membership, roles, channels, and categories at `Chat/set` and `Space/set` time, returning `overQuota` SetErrors ({{RFC8620}} §5.3) as specified in those methods. Rate-limit `Peer/typing` per peer. For limits on placeholder ChatContacts created by peers, see {{JMAP-CHAT-FED}}.

## Chat ID Integrity

Chat IDs are server-assigned ULIDs. Security against cross-conversation injection relies on sender authentication and chat membership verification, not on ID derivation.

## Broadcast Mention Abuse {#broadcast-mention-abuse}

Broadcast mentions (`@everyone`, `@here`, `@admins`; see {{broadcast-mention}}) elevate push urgency for every member targeted by the receiving server's delivery-time set, and bypass each targeted owner's `Chat.muted` and `Chat.muteUntil` settings ({{chat}}). The blast radius scales with Space or Chat membership, and a single send can wake every active device in a large Space. This makes broadcast mentions a high-leverage target for both abuse and accidental noise.

The `"mention_broadcast"` permission is the spec's primary control: only members with the permission can send a broadcast mention, and servers reject offending `Message/set` requests with `forbidden` as {{broadcast-mention}} requires. The permission SHOULD NOT be granted by default to the `@everyone` role; deployments SHOULD scope it to roles that already carry administrative or moderator responsibility, and SHOULD audit grant changes. Rate-limiting broadcast mentions per sender, per Space, and per scope is RECOMMENDED to bound damage when the permission is misused or a sender's credentials are compromised.

The `Chat.muted` bypass for targeted recipients is a deliberate UX choice for the common case (a moderator paging the on-call set during an incident). It is also the most abusable property of the feature. Deployments SHOULD provide owners a way to override the bypass — for example, a "respect mute regardless of broadcast scope" account preference, or a per-Chat preference that disables broadcast elevation for that Chat — and SHOULD make that preference discoverable to owners who report ongoing abuse. Such preferences are deployment-defined; this specification does not assign a wire field for them, and federated peers SHOULD NOT be told whether a remote owner has chosen to suppress broadcast elevation, since that information would let a sender confirm which recipients are silently filtering them.

The send-time recipient set computed by the sending server (the informational list described in {{broadcast-mention}}) SHOULD NOT be used by a receiving server for authorization or for deciding whether to elevate push: trusting it would let a hostile or buggy sending server force elevation against recipients who would not otherwise qualify under the receiving server's own predicate. Each receiving server resolves its own set and acts on its own resolution.

Because rich-body spans are display-only ({{rich-body}}) and servers do not compare them with `mentions` or `broadcastMentions`, a span can display a mention that has no effect, and the arrays can name users or scopes that no span displays. A rendering client SHOULD style a `"mention"` or `"broadcast"` span as a mention only when the corresponding entry is present in `mentions` or `broadcastMentions`, and otherwise SHOULD render its `text` as plain text. Clients SHOULD make array mentions that no span displays discoverable to the recipient.

## Blocked Contacts

Messages from a ChatContact whose `blocked` field is `true` are silently dropped regardless of whether they arrive in a direct chat or a group chat context.

Typing and presence push events whose sending or referenced ChatContact has `blocked: true` on the receiving owner's contact list are similarly dropped server-side before delivery to any of the owner's clients (see the `Chat/typing` server behavior in {{chat-typing}}, the typing and presence event sections in {{push}}, and the analogous rules in {{JMAP-CHAT-WSS}} and {{JMAP-CHAT-FED}}). The sender is not informed. This prevents a blocked user from leaking presence or attention patterns on any transport even though their messages are dropped.

Companion capabilities also consume the `blocked` field for ephemeral event suppression: {{JMAP-SCENE-WSS}} suppresses ring, call-end, spatial presence, object, and interaction events from blocked contacts. See that specification for the per-event-type rules.

When `Chat.receiveTypingIndicators` is `false`, typing push events for that Chat are suppressed server-side (see `receiveTypingIndicators` in {{chat}} and the `Chat/typing` server behavior in {{chat-typing}}) before delivery to the owner. The sender is not informed; `Chat/typing` succeeds normally. This prevents a sender from inferring that typing indicators are being suppressed.

## Read Receipt Privacy

When the effective `receiptSharing` preference is `false`, the server suppresses outbound and inbound read-receipt events as specified in `PresenceStatus.receiptSharing` ({{presence-status}}), `Chat.receiptSharing` ({{chat}}), and the `Peer/receipt` Sender and Receiver Behavior in {{JMAP-CHAT-FED}}. This prevents remote peers from inferring when the owner reads messages.

Clients SHOULD NOT expose sub-minute precision for `readAt`, `lastReadAt`, or `deviceDeliveredAt` timestamps in the user interface. Displaying relative representations (such as "today", "yesterday", or hour-granularity) reduces the risk of exposing behavioral patterns through precise read timing. The underlying UTCDate values retain full precision for protocol purposes; coarsening is a presentation concern only.

## Out-of-Band Endpoints and Actions

`ChatContact.endpoints`, `Session.ownerEndpoints`, and `Message.actions` carry peer-supplied URIs, which are untrusted at every level ({{endpoint}}, {{message-action}}):

- Clients do not fetch or connect to any OOB URI automatically; all OOB interactions require explicit user initiation ({{message-action}}, {{chat-contact}}). Auto-joining a call is a privacy violation.
- Payment URIs (`urn:jmap:chat:cap:payment`) are validated by the client wallet before any funds are transferred. Servers store and forward payment URI values without inspection ({{message-action}}).
- Blob/file URIs (`urn:jmap:chat:cap:blob`) used for OOB fetch are an SSRF vector; servers that fetch from peer-supplied blob endpoints SHOULD restrict connections to the known peer address space.
- `metadata` values are peer-supplied and MUST be ignored if they do not conform to the expected shape for the known `type`.
- Clients ignore unknown `type` URIs, as {{endpoint}} and {{message-action}} require; servers SHOULD silently ignore them.

## End-to-End Encrypted Deployments {#e2ee}

In relay deployments, the relay routes Peer/* messages but SHOULD NOT have access to plaintext message content. Implementations SHOULD ensure:

- The `body` field carries ciphertext only; plaintext SHOULD NOT be transmitted to the relay in an encrypted deployment.
- The relay is architecturally excluded from the encryption key schedule (e.g., by using MLS {{RFC9420}} or a similar protocol that does not involve the relay in key agreement).
- Servers store and forward `body` without inspection or transformation when `bodyType` indicates an encrypted type, as the `body` definition in {{message}} requires.
- Metadata visible to the relay (the routing fields listed below, timestamps, body size, and the arguments of `reactionUpdate` payloads and `Peer/retract`) remains an information-leakage surface. Deployments requiring metadata privacy SHOULD apply message padding and cover traffic at the transport layer; those techniques are outside the scope of this document.

### Fields Inside the Ciphertext {#e2ee-fields}

When `bodyType` indicates an encrypted type, in any topology, the Message fields `mentions`, `broadcastMentions`, `attachments`, `actions`, `replyTo`, `threadRootId`, `topicId`, and `subject` MUST be empty or absent, and the federation causal hint ({{JMAP-CHAT-FED}}) MUST be absent. A client that needs any of this information MUST carry it inside the ciphertext. For attachments this includes the filename, content type, size, `sha256`, and the location from which the blob is fetched. An Attachment carried inside the ciphertext carries the key and location fields defined by the end-to-end encryption scheme in use as extension properties; this document defines none. The server never sees these properties, since they exist only inside the ciphertext. Servers MUST reject a `Message/set` create or update that has an encrypted `bodyType` and any of these fields non-empty with an `invalidProperties` SetError whose `properties` lists each such field, and MUST reject such a `Peer/deliver` payload with an `invalidArguments` method error ({{JMAP-CHAT-FED}}).

As a consequence, the server cannot validate mentions, broadcast-mention scopes, or attachment hashes for an encrypted message, and skips those checks (steps 11 through 13 of new-message validation in {{JMAP-CHAT-FED}}). The server also cannot compute mention-based push elevation, broadcast-mention recipient sets, `replyCount`, or thread membership for encrypted messages; clients compute these after decryption. Likewise, the `topicId` filter of `Message/query` matches only messages with a non-encrypted `bodyType`; clients group encrypted messages into topics after decryption. The `"mention_broadcast"` check of {{broadcast-mention}} cannot be applied by the server to content inside the ciphertext; clients enforce it as described in {{e2ee-enforcement}}.

In a relay deployment, the `Peer/deliver` arguments outside the ciphertext are therefore limited to the routing arguments `accountId`, `chatId`, `chatKind`, and `senderUserId`, and the `message` fields `senderMsgId`, `bodyType`, `body` (as ciphertext), `sentAt`, `senderExpiresAt`, `burnOnRead`, and `expiresAfterReadSeconds`. For `edit` payloads they are `senderMsgId`, `bodyType`, `body` (as ciphertext), and `editedAt`.

### Sender Authentication and Relay Enforcement {#e2ee-enforcement}

The sender's identity MAY be authenticated inside the end-to-end encryption layer instead of by the relay (for example, MLS {{RFC9420}} authenticates every sender through its group credential). In that case the relay needs only a recipient or group routing handle (`accountId` or `chatId`) to route the message, and `senderUserId` MAY be an opaque value bound to the sender's transport credential rather than the sender's identity. A receiving server maps such a value to a ChatContact like any other identity. The relay still compares `senderUserId` with the authenticated transport credential before accepting the call. Receiving clients SHOULD authenticate the sender from the end-to-end encryption layer and SHOULD NOT rely on a relay-supplied `senderId` for that purpose.

Relay-side enforcement of SpaceBan and slow mode (steps 4 and 6 of new-message validation in {{JMAP-CHAT-FED}}) is OPTIONAL. A relay that sees only an opaque sender value also cannot evaluate `"send"` permission or block state (steps 5 and 7). Receiving clients SHOULD perform each of these checks that the relay does not perform, and the `"mention_broadcast"` check for broadcast mentions carried inside the ciphertext (that is, the inner `broadcastMentions`; spans remain display-only), against signed group state (for example, the MLS group context, or Space state signed by the Space's authority), and SHOULD discard messages that fail them.

## Federation Security

Security considerations for server-to-server federation are defined in {{JMAP-CHAT-FED}}.

# IANA Considerations

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:jmap:chat`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.

## The urn:jmap Namespace {#iana-urn}

This document and its companion specifications use the URN namespace identifier "jmap". The identifier is not registered with IANA; registration as a formal URN namespace ({{RFC8141}}) is deferred. No IANA action is requested for it at this time. Names under `urn:jmap:` are structured and allocated as follows.

Syntax:
: A URN in this namespace matches `jmap-urn` ({{RFC5234}}):

  ~~~abnf
  jmap-urn = "urn:jmap:" segment *( ":" segment )
  segment  = 1*( ALPHA / DIGIT / "-" / "_" )
  ~~~

  The first segment is allocated as described under Assignment and determines how the rest of the name is structured. Names contain no percent-encoded characters. URN-equivalence is that of {{RFC8141, Section 3}}: the NSS is compared case-sensitively, because JMAP account and object ids are case-sensitive. No r-component or q-component semantics are defined, and f-components are not used.

Assignment:
: Each first segment is held by exactly one of the following, and the two sets are disjoint:

  1. A capability: a URN registered in the "JMAP Capabilities" registry ({{RFC8620, Section 9.4}}) whose first segment is that segment. Additional capability URNs under the segment (for example, `urn:jmap:chat:calendars`) are registered the same way. Other names under the segment are assigned by the specification that defines the capability or by documents that extend it; this document assigns `urn:jmap:chat:cap:*` ({{endpoint}}).
  2. An object URN type: a type name registered in the "JMAP Object URN Types" registry ({{JMAP-CHAT-CALENDARS}}). The rest of the name is `<accountId>:<objectId>` as that document defines.

  The collision rule of {{JMAP-CHAT-CALENDARS}} keeps the two sets apart: a type name is not registered if it equals the first segment of an assigned capability URN, and a capability URN is not registered if its first segment equals a registered type name. Uniqueness of capability URNs and type names follows from the registries. A name, once assigned, is not reassigned.

Uniqueness:
: {{RFC8620}} does not make account ids globally unique, so an object URN identifies an object only relative to the JMAP server that issued the account id; equal object URNs received from two different servers can name different objects. Implementations interpret an object URN only in the context of the JMAP session or federation peer from which it was received. Names in this namespace never equal names in `urn:ietf:params:jmap:`, because the NIDs differ.

## JMAP Data Types Registration

IANA is requested to register the following entries in the "JMAP Data Types" registry:

Type Name:
: ChatContact

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: Chat

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: Message

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: Space

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: SpaceInvite

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: CustomEmoji

Can Reference Blobs:
: Yes

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: SpaceBan

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: ReadPosition

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

Type Name:
: PresenceStatus

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:jmap:chat`

Reference:
: This document

## JMAP Error Code Registration

IANA is requested to register the following entries in the "JMAP Error Codes" registry ({{RFC8620}} Section 9.5):

JMAP Error Code:
: `rateLimited`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document

Description:
: A method-level error: the calling peer has exceeded a server-defined request rate limit. Returned by the Peer/* methods of {{JMAP-CHAT-FED}}. The caller waits before retrying. (Per-object rate limits on `/set`, including channel slow mode, use the `rateLimit` SetError of {{RFC8620, Section 5.3}}.)

JMAP Error Code:
: `notPermitted`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document

Description:
: The requested Space does not exist or is not joinable by the caller (for example, `Space/join` by `spaceId` for a Space that is not public). Distinct from `forbidden`, which means the caller is specifically refused (for example, because it is banned).

--- back

# Acknowledgements

The author thanks the JMAP working group for {{RFC8620}}.
