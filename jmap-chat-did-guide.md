# JMAP Chat DID — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-chat-did-00`.
Covers the operational and policy posture decisions that the DID draft
deliberately leaves open.

Read the draft and W3C DID Core first. This guide does not re-state
normative requirements; it covers what the spec leaves to deployments
and offers patterns and starting points for using Decentralized
Identifiers as JMAP Chat identities.

The DID draft itself defines a minimal wire contract: the capability
URN `urn:ietf:params:jmap:chat:did`, the three mandatory-to-implement
methods (`did:web`, `did:key`, `did:plc`) with an open extensibility
hook, optional ChatContact extensions for explicit DID binding, and
the high-level requirement that federation authentication verify a
DID-bound signature. The main `jmap-chat-implementer-guide.md` covers
identity topics that overlap with core JMAP Chat concerns (the
opaque-id rule, the `@user@host` mention form, ChatContact federation
caching). This guide focuses on what those documents do not cover:
the concrete realization of DID-based identity and authentication,
plus the operational practice of running DIDs as identities at scale.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, deployment policy, or
  default value choice (e.g., "operators SHOULD pin TLS chains for did:web resolution"),
  the keyword reflects implementer best-practice. Deviation does not affect protocol
  interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## How to read this guide

`draft-atwood-jmap-chat-did-00` deliberately specifies a minimal wire
contract — capability URN, mandatory method list, ChatContact
extensions, federation auth requirement — and defers the signature
mechanism, signed-component list, replay-prevention parameters,
DID-document role conventions, multi-device handling, provisioning
workflows, and recovery mechanisms to deployments. DID deployment
models range from "single user with a `did:key`" through "small group
of cooperating organizations each running their own `did:web`" to
"large enterprise issuing `did:web` identities for every employee via
an existing IdP" and on to "AT Protocol-adjacent service issuing
`did:plc` identities at scale". The spec cannot predict which model
fits any one deployment; this guide helps operators think through the
choices.

Each section follows the same shape as the broader
`jmap-chat-implementer-guide.md` and the other companion guides:

1. **What the spec leaves open** — with a citation.
2. **What you must decide.**
3. **Considerations.**
4. **Common patterns** from production deployments.
5. **Recommended starting point** — defensible default, not normative.

Where this guide cites the Agora protocol specification (a separate
project that uses overlapping DID-based identity primitives), Agora is
treated as one reference deployment among possibilities, not as the
canonical answer. Other deployment patterns are equally valid.

---

## 1. Federation auth mechanism — concrete realization

### What the spec leaves open

The DID draft (§Federation Authentication) requires that a federated peer's
DID-based identity be verified by validating a cryptographic signature
made by a key bound in the peer's DID Document. The draft RECOMMENDS
RFC 9421 HTTP Message Signatures with Ed25519 over a key from the
DID Document's `authentication` verification relationship, and
explicitly leaves the on-the-wire signed-component list, the
replay-prevention parameters, the signature header location, and the
rejection-error code values to deployments. Alternative signature
mechanisms (JWS-over-challenge, custom binary schemes) are permitted
provided they bind a signature to a request such that replay is
infeasible and the signing key resolves to the requesting DID.

### What you must decide

- The signature mechanism: RFC 9421 HTTP Message Signatures, JWS,
  or something else.
- The signed components (which request elements the signature covers).
- The header or body location in which the signature is presented.
- The DID Document verification relationship used to select the
  signing key (`authentication`, `assertionMethod`, etc.).
- The replay-prevention parameters: nonce semantics, nonce-cache TTL,
  timestamp tolerance.
- The error code returned for each failure mode (mechanism mismatch,
  signature invalid, DID resolution failed, replay detected, clock
  skew).
- Whether multiple mechanisms are accepted (per-peer choice) or a
  single mechanism is mandatory.
- How peers learn which mechanism a given deployment expects
  (out-of-band documentation, capability advertisement extension,
  trial-and-error rejection signaling).

### Considerations

- *RFC 9421 HTTP Message Signatures* is the recommended starting
  point. It has broad implementation support, a flexible
  signed-component model, and is well-suited to authenticating
  individual HTTP requests in the JMAP federation pattern. Library
  availability is good in Go, JavaScript, Rust, and Python.
- *Signed components* MUST include enough request context to defeat
  replay across different requests. A typical minimum is `@method`,
  `@target-uri`, `@authority`, `content-digest` (for requests with a
  body), the requesting DID, a per-request nonce, and a timestamp.
  Signing too few components opens replay holes; signing too many
  creates brittleness against benign request rewriting by proxies.
- *Nonce-cache TTL and timestamp tolerance* together define the
  replay window. A 300-second timestamp window with a 600-second
  nonce cache is a defensible default for cooperating peers across
  the public Internet; tighten for hostile environments, loosen for
  embedded or asynchronous-relay deployments.
- *Verification-method selection* is in scope for the deployment, not
  the spec. The W3C DID Core `authentication` verification
  relationship is the natural choice for authenticating a request;
  `assertionMethod` is more appropriate for signing standalone
  assertions (e.g., credentials). Pick one and document it.
- *Mechanism heterogeneity* between deployments is an interop risk.
  The spec does not define a negotiation protocol; deployments
  expecting to federate broadly SHOULD support the recommended
  mechanism (RFC 9421 + Ed25519 + `authentication` VM) even if their
  internal deployments also accept others.
- *Library maturity* varies. JOSE/JWS libraries are older and more
  widely deployed than RFC 9421 implementations. Some deployments may
  prefer JWS-over-challenge for that reason alone; that is a
  legitimate trade-off.
- *Error code design* matters for diagnostics. Returning a single
  generic "401 Unauthorized" for all failure modes hides root causes.
  Returning highly specific codes (signature-malformed,
  nonce-replayed, clock-skew, DID-unresolvable, key-not-in-VM-list)
  helps operators triage federation problems quickly.

