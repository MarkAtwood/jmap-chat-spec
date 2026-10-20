# JMAP DID — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-did-00`. Covers
binding strategies, DID method tradeoffs, verification patterns, lifecycle
handling, and integration decisions that the spec deliberately leaves to
implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what implementers need to know beyond the wire contract.

---

## How to read this guide

Each section below follows the same shape:

1. **What the spec defines** — with a draft citation.
2. **What you must decide** — the concrete implementation choice.
3. **Considerations** — trade-offs that inform the choice.
4. **Recommended starting point** — a defensible default.

This guide is non-normative. The draft is the source of truth. Where this guide
and the draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords for clarity. Where the keyword reflects a spec
requirement, the normativity belongs to the spec. Where the keyword reflects
implementer best-practice, deviation does not affect protocol interop. Cite the
spec, not the guide, when claiming normative authority.

---

## 1. What JMAP DID provides

JMAP (RFC 8620) identifies users by server-assigned `accountId` values. An
accountId is meaningful only within the server that issued it. Two JMAP servers
have no standard way to determine whether accounts represent the same person.

JMAP DID adds a DID binding — a globally resolvable, cryptographically verifiable
identifier derived from W3C Decentralized Identifiers (DID Core). It complements
`accountId`; it does not replace it.

| Property | `accountId` | DID |
|---|---|---|
| Assigned by | Server | User or organization (via DID method) |
| Scope | Single server | Universal |
| Verifiable | Only by the issuing server | By any party that can resolve the DID document |
| Portability | Not portable | Portable across servers |
| Key material | None | DID document contains public keys |

The capability identifier is `urn:ietf:params:jmap:did`. When a server
advertises it, every account gains a `did` binding in the account-level
capability object, and consuming JMAP specifications may add DID-bearing
properties to their data types.

---

## 2. DID method selection

### 2.1 Choosing a DID method

**What the spec defines.** The server declares `supportedMethods` (the DID
methods it can resolve and verify) in the session-level capability object.
Companion specs such as Chat DID reuse this field. The spec does not mandate any specific DID
method (draft §Capability (Session-Level)).

**What you must decide.** Which DID methods to support.

**Considerations.**

| Method | Resolution | Key recovery | Censorship resistance | Complexity |
|---|---|---|---|---|
| `did:key` | Self-resolving (no network) | None (key loss = identity loss) | Complete (no external dependency) | Minimal |
| `did:web` | HTTPS fetch from domain | Key rotation via document update | DNS/HTTP dependent | Moderate |
| `did:webvh` | HTTPS fetch of DID log, then full log verification | Key rotation via signed log entry; optional pre-rotation | DNS/HTTP dependent; portable DIDs can change domain | Moderate |
| `did:pkh` | Blockchain address derivation | Wallet recovery mechanisms | Chain-dependent | Moderate |
| `did:ion` | Bitcoin-anchored (via Sidetree) | Key rotation via DID operations | Bitcoin-resilient | High |
| `did:peer` | Peer-to-peer exchange | N/A (ephemeral by design) | Complete | Minimal |

**Recommended starting point.** Support `did:webvh`, `did:key`, and `did:web`,
each for its own kind of principal.

- `did:webvh` as the default for user identities that the user controls and
  must be able to rotate. The user holds the update keys; the host serving
  the log cannot add a key. Keys rotate without changing the DID, and a
  portable DID survives a domain move. Requires HTTPS hosting of a log file
  but no blockchain infrastructure.
- `did:key` for zero-infrastructure use: the DID is the public key. Users who
  lose their key lose their identity, and a key change is a DID migration —
  acceptable where nothing can be hosted, and the honest tradeoff.
- `did:web` only for infrastructure principals (servers, relays, services),
  where the operator of the domain is the principal and trusting the host is
  not a weakness. Do not use it for user identities: the host can silently
  replace the keys.

Add other methods when your deployment has a concrete need. Do not support
methods speculatively.

### 2.2 `did:key` implementation notes

A `did:key` DID is deterministically derived from a public key. The DID
document is not hosted anywhere — it is constructed from the DID itself.

```
did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
```

The `z6Mk` prefix indicates an Ed25519 public key (Multicodec 0xed). The
remaining characters are the Multibase-encoded (base58btc) public key bytes.

To resolve:

