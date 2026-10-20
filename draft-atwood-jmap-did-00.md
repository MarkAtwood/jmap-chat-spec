---
title: JMAP Decentralized Identifiers
abbrev: JMAP DID
docname: draft-atwood-jmap-did-00
category: exp
stream: independent

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
  DID-WEBVH:
    title: "The did:webvh DID Method v1.0"
    author:
      org: Decentralized Identity Foundation
    target: https://identity.foundation/didwebvh/v1.0/

informative:
  RFC7515:
  RFC8621:
  RFC8984:
  RFC9421:
  RFC9610:
  JMAP-CALENDARS:
    title: "JSON Meta Application Protocol (JMAP) for Calendars"
    author:
      - fullname: Neil Jenkins
      - fullname: Michael Douglass
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-calendars-33
    date: 2026-10-13
  JMAP-FILENODE:
    title: "JMAP File Storage extension"
    author:
      - fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-filenode-17
    date: 2026-10-18
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-VTC:
    title: "JMAP for Video/Voice Teleconferencing"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  JMAP-SCENE:
    title: "JMAP for Scenes"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026
  JMAP-CID:
    title: "JMAP Blob Content Identifiers"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-cid-00
    date: 2026
  DID-WEB:
    title: "did:web Method Specification"
    author:
      - fullname: Michael Prorock
        role: editor
      - fullname: Oliver Terbu
        role: editor
    target: https://w3c-ccg.github.io/did-method-web/
    annotation: "W3C Credentials Community Group unofficial draft."
  DID-KEY:
    title: "The did:key Method v0.9"
    author:
      - fullname: Dave Longley
      - fullname: Dmitri Zagidulin
      - fullname: Manu Sporny
    target: https://w3c-ccg.github.io/did-key-spec/
    annotation: "W3C Credentials Community Group draft report."
  VC-DATA-MODEL:
    title: "Verifiable Credentials Data Model v2.0"
    author:
      - fullname: Manu Sporny
        role: editor
      - fullname: Ted Thibodeau Jr
        role: editor
      - fullname: Ivan Herman
        role: editor
      - fullname: Gabe Cohen
        role: editor
      - fullname: Michael B. Jones
        role: editor
    seriesinfo:
      W3C: REC-vc-data-model-2.0-20250515
    date: 2025-05-15
    target: https://www.w3.org/TR/2025/REC-vc-data-model-2.0-20250515/

--- abstract

This document defines the `urn:jmap:did` JMAP capability. When a server advertises this capability, every JMAP account is bound to a Decentralized Identifier (DID) as defined in {{DID-CORE}}. The capability adds a `did` property to JMAP data types designated by consuming specifications, enabling globally resolvable, cryptographically verifiable identity across JMAP servers. The account-to-DID binding is server-authoritative; clients MAY independently verify the binding by resolving the DID document.

--- middle

# Introduction {#introduction}

JMAP ({{RFC8620}}) identifies users by server-assigned, server-local `accountId` values. An accountId is meaningful only within the server that issued it. Two JMAP servers have no standard mechanism to determine whether accounts on each server represent the same person, and clients have no way to verify an accountId claim without trusting the server.

Decentralized Identifiers (DIDs) ({{DID-CORE}}) are globally unique, self-sovereign identifiers that resolve to DID documents containing cryptographic key material. A DID is not scoped to any server. A user's DID persists across servers, survives server migration, and can be verified by any party that can resolve the DID document.

This document bridges the two by defining a JMAP capability that binds each account to a DID. When this capability is present:

- Every account has an associated DID, advertised in the account-level capability object.
- JMAP data types designated by consuming specifications gain a server-set `did` property recording the owning account's DID at creation.
- Clients MAY verify the server's DID attestation independently.
- DID lifecycle events (key rotation, DID migration, suspension, revocation) have defined effects on the JMAP session.

The capability is horizontal infrastructure. It does not define new data types or methods. It extends existing JMAP objects with identity metadata, the same way {{JMAP-CID}} extends blob uploads with integrity metadata. Any JMAP specification — mail ({{RFC8621}}), calendars ({{JMAP-CALENDARS}}), contacts ({{RFC9610}}), file storage ({{JMAP-FILENODE}}), chat ({{JMAP-CHAT}}), video conferencing ({{JMAP-VTC}}), spatial environments ({{JMAP-SCENE}}), or any future JMAP capability — may declare which of its data types carry DID-derived properties.

## Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Capability {#capability}

A server supporting this extension MUST advertise the `urn:jmap:did` capability in the JMAP Session resource ({{RFC8620}} Section 2).

## Session-Level Capability Object

The value of `capabilities["urn:jmap:did"]` is a JSON object with the following fields:

`supportedMethods` (String[]):
: The DID methods the server is able to resolve and verify. Each value is a DID method name as defined in {{DID-CORE}} Section 8 (e.g., `"key"`, `"web"`). Servers MUST support at least one method. Every account's bound DID MUST use a listed method. Companion specifications that need to advertise DID method support reuse this field rather than defining their own.

`verificationEndpoint` (String|null):
: A URL where clients can submit a DID for server-side verification. `null` if the server does not offer a verification endpoint. The verification protocol is out of scope for this document.

Example:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": {},
    "urn:jmap:did": {
      "supportedMethods": ["key", "web"],
      "verificationEndpoint": null
    }
  }
}
~~~

## Account-Level Capability Object {#account-capability}

The value of `accountCapabilities["urn:jmap:did"]` is a JSON object with the following fields:

`did` (String):
: The DID bound to this account. This is the canonical, globally resolvable identifier for the account holder. The value MUST be a valid DID as defined in {{DID-CORE}} Section 3.1.

`didDocumentUri` (String|null):
: A URI where the DID document can be fetched. For `did:web` ({{DID-WEB}}) this is derivable from the DID itself; for `did:webvh` it is the URL of the DID log (`did.jsonl`), also derivable from the DID ({{DID-WEBVH}}), and a log fetched from this URI is resolved as defined in {{did-webvh}}, like one fetched from the method-derived URL. The witness proofs file (`did-witness.json`) always comes from the method-derived URL ({{DID-WEBVH}}), even when this URI served the log. For `did:key` ({{DID-KEY}}) the document is constructed from the DID. Servers MAY provide this as a convenience; `null` means the client should resolve the DID document using the standard resolution mechanism for the DID method. When `didDocumentUri` is non-null but unreachable (network error, HTTP 4xx/5xx), clients SHOULD fall back to standard method-specific resolution for the DID. The `didDocumentUri` is a convenience hint, not a binding constraint.

`previousDid` (String|null):
: The DID bound to this account immediately before the most recent DID migration ({{migration}}). The server retains it for a deployment-defined transition period after the migration. `null` when no migration has occurred or the transition period has expired.

`keyCustody` (String|null):
: Who holds the private keys for the verification methods in the bound DID's `authentication` verification relationship. `"server"` means the server, or a party operating on its behalf, holds at least one of those private keys. `"user"` means the server holds none of them. `null` means the server does not disclose custody; clients and peers SHOULD NOT infer `"user"` from `null`. See {{binding}}.

Example:

~~~json
{
  "accounts": {
    "acct-001": {
      "accountCapabilities": {
        "urn:jmap:did": {
          "did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
          "didDocumentUri": null,
          "previousDid": null,
          "keyCustody": "user"
        }
      }
    }
  }
}
~~~

# Account-to-DID Binding {#binding}

The server is authoritative for the mapping between an accountId and a DID. This document does not prescribe how the binding is established. Deployments MAY use any mechanism, including but not limited to:

- **OIDC claim mapping.** The identity provider includes a DID in an ID token claim; the server extracts it at authentication time.
- **Employer-managed sidecar.** An external process watches identity provider events (Okta, Google Workspace, Active Directory, etc.) and maintains DID documents on behalf of users. The server queries the sidecar for the accountId-to-DID mapping.
- **Self-registration.** The user presents a DID and proves control of the corresponding private key (e.g., by signing a challenge). The server records the binding after verification.
- **Server-generated.** The server generates a `did:key` from a keypair it controls, on behalf of the user. This is the simplest deployment model but provides no user-sovereign key control.

When the server, or a party operating on its behalf, holds a private key for the bound DID's `authentication` verification relationship (as in the server-generated model), the server can produce signatures indistinguishable from the user's, and the DID gives no assurance independent of the server ({{server-generated-keys}}). Such a server cannot set `keyCustody` to `"user"` ({{account-capability}}), SHOULD NOT otherwise represent the binding as verifiable independently of the server, and SHOULD set `keyCustody` to `"server"` so that clients and federated peers can choose a trust level. The server-generated model is NOT RECOMMENDED outside employer-managed deployments ({{employer-managed}}).

