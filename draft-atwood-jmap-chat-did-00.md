---
title: JMAP Chat DID
abbrev: JMAP Chat DID
docname: draft-atwood-jmap-chat-did-00
category: std
stream: independent

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
  RFC8174:
  RFC8620:
  DID-CORE:
    title: "Decentralized Identifiers (DIDs) v1.0"
    author:
      org: W3C
    target: https://www.w3.org/TR/did-core/
    date: 2022
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026

informative:
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  RFC9421:
  DID-WEB:
    title: "did:web Method Specification"
    target: https://w3c-ccg.github.io/did-method-web/
  DID-KEY:
    title: "The did:key Method v0.7"
    target: https://w3c-ccg.github.io/did-method-key/
  DID-PLC:
    title: "did:plc Method Specification"
    target: https://web.plc.directory/spec/v0.1/did-plc

--- abstract

This document defines JMAP Chat DID, a companion specification to JMAP Chat ({{JMAP-CHAT}}) that specifies how Decentralized Identifiers ({{DID-CORE}}) are used as JMAP Chat ChatContact identifiers and as authentication principals between federated peers. The integration is intentionally minimal: it specifies which DID methods MUST be supported, how DID URIs interact with the existing ChatContact opaque-id model in {{JMAP-CHAT}}, and the high-level requirement for DID-based federation authentication. The concrete signature mechanism, resolution caching strategy, DID-document conventions, and DID-provisioning workflow are all deployment-defined. The integration is optional: deployments that do not advertise this capability remain fully functional and interoperable with the rest of the JMAP Chat corpus.

--- middle

# Introduction

{{JMAP-CHAT}} defines the ChatContact data type as the JMAP Chat representation of a participant identity. {{JMAP-CHAT}} treats `ChatContact.id` as an opaque string supplied by the authentication layer and explicitly accommodates Decentralized Identifier ({{DID-CORE}}) URIs as a permissible form of that identifier without prescribing DID resolution, DID-based authentication, or any DID-specific capability. This document fills that gap for deployments that want real DID interop: it defines a JMAP capability that signals DID support, names the DID methods a conforming server must resolve and verify, specifies the optional ChatContact extensions for explicit DID binding, and requires DID-based federation authentication for peers that present DID-form identities.

## Design philosophy

This specification follows three principles consistent with the broader JMAP Chat corpus:

- **Wire contract is minimal.** A single capability URN, an array of supported DID methods, an optional ChatContact field, and one normative authentication requirement. No new methods. No new data types beyond an optional field on ChatContact.
- **Operational and computational policy is deployment-defined.** Concrete signature mechanisms, signed-component lists, clock-skew tolerances, nonce caches, DID-document role conventions, multi-device handling, key-rotation cadences, DID-provisioning workflows, and recovery mechanisms are explicitly out of scope. Implementer guidance for these is in the companion implementer guide.
- **Method set is open.** Three DID methods are mandatory-to-implement; deployments MAY support any other DID method. The capability advertises the actual supported set so peers can negotiate at the wire level. No `did:*` method spec is foreclosed.

## Relationship to JMAP Chat

This document does not redefine ChatContact, does not introduce new JMAP methods, and does not modify the JMAP Chat federation method set ({{JMAP-CHAT-FED}}). It defines:

- A new JMAP capability `urn:ietf:params:jmap:chat:did` and its session-level and account-level capability objects.
- A non-foreclosing list of mandatory-to-implement DID methods.
- An optional `did` field on the ChatContact data type for the pseudonymous case (opaque id with a separately bound DID).
- A normative authentication requirement for federated peers that present DID-form identities, layered onto the {{JMAP-CHAT-FED}} authentication model.
- A reaffirmation of the textual mention form for DID URIs already noted in {{JMAP-CHAT}}.

Implementations of this specification MUST also implement {{JMAP-CHAT}}. A deployment that supports JMAP Chat but does not resolve any DID method MUST NOT advertise the capability defined here.

## Relationship to W3C DID Core

