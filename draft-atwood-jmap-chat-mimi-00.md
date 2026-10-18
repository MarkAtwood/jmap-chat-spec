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
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC4648:
  RFC8174:
  RFC8620:
  RFC9420:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  MIMI-CONTENT:
    title: "More Instant Messaging Interoperability (MIMI) message content"
    author:
      fullname: R. Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-content-09
    date: 2026
  MIMI-PROTOCOL:
    title: "More Instant Messaging Interoperability (MIMI) using HTTPS and MLS"
    author:
      - fullname: R. L. Barnes
      - fullname: M. Hodgson
      - fullname: K. Kohbrok
      - fullname: R. Mahy
      - fullname: T. Ralston
      - fullname: R. Robert
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-protocol-06
    date: 2026
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

informative:
  RFC6838:
  MIMI-MESSAGE-STATUS:
    title: "A Message Status format for the More Instant Messaging Interoperability (MIMI) content format"
    author:
      fullname: R. Mahy
    seriesinfo:
      Internet-Draft: draft-mahy-mimi-message-status-01
    date: 2026
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/

--- abstract

This document defines how a gateway converts messages between the IETF More Instant Messaging Interoperability (MIMI) content format ({{MIMI-CONTENT}}) and JMAP Chat ({{JMAP-CHAT}}). It specifies how MIMI message identifiers, edits, topics, media reactions, and message statuses map onto JMAP Chat Messages, so that independent gateways derive the same JMAP Chat identifiers for the same MIMI messages without coordination. It also defines a JMAP capability with which a JMAP Chat server acts as an MLS client's MIMI provider ({{MIMI-PROTOCOL}}): it publishes KeyPackages, fetches key material, creates and updates rooms, fetches GroupInfo, and carries end-to-end encrypted MIMI messages without decrypting them.

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

# Message Identifiers {#message-ids}

