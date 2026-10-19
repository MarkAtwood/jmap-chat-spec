---
title: JMAP Chat Federation
abbrev: JMAP Chat Federation
docname: draft-atwood-jmap-chat-federation-00
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
  RFC8615:
  RFC9421:
  RFC9530:
  RFC7517:
  RFC7638:
  RFC8037:
  RFC9110:
  RFC9525:
  RFC4033:
  RFC6698:
  RFC7671:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026

informative:
  RFC9420:
  RFC8552:
  DANCE-CLIENT-AUTH:
    title: "TLS Client Authentication via DANE TLSA records"
    author:
      - fullname: S. Huque
      - fullname: V. Dukhovni
    seriesinfo:
      Internet-Draft: draft-ietf-dance-client-auth-14
    date: 2026
  JMAP-CHAT-DID:
    title: JMAP Chat DID
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-did-00
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
  MIMI-ROOM-POLICY:
    title: "Room Policy for the More Instant Messaging Interoperability (MIMI) Protocol"
    author:
      - fullname: R. Mahy
    seriesinfo:
      Internet-Draft: draft-ietf-mimi-room-policy-04
    date: 2026
  JMAP-CHAT-MIMI:
    title: JMAP Chat MIMI Mapping
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-mimi-00
    date: 2026
  JMAP-CHAT-TASKS:
    title: JMAP Chat Tasks
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-tasks-00
    date: 2026

--- abstract

