---
title: JMAP Decentralized Identifiers
abbrev: JMAP DID
docname: draft-atwood-jmap-did-00
category: std
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
      org: W3C
    target: https://www.w3.org/TR/did-core/
    date: 2022

informative:
  RFC8621:
  RFC8984:
  JMAP-CONTACTS:
    title: "JMAP for Contacts"
    author:
      fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-contacts-01
    date: 2025
  JMAP-FILENODE:
    title: JMAP File Storage Extension
    author:
      fullname: Bron Gondwana
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-filenode-13
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-filenode/
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-VTC:
    title: JMAP for Video Teleconferencing
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  JMAP-SCENE:
    title: JMAP Scene
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026
  JMAP-CID:
    title: JMAP Blob Content Identifiers
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-cid-00
    date: 2026
  VC-DATA-MODEL:
    title: "Verifiable Credentials Data Model v2.0"
    author:
      org: W3C
    target: https://www.w3.org/TR/vc-data-model-2.0/
    date: 2024

--- abstract

This document defines the `urn:ietf:params:jmap:did` JMAP capability. When a server advertises this capability, every JMAP account is bound to a Decentralized Identifier (DID) as defined in {{DID-CORE}}. The capability adds a `did` property to JMAP data types designated by consuming specifications, enabling globally resolvable, cryptographically verifiable identity across JMAP servers. The account-to-DID binding is server-authoritative; clients MAY independently verify the binding by resolving the DID document.

--- middle

# Introduction {#introduction}

JMAP ({{RFC8620}}) identifies users by server-assigned, server-local `accountId` values. An accountId is meaningful only within the server that issued it. Two JMAP servers have no standard mechanism to determine whether accounts on each server represent the same person, and clients have no way to verify an accountId claim without trusting the server.

Decentralized Identifiers (DIDs) ({{DID-CORE}}) are globally unique, self-sovereign identifiers that resolve to DID documents containing cryptographic key material. A DID is not scoped to any server. A user's DID persists across servers, survives server migration, and can be verified by any party that can resolve the DID document.

This document bridges the two by defining a JMAP capability that binds each account to a DID. When this capability is present:

- Every account has an associated DID, advertised in the account-level capability object.
- JMAP data types designated by consuming specifications gain a server-set `did` property derived from the owning account's DID.
- Clients MAY verify the server's DID attestation independently.
- DID lifecycle events (rotation, suspension, revocation) have defined effects on the JMAP session.

The capability is horizontal infrastructure. It does not define new data types or methods. It extends existing JMAP objects with identity metadata, the same way {{JMAP-CID}} extends blob uploads with integrity metadata. Any JMAP specification — mail ({{RFC8621}}), calendars ({{RFC8984}}), contacts ({{JMAP-CONTACTS}}), file storage ({{JMAP-FILENODE}}), chat ({{JMAP-CHAT}}), video conferencing ({{JMAP-VTC}}), spatial environments ({{JMAP-SCENE}}), or any future JMAP capability — may declare which of its data types carry DID-derived properties.

## Conventions Used in This Document

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Capability {#capability}

A server supporting this extension MUST advertise the `urn:ietf:params:jmap:did` capability in the JMAP Session resource ({{RFC8620}} Section 2).

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:did"]` is a JSON object with the following fields:

`supportedMethods` (String[]):
: The DID methods the server supports for account binding. Each value is a DID method name as defined in {{DID-CORE}} Section 8 (e.g., `"key"`, `"web"`). Servers MUST support at least one method. (Note: the Chat DID companion spec uses the field name `supportedDidMethods` for a semantically equivalent field in its own capability object. Servers advertising both capabilities SHOULD ensure the two lists are consistent.)

`verificationEndpoint` (String|null):
: A URL where clients can submit a DID for server-side verification. `null` if the server does not offer a verification endpoint. The verification protocol is out of scope for this document.

Example:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": {},
    "urn:ietf:params:jmap:did": {
      "supportedMethods": ["key", "web"],
      "verificationEndpoint": null
    }
  }
}
~~~

## Account-Level Capability Object {#account-capability}