1. Strip the `did:key:` prefix.
2. Decode the Multibase string (base58btc, indicated by the `z` prefix).
3. The first two bytes are the Multicodec identifier (0xed01 for Ed25519).
4. The remaining 32 bytes are the raw Ed25519 public key.
5. Construct the DID document with an `authentication` verification method
   using this key, and a `keyAgreement` verification method using the
   corresponding X25519 key (computed from the Ed25519 key).

Libraries: `did-key` (Rust), `@digitalbazaar/did-method-key` (JS),
`did-key-creator` (Python). Do not hand-roll the Multicodec/Multibase
encoding.

### 2.3 `did:web` implementation notes

A `did:web` DID resolves to a DID document hosted at a well-known HTTPS URL.

```
did:web:example.com       → https://example.com/.well-known/did.json
did:web:example.com:services:relay → https://example.com/services/relay/did.json
```

The DID document is a JSON-LD document with at minimum an `authentication`
verification method:

```json
{
  "@context": "https://www.w3.org/ns/did/v1",
  "id": "did:web:example.com:services:relay",
  "authentication": [{
    "id": "did:web:example.com:services:relay#key-1",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:web:example.com:services:relay",
    "publicKeyMultibase": "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
  }]
}
```

**Hosting considerations:**

- Serve DID documents over HTTPS with valid TLS certificates. HTTP is not
  acceptable — DID resolution over HTTP is trivially MITM-able.
- Set `Cache-Control` headers appropriately. Short TTLs (5-15 minutes) allow
  key rotation to propagate quickly. Long TTLs reduce resolution traffic but
  delay rotation visibility.
- DID documents MUST be served with `Content-Type: application/did+json` or
  `application/did+ld+json`.
- Host DID documents on infrastructure you control. A CDN is fine for
  performance, but the origin must be under your administrative control.

### 2.4 `did:webvh` implementation notes

A `did:webvh` DID (DIF did:webvh v1.0) names a self-certifying identifier
(SCID) and a web location. It resolves to a hash-chained DID log, one JSON
entry per line; the DID document is the state in the latest verified entry.
The SCID is not part of the URL.

```
did:webvh:{SCID}:example.com             → https://example.com/.well-known/did.jsonl
did:webvh:{SCID}:example.com:users:alice → https://example.com/users/alice/did.jsonl
did:webvh:{SCID}:example.com%3A3000:dids → https://example.com:3000/dids/did.jsonl
```

A log that declares witnesses also has `did-witness.json` next to
`did.jsonl`. Fetch it from the method-derived URL even when the log itself
came from `didDocumentUri` or a watcher.

**What the spec defines.** The draft defers `did:webvh` resolution to DIF
did:webvh v1.0 and adds no resolver requirements of its own. It says parties
SHOULD record the `versionId` in effect when they accept an object, because
without it they cannot re-verify history after a key rotation (draft
§Resolving `did:webvh`).

**What DIF requires.** DIF requires the resolver to verify every log entry
(SCID, entry hash, version numbering, `versionTime` ordering, update-key
authorization including pre-rotation) and, when the log declares witnesses,
the witness threshold. Resolution fails on an invalid log. DIF allows treating
the first invalid entry and everything after it as invalid; earlier versions
stay usable for checking stored objects. DIF also warns that a parallel
`did:web` document carries none of the log's guarantees.

**Where verification lives.** Split resolution into two parts:

- A pure verifier: input is the DID string plus the `did.jsonl` bytes (and
  the `did-witness.json` bytes, when the log declares witnesses); output is
  the verified sequence of versions, or an error. No I/O. It checks the
  SCID, the entry hash chain, `versionTime` ordering, update-key authorization (pre-rotation via
  `nextKeyHashes` when active), and witness thresholds. This is the part to
  test exhaustively, against the DIF test vectors and logs produced by an
  independent implementation.
- A fetcher: the DID-to-HTTPS transform and the HTTPS GET. It returns bytes
  and nothing else. It is untrusted by construction — a log fetched from a
  watcher, a cache, or the `didDocumentUri` hint goes through the same
  verifier as one fetched from the method-derived URL.

**Caching by `versionId`.**

- A verified version is immutable: cache it keyed by `(DID, versionId)`
  indefinitely. Stored objects need the version they were accepted under, and
  a key removed by a later entry must not make them fail verification.
- Record the `versionId` alongside each accepted DID-bearing or signed object.
  When verifying it later, look up that version, not the latest.
