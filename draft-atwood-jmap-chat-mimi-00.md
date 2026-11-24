---
title: JMAP Chat MIMI Mapping
abbrev: JMAP Chat MIMI
docname: draft-atwood-jmap-chat-mimi-00
category: std
stream: IETF

ipr: trust200902
date: 2026

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    organization: Review Commit, LLC
    city: Seattle
    region: WA
    country: US
    email: mark@reviewcommit.com

normative:
  RFC8866:
  RFC8122:
  RFC2119:
  RFC4648:
  RFC8174:
  RFC8620:
  RFC9420:
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) message content"
    author:
      - fullname: Rohan Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-content-10
    date: 2026-11-02
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
      Internet-Draft: draft-ietf-mimi-protocol-07
    date: 2026-11-02
  MIMI-ROOM-POLICY:
    title: "Room Policy for the More Instant Messaging Interoperability (MIMI) Protocol"
    author:
      - fullname: Rohan Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-room-policy-04
    date: 2026-07-06
  MLS-EXTENSIONS:
    title: "The Messaging Layer Security (MLS) Extensions"
    author:
      - fullname: Raphael Robert
    seriesinfo:
      Internet-Draft: draft-ietf-mls-extensions-11
    date: 2026-11-02
  ULID:
    title: "Universally Unique Lexicographically Sortable Identifier"
    target: https://github.com/ulid/spec

informative:
  RFC6750:
  RFC6838:
  RFC9457:
  JMAP-METADATA:
    title: "JMAP Object Metadata"
    author:
      - fullname: Mauro De Gennaro
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-metadata-05
    date: 2026-10-28
  MLS-RATCHET-TREE-OPTIONS:
    title: "Ways to convey the Ratchet Tree in Messaging Layer Security"
    author:
      - fullname: Rohan Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mls-ratchet-tree-options-02
    date: 2026-11-01
  MIMI-MESSAGE-STATUS:
    title: "A Message Status format for the More Instant Messaging Interoperability (MIMI) content format"
    author:
      - fullname: Rohan Mahy
    seriesinfo:
      Internet-Draft: draft-mahy-mimi-message-status-02
    date: 2026-11-01
  JMAP-CHAT-FED:
    title: "JMAP Chat Federation"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026
  JMAP-VTC:
    title: "JMAP for Video/Voice Teleconferencing"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  MIMI-CALLS:
    title: "Media Keys for Calls in More Instant Messaging Interoperability (MIMI)"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-mimi-calls-00
    date: 2026
  JMAP-OBJ-HISTORY:
    title: "JMAP Object History"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-object-history-03
    date: 2026-11-02

--- abstract

This document defines how a gateway converts messages between the IETF More Instant Messaging Interoperability (MIMI) content format and JMAP Chat. It specifies how MIMI message identifiers, edits, topics, media reactions, message statuses, and calls to join map onto JMAP Chat Messages, so that independent gateways derive the same JMAP Chat identifiers for the same MIMI messages without coordination. It also defines a JMAP capability with which a JMAP Chat server acts as an MLS client's MIMI provider: it publishes KeyPackages, fetches key material, asks for and gives consent to be added to rooms, creates and updates rooms, records that the account has left a room, fetches GroupInfo, and sends and delivers end-to-end encrypted MIMI messages without decrypting them.

--- middle

# Introduction

{{JMAP-CHAT}} and MIMI ({{MIMI-PROTOCOL}}, {{MIMI-CONTENT}}) address different layers of messaging interoperability: MIMI federates large providers through hub-and-spoke rooms, while JMAP Chat defines a client-server API and, in {{JMAP-CHAT-FED}}, mailbox-to-mailbox federation. A deployment that participates in both needs a gateway that converts messages between them.

This document collects the conversion rules that such a gateway follows. Servers and clients that never convert MIMI messages are not affected by it. Keeping these rules separate from {{JMAP-CHAT}} lets them track revisions of the MIMI drafts without revising the core specification.

MIMI leaves the client-server interface out of scope. This document also fills that gap for JMAP: {{capability}} through {{mls-messages}} define the `urn:ietf:params:jmap:chat:mimi` capability, with which an MLS {{RFC9420}} client uses a JMAP Chat server as its MIMI provider. Such a server routes MLS messages between the client and the MIMI hub and never decrypts them. The conversion rules of {{message-ids}} through {{status}} apply only to a party that decrypts MIMI messages and stores the result as JMAP Chat Messages.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{JMAP-CHAT}} and {{MIMI-CONTENT}} is used throughout. A "converting party" is any server or client that converts `application/mimi-content` messages to JMAP Chat Messages.

Terminology from {{RFC8620}}, {{RFC9420}}, and {{MIMI-PROTOCOL}} (hub, follower, room, participant list) is also used. A "MIMI provider" is a server that advertises the `urn:ietf:params:jmap:chat:mimi` capability. A MIMI provider is the hub of rooms created through it ({{room-create}}) and a follower in rooms hosted elsewhere.

Every byte-string property defined in {{capability}} through {{mls-messages}} is the unpadded base64url encoding ({{RFC4648, Section 5}}) of the exact octets of the named MLS {{RFC9420}} or MIMI {{MIMI-PROTOCOL}} structure in the TLS presentation language encoding those documents use. Servers MUST NOT re-encode these octets, because clients sign some of them and verify signatures over others.

The encoding is that of the whole structure. For a structure that is a variable-size vector, such as a KeyPackageRef or a ProposalRef (`opaque HashReference<V>`, {{RFC9420, Section 5.2}}), it begins with the vector's variable-length length prefix ({{RFC9420, Section 2.1.2}}). For example, a KeyPackageRef whose 32-octet hash value is the octets 0x00 through 0x1f is the 33 octets

~~~
20 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e
0f 10 11 12 13 14 15 16 17 18 19 1a 1b 1c 1d 1e 1f
~~~

and its `keyPackageRef` is `"IAABAgMEBQYHCAkKCwwNDg8QERITFBUWFxgZGhscHR4f"`.

# Message Identifiers {#message-ids}