Regardless of mechanism, the server MUST ensure that:

1. Each account is bound to exactly one DID at any given time.
2. The binding is reflected in the account-level capability object ({{account-capability}}).
3. Changes to the binding (DID migration, revocation) are propagated as defined in {{lifecycle}}.

# The `did` Property {#did-property}

## Definition

When `urn:jmap:did` is advertised, consuming specifications MAY designate their data types as DID-bearing. A DID-bearing data type gains the following property:

A DID-bearing property is a String\|null value, server-set, whose value is the DID given by the account-to-DID binding defined in {{binding}} at the time the object was created. The value is `null` when the account has no DID binding (transitional state during migration) or when the object represents a non-user entity (system-generated objects, anonymous participants).

The property name is not required to be `did`.

A DID-referencing property holds the DID of a party other than the owning account: a peer, a participant, an organizer, or a sender as seen by a recipient (e.g., the DID of a remote peer on a ChatContact record). Its value is set as the consuming specification defines, not from the binding, and is not trusted until verified ({{verification}}). A data type MAY carry DID-referencing properties alongside a DID-bearing one; for example, a delegation record might carry `delegatorDid` (DID-bearing) and `delegateeDid` (DID-referencing). The rest of this section applies to DID-bearing properties only.

All DID-bearing properties share these semantics:

- **Server-set.** Clients omit DID-bearing properties when creating objects, as {{RFC8620}} Section 1.1 requires for server-set properties; an update can include one only with its current value ({{RFC8620}} Section 5.3). Servers MUST reject a create that includes a DID-bearing property, or an update that changes one, with a SetError of type `invalidProperties`, listing the DID-bearing property name in the `properties` array.
- **Recorded at creation.** The value is the DID bound to the relevant account when the object was created, and the server records it with the object. A later DID migration ({{migration}}) does not change it.
- **Immutable.** The value never changes after creation, neither as a result of object mutation nor as a result of a change to the account-to-DID binding.

Signatures and other attestations on an object verify against the DID recorded on that object, not against the account's current DID.

## What Consuming Specifications Define

A consuming specification that designates a data type as DID-bearing needs to define:

1. Which data type gains DID-bearing properties, and the property names.
2. Which DID-referencing properties the type carries, how their values are set, and how they are verified.
3. The semantic role of the DID on that type (see {{did-roles}}).
4. Any type-specific behavior when the DID is migrated or revoked.

This document does not unilaterally add DID-bearing properties to any JMAP data type. Each consuming specification opts in by declaring its DID-bearing types.

## Semantic Roles {#did-roles}

A DID on a JMAP object can serve different purposes depending on the data type and the consuming specification's intent. Common roles include:

**Authorship.** The DID identifies who created or sent the object. Enables cross-server sender verification without trusting the originating server. Applicable to email messages, chat messages, calendar event invitations, and file uploads.

**Ownership.** The DID identifies who owns or controls the object. Enables portable ownership claims that survive server migration. Applicable to files, contacts, 3D assets, virtual objects, and any resource with an access control model.

**Participation.** The DID identifies a participant in a multi-party interaction. Enables verifiable participant identity in contexts where trust in the hosting server is insufficient. Applicable to call participants, shared document editors, calendar event attendees, and chat room members.

**Delegation.** The DID identifies a party in a delegated authority relationship. A calendar delegate, a shared mailbox accessor, or an authorized API client may each carry a DID that enables the receiving party to verify the delegation chain. An object that involves more than one identity carries one DID-bearing property for the owning account (e.g., `ownerDid`) and DID-referencing properties for the others (e.g., `delegateDid`).

**Provenance.** The DID establishes a verifiable chain of custody. Combined with content-addressed references (e.g., SHA-256 digests from {{JMAP-CID}}), a DID on an asset or file creates a binding between content identity and owner identity that is verifiable without trusting the hosting server.

**Contact identity.** The DID provides a stable, cross-server identifier for a person or organization in a contact record. Enables contact deduplication across servers and portable block/allow lists.

Consuming specifications SHOULD identify which role or roles apply to each DID-bearing or DID-referencing property they define.