{{DID-CORE}} is the normative source of truth for DID URI syntax, the DID Document data model, and verification-method semantics. This document does not redefine any of these. It does:

- Constrain which DID methods a conforming JMAP Chat server MUST be able to resolve.
- Layer a JMAP-Chat-specific signaling capability on top of {{DID-CORE}}.
- Require that federation authentication verify a signature against a public key bound to the requesting DID per its DID Document; the specific verification-method selection rule is deployment-defined within the bounds of {{DID-CORE}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{JMAP-CHAT}}, and {{DID-CORE}} is used throughout. In particular, "DID" refers to a Decentralized Identifier URI as defined in {{DID-CORE}}, and "DID Document" refers to the JSON-LD or equivalent document obtained by resolving a DID.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:chat:did` capability in the JMAP Session object. This capability is meaningful only when `urn:ietf:params:jmap:chat` is also advertised on the same account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:did"]` at the session level is an empty JSON object `{}`. The session-level capability carries no parameters; per-account policy is exposed in the account-level capability object.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:did"]` is a JSON object with the following field:

`supportedDidMethods` (String[]):
: A non-empty array of DID method names (the portion of a DID URI between the leading `did:` and the second `:`, for example `web`, `key`, `plc`) that this server is able to resolve and verify. The array MUST include at least the three mandatory-to-implement methods specified in {{methods}}. Servers MAY include additional method names corresponding to any other DID methods they support; peers MUST NOT assume the absence of a method name from this array implies that the method is invalid, only that this server cannot resolve it.

Servers MAY include other fields in the account-level capability object as deployment policy hints; clients and peers MUST ignore unknown fields. This specification does not define any other fields in this revision.

# Mandatory DID Methods {#methods}

A server advertising the `urn:ietf:params:jmap:chat:did` capability MUST be able to resolve DIDs of the following methods and verify signatures made by keys bound in their DID Documents:

- **`did:web`** ({{DID-WEB}}) — resolution by HTTPS GET of a JSON DID Document at a well-known path derived from the DID's authority component.
- **`did:key`** ({{DID-KEY}}) — resolution by deterministic local derivation of the DID Document from the public key material encoded in the DID URI itself. No network resolution is performed.
- **`did:plc`** ({{DID-PLC}}) — resolution via the PLC directory protocol, including the operation-log integrity check defined by that protocol.

This list is mandatory-to-implement. A server that cannot resolve any one of these three methods MUST NOT advertise the `urn:ietf:params:jmap:chat:did` capability.

Servers MAY additionally support any other DID method registered in the W3C DID Methods registry or otherwise specified. Support for additional methods is advertised by including the method name in the `supportedDidMethods` account capability array.

This specification does not foreclose any DID method. Future companion specifications MAY raise additional methods to mandatory-to-implement status for specific deployment contexts; this document does not.

# ChatContact Extensions {#contact-extensions}

## DID URIs as ChatContact.id {#did-as-id}

{{JMAP-CHAT}} establishes that `ChatContact.id` is the stable, opaque identity string supplied by the authentication layer, and that DID URIs are a permissible form of that identifier. This specification reaffirms that:

- A server advertising `urn:ietf:params:jmap:chat:did` MAY return ChatContact records whose `id` field is itself a DID URI.
- A server receiving a request that references a ChatContact by a DID-form `id` MUST treat the id as opaque per {{JMAP-CHAT}}; the wire-level semantics of ChatContact are unchanged.
- A ChatContact whose `id` is a DID URI imposes no special handling on JMAP Chat methods beyond what is specified elsewhere in this document for federation authentication and resolution.

## Optional `did` field {#did-field}

ChatContacts MAY carry an optional `did` field to accommodate the pseudonymous case, in which a stable opaque `ChatContact.id` is preserved (for example, to keep MLS group membership stable across DID rotations in a relay deployment) while a separately bound DID provides the cryptographic identity used for federation authentication and key resolution.