- Cache only the pointer to the latest version with a TTL (same 5-15 minute
  range as `did:web`). Refresh it on signature-verification failure.
- Remember the latest `versionId` you have seen for each DID (a DIF SHOULD).
  A freshly fetched log that ends before that version is truncated: warn or
  fail resolution, as DIF recommends, and keep the newer cached versions for
  stored objects. Without watchers, this is the only rollback check, and it
  protects only DIDs you have resolved before.
- Never cache a log that fails verification.

**Witnesses.** A verified log proves only that each entry you received was
authorized. It does not show that you received the latest entry, or the same
log other resolvers received. Witnesses mitigate split view: under DIF a
witness must not sign two children of one parent. They do not catch rollback.
DIF catches rollback only against a newer `versionId` you have already seen,
or by comparing watchers. A first-time resolver with no witnesses or watchers
has no protection against either. When a log declares witnesses, DIF requires
the verifier to enforce the threshold.

**Parallel `did:web`.** A holder may also publish `did.json` at the same
location, with `alsoKnownAs` naming the `did:webvh` DID. Do not take keys from
it unless they appear in the verified log's current document.

Libraries: `didwebvh-py` (Python), `didwebvh-ts` (TypeScript).

---

## 3. Account-to-DID binding

### 3.1 Binding strategies

**What the spec defines.** The server is authoritative for the accountId-to-DID
mapping. The spec lists four binding mechanisms but does not prescribe one
(draft §Account-to-DID Binding).

**What you must decide.** Which binding strategy to implement.

#### Strategy A: Server-generated `did:key`

The simplest approach. The server generates an Ed25519 keypair for each
account and derives a `did:key`. The server holds the private key.

```
User creates account
  → Server generates Ed25519 keypair
  → Server derives did:key from public key
  → Server stores (accountId, did, privateKey) mapping
  → Account capability object populated with did
```

**When to use:** Employer-managed deployments, or as a temporary bootstrap
before users move to self-sovereign keys. The draft makes this model NOT
RECOMMENDED outside employer-managed deployments (draft §Account-to-DID
Binding).

**Tradeoff:** The server holds the private key, so the DID is not truly
self-sovereign. The server can impersonate the user, and verifying the DID
gives no assurance independent of the server. The draft forbids presenting
this binding as user-verifiable. Set `keyCustody` to `"server"` in the
account capability object so clients and federated peers can see this and
choose a trust level.

#### Strategy B: Self-registration

The user brings their own DID and proves control of the private key.

```
User presents DID to server
  → Server fetches DID document
  → Server extracts authentication key
  → Server sends challenge (random nonce)
  → User signs challenge with private key
  → Server verifies signature against DID document key
  → Server stores (accountId, did) mapping
```

**When to use:** Consumer deployments where users manage their own keys
(wallet apps, privacy-focused services). The server never sees the private key.

**Tradeoff:** Users who lose their private key lose access. The server
cannot recover the account. Social recovery or recovery keys (out of scope
for this spec) mitigate this.

#### Strategy C: OIDC claim mapping

The identity provider includes a DID in the ID token (e.g., as a custom claim).

```
User authenticates via OIDC
  → IdP issues ID token with "did" claim
  → Server extracts DID from token
  → Server stores (accountId, did) mapping
```

**When to use:** Enterprise deployments where the IdP (Okta, Entra, Google
Workspace, etc.) is the authoritative identity source and has been extended
to issue DIDs.

**Tradeoff:** Requires IdP customization. Not all IdPs support custom claims
natively. Some require middleware or claim transformation rules.

#### Strategy D: Employer-managed sidecar

An external process watches IdP lifecycle events and maintains DID documents.

```
IdP fires user.create event
  → Sidecar generates update keys, signing keys, and genesis log entry
  → Sidecar publishes did:webvh log (did.jsonl)
  → JMAP server queries sidecar for accountId → DID mapping
```

The sidecar holds the update keys, so the organization controls the log.
Witnesses are optional, a deployment choice. Without them, the organization
can show different valid logs to different resolvers (split view). Witnesses
mitigate split view only; rollback to an old prefix is caught only by a
resolver that has already seen a newer `versionId`, or by comparing watchers
(Section 2.4). Enable portability at creation if the organization may change domain.

**When to use:** Enterprise deployments where the IdP cannot be extended to
issue DIDs directly. The sidecar sits between the IdP and the JMAP server,
translating identity lifecycle events into DID lifecycle events.