### Common patterns

- **RFC 9421 + Ed25519 + `authentication` VM** is the configuration
  in the Agora protocol spec (§3.5.3), which is the closest existing
  production-shaped deployment using overlapping primitives. The
  Agora component list — `@method`, `@target-uri`, `@authority`,
  `content-digest`, peer-DID header, timestamp, nonce — is a
  defensible starting point and is reproduced here as one concrete
  example.
- **JWS over a JSON challenge envelope** is the alternative seen in
  early DIDComm and Aries deployments. The peer signs a JSON object
  containing the request fingerprint plus nonce and timestamp; the
  signature accompanies the request as a header or body field. Works
  well for binary or non-HTTP transports; less natural fit for JMAP's
  HTTP request/response pattern.
- **mTLS with the DID document's key as the TLS client cert** has
  been proposed for `did:web` deployments. It works but ties key
  rotation to TLS certificate issuance, which is awkward for
  `did:plc` (where rotation is via PLC operations) and impossible
  for `did:key` (no rotation at all). Not recommended as the only
  mechanism.

### Recommended starting point

Implement RFC 9421 HTTP Message Signatures using an Ed25519 signing
key selected from the requesting DID Document's `authentication`
verification relationship. Sign the following components:

```
"@method"
"@target-uri"
"@authority"
"content-digest"      (POST requests with a body)
"JMAP-Chat-DID"       (requesting peer's DID URI)
"JMAP-Chat-Timestamp" (integer seconds Unix epoch)
"JMAP-Chat-Nonce"     (random 128-bit value, base64url)
```

Reject any request whose timestamp differs from the receiving
server's clock by more than 300 seconds. Maintain a nonce cache with
a 600-second TTL; reject any request whose nonce has been seen in
that window. Refresh the cached DID Document for the requesting DID
when signature verification fails (as required by the draft); reject
the request with a specific error code distinguishing
signature-malformed, nonce-replayed, clock-skew, DID-unresolvable,
and key-not-in-authentication-VM-list failures.

---

## 2. DID resolution and caching strategy

### What the spec leaves open

The DID draft (§DID Resolution) requires that a server refresh its
cached DID Document on signature-verification failure (the assumption
being that the failure is the strongest available signal of key
rotation). Beyond that single normative requirement, every aspect of
the caching strategy is deployment-defined: TTL defaults, per-method
behavior, refresh triggers, stale-key handling, persistent-failure
escalation.

### What you must decide

- Per-method caching policy for `did:web`, `did:key`, `did:plc`, and
  any other supported methods.
- TTL floors and ceilings (the minimum and maximum time a cached DID
  Document remains valid without refresh).
- Whether and how to honor HTTP cache headers for `did:web`
  resolutions.
- Whether and how to track operation-log progress for `did:plc`
  resolutions.
- Behavior on transient resolution failure (retry policy, fallback to
  stale cache, rejection).
- Behavior on persistent resolution failure (after how many failed
  attempts is the DID considered unverified?).
- Cache eviction strategy (size limits, LRU, time-based).
- Per-peer rate limiting on resolution requests (DoS hardening).

### Considerations

- *`did:key` requires no caching.* The DID Document is deterministically
  derived from the public key encoded in the DID URI. Caching is
  computationally pointless. The "resolution" step is a few
  microseconds of multibase decoding; treat it as free.
- *`did:web` caching MAY honor HTTP cache headers* (Cache-Control,
  Expires, ETag). Deployments SHOULD impose floor and ceiling values
  (e.g., minimum 5 minutes regardless of `no-cache`, maximum 24 hours
  regardless of long-lived `max-age`) to bound staleness and
  refresh-load impact. Treat `must-revalidate` as a hint, not as a
  binding constraint.
- *`did:plc` requires operation-log integrity tracking.* The cached
  DID Document is only valid as long as the cached operation-log
  head remains the current head. Cache the head along with the
  document, and refresh the document when the head advances. The PLC
  directory exposes the log; deployments SHOULD poll the log head at
  a low frequency (e.g., once per hour for any given DID) and trigger
  full document refresh on advancement.
- *Refresh-on-failure (mandated by the spec) is the strongest signal,*
  not the only signal. A deployment SHOULD ALSO refresh on TTL
  expiry, on operation-log advancement (for `did:plc`), on observed
  rotation signals via gossip or out-of-band messages, and on
  operator request.
- *Stale-key handling* defines whether a server serves a cached entry
  while a refresh is in flight. Serving stale entries minimizes
  latency spikes; rejecting them during refresh maximizes security.
  A reasonable default is to serve stale entries for up to a few
  seconds after refresh starts, then reject if the refresh has not
  completed.
- *Persistent failure escalation* matters for federation reliability.
  If a peer's DID has been unresolvable for an extended period (e.g.,
  24 hours), the peer's auth status should be downgraded and
  human-operator-level alerts SHOULD fire. Do not silently lock out a
  peer because a DNS hiccup made their `did:web` document
  unreachable for an hour.
- *Cache eviction* matters at scale. A relay federating with
  thousands of peers caches thousands of DID Documents. LRU is the
  obvious default; size-based bounds are necessary for memory-budget
  predictability.
- *Resolution rate-limiting* defends against DoS attacks that force a
  server to resolve many DIDs in a short window. A reasonable
  starting point: cap per-peer DID-resolution requests at 100/minute,
  cap total DID-resolution requests at 10000/minute, and serve
  cached entries from the rate-limited path.

### Common patterns

- **HTTP cache-header-respecting `did:web` resolution** with floor 5
  minutes and ceiling 24 hours. Refresh on signature failure plus a
  background poll near TTL expiry. Production-shaped pattern.