`did` (String, optional):
: A DID URI ({{DID-CORE}}) bound to this ChatContact for cryptographic purposes. When present, the DID's verification methods are the authoritative source of public-key material for verifying signatures attributed to this ChatContact. When absent, no DID is bound to this ChatContact; cryptographic identity is derived from the authentication layer as defined in {{JMAP-CHAT}}.

If both `id` and `did` are present AND `id` parses as a DID URI, the two MUST refer to the same DID. A server detecting a mismatch (the parsed DID from `id` differs from the DID in `did`) MUST treat the ChatContact record as invalid and MUST NOT use either DID for authentication.

The `did` field is OPTIONAL. Implementations MAY ignore it entirely if their deployment topology does not require pseudonymous DID binding; in that case, deployments SHOULD use the `id`-as-DID form ({{did-as-id}}) instead.

# Federation Authentication {#federation-auth}

When a federated peer per {{JMAP-CHAT-FED}} authenticates to a local server using a DID-form principal identity, the local server MUST verify that the peer controls the claimed DID by validating a cryptographic signature made by a public key bound in the DID's DID Document.

## Signature requirement

The verification flow at the level mandated by this document is:

1. The receiving server resolves the requesting peer's claimed DID (per {{methods}} for mandatory methods, or per the relevant method specification otherwise).
2. The receiving server selects a verification method from the resolved DID Document appropriate for authenticating the request. The specific verification-method selection rule is deployment-defined within the bounds of {{DID-CORE}}.
3. The receiving server verifies a cryptographic signature attached to the request against the selected verification method's public key.
4. If verification fails, the receiving server MUST reject the request with an authentication error.
5. If verification succeeds, the receiving server MUST treat the request as authenticated for the claimed DID and proceed with the rest of the {{JMAP-CHAT-FED}} authorization model.

This document does NOT specify the precise signature scheme, the on-the-wire location of the signature, the set of signed components, replay-prevention parameters (nonce, timestamp, clock-skew tolerance), or rejection-error code values. These are deployment-defined.

## Recommended concrete realization

The RECOMMENDED concrete realization is RFC 9421 ({{RFC9421}}) HTTP Message Signatures using an Ed25519 signing key selected from the DID Document's `authentication` verification relationship. The companion implementer guide to this specification documents a working component list, replay-prevention parameters, and operational defaults consistent with this realization.

Deployments MAY use a different signature mechanism (for example, JWS over a JSON-formatted challenge payload, or a custom binary scheme) provided that:

- The mechanism cryptographically binds a signature to the request such that replay across requests is infeasible.
- The signing key resolves to the requesting DID's DID Document.
- Receiving servers correctly reject requests that do not authenticate.

Federated peers MUST agree on the signature mechanism out of band. This document does not specify a negotiation protocol for the signature mechanism; deployments expecting heterogeneous federation MUST handle mechanism mismatch as an authentication failure.

# DID Resolution {#resolution}

## Caching

DID resolution caching strategy — including TTL defaults, stale-key handling, refresh triggers, and per-method behavior — is deployment-defined. This document imposes a single normative requirement on caching behavior:

A server MUST refresh its cached DID Document for a given DID before next use if a signature verification against that cached document fails. The rationale is that signature-verification failure is the strongest available signal that the cached key material has been rotated and is no longer current. A server MAY refresh for other reasons; this is the minimum.

## Resolution failure handling

When a server attempts to resolve a DID and resolution fails (transient or permanent), the server MUST treat the DID as unverified and MUST reject any authentication attempt that depends on it. Servers MUST NOT silently downgrade to an unauthenticated mode or proceed with stale cached state when the cache entry has been explicitly invalidated. Servers MAY continue serving cached entries until a refresh succeeds, subject to the deployment's caching policy.

# Mention Textual Form {#mentions}

{{JMAP-CHAT}} notes that a DID URI prefixed with `@` (for example, `@did:web:alice.example`) is a recognized composer-side textual form for a Mention whose `id` is a DID. This document reaffirms that:

- The textual form for a DID-form mention is `@` followed by the DID URI verbatim.
- The wire form is unchanged: the resulting Mention object has `id` set to the DID URI; no separate field carries the textual rendering.
- Parsing the textual form into a candidate id, and resolving that candidate to a known ChatContact, are deployment-defined per {{JMAP-CHAT}}.