This document defines the server-to-server federation protocol for JMAP Chat ({{JMAP-CHAT}}). It specifies peer discovery via `/.well-known/jmap`, the peer authentication model, the `role` field in the JMAP Chat account capability, ChatContact and Session object extensions required for federation, and the Peer/* methods used to exchange messages and events between mailbox servers. Five methods are REQUIRED: `Peer/deliver`, `Peer/receipt`, `Peer/typing`, `Peer/retract`, and `Peer/groupUpdate`. Three additional methods for federated presence are OPTIONAL: `Peer/subscribePresence`, `Peer/unsubscribePresence`, and `Peer/presence`. This document is a companion to {{JMAP-CHAT}} and is intended to be read alongside it.

--- middle

# Introduction

{{JMAP-CHAT}} defines the JMAP Chat capability, which supports a mailbox-per-user deployment topology in which each participant operates their own JMAP server. In this topology, mailboxes must exchange messages and events directly with one another over the open Internet. This document formalizes the mechanisms by which that exchange occurs.

Specifically, this document defines:

- How a peer server discovers another server's JMAP session URL ({{peer-discovery}}).
- How a peer server authenticates to another server and obtains peer-role access ({{peer-authentication}}).
- Extensions to the ChatContact data type and the JMAP Session object that support federation ({{chat-contact-extensions}} and {{session-extensions}}).
- The `role` field in the JMAP Chat account-level capability ({{account-capability}}).
- The five mandatory server-to-server Peer/* methods ({{peer-methods}}).
- Three optional Peer/* methods for federated presence: `Peer/subscribePresence`, `Peer/unsubscribePresence`, and `Peer/presence` ({{peer-subscribepresence}}, {{peer-unsubscribepresence}}, {{peer-presence}}).
- Outbox and delivery semantics, including retry behavior ({{outbox}}).
- Security considerations specific to the federation protocol ({{security}}).

## Deployment Context

In the mailbox-per-user model, each participant runs their own JMAP server (a "mailbox") that stores only their own messages. There is no central server, no central message store, and no central operator. Mailboxes exchange messages directly with each other over a secure transport using the Peer/* server-to-server methods defined in this document.

This document does not redefine the relay topology described in {{JMAP-CHAT}}. In a relay deployment, the relay itself implements the Peer/* methods internally; this federation protocol applies only when distinct, independently operated mailbox servers communicate with one another.

## Relationship to JMAP Chat

This document extracts and formalizes the server-to-server elements of {{JMAP-CHAT}}. Implementations MUST support {{JMAP-CHAT}} as a prerequisite to implementing this document. Terminology, data types, and method semantics from {{JMAP-CHAT}} are used throughout without re-definition.

## Relationship to MIMI

The IETF MIMI working group {{MIMI-PROTOCOL}} is developing a provider-to-provider federation protocol for messaging interoperability. MIMI adopts a hub-and-spoke architecture in which each room is owned by one provider (the "hub") and other providers connect as "followers." The hub is authoritative for room state and distributes messages to follower providers.

This document takes a different approach: each participant operates an independent mailbox server with no central room owner. Message delivery is point-to-point between mailboxes; room state is derived from the union of messages each mailbox has received. This design prioritizes decentralization and avoids any single provider holding authority over a conversation, at the cost of additional complexity in race condition handling (see {{direct-chat-race}}). MIMI defines room roles and capabilities in {{MIMI-ROOM-POLICY}}, while JMAP Chat uses Space roles as defined in {{JMAP-CHAT}}.

The two protocols are not interoperable at the federation layer and serve different deployment contexts. They are not in conflict; a deployment could implement both, using this protocol for JMAP-native mailbox-to-mailbox federation and MIMI for cross-provider interoperability with non-JMAP messaging platforms. {{JMAP-CHAT-MIMI}} defines how a gateway between the two converts MIMI messages to JMAP Chat Messages.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} and {{JMAP-CHAT}} is used throughout.

The following terms are specific to this document:

Local server:
: The JMAP Chat mailbox server receiving a federation connection from a remote peer.

Remote server / peer server:
: A JMAP Chat mailbox server initiating a federation connection to the local server.

Peer domain:
: The DNS domain name that identifies a mailbox server for federation: the host at which the server publishes its `/.well-known/jmap` Session object ({{peer-discovery}}). A peer domain is compared in lowercase A-label form, without a trailing dot or port.

Peer identity:
: The `ownerUserId` advertised in the Session object published at a peer domain whose request signature the local server has verified ({{peer-authentication}}).

# Peer Discovery {#peer-discovery}

Peer servers discover each other's JMAP session URL via the Well-Known URI mechanism defined in {{RFC8615}}.

## Discovery Procedure

To discover a peer's JMAP session URL, a server issues:

~~~
GET /.well-known/jmap HTTP/1.1
Host: <peer-hostname>
~~~

The response is the peer's JMAP Session object as defined in {{RFC8620}} Section 2. The Session object MUST include an `ownerUserId` field identifying the mailbox owner (see {{session-extensions}}).

Servers SHOULD cache discovered session data, including the session URL and `ownerUserId`, to avoid redundant discovery on each request. Servers SHOULD re-run discovery when delivery fails, as the peer's session URL or capabilities may have changed.

## Direct Address Hints

The Session object MAY include an `ownerDirectAddress` field (see {{session-extensions}}). This field is an optional hint for out-of-band direct delivery and is deployment-defined in format and semantics. Servers MUST fall back to the standard JMAP API path on any failure when using a direct address. Servers MUST NOT use `ownerDirectAddress` as a blob fetch target (see {{ssrf}}).

# Peer Authentication Model {#peer-authentication}

## Overview

A peer is a domain. A server authenticates to a peer by signing each HTTP request with a key that its peer domain publishes; the receiving server verifies the signature and so learns which peer domain sent the request. Every server MUST support the baseline mechanism ({{request-signing}} through {{key-publication}}). Publishing the signing keys in DNSSEC-signed DNS ({{dnssec-keys}}) is RECOMMENDED where DNSSEC is available, and mutual TLS with a DANE-verified client certificate ({{mutual-tls}}) is OPTIONAL. Neither replaces the request signature.

Before a remote server can invoke any Peer/* method on a local server, it MUST obtain a JMAP session at the local server by fetching the local server's Session resource with a signed request. The local server MUST provide a JMAP account in the resulting session that carries the `urn:ietf:params:jmap:chat` capability with `role: "peer"` (see {{account-capability}}).

The local server MUST verify that the `ownerUserId` advertised at the authenticated peer domain is bound to that domain ({{identity-binding}}) before granting peer-role access. If the binding does not hold, the local server MUST reject the request.

## Request Signing {#request-signing}

A server MUST sign every HTTP request it sends to a peer to obtain a peer-role session or to invoke a Peer/* method, using HTTP Message Signatures {{RFC9421}}. Each such request carries exactly one signature whose `tag` parameter is `"jmap-chat-peer"`; the signature label is chosen by the signer. The signature:

- MUST use the `ed25519` algorithm ({{RFC9421}} Section 3.3.6), and MUST include the `alg` parameter with that value.
- MUST include the `keyid` parameter, whose value is the signer's peer domain, the character `#`, and the `kid` of the signing key ({{key-publication}}); for example, `alice.example#kPrK_qmxVWaYVA9wwBF6Iuo3vVzz7TxHCTwXBygrS4k`.
- MUST include the `created` and `expires` parameters. `expires` MUST NOT be more than 300 seconds after `created`.
- MUST include the `nonce` parameter, carrying at least 128 bits of output from a cryptographically secure random number generator, base64url-encoded. A signer MUST NOT reuse a nonce.
- MUST cover the `@method` and `@target-uri` derived components ({{RFC9421}} Section 2.2). Covering `@target-uri` binds the signature to the receiving server, so a signature made for one peer cannot be replayed to another.
- MUST, when the request has content, cover the `content-digest` field {{RFC9530}}, which the signer MUST include and which MUST contain a `sha-256` digest of the content.

A receiving server MUST reject, without processing any method call, a request that lacks such a signature or whose signature does not verify. Verification succeeds only if all of the following hold:

1. The request carries exactly one signature tagged `"jmap-chat-peer"`, its `alg` is `ed25519`, and it covers the components listed above.
2. The `keyid` parses as a peer domain and a `kid`, and the `kid` names a usable key for that peer domain ({{key-publication}}, {{dnssec-keys}}).
3. The signature verifies under that key ({{RFC9421}} Section 3.2).
4. `created` is no more than 60 seconds later than the receiver's current time, `expires` is no more than 300 seconds after `created`, and the receiver's current time is no more than 60 seconds after `expires`.
5. The receiver has not accepted a request carrying the same peer domain and `nonce` before. The receiver MUST remember each accepted (peer domain, `nonce`) pair at least until 60 seconds after that signature's `expires`.
6. For a request with content, the `content-digest` field contains a `sha-256` digest and that digest matches the content.

The receiving server rejects a request that fails any of these checks with HTTP status 403 (Forbidden). The peer domain named in a verified `keyid` is the request's authenticated peer domain.

## Key Publication {#key-publication}

A server publishes its signing public keys in the `peerSigningKeys` field of its Session object ({{session-extensions}}). Each key is a JSON Web Key {{RFC7517}} with `kty` `"OKP"` and `crv` `"Ed25519"` {{RFC8037}}, and its `kid` member MUST be the base64url-encoded SHA-256 JWK Thumbprint {{RFC7638}} of the key. A JWK in `peerSigningKeys` MUST NOT contain private key members.

To find the keys of a peer domain, the receiving server fetches `https://<peer-domain>/.well-known/jmap` ({{peer-discovery}}) over HTTPS, validating the server certificate against the peer domain as {{RFC9110}} Section 4.3.4 and {{RFC9525}} describe. The receiving server MAY follow redirects, provided each one is to an `https` URI whose certificate it validates the same way; the fetch is subject to {{ssrf}}. A key is usable only if its `kty` and `crv` are as above and the receiving server, recomputing the thumbprint, obtains the key's `kid`. The receiving server MUST ignore any other entry.

This key fetch, like every discovery fetch of `/.well-known/jmap` ({{peer-discovery}}), is an unsigned GET: the fetching server MUST NOT sign it. A server MUST answer an unsigned GET of its `/.well-known/jmap` with a Session object that includes `ownerUserId` and `peerSigningKeys`, and MUST NOT require any authentication for that response; in it, `accounts` and `primaryAccounts` MAY be empty objects and `username` MAY be the empty string. Only the request that obtains a peer-role session and Peer/* method calls are signed ({{request-signing}}). Verifying a signature therefore never requires a signed request back to the signer, so two servers that verify each other's requests cannot wait on each other.

The receiving server MAY cache the keys of a peer domain for the HTTP freshness lifetime of the response, but for no longer than one hour. When a request names a `kid` that is not among the usable keys of its peer domain (the cached keys that are usable under the precedence rule of {{dnssec-keys}}), the receiving server MUST fetch the keys again, bypassing its cache, before rejecting the request; if it uses the DNSSEC binding ({{dnssec-keys}}), it also queries the `_jmap-peer` name again, bypassing any DNS answer it has cached. It MUST NOT refetch the keys of one peer domain, or repeat that query, more often than once every 60 seconds; a request that arrives within that interval and names a `kid` that is not among the usable keys is rejected.

A server rotates its signing key by publishing the new key alongside the old one. It SHOULD publish a new key at least one hour before first signing with it, and SHOULD keep a retired key published for at least 10 minutes after last signing with it, so that signatures still within their validity window verify. A server whose private key is compromised SHOULD remove the key at once (see {{key-compromise}}).

## DNSSEC Key Binding {#dnssec-keys}

Where its zone is signed with DNSSEC {{RFC4033}}, a server SHOULD also publish each signing key's `kid` in DNS, as a TXT record at the name `_jmap-peer.<peer-domain>` whose value is `v=jmappeer1; kid=<kid>`, one record per key. Because the `kid` is the key's thumbprint, the record commits to the key itself.

A receiving server that has a DNSSEC-validating resolver SHOULD query this name when it fetches a peer domain's keys. The DNSSEC binding takes precedence over the HTTPS-published keys:

- If the answer is secure ({{RFC4033}} Section 5) and contains at least one record of this form, a key from `peerSigningKeys` is usable only if its recomputed thumbprint equals the `kid` of one of those records.
- If the answer is a secure denial of existence, is insecure, or contains no record of this form, the HTTPS-published keys are used alone.
- If the validation result is indeterminate ({{RFC4033}} Section 5), or the lookup fails (for example, SERVFAIL or a timeout), the HTTPS-published keys are used alone, as for an insecure answer.
- If the answer is bogus, the receiving server MUST NOT fall back to the HTTPS-published keys and MUST reject the request.

A receiving server that uses this binding MUST perform DNSSEC validation itself, or obtain validation results from a resolver over a trusted channel, such as a resolver on the same host or one reached over a channel that authenticates the resolver and protects the integrity of its answers. A receiving server that can do neither MUST treat every zone as unsigned and use the HTTPS-published keys alone.

A receiving server MUST NOT cache the DNS answer for longer than its TTL.

## Mutual TLS with DANE {#mutual-tls}

A server MAY additionally authenticate the TLS connection it uses to send peer requests by presenting a client certificate, which MAY be self-signed. The certificate is verified against a DNSSEC-signed TLSA record {{RFC6698}} {{RFC7671}} at `_jmap-peer.<peer-domain>`, using the client identity procedure of {{DANCE-CLIENT-AUTH}}, a work in progress. A self-signed certificate requires certificate usage DANE-EE(3).

A receiving server that verifies such a certificate MUST check that the TLS-authenticated domain equals the authenticated peer domain of every request on that connection, and MUST reject a request where it does not. A receiving server that does not verify client certificates, or a connection without one, relies on the request signature alone.

## Identity Binding {#identity-binding}

Peer authentication establishes a peer domain, while Peer/* arguments name users by identity string ({{identity-mapping}}). The local server binds the two as follows:

1. The peer identity of an authenticated request is the `ownerUserId` in the Session object published at its authenticated peer domain, obtained by the fetch that supplied the signing keys ({{key-publication}}).
2. If the peer identity has a host, that host MUST equal the authenticated peer domain. The host of an identity is: for a URI with an authority component, its host; for a `did:web` DID, the domain name it encodes; for a string of the form `local@domain` without a URI scheme, the part after the last `@`; for a bare DNS name, the name itself. Hosts are compared as peer domains are.
3. If the peer identity has no host (for example, a `did:key` DID), the peer domain does not vouch for it. The local server MUST NOT accept such an identity on the strength of peer authentication alone. A DID-form identity is accepted only after the DID-bound verification of {{JMAP-CHAT-DID}} succeeds; any other host-less identity MUST be rejected.

If the binding does not hold, the local server MUST reject the request. This binding ensures that:

1. The ChatContact record created or updated for a peer on the local server has, as its `identity`, the same string that the peer itself advertises.
2. The `senderUserId` argument in Peer/* methods can be validated against the peer identity of the authenticated domain.
3. A peer domain cannot claim an identity whose host is another domain.

The peer identity is the `identity` of the peer's ChatContact on the local server (see ChatContact in {{JMAP-CHAT}}) and is the identity against which all Peer/* method arguments are validated. The ChatContact's `id` is assigned by the local server and is unrelated to this string.

# Account-Level Capability {#account-capability}

The value of `accountCapabilities["urn:ietf:params:jmap:chat"]` is a JSON object. This document defines the following field within that object, which is relevant to federation:

`role` (String):
: Either `"owner"` or `"peer"`. An account with `role: "owner"` has full access to all JMAP Chat methods. An account with `role: "peer"` is granted access only to the Peer/* methods defined in this document. Callers authenticated as a peer and operating against an account where `role` is `"peer"` MUST receive `forbidden` for any method not listed in {{peer-methods}}.

The local server MUST include this field in the account capability for any account it provisions for peer use.

# ChatContact Type Extensions for Federation {#chat-contact-extensions}

The ChatContact data type defined in {{JMAP-CHAT}} includes the following fields that are specific to federation. They are defined here as the normative reference for their federation semantics.

`serverUrl` (String):
: Base HTTPS URL of this contact's mailbox. Used for outbound delivery and for probing `/.well-known/jmap`. Servers MUST treat this value as the base URL for discovery and delivery to this contact's mailbox. Servers populate this field at contact creation time and SHOULD update it when a more recent value is discovered via `/.well-known/jmap`.

`directAddress` (String, optional):
: A deployment-specific address at which this contact's client may be reachable directly, without routing through their mailbox. The format and semantics are deployment-defined (examples: a Tailscale node name, a WebRTC signaling URI, an IP:port tuple). This field is a hint only: senders MAY attempt delivery to this address when both parties are online and the deployment supports it, but MUST fall back to the standard mailbox path on any failure. This field has no effect on message storage or multi-device sync. Servers populate this field from the `ownerDirectAddress` advertised in the contact's `/.well-known/jmap` Session object (see {{session-extensions}}). This value is peer-supplied and MUST be treated as untrusted (see {{direct-address-security}}).

# Session Object Extensions for Federation {#session-extensions}

Servers implementing this specification MUST include the following fields in their JMAP Session object (advertised at `/.well-known/jmap`):

`ownerUserId` (String):
: The identity string of the mailbox owner. This value equals the `identity` of the owner's ChatContact on any peer server that has recorded this mailbox as a contact; the `id` of that ChatContact is assigned by the peer server. Servers MUST advertise this field. Peer servers use this value as the authoritative identity for the mailbox owner when performing the identity binding check described in {{peer-authentication}}. On the advertising server itself, the owner's local id is the `accountId` of the owner's account ({{JMAP-CHAT}}); in Peer/* arguments that `accountId` maps to `ownerUserId` and back ({{identity-mapping}}).

`peerSigningKeys` (JWK[]):
: The public keys with which this server signs peer requests ({{key-publication}}). Each entry is a JSON Web Key {{RFC7517}} with `kty` `"OKP"`, `crv` `"Ed25519"` {{RFC8037}}, `x`, and `kid` equal to the key's base64url-encoded SHA-256 JWK Thumbprint {{RFC7638}}. Servers MUST advertise this field with at least one key. For example:

~~~ json
"peerSigningKeys": [
  {
    "kty": "OKP",
    "crv": "Ed25519",
    "x": "11qYAYKxCrfVS_7TyWQHOg7hcvPapiMlrwIaaPcHURo",
    "kid": "kPrK_qmxVWaYVA9wwBF6Iuo3vVzz7TxHCTwXBygrS4k"
  }
]
~~~

Servers MAY include the following additional field:

`ownerDirectAddress` (String, optional):
: A deployment-specific address at which the owner's client may be reachable directly. Peers that probe `/.well-known/jmap` SHOULD store this value as the `directAddress` on the corresponding ChatContact record (see {{chat-contact-extensions}}). The format and semantics are deployment-defined. This value is advisory only; receiving servers MUST NOT treat it as authoritative or use it as a blob fetch target.

# Server-to-Server Methods {#peer-methods}

The following methods are used between mailbox servers only. An authenticated peer server accesses these methods using the JMAP account in which `role: "peer"` is set in the `urn:ietf:params:jmap:chat` accountCapabilities (see {{account-capability}}). Callers without the `"peer"` role MUST receive `forbidden` for all methods in this section.

All Peer/* methods carry a caller-identity field (`senderUserId`, `readerUserId`, `subscriberId`, or `identity` depending on the method). Despite the varying names, all serve the same role: the peer identity of the calling server. Servers MUST validate all such fields identically against the peer identity.

## User References in Peer Methods {#identity-mapping}

A ChatContact `id` identifies the ChatContact only on the server that assigned it ({{JMAP-CHAT}}); another server holds a different `id` for the same user. Peer/* arguments therefore never carry a ChatContact `id`. Every Peer/* argument or payload field that names a user carries that user's identity string: the `identity` of the user's ChatContact ({{JMAP-CHAT}}), or, for a server's own mailbox owner, that server's `ownerUserId` ({{session-extensions}}). These fields are `senderUserId`, `readerUserId`, `subscriberId`, `identity`, the `id` and `invitedBy` of each ChatMember in `members` and `addedMembers`, each entry of `removedMemberIds`, the `id` of each `updatedRoles` entry, and the `id` of each Mention in `mentions`. The same rule applies to every user-reference field that {{JMAP-CHAT}} or a companion specification defines for data carried in Peer/* payloads, including fields inside MessageAction metadata (for example, `assigneeIds` in the task metadata defined by {{JMAP-CHAT-TASKS}}). The field names are unchanged: each field still names a user, and the identity string is the only user identifier that both servers share.

The sending server replaces each local ChatContact `id` in these fields with that ChatContact's `identity`, and each occurrence of its owner's `accountId`, which is the owner's local id ({{JMAP-CHAT}}), with its `ownerUserId`.

The receiving server compares identity strings octet for octet and resolves each one to a local user:

1. An identity equal to the receiving server's own `ownerUserId` refers to the mailbox owner, whose local id is the owner's `accountId`.
2. Otherwise, an identity equal to the `identity` of a ChatContact refers to that ChatContact.
3. Otherwise, the receiving server MUST create a placeholder ChatContact with that `identity` and a server-assigned `id`, and the identity refers to the placeholder. The placeholder's other properties are learned through later interactions.

The receiving server stores the resolved local reference wherever {{JMAP-CHAT}} or a companion specification stores a user reference, for example in Chat members, `Message.senderId`, Mention `id` values, and `deliveryReceipts` keys. A server that does not recognise a field passes it through unmapped. Neither server maps, validates, or rewrites the `userId` of a `"mention"` span in a rich `body`: spans are display data with no server meaning ({{JMAP-CHAT}}), and servers never modify `body`. Only the `mentions` array is mapped and authoritative. A receiving server creates placeholders only as part of a request it accepts; a rejected request creates none.

## Peer/deliver {#peer-deliver}

Delivers a new message, an edit, or a reaction update from a remote mailbox. Exactly one of `message`, `edit`, or `reactionUpdate` MUST be present in a given request.

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`chatId` (String):
: The Chat ID. For direct chats, if the receiving server already has a direct Chat whose `contactId` is the id of the sender's ChatContact, it MUST verify that `chatId` matches the stored chatId. For group and channel chats, the receiver MUST verify this value matches the chatId of a known chat of which the sender is a current member.

`senderUserId` (String):
: The sender's identity string ({{identity-mapping}}). MUST match the peer identity of the request's authenticated peer domain ({{identity-binding}}). The receiver MUST compare these and MUST reject the request if they differ.

`chatKind` (String):
: `"direct"`, `"group"`, or `"channel"`. Informs which chatId verification procedure applies.

`message` (Object, optional):
: A new message to deliver. Fields:

  - `senderMsgId` (String) — Sender-assigned ULID. Functions as an idempotency key; if this value is already known for the given chat, the receiver MAY silently discard the duplicate.
  - `body` (String) — Message content. Validated per step 8 of {{new-message-validation}}.
  - `bodyType` (String) — MIME type of `body`. Validated against `supportedBodyTypes`.
  - `sentAt` (UTCDate) — Sender's claimed composition time. Stored as-is; MUST NOT be used for ordering.
  - `attachments` (Object[]) — Each entry carries Attachment fields (as defined in {{JMAP-CHAT}}) plus `fetchUrl` (String): the URL from which the receiver fetches the blob.
  - `mentions` (Mention[], optional) — Structured per-user @mention annotations. Each Mention `id` carries an identity string ({{identity-mapping}}).
  - `broadcastMentions` (BroadcastMention[], optional) — Structured broadcast-scope mention annotations (`@everyone`, `@here`, `@admins`). The receiving server is authoritative for computing the local recipient set; the sending server's set is informational only. See {{broadcast-mention-resolution}}.
  - `actions` (MessageAction[], optional) — Out-of-band action invitations. Servers MUST store and forward without inspection, except that user-reference fields the server recognises are mapped per {{identity-mapping}}.
  - `replyTo` (String, optional) — The `senderMsgId` of the message being replied to, as assigned by the sending server. The receiver resolves this to a local message `id` via the `senderMsgId` index. See {{outbound-preparation}} for how the sending server constructs this value.
  - `threadRootId` (String, optional) — The `senderMsgId` of the thread root message, as assigned by the sending server. The receiver resolves this similarly. See {{outbound-preparation}}. If the receiver cannot resolve a `replyTo` or `threadRootId` senderMsgId to a local message id (because the referenced message was never received or was hard-deleted), the receiver MUST store the message with `replyTo` and/or `threadRootId` set to null. The message MUST NOT be rejected on this basis.
  - `topicId` (String, optional): Topic identifier, carried unchanged. Validated per `topicId` in {{JMAP-CHAT}}; reject with `invalidArguments` if invalid.
  - `subject` (String, optional): Topic name, carried unchanged. Validated per `subject` in {{JMAP-CHAT}}; reject with `invalidArguments` if invalid.
  - `lastSeenSenderMsgId` (String, optional): causal hint. The `senderMsgId` of the latest message in this chat that the sending server had stored when the message was composed. Informational only. See {{message-ordering}}.
  - `senderExpiresAt` (UTCDate, optional) — Hard-deletion deadline. Servers MUST reject a value that is already in the past at delivery time with `invalidArguments`.
  - `burnOnRead` (Boolean, optional) — When `true`, the receiver MUST permanently hard-delete the message immediately after setting `readAt`. When the recipient's effective `receiptSharing` preference is `false`, no `Peer/receipt` notification will reach the sender's server after the message is read — but the hard-delete on the receiving mailbox still occurs. See `burnOnRead` in {{JMAP-CHAT}}.
  - `expiresAfterReadSeconds` (UnsignedInt, optional): When present, the receiver MUST permanently hard-delete the message this many seconds after setting `readAt`. If `burnOnRead` is also `true`, `burnOnRead` takes precedence. See `expiresAfterReadSeconds` in {{JMAP-CHAT}}.

`edit` (Object, optional):
: An edit to an existing message. Fields:

  - `senderMsgId` (String) — Identifies the message to edit via the `senderMsgId` index.
  - `body` (String) — New body content. Validated against `maxBodyBytes`.
  - `bodyType` (String) — MIME type of the new body. Validated against `supportedBodyTypes`.
  - `editedAt` (UTCDate) — Claimed edit time. Stored as-is.
  - `mentions` (Mention[], optional) — Structured per-user @mention annotations for the new body. Each Mention `id` carries an identity string ({{identity-mapping}}).
  - `broadcastMentions` (BroadcastMention[], optional) — Structured broadcast-scope mention annotations for the new body. Receiver-side resolution rules are identical to the new-message case (see {{broadcast-mention-resolution}}).

  The receiver MUST verify the sender is the original sender of the identified message before applying the edit.

`reactionUpdate` (Object, optional):
: A reaction change on an existing message. Fields:

  - `senderMsgId` (String) — Identifies the target message via the `senderMsgId` index.
  - `senderReactionId` (String) — Sender-assigned ULID for this reaction. For `"add"`, the receiver stores the reaction in the `reactions` map using `senderReactionId` as the map key. For `"remove"`, identifies which reaction to delete.
  - `emoji` (String) — Non-empty string identifying the reaction.
  - `customEmojiId` (String, optional) — Space-scoped custom emoji id, if applicable.
  - `action` (String) — `"add"` or `"remove"`.
  - `sentAt` (UTCDate) — Time of the reaction event.

### Broadcast Mention Resolution {#broadcast-mention-resolution}

The `broadcastMentions` field on `message` and `edit` payloads carries the wire form defined by the BroadcastMention type in {{JMAP-CHAT}}: a list of `{scope}` objects. The `scope` value identifies a member set (`"everyone"`, `"here"`, or `"admins"`) whose composition is resolved by the receiving server at delivery time against its own local view of membership, presence, and administrative permissions.

The receiving server's delivery-time set is authoritative for that server's owners. The sending server MAY have computed its own send-time set for sender-side UX; that set is informational only. The Peer/deliver wire payload does not carry a resolved id list for broadcast mentions, and a receiving server MUST NOT use any out-of-band "send-time set" hint from the sender for authorization, push elevation, or notification routing. Each receiving server evaluates its own predicate.

Two servers federating the same Chat will commonly produce different recipient sets for `"here"` and `"admins"` on the same message: each server sees only its own local owners, and the deployment-defined predicates may differ between deployments. This is expected; the wire contract does not require predicate alignment between peers.

Send-side authorization (the `"mention_broadcast"` permission gate defined in {{JMAP-CHAT}}) is enforced at the sending server. A receiving server generally does not have authority over the sending deployment's permission graph, and in that case it MUST NOT re-check the sender's permission state. A receiving server that holds an authoritative or signed copy of the Space's permission state (for example, because it hosts the Space, or because the Space state is signed by the Space's authority) MAY re-check the sender's `"mention_broadcast"` permission against that state, and MAY reject the message with `forbidden` if the sender lacks it. Apart from such a re-check, the receiving server's enforcement responsibility is limited to wire-format validation (scope enum) and to its own targeted owners' opt-out preferences (which are deployment-defined and NOT exposed via federation).

### Outbound Message Preparation {#outbound-preparation}

Before populating the `message` fields of an outbound `Peer/deliver` call, the sending server MUST translate any `replyTo` or `threadRootId` values from locally-stored receiver-assigned `id` values to the corresponding `senderMsgId` values, since the peer only knows messages by the `senderMsgId` it originally assigned.

**`replyTo` translation:** Look up the `senderMsgId` for the locally-stored message identified by `replyTo`. If that message was composed by the owner (`senderId` is the owner's `accountId`), its `senderMsgId` equals its `id` and no translation is needed. If the message was received from a peer, use the stored `senderMsgId`. If no `senderMsgId` can be found for the referenced message, the `replyTo` field MUST be omitted from the outbound payload; sending a locally-meaningful id that the peer cannot resolve would produce silently broken threading.

**`threadRootId` translation:** Apply the same procedure. If the thread root message cannot be resolved to a `senderMsgId`, `threadRootId` MUST be omitted from the outbound payload.

### New Message Validation {#new-message-validation}

Before storing a new message, the server MUST perform the following steps in order:

1. Verify the request signature and the identity binding ({{peer-authentication}}).
2. Confirm `senderUserId` matches the peer identity. Reject immediately if they differ. Resolve `senderUserId` to the sender's local ChatContact per {{identity-mapping}}; the remaining steps refer to that ChatContact as the sender.
3. For direct chats: if a chat with this sender already exists, confirm `chatId` matches its stored id; otherwise create a new Chat record with this chatId and set `contactId` to the sender's ChatContact `id` (see {{direct-chat-race}} for the simultaneous-initiation race condition). For group and channel chats: confirm `chatId` matches a known chat and the sender is a current member of that chat. If the group chat's `closedAt` is set, or the channel's Space has `closedAt` set ({{JMAP-CHAT}}), reject with `forbidden`.
4. For channel chats: verify that no active SpaceBan exists for the sender in the Space containing the identified channel. If a SpaceBan exists, silently drop the message and return success.
5. For channel chats: verify that the sender holds the `"send"` permission in the identified channel per the Space permission resolution defined in {{JMAP-CHAT}}. If the sender lacks `"send"`, reject with `forbidden`.
6. For channel chats where the channel's `slowModeSeconds` is non-zero: verify that the sender is not rate-limited by the slow-mode policy. If the sender's last accepted message in this channel was received within `slowModeSeconds` seconds and the sender does not hold `"manage_channels"` permission, reject with `rateLimited`.
7. Confirm the sender is not blocked by the owner. If `blocked` is `true` on the sender's ChatContact record, silently drop the message and return success (to avoid disclosing block status to the sender).
8. Validate `body` byte length against `maxBodyBytes`; reject with `invalidArguments` if exceeded. For plaintext `bodyType` values, also validate UTF-8 encoding. For encrypted `bodyType` values (e.g., `"application/mls-ciphertext"`), `body` is opaque ciphertext; servers MUST NOT parse or transform it beyond byte-length checking.
9. Validate `bodyType` against the server's `supportedBodyTypes`; reject with `invalidArguments` if not present. Servers implementing {{JMAP-CHAT}} that support the `application/jmap-chat-rich` body type SHOULD include it in `supportedBodyTypes`.
10. Validate each attachment `filename` (MUST NOT contain `/`, `\`, or null bytes), `contentType` (MUST be a syntactically valid MIME type string), and `size`.
11. Fetch each attachment blob from its `fetchUrl`; verify the byte count against `size` and the content against `sha256`. Reject with `invalidArguments` if either check fails. See {{ssrf}} for restrictions on `fetchUrl` targets.
12. Take mentions only from the `mentions` array and broadcast mentions only from the `broadcastMentions` array, for every `bodyType` (per {{JMAP-CHAT}}). The receiving server MUST NOT parse `body`, including a rich body's spans, to find mentions for push-notification mention elevation or delivery-time recipient-set computation.
13. Validate each `broadcastMentions` entry. Reject with `invalidArguments` if any entry's `scope` is not one of `"everyone"`, `"here"`, or `"admins"`. The receiving server MUST NOT use the sending server's send-time recipient set (if any) for authorization or push-elevation decisions; the recipient set is computed by the receiving server at delivery time as defined in {{broadcast-mention-resolution}}.
14. If `senderExpiresAt` is present, confirm it is strictly in the future; reject with `invalidArguments` if it is in the past or equal to the current time. Schedule hard deletion at that time. Whether or not `senderExpiresAt` is present, if `burnOnRead` is `true`, register a trigger to hard-delete the message when `readAt` is set; otherwise, if `expiresAfterReadSeconds` is present, register a trigger to hard-delete the message that many seconds after `readAt` is set.

When `bodyType` indicates an encrypted type (e.g., `"application/mls-ciphertext"`), the server MUST, at step 10, reject with `invalidArguments` any message in which `attachments`, `mentions`, `broadcastMentions`, `actions`, `replyTo`, `threadRootId`, `topicId`, `subject`, or `lastSeenSenderMsgId` is present and non-empty, and MUST skip steps 11 through 13. That information, if any, is carried inside the ciphertext, as specified in the End-to-End Encrypted Deployments section of {{JMAP-CHAT}}.

Failure at any step MUST result in rejection with no data stored and no side effects.

### Edit and Reaction Validation

For `edit` payloads: perform steps 1 through 7 of {{new-message-validation}}, then validate `body` and `bodyType` per steps 8 and 9, apply step 12 to `mentions` and `broadcastMentions`, and validate `broadcastMentions` per step 13 (for an encrypted `bodyType`, apply the empty-field check and skip steps 12 and 13 as described above). Verify that the identified message exists in the given chat and that its `senderId` is the sender's ChatContact `id`. Reject if the sender does not match.

For `reactionUpdate` payloads: perform steps 1 through 7. Validate `emoji` as a non-empty string. For `"add"`, verify the target message exists. For `"remove"`, verify the target reaction exists and was originally added by the sender. Reject if any check fails.

### Response

`accountId` (String):
: The account ID from the request.

`receivedMsgId` (String):
: For a new message: the receiver's assigned ULID for the stored message. For an edit or reaction update: the local `id` of the affected message.

`receivedAt` (UTCDate):
: The time at which the receiving server stored the message or applied the edit or reaction.

`canonicalChatId` (String, optional):
: Present only when the receiving server has resolved a direct-chat simultaneous-initiation race (see {{direct-chat-race}}). Contains the canonical chatId that both servers SHOULD adopt. Absent in all other cases.

## Peer/receipt

Notifies the sending mailbox that a message has been stored and/or read by a recipient.

Method name: `Peer/receipt`

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`senderMsgId` (String):
: The sender-assigned ULID identifying the message for which this receipt is being sent.

`deliveredAt` (UTCDate, optional):
: Time the message was stored by the recipient's mailbox. SHOULD be present on first delivery acknowledgement.

`deviceDeliveredAt` (UTCDate, optional):
: Time the message notification was received by the recipient's device or application process, as distinct from mailbox storage (`deliveredAt`) and human reading (`readAt`). MAY be sent when the recipient's platform provides a delivery callback (e.g., OS push delivery receipt). Servers MUST store this value in `Message.deliveryReceipts` if provided.

`readAt` (UTCDate, optional):
: Time the message was read by the recipient. SHOULD be present when the recipient's owner has read the message.

`readDisposition` (String, optional):
: The ReadDisposition value indicating why the message was acknowledged. SHOULD be present when `readAt` is present. If `readAt` is present and `readDisposition` is absent, the receiving server MUST store `"displayed"`. See ReadDisposition in {{JMAP-CHAT}}. Unrecognized values MUST be stored as-is.

`readerUserId` (String):
: The identity string of the acknowledging user ({{identity-mapping}}). For group chats, the receiver updates the per-recipient delivery receipt in `deliveryReceipts` under that user's local ChatContact `id`. The `readerUserId` MUST match the peer identity of the calling server; the receiver MUST reject the request with `forbidden` if they differ.

### Sender Behavior

Before invoking `Peer/receipt` on a remote peer's server, the sending server MUST determine the effective receipt-sharing preference for the reading account (the recipient of the original message, whose read event is being reported — not the original message sender): if `Chat.receiptSharing` is present for the relevant Chat, that value is used; otherwise the account-level `PresenceStatus.receiptSharing` applies (both defined in {{JMAP-CHAT}}). If the effective preference is `false`, the server MUST NOT invoke `Peer/receipt` and MUST silently suppress the outbound call. The remote peer is not informed of the suppression.

### Response

`accountId` (String):
: The account ID from the request.

### Receiver Behavior

When the receiving server processes an inbound `Peer/receipt` call, the
server MUST determine the effective receipt-sharing preference for the
relevant Chat using the same rule as the Sender Behavior (defined above).
The receiving server (which hosts the original message sender's account)
checks the *original sender's* effective `receiptSharing` preference for
the relevant Chat. This is semantically distinct from the Sender Behavior
check (which checks the *reader's* preference). The combined effect: read
receipts are exchanged only when both parties opt in.

If the effective preference is `false`, the server MUST NOT update
`Message.deliveryReceipts` with the supplied `readAt` or `readDisposition`
values, and MUST NOT deliver any resulting state-change event to the
owner's connected clients. The server MUST NOT record `deviceDeliveredAt`
when the effective preference is `false`; like `readAt` and
`readDisposition`, it is a privacy-sensitive value reflecting user activity.

The server MAY still record `deliveredAt` if present, as delivery
acknowledgement is not affected by the receipt-sharing preference.

The calling server is not informed of this suppression; `Peer/receipt`
MUST return a normal success response regardless.

## Peer/typing

Notifies a remote mailbox that the owner of the calling server is or is not currently typing in a shared chat.

Method name: `Peer/typing`

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`chatId` (String):
: The Chat ID of the conversation in which typing is occurring.

`senderUserId` (String):
: The identity string of the typing user ({{identity-mapping}}). MUST match the peer identity; the receiver MUST reject the request if they differ.

`typing` (Boolean):
: `true` if the user is typing; `false` if typing has stopped.

### Response

`accountId` (String):
: The account ID from the request.

### Receiver Behavior

The receiving server MUST NOT persist this event.

The receiving server MUST verify that the ChatContact that `senderUserId`
resolves to ({{identity-mapping}}) is a current member of the identified
Chat before forwarding. If it is not a member, the server MUST return a `forbidden` method error and MUST NOT
deliver any event.

If `senderUserId` corresponds to a ChatContact whose `blocked` field is
`true` in the receiving account's contact list, the server MUST silently
drop the event and MUST NOT deliver it to any of the owner's clients.

The receiving server MUST apply the recipient's `receiveTypingIndicators`
preference (defined in {{JMAP-CHAT}}); if `false` for the identified
Chat (and the Chat kind is `"direct"` or `"group"`), the event MUST be
silently dropped and MUST NOT be forwarded to any of the owner's clients.
The calling server has no visibility into this preference and MUST invoke
`Peer/typing` as usual; the efficiency cost is accepted as a consequence
of the privacy-preserving design.

If all checks pass, the server MUST forward a typing push event (as
defined in {{JMAP-CHAT}}) to the owner's connected clients.

Servers MUST rate-limit inbound `Peer/typing` calls per peer to prevent
abuse. Servers SHOULD accept no more than one `Peer/typing` call per
peer per chat per 3 seconds; calls received above this rate MAY be
silently discarded without error.

Servers that cache a remote participant's `receiveTypingIndicators` value
SHOULD use a short TTL; the specific value is implementation-defined. As
there is no federation notification mechanism when a remote user updates
this preference, the TTL chosen affects how long a stale value may
persist after a remote user opts out of typing-indicator forwarding.

## Peer/retract

Requests that a remote mailbox tombstone a specific message on behalf of the original sender.

Method name: `Peer/retract`

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`chatId` (String):
: The Chat ID containing the message to be retracted.

`senderUserId` (String):
: The identity string of the user requesting retraction ({{identity-mapping}}). MUST match the peer identity.

`senderMsgId` (String):
: The sender-assigned ULID of the message to retract.

### Sender Behavior

When a `Message/set` update sets `deletedForAll: true` on a message whose `senderId` is the owner's `accountId` ({{JMAP-CHAT}}), the server MUST send `Peer/retract` to the server of every other participant in the Chat before marking the local record.

### Receiver Behavior

The receiving server MUST look up the message by `senderMsgId` within `chatId`. It MUST verify that the stored message's `senderId` is the `id` of the ChatContact that `senderUserId` resolves to ({{identity-mapping}}) before applying any changes. If the sender does not match, the server MUST reject the request.

On success, the server MUST: clear `body` to an empty string, clear `attachments` to an empty array, remove `editHistory`, set `deletedAt` to the current time, and set `deletedForAll` to `true`. The message record is retained as a tombstone.

### Response

`accountId` (String):
: The account ID from the request.

`retractedAt` (UTCDate):
: The time at which the retraction was applied.

## Peer/groupUpdate {#peer-groupupdate}

Notifies participant mailboxes of a new group chat or of a membership or metadata change to an existing one.

Method name: `Peer/groupUpdate`

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`chatId` (String):
: The group chat ULID.

`senderUserId` (String):
: The identity string of the user making the change ({{identity-mapping}}). MUST match the peer identity. The sending server SHOULD also verify locally that `senderUserId` is authorized to perform the action under its own group-chat admin model before invoking `Peer/groupUpdate`; this courtesy check avoids needless network traffic for operations the receiver will reject. The wire contract is the receiver-side admin verification described in the Receiver Behavior subsection below and in {{security}}.

`action` (String):
: One of the following values:

  - `"create"` — Initial group creation notification. The receiver MUST create a new Chat record of `kind: "group"` with this chatId.
  - `"addMembers"` — One or more members were added to the group.
  - `"removeMembers"` — One or more members were removed from the group.
  - `"updateRoles"` — One or more members' roles within the group were changed.
  - `"updateMetadata"` — The group's `name`, `description`, or `avatarBlobId` was changed.
  - `"leave"` — The sender is voluntarily departing the group. Arguments: none beyond the standard fields. The receiver MUST remove the sender from the group's member list without requiring admin authorization. This is equivalent to `"removeMembers"` with `removedMemberIds: [senderUserId]` but does not require admin role.
  - `"close"` — The sender, the group's last admin, is departing while other members remain, which closes the group ({{JMAP-CHAT}}). Arguments: none beyond the standard fields. The receiver MUST remove the sender from the group's member list and set the local Chat record's `closedAt`. The sending server sends `"close"` in place of `"leave"`.

`members` (ChatMember[], required for `"create"`):
: Full membership list at the time of this update, including the calling server's owner. Each ChatMember `id` and `invitedBy` carries an identity string. The receiver MUST resolve each one per {{identity-mapping}} and MUST create a placeholder ChatContact, populated with the `identity` only, for each identity it does not know.

`addedMembers` (ChatMember[], required for `"addMembers"`):
: The newly added members. Each ChatMember `id` and `invitedBy` is resolved as for `members`, including placeholder creation.

`removedMemberIds` (String[], required for `"removeMembers"`):
: The identity strings of the members who were removed ({{identity-mapping}}).

`updatedRoles` (Object[], required for `"updateRoles"`):
: Each entry MUST contain `id` (String, the identity string of the member, per {{identity-mapping}}) and `role` (String, the new role value: `"admin"` or `"member"`).

`name` (String, optional):
: Updated group display name. MAY be present for `"create"` or `"updateMetadata"` actions.

`description` (String, optional):
: Updated group description. MAY be present for `"create"` or `"updateMetadata"` actions.

`avatarBlobId` (String, optional):
: Updated group avatar blob ID. MAY be present for `"create"` or `"updateMetadata"` actions. For `"create"`, the receiver MAY fetch this blob from the sending server.

### Receiver Behavior

The receiving server MUST verify that `senderUserId` is authenticated. For all actions other than `"create"` and `"leave"`, it MUST also verify that the ChatContact that `senderUserId` resolves to ({{identity-mapping}}) holds an admin role in the named group on the receiving server's local Chat record before applying any changes. For `"create"`, it MUST verify that `senderUserId` is one of the members listed in `members`. For `"leave"`, it MUST verify that that ChatContact is a current member.

Once the local Chat record has `closedAt` set, the receiving server MUST reject every subsequent `Peer/groupUpdate` for that chat with `forbidden`.

On success, the receiving server updates its local Chat record to reflect the change indicated by `action`.

### Response

`accountId` (String):
: The account ID from the request.

### Space-Level Federation

This specification defines federation for group chats only. Space-level operations (channel creation, permission changes, SpaceBan propagation) are enforced by the Space-hosting server at delivery time (per the validation steps in Peer/deliver) but are not proactively federated to remote members. Remote members discover Space-level changes through rejected delivery attempts or via periodic `Space/get` synchronization. This includes Space closure ({{JMAP-CHAT}} `Space.closedAt`): no Peer/* method signals it, and the Space-hosting server rejects deliveries to a closed Space's channels with `forbidden` (see {{new-message-validation}}).

## Peer/subscribePresence {#peer-subscribepresence}

Requests that the remote server push presence updates for its owner to the calling server. Servers MAY implement this method; callers MUST handle `unknownMethod` gracefully.

Method name: `Peer/subscribePresence`

### Request Arguments

`accountId` (String):
: Account ID on the remote server whose owner's presence is being subscribed to.

`subscriberId` (String):
: The `ownerUserId` of the calling server. MUST match the peer identity; the remote server MUST reject the request if they differ.

`ttl` (UnsignedInt, optional):
: Requested subscription lifetime in seconds. Servers MAY cap or adjust this value. If absent, the server assigns an implementation-defined default TTL. Subscriptions that are not renewed before expiry SHOULD be silently dropped by the server.

### Response

`accountId` (String):
: The account ID from the request.

`presence` (String):
: The owner's current presence state at the time of subscription.

`lastActiveAt` (UTCDate, optional):
: Current value of the owner's `lastActiveAt`, if known.

`statusText` (String, optional):
: Current custom status text, if set.

`statusEmoji` (String, optional):
: Current status emoji, if set.

`ttl` (UnsignedInt):
: The actual subscription lifetime in seconds as granted by the server.

### Behavior

On success, the remote server MAY push subsequent `Peer/presence` calls to the subscriber whenever the owner's presence, `statusText`, or `statusEmoji` changes. Servers SHOULD rate-limit outbound `Peer/presence` calls per subscriber; the specific rate is implementation-defined. Servers MAY drop presence push deliveries that fail; presence delivery is best-effort.

Calling `Peer/subscribePresence` again before the TTL expires renews the subscription and SHOULD reset the TTL to the newly granted value.

## Peer/unsubscribePresence {#peer-unsubscribepresence}

Cancels a presence subscription before its TTL expires. Servers MAY implement this method; callers MUST handle `unknownMethod` gracefully. Servers MUST succeed if no active subscription exists for the given `subscriberId`.

Method name: `Peer/unsubscribePresence`

### Request Arguments

`accountId` (String):
: Account ID on the remote server.

`subscriberId` (String):
: The `ownerUserId` of the calling server. MUST match the peer identity.

### Response

`accountId` (String):
: The account ID from the request.

## Peer/presence {#peer-presence}

Delivers a presence update for the calling server's owner to a subscriber. Servers MAY implement this method; callers MUST handle `unknownMethod` gracefully.

Method name: `Peer/presence`

### Request Arguments

`accountId` (String):
: Account ID on the receiving server.

`identity` (String):
: The `ownerUserId` of the pushing server, an identity string ({{identity-mapping}}). MUST match the peer identity; the receiver MUST reject the request if they differ.

`presence` (String):
: The updated presence state of the remote owner. One of `"online"`, `"away"`, `"busy"`, `"invisible"`, or `"offline"`.

`lastActiveAt` (UTCDate, optional):
: Updated `lastActiveAt` timestamp, if available.

`statusText` (String, optional):
: Updated custom status text. If absent, the receiver SHOULD treat the current value as unchanged. To explicitly clear the field, the sender MUST send `null`.

`statusEmoji` (String, optional):
: Updated status emoji. If absent, the receiver SHOULD treat the current value as unchanged. To explicitly clear the field, the sender MUST send `null`.

### Response

`accountId` (String):
: The account ID from the request.

### Receiver Behavior

If the ChatContact whose `identity` equals the `identity` argument has its `blocked` field set to `true` in the receiving account's contact list, the server MUST silently drop the request: it MUST NOT update the ChatContact record and MUST NOT fire any local presence push event. The server SHOULD return a success response to avoid disclosing block status to the calling server.

Otherwise, the receiving server MUST update the `presence`, `lastActiveAt`, `statusText`, and `statusEmoji` fields on the ChatContact record whose `identity` equals the `identity` argument. The `statusText` and `statusEmoji` fields mirror `PresenceStatus.statusText` and `PresenceStatus.statusEmoji` from the remote owner's server; a `null` value in the request MUST clear the corresponding ChatContact field (setting it to absent). The server MUST then fire a local presence push event (as defined in {{JMAP-CHAT}}) to the owner's connected clients.

# Outbox and Delivery {#outbox}

## Persistent Queue

Outbound messages MUST be durable across local server restart prior to delivery confirmation; the specific mechanism (persistent outbox, write-through queue, transactional store with the local message-store write, replicated in-memory queue backed by an upstream broker with at-least-once delivery, etc.) is implementation-defined. This ensures that messages are not lost if the remote server is temporarily unreachable or if the local server restarts between the time a message is created and the time delivery is confirmed.

## Retry and Backoff

Servers MUST retry failed delivery attempts with exponential backoff. The minimum initial retry interval, maximum retry interval, and total retry duration are implementation-defined, but implementations SHOULD apply a jitter factor to avoid synchronized retry storms from multiple servers.

A delivery attempt is considered failed when the remote server returns an error response, when the connection cannot be established, or when no response is received within an implementation-defined timeout.

## Group Chat Delivery

For group chats, the sender delivers independently to each participant's mailbox and tracks per-recipient state in the `deliveryReceipts` field of the outgoing Message record. The aggregate `deliveryState` advances to `"delivered"` when all participants have acknowledged receipt.

A message whose `senderMsgId` is already known for the given chat at the receiving server MAY be silently discarded by that server. Sending servers SHOULD treat a successful response (even for a discarded duplicate) as confirmation that delivery is complete for that recipient.

## Delivery State Transitions

The `deliveryState` field on an outbound Message transitions as follows:

- `"pending"` — Initial state. The message is queued but no successful delivery has occurred.
- `"delivered"` — All recipients (for group chats: all participants) have acknowledged receipt.
- `"failed"` — Delivery has been abandoned after exhausting retries for one or more recipients.
- `"received"` — The message has been explicitly read by the recipient (indicated via `Peer/receipt`). (Note: despite the name, this state indicates the message was *read* by the recipient, not merely received by the server. The `"delivered"` state indicates server-level receipt.)

In group chats delivered to multiple peers, `deliveryState` reflects the aggregate: `"delivered"` when at least one peer has accepted, `"failed"` only when ALL peers have permanently failed. Per-recipient granularity is available via `deliveryReceipts` if supported.

### Message Ordering {#message-ordering}

Message ordering is receiver-determined: each server assigns `receivedAt` independently upon acceptance, and message order MAY differ across servers in the same group chat. This is a known property of the decentralized model. There is no vector clock or consensus mechanism. Concurrent operations (retract vs. react, edit vs. edit) resolve by last-write-wins on each server independently.

Cross-server references to a message use its `senderMsgId` (see {{outbound-preparation}}). A sending server MAY include the optional `lastSeenSenderMsgId` causal hint in a `message` payload. A receiving server MAY store the hint and MAY use it, or expose it to clients, to order the display of messages that arrive out of causal order, for example placing a message after the message it names. The hint MUST NOT change `receivedAt`, MUST NOT be used for expiry or rate-limit calculations, and MUST NOT cause rejection; a hint that does not resolve to a known message MUST be ignored.

### Error Handling

Errors in Peer/* methods follow the standard JMAP method-level error format ({{RFC8620}} Section 3.6.2). Error types used: `invalidArguments` (permanent — do not retry), `forbidden` (permanent — do not retry), `rateLimited` (transient — retry after backoff), `serverFail` (transient — retry with exponential backoff). HTTP-level 5xx responses or timeouts are treated as transient failures. Error responses MAY include a `description` field for debugging but MUST NOT disclose private information such as block status.

# Security Considerations {#security}

## Identity Verification {#identity-verification}

The `senderUserId` argument in all Peer/* methods is caller-supplied and MUST be treated as untrusted. The local server MUST obtain the peer identity independently, as the identity bound to the request's authenticated peer domain ({{identity-binding}}), and MUST compare it to `senderUserId` before any storage, state change, or side effect. This comparison MUST precede all effects of the method. If the values differ, the request MUST be rejected immediately.

## Signing Key Compromise {#key-compromise}

Anyone holding a server's signing private key can act as that peer domain toward every other server until the key stops being usable. Removing the key from `peerSigningKeys` (and from DNS, if published there) ends its use, but receiving servers may keep accepting it for up to the one-hour key cache limit ({{key-publication}}), or the DNS TTL when a DNSSEC binding is in use. Servers SHOULD keep signing keys out of the process that handles untrusted input, for example in a hardware security module or a separate signing service.

## Replay {#replay}

The `nonce`, `created`, and `expires` checks ({{request-signing}}) bound replay: a captured request is rejected after its validity window, and within it the nonce cache rejects a second use. Covering `@target-uri` prevents a request from being replayed to a different server or endpoint, and covering `content-digest` prevents the content from being altered. A receiving server whose nonce cache is lost, for example on restart, SHOULD reject requests whose `created` precedes the restart, since it can no longer detect their replay.

## DNS and Certificate Authority Trust {#dns-ca-trust}

The baseline mechanism trusts the WebPKI: anyone who can obtain a certificate for a peer domain from a certificate authority the receiving server trusts, or who controls the domain's DNS, can publish keys for that domain. The DNSSEC binding ({{dnssec-keys}}) moves that trust to the DNSSEC chain of the peer domain's zone, so a misissued certificate alone no longer suffices; it does not help a domain whose zone is unsigned. A bogus DNSSEC answer is rejected, not treated as absent, so that an attacker who forges DNS answers cannot strip the binding. A lookup that fails or whose result is indeterminate falls back to the HTTPS-published keys ({{dnssec-keys}}), so an attacker who can block the receiving server's DNS lookups can strip the binding and leave only the baseline WebPKI trust. Mutual TLS with DANE ({{mutual-tls}}) adds a DNSSEC-anchored check at the TLS layer but rests on the same DNSSEC trust.

## Blob Fetch and SSRF {#ssrf}

When fetching attachment blobs from peer-supplied `fetchUrl` values, servers MUST restrict outbound HTTP connections to the known peer address space. Specifically, servers MUST NOT follow redirects to addresses outside the peer's known IP range or hostname, and MUST NOT connect to link-local, loopback, or private-network addresses in response to a peer-supplied URL. Unrestricted fetches from peer-supplied URLs are a Server-Side Request Forgery (SSRF) vector that could allow a malicious peer to probe internal services.

The same restriction applies to any other context in which a server fetches from a URL supplied by a peer, including blob URIs carried in `Message.actions` or `ChatContact.endpoints`.

## Chat ID Integrity {#chat-id-integrity}

Chat IDs are server-assigned ULIDs. Security against cross-conversation injection relies on sender authentication and chat membership verification, not on ID unpredictability.

For direct chats, the receiving server MUST confirm that the incoming `chatId` matches the chatId already associated with the sending contact, if one exists. This prevents a sender from injecting messages into a chatId that belongs to a different conversation.

For group and channel chats, servers MUST confirm the sender is a current member of the identified chat before accepting any `Peer/deliver` or `Peer/groupUpdate` call.

## Direct Chat Simultaneous Initiation {#direct-chat-race}

A race condition exists when two servers simultaneously initiate a direct chat with each other. Each server assigns its own ULID as the chatId before any delivery occurs, resulting in two distinct chatIds for the same logical conversation. This is a known limitation of the stateless ULID assignment model.

Implementations SHOULD resolve such conflicts using lexicographic ULID ordering: when a `Peer/deliver` arrives for a direct chat with an unknown chatId X, and the receiving server already has a direct chat chatId Y with the same sender, both servers SHOULD adopt `min(X, Y)` (lexicographically smaller) as the canonical chatId. The receiving server SHOULD migrate any messages stored under the non-canonical chatId to the canonical one, and SHOULD return the canonical chatId in the `Peer/deliver` response so that the sending server can update its records.

Servers that do not implement this resolution will produce duplicate direct-chat records for the affected pair. Client applications SHOULD detect and surface duplicate direct chats with the same contact to the user.

## Retract Authorization

`Peer/retract` MUST verify, via the `senderMsgId` index, that the ChatContact that `senderUserId` resolves to ({{identity-mapping}}) is the original sender of the identified message before applying any tombstone. Servers MUST NOT allow retraction of messages sent by other users.

## Group Admin Verification

`Peer/groupUpdate` MUST verify that the ChatContact that `senderUserId` resolves to ({{identity-mapping}}) holds an admin role in the named group on the receiving server's local Chat record before applying any membership or metadata changes, except for `"leave"` actions, which require only that the sender be a current member. For `"create"` actions, the sender MUST be listed as a member in the provided `members` array. Servers MUST reject requests where these conditions are not met.

## Direct Address Hints {#direct-address-security}

The `directAddress` field on ChatContact records and the `ownerDirectAddress` field in Session objects are peer-supplied values and MUST be treated as untrusted. Implementations that use these values for direct delivery MUST apply the same authentication requirements to the direct path as to the standard mailbox path. Servers MUST NOT use `directAddress` or `ownerDirectAddress` as `fetchUrl` targets for blob fetches or in any other context subject to the SSRF restrictions described in {{ssrf}}.

Senders SHOULD NOT treat successful delivery to a `directAddress` as a substitute for mailbox delivery. The standard mailbox path SHOULD always be used to ensure message persistence and multi-device visibility. Direct address delivery, when attempted, is an optimization only.

## Blocked Contacts

Messages from a contact whose `blocked` field is `true` MUST be silently dropped by the receiving server, regardless of whether they arrive in a direct chat or a group chat context. The server SHOULD return a success response to avoid disclosing the block status to the sender.

## Cross-Server Message Injection

A malicious peer could attempt to inject messages into chats it is not a party to by forging `chatId` or `senderUserId` values. The validation steps in {{new-message-validation}} — particularly the peer identity check in step 2, the chat membership check in step 3, and the chatId correspondence check for direct chats — collectively prevent this attack. Implementations MUST perform all of these steps and MUST NOT skip any of them.

## Broadcast Mention Resolution Trust

The recipient set targeted by a broadcast mention (`@everyone`, `@here`, or `@admins`) is computed by the receiving server at delivery time per {{broadcast-mention-resolution}}. A receiving server MUST NOT accept any out-of-band hint from the sending server about which recipients were "intended" by the sender; trusting such a hint would let a hostile or buggy peer force broadcast-mention push elevation against recipients who would not qualify under the receiving server's own predicate. The wire payload defined in this document does not carry a resolved id list, and any extension that does carry one MUST treat it as informational only.

A receiving server's targeted-owner opt-out preferences (for example, "always honor mute for this account") are deployment-defined and MUST NOT be exposed via the federation protocol. Disclosing whether a remote owner has chosen to suppress broadcast elevation would let a sender confirm which recipients are silently filtering them; servers MUST NOT reveal this state in any Peer/* response or error.

## Placeholder ChatContact Creation

A peer can cause a receiving server to create placeholder ChatContacts ({{identity-mapping}}) by naming identities the receiver does not know. Servers MAY limit how many placeholder ChatContacts a peer can cause to be created. A server that applies such a limit rejects, with `rateLimited`, any request that would exceed it, and that request creates no placeholders.

## Presence Subscription Abuse

Servers implementing `Peer/subscribePresence` MUST authenticate the subscriber before recording the subscription (see {{peer-subscribepresence}}). Unauthenticated or mismatched `subscriberId` values MUST be rejected. Servers SHOULD limit the number of active presence subscriptions per peer to bound fan-out costs on presence changes. TTL-based expiry ensures that subscriptions from peers that have gone away do not accumulate indefinitely.

## End-to-End Encryption Relay Considerations

When the federation protocol is used in conjunction with end-to-end encrypted deployments (as described in {{JMAP-CHAT}}), the relay or forwarding server routes Peer/* messages but MUST NOT have access to plaintext message content. Servers operating in an encrypted deployment MUST ensure that `body` carries only ciphertext when `bodyType` indicates an encrypted payload type (e.g., `"application/mls-ciphertext"`). The encryption key schedule MUST exclude the relay (e.g., by using MLS {{RFC9420}} or a similar protocol that does not involve the relay in key agreement).

The End-to-End Encrypted Deployments section of {{JMAP-CHAT}} defines which `Peer/deliver` fields are outside the ciphertext. When `bodyType` is encrypted, `mentions`, `broadcastMentions`, attachment metadata, `actions`, `replyTo`, `threadRootId`, `topicId`, `subject`, and `lastSeenSenderMsgId` are carried inside the ciphertext or omitted, and steps 11 through 13 of {{new-message-validation}} are skipped. The sender MAY be authenticated inside the end-to-end encryption layer, in which case a relay needs only a recipient or group routing handle, and the sending server's `ownerUserId`, which `senderUserId` carries, MAY be an opaque value that does not reveal the sender's address, provided its host is the sending server's peer domain (for example, `k3x9q7@relay.example`). That opaque value is an identity string like any other: it is bound to the authenticated peer domain per {{identity-binding}} and mapped per {{identity-mapping}}; it is not exempt. A host-less opaque value fails {{identity-binding}} and is rejected. Relay-side enforcement of SpaceBan and slow mode is OPTIONAL; receiving clients enforce any check the relay does not perform against signed group state.

Metadata visible to relay servers (including the routing fields, timestamp, body size, and `reactionUpdate` and `Peer/retract` arguments) remains an information-leakage surface even in encrypted deployments. Deployments with strong metadata-privacy requirements SHOULD apply message padding and cover traffic at the transport layer; those techniques are outside the scope of this document.

# IANA Considerations

IANA is requested to add the following entries to the "Underscored and Globally Scoped DNS Node Names" registry {{RFC8552}}:

| RR Type | _NODE NAME | Reference |
|---|---|---|
| TXT | _jmap-peer | This document, {{dnssec-keys}} |
| TLSA | _jmap-peer | This document, {{mutual-tls}} |

This document requests no other IANA registrations. The `urn:ietf:params:jmap:chat` capability URI is registered by {{JMAP-CHAT}}. This document extends the semantics of that capability by defining the `role: "peer"` account capability value and the Peer/* server-to-server methods; no additional capability URI is required.

Servers that implement this document MUST advertise the `urn:ietf:params:jmap:chat` capability in peer-role accounts as specified in {{account-capability}}.

--- back

# Acknowledgements

The author thanks the JMAP working group for {{RFC8620}} and the author of {{RFC8615}} for the Well-Known URI mechanism that peer discovery relies upon.