A converting party MUST derive each message's `senderMsgId` as the ULID {{ULID}} whose 48-bit time component is the hub's acceptance time of the message in milliseconds (the FanoutMessage `timestamp` ({{MIMI-PROTOCOL, Section 5.5}}), or, for the party's own message, the `accepted_timestamp` of the SubmitMessageResponse ({{MIMI-PROTOCOL, Section 5.4}})) and whose 80-bit random component is octets 1 through 10 of its MIMI message ID, counting from 0: the ten octets that follow the hash algorithm octet. Every party that receives a MIMI message therefore derives the same `senderMsgId` without coordination.

A MIMI reference to another message (`replaces` or `inReplyTo` {{MIMI-CONTENT}}, the latter also naming the message a reaction applies to) carries only a MIMI message ID. A converting party resolves it to the Message in the same Chat whose `senderMsgId` random component, or one of whose aliases ({{edits}}), equals octets 1 through 10 of that ID. A reference to a message the party has not stored does not resolve.

# Edits {#edits}

A MIMI edit, whose `replaces` field always names the first version of the message {{MIMI-CONTENT}}, maps to an edit of the message to which `replaces` resolves ({{message-ids}}). The converting party records octets 1 through 10 of each edit's own MIMI message ID as an alias of that message, so a MIMI reply or reaction that names a later version refers to the message as a whole.

# Topics {#topics}

MIMI's `topicId` {{MIMI-CONTENT}}, a byte string, maps to the Message `topicId` field ({{JMAP-CHAT}}) as its unpadded base64url encoding ({{RFC4648, Section 5}}).

# Reactions {#reactions}

A MIMI media reaction (a reaction that is not an emoji) is carried in JMAP Chat as a reply containing the media, as {{MIMI-CONTENT}} permits; JMAP Chat Reactions ({{JMAP-CHAT}}) remain emoji and custom emoji.

# Message Status {#status}

MIMI message statuses `expired`, `hidden`, and `error` {{MIMI-MESSAGE-STATUS}} are not read acknowledgements, so a converting party does not set `readAt` or a ReadDisposition for them.

# The urn:ietf:params:jmap:chat:mimi Capability {#capability}

Servers supporting the client interface of this document advertise the `urn:ietf:params:jmap:chat:mimi` capability in the JMAP Session object. This capability is meaningful only when `urn:ietf:params:jmap:chat` is also present.

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

`cipherSuites` (UnsignedInt[]):
: The MLS cipher suites ({{RFC9420, Section 17.1}}) this server accepts in KeyPackages and new rooms. MUST NOT be empty.

`maxKeyPackages` (UnsignedInt):
: The maximum number of KeyPackage records, last-resort KeyPackages included, the server stores for this account.

A server that advertises this capability MUST include `"message/mls"` and `"application/mimi-fanout"` in `supportedBodyTypes` ({{JMAP-CHAT}}).

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
: When `true`, this is the client's last-resort KeyPackage ({{RFC9420, Section 16.8}}). Default: `false`.

## KeyPackage/get

Standard JMAP `/get` ({{RFC8620, Section 5.1}}).

## KeyPackage/set

Standard JMAP `/set` ({{RFC8620, Section 5.3}}), except that `update` is not supported: the server MUST reject every update with a SetError of type `forbidden`.

On create, the server MUST reject the record with `invalidProperties` naming `keyPackage` if `keyPackage` does not parse as a KeyPackage, if its signature does not verify, if its cipher suite is not in `cipherSuites`, if its leaf node lifetime has expired, if its credential is not a Basic credential ({{RFC9420, Section 5.3}}), or if the credential identity is not a client URI ({{MIMI-PROTOCOL}}) whose authority is this server's domain. The first KeyPackage the server accepts for a client URI binds that client URI to the account. The server MUST reject, with `invalidProperties` naming `keyPackage`, a KeyPackage whose client URI is bound to another account.

The server MUST reject, with `invalidProperties` naming `keyPackage`, a KeyPackage whose KeyPackageRef equals that of a KeyPackage it stores or has handed out, and MUST reject a create that would exceed `maxKeyPackages` with `overQuota`.

Each client has at most one last-resort KeyPackage. Creating a last-resort KeyPackage for a client destroys that client's previous last-resort KeyPackage.

## Handing Out KeyPackages {#keypackage-claim}

When this server answers a key material request for the owner ({{MIMI-PROTOCOL, Section 5.2}}), it returns at most one KeyPackage for each of the owner's clients. For each client it returns a KeyPackage compatible with the request that is not last-resort and destroys that record in the same transaction, so that no KeyPackage is handed out twice. When the client has no such KeyPackage, the server returns the client's compatible last-resort KeyPackage, if any, and keeps it. Returning a last-resort KeyPackage more than once departs from {{MIMI-PROTOCOL, Section 5.2}}, which forbids repeating key material across responses, and follows {{RFC9420, Section 16.8}}.

The server MUST keep the KeyPackageRef of every KeyPackage it hands out, with the account it belongs to, until a Welcome naming it has been delivered or its leaf node lifetime has passed. A destroyed record appears in a new state for the KeyPackage type, so a client that receives the `StateChange` can replenish its KeyPackages.

# KeyMaterial/fetch {#keymaterial-fetch}

KeyMaterial/fetch obtains KeyPackages for another user's clients so that the owner can add that user to a room ({{MIMI-PROTOCOL, Section 5.2}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account bound to its room ({{chat-roomid}}) to which the target user is to be added.

`request` (String):
: A KeyMaterialRequest signed by the client.

The server MUST reject the call with `invalidArguments` if `request` does not parse as a KeyMaterialRequest, if its `requestingUser` is not `userUri`, if its `roomId` is not `roomId`, or if no Chat in the account is bound to that `roomId`. Otherwise the server obtains a KeyMaterialResponse. When this server is the room's hub, it answers from its own KeyPackage store ({{keypackage-claim}}) if the target user is local and asks the target provider otherwise. When this server is not the room's hub, it MUST send the request to the hub, even if the target user is local, because a hub that did not see the request cannot forward the Welcome to the target provider ({{MIMI-PROTOCOL, Section 5.2}}). If the provider it asks cannot be reached, the server returns `serverUnavailable`.

The response has the following arguments, decoded from the KeyMaterialResponse:

`accountId` (Id):
: The account used.

`userStatus` (String):
: The `userStatus` code, written as its name in {{MIMI-PROTOCOL, Section 5.2}} (for example `"partialSuccess"`).

`userUri` (String):
: The target user URI.

`clients` (Object[]):
: One object per ClientKeyMaterial, with the fields `clientStatus` (String, the code name), `clientUri` (String), `keyPackage` (String, present when `clientStatus` is `"success"`), and `clientCapabilities` (String, the MLS Capabilities, present when the provider included it).

# Rooms {#rooms}

## Chat.roomId {#chat-roomid}

When this capability is present, the Chat object ({{JMAP-CHAT}}) has one more field:

`roomId` (String, immutable, server-set, optional):
: The MIMI room ID of the room this Chat is bound to. Absent for Chats not bound to a MIMI room.

A Chat with a `roomId` has kind `"group"`. Its participant list, roles, and metadata are MLS group state, changed only through `Room/update` ({{room-update}}). The server MUST reject, with `forbidden`, a `Chat/set` update on such a Chat that sets `name`, `description`, or `avatarBlobId` or that uses the `addMembers`, `removeMembers`, or `updateMemberRoles` patch key ({{JMAP-CHAT}}), and clients MUST take the room's name and participants from the MLS group state rather than from these fields. The server sets `name` to the `room_name` of the room metadata ({{MIMI-PROTOCOL, Section 7.6}}) when it knows it and to the empty string otherwise.

The server does not mirror the participant list into ChatMember records: `members` of a Chat with a `roomId` is the empty list, whether `Room/create` ({{room-create}}) or a Welcome ({{fanout-messages}}) created it. The {{JMAP-CHAT}} requirement that the sender of a `Message/set` create in a group Chat have a ChatMember record does not apply to such a Chat. Instead, the server MUST reject the create with `forbidden` unless the Chat is bound to its room.

A Chat is bound to its room from its creation until the server stores a commit that removes from the MLS group every client whose client URI is bound to the account ({{keypackage}}). The server stores that commit ({{fanout-messages}}) and then unbinds the Chat. An unbound Chat keeps its `roomId` and its Messages. The server stores no further FanoutMessage in it, and `KeyMaterial/fetch`, `Room/update`, and `GroupInfo/fetch` treat it as if no Chat in the account had that `roomId`. A later Welcome for the account binds the Chat again ({{fanout-messages}}). A server learns of a removal from commits it can read; a commit sent as a SemiPrivateMessage ({{MIMI-PROTOCOL, Section 5.3}}) is readable only by the hub. A client that processes a commit removing the last of its account's clients, which the server could not read, destroys the Chat with `Chat/set` so that the server stops storing FanoutMessages for the account.

## Room/create {#room-create}

Room/create creates a room with this server as hub ({{MIMI-PROTOCOL, Section 3.1}}) and a Chat bound to it. The client first creates the MLS group with no member other than its own client.

The request arguments are:

`accountId` (Id):
: The account to use.

`groupInfo` (String):
: An MLSMessage ({{RFC9420, Section 6}}) of wire format `mls_group_info` carrying the group's GroupInfo, which MUST contain the `ratchet_tree` extension.

The server MUST reject the call with `invalidArguments` unless all of the following hold:

1. The GroupInfo signature verifies, and its cipher suite is in `cipherSuites`.
2. The ratchet tree contains exactly one leaf, whose credential identity is a client URI bound to the account ({{keypackage}}).
3. The `external_senders` extension contains `hubSender`.
4. The participant list ({{MIMI-PROTOCOL, Section 7.5}}) contains exactly one participant, `userUri`.
5. The `room_uri` of the room metadata ({{MIMI-PROTOCOL, Section 7.6}}) is `roomIdPrefix` followed by the unpadded lowercase base32 encoding ({{RFC4648, Section 6}}) of 16 octets, and no room this server hosts has that room ID.
6. The group ID is 16 octets and no room this server hosts uses it.

Clients MUST generate the 16 octets of the room ID and the 16 octets of the group ID independently from a cryptographically secure random source, and MUST use a fresh group ID for every MLS group of a room.

On success the server creates a Chat bound to the room, with `roomId` set to the room ID ({{chat-roomid}}), and returns:

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
: The `roomId` of a Chat in this account.

`request` (String):
: An UpdateRequest.

The server MUST reject the call with `invalidArguments` if `request` does not parse as an UpdateRequest or if no Chat in the account is bound to `roomId`. Otherwise it processes the request as hub, or submits it to the hub, and returns `serverUnavailable` if the hub cannot be reached.

The response has the following arguments, decoded from the hub's response:

`accountId` (Id):
: The account used.

`responseCode` (String):
: The `responseCode`, written as its name in {{MIMI-PROTOCOL, Section 5.3}} (`"success"`, `"wrongEpoch"`, `"notAllowed"`, or `"invalidProposal"`).

`errorDescription` (String):
: The hub's `errorDescription`, possibly empty.

`acceptedAt` (UTCDate):
: When `responseCode` is `"success"`: the hub's `accepted_timestamp`.

`currentEpoch` (UnsignedInt):
: When `responseCode` is `"wrongEpoch"`: the hub's current epoch.

`invalidProposals` (String[]):
: When `responseCode` is `"invalidProposal"`: the ProposalRefs of the invalid proposals.

On success the server MUST store, in the owner's Chat for the room, a Message ({{fanout-messages}}) whose body is the FanoutMessage with `timestamp` set to `accepted_timestamp`, `message` set to `proposalOrCommit`, and `moreProposals` copied from the request (or an empty `externalProposals` for a commit), so that the owner's other clients receive the handshake.

## GroupInfo/fetch {#groupinfo-fetch}

GroupInfo/fetch obtains the GroupInfo and ratchet tree of a room's MLS group for an external join ({{MIMI-PROTOCOL, Section 5.6}}).

The request arguments are:

`accountId` (Id):
: The account to use.

`roomId` (String):
: The `roomId` of a Chat in this account.

`request` (String):
: A GroupInfoRequest signed by the client.

The server MUST reject the call with `invalidArguments` if `request` does not parse as a GroupInfoRequest or if no Chat in the account is bound to `roomId`. Otherwise it answers as hub, or forwards the request to the hub, and returns `serverUnavailable` if the hub cannot be reached.

The response arguments are:

`accountId` (Id):
: The account used.

`status` (String):
: The GroupInfoResponse `status`, written as its name in {{MIMI-PROTOCOL, Section 5.6}} (`"success"`, `"notAuthorized"`, or `"noSuchRoom"`).

`response` (String):
: The whole GroupInfoResponse, unchanged.

The GroupInfo and ratchet tree are encrypted to the client's `groupInfoPublicKey`, so the server passes them through. The client MUST verify the response signature with `hub_sender`, and MUST verify that the decrypted GroupInfo's `external_senders` extension contains that `hub_sender`, before it joins.

A local user added to the participant list without being added to the MLS group ({{MIMI-PROTOCOL, Section 5.6}}) has no Chat for the room, and so cannot use GroupInfo/fetch. This document does not define how such a user learns of the room.

# MLS Messages {#mls-messages}

In a Chat with a `roomId`, two body types carry MLS messages. Both are encrypted body types for the purposes of {{JMAP-CHAT}}, including its rules on fields inside the ciphertext. Neither is `application/mimi-content`, which {{MIMI-CONTENT, Section 8.1}} registers for unencrypted MIMI content.

## message/mls {#private-messages}

A Message with `bodyType` `"message/mls"` ({{RFC9420, Section 17.10}}) in a Chat with a `roomId` has as `body` an MLSMessage ({{RFC9420, Section 6}}) of wire format PrivateMessage and content type application, whose plaintext is MIMI content {{MIMI-CONTENT}}. In such a Chat, `Message/set` create MUST use this body type; the server MUST reject any other `bodyType` with `invalidArguments`.

On create, the server submits the PrivateMessage to the hub as a SubmitMessageRequest with `sendingUri` set to `userUri` ({{MIMI-PROTOCOL, Section 5.4}}) and waits for the response before answering. When the hub accepts the message, the server stores the Message with `sentAt` set to `accepted_timestamp`, replacing the client's `sentAt`. Otherwise it rejects the create with a SetError of type `notAllowed`, or of type `epochTooOld` with a `currentEpoch` (UnsignedInt) property holding the hub's current epoch.

MIMI edits, reactions, and retractions are MIMI content in new messages. The server MUST reject a `Message/set` update that changes `body`, `bodyType`, or `reactions` of a Message in a Chat with a `roomId` with `invalidArguments`.

## application/mimi-fanout {#fanout-messages}

A Message with `bodyType` `"application/mimi-fanout"` has as `body` one FanoutMessage ({{MIMI-PROTOCOL, Section 5.5}}). Only servers create such Messages; the server MUST reject a `Message/set` create with this body type with `invalidArguments`.

The server stores each FanoutMessage it receives or originates for a room as one Message, in the order received, in every Chat of a local account bound to that room ({{chat-roomid}}). A FanoutMessage carrying a Welcome is stored only in the account owning a KeyPackageRef named in the Welcome; if that account has no Chat bound to the room, the server first binds the account's unbound Chat for the room again or, if it has none, creates a Chat bound to the room. The server stores a FanoutMessage that carries a commit removing the owner's clients like any other, so that those clients learn of their removal, and then unbinds the Chat if the commit removed the last of them ({{chat-roomid}}).

For such a Message:

- `senderMsgId` is the ULID {{ULID}} whose 48-bit time component is the FanoutMessage `timestamp` and whose 80-bit random component is the first 10 octets of the SHA-256 hash of the FanoutMessage octets. The server MUST NOT store a FanoutMessage whose `senderMsgId` it has already stored in the Chat, so a repeated `/notify` yields one Message.
- The server MUST NOT store a FanoutMessage whose `message` is byte-identical to the body of a `message/mls` Message already stored in the Chat; that is the hub returning the owner's own message.
- `sentAt` is the FanoutMessage `timestamp`.
- `senderId` is the id of the ChatContact ({{JMAP-CHAT}}) that represents the room's hub, whose `id` and `login` are the hub's provider URI as authenticated by the MIMI transport ({{MIMI-PROTOCOL}}); the server creates that ChatContact if it does not exist. This holds also for the Message that `Room/update` stores ({{room-update}}), which is not owner-composed in the sense of {{JMAP-CHAT}}. The `blocked` field of that ChatContact applies as {{JMAP-CHAT}} defines it, so blocking the hub drops every FanoutMessage of the rooms it hosts. Clients authenticate the MLS sender as {{JMAP-CHAT}} requires for encrypted deployments.

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
    "k1": { "id": "kp42", "keyPackageRef": "3qJ4dWvT..." }
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
  ]
}, "c2"]
~~~