No additional bracketed or otherwise-delimited textual form is defined. Clients SHOULD render DID-form mentions in a manner visually distinct from local-handle mentions, but the specific rendering is a client UX decision and is not constrained by this specification.

# Federation Considerations

DID-based identity interacts with {{JMAP-CHAT-FED}} federation in the following ways:

- When a federated peer presents a DID-form principal, the receiving server MUST verify the DID-bound signature ({{federation-auth}}) before accepting the peer's request as authenticated.
- A ChatContact returned by a peer MAY have a DID-form `id`; cross-server references to such ChatContacts use the DID URI verbatim and are resolved by the receiving server per {{methods}}.
- The federation method set defined in {{JMAP-CHAT-FED}} is unchanged. This specification adds the authentication layer; it does not add new federation methods.
- Cross-server DID resolution is independently performed by each server. There is no protocol for one server to delegate resolution to another. A server that cannot resolve a peer's DID MUST treat that peer as unauthenticated.

# Security Considerations {#security}

## Authentication is deployment-defined in detail

The signature mechanism, signed-component list, and replay-prevention parameters are deployment-defined ({{federation-auth}}). Implementations MUST choose parameters that defend against replay attacks (typical defenses: per-request nonces with bounded cache TTL, timestamp checks with bounded skew tolerance) and MUST document their chosen parameters in operator-facing documentation. A weak choice (no nonce, no timestamp, signature over only the request body) is a deployment vulnerability, not a protocol bug.

## DID-document trust

The trust placed in a DID Document is exactly the trust placed in the resolution mechanism for that DID method. A `did:web` DID Document is only as trustworthy as the TLS-protected HTTPS GET that returned it and the DNS resolution that preceded it. A `did:key` DID Document is self-authenticating because it is derived from the public key in the DID URI. A `did:plc` DID Document is only as trustworthy as the PLC directory operator and the operation-log integrity checks the resolver performs.

Implementations MUST document the trust assumptions of each supported DID method to operators. Implementations SHOULD prefer resolution paths that minimize third-party trust (for example, by preferring direct HTTPS resolution for `did:web` over a delegated resolver).

## Key compromise and rotation

When a DID's signing key is compromised, the DID's controller is expected to publish an updated DID Document removing the compromised verification method. Receiving servers learn of the rotation by refreshing their cached DID Document, which is mandated ({{resolution}}) only on signature-verification failure. Between the time of compromise and the time the receiving server's cache refreshes, the compromised key may continue to authenticate fraudulent requests. Implementations SHOULD therefore:

- Use cache TTLs no longer than what their threat model tolerates.
- Implement out-of-band signaling for high-priority rotations where the controller can notify resolving peers proactively (such mechanisms are deployment-defined and not part of this protocol).
- Treat any signature failure as a refresh trigger even if the same DID has previously authenticated successfully.

## did:plc operation log integrity

`did:plc` resolution depends on an operation log served by a PLC directory operator. Implementations MUST verify operation-log integrity per {{DID-PLC}} and MUST NOT trust a DID Document obtained from a PLC directory that has presented an inconsistent or non-monotonic operation log. A `did:plc` resolver that does not perform this integrity check is vulnerable to directory-operator compromise.

## did:web DNS and TLS dependency

`did:web` resolution depends on DNS and TLS. A compromise of the DNS resolver, the TLS infrastructure, or the HTTPS host serving the DID Document is a compromise of any `did:web` DID resolved through that path. Implementations SHOULD pin TLS chains for `did:web` resolutions where operator policy permits and SHOULD treat TLS handshake failures as resolution failures.

## did:key non-rotatability

`did:key` DIDs are deterministically derived from the public key encoded in the DID URI. There is no key-rotation mechanism: rotating the key creates a new DID. A `did:key` whose private key is compromised cannot be recovered or rotated; the DID itself must be retired and a new DID adopted. Deployments using `did:key` as a long-term identity SHOULD plan for key-loss as a likely event and SHOULD provide deployment-level migration paths (typically out of scope for this protocol; see the companion implementer guide).