**Lifecycle mapping:**

| IdP event | DID lifecycle | JMAP effect |
|---|---|---|
| User created | Genesis log entry published (ACTIVE) | Account binding established |
| User suspended | Log entry removes verification methods | Session creation blocked (draft §Suspension) |
| User deleted | Log entry sets `deactivated: true` (permanent) | Sessions terminated (draft §Revocation) |
| User reactivated | Log entry restores verification methods | Account restored |

**Tradeoff:** Additional infrastructure (the sidecar process). But it
decouples DID management from both the IdP and the JMAP server, which is
operationally clean.

### 3.2 Binding storage

**What you must decide.** How to store the accountId-to-DID mapping.

**Recommended starting point.** Add a `did` column (or equivalent) to your
account table. Index it for reverse lookups (DID → accountId). The column is:

- NOT NULL when the DID capability is active.
- UNIQUE — the spec requires exactly one DID per account (draft §Account-to-DID Binding).
  The reverse (one account per DID) is not required by the spec but is
  RECOMMENDED to prevent confusion.
- Updated on DID migration (the old value is replaced, not appended). Key
  rotation does not touch this column; the DID stays the same.

For `previousDid` (draft §Account-Level Capability Object, §DID Migration),
store the old DID in a separate column or audit log with a timestamp. Clear it
after the deployment-defined transition period.

Store the custody state alongside the binding so you can populate
`keyCustody` (`"server"` for Strategy A, and for any strategy where your
server or a party acting for it holds an authentication private key).

### 3.3 Startup and migration

**What you must decide.** How to handle existing accounts when the DID
capability is first enabled.

**Recommended starting point.**

1. Generate a `did:key` for every existing account that lacks a DID binding
   (Strategy A, server-generated), with `keyCustody` set to `"server"`. This
   ensures the capability object is immediately populated for all accounts.
   Outside employer-managed deployments, treat this as a temporary bootstrap
   only (Strategy A is NOT RECOMMENDED there).
2. Allow users to replace the server-generated DID with a self-sovereign DID
   later (Strategy B, self-registration). The replacement is a DID migration
   (Section 6.1).
3. Document the migration path so users know their initial DID is server-held
   and can be upgraded.

This avoids a flag day where all users must register a DID before the
capability can be enabled.

---

## 4. DID-bearing properties on the server

### 4.1 Populating DID-bearing properties

**What the spec defines.** DID-bearing properties are server-set, recorded
at object creation from the account binding, and immutable afterward. A DID
migration does not change them, and signatures on an object verify against the
DID recorded on it (draft §The `did` Property). The server MUST reject a
client create that includes them or an update that changes them.

The draft separates these from **DID-referencing** properties, which hold the
DID of another party (a peer, participant, organizer, or a sender as seen by a
recipient). Those are set as the consuming spec defines, not from the binding,
and are not trusted until verified (Section 5). This section covers
DID-bearing properties only.

**What you must decide.** How to record the DID at creation.

**Recommended starting point.** Write the account's current DID into a column
on the object in the same transaction that creates it. Never update it
afterward. Do not compute the value at response time from the current binding:
after a DID migration that would relabel old objects with the new DID, which
the draft forbids.

A response-time lookup is still acceptable if it resolves the DID that was
bound at the object's creation time (for example, from a binding history table
keyed by accountId and validity interval). Storing the value on the object is
simpler.

### 4.2 Rejecting client-supplied DIDs

**What the spec defines.** Clients omit DID-bearing properties on create; an
update may include one only with its current value (RFC 8620 Section 5.3).
Servers MUST reject a create that includes a DID-bearing property, or an
update that changes one, with an `invalidProperties` SetError naming the
property. An update that repeats the current value is accepted.

**What you must decide.** Nothing — this is mandatory. But note:

- Validate at the JMAP method handler level, not at the database level. A
  database constraint is a safety net, not a substitute for returning a clean
  JMAP error.
- Return the `invalidProperties` SetError with the offending property in
  `properties`. Do not silently strip it.

---

## 5. Client verification

### 5.1 When to verify

**What the spec defines.** Verification is OPTIONAL (draft §Client Verification). Clients
MAY resolve the DID document independently and check that the server's
attestation is consistent.

**What you must decide.** Whether and when your client verifies.

**Considerations.**