Alice's client creates a room:

~~~json
["Room/create", {
  "accountId": "u1",
  "groupInfo": "AAEABEAAEJr1..."
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
  "request": "AAEAAQAQmvVK..."
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
["Message/set", {
  "accountId": "u1",
  "create": {
    "m1": {
      "chatId": "01JA8Z3K6V2Q4R5S7T9W0XYZAB",
      "bodyType": "message/mls",
      "body": "AAEAAgAQmvVK...",
      "sentAt": "2026-10-18T17:05:00Z"
    }
  }
}, "c6"]
~~~

Response:

~~~json
["Message/set", {
  "accountId": "u1",
  "oldState": "m90",
  "newState": "m90",
  "notCreated": {
    "m1": { "type": "epochTooOld", "currentEpoch": 4 }
  }
}, "c6"]
~~~

# Security Considerations {#security}

A converting party that relays end-to-end encrypted MIMI messages ({{RFC9420}}) into JMAP Chat must decrypt them to convert their content, and so holds plaintext. It SHOULD join each MLS group as a member visible to the other members, so that users can see that a gateway holds plaintext, rather than obtaining keys by any path that does not appear in the group membership.

A derived `senderMsgId` keeps 80 bits of the MIMI message ID it comes from, and references resolve on those 80 bits alone ({{message-ids}}). A converting party MUST NOT accept a MIMI message whose octets 1 through 10 equal the `senderMsgId` random component or an alias of a different message already stored in the same Chat; it rejects the message rather than overwriting or merging the stored one, so that every reference resolves to at most one message.

Alias records ({{edits}}) let a MIMI reply, reaction, or edit that names a later version reach the original message. A converting party MUST verify that the sender of an edit is the sender of the original message before recording its alias, so that a third party cannot attach an alias to, and thereby edit or retract, someone else's message.

A MIMI provider is not a member of any MLS group and never holds message keys. It does see room metadata in GroupInfo and in handshake messages sent as PublicMessage, the participant list, message sizes, and timing, as any MIMI hub or follower does ({{MIMI-PROTOCOL}}).

As hub, a MIMI provider's `hubSender` is an external sender of every room it hosts and can propose removals. Clients that create a room through `Room/create` grant it that role.

Every request that consumes a KeyPackage brings the owner's clients closer to their last-resort KeyPackages, whose reuse weakens the protection that single-use KeyPackages give ({{RFC9420, Section 16.8}}). Servers SHOULD rate-limit key material requests per requester.

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

IANA is requested to register the following entry in the "JMAP Data Types" registry:

Type Name:
: KeyPackage

Can Reference Blobs:
: No

Can Use for State Change:
: Yes

Capability:
: `urn:ietf:params:jmap:chat:mimi`

Reference:
: This document

## JMAP Error Types Registration

IANA is requested to register the following entry in the "JMAP Error Types" registry ({{RFC8620, Section 9.5}}):

Error Type:
: `epochTooOld`

Description:
: The hub rejected a message because it was sent in an MLS epoch older than the room's current epoch. Returned as a SetError on `Message/set` create in a Chat with a `roomId`, with a `currentEpoch` property holding the hub's current epoch ({{private-messages}}).

Capability:
: `urn:ietf:params:jmap:chat:mimi`

Reference:
: This document

## Media Type Registration

The media type `message/mls` is registered by {{RFC9420, Section 17.10}}; this document uses it as {{mls-messages}} describes and does not register it.

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