- **Zero-cache `did:key`.** Recompute on every use; the cost is
  negligible.
- **Polling-based `did:plc` operation-log tracking.** Poll the log
  head every 30-60 minutes per active DID; on advancement, fetch
  fresh DID Document and increment the cached head. Active DIDs (those
  seen in recent traffic) are prioritized.
- **Out-of-band rotation hints.** Some deployments (Agora is one
  example) gossip key-rotation announcements between federating
  peers; a server receiving such an announcement refreshes the
  affected DID proactively. This is a deployment-specific extension,
  not a JMAP Chat protocol feature.

### Recommended starting point

For each supported method:

- **`did:web`**: HTTPS GET, honor HTTP cache headers with floor 5
  minutes and ceiling 24 hours. Background refresh at 75% of TTL.
  Hard refresh on signature failure. Pin TLS chain where operator
  policy permits.
- **`did:key`**: no caching. Recompute on each use.
- **`did:plc`**: cache the resolved DID Document plus the operation-
  log head from the PLC directory. Poll the log head every 60 minutes
  per active DID. Refresh the document on log-head advancement and
  on signature failure.
- **All methods**: serve stale entries for up to 3 seconds during an
  in-flight refresh; reject if refresh has not completed. Treat
  persistent (24-hour) resolution failure as a peer downgrade event
  and notify operators. Rate-limit resolution to 100/minute per peer
  and 10000/minute total; cache hits bypass the rate limit.

---

## 3. DID document conventions for interop

### What the spec leaves open

The DID draft (§Federation Authentication, "Explicit non-prescriptions") does
not prescribe DID Document conventions. Which verification
relationships are used for which purpose, which key types are
accepted, how multiple devices are represented in a single DID
Document, and how device revocation is signaled are all
deployment-defined. The W3C DID Core data model is the only normative
source of truth on DID Document structure.

### What you must decide

- Which verification relationships you populate and use:
  `authentication`, `assertionMethod`, `keyAgreement`, `capabilityInvocation`,
  `capabilityDelegation`.
- Which key types are accepted in each verification method:
  Ed25519, P-256, P-384, secp256k1, X25519 (for key agreement only),
  others.
- Multi-device representation: one verification method per device, or
  shared keys with key-derivation, or a hybrid.
- How device-specific verification method IDs are constructed
  (fingerprint suffix, sequential numbering, device-label).
- Service endpoints (DID Document `service` entries) and which types
  your deployment recognizes.
- Whether your DID Documents carry deployment-specific metadata in
  the `service` array or via DID Document extensions.

### Considerations

- *The `authentication` verification relationship* is the natural fit
  for keys used to authenticate JMAP Chat federation requests. The
  recommended starting point in §1 selects the signing key from this
  relationship; deployments diverging from this should document the
  alternative in operator-facing docs.
- *The `keyAgreement` verification relationship* is the natural fit
  for X25519 keys used in MLS KeyPackage init keys or HPKE-style
  payload encryption. JMAP Chat itself does not currently require
  `keyAgreement` keys, but deployments that layer end-to-end
  encryption on top of JMAP Chat will need them. Populating
  `keyAgreement` proactively is forward-compatible.
- *Key type choice* affects interoperability and security posture.
  Ed25519 is the strongest default for signing: small keys, fast
  verification, no curve-choice surprises. P-256 / ECDSA is a common
  alternative when hardware constraints favor it (e.g., HSMs without
  Ed25519 support).
- *Multi-device handling* is a real operational concern. A single
  user typically has several devices (laptop, phone, tablet). Each
  device generates its own keypair inside its secure hardware
  boundary. The DID Document needs to enumerate all active device
  keys; revocation removes a key from the document.
- *Device-specific verification method IDs* SHOULD be predictable but
  collision-free. A common pattern is `<did>#device-<fingerprint>`
  where `<fingerprint>` is a hex prefix of the SHA-256 of the public
  key. Avoid embedding device labels or platform names in the ID
  string itself — those are metadata, not identity.
- *Service endpoints* in the DID Document MAY advertise the
  preferred JMAP Chat server for this DID. A `service` entry of type
  `JMAPChatServer` with `serviceEndpoint` pointing at the user's
  JMAP server lets remote peers discover the routing target without
  out-of-band coordination. This is a deployment convention, not a
  spec requirement.
- *Document extension fields* outside of the W3C DID Core schema
  SHOULD be avoided in interop-facing DID Documents. Use service
  endpoints with custom types if deployment-specific metadata is
  needed; reserve DID Document schema extension for cases where the
  metadata must be inside the signed core.

### Common patterns

- **Two-role split: `authentication` (Ed25519) + `keyAgreement` (X25519)**
  is the convention in the Agora protocol spec (§2.1) and in
  Hyperledger Aries / DIDComm deployments. Authentication keys sign
  protocol messages; key-agreement keys handle encryption. Clean
  separation, easy to reason about, forward-compatible with E2EE.
- **One-key-per-device with `#device-<fingerprint>` suffix.** Each
  device's keypair appears as a single verification method in the
  DID Document. Multiple devices = multiple entries. Revocation
  removes the entry. Used in Agora §2.5.4 and in many self-sovereign
  identity systems.
- **Shared key with derived child keys.** Less common but used in
  some hierarchical-deterministic key systems. One root key, child
  keys derived per device, only child public keys appear in the DID
  Document. Compact and well-suited to backup but creates a single
  point of compromise.
- **Service endpoint advertising the JMAP server.** A
  `service` entry of type `JMAPChatServer` with `serviceEndpoint`
  pointing to the JMAP Session URL. Used (in Agora) as `AgoraRelay`
  type; the same pattern fits JMAP Chat directly.

### Recommended starting point

Populate two verification relationships per device:

- `authentication` with an Ed25519 verification method (signing key
  for protocol messages).