| Deployment | Verify? | Rationale |
|---|---|---|
| Single-server, trusted operator | No | The client already trusts the server for everything else |
| Enterprise, employer-managed DIDs | Seldom | The employer controls both the server and the DID documents; verification confirms infrastructure consistency but adds latency |
| Federated, multi-server | Yes, on first contact | Verify DIDs from unfamiliar servers; cache the result for subsequent interactions |
| Zero-trust, adversarial | Always | Every DID attestation is verified against the DID document before display |

**Recommended starting point.** Verify on first contact with a new DID. Cache
the verification result (DID → verified/unverified) with a TTL matching the DID
document's `Cache-Control` header. Re-verify on cache expiry, after a key rotation is suspected (for example,
a signature fails), or when an account's DID changes (DID migration, seen as a
Session `state` change). DID values on existing objects never change.

### 5.2 Verification procedure

For `did:key`:

1. Parse the DID to extract the public key (see Section 2.2).
2. The DID document is deterministic — no network fetch needed.
3. Verification succeeds if the DID is syntactically valid and the key type
   is recognized.

For `did:web`:

1. Derive the DID document URL from the DID.
2. Fetch the document over HTTPS.
3. Verify the TLS certificate chain (standard HTTPS validation).
4. Parse the DID document and extract the `authentication` verification method.
5. Optionally: verify that a signed assertion from the server (e.g., a
   JWS-signed session token) can be validated against the DID document's
   authentication key.

For `did:webvh`:

1. Derive the `did.jsonl` URL from the DID, or use `didDocumentUri`, and
   fetch the log. If the log declares witnesses, fetch `did-witness.json`
   from the method-derived URL, even when the log came from `didDocumentUri`
   or a watcher.
2. Run the pure verifier over the bytes (see Section 2.4). On failure, treat
   the DID as unverified; do not fall back to a parallel `did:web` document.
3. Take the `authentication` verification method from the current verified
   document (or, for a stored object, from the version recorded at
   acceptance).
4. Require a signed assertion from the server. Unlike `did:web`, a TLS fetch
   from the server's own domain proves nothing here: the host need not hold
   any key in the log.

**Failure handling:**

- DID document fetch fails (network error, HTTP 404, TLS error): treat the
  DID as unverified. Display it to the user with a visual indicator (e.g.,
  "identity not verified"). Do not block the interaction.