A converting party MUST derive each message's `senderMsgId` as the ULID {{ULID}} whose 48-bit time component is the hub's acceptance time of the message in milliseconds (the FanoutMessage `timestamp` ({{MIMI-PROTOCOL, Section 5.5}}), or, for the party's own message, the `accepted_timestamp` of the SubmitMessageResponse ({{MIMI-PROTOCOL, Section 5.4}})) and whose 80-bit random component is octets 1 through 10 of its MIMI message ID, counting from 0: the ten octets that follow the hash algorithm octet. Every party that receives a MIMI message therefore derives the same `senderMsgId` without coordination.

A MIMI reference to another message (`replaces` or `inReplyTo` {{MIMI-CONTENT}}, the latter also naming the message a reaction applies to) carries only a MIMI message ID. A converting party resolves it to the Message in the same Chat whose `senderMsgId` random component, or one of whose aliases ({{edits}}), equals octets 1 through 10 of that ID. When an `inReplyTo` reference resolves, the converted Message's `replyTo` is that Message's `id`. A reference to a message the party has not stored does not resolve, and an unresolved `inReplyTo` leaves `replyTo` absent.

# Senders {#senders}

The sender's MIMI user URI ({{MIMI-PROTOCOL}}) is the identity of the credential ({{RFC9420, Section 5.3}}) in the MLS sender's leaf, whatever the credential's type ({{keypackage}}). A converted Message's `senderId` is the `id` of the local ChatContact ({{JMAP-CHAT}}) whose `identity` is that user URI. If no ChatContact has that `identity`, the converting party MUST create one, with that `identity` and an `id` the server assigns, before storing the Message. When that user URI equals the account's `ownerUserId` ({{JMAP-CHAT}}), the same identity mapping {{JMAP-CHAT}} and {{JMAP-CHAT-FED}} use, `senderId` is the `accountId` ({{JMAP-CHAT}}) and no ChatContact is created.

# Edits {#edits}

A MIMI edit, whose `replaces` field always names the first version of the message {{MIMI-CONTENT}}, maps to an edit of the message to which `replaces` resolves ({{message-ids}}). The converting party records octets 1 through 10 of each edit's own MIMI message ID as an alias of that message, so a MIMI reply or reaction that names a later version refers to the message as a whole.

# Topics {#topics}

MIMI's `topicId` {{MIMI-CONTENT}}, a byte string, maps to the Message `topicId` field ({{JMAP-CHAT}}) as UTF-8 in both directions: the JMAP `topicId` is the MIMI `topicId` octets decoded as UTF-8, and the MIMI `topicId` is the UTF-8 encoding of the JMAP `topicId`. A MIMI `topicId` that is empty, is not valid UTF-8, or exceeds 255 octets maps to no `topicId` and no `subject`; the converting party still converts the message.

# Reactions {#reactions}

A MIMI media reaction (a reaction that is not an emoji) is carried in JMAP Chat as a reply containing the media, as {{MIMI-CONTENT}} permits; JMAP Chat Reactions ({{JMAP-CHAT}}) remain emoji and custom emoji.

# Calls {#calls}

A MIMI part whose disposition is `session` ({{MIMI-CONTENT, Section 5.10}}) carries an endpoint for a multimedia call, not content to fetch. A converting party maps such a part whose `url` is an `https` or `wss` URI to a MessageAction ({{JMAP-CHAT}}) of type `urn:ietf:params:jmap:chat:cap:vtc`, with the part's `url` as the action's `uri` and its `description` as the action's `label`, and maps such a MessageAction back to a `session` part of external cardinality. It does not fetch the URL, and ignores the part's `expires`, `size`, `encAlg` and `hashAlg`, which Section 5.10's own example leaves at zero or none. A `session` part whose `url` carries any other scheme maps to a line of `body` instead, so that the URI still reaches the user; the converting party still converts the message. A converting party sends no MessageAction type other than this one over MIMI, and refuses a MessageAction that carries `metadata`, since {{MIMI-CONTENT}} gives a `session` part nowhere to put it.

The `session` disposition covers "a description of a multimedia session, or a URI used to join one" ({{MIMI-CONTENT}}). The paragraph above maps the second. The first is call signalling, and is mapped as follows.

An inline `session` part whose content type is `application/sdp` ({{RFC8866}}) carries a session description for a call held in the room it is sent to, not content for a person to read. A converting party MUST NOT put such a part's content in a Message `body` and MUST NOT map it to a MessageAction: a client that cannot hold a call has nothing to do with a session description, and a client that can reaches it through {{JMAP-VTC}} rather than as a message. In the room of a call that has one of its own ({{MIMI-CALLS}}), which is parent-dependent on the room of the Chat the call belongs to ({{MIMI-ROOM-POLICY, Section 5}}), the converting party delivers such a part to that capability and converts no Message for it. In any other room it discards the part, having nowhere to deliver it.

Carrying a session description this way authenticates it. The part travels in the room's MLS group, so its sender is the authenticated MLS sender and the certificate fingerprints ({{RFC8122}}) it carries are bound to that sender's credential. A call signalled this way therefore needs no separate fingerprint comparison between its participants, which is what {{JMAP-VTC}}'s `e2eeFingerprint` is for in a call signalled some other way.

A `session` part that is neither a join URI nor an `application/sdp` session description maps as an ordinary part of its cardinality and content type, so its content still reaches the user.

Nothing here says how a call's media is keyed. {{MIMI-CONTENT}}, Section 5.10 puts "further discussion of calling and conferencing functionality" out of scope, and {{JMAP-VTC}}'s `e2eeEnabled` is a state signal whose mechanism is media-layer-defined. {{MIMI-CALLS}} says where a call's media keys come from when the call has a MIMI room of its own: they are exported from that room's MLS group, so the call's media is confined to the room's membership and to nothing smaller. The mapping in this section is unaffected either way, because an `https` or `wss` join URI names an endpoint outside MIMI, which such a key does not reach.

# Message Status {#status}

A MIMI message status report {{MIMI-MESSAGE-STATUS}} is an application message from one member that gives a status for each of several message IDs. A converting party maps it to and from the JMAP Chat read state ({{JMAP-CHAT}}) as follows. It resolves each message ID as for any other MIMI reference ({{message-ids}}) and ignores an entry that does not resolve.

A report from another member about a message the owner sent:

* `delivered` (1) sets that member's `deliveredAt` in the Message's `deliveryReceipts`.
* `read` (2) sets that member's `readAt` there, with ReadDisposition `"displayed"`, and also `deliveredAt` if it is not set.
* `unread` (0) clears that member's `readAt` and `readDisposition`.
* `expired` (3), `deleted` (4), `hidden` (5) and `error` (6) are not read acknowledgements: a converting party does not set `readAt` or a ReadDisposition for them, and does not change the owner's own copy. A status of 7 or more is ignored.

The time recorded is the time the converting party received the report, since the report carries none. The aggregate `deliveryStatus` follows from `deliveryReceipts` as {{JMAP-CHAT}} describes.

A report from another of the owner's own clients is the owner's own read state: `read` (2) sets `readAt` on the named Messages, and the converting party MAY advance the owner's ReadPosition to the latest of them. Other statuses from the owner's own clients change nothing.

When the owner sets `readAt` on a Message another member sent, the converting party MAY send a report with `read` (2) for it, and on receiving a message it MAY send `delivered` (1). It does not send either when the owner's effective `receiptSharing` is `false`, or when the room's status notification policy ({{MIMI-ROOM-POLICY, Section 6.1}}) forbids them. It MAY combine several messages in one report.

# Moderation {#moderation}

A converting party that is itself a member of a room's MLS group, and so builds the room's commits, maps the moderation actions of {{JMAP-CHAT}} on a group Chat bound to that room to room-policy actions ({{MIMI-ROOM-POLICY, Section 8.1}}). Each action becomes one commit that the hub accepts or refuses under the room's policy; a refusal is a `forbidden` SetError, and the Chat's `members` change when the commit is accepted.

* `removeMembers` naming another user removes that user from the participant list and removes all of the user's clients from the group (`canRemoveParticipant`).
* `updateMemberRoles` changes the user's role in the participant list (`canChangeUserRole`). The role `"admin"` maps to the room's role named "admin" and `"member"` to the room's role named "member"; a room without such a role refuses the change.
* Creating a ChatBan ({{JMAP-CHAT}}) gives the user the room's role named "banned" and removes all of the user's clients, in one commit (`canBan`). Destroying it gives the user the member role again without adding any client (`canUnban`).
* The `kickMembers` patch key of `Chat/set` (String\[], ChatContact ids), which this document adds for Chats bound to a room, removes the listed users' clients from the group but keeps them in the participant list, so that they can rejoin (`canKick`). {{JMAP-CHAT}} has no such action, because there a member and its clients are not separate.

Role names are matched against `role_name` in the room's `roles_list` component ({{MIMI-ROOM-POLICY, Section 3}}), octet for octet. A room created through `Room/create` ({{room-create}}) SHOULD name its roles "member", "admin" and "banned", so that these mappings work.

This section does not apply to a MIMI provider's own Chats with a `roomId` ({{chat-roomid}}), whose membership changes only through `Room/update`.

# Reporting Abuse from a Converting Party {#report-abuse}

A converting party keeps, for each Message it converts from an application message, the application data as received, the hub's acceptance time, and the message's frank when it has one ({{MIMI-PROTOCOL, Section 5.4.1.3}}: "The receiver needs to store the server_frank ... and context fields"). It keeps them as long as it keeps the Message.

`Message/reportAbuse` reports Messages as abusive. Its arguments are `accountId`; `messageIds` (Id\[]), Messages of one Chat bound to a room, all sent by one other user; `reasonCode` (UnsignedInt, the AbuseType); and `note` (String, optional). The converting party builds one AbuseReport naming the account's `userUri` as `reportingUser` and the sender as `allegedAbuserUri`. A Message with a frank becomes an AbusiveMessage. A Message without one, as in a room without franking, is carried in the report's `abuse_extensions` as an unverified message ({{unverified-messages}}), because an AbusiveMessage needs a frank. The converting party sends the report as `Room/reportAbuse` does ({{room-report-abuse}}) and answers with the same `responseCode`. It MUST reject a call that names the owner's own Messages, Messages of more than one Chat or more than one sender, or Messages it has nothing kept for, with `invalidArguments`.

## Unverified Messages {#unverified-messages}

The `unverified_abusive_messages` component (TBD; mimid uses the private-use value 0xF020) of an AbuseReport's `abuse_extensions` carries reported messages that have no frank:

~~~ tls-presentation
struct {
    opaque message_content<V>;
    uint64 accepted_timestamp;
} UnverifiedAbusiveMessage;

struct {
    UnverifiedAbusiveMessage messages<V>;
} UnverifiedAbusiveMessages;
~~~

`message_content` is the application data as received and `accepted_timestamp` the hub's acceptance time in milliseconds. A hub cannot verify these messages; it MAY record them as unverified alongside the AbusiveMessages it verifies. A hub that does not know the component ignores it, as for any unknown component.

# The urn:ietf:params:jmap:chat:mimi Capability {#capability}

Servers supporting the client interface of this document advertise the `urn:ietf:params:jmap:chat:mimi` capability in the JMAP Session object. This capability is meaningful only when `urn:ietf:params:jmap:chat` is also present.

A server MAY authenticate clients with OAuth 2.0 bearer tokens ({{RFC6750}}); this document does not require any particular mechanism.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:mimi"]` at the session level is an empty object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:mimi"]` is a JSON object with the following fields:

`userUri` (String):
: The account owner's MIMI user URI ({{MIMI-PROTOCOL}}). The server uses it as `requestingUser` and `sendingUri` on the owner's behalf.

`roomIdPrefix` (String):
: The prefix of every room ID this server accepts in `Room/create` ({{room-create}}), for example `"mimi://a.example/r/"`.

`hubSender` (String):
: The ExternalSender ({{RFC9420, Section 12.1.8.1}}) with which this server signs as hub of the rooms it hosts.

`cipherSuites` (UnsignedInt\[]):
: The MLS cipher suites ({{RFC9420, Section 17.1}}) this server accepts in KeyPackages and new rooms. MUST NOT be empty.

`credentialTypes` (UnsignedInt\[], optional):
: The MLS credential types ({{RFC9420, Section 17.5}}) this server accepts in a KeyPackage's leaf ({{keypackage}}). A client that holds a credential whose type is not listed knows not to offer it, rather than learning so from a rejected `KeyPackage/set`. When absent, a client learns which types the server accepts only by offering one. The server authorizes a KeyPackage by the credential's identity in either case ({{keypackage}}).

`maxKeyPackages` (UnsignedInt):
: The maximum number of KeyPackage records, last-resort KeyPackages included, the server stores for this account.

`maxKeyPackageLifetime` (UnsignedInt, optional):
: The longest leaf node lifetime ({{RFC9420, Section 7.2}}), from `not_before` to `not_after`, in seconds, that the server accepts in a KeyPackage ({{keypackage}}). Absent if the server states no limit.

`maxClients` (UnsignedInt, optional):
: The maximum number of signature keys the server binds to this account ({{client}}), that is, the number of the owner's clients. Absent if the server states no limit.

`assetUrl` (String, optional):
: A URL template ({{!RFC6570}}, level 1) with the variable `blobId`. A blob the account uploaded ({{RFC8620, Section 6.1}}) is served, without authentication, at the URL this forms, so that the URL can be the `url` of an ExternalPart ({{MIMI-CONTENT, Section 4.5}}) that other providers' clients fetch. The blob id is the only credential for it, so clients upload only encrypted content. Absent if the server does not store attachments.

`assetDownloadUrl` (String, optional):
: A URL template ({{RFC6570}}, level 1) with the variables `roomId` and `url`. A GET to the URL this forms, authenticated as for the API, returns the content at `url` for a message of room `roomId`: the server's own asset directly, and another provider's through the room's hub (`proxyDownload`, {{MIMI-PROTOCOL, Section 5.10.2}}), so that the client never contacts another provider. The server MUST refuse a `roomId` that no Chat of the account has. Absent if the server does not fetch attachments.

  The server answers a `roomId` that no Chat of the account has, and a `url` it cannot find, with status 404; a `url` the room's hub or the asset's provider refuses with 403; and a provider that cannot be reached with 502. It MAY add a problem details body ({{RFC9457}}).

`assetLifetime` (UnsignedInt, optional):
: The number of seconds after its upload that a blob stays available at `assetUrl`. The server may delete it after that. A client that puts the URL in an ExternalPart MAY set its `expires` ({{MIMI-CONTENT, Section 4.5}}) to the upload time plus this value, so that recipients know when the URL stops working. Absent if the server states no limit; present only with `assetUrl`.

`maxMlsObjectSize` (UnsignedInt, optional):
: The largest MLS or MIMI structure, in octets before base64url encoding, that the server accepts in any argument of this document's methods. A client that would send a larger one MAY choose a ratchet tree option ({{MLS-RATCHET-TREE-OPTIONS}}) that keeps the tree out of the message.

`messageRetention` (UnsignedInt, optional):
: The number of seconds the server keeps a Message in a Chat with a `roomId` ({{chat-roomid}}) after storing it. The server MAY destroy such a Message after that time. Absent if the server states no limit.

  A client that has not read a Chat's Messages for longer than this may have missed some, handshake messages among them, and then cannot follow the room's MLS group. It may find that the server cannot calculate changes from its saved Message state, and that the Message it would use as `anchor` ({{message-order}}) is gone (`anchorNotFound`, {{RFC8620, Section 5.5}}). Such a client MAY rejoin the room by external join ({{groupinfo-fetch}}, {{room-update}}); the application messages it missed are lost to it.

A server that advertises this capability MUST include `"application/mimi-fanout"` in `supportedBodyTypes` ({{JMAP-CHAT}}).

A server that limits how often an account calls the methods of this document answers a call over its limit with the method-level error `rateLimited` ({{JMAP-CHAT}}). The error object MAY include `retryAfter` (UnsignedInt), the number of seconds after which the server expects the call can succeed. A client that receives `rateLimited` waits before it calls again, at least `retryAfter` seconds when it is present.

When a server rejects a call, or a create, update or destroy, under a rule of this document, it MAY say in the error's `description` ({{RFC8620}}) which check failed.

A client of this capability that does not convert content needs `StateChange` events for the Message, Chat, Consent and KeyPackage types; it MAY ask for only these in the event source's `types`.

# KeyPackages {#keypackage}

A KeyPackage record holds one MLS KeyPackage that the server publishes for one of the owner's clients and hands out in response to MIMI key material requests ({{MIMI-PROTOCOL, Section 5.2}}).

## KeyPackage Object Fields

`id` (Id, immutable, server-set):
: The id of the record.

`keyPackage` (String, immutable):
: The KeyPackage ({{RFC9420, Section 10}}).

`keyPackageRef` (String, immutable, server-set):
: The KeyPackageRef of `keyPackage` ({{RFC9420, Section 5.2}}).

`lastResort` (Boolean, immutable):
: `true` when `keyPackage` is the client's last-resort KeyPackage ({{RFC9420, Section 16.8}}), which its `app_data_dictionary` extension ({{MLS-EXTENSIONS, Section 4.6}}) marks with a `last_resort_key_package` component ({{MLS-EXTENSIONS, Section 6.4}}); `false` otherwise. The server sets it from `keyPackage`. A create MAY include it; the server MUST reject a create whose `lastResort` differs from the value derived from `keyPackage` with `invalidProperties` naming `lastResort`.

## KeyPackage/get

Standard JMAP `/get` ({{RFC8620, Section 5.1}}).

## KeyPackage/changes

Standard JMAP `/changes` ({{RFC8620, Section 5.2}}). A KeyPackage handed out appears in `destroyed`.

## KeyPackage/set

Standard JMAP `/set` ({{RFC8620, Section 5.3}}), except that `update` is not supported: the server MUST reject every update with a SetError of type `forbidden`.

On create, the server MUST reject the record with `invalidProperties` naming `keyPackage` if `keyPackage` does not parse as a KeyPackage, if its signature does not verify, if its cipher suite is not in `cipherSuites`, if its leaf node lifetime has expired or is longer than the server allows (`maxKeyPackageLifetime`), or if the credential identity is not `userUri`.

The leaf node's signature key identifies the client and binds the KeyPackage to the account: the first KeyPackage the server accepts with a given signature key binds that signature key to the account and creates a Client record for it ({{client}}). The server SHOULD reject, with `invalidProperties` naming `keyPackage`, a KeyPackage whose signature key is bound to another account.

This document does not constrain the leaf credential's type ({{RFC9420, Section 5.3}}). What it requires is that the credential's identity be the account's `userUri`: that is what authorizes the KeyPackage for the account, and it holds for a Basic credential, for an X.509 credential, and for any credential type registered later. A server MAY reject, with `invalidProperties` naming `keyPackage`, a credential whose type it cannot verify, and SHOULD advertise the types it accepts in `credentialTypes` ({{capability}}).

The server SHOULD reject, with `invalidProperties` naming `keyPackage`, a KeyPackage whose KeyPackageRef equals that of a KeyPackage it stores or has handed out, and MUST reject a create that would exceed `maxKeyPackages` with `overQuota`. When `maxClients` signature keys are already bound to the account, it rejects with `overQuota` a create whose signature key is not one of them.

Each client has at most one last-resort KeyPackage. Creating a last-resort KeyPackage for a client destroys that client's previous last-resort KeyPackage.

## Handing Out KeyPackages {#keypackage-claim}

When this server answers a key material request for the owner ({{MIMI-PROTOCOL, Section 5.2}}) and the requester has consent ({{consent-received}}), it returns at most one KeyPackage for each of the owner's clients. For each client it returns a KeyPackage compatible with the request that is not last-resort and destroys that record in the same transaction, so that no KeyPackage is handed out twice. When the client has no such KeyPackage, the server returns the client's compatible last-resort KeyPackage, if any. A client's last-resort KeyPackage is compatible whenever its other KeyPackages would be. {{MIMI-PROTOCOL, Section 5.2}} forbids repeating key material across responses, while {{RFC9420, Section 16.8}} and {{MLS-EXTENSIONS, Section 6.4}} allow a last-resort KeyPackage to be used more than once. A server that follows {{MIMI-PROTOCOL, Section 5.2}} returns a last-resort KeyPackage once and does not return it again; it MAY keep the record, so that the owner sees it until the client replaces it. A server MAY instead keep it and return it again; it then departs from {{MIMI-PROTOCOL, Section 5.2}}.

When every client the server lists has `clientStatus` `keyMaterialExhausted`, which happens only when none of them has a last-resort KeyPackage the server returns, {{MIMI-PROTOCOL, Section 5.2}} gives no fitting user code: `partialSuccess` means that at least one client got key material. The server answers with `userStatus` `noCompatibleMaterial`, the one failure code with which the response may list the clients, and lists each client with `clientStatus` `keyMaterialExhausted`. A later revision of {{MIMI-PROTOCOL}} may define a code for this case.

The `clientUri` the server returns for each client ({{MIMI-PROTOCOL, Section 5.2}}) is implementation-defined, but MUST be stable and unique per leaf: the server MUST return the same `clientUri` in every response for a given signature key ({{keypackage}}), and different `clientUri` values for different signature keys.

The server MUST keep the KeyPackageRef of every KeyPackage it hands out, with the account it belongs to, until a Welcome naming it has been delivered or its leaf node lifetime has passed. A destroyed record appears in a new state for the KeyPackage type, so a client that receives the `StateChange` can replenish its KeyPackages.

# Clients {#client}

A Client record stands for one signature key bound to the account, that is, one of the owner's MLS clients. The server creates it when it binds a signature key, on a KeyPackage/set create ({{keypackage}}) or when the hub accepts an external Commit ({{room-update}}). Destroying it ends the binding, which frees a place under `maxClients`.

## Client Object Fields

`id` (Id, immutable, server-set):
: The id of the record.

`signatureKey` (String, immutable, server-set):
: The client's SignaturePublicKey ({{RFC9420, Section 5.1.1}}), the signature key of its leaf nodes.

## Client/get

Standard JMAP `/get` ({{RFC8620, Section 5.1}}).

## Client/changes

Standard JMAP `/changes` ({{RFC8620, Section 5.2}}).

## Client/set

Standard JMAP `/set` ({{RFC8620, Section 5.3}}), except that only `destroy` is supported: the server MUST reject every create and update with a SetError of type `forbidden`.

Destroying a Client record unbinds its signature key from the account. The server destroys the account's KeyPackage records with that signature key, and the key no longer counts toward `maxClients` or passes the check of `Room/create` ({{room-create}}). A later KeyPackage/set create or external Commit with the same signature key binds it again, as for a new client.

A server MAY unbind a signature key that has had no KeyPackage record and has sent nothing through this capability for a period it chooses, and destroy its Client record, as if the owner had destroyed it.

Unbinding does not remove the client from any room. The server is not a member of the rooms' MLS groups and cannot remove its leaves, and until a commit removes them the client can still decrypt those rooms' messages. A client that destroys the record of another of the owner's clients, such as a lost device, removes that client's leaves, with `Room/update` ({{room-update}}), from the rooms they share. Nor does unbinding end the removed client's access to the account; that is a matter for the account's authentication.

# KeyMaterial/fetch {#keymaterial-fetch}

KeyMaterial/fetch obtains KeyPackages for another user's clients so that the owner can add that user to a room ({{MIMI-PROTOCOL, Section 5.2}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account bound to its room ({{chat-roomid}}) to which the target user is to be added.

`request` (String):
: A KeyMaterialRequest signed by the client.

A client SHOULD copy the room's `required_capabilities` extension ({{RFC9420, Section 11.1}}) into the request's `requiredCapabilities`. `requiredCapabilities` cannot name app components ({{MLS-EXTENSIONS, Section 5}}), so a server that knows the room MAY also skip KeyPackages whose leaf lacks a component named in the room's `app_components`, and report such a client as `nothingCompatible`.

The server MUST reject the call with `invalidArguments` if `request` does not parse as a KeyMaterialRequest, if its `requestingUser` is not `userUri`, if its `roomId` is not `roomId`, or if no Chat in the account is bound to that `roomId`. Otherwise the server obtains a KeyMaterialResponse. When this server is the room's hub, it answers from its own KeyPackage store ({{keypackage-claim}}) if the target user is local and asks the target provider otherwise. When this server is not the room's hub, it MUST send the request to the hub, even if the target user is local, because a hub that did not see the request cannot forward the Welcome to the target provider ({{MIMI-PROTOCOL, Section 5.2}}). If the provider it asks cannot be reached, the server returns `serverUnavailable`.

The response has the following arguments. All but `response` are decoded from the KeyMaterialResponse:

`accountId` (Id):
: The account used.

`userStatus` (String):
: The `userStatus` code, written as its name in {{MIMI-PROTOCOL, Section 5.2}} (for example `"partialSuccess"`).

`userUri` (String):
: The target user URI.

`clients` (Object\[]):
: One object per ClientKeyMaterial, with the fields `clientStatus` (String, the code name), `clientUri` (String), `keyPackage` (String, present when `clientStatus` is `"success"`), and `clientCapabilities` (String, the MLS Capabilities, present when the provider included it).

`response` (String):
: The whole KeyMaterialResponse, unchanged.

# Consent {#consent}

A provider can refuse to hand out a user's KeyPackages to a user of another provider unless the target user has consented ({{MIMI-PROTOCOL, Section 8}}). A Consent record lets the owner see and change explicit consent ({{MIMI-PROTOCOL, Section 5.7}}): requests the owner sent or received, and decisions the owner made or received. How a server determines consent implicitly, for example from contacts or shared rooms, is a deployment choice outside this document.

Each record has one ConsentScope ({{MIMI-PROTOCOL, Section 5.7}}): a requester, a target, and optionally a room. The owner is either the requester or the target of every record, not both. An account holds at most one record per ConsentScope.

## Consent Object Fields

`id` (Id, immutable, server-set):
: The id of the record.

`requesterUri` (String, immutable):
: The user who would fetch the target's KeyPackages.

`targetUri` (String, immutable):
: The user whose KeyPackages would be fetched.

`roomId` (String\|null, immutable, default: null):
: The room the scope is limited to, or `null` for any room.

`requested` (Boolean):
: `true` while a request for this scope is outstanding: one the owner sent, when the owner is the requester, or one the owner received, when the owner is the target. Server-set when the owner is the target.

`decision` (String\|null, default: null):
: `"grant"`, `"revoke"`, or `null` when no decision is known. A revoke also covers a denial given before any request. When the owner is the target, this is the owner's decision. When the owner is the requester, it is the decision the target's provider sent, and it is server-set.

`updated` (UTCDate, server-set):
: When the record last changed.

## Consent/get

Standard JMAP `/get` ({{RFC8620, Section 5.1}}).

## Consent/changes

Standard JMAP `/changes` ({{RFC8620, Section 5.2}}). A client that watches the Consent type in `StateChange` events learns of requests the owner receives.

## Consent/query

Standard JMAP `/query` ({{RFC8620, Section 5.5}}). A FilterCondition has the following properties, each optional:

`asTarget` (Boolean):
: Whether the owner is the target (`true`) or the requester (`false`).

`requested` (Boolean):
: The value of `requested`.

`decision` (String):
: The value of `decision`; the condition matches no record whose `decision` is `null`.

The only sort property is `updated`.

## Consent/set {#consent-set}

Standard JMAP `/set` ({{RFC8620, Section 5.3}}), with the following rules. The server rejects, with `invalidProperties`, a create in which neither or both of `requesterUri` and `targetUri` is `userUri`, and a create or update that sets a property this section does not let the owner set. It rejects a create whose ConsentScope matches an existing record with `alreadyExists` ({{RFC8620, Section 5.4}}), giving that record's id in `existingId`.

When the owner is the requester:

- A create asks the target for consent. `requested` is `true` or omitted, and `decision` is `null` or omitted. The server sends a requestConsent with operation `request` for the record's ConsentScope.
- An update can change `requested`. Setting it to `false` withdraws the request: the server sends a requestConsent with operation `cancel`. Setting it to `true` sends a new request.
- A destroy removes the record. If `requested` is `true`, the server first sends a `cancel`.

When the owner is the target:

- A create records a decision that answers no request, such as a grant made in advance or a denial. `decision` is `"grant"` or `"revoke"`.
- An update can change `decision`. Setting it to `"grant"` or `"revoke"` sends an updateConsent with that operation, and the server sets `requested` to `false`. A server MAY skip sending a revoke, because a revoke tells the requester that it was refused ({{MIMI-PROTOCOL, Section 5.7}}). Setting `decision` to `null` sends nothing, and the server goes back to its own policy for that scope.
- A destroy removes the record and sends nothing.

The server sends each ConsentEntry ({{MIMI-PROTOCOL, Section 5.7}}) to the other user's provider before it answers. When the other user is an account of this server, the server changes that account's record as if it had sent and received the ConsentEntry. In a grant, the server MAY include KeyPackages of the owner's clients, chosen and destroyed as in {{keypackage-claim}}; otherwise the list is empty. If the other provider answers 404, the server rejects the create or update with `invalidProperties`, naming the other user's URI. If the other provider cannot be reached or answers with another error, the server rejects it with the SetError type `consentNotSent`, and the record does not change.

## Consent From Other Providers {#consent-received}

When this server accepts a requestConsent for the owner as target, it creates or updates the record for its ConsentScope with `requested` set to `true`, and a `cancel` sets it to `false`. When it accepts an updateConsent for the owner as requester, it creates or updates the record for its ConsentScope with that `decision`, and sets `requested` to `false`. A server MAY destroy a record whose `requested` is `false` and whose `decision` is `null`.

When this server answers a key material request for the owner from a user of another provider ({{keypackage-claim}}), a record with the owner as target whose ConsentScope matches the request's requester and room decides consent: a record for that room before a record for any room. A `"grant"` allows the request, and a `"revoke"` refuses it with `noConsent` or `noConsentForThisRoom` as {{MIMI-PROTOCOL, Section 5.2}} describes. With no matching decision, the server applies its own policy.

# Rooms {#rooms}

## Chat.roomId {#chat-roomid}

When this capability is present, the Chat object ({{JMAP-CHAT}}) has two more fields:

`roomId` (String, immutable, server-set, optional):
: The MIMI room ID of the room this Chat is bound to. Absent for Chats not bound to a MIMI room.

`roomBound` (Boolean, server-set, optional):
: `true` while the Chat is bound to its room, `false` once it is unbound. Present exactly when `roomId` is.

`parentRoomId` (String|null, server-set, optional):
: The MIMI room ID of the parent of the room this Chat is bound to, when that room is parent-dependent ({{MIMI-ROOM-POLICY, Section 5}}); `null` when it is not. Present exactly when `roomId` is, and only from a server that knows the room's policy.

A parent-dependent room is the shape a call with a room of its own takes ({{MIMI-CALLS}}), and `kind` ({{JMAP-CHAT}}) has no value that describes a call, so a Chat bound to such a room is one a client SHOULD NOT list among the user's conversations. It is still a Chat: a Chat bound to a room is that room's message container and the handle its messages are sent and received through, so omitting it would leave the room with no messages rather than with no conversation. `parentRoomId` is how a client tells the two apart.

Only a server that holds the room's policy can set `parentRoomId`. {{MIMI-ROOM-POLICY, Section 5}} puts a parent-dependent room on the same Hub as its parent, so the Hub's own server can set it, and a server following the room for one of its users cannot: it holds no group state and cannot read a Welcome addressed to a client. Such a server omits the field, and a client that needs to know reads the room's `base_room_policy` from the group itself.

A Chat with a `roomId` has kind `"group"`. Its participant list, roles, and metadata are MLS group state, changed only through `Room/update` ({{room-update}}), and its messages are sent with `Room/send` ({{room-send}}). The server MUST reject, with `forbidden`, a `Chat/set` update on such a Chat that sets `name`, `description`, or `avatarBlobId` or that uses the `addMembers`, `removeMembers`, or `updateMemberRoles` patch key ({{JMAP-CHAT}}), and clients take the room's name and participants from the MLS group state rather than from these fields. The server sets `name` to the `room_name` of the room metadata ({{MIMI-PROTOCOL, Section 7.6}}) when it knows it and to the empty string otherwise.

The server does not mirror the participant list into ChatMember records: `members` of a Chat with a `roomId` is the empty list, whether `Room/create` ({{room-create}}) or a Welcome ({{fanout-messages}}) created it.

A Chat is bound to its room from its creation until it is unbound. The account leaves the room's MLS group when a commit removes from the group every leaf whose credential identity is the account's `userUri` ({{keypackage}}). A server that is not a member of the group cannot in general tell which clients a commit removes, so the client unbinds the Chat: a client that processes a commit removing the last of its account's clients SHOULD then call `Room/unbind` ({{room-unbind}}). A server that can determine from a commit it has stored that the commit removed the last of the account's clients MAY unbind the Chat itself. An unbound Chat keeps its `roomId` and its Messages. The server stores no further FanoutMessage in it, and `KeyMaterial/fetch` and `Room/send` treat it as if no Chat in the account had that `roomId`; `Room/update` accepts only an external Commit for it ({{room-update}}). A later Welcome for the account binds the Chat again ({{fanout-messages}}), and so does an external Commit the hub accepts.

Binding and unbinding a Chat change its Chat state ({{RFC8620, Section 5.1}}).

A client MAY record per-user state about the room, such as whether the user accepted it, in the Chat's `privateMetadata` ({{JMAP-METADATA}}), so that the user's other clients see it.

## Room/create {#room-create}

Room/create creates a room with this server as hub ({{MIMI-PROTOCOL, Section 3.1}}) and a Chat bound to it. The client first creates the MLS group with no member other than its own client.

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The room ID the client chose for the room.

`groupInfo` (String):
: An MLSMessage ({{RFC9420, Section 6}}) of wire format `mls_group_info` carrying the group's GroupInfo, which MUST NOT contain the `ratchet_tree` extension ({{MIMI-PROTOCOL, Section 5.6}}).

`ratchetTree` (String):
: The group's ratchet tree, a `ratchet_tree` vector of `optional<Node>` entries ({{RFC9420, Section 12.4.3.3}}).

The server MUST reject the call with `invalidArguments` unless all of the following hold:

1. The GroupInfo signature verifies, and its cipher suite is in `cipherSuites`.
2. The `ratchetTree` matches the tree hash in the GroupInfo's GroupContext and contains exactly one leaf, whose credential identity is `userUri` and whose signature key is bound to the account ({{keypackage}}).
3. The `external_senders` extension contains exactly one ExternalSender, equal to `hubSender`.
4. The participant list ({{MIMI-PROTOCOL, Section 7.5}}) contains exactly one participant, `userUri`.
5. `roomId` is `roomIdPrefix` followed by the unpadded lowercase base32 encoding ({{RFC4648, Section 6}}) of 16 octets, and no room this server hosts has that room ID. If the group carries room metadata ({{MIMI-PROTOCOL, Section 7.6}}), its `room_uri` is `roomId`.
6. The group ID is 16 octets and no room this server hosts uses it.
7. The group's `app_data_dictionary` extension ({{MLS-EXTENSIONS, Section 4.6}}) contains a `franking_agent` component ({{MIMI-PROTOCOL, Section 5.4.1}}) whose `franking_signature_key` and `credential` are those of `hubSender`, and a `base_room_policy` component ({{MIMI-ROOM-POLICY, Section 5}}) that parses.
8. If that `base_room_policy` has `parent_dependant` set, its `parent_room` is a room this server hosts. {{MIMI-ROOM-POLICY, Section 5}} requires a parent-dependent room to be "always hosted on the same Hub as the parent room", and a server that does not host the parent cannot apply the rest of Section 5 to the room.
9. If that `base_room_policy` has `parent_dependant` set, `userUri` is a participant of the parent room. Section 5 keeps a parent-dependent room's participants within the parent's, and by condition 4 the creator is the room's only participant.

Clients SHOULD generate the 16 octets of the room ID and the 16 octets of the group ID independently from a cryptographically secure random source, and SHOULD use a fresh group ID for every MLS group of a room.

The GroupInfo SHOULD contain the `external_pub` extension ({{RFC9420, Section 12.4.3.2}}), so that the hub can answer `GroupInfo/fetch` ({{groupinfo-fetch}}). The room SHOULD carry a `roles_list` component ({{MIMI-ROOM-POLICY, Section 3}}) that gives the creator a role with `canAddParticipant`, since without one the hub can authorize no later add; the server MAY refuse a room without one. The group's `app_components` component ({{MLS-EXTENSIONS, Section 5}}) names the components that have state in the GroupContext, such as `franking_agent`, `participant_list`, `roles_list`, `base_room_policy`, and `room_metadata` when present. A room whose hub franks messages also lists `frank_aad`, in both `safe_aad` and `app_components` ({{MLS-EXTENSIONS, Section 5}}).

Room metadata is optional. A room that carries it requires every member's client to support the `room_metadata` component, since a component with state in the GroupContext is a required component ({{MLS-EXTENSIONS, Section 5}}). A client creating a room it will share with users of other providers can leave the metadata out, and `room_metadata` out of the room's `app_components`, or add both later with AppDataUpdates once it knows every member supports it. Leaving the metadata out has a cost: some clients do not show an invitation to a room that has no room metadata.

On success the server creates a Chat bound to the room, with `roomId` set to the room ID ({{chat-roomid}}), whatever kind of room it is. A parent-dependent room gets one like any other: a Chat bound to a room is that room's message container, so a room without one has no messages. A client tells such a Chat apart by its `parentRoomId` ({{chat-roomid}}) rather than by its absence.

Room/create returns:

`accountId` (Id):
: The account used.

`roomId` (String):
: The room ID.

`chatId` (Id):
: The id of the new Chat.

The owner's other clients join the room by external join ({{groupinfo-fetch}}), as any new client of a participant does.

## Room/update {#room-update}

Room/update submits a proposal or commit for a room ({{MIMI-PROTOCOL, Section 5.3}}). Adding a user, removing one, leaving, changing room metadata or policy, and the external commit of an external join all use it.

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The MIMI room ID. Unless `request` carries an external Commit, the `roomId` of a Chat in this account bound to its room.

`request` (String):
: An UpdateRequest ({{MIMI-PROTOCOL, Section 5.3}}).

`request` is the HandshakeBundle with no Protocol octet before it. The Welcome and the GroupInfo in it are the bare structures ({{RFC9420, Section 12.4.3}}), not MLSMessages, unlike `groupInfo` of `Room/create`. A client SHOULD send the ratchet tree as `full` ({{MLS-RATCHET-TREE-OPTIONS}}), unless `maxMlsObjectSize` forces another option. {{examples}} shows the octets of such a request.

An external Commit ({{MIMI-PROTOCOL, Section 3.5}}) joins the account's user to a room, whether or not the account is in it: this is how a client joins an open room or by a join link, rejoins after leaving, or adds another of its own clients. Its signature key need not be bound to the account ({{client}}), so the server checks the credential instead. When the hub accepts an external Commit whose UpdatePath leaf's signature key is not bound to the account, the server binds it and creates a Client record for it, as a KeyPackage/set create does. When `maxClients` signature keys are already bound to the account, the server rejects, with the method-level error `forbidden` and without submitting it, an external Commit whose signature key is not one of them. The server SHOULD reject, with `invalidArguments`, an external Commit whose signature key is bound to another account.

The server MUST reject the call with `invalidArguments` if `request` does not parse as an UpdateRequest, if it carries an external Commit whose UpdatePath leaf's credential identity is not `userUri`, or if it does not carry an external Commit and no Chat in the account is bound to `roomId`. Otherwise it processes the request as hub, or submits it to the hub, and returns `serverUnavailable` if the hub cannot be reached. When the hub accepts an external Commit from an account that has no Chat bound to the room, the server creates a Chat bound to it, or binds the account's unbound Chat with that `roomId` again.

The response has the following arguments, decoded from the hub's response:

`accountId` (Id):
: The account used.

`responseCode` (String):
: The `responseCode`, written as its name in {{MIMI-PROTOCOL, Section 5.3}} (`"success"`, `"wrongEpoch"`, `"notAllowed"`, or `"invalidProposal"`).

`errorDescription` (String):
: The hub's `errorDescription`, possibly empty. A hub that refuses a commit because it is not valid MLS, rather than by policy, MAY say so here.

`acceptedAt` (UTCDate):
: When `responseCode` is `"success"`: the hub's `accepted_timestamp`.

`messageIds` (Id\[], optional):
: The ids of the Messages the server stored in this account for the accepted message, so that the sending client can recognise them when it reads its Messages.

`currentEpoch` (UnsignedInt):
: When `responseCode` is `"wrongEpoch"`: the hub's current epoch.

`invalidProposals` (String\[]):
: When `responseCode` is `"invalidProposal"`: the ProposalRefs of the invalid proposals.

On success the server MUST store, in every Chat of a local account bound to the room, a Message ({{fanout-messages}}) whose body is the FanoutMessage with `timestamp` set to `accepted_timestamp`, `message` set to `proposalOrCommit`, and `moreProposals` copied from the request (or an empty `externalProposals` for a commit), unless {{fanout-messages}} forbids storing it. The hub does not forward the owner's own handshake to the owner's provider ({{MIMI-PROTOCOL, Section 5.5}}), so this is how the owner's other clients and the room's other local participants receive it. When the hub staples `externalProposals` to the owner's external commit, the owner's other clients and the room's other local participants do not receive them.

On `success` the client merges its own Commit and does not process the Messages named in `messageIds`. A client that lost the response recognises its Commit by comparing the stored MLSMessage octets with the ones it sent.

A client that did not receive the response MAY submit the same `request` again. The server SHOULD answer a `request` byte-identical to one the hub already accepted for the room with the earlier response, without submitting it again.

## Room/send {#room-send}

Room/send submits an application message to a room's hub ({{MIMI-PROTOCOL, Section 5.4}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account.

`message` (String):
: An MLSMessage ({{RFC9420, Section 6}}) of wire format PrivateMessage and content type application, whose plaintext is MIMI content {{MIMI-CONTENT}}.

`silent` (Boolean, optional, default `false`):
: When `true`, the server MAY skip push notifications ({{RFC8620, Section 7.2}}) for the Messages it stores for this message in local accounts. A client MAY set it for a message that recipients need but need not be woken for, such as a message status report ({{status}}) or an edit. The flag does not reach other providers: a SubmitMessageRequest and a FanoutMessage ({{MIMI-PROTOCOL}}) carry no such flag, so another provider notifies its users as it would for any message.

The server MUST reject the call with `invalidArguments` if `message` does not parse as such an MLSMessage or if no Chat in the account is bound to `roomId`. Otherwise it submits the message to the hub as a SubmitMessageRequest with `sendingUri` set to `userUri` ({{MIMI-PROTOCOL, Section 5.4}}), or processes it as hub, and returns `serverUnavailable` if the hub cannot be reached.

The response has the following arguments, decoded from the SubmitMessageResponse:

`accountId` (Id):
: The account used.

`responseCode` (String):
: The `statusCode`, written as its name in {{MIMI-PROTOCOL, Section 5.4}} (`"accepted"`, `"notAllowed"`, or `"epochTooOld"`).

`acceptedAt` (UTCDate):
: When `responseCode` is `"accepted"`: the hub's `accepted_timestamp`.

`frank` (String, optional):
: When `responseCode` is `"accepted"` and the hub franked the message: the hub's Frank ({{MIMI-PROTOCOL, Section 5.4}}). The hub franks a message only if its PrivateMessage carries a `frank_aad` SafeAAD item ({{MIMI-PROTOCOL, Section 5.4.1}}).

`messageIds` (Id\[], optional):
: The ids of the Messages the server stored in this account for the accepted message, so that the sending client can recognise them when it reads its Messages.

`currentEpoch` (UnsignedInt):
: When `responseCode` is `"epochTooOld"`: the hub's current epoch.

When the hub accepts the message, the server MUST store, in every Chat of a local account bound to the room, a Message ({{fanout-messages}}) whose body is the FanoutMessage with `timestamp` set to `accepted_timestamp`, `message` set to `message`, and `frank` copied from the response when present, unless {{fanout-messages}} forbids storing it. The hub does not forward the owner's own message to the owner's provider ({{MIMI-PROTOCOL, Section 5.5}}), so this is how the owner's clients and the room's other local participants receive it.

A client that did not receive the response MAY submit the same `message` again. The server SHOULD answer a `message` byte-identical to one the hub already accepted for the room with the earlier response, without submitting it again.

## Room/reportAbuse {#room-report-abuse}

Room/reportAbuse sends an abuse report ({{MIMI-PROTOCOL, Section 5.9}}) to a room's hub.

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account, bound or unbound ({{chat-roomid}}), so that a user removed from a room can still report what it received.

`report` (String):
: An AbuseReport ({{MIMI-PROTOCOL, Section 5.9}}).

The server MUST reject the call with `invalidArguments` if `report` does not parse as an AbuseReport, if no Chat in the account has `roomId`, or if the report's `reportingUser` is not `userUri`: a report sent through this method always names its reporter. Otherwise it sends the report to the room's hub, or processes it as hub, and returns `serverUnavailable` if the hub cannot be reached.

The response has the following arguments:

`accountId` (Id):
: The account used.

`responseCode` (String):
: `"accepted"` if the hub accepted the report; `"notAllowed"` if the hub refused it because the room's policy does not let the reporter report abuse (`canReportAbuse`, {{MIMI-ROOM-POLICY, Section 8.3}}); `"unverifiedMessage"` if the frank of a reported message did not verify; or `"refused"` for any other refusal.

`status` (UnsignedInt, optional):
: When `responseCode` is `"refused"` and the hub is another provider: the HTTP status of its answer ({{MIMI-PROTOCOL, Section 5.9}}).

A hub that reports a refusal reason the server cannot tell apart from others is answered as `"refused"`.

## Room/unbind {#room-unbind}

Room/unbind tells the server that the account has left a room's MLS group ({{chat-roomid}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account.

The server MUST reject the call with `invalidArguments` if no Chat in the account has that `roomId`. Otherwise it unbinds the Chat, if it is bound, and returns:

`accountId` (Id):
: The account used.

## GroupInfo/fetch {#groupinfo-fetch}

GroupInfo/fetch obtains the GroupInfo and ratchet tree of a room's MLS group for an external join ({{MIMI-PROTOCOL, Section 5.6}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The MIMI room ID of the room to join. The account need not have a Chat for it: the hub decides by its room policy whether to answer ({{MIMI-ROOM-POLICY}}).

`request` (String):
: A GroupInfoRequest signed by the client.

{{MIMI-PROTOCOL, Section 5.6}} makes `joiningCode` optional in GroupInfoRequestTBS but a plain vector in GroupInfoRequest. A client without a joining code signs a GroupInfoRequestTBS in which `joiningCode` is absent and sends a zero-length `joiningCode` in the GroupInfoRequest.

The server MUST reject the call with `invalidArguments` if `request` does not parse as a GroupInfoRequest, if `roomId` is not a MIMI room ID, or if the hub hosts no such room. Otherwise it answers as hub, or forwards the request to the hub, and returns `serverUnavailable` if the hub cannot be reached.

The response arguments are:

`accountId` (Id):
: The account used.

`status` (String):
: The GroupInfoResponse `status`, written as its name in {{MIMI-PROTOCOL, Section 5.6}} (`"success"`, `"notAuthorized"`, or `"noSuchRoom"`).

`response` (String):
: The whole GroupInfoResponse, unchanged.

The GroupInfo and ratchet tree are encrypted to the client's `groupInfoPublicKey`, so the server passes them through. The client SHOULD verify the response signature with `hub_sender`, and SHOULD verify that the decrypted GroupInfo's `external_senders` extension contains that `hub_sender`, before it joins. The client frames the external Commit's `authenticated_data` according to the GroupContext in the fetched GroupInfo, as {{MLS-EXTENSIONS, Section 4.9}} requires: when that GroupContext has a `safe_aad` component, `authenticated_data` is a SafeAAD, empty if the client adds no items.

A FanoutMessage carrying a Welcome ({{fanout-messages}}) whose RatchetTreeOption is `distributionService` ({{MLS-RATCHET-TREE-OPTIONS}}) does not carry the tree. The client gets the tree with GroupInfo/fetch, which returns the tree of the group's current epoch. A client that cannot get the tree for the Welcome's epoch, because a later commit has been accepted, MAY join by external join instead.

A local user added to the participant list without being added to the MLS group ({{MIMI-PROTOCOL, Section 5.6}}) has no Chat for the room, and so cannot use GroupInfo/fetch. This document does not define how such a user learns of the room.

# MLS Messages {#mls-messages}

In a Chat with a `roomId`, every Message has `bodyType` `"application/mimi-fanout"`. It is an encrypted body type for the purposes of {{JMAP-CHAT}}, including its rules on fields inside the ciphertext. Neither it nor `message/mls` is `application/mimi-content`, which {{MIMI-CONTENT, Section 8.1}} registers for unencrypted MIMI content.

Push events carry no Messages. A client that reconnects its event source, or starts, calls `Message/changes` from its saved Message state before it relies on events again. A server MAY set an event id on each `state` event; a client that has one sends it in `Last-Event-ID`.

## message/mls {#private-messages}

A client sends an application message as an MLSMessage ({{RFC9420, Section 6}}) of wire format PrivateMessage and content type application, the content that {{RFC9420, Section 17.10}} registers as `message/mls`, with `Room/send` ({{room-send}}). The server MUST reject a `Message/set` create in a Chat with a `roomId` with a `forbidden` SetError, whatever its `bodyType`.

MIMI edits, reactions, and retractions are MIMI content in new messages. A server acting as the account's MIMI provider MUST reject a `Message/set` update that changes `body`, `bodyType`, or `reactions` of a Message in a Chat with a `roomId` with an `invalidProperties` SetError naming the changed properties. A converting party that builds the room's messages itself instead maps such an update to MIMI content that replaces the original ({{edits}}).

## application/mimi-fanout {#fanout-messages}

A Message with `bodyType` `"application/mimi-fanout"` has as `body` one FanoutMessage ({{MIMI-PROTOCOL, Section 5.5}}). The FanoutMessage carries no Protocol octet; its last field is chosen by the MLSMessage's wire format and, for a PublicMessage, its content type: `frank` for a PrivateMessage, `ratchetTreeOption` for a Welcome, `moreProposals` for a proposal, and `externalProposals` for a commit. Only servers create such Messages: in a Chat without a `roomId`, the server MUST reject a `Message/set` create with this body type with an `invalidProperties` SetError naming `bodyType`; in a Chat with a `roomId`, the rule of {{private-messages}} applies.

The server stores each FanoutMessage it receives or originates for a room as one Message, in the order received, in every Chat of a local account bound to that room ({{chat-roomid}}). A FanoutMessage carrying a Welcome is stored only in the account owning a KeyPackageRef named in the Welcome; if that account has no Chat bound to the room, the server first binds the account's unbound Chat for the room again or, if it has none, creates a Chat bound to the room. The server stores a FanoutMessage that carries a commit removing the owner's clients like any other, so that those clients learn of their removal ({{chat-roomid}}).

A client joined by a Welcome takes the room ID from the Chat's `roomId`. If the group carries room metadata ({{MIMI-PROTOCOL, Section 7.6}}), the client MAY check that its `room_uri` equals the `roomId`.

A Chat that is created or bound again holds the Messages the server stores for its room from then on. A server MAY also place in it Messages it stored for the room earlier, for example those stored while the hub was still answering the request that bound it. Such Messages appear in `Message/changes` as created, even from a state given after they were stored. A client therefore cannot treat a Message state as "every Message up to some point"; it can only resume from a state the server gave it.

A room's hub is the provider named by the authority component of its room ID ({{MIMI-PROTOCOL}}). The server MUST reject a `/notify` request for a room, and MUST NOT store its FanoutMessages, unless the MIMI transport authenticated the sender as that provider. Without this check, another provider with participants in the room could inject Commits and Welcomes for a room it does not host.

For such a Message:

- `senderMsgId` is the ULID {{ULID}} whose 48-bit time component is the FanoutMessage `timestamp` and whose 80-bit random component is the first 10 octets of the SHA-256 hash of the FanoutMessage octets. The server MUST NOT store a FanoutMessage whose `senderMsgId` it has already stored in the Chat, so a repeated `/notify` yields one Message.
- The server MUST NOT store a FanoutMessage whose `message` is byte-identical to the `message` of a FanoutMessage already stored in the Chat. A hub may return the owner's own message, for example after failover ({{MIMI-PROTOCOL, Section 5.5}}).
- `sentAt` is the FanoutMessage `timestamp`.
- `senderId` is the id of the ChatContact ({{JMAP-CHAT}}) that represents the room's hub, whose `identity` and `login` are the hub's provider URI as authenticated by the MIMI transport ({{MIMI-PROTOCOL}}); the server creates that ChatContact, with an `id` it assigns, if it does not exist. This holds also for the Messages that `Room/send` and `Room/update` store ({{room-send}}, {{room-update}}), which are not owner-composed in the sense of {{JMAP-CHAT}}. The `blocked` field of that ChatContact applies as {{JMAP-CHAT}} defines it, so blocking the hub drops every FanoutMessage of the rooms it hosts. Clients authenticate the MLS sender as {{JMAP-CHAT}} requires for encrypted deployments.

The server assigns Message `id`s in the order in which it stores the Messages of a Chat, so that sorting a Chat's Messages by `id` gives that order. A client processes the handshake messages of a room in that order. A server MAY also return Messages in that order from `Message/get` and `Message/changes`.
{: #message-order}

A client that cannot calculate Message changes from its state MAY use `Message/query` on each Chat, sorted by `id`, with `anchor` set to the last Message it processed and `anchorOffset` 1, to fetch only the later Messages ({{message-order}}). This document adds `id` as a sort property of `Message/query` ({{JMAP-CHAT}}) for Chats with a `roomId`.

Storing a Message changes the account's Message state, and creating or binding a Chat again changes its Chat state ({{RFC8620, Section 5.1}}), so a client that watches both types in `StateChange` events learns of every FanoutMessage stored for it, its own included.

The server sends a `StateChange` for a stored Message only once `Message/changes` returns it.

# Examples {#examples}

Payloads are shortened with `...`.

A client publishes a KeyPackage:

~~~json
["KeyPackage/set", {
  "accountId": "u1",
  "create": {
    "k1": { "keyPackage": "AAEAAkBBxu1...", "lastResort": false }
  }
}, "c1"]
~~~

Response:

~~~json
["KeyPackage/set", {
  "accountId": "u1",
  "oldState": "s7",
  "newState": "s8",
  "created": {
    "k1": { "id": "kp42", "keyPackageRef": "IJ3qJ4dWvT..." }
  }
}, "c1"]
~~~

Alice's client fetches KeyPackages for Bob, who has two clients:

~~~json
["KeyMaterial/fetch", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "request": "AAEYbWltaTov..."
}, "c2"]
~~~

Response:

~~~json
["KeyMaterial/fetch", {
  "accountId": "u1",
  "userStatus": "partialSuccess",
  "userUri": "mimi://b.example/u/bob",
  "clients": [
    { "clientStatus": "success",
      "clientUri": "mimi://b.example/d/ClientB1",
      "keyPackage": "AAEAAkBBy2c..." },
    { "clientStatus": "keyMaterialExhausted",
      "clientUri": "mimi://b.example/d/ClientB2" }
  ],
  "response": "AAEAAhZtaW1p..."
}, "c2"]
~~~

Alice's client creates a room:

~~~json
["Room/create", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "groupInfo": "AAEABEAAEJr1...",
  "ratchetTree": "QAABAQ..."
}, "c3"]
~~~

Response:

~~~json
["Room/create", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "chatId": "01JA8Z3K6V2Q4R5S7T9W0XYZAB"
}, "c3"]
~~~

Alice's client commits Bob's addition and the hub accepts it:

~~~json
["Room/update", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "request": "AAEAARBeDKF9..."
}, "c4"]
~~~

Response:

~~~json
["Room/update", {
  "accountId": "u1",
  "responseCode": "success",
  "errorDescription": "",
  "acceptedAt": "2026-10-18T17:02:11.482Z"
}, "c4"]
~~~

The octets of such a `request`, before base64url encoding, follow ({{room-update}}). They are those of an UpdateRequest that the mimid hub accepted in its tests, for a Commit that adds one user with one client. Long fields are cut short with `...`. A vector's octets begin with its length prefix, and the length its note gives does not count the prefix.

~~~
00 01 00 01        MLSMessage of the Commit: version mls10 (1),
                   wire_format mls_public_message (1). No Protocol
                   octet comes before it.
10 5e 0c a1 7d 33 92 4b 08 a6 1f 70 d4 2e 95 c3 6b
                   group_id (16 octets)
00 00 00 00 00 00 00 01
                   epoch 1
01 00 00 00 00     sender: member (1), leaf_index 0
01 00              authenticated_data (1 octet): a SafeAAD with no
                   items
03                 content_type commit (3)
41 88 01 00 01 ... proposals (392 octets), both by value: an Add
                   carrying the new client's KeyPackage, and an
                   AppDataUpdate of the participant list
00                 path: absent
40 40 1e ab ...    signature (64 octets)
20 5b b0 6f ...    confirmation_tag (32 octets)
20 80 41 a3 ...    membership_tag (32 octets)
01                 welcome: present
00 01              Welcome, bare, with no MLSMessage header:
                   cipher_suite 1
40 76 20 c6 ...    secrets (118 octets): one EncryptedGroupSecrets
42 1d 89 5d ...    encrypted_group_info (541 octets)
01                 GroupInfoOption: representation full (1)
00 01 00 01 10 5e 0c ...
                   GroupInfo, bare (561 octets); its GroupContext
                   starts with version mls10 (1), cipher_suite
                   1, group_id
01                 RatchetTreeOption: representation full (1)
42 df 01 01 20 ... ratchet_tree (735 octets)
~~~

A new client of Alice fetches GroupInfo before an external join:

~~~json
["GroupInfo/fetch", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "request": "AAEAAQAgsJ4f..."
}, "c5"]
~~~

Response:

~~~json
["GroupInfo/fetch", {
  "accountId": "u1",
  "status": "success",
  "response": "AAEbbWltaTov..."
}, "c5"]
~~~

A client sends a message in a stale epoch:

~~~json
["Room/send", {
  "accountId": "u1",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a",
  "message": "AAEAAgAQmvVK..."
}, "c6"]
~~~

Response:

~~~json
["Room/send", {
  "accountId": "u1",
  "responseCode": "epochTooOld",
  "currentEpoch": 4
}, "c6"]
~~~

Bob's client processes a commit that removes Bob's last client from the room, and tells Bob's server:

~~~json
["Room/unbind", {
  "accountId": "u2",
  "roomId": "mimi://a.example/r/k5qwmzlqmfsgk3tbnvsxi2lf4a"
}, "c7"]
~~~

Alice asks Erin, a user of another provider, for consent to add her to rooms:

~~~json
["Consent/set", {
  "accountId": "u1",
  "create": {
    "c1": { "requesterUri": "mimi://a.example/u/alice",
            "targetUri": "mimi://e.example/u/erin" }
  }
}, "c5"]
~~~

The server sends a requestConsent to Erin's provider and answers:

~~~json
["Consent/set", {
  "accountId": "u1",
  "oldState": "q3",
  "newState": "q4",
  "created": {
    "c1": { "id": "cs7", "roomId": null, "requested": true,
            "decision": null, "updated": "2026-11-15T08:00:00Z" }
  }
}, "c5"]
~~~

When Erin's provider later sends an updateConsent with a grant, the record changes, and Alice's client sees it after a `StateChange` for the Consent type:

~~~json
["Consent/get", {
  "accountId": "u1",
  "state": "q5",
  "list": [
    { "id": "cs7",
      "requesterUri": "mimi://a.example/u/alice",
      "targetUri": "mimi://e.example/u/erin",
      "roomId": null, "requested": false, "decision": "grant",
      "updated": "2026-11-15T08:03:12Z" }
  ],
  "notFound": []
}, "c6"]
~~~

# Security Considerations {#security}

A converting party that relays end-to-end encrypted MIMI messages ({{RFC9420}}) into JMAP Chat must decrypt them to convert their content, and so holds plaintext. It SHOULD join each MLS group as a member visible to the other members, so that users can see that a gateway holds plaintext, rather than obtaining keys by any path that does not appear in the group membership.

A derived `senderMsgId` keeps 80 bits of the MIMI message ID it comes from, and references resolve on those 80 bits alone ({{message-ids}}). A converting party MUST NOT accept a MIMI message whose octets 1 through 10 equal the `senderMsgId` random component or an alias of a different message already stored in the same Chat; it rejects the message rather than overwriting or merging the stored one, so that every reference resolves to at most one message.

Alias records ({{edits}}) let a MIMI reply, reaction, or edit that names a later version reach the original message. A converting party SHOULD verify that the sender of an edit is the sender of the original message before recording its alias, so that a third party cannot attach an alias to, and thereby edit or retract, someone else's message.

A MIMI provider is not a member of any MLS group and never holds message keys. It does see room metadata in GroupInfo and in handshake messages sent as PublicMessage, the participant list, message sizes, and timing, as any MIMI hub or follower does ({{MIMI-PROTOCOL}}).

As hub, a MIMI provider's `hubSender` is an external sender of every room it hosts and can propose removals. Clients that create a room through `Room/create` grant it that role.

Every request that consumes a KeyPackage brings the owner's clients closer to their last-resort KeyPackages, whose reuse weakens the protection that single-use KeyPackages give ({{RFC9420, Section 16.8}}). Servers SHOULD rate-limit key material requests per requester. Clients SHOULD fetch key material only when they are ready to commit, and reuse a fetched KeyPackage when they retry the same Commit.

`Room/unbind` is not checked against the MLS group. A client that calls it while other clients of the account remain in the group cuts those clients off from the room: the server stores no further FanoutMessage for them, so they no longer receive the room's messages; they can rejoin by external join if the room's policy lets them.

Anyone on a peer provider can send the owner a consent request, so requests are themselves a way to reach a user who has not agreed to be contacted. Servers SHOULD rate-limit requests per requester and per provider, and MAY drop requests without creating a record. A Consent record is visible only to its owner, but sending a revoke tells the requester's provider that the owner refused; a server can withhold revokes ({{consent-set}}).

A Welcome is stored in the owner's account, so all of the owner's clients receive it, and each can open only the part encrypted to its own KeyPackage. This exposes the Welcome's existence and size to the owner's other clients only.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:mimi`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Data Types Registration