- `keyAgreement` with an X25519 verification method (for future
  E2EE integration).

Use the verification method ID pattern
`<did>#device-<fingerprint>` where `<fingerprint>` is the first 16
hex characters of the SHA-256 of the public key. Multiple devices
appear as separate pairs of verification methods sharing the device
fingerprint suffix. Revocation removes both methods for the
revoked device.

Advertise the JMAP Session URL via a `service` entry:

```json
{
  "id": "<did>#jmap",
  "type": "JMAPChatServer",
  "serviceEndpoint": "https://jmap.example.com/.well-known/jmap"
}
```

Document the chosen conventions in operator-facing documentation so
peers can implement against them.

---

## 4. Provisioning workflows

### What the spec leaves open

The DID draft ("Explicit non-prescriptions") explicitly defers DID
provisioning workflows — how DIDs are created, bound to
identity-provider principals (SAML, OIDC, SCIM), and lifecycled
(onboarding, suspension, offboarding, tombstoning) — to deployments.
The wire contract is unchanged by the choice of provisioning
mechanism: a provisioned DID Document looks the same on the wire
regardless of how it was constructed.

### What you must decide

- The provisioning model for each supported DID method:
  - For `did:web`, who hosts the DID Document and how it is
    populated.
  - For `did:key`, how users generate and protect their private keys.
  - For `did:plc`, which PLC directory the DIDs live in and how
    PLC operations are signed.
- The lifecycle model: stub → active → suspended → tombstoned, or a
  simplified version.
- The identity-provider integration: SAML, OIDC, SCIM, webhooks, AD
  sync, HR system feeds, or no IdP at all.
- The mapping from IdP principal identifiers (UPNs, email, employee
  IDs) to DID URIs.
- The first-device enrollment flow and any subsequent-device flows.
- The revocation flows: user-initiated, organization-initiated, or
  both.
- The audit logging level for provisioning operations.

### Considerations

- *Three deployment archetypes dominate.* (a) Individual: a single
  user with a `did:key` they hold themselves, no provisioning service.
  (b) Self-hosted organization: a small group with a `did:web`
  domain, DID Documents managed by hand or by a lightweight script.
  (c) Enterprise: organization with thousands of users, DID Documents
  provisioned automatically by a sidecar service tied to the
  corporate IdP. Each archetype has different provisioning needs.
- *`did:web` provisioning at enterprise scale* is the most complex
  case. The DID Document is hosted at a well-known HTTPS path; the
  enterprise's IdP drives lifecycle events (account create, update,
  suspend, terminate); a sidecar service translates IdP events into
  DID Document changes. This is the model Agora's universal-IdP-DID
  spec elaborates in detail across SAML, OIDC, SCIM, webhooks, and
  AD integration.
- *Trust separation is crucial in enterprise deployments.* The
  organization owns the DID Document (it controls the domain, the
  HTTPS endpoint, and therefore revocation authority). The user
  agent owns the device private keys (generated in Secure Enclave /
  StrongBox / TPM, never leaving the hardware boundary). The
  organization can revoke an identity but cannot impersonate a user
  it has provisioned. This mirrors how S/MIME and WebAuthn work in
  enterprise contexts.
- *`did:key` provisioning is trivial in concept but error-prone in
  practice.* The user generates a keypair; the DID is derived from
  the public key. The hard part is making sure the private key
  survives device loss without being so easy to extract that it can
  be stolen. See §5 (recovery mechanisms).
- *`did:plc` provisioning is intermediate.* The DID is created by
  publishing an operation to the PLC directory; rotation is via
  subsequent operations. The PLC directory operator is a trust
  dependency. AT Protocol clients typically handle this for the
  user; bespoke JMAP Chat deployments using `did:plc` will need to
  integrate with PLC tooling (or run their own PLC instance if the
  protocol permits, which is operator-specific policy).
- *IdP-to-DID mapping* needs to be stable across the user's tenure.
  Use the IdP's stable internal identifier (not email, which changes
  on name change) and document the mapping. Never reuse a DID
  localIdentifier for a different person after the original departs.
- *Lifecycle modeling* — STUB → ACTIVE → SUSPENDED → TOMBSTONED is the
  Agora model. STUB = provisioned but no device keys yet; ACTIVE =
  device keys registered, normal use; SUSPENDED = device keys
  blanked but document retained for audit; TOMBSTONED = document
  retained (URL still resolves) but verification methods empty,
  used for archival reference integrity. Simpler deployments may
  collapse this to ACTIVE / REVOKED.

### Common patterns

- **Manual `did:web` for small teams.** A trusted admin maintains the
  DID Documents by hand (or via a simple JSON editor + git workflow).
  Works for groups of 10-100; breaks down beyond that.
- **Sidecar-driven `did:web` for enterprises.** The Agora
  universal-idp-did-spec is the canonical example: a sidecar service
  consumes IdP lifecycle events (SAML assertions, OIDC tokens, SCIM
  push, webhooks from Okta / JumpCloud / Ping / OneLogin, or polling
  from AD LDAP) and translates them into DID Document state changes.
  The model handles four IdP categories (webhook-capable, SCIM-only,
  SAML-only / legacy, on-prem AD); see Agora's spec for working
  patterns.
- **Self-managed `did:key` for individuals.** The user agent
  generates the keypair in secure hardware; the DID is the public
  key. No service to provision. Recovery is the only operational
  concern (see §5).
- **PLC-tooling-mediated `did:plc` for AT Protocol-adjacent
  deployments.** The PLC directory is the source of truth;
  provisioning is via PLC operations signed by a rotation key. AT
  Protocol's PDS (Personal Data Server) typically handles this.

### Recommended starting point

Choose the model that fits your archetype:

- **Individual / small group**: `did:key` for individuals,
  hand-maintained `did:web` for the group. Document the lifecycle in
  prose; no machinery needed. Backup and recovery (§5) are the only
  operational concerns.
- **Enterprise (>100 users)**: build or adopt a sidecar service
  driven by your IdP. Use Agora's universal-idp-did-spec as the
  reference design — even if not adopting Agora itself, the SAML /
  OIDC / SCIM / webhook integration patterns and the STUB → ACTIVE →
  SUSPENDED → TOMBSTONED lifecycle model are reusable. Enforce trust
  separation: the sidecar owns the DID Document, the user agents own
  device keys in secure hardware, and never the twain shall meet.
- **AT Protocol-adjacent**: use PLC directory tooling. Treat the
  PDS-equivalent component as the provisioning surface.

In all cases, log provisioning operations (creation, key registration,
revocation, suspension, tombstoning) to a deployment audit trail with
operator identity and timestamp. Do not log key material.

---

## 5. Recovery mechanisms

### What the spec leaves open

The DID draft ("Explicit non-prescriptions") explicitly defers key
recovery mechanisms to deployments. Recovery keys, social recovery,
encrypted backup, and any other key-loss-mitigation approach are
deployment choices; the JMAP Chat DID wire contract is unchanged by
which mechanism is in use.

### What you must decide

- Whether your deployment offers a recovery mechanism at all.
- If yes, which mechanism(s): recovery key, social recovery,
  encrypted backup, or a combination.
- The user UX: when is recovery setup prompted, how often is it
  validated, what happens at recovery time?
- The trust relationships: who holds shards / who holds backup keys /
  who can initiate organization-initiated recovery?
- The conflict-resolution policy when multiple recovery paths are
  triggered.

### Considerations

- *`did:key` is the hard case.* A `did:key` whose private key is lost
  cannot be recovered protocol-internally; the DID is gone and a new
  one must be issued. Deployments using `did:key` as long-term
  identity MUST plan for key loss as a likely event. The recovery
  mechanism is a deployment-layer construct (separate-key rotation,
  social recovery to a new DID, organization-initiated reissue) that
  conceptually retires the old DID and issues a new one with the
  user's history migrated.
- *`did:web` has organization-initiated recovery built in.* The
  organization controls the DID Document and can register new device
  keys for a user who has lost all theirs. This is the normal
  recovery path in enterprise deployments. Document the audit
  requirements for this privileged operation.
- *`did:plc` supports recovery via PLC rotation operations.* The DID
  controller can rotate the signing key by publishing a new PLC
  operation signed by a separately-held rotation key. The rotation
  key is the recovery primitive. Lose the rotation key too and the
  DID is permanently locked.
- *Recovery key (Agora §2.4.1 model).* A dedicated, never-used key
  held offline (paper backup, air-gapped device, HSM). Its only
  purpose is to sign a recovery assertion that adds a new device
  key. Strong primitive; depends on the user not losing the recovery
  key, which is a real failure mode.
- *Social recovery (Agora §2.4.2 model).* The user picks N guardians;
  M-of-N can collectively authorize recovery. The user's recovery
  secret is split via Shamir's Secret Sharing and each shard is
  encrypted to a guardian. At recovery time, M guardians decrypt and
  contribute their shards; the user reconstructs the secret and
  uses it to install a new device key. Strong primitive for
  socially-connected users; useless for isolated users.
- *Encrypted backup (Agora §2.4.3 model).* The client exports an
  encrypted copy of device keys (or MLS state, for non-extractable
  keys) to user-chosen storage. Argon2id key derivation from a
  user-chosen passphrase. Strong primitive for users who can
  remember a passphrase; useless if the passphrase is lost.
- *Multiple mechanisms reduce single-point-of-failure risk* but
  multiply the social-engineering attack surface. The
  conflict-resolution policy when multiple paths produce competing
  recovery assertions matters: typically, the earlier valid
  assertion wins, with relays rejecting subsequent assertions
  within a deduplication window.
- *Organization-initiated recovery is privileged and must be
  audited.* The organization can register new device keys on behalf
  of a user who has lost all theirs. This is necessary for
  enterprise viability but also creates an impersonation path if
  abused. Audit every invocation; require multi-admin approval for
  high-privilege accounts; document the policy.

### Common patterns

- **Recovery key + encrypted backup combined.** The user generates a
  recovery key at account creation, prints it on paper, and also
  exports an encrypted backup of device state. The backup recovers
  state on a known passphrase; the recovery key recovers identity
  if the device is gone. Belt-and-suspenders; the Agora reference
  model.
- **Social recovery for security-conscious users.** Setup is more
  involved (designate N guardians, distribute shards); recovery
  requires coordinating M guardians. Best fit for users in tight
  social or organizational groups.
- **Organization-initiated recovery in enterprise deployments.** IT
  helpdesk issues a new device key registration via an admin
  interface; the user authenticates with their IdP credentials
  (plus MFA) and the new key is registered. Standard enterprise
  recovery flow.
- **No recovery (acceptance of loss) for transient identities.**
  Some `did:key` use cases — ephemeral session identities,
  short-lived bot accounts — don't need recovery. Loss is expected
  and the DID is simply abandoned. Valid for narrow contexts.

### Recommended starting point

Match the recovery mechanism to the deployment archetype:

- **Individual using `did:key`**: encrypted backup with a strong
  passphrase, plus a recovery key printed on paper. Prompt at
  account creation; verify the recovery key annually.
- **Self-hosted small group**: the group's admin SHOULD have a
  privileged recovery path (analogous to enterprise IT helpdesk)
  for users who lose all their keys.
- **Enterprise**: organization-initiated recovery via IT helpdesk
  is the primary path. Audit every invocation. Offer social
  recovery and encrypted backup as secondary user-initiated paths
  for users who prefer to retain control.