## Illustrative Extensions {#extensions}

The following are non-normative illustrations of how JMAP specifications could designate DID-bearing types and DID-referencing properties; each is labeled with its kind. The normative declarations belong in the consuming specifications themselves. These examples demonstrate the breadth of applicable roles; they are not exhaustive.

### JMAP Mail ({{RFC8621}})

**Email** — `senderDid` (role: authorship; DID-bearing on the sender's own copy, DID-referencing as received). The DID of the authenticated sender. Complements existing email authentication mechanisms (DKIM, SPF, DMARC) with a sender-controlled identity that is independent of the sending domain. A recipient's client can resolve the DID document and verify the sender's key material without trusting the sending server or relying on DNS-based authentication alone.

**Identity** — `did` (role: ownership; DID-bearing). The DID bound to the sending identity. When a user has multiple JMAP identities (e.g., personal and work email addresses), each identity MAY be bound to the same or different DIDs.

### JMAP Calendars ({{JMAP-CALENDARS}})

CalendarEvent and Participant are defined by {{JMAP-CALENDARS}}; their data format is JSCalendar ({{RFC8984}}).

**CalendarEvent** — `organizerDid` (role: authorship; DID-bearing on the organizer's copy, DID-referencing on an invitee's copy). The DID of the event organizer. Enables cross-server invitation verification: an invitee can verify that the invitation was issued by the claimed organizer without trusting the calendar server that delivered it.

**CalendarEvent** — `delegateDid` (role: delegation; DID-referencing). When a calendar event was created by a delegate on behalf of another user, the delegate's DID enables the receiving party to verify the delegation chain.

**Participant** — `did` (role: participation; DID-referencing). The DID of a calendar event participant. Enables deduplication of participants across servers (two servers may have different accountIds for the same person, but the DID is the same).

### JMAP Contacts ({{RFC9610}})

**ContactCard** — `did` (role: contact identity; DID-referencing). A stable, globally resolvable identifier for the person or organization described by the contact. Enables contact deduplication across servers, portable block/allow lists, and cross-server contact lookup.

### JMAP File Storage ({{JMAP-FILENODE}})

**FileNode** — `ownerDid` (role: ownership/provenance; DID-bearing). The DID of the file owner. Combined with the `sha256` field from {{JMAP-CID}}, creates a verifiable content-addressed ownership binding: "the file with content hash H belongs to DID D." Enables portable file provenance that survives migration between storage servers.

### JMAP Chat ({{JMAP-CHAT}})

**Message** — `senderDid` (role: authorship; DID-bearing on the sender's own copy, DID-referencing as received). The DID of the message sender. Enables cross-server sender verification: a recipient on Server B can verify that a federated message from Server A was authored by the claimed DID without trusting Server A.

### JMAP VTC ({{JMAP-VTC}})

**VTCParticipant** — `did` (role: participation; DID-referencing). The DID of the call participant. Enables cryptographic participant identity verification in calls that span server boundaries.

### JMAP Scene ({{JMAP-SCENE}})

**SceneAvatar** — `did` (role: participation; DID-referencing). The DID of the user whose avatar this is. Persistent cross-server avatar identity: the same DID appears on your avatar regardless of which JMAP Scene server hosts the region.

**SceneObject** — `ownerDid` (role: ownership/provenance). The DID of the object creator. Combined with the `sha256` field on the object's SceneAsset and {{JMAP-CID}}, enables verifiable content-addressed ownership of 3D assets.

**SceneAsset** — `uploaderDid` (role: provenance). The DID of the asset uploader. Provenance chain for 3D models, textures, and audio resources.

# Client Verification {#verification}

Clients receiving a `did` value on a JMAP object MAY verify the server's attestation independently:

1. Resolve the DID to a DID document using the standard resolution mechanism for the DID method ({{DID-CORE}} Section 7).
2. Extract the authentication verification method from the DID document.
3. Verify that the server has demonstrated control of a key listed in the DID document's `authentication` verification relationship. For `did:web` where the JMAP server's domain matches the DID's domain component, successful TLS-authenticated retrieval of the DID document provides implicit domain binding. This implicit binding does not apply to `did:webvh`, because the host serving the DID log need not hold any key in it. For all other cases, the server MUST present a signed assertion (e.g., a JWS ({{RFC7515}}) or HTTP Message Signature ({{RFC9421}})) that validates against the DID document's authentication key.

Verification is OPTIONAL. In a single-operator deployment where the client trusts the server, verification adds overhead with no security benefit. In a federated or zero-trust deployment, verification allows clients to detect a compromised or dishonest server that claims a DID it does not control.

A client that performs verification and detects a mismatch SHOULD treat the `did` value as unverified and can surface the discrepancy to the user. The client SHOULD NOT silently discard the object or terminate the session; the mismatch may indicate a configuration error rather than an attack.

## Cross-Server Verification

When a client receives objects that originate from a different server (e.g., federated email, cross-server calendar invitations, federated chat messages, shared files), the originating server's DID-bearing value arrives as a DID-referencing value, which the client can verify without trusting the originating server:

1. The receiving server includes the `did` as attested by the originating server.
2. The client resolves the DID document independently.
3. If the originating server's attestation matches the DID document, the identity is verified. If not, the client knows the originating server's claim is untrustworthy.

This pattern requires no direct trust relationship between the client and the originating server. The DID document, resolvable by any party, serves as the trust anchor.

## Resolving `did:webvh` {#did-webvh}

A `did:webvh` DID ({{DID-WEBVH}}) has the form `did:webvh:<SCID>:<domain>[:<path>]`. Its DID document is the state in the latest valid entry of a hash-chained DID log (`did.jsonl`). Each entry is signed by an active update key: from the previous entry, or under pre-rotation from the entry itself, committed to by the previous entry's `nextKeyHashes`. The self-certifying identifier (SCID) is derived from the genesis entry.

A client or server resolves a `did:webvh` DID as defined in {{DID-WEBVH}}, which specifies verification of the DID log, the handling of invalid entries, witness proofs, and the treatment of a parallel `did:web` document. This document adds no resolution requirements.

Note: {{DID-WEBVH}} allows a resolver to treat the first invalid entry and every later entry as invalid. Versions before the first invalid entry remain usable for checking stored objects accepted under them.

A valid log shows only that each entry the resolver received was authorized. A holder of an update key can create two valid children of one entry and present them to different resolvers (a split view), and whoever serves the log can present an older, valid prefix of it (a rollback). Witnesses ({{DID-WEBVH}}) mitigate split view, because a witness does not sign two children of one parent. A rollback is detected only against a newer `versionId` the resolver has already seen, or by comparing sources such as watchers ({{DID-WEBVH}}). A resolver seeing a DID for the first time, with no witnesses or watchers, has no protection against either.

To verify a signature or attestation on an object it accepted earlier and stored, a party uses the DID document version that was in effect when it accepted the object, not the current version. A party that stores such objects SHOULD record with each object the `versionId` in effect at acceptance, and verify against that version. Without it, the party cannot re-verify stored history after a key rotation ({{key-rotation}}), because a key removed by a later log entry no longer appears in the current document. For an object being received, the current version applies.

A holder MAY publish a parallel `did:web` document for the same domain and path ({{DID-WEBVH}}). That document is outside the log's verification, and its keys carry none of the log's guarantees ({{DID-WEBVH}}).

# DID Lifecycle {#lifecycle}

This document uses "rotation" only for key rotation. A change of the DID itself is a DID migration.

## Key Rotation {#key-rotation}

A key rotation replaces verification methods in the DID document while the DID stays the same. Whether and how keys can be rotated depends on the DID method (for example, `did:web` supports it by updating the hosted document; `did:webvh` supports it by appending a DID log entry signed by an authorized update key, which yields a new `versionId` ({{did-webvh}}); `did:key` does not, because a new key is a new DID).

A key rotation requires no action at the JMAP layer. The account-level capability object and all DID-bearing property values are unchanged. Clients and peers learn of the new keys by resolving the DID document again.

## DID Migration {#migration}

A DID migration occurs when an account's bound DID changes. This may happen due to key compromise recovery for a method without key rotation, method migration (e.g., `did:key` to `did:web`), or organizational policy.

A `did:webvh` DID whose log enables portability can move to a new domain or path while keeping its SCID and log ({{DID-WEBVH}}). The DID string changes, so the move is a DID migration as defined here. The moved DID document lists the previous DID in `alsoKnownAs`, and a verifier can confirm that both DIDs share one verified log.

When a DID migration occurs:

1. The server MUST update the account-level capability object ({{account-capability}}) to reflect the new DID. This changes the Session resource `state`, which clients observe through `sessionState` in method responses ({{RFC8620}} Section 3.4).
2. Objects created after the migration carry the new DID. DID-bearing property values on existing objects are unchanged ({{did-property}}).
3. The server MUST set `previousDid` on the account-level capability object to the DID that was bound before the migration ({{account-capability}}), to help clients that cached the old DID update their records.

## Suspension {#suspension}

A DID suspension is a temporary, reversible deactivation. The DID document may be updated to indicate suspension (e.g., via a status property or by removing verification methods), or the binding may be suspended at the JMAP server level without modifying the DID document.

When a DID is suspended:

1. The server SHOULD reject new object creation for the suspended account with `forbidden`.
2. The server SHOULD prevent the suspended account from authenticating new sessions.
3. Existing objects owned by the suspended account SHOULD remain visible to other users. The `did` property on those objects continues to return the suspended DID.
4. If the consuming specification defines presence or real-time participation (e.g., SceneAvatar, VTCParticipant), the server SHOULD terminate active sessions and remove the account's live presence.

Suspension is reversible. When the DID is reactivated, the server restores normal account functionality. No StateChange is required for suspension or reactivation unless object properties are modified.

## Revocation {#revocation}

A DID revocation is a permanent, irreversible deactivation. The DID document is tombstoned or deleted; the DID will never be valid again.

When a DID is revoked:

1. The server SHOULD terminate all active sessions for the account.
2. The server SHOULD prevent the account from authenticating new sessions.
3. The server SHOULD mark the account's objects as orphaned. The `did` property on those objects continues to return the revoked DID, not `null` ({{did-property}}), so that provenance history is preserved.
4. The server MAY delete the account and its objects after a deployment-defined retention period, subject to legal and compliance requirements.

# Security Considerations {#security}

## Binding Trust

The account-to-DID binding is server-authoritative. A compromised server can bind any DID to any account. Client verification ({{verification}}) mitigates this for `did:key` (where the DID is derived from the public key and can be verified without contacting any server) and for `did:web` (where the DID document is hosted at a well-known URL under the DID controller's domain). It mitigates it further for `did:webvh` ({{did-webvh}}): the SCID binds the DID to its genesis log entry, and the host serving the log cannot add a key without an update key. DID methods that rely on the JMAP server itself for resolution do not benefit from client verification.

Deployments where the DID binding is security-critical SHOULD use DID methods with independent resolution (not dependent on the JMAP server) and SHOULD encourage clients to perform verification.

## DID Correlation Across Servers {#correlation}

A user who presents the same DID to multiple JMAP servers enables those servers (and any observers) to correlate the user's activity across servers. This is inherent to globally unique identifiers and is a feature, not a bug, for use cases like cross-server identity continuity and portable reputation.

Users who require unlinkability across servers SHOULD use different DIDs for different servers. `did:key` supports this naturally: a user can generate a fresh keypair for each server. However, this sacrifices cross-server identity continuity — the user is a different identity on each server.

Users can make informed choices only if they know a deployment's DID correlation properties, which the protocol does not expose. A deployment that requires a single DID across all servers (e.g., an enterprise deployment) should disclose this requirement at enrollment time.

## DID Method Security

The security properties of the `did` field depend entirely on the DID method used. This document does not mandate any specific DID method. Deployments SHOULD evaluate:

- **Key compromise recovery.** `did:key` has no recovery mechanism; if the private key is lost, the identity is lost. `did:web` can rotate keys by updating the hosted DID document. `did:webvh` rotates keys through signed DID log entries; with pre-rotation (`nextKeyHashes`), the keys authorized for the next update are committed in advance.
- **Censorship resistance.** `did:web` depends on DNS and HTTP infrastructure; a domain seizure disables the DID. `did:webvh` has the same dependency, but a DID whose log enables portability can move to another domain and keep its log ({{migration}}). `did:key` has no external dependency.
- **Resolution availability.** `did:web` requires the hosting server to be reachable. `did:webvh` requires a copy of the DID log; the log verifies regardless of where it was obtained. `did:key` is self-resolving.

## Phishing via `did:web` and `did:webvh`

A `did:web` DID is anchored to a domain name. An attacker who registers a visually similar domain (e.g., `did:web:examp1e.com` vs. `did:web:example.com`) can create DIDs that appear legitimate to users who do not inspect the domain carefully. The same applies to the domain component of a `did:webvh` DID: the SCID does not help, because an attacker can create a new log with its own SCID under the look-alike domain. Clients displaying `did:web` or `did:webvh` identities can highlight the domain component and warn users when a domain is visually similar to a domain they have previously trusted.

## Revoked DID Reuse

DID methods that allow identifier reuse after revocation create a risk: a new controller could impersonate the previous controller's historical objects. Deployments SHOULD prefer DID methods where revoked identifiers cannot be reassigned (e.g., `did:key`, where the identifier is derived from the public key and cannot be reused with a different key, or `did:webvh`, where the SCID is derived from the genesis log entry and a new controller of the domain cannot recreate the DID with a different history).

### Server-Generated DID Key Pairs {#server-generated-keys}

When the server generates a `did:key` on behalf of a user and retains the private key (as in the server-generated binding model), the DID provides no cryptographic assurance independent of the server itself. The server can produce signatures indistinguishable from those of the user. {{binding}} requires such servers not to represent the binding as user-verifiable and to disclose custody through `keyCustody`. Client verification ({{verification}}) provides no benefit in this model because the server trivially controls the signing key. Only binding mechanisms where the private key is held exclusively by the user or by an independent identity provider let DID verification provide security guarantees independent of the server.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry established by {{RFC8620}}:

Capability Name:
: `urn:jmap:did`

Intended Use:
: common

Change Controller:
: IETF

Reference:
: This document.

Security and Privacy Considerations:
: See {{security}} of this document.

--- back

# Employer-Managed DID Deployments {#employer-managed}

The account-to-DID binding mechanisms described in {{binding}} are intentionally abstract. One deployment pattern worth noting is employer-managed DIDs, where an organization operates a sidecar process that watches enterprise identity provider events (Okta, Google Workspace, Microsoft Entra, Active Directory, SCIM, SAML) and maintains `did:webvh` DIDs ({{did-webvh}}) on behalf of employees.

The sidecar holds the update keys, so the organization controls each employee's DID log. Witnesses ({{DID-WEBVH}}) are OPTIONAL and a deployment choice. Without witnesses, the organization can present different valid logs to different resolvers (a split view). Witnesses mitigate split view only. Whether or not witnesses are used, a rollback to an older prefix of a log is detected only by a resolver that has already seen a newer `versionId`, or by comparing watchers ({{did-webvh}}).

In this model, the sidecar maps IdP user identifiers to DIDs, and the JMAP server queries the sidecar (or reads the DID documents it publishes) to populate the account-level capability object. Employee offboarding in the IdP triggers DID suspension or revocation via the sidecar, which propagates to the JMAP server's lifecycle handling as defined in {{lifecycle}}. This pattern ensures that identity lifecycle is driven by the organization's existing HR and IT infrastructure rather than requiring a separate identity management system.

# Relationship to JMAP CID {#cid-relationship}

{{JMAP-CID}} provides content-addressed blob references via SHA-256 digests. Combined with the DID binding defined in this document, a JMAP deployment can offer verifiable content-addressed ownership: an asset with content hash H (from JMAP CID) belongs to identity D (from JMAP DID). Neither specification depends on the other, but together they provide the two primitives needed for portable digital assets: location-independent content identity (CID) and location-independent owner identity (DID).

# Relationship to Verifiable Credentials {#vc-relationship}

This document defines identity binding — "this account belongs to this DID." It does not define attribute attestation — "this DID holds these credentials." Verifiable Credentials ({{VC-DATA-MODEL}}) layer naturally on top: once a DID is bound to an account, any VC issued to that DID (achievements, roles, certifications, reputation scores) can be presented to JMAP servers that understand the relevant VC schemas.

A future JMAP capability could define VC presentation and verification mechanisms. This document provides the identity substrate that such a capability would require.

# Acknowledgements

This document builds on the W3C Decentralized Identifiers specification ({{DID-CORE}}) and foundational JMAP work in {{RFC8620}}. The JMAP CID specification ({{JMAP-CID}}) established the pattern of horizontal JMAP capability extensions that this document follows.