IANA is requested to register the following entry in the "JMAP Data Types" registry, including the "Supports History" field defined by {{JMAP-OBJ-HISTORY}}:

Type Name:
: KeyPackage

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Supports History:
: No

Capability:
: `urn:ietf:params:jmap:chat:mimi`

Reference:
: This document

IANA is requested to register the following entry in the "JMAP Data Types" registry:

Type Name:
: Consent

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Supports History:
: No

Capability:
: `urn:ietf:params:jmap:chat:mimi`

Reference:
: This document

IANA is requested to register the following entry in the "JMAP Data Types" registry:

Type Name:
: Client

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Supports History:
: No

Capability:
: `urn:ietf:params:jmap:chat:mimi`

Reference:
: This document

## JMAP Error Code Registration

IANA is requested to add this document to the Reference of the `rateLimited` entry that {{JMAP-CHAT}} registers in the "JMAP Error Codes" registry ({{RFC8620, Section 9.5}}), whose Description names this document. This document returns it from its own methods ({{capability}}), with the optional `retryAfter` property.

## Media Type Registration

The media type `message/mls` is registered by {{RFC9420, Section 17.10}}; this document refers to it in {{mls-messages}} and does not register it.

IANA is requested to register the following media type in the "Media Types" registry ({{RFC6838}}):