## Cross-method consistency

A ChatContact MAY carry both an `id` and a `did` field ({{did-field}}). The consistency rule (if `id` parses as a DID URI, both MUST refer to the same DID) defends against impersonation in which an attacker submits a ChatContact with an `id` they control and a `did` belonging to a different principal. Implementations MUST enforce the consistency rule; failure to do so is a security bug.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:did`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## DID Method Registry

This document does not request changes to the W3C DID Methods registry. The three DID methods raised to mandatory-to-implement status here ({{DID-WEB}}, {{DID-KEY}}, {{DID-PLC}}) are registered by their respective method specifications.

--- back

# Design Influences and Non-Normative Notes

This non-normative section documents the design influences and explicit non-decisions.

## Influences

- **Self-sovereign identity practice (`did:key`).** The individual-controlled `did:key` method, with no resolution infrastructure and no rotation, is recognized as the floor of self-sovereign identity. This specification's choice to mandate `did:key` follows the convention established by self-sovereign identity systems and adjacent JMAP-Chat-compatible deployments.
- **Institutional self-sovereign identity (`did:web`).** The DNS-anchored `did:web` method is recognized as the practical bridge between self-sovereign identity primitives and existing institutional identity infrastructure (corporate IdPs, organizational domains). This specification's choice to mandate `did:web` follows that practice.
- **AT Protocol identity (`did:plc`).** Bluesky and the AT Protocol ecosystem use `did:plc` as their default identity method. This specification's choice to mandate `did:plc` is motivated by enabling interoperability with that ecosystem.
- **HTTP Message Signatures ({{RFC9421}}).** The recommended concrete realization of federation authentication is RFC 9421, chosen for its general applicability, signed-component flexibility, and existing implementation base. Alternative mechanisms are accommodated by the deployment-defined latitude in {{federation-auth}}.

## Explicit non-prescriptions

The following design choices were left to deployments rather than prescribed in the wire contract. Implementer guidance is in the companion implementer guide.

- **DID Document verification-method conventions.** Which verification relationships (`authentication`, `assertionMethod`, `keyAgreement`, etc.) are used for which purpose, and which key types are accepted, are deployment-defined. The W3C DID Core data model is the only normative source of truth on DID Document structure.
- **Multi-device handling.** How a single DID represents multiple devices, how device-specific verification methods are named, and how device revocation is signaled are all deployment-defined.
- **DID provisioning workflows.** How DIDs are created, bound to identity-provider principals (SAML, OIDC, SCIM), and lifecycled (onboarding, suspension, offboarding, tombstoning) is deployment-defined.
- **Key recovery mechanisms.** Recovery keys, social recovery, encrypted backup, and other key-loss-mitigation mechanisms are deployment-defined.
- **DID resolution caching parameters.** TTL defaults, stale-key handling beyond the single normative refresh-on-failure requirement, refresh triggers, and refresh signaling are deployment-defined.
- **Federation signature parameters.** Signed-component list, replay-prevention nonce semantics, clock-skew tolerance, and rejection-error code values are deployment-defined.
- **Federated DID resolution delegation.** No protocol for one server to delegate DID resolution to another is defined; each server resolves independently.
- **Trust levels and reputation.** Any tier-based or reputation-based handling of peers by DID is deployment-defined.
- **Hardware attestation.** Whether and how a deployment attests that a verification method's private key is held in trusted hardware is deployment-defined.

# Acknowledgements

The author thanks the authors of {{DID-CORE}} for the DID data model this specification depends on; the authors of {{DID-WEB}}, {{DID-KEY}}, and {{DID-PLC}} for the three method specifications raised to mandatory-to-implement status here; the authors of {{RFC9421}} for the signature framework that informs the recommended concrete authentication realization; and the design teams of self-sovereign identity systems and AT Protocol-adjacent deployments for prior art in DID-based interop that informed this work.