- DID document key does not match: treat as a verification failure. Display a
  warning ("identity could not be verified — the server's claim does not match
  the DID document"). Log the mismatch for debugging. Do not silently proceed.
- DID method unrecognized: treat as unverifiable. Display normally without a
  verified badge. Do not fail.

### 5.3 Cross-server verification

**What the spec defines.** When objects originate from a different server
(federated messages, cross-server calendar invitations, shared files), the
originating server's DID-bearing value arrives as a DID-referencing value,
which the client can verify without trusting the originating server (draft
§Cross-Server Verification).

**What you must decide.** How to present cross-server identity to the user.

**Recommended starting point.** Display the DID alongside the server-local
display name. Use visual differentiation:

- Verified DID (document resolved, key matches): show a verification indicator.
- Unverified DID (not yet checked or check failed): show the DID without the
  indicator, or with an "unverified" label.
- No DID (capability not present on originating server): show server-local
  identity only.

Do not block interactions based on verification status. Users may communicate
with unverified identities; the verification indicator is informational.

---

## 6. DID lifecycle handling

### 6.1 Key rotation and DID migration

**What the spec defines.** The draft uses "rotation" only for key rotation:
new keys in the DID document, same DID. Key rotation needs no JMAP-layer
action. A DID migration is a change of the bound DID. On migration the server
MUST update the account capability object, new objects carry the new DID,
existing objects keep the DID recorded at creation, and the server MUST set
`previousDid` (draft §Key Rotation, §DID Migration).

**What you must decide.**

- How long to retain `previousDid`.

**Recommended starting point.**

- Key rotation: do nothing in the JMAP layer. Invalidate any cached DID
  document for the DID so the next verification picks up the new keys. For
  `did:webvh`, invalidate only the latest-version pointer; keep the cached
  versions that stored objects reference (Section 2.4).
- `did:webvh` domain move: a portable DID that moves gets a new DID string
  with the same SCID. Handle it as a DID migration. Before setting
  `previousDid`, confirm the new log continues the old one and lists the
  old DID in `alsoKnownAs`.
- DID migration: update the account table, set `previousDid`, and let the
  Session `state` change. No per-type StateChange is needed, because no
  existing object changes.
- Retain `previousDid` for 30 days. This gives clients time to update cached
  DID-to-identity mappings. After 30 days, set `previousDid` to null.

**Client-side handling:**

- When `sessionState` changes, refetch the Session and compare the account's
  `did` with the cached value.
- If it differs, update local identity mappings that should follow the person
  (display name associations, block lists). Keep the old DID on old objects for
  signature verification and attribution.
- If `previousDid` is present, clients that cached the old DID can correlate
  old and new identities without user intervention.

### 6.2 Suspension

**What the spec defines.** The server SHOULD reject new object creation, block
new sessions, and terminate live presence for suspended accounts
(draft §Suspension).

**What you must decide.**

- Whether suspension is driven by the DID layer (DID document deactivated) or
  the JMAP layer (account flagged as suspended), or both.
- How to communicate suspension to the user.

**Recommended starting point.**

- Check DID status at authentication time. If the DID document indicates
  deactivation (or the sidecar reports suspension), reject the session.
- For already-authenticated sessions: check DID status periodically (e.g.,
  every 5 minutes) or on push from the binding layer. Terminate the session
  if the DID is suspended.
- Return `403 Forbidden` on API calls from suspended accounts, with a
  human-readable error indicating the account is suspended. Do not return
  `401 Unauthorized` — the credentials may still be technically valid; the
  account state is the issue.
- Existing objects remain visible to other users. Do not delete or hide them.

### 6.3 Revocation

**What the spec defines.** The server SHOULD terminate all sessions and prevent
new authentication. Objects SHOULD retain the revoked DID for provenance
(draft §Revocation).

**What you must decide.**

- Retention policy for orphaned objects.
- Whether to allow account recovery (re-binding to a new DID).

**Recommended starting point.**

- Terminate all active sessions immediately. Invalidate all refresh tokens,
  session cookies, and API keys associated with the account.
- Mark the account as revoked in your account table. Do not delete the account
  record or its objects.
- Retain the revoked DID on all objects. This preserves provenance: "this
  message was sent by did:key:z6Mk..., which has since been revoked."
- Define a retention period (e.g., 90 days, 1 year, or indefinite) based on
  your compliance requirements. After the retention period, objects MAY be
  deleted.
- Account recovery (re-binding to a new DID) is a deployment decision. If
  supported, handle it as a DID migration (Section 6.1). The revoked DID
  becomes `previousDid`.

---

## 7. Integration with consuming specifications

### 7.1 General pattern

When your JMAP server implements both `urn:ietf:params:jmap:did` and a
consuming specification (Mail, Calendars, Contacts, FileNode, Chat, VTC,
Scene), the DID capability adds properties to the consuming specification's
data types. A property is either DID-bearing (the owning account's own DID,
from the binding) or DID-referencing (another party's DID, set as the
consuming spec defines and verified before it is trusted). The sections
below label each. For DID-bearing properties the pattern is always the same:

1. Identify which data types are DID-bearing (defined by the consuming spec).
2. For each DID-bearing type, identify which account binding the DID derives
   from (usually the object's `accountId`).
3. At creation time, record the owning account's current DID on the object
   (Section 4.1). Return the recorded value in responses.
4. Reject a create that includes a DID-bearing property, or an update that
   changes one; accept an update that repeats the current value.

### 7.2 JMAP Mail

**Email objects.** Add `senderDid` — the DID of the authenticated sender
(DID-bearing on the sender's own copy, DID-referencing as received).

For locally-originated mail, `senderDid` is the DID bound to the sender's account
when the Email was created; a later DID migration does not change it. For inbound mail from external servers, `senderDid` is `null` unless
the originating server provides a DID attestation via a mail header or
out-of-band mechanism. Do not fabricate a DID for external senders.

**Identity objects.** Add `did` — the DID bound to this sending identity
(DID-bearing).

**Interaction with DKIM/SPF/DMARC:** DID-based sender identity is orthogonal
to DNS-based email authentication. They verify different things:

| Mechanism | What it verifies | Trust anchor |
|---|---|---|
| DKIM | The message body was signed by the sending domain | DNS (domain key) |
| SPF | The sending IP is authorized by the domain | DNS (SPF record) |
| DMARC | DKIM/SPF alignment with the From header | DNS (DMARC policy) |
| DID | The sender controls a specific cryptographic identity | DID document |

A message can pass DMARC and fail DID verification (server claims a DID it
doesn't control), or fail DMARC and pass DID verification (sender's domain DNS
is misconfigured but the sender's DID key is valid). Display both signals
independently.

### 7.3 JMAP Calendars

**CalendarEvent objects.** Add `organizerDid` for the event organizer
(DID-bearing on the organizer's copy, DID-referencing on an invitee's copy) and
optionally `delegateDid` for delegate-created events (DID-referencing).

**Participant objects.** Add `did` for each participant (DID-referencing).

**Invitation verification:** When a user receives a calendar invitation from
an unfamiliar organizer, the client can resolve `organizerDid` and verify the
invitation's provenance without trusting the calendar server that delivered it.
This is especially valuable for cross-organization scheduling where the
organizer's server is not trusted.

### 7.4 JMAP Contacts

**ContactCard objects.** Add `did` as a stable cross-server identifier
(DID-referencing).

**Deduplication:** When a user has contacts on multiple JMAP servers, matching
`did` values identify the same person without heuristic matching on name,
email, or phone number.

**Portable block/allow lists:** Block or allow decisions keyed by DID travel
with the user across servers. A contact blocked by DID on Server A is
immediately blockable on Server B without re-identifying them.

### 7.5 JMAP File Storage

**FileNode objects.** Add `ownerDid` for the file owner (DID-bearing).

**Combined with JMAP CID:** A FileNode with `sha256` (from JMAP CID) and
`ownerDid` (from JMAP DID) creates a verifiable content-addressed ownership
claim: "the file with content hash H belongs to DID D." This binding is
verifiable by any party without trusting the storage server.

**Migration:** When migrating files between JMAP servers, the `ownerDid`
preserves ownership provenance. The receiving server can verify that the
claimed owner matches the DID on the source server's FileNode.

### 7.6 JMAP Chat

**Message objects.** Add `senderDid` for the message sender (DID-bearing on
the sender's own copy, DID-referencing as received).

**ChatContact objects.** Add `did` for the contact (DID-referencing).

**Federation:** In federated chat, `senderDid` enables recipient-side
verification. The recipient's client resolves the sender's DID document and
verifies the sender's identity independently of the originating server. This
is the strongest cross-server identity guarantee available without end-to-end
encryption.

**Blocked-sender portability:** When `ChatContact.blocked` is keyed by DID
rather than server-local contactId, the block list is portable. A user who
blocks `did:key:z6Mk...` on one server can enforce that block on any server
that supports JMAP DID.

### 7.7 JMAP VTC

**VTCParticipant objects.** Add `did` for the call participant
(DID-referencing).

**Call verification:** In calls that span server boundaries (federated calls,
SFU-mediated calls), `did` enables each participant to verify the identity of
other participants without trusting the call server. The client resolves each
participant's DID document and verifies the key material.

### 7.8 JMAP Scene

**SceneAvatar objects.** Add `did` for the avatar owner (DID-referencing).

**SceneObject objects.** Add `ownerDid` for the object creator.

**SceneAsset objects.** Add `uploaderDid` for the asset uploader.

**Cross-server avatar identity:** A user visiting a Scene region on a
different server presents the same DID. The hosting server can verify the
visitor's identity by resolving the DID, without federation-level trust.

**Asset provenance with JMAP CID:** A SceneAsset with `sha256` (content hash)
and `uploaderDid` (creator identity) creates a portable, verifiable asset
provenance chain. The asset can be migrated between servers while preserving
both content integrity and creator attribution.

---

## 8. Privacy and correlation

### 8.1 Same DID across servers

**What the spec defines.** A user who presents the same DID to multiple servers
enables cross-server activity correlation (draft §DID Correlation Across Servers).

**What you must decide.** Whether your deployment requires same-DID or permits
per-server DIDs.

**Considerations.**

- *Enterprise deployments* typically require a single DID per employee across
  all servers. Cross-server correlation is a feature (IT can track the
  employee's identity across systems). Privacy cost: servers can collude to
  build a comprehensive activity profile.
- *Consumer deployments* may permit per-server DIDs. Users who want cross-server
  identity use the same DID everywhere. Users who want unlinkability generate a
  fresh `did:key` for each server. Privacy benefit: servers cannot correlate.
  Cost: the user is a different identity on each server.

**Recommended starting point.** Let the user choose. Support both same-DID and
per-server-DID usage patterns. Document the privacy implications of each.

### 8.2 DID in responses to other users

DID-bearing and DID-referencing properties are returned to any client that can
read the object. This means:

- Every participant in a chat room can see every other participant's DID.
- Every visitor to a Scene region can see every avatar's DID.
- Every recipient of a calendar invitation can see the organizer's DID.

This is by design — the DID is the identity, and identity must be visible for
verification to work. But it means DIDs are not private. Users should
understand that their DID is visible to anyone they interact with.

**Recommended starting point.** Display DIDs only on request or in identity
verification flows. Do not make the raw DID string a prominent part of the UI.
Use display names from the contact or profile system, with the DID available
for verification when the user explicitly requests it.

---

## 9. DID method tradeoffs at a glance

| Concern | `did:key` | `did:web` | `did:webvh` |
|---|---|---|---|
| Setup cost | None (generate keypair) | Moderate (host DID document over HTTPS) | Moderate (host DID log over HTTPS) |
| Key rotation | Impossible (new key = new DID, a DID migration) | Supported (update document) | Supported (signed log entry; optional pre-rotation) |
| Key recovery | None | Organization controls the document | Update-key holder controls the log |
| Resolution speed | Instant (no network) | HTTPS fetch + TLS handshake | HTTPS fetch + full log verification |
| Censorship risk | None | Domain seizure disables the DID | Domain seizure disables the location; a portable DID can move |
| Phishing risk | None (opaque key hash) | Domain confusion attacks | Domain confusion attacks |
| Auditability | Low (no trail of key changes) | High (document versioning, access logs) | High (verifiable log of every change) |
| Host can replace keys | Not applicable | Yes | No (needs an update key) |
| Enterprise fit | Poor (no key management) | Infrastructure principals only | Good (organization holds update keys; witnesses optional) |
| Consumer fit | Good (zero infrastructure) | Poor (requires HTTPS hosting) | Good (default for user-controlled rotating identities) |

Most deployments will support all three and let the binding strategy
determine which method is used for each account.

---

## 10. Operational checklist

Before enabling `urn:ietf:params:jmap:did` in production:

- [ ] Binding strategy selected and implemented (Section 3.1)
- [ ] Account table extended with DID column (Section 3.2)
- [ ] Existing accounts migrated or bootstrapped (Section 3.3)
- [ ] DID-bearing properties added to relevant data types (Section 4.1)
- [ ] Client-supplied DID rejection tested (Section 4.2)
- [ ] Session capability object populated with `supportedMethods` (draft §Capability (Session-Level))
- [ ] Account capability object populated with `did`, `didDocumentUri`, and `keyCustody` (draft §Capability (Account-Level))
- [ ] DID migration handling implemented: account update + `previousDid`, existing objects untouched (Section 6.1)
- [ ] Suspension handling implemented: session rejection + presence termination (Section 6.2)
- [ ] Revocation handling implemented: session termination + object retention (Section 6.3)
- [ ] `did:web` document hosting validated (if applicable): HTTPS, correct Content-Type, Cache-Control (Section 2.3)
- [ ] `did:webvh` log verifier tested against independent vectors, `versionId` recorded on accepted objects, version cache in place (if applicable) (Section 2.4)
- [ ] Privacy documentation updated: correlation properties, DID visibility (Section 8)

---

## Cross-references

| If you are implementing... | Read also... |
|---|---|
| Core JMAP (sessions, accounts, blobs) | RFC 8620 |
| Content-addressed blob integrity | `jmap-cid-guide.md` and `draft-atwood-jmap-cid-00.md` |
| JMAP Mail | RFC 8621 |
| JMAP Calendars | RFC 8984 |
| JMAP Contacts | RFC 9610 |
| JMAP File Storage | `draft-ietf-jmap-filenode` |
| JMAP Chat (governance, authorization) | `jmap-chat-implementer-guide.md` |
| JMAP VTC (calls, participants) | `jmap-vtc-implementer-guide.md` |
| JMAP Scene (regions, objects, avatars) | `jmap-scene-implementer-guide.md` |
| W3C DID Core | https://www.w3.org/TR/did-core/ |
| DIF did:webvh v1.0 | https://identity.foundation/didwebvh/v1.0/ |
| W3C Verifiable Credentials | https://www.w3.org/TR/vc-data-model-2.0/ |

This guide focuses on the DID capability and its integration points. Transport,
push, federation, and data-type-specific operational details live in their
dedicated guides.