- **AT Protocol-adjacent**: the PLC rotation key is the primary
  recovery primitive. Educate users on rotation-key custody and
  prompt for backup at account creation.

Document the chosen mechanisms and their failure modes in
user-facing documentation. Treat recovery setup as an onboarding
step, not an optional afterthought.

---

## 6. Mention textual form rendering

### What the spec leaves open

The DID draft (§Mentions) reaffirms the core draft's convention
that a DID-form mention is the textual sequence `@` followed by the
DID URI verbatim (e.g., `@did:web:alice.example`). The draft does
not specify how clients render these mentions, how they truncate
visually-long DIDs, or how they cache the DID-to-display-name
resolution.

### What you must decide

- How DID-form mentions render in the chat UI.
- Whether and how to truncate long DID URIs in the rendered form.
- Whether to display a resolved display name alongside or instead of
  the raw DID.
- Cache strategy for DID-to-display-name resolution.
- How to render mentions for DIDs the client cannot resolve.
- Composer UX: how does the user enter a DID-form mention?

### Considerations

- *Raw DIDs are long and ugly.* `did:plc:abc123def456...` is 32+
  characters of opaque text. Showing it verbatim in chat is a poor
  UX. Resolution to a display name (from the DID Document, the
  ChatContact, or a local address book) is essential for usability.
- *Display-name resolution introduces a trust question.* The display
  name shown to the user is the client's interpretation, not the
  DID itself. A malicious DID Document could claim a display name
  designed to impersonate another principal. Clients SHOULD show
  the underlying DID on hover or click, and SHOULD warn the user
  when a DID's display name has changed since last seen.
- *Truncation balances readability against ambiguity.* Showing
  `did:plc:abc1...` creates collision risk in dense user lists;
  showing the full DID destroys readability. A reasonable
  compromise: show the first 8 and last 4 characters of the
  method-specific portion in dense contexts (`did:plc:abc123de...f456`),
  show full DID on hover.
- *Resolution caching* SHOULD honor the DID Document's own freshness
  (per §2); the display-name-to-DID cache is downstream of that. A
  display name shown for a stale DID Document is a security smell;
  a stale display name from a current document is not.
- *Unresolved DIDs* (network failure, deleted DID, invalid format)
  SHOULD render with a clear "unverified" indicator. Do not silently
  show a placeholder name; the user needs to know the resolution
  failed.
- *Composer UX* is a real challenge. Typing 60 characters of
  `did:plc:...` is impractical. Common patterns: autocompletion from
  the client's contact list, scanning a QR code that encodes the
  DID, pasting the DID from a profile URL.
- *The textual mention form is shared with the core draft* (`@did:method:value`,
  no braces). Do not introduce a different textual form here, even
  in client UI; that would create inconsistency across deployments.

### Common patterns

- **Resolved display name with DID on hover.** The chat surface
  shows the resolved name (`@alice`); hovering or long-pressing
  reveals the underlying DID. Common in clients that have a strong
  contact-list model.
- **DID prefix preview with full DID on click.** The chat surface
  shows a truncated DID (`@did:plc:abc1...f456`); clicking opens a
  profile pane with the full DID and verification info. Common in
  clients with a DID-first UX.
- **Hybrid: name plus partial DID.** Show `@alice (did:plc:abc1...)`
  in dense contexts where ambiguity would matter. Used when two
  contacts have the same display name.
- **Verification-state indicators.** Render mentions with a small
  icon indicating whether the DID is verified (signature OK,
  DID Document fresh), stale (cache TTL exceeded but not yet
  refreshed), or unresolved (resolution failed). UX patterns
  similar to TLS lock-icon conventions.

### Recommended starting point

Render DID-form mentions with:

- The resolved display name from the ChatContact record (if
  available) as the primary text.
- A small verification-state icon adjacent to the name (verified /
  stale / unresolved).
- The underlying DID accessible via hover, long-press, or click.

When no display name is available, render a truncated form of the
DID: `did:method:` prefix plus the first 8 and last 4 characters of
the method-specific portion (e.g., `did:plc:abc123de...f456`).

For unresolved DIDs, render with an explicit "unverified" indicator
and the truncated form; do not invent a placeholder display name.

In the composer, support autocompletion from the client's contact
list keyed on display name, DID prefix, and any user-set alias.
Allow pasting full DIDs into the composer for first-contact cases.

Cache the DID-to-display-name resolution alongside the DID Document
cache (§2); expire the display-name cache whenever the underlying
DID Document is refreshed.

---

## 7. Cross-method consistency

### What the spec leaves open

The DID draft (§ChatContact Extensions) establishes a consistency rule:
if a ChatContact carries both an `id` field and a `did` field, and
the `id` parses as a DID URI, the two MUST refer to the same DID.
Beyond that single rule, the spec is silent on how a deployment
handles a user with multiple DIDs across methods, how related DIDs
are linked, and what UX surfaces these relationships.

### What you must decide

- Whether your deployment recognizes a single user with multiple
  DIDs (e.g., institutional `did:web` for work + personal `did:key`
  for off-hours).
- How related DIDs are linked at the data-model level.
- Whether ChatContacts are per-DID or per-person (with multiple DIDs
  attached).
- UX patterns for users with multiple DIDs.
- Whether and how cross-method consistency claims (DID A and DID B
  are the same person) are verified.

### Considerations

- *Multi-DID users are real.* A typical professional may have a
  `did:web:acme.com:users:alice` for work, a personal
  `did:key:z6Mk...`, and a `did:plc:...` for AT Protocol presence.
  Each DID has its own key material, its own resolution path, and
  its own trust model. The user perceives one identity; the
  protocol sees three.