The value of `accountCapabilities["urn:ietf:params:jmap:did"]` is a JSON object with the following fields:

`did` (String):
: The DID bound to this account. This is the canonical, globally resolvable identifier for the account holder. The value MUST be a valid DID as defined in {{DID-CORE}} Section 3.1.

`didDocumentUri` (String|null):
: A URI where the DID document can be fetched. For `did:web` this is derivable from the DID itself; for `did:key` the document is constructed from the DID. Servers MAY provide this as a convenience; `null` means the client should resolve the DID document using the standard resolution mechanism for the DID method. When `didDocumentUri` is non-null but unreachable (network error, HTTP 4xx/5xx), clients SHOULD fall back to standard method-specific resolution for the DID. The `didDocumentUri` is a convenience hint, not a binding constraint.

`previousDid` (String|null):
: The DID that was previously bound to this account, retained after rotation until the grace period expires. `null` when no rotation has occurred. See {{rotation}}.

Example:

~~~json
{
  "accounts": {
    "acct-001": {
      "accountCapabilities": {
        "urn:ietf:params:jmap:did": {
          "did": "did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK",
          "didDocumentUri": null,
          "previousDid": null
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

Regardless of mechanism, the server MUST ensure that:

1. Each account is bound to exactly one DID at any given time.
2. The DID is a valid DID as defined in {{DID-CORE}}.
3. The binding is reflected in the account-level capability object ({{account-capability}}).
4. Changes to the binding (rotation, revocation) are propagated as defined in {{lifecycle}}.

# The `did` Property {#did-property}

## Definition

When `urn:ietf:params:jmap:did` is advertised, consuming specifications MAY designate their data types as DID-bearing. A DID-bearing data type gains the following property:

A DID-bearing property is a String|null value, server-set, whose value is a DID derived from the account-to-DID binding defined in {{binding}}. The value is `null` when the account has no DID binding (transitional state during migration) or when the object represents a non-user entity (system-generated objects, anonymous participants).

The property name is not required to be `did`. A consuming specification MAY define multiple DID-bearing properties on a single data type when different fields reference different identities. For example, a delegation record might carry both `delegatorDid` and `delegateeDid`.

Properties that reference the DID of an entity other than the owning account (e.g., the DID of a remote peer on a ChatContact record) are not DID-bearing properties as defined in this section, even though they contain DID values. Such properties are governed by the consuming specification that defines them.

All DID-bearing properties share these semantics:

- **Server-set.** Clients MUST NOT include DID-bearing properties in create or update operations. Servers MUST reject such requests with a SetError of type `invalidProperties`, listing the DID-bearing property name in the `properties` array.
- **Derived.** The value is always derived from the account binding, never stored independently. When the account's DID changes (rotation), all DID-bearing properties derived from that account reflect the new DID.
- **Immutable per binding.** The value changes only when the underlying account-to-DID binding changes, never as a result of object mutation.

## Consuming Specification Obligations

A consuming specification that designates a data type as DID-bearing MUST document:

1. Which data type gains DID-bearing properties, and the property names.
2. Which identity each property represents and from which account binding it is derived.
3. The semantic role of the DID on that type (see {{did-roles}}).
4. Any type-specific behavior when the DID is rotated or revoked.

This document does not unilaterally add DID-bearing properties to any JMAP data type. Each consuming specification opts in by declaring its DID-bearing types.

## Semantic Roles {#did-roles}

A DID on a JMAP object can serve different purposes depending on the data type and the consuming specification's intent. Common roles include:

**Authorship.** The DID identifies who created or sent the object. Enables cross-server sender verification without trusting the originating server. Applicable to email messages, chat messages, calendar event invitations, and file uploads.

**Ownership.** The DID identifies who owns or controls the object. Enables portable ownership claims that survive server migration. Applicable to files, contacts, 3D assets, virtual objects, and any resource with an access control model.

**Participation.** The DID identifies a participant in a multi-party interaction. Enables verifiable participant identity in contexts where trust in the hosting server is insufficient. Applicable to call participants, shared document editors, calendar event attendees, and chat room members.

**Delegation.** The DID identifies a party in a delegated authority relationship. A calendar delegate, a shared mailbox accessor, or an authorized API client may each carry a DID that enables the receiving party to verify the delegation chain. A single object MAY carry multiple DID-bearing properties (e.g., `ownerDid` and `delegateDid`) when the object involves more than one identity.

**Provenance.** The DID establishes a verifiable chain of custody. Combined with content-addressed references (e.g., SHA-256 digests from {{JMAP-CID}}), a DID on an asset or file creates a binding between content identity and owner identity that is verifiable without trusting the hosting server.

**Contact identity.** The DID provides a stable, cross-server identifier for a person or organization in a contact record. Enables contact deduplication across servers and portable block/allow lists.

Consuming specifications SHOULD identify which role or roles apply to each DID-bearing property they define.

## Illustrative Extensions {#extensions}

The following are non-normative illustrations of how JMAP specifications could designate DID-bearing types. The normative declarations belong in the consuming specifications themselves. These examples demonstrate the breadth of applicable roles; they are not exhaustive.

### JMAP Mail ({{RFC8621}})

**Email** — `senderDid` (role: authorship). The DID of the authenticated sender. Complements existing email authentication mechanisms (DKIM, SPF, DMARC) with a sender-controlled identity that is independent of the sending domain. A recipient's client can resolve the DID document and verify the sender's key material without trusting the sending server or relying on DNS-based authentication alone.

**Identity** — `did` (role: ownership). The DID bound to the sending identity. When a user has multiple JMAP identities (e.g., personal and work email addresses), each identity MAY be bound to the same or different DIDs.

### JMAP Calendars ({{RFC8984}})

**CalendarEvent** — `organizerDid` (role: authorship). The DID of the event organizer. Enables cross-server invitation verification: an invitee can verify that the invitation was issued by the claimed organizer without trusting the calendar server that delivered it.

**CalendarEvent** — `delegateDid` (role: delegation). When a calendar event was created by a delegate on behalf of another user, the delegate's DID enables the receiving party to verify the delegation chain.

**Participant** — `did` (role: participation). The DID of a calendar event participant. Enables deduplication of participants across servers (two servers may have different accountIds for the same person, but the DID is the same).

### JMAP Contacts ({{JMAP-CONTACTS}})

**ContactCard** — `did` (role: contact identity). A stable, globally resolvable identifier for the person or organization described by the contact. Enables contact deduplication across servers, portable block/allow lists, and cross-server contact lookup.

### JMAP File Storage ({{JMAP-FILENODE}})

**FileNode** — `ownerDid` (role: ownership/provenance). The DID of the file owner. Combined with the `sha256` field from {{JMAP-CID}}, creates a verifiable content-addressed ownership binding: "the file with content hash H belongs to DID D." Enables portable file provenance that survives migration between storage servers.

### JMAP Chat ({{JMAP-CHAT}})

**Message** — `senderDid` (role: authorship). The DID of the message sender. Enables cross-server sender verification: a recipient on Server B can verify that a federated message from Server A was authored by the claimed DID without trusting Server A.

**ChatContact** — `did` (role: contact identity). Enables portable block lists: blocking a DID rather than a server-local contactId means the block is effective across servers.

### JMAP VTC ({{JMAP-VTC}})

**VTCParticipant** — `did` (role: participation). The DID of the call participant. Enables cryptographic participant identity verification in calls that span server boundaries.

### JMAP Scene ({{JMAP-SCENE}})

**SceneAvatar** — `did` (role: participation). The DID of the user whose avatar this is. Persistent cross-server avatar identity: the same DID appears on your avatar regardless of which JMAP Scene server hosts the region.

**SceneObject** — `ownerDid` (role: ownership/provenance). The DID of the object creator. Combined with the `sha256` field on the object's SceneAsset and {{JMAP-CID}}, enables verifiable content-addressed ownership of 3D assets.

**SceneAsset** — `uploaderDid` (role: provenance). The DID of the asset uploader. Provenance chain for 3D models, textures, and audio resources.

# Client Verification {#verification}

Clients receiving a `did` value on a JMAP object MAY verify the server's attestation independently:

1. Resolve the DID to a DID document using the standard resolution mechanism for the DID method ({{DID-CORE}} Section 7).
2. Extract the authentication verification method from the DID document.
3. Verify that the server has demonstrated control of a key listed in the DID document's `authentication` verification relationship. For `did:web` where the JMAP server's domain matches the DID's domain component, successful TLS-authenticated retrieval of the DID document provides implicit domain binding. For all other cases, the server MUST present a signed assertion (e.g., a JWS or HTTP Message Signature) that validates against the DID document's authentication key.

Verification is OPTIONAL. In a single-operator deployment where the client trusts the server, verification adds overhead with no security benefit. In a federated or zero-trust deployment, verification allows clients to detect a compromised or dishonest server that claims a DID it does not control.

A client that performs verification and detects a mismatch SHOULD treat the `did` value as unverified and SHOULD surface the discrepancy to the user. The client SHOULD NOT silently discard the object or terminate the session; the mismatch may indicate a configuration error rather than an attack.

## Cross-Server Verification

When a client receives objects that originate from a different server (e.g., federated email, cross-server calendar invitations, federated chat messages, shared files), the DID-bearing property enables verification without trusting the originating server:

1. The receiving server includes the `did` as attested by the originating server.
2. The client resolves the DID document independently.
3. If the originating server's attestation matches the DID document, the identity is verified. If not, the client knows the originating server's claim is untrustworthy.

This pattern requires no direct trust relationship between the client and the originating server. The DID document, resolvable by any party, serves as the trust anchor.

# DID Lifecycle {#lifecycle}

## Rotation {#rotation}

A DID rotation occurs when an account's bound DID changes. This may happen due to key compromise recovery, method migration (e.g., `did:key` to `did:web`), or organizational policy.

When a DID rotation occurs:

1. The server MUST update the account-level capability object ({{account-capability}}) to reflect the new DID.
2. The server MUST ensure that all `did` properties on the account's objects return the new DID in subsequent responses.
3. The server SHOULD emit a JMAP StateChange ({{RFC8620}} Section 7.1) for every data type that has objects owned by the rotated account. Clients subscribed to those types will receive the change and can refetch. Servers that compute DID-bearing properties at response time (rather than materializing them on stored objects) SHOULD emit StateChange only for the account's Session resource rather than for every data type, to avoid triggering unnecessary client re-syncs.
4. The server SHOULD include the previous DID in a `previousDid` field on the account-level capability object for a deployment-defined transition period, to help clients that cached the old DID update their records.

`previousDid` (String|null):
: The DID that was previously bound to this account, retained for a deployment-defined transition period after rotation. `null` when no rotation has occurred or the transition period has expired.

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

1. The server MUST terminate all active sessions for the account.
2. The server MUST prevent the account from authenticating new sessions.
3. The server SHOULD mark the account's objects as orphaned. The `did` property on those objects SHOULD return the revoked DID (not `null`) so that provenance history is preserved.
4. The server MAY delete the account and its objects after a deployment-defined retention period, subject to legal and compliance requirements.

# Security Considerations {#security}

## Binding Trust

The account-to-DID binding is server-authoritative. A compromised server can bind any DID to any account. Client verification ({{verification}}) mitigates this for `did:key` (where the DID is derived from the public key and can be verified without contacting any server) and for `did:web` (where the DID document is hosted at a well-known URL under the DID controller's domain). DID methods that rely on the JMAP server itself for resolution do not benefit from client verification.

Deployments where the DID binding is security-critical SHOULD use DID methods with independent resolution (not dependent on the JMAP server) and SHOULD encourage clients to perform verification.

## DID Correlation Across Servers {#correlation}

A user who presents the same DID to multiple JMAP servers enables those servers (and any observers) to correlate the user's activity across servers. This is inherent to globally unique identifiers and is a feature, not a bug, for use cases like cross-server identity continuity and portable reputation.

Users who require unlinkability across servers SHOULD use different DIDs for different servers. `did:key` supports this naturally: a user can generate a fresh keypair for each server. However, this sacrifices cross-server identity continuity — the user is a different identity on each server.

Deployments MUST document their DID correlation properties so users can make informed choices. A deployment that requires a single DID across all servers (e.g., an enterprise deployment) should disclose this requirement at enrollment time.

## DID Method Security

The security properties of the `did` field depend entirely on the DID method used. This document does not mandate any specific DID method. Deployments SHOULD evaluate:

- **Key compromise recovery.** `did:key` has no recovery mechanism; if the private key is lost, the identity is lost. `did:web` can rotate keys by updating the hosted DID document.
- **Censorship resistance.** `did:web` depends on DNS and HTTP infrastructure; a domain seizure disables the DID. `did:key` has no external dependency.
- **Resolution availability.** `did:web` requires the hosting server to be reachable. `did:key` is self-resolving.

## Phishing via `did:web`

A `did:web` DID is anchored to a domain name. An attacker who registers a visually similar domain (e.g., `did:web:examp1e.com` vs. `did:web:example.com`) can create DIDs that appear legitimate to users who do not inspect the domain carefully. Clients displaying `did:web` identities SHOULD highlight the domain component and SHOULD warn users when a domain is visually similar to a domain they have previously trusted.

## Revoked DID Reuse

DID methods that allow identifier reuse after revocation create a risk: a new controller could impersonate the previous controller's historical objects. Deployments SHOULD prefer DID methods where revoked identifiers cannot be reassigned (e.g., `did:key`, where the identifier is derived from the public key and cannot be reused with a different key).

### Server-Generated DID Key Pairs

When the server generates a `did:key` on behalf of a user and retains the private key (as in the server-generated binding model), the DID provides no cryptographic assurance independent of the server itself. The server can produce signatures indistinguishable from those of the user. Client verification (Section 6) provides no benefit in this model because the server trivially controls the signing key. Deployments that require DID verification to provide security guarantees independent of the server MUST use binding mechanisms where the private key is held exclusively by the user or by an independent identity provider.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry established by {{RFC8620}}:

Capability Name:
: `urn:ietf:params:jmap:did`

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

The account-to-DID binding mechanisms described in {{binding}} are intentionally abstract. One deployment pattern worth noting is employer-managed DIDs, where an organization operates a sidecar process that watches enterprise identity provider events (Okta, Google Workspace, Microsoft Entra, Active Directory, SCIM, SAML) and maintains `did:web` documents on behalf of employees.

In this model, the sidecar maps IdP user identifiers to DIDs, and the JMAP server queries the sidecar (or reads the DID documents it publishes) to populate the account-level capability object. Employee offboarding in the IdP triggers DID suspension or revocation via the sidecar, which propagates to the JMAP server's lifecycle handling as defined in {{lifecycle}}. This pattern ensures that identity lifecycle is driven by the organization's existing HR and IT infrastructure rather than requiring a separate identity management system.

# Relationship to JMAP CID {#cid-relationship}

{{JMAP-CID}} provides content-addressed blob references via SHA-256 digests. Combined with the DID binding defined in this document, a JMAP deployment can offer verifiable content-addressed ownership: an asset with content hash H (from JMAP CID) belongs to identity D (from JMAP DID). Neither specification depends on the other, but together they provide the two primitives needed for portable digital assets: location-independent content identity (CID) and location-independent owner identity (DID).

# Relationship to Verifiable Credentials {#vc-relationship}

This document defines identity binding — "this account belongs to this DID." It does not define attribute attestation — "this DID holds these credentials." Verifiable Credentials ({{VC-DATA-MODEL}}) layer naturally on top: once a DID is bound to an account, any VC issued to that DID (achievements, roles, certifications, reputation scores) can be presented to JMAP servers that understand the relevant VC schemas.

A future JMAP capability could define VC presentation and verification mechanisms. This document provides the identity substrate that such a capability would require.

# Acknowledgements

This document builds on the W3C Decentralized Identifiers specification ({{DID-CORE}}) and foundational JMAP work in {{RFC8620}}. The JMAP CID specification ({{JMAP-CID}}) established the pattern of horizontal JMAP capability extensions that this document follows.