Type name:
: application

Subtype name:
: mimi-fanout

Required parameters:
: N/A

Optional parameters:
: N/A

Encoding considerations:
: binary. In JMAP Chat a body of this type is carried as unpadded base64url ({{RFC4648, Section 5}}).

Security considerations:
: See {{security}} of this document.

Interoperability considerations:
: The content is one FanoutMessage ({{MIMI-PROTOCOL, Section 5.5}}) in the TLS presentation language encoding: one MLSMessage with the hub's timestamp and fanout metadata. A bare MLSMessage is `message/mls`, not this type.

Published specification:
: This document.

Applications that use this media type:
: JMAP Chat servers and clients that implement the `urn:ietf:params:jmap:chat:mimi` capability.

Fragment identifier considerations:
: N/A

Additional information:
: Deprecated alias names for this type: N/A; Magic number(s): N/A; File extension(s): N/A; Macintosh file type code(s): N/A

Person and email address to contact for further information:
: Mark Atwood <mark@reviewcommit.com>

Intended usage:
: LIMITED USE

Restrictions on usage:
: Used only as the `bodyType` of a JMAP Chat Message ({{fanout-messages}}).

Author:
: Mark Atwood

Change controller:
: IETF

--- back

# Acknowledgements

The author thanks the MIMI working group, and the authors of {{MIMI-CONTENT}} and {{MIMI-PROTOCOL}} in particular, for the content format and protocol this mapping targets.