- *One ChatContact per DID is the simplest model.* Each DID gets a
  ChatContact row. The deployment tracks them as separate
  participants. Linking happens at the UI layer (the client's
  address book groups related DIDs under one display entry). Wire
  contract is unchanged; ChatContact federation is unchanged.
- *One ChatContact with multiple DIDs is more complex.* The
  `ChatContact.id` is opaque (per the core draft); a separate field
  could list additional DIDs. This breaks the consistency rule of
  the DID draft if not carefully designed. The DID draft does NOT
  introduce such a field; deployments wanting one would need to
  extend the data model out-of-band or in a future spec revision.
- *Cross-method consistency claims need verification.* If Alice
  claims `did:web:acme.com:users:alice` and `did:key:z6MkAlice...`
  are both hers, the claim needs cryptographic proof: a signed
  assertion from one DID asserting the other. The W3C Verifiable
  Credentials data model is the natural fit; the JMAP Chat DID
  draft does not specify a VC format, but deployments may adopt
  one.
- *Verification at the client level is acceptable.* Even without
  spec-level cross-method consistency primitives, clients can
  maintain a local address book that links related DIDs based on
  user input. The user vouches for the relationship; the client
  surfaces it; no protocol-level verification is performed. This
  works for most deployment contexts.
- *Pseudonymous-id case (DID draft, §Optional `did` Field).* The optional
  `did` field on a ChatContact accommodates an opaque `id` plus a
  bound `did` for crypto. This is NOT a multi-DID case; it is a
  single-DID-with-opaque-handle case. Multi-DID handling is
  separate.

### Common patterns

- **Per-DID ChatContacts with client-side address-book linking.**
  Each DID is a separate ChatContact; the client's local address
  book maintains a "this is the same person" mapping based on user
  input. Most common pattern; no protocol changes needed.
- **Single ChatContact with bound DID for pseudonymity.** Uses the
  optional `did` field per the spec. Stable opaque `id`, bound
  DID for crypto. Specifically for the case where the user wants
  identity stability under DID rotation (e.g., MLS group
  membership across `did:plc` rotations).
- **Verifiable Credential-based cross-method linking.** Out of
  scope for the current spec; some deployments may adopt a VC
  format where DID A signs a credential asserting "I am the same
  person as DID B", verifiable on inspection. Useful for
  high-assurance cross-method identity binding.

### Recommended starting point

Treat each DID as a separate ChatContact at the protocol level.
Implement client-side address-book linking for users who want to
group related DIDs under a single display entry. Surface the
underlying DIDs (with verification indicators) when the user
inspects the linked entry; do not hide the multi-DID structure.

Do not extend the data model with multi-DID fields without a
companion spec defining the consistency and verification rules.
Wait for a future spec revision or adopt Verifiable Credentials
out-of-band for cross-method binding.

Use the optional `did` field on ChatContact only for the
pseudonymous-handle case explicitly described in the draft. Do not
overload it with multi-DID semantics; that would break the
consistency rule.

---

## 8. Security and privacy considerations expansion

### What the spec leaves open

The DID draft (§Security Considerations) lists security considerations at a
spec-level granularity: DID-document trust, key compromise and
rotation, did:plc operation-log integrity, did:web DNS/TLS
dependency, did:key non-rotatability, cross-method consistency. The
draft does not elaborate the operational practice that makes these
security postures real in a deployment. This guide expands on each.

### What you must decide

- TLS pinning policy for `did:web` resolution.
- PLC directory operator trust posture for `did:plc` deployments.
- Key-rotation cadence and triggering policy.
- Threat model for the deployment: insider threat, network attacker,
  hostile peer.
- Audit logging level for identity events (resolution failures, key
  rotations, signature failures, revocations).
- User communication patterns for security events (key compromise
  notifications, rotation prompts).
- Hardware-key-storage policy for high-assurance accounts.

### Considerations

- *`did:web` TLS pinning* is a strong defense against the DNS-and-TLS
  trust dependency the spec calls out. Pin the certificate chain (or
  the SPKI) per resolved DID; warn or fail closed when the pinned
  chain changes. Pinning conflicts with normal CA rotation; deployments
  SHOULD support a managed pinning workflow with a grace window for
  legitimate rotation. Best applied to high-assurance peers (the
  organization's own infrastructure, regulated counterparties);
  excessive for casual federation.
- *PLC directory operator trust* is the structural dependency of
  `did:plc`. The PLC directory operator can in principle present an
  inconsistent operation log to one resolver and a different one to
  another. The integrity check defined in DID-PLC defends against
  retroactive log rewriting but not against present-tense divergence.
  Mitigations: cross-reference the operation log against multiple
  resolvers; gossip log heads between cooperating peers; treat the
  PLC operator as a trust anchor and select operators accordingly.
- *Key-rotation cadence* balances security against operational burden.
  An annual rotation cadence is a defensible baseline for routine
  hygiene; faster cadences (quarterly, monthly) are appropriate for
  high-risk environments. Rotation MUST be triggered immediately on
  suspected or confirmed compromise. For `did:key`, rotation creates
  a new DID; for `did:web`, rotation updates the DID Document; for
  `did:plc`, rotation publishes a PLC operation.
- *Threat models vary.* A self-sovereign individual using `did:key`
  defends primarily against device theft and account loss; an
  enterprise using `did:web` defends primarily against insider
  threat and infrastructure compromise; an AT Protocol-adjacent
  deployment using `did:plc` defends primarily against PLC operator
  trust and key rotation. Document the deployment's threat model
  explicitly; choose mechanisms appropriate to it.
- *Audit logging* of identity events MUST capture sufficient detail
  for incident response. Include: DID resolution failures (when, for
  which DID, what error); signature-verification failures (which
  DID, which key, which request); key rotations (when, initiated by
  whom, new fingerprints); revocations (when, by whom, which keys).
  MUST NOT log private key material; SHOULD NOT log signed-message
  contents (which may include sensitive chat data).
- *User communication* for security events matters. When a peer's
  DID Document changes substantively (verification methods removed,
  recovery flags toggled), inform the user. When a key is rotated
  for routine reasons, that's noise; when a key is rotated outside a
  scheduled cadence, that's a signal. Calibrate notifications to
  avoid alert fatigue.
- *Hardware key storage* (Secure Enclave, StrongBox, TPM, HSM) is the
  strongest defense against private-key extraction. Hardware-stored
  keys cannot be backed up via encrypted-backup mechanisms (§5);
  recovery from hardware-key loss requires alternative paths
  (recovery key, social recovery, organization-initiated recovery).
  Deployments enforcing hardware key storage MUST document the
  recovery interaction.

### Common patterns

- **TLS pinning per peer for high-assurance federation.** The
  deployment maintains a manifest of trusted peer DIDs and their
  current certificate fingerprints; resolutions fail closed on
  mismatch. Used in inter-organizational federation between
  regulated entities. Cost: manual maintenance of the pin manifest.
- **Multi-resolver cross-checking for `did:plc`.** Resolve via the
  primary PLC directory and verify the result against a secondary
  resolver or a peer's cached state. Detects directory divergence.
  Cost: additional resolution latency.
- **Annual rotation as routine hygiene.** Schedule key rotations
  annually for all active DIDs; treat off-cadence rotations as
  incidents. Cost: scheduling and communication; benefit: rotation
  becomes routine rather than emergency.
- **Audit dashboards keyed on DID.** Operations teams maintain a
  per-DID view of resolution history, signature-verification
  successes and failures, key rotations, and access patterns.
  Helpful for incident response; intrusive privacy-wise. Restrict
  access.

### Recommended starting point

For each supported method:

- **`did:web`**: pin TLS chains for peers explicitly enrolled as
  high-assurance; do not pin for casual federation. Maintain a peer
  manifest with current pinned chains; refresh on legitimate
  rotation events through a managed workflow.
- **`did:plc`**: cross-check resolution against a secondary
  resolver for peers explicitly enrolled as high-assurance. Refresh
  operation-log head every 60 minutes per active DID. Treat
  operation-log inconsistencies as a peer downgrade event.
- **`did:key`**: enforce strong key generation (hardware-backed
  where possible); do not permit weak-RNG-generated keys. Recovery
  is the dominant security concern (§5).

Schedule annual rotations as routine hygiene. Treat off-cadence
rotations and any signature-verification failures as audit events;
escalate persistent or anomalous events to operator review.

Log identity events to an audit trail with the granularity needed
for incident response. Do not log key material. Do not log message
contents.

Communicate substantive DID Document changes to affected users.
Calibrate to avoid alert fatigue; differentiate routine rotations
from off-cadence events.

Enforce hardware key storage for high-assurance accounts and
document the recovery interaction.

---

## Cross-references

### To the JMAP Chat DID draft

| Topic | Draft Section |
|---|---|
| Capability URN and account-level `supportedDidMethods` | `draft-atwood-jmap-chat-did-00.md` §Capability |
| Mandatory DID methods | `draft-atwood-jmap-chat-did-00.md` §Methods |
| ChatContact `id`-as-DID and optional `did` field | `draft-atwood-jmap-chat-did-00.md` §ChatContact Extensions |
| Federation authentication requirement | `draft-atwood-jmap-chat-did-00.md` §Federation Authentication |
| DID resolution refresh-on-failure rule | `draft-atwood-jmap-chat-did-00.md` §DID Resolution |
| Mention textual form | `draft-atwood-jmap-chat-did-00.md` §Mentions |
| Security considerations | `draft-atwood-jmap-chat-did-00.md` §Security Considerations |

### To other JMAP Chat documents

| Topic | File |
|---|---|
| Core opaque-id rule (`ChatContact.id` is opaque) | `draft-atwood-jmap-chat-00.md` |
| `@user@host` mention form | `draft-atwood-jmap-chat-00.md` |
| Federation method set (`Peer/*`) | `draft-atwood-jmap-chat-federation-00.md` |
| Federation peer authentication framing | `jmap-chat-federation-guide.md` §1 |
| Identity and federation deployment posture | `jmap-chat-implementer-guide.md` §3 |

### Informative external references

| Topic | Reference |
|---|---|
| DID URI syntax, DID Document data model | W3C DID Core v1.0 |
| `did:web` method specification | did:web v0.1 (w3c-ccg.github.io/did-method-web) |
| `did:key` method specification | did:key v0.7 (w3c-ccg.github.io/did-method-key) |
| `did:plc` method specification | PLC directory spec (web.plc.directory/spec/v0.1/did-plc) |
| HTTP Message Signatures | RFC 9421 |

### Reference deployment

The Agora protocol specification is a separate project that uses
overlapping DID-based identity primitives and provides a working
reference for several of the operational patterns in this guide.
Agora is informative only; this guide does not require any of its
choices.

| Topic | Agora Reference |
|---|---|
| Two-role verification-method split (Ed25519 + X25519) | Agora protocol spec §2.1 |
| Multi-device verification methods with `#device-<fingerprint>` | Agora protocol spec §2.5.4 |
| Enterprise IdP-driven DID provisioning sidecar | Agora universal-IdP-DID spec |
| STUB → ACTIVE → SUSPENDED → TOMBSTONED lifecycle | Agora protocol spec §2.5 |
| Recovery key offline-backup pattern | Agora protocol spec §2.4.1 |
| Social recovery with Shamir's Secret Sharing | Agora protocol spec §2.4.2 |
| Encrypted backup with Argon2id | Agora protocol spec §2.4.3 |
| RFC 9421 federation authentication with Ed25519 | Agora protocol spec §3.5.3 |
