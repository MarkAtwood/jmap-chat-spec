# JMAP and the ToIP Stack — Mapping Guide

For DID, DIF, and Trust over IP (ToIP) reviewers of the JMAP drafts in this
repository. Maps the JMAP DID, Chat, Chat Federation, Chat DID, and Chat
MIMI drafts onto the ToIP Technology Architecture Specification (TAS)
stack, and states where the mapping stops.

Read the drafts first. This guide describes them; it does not extend them.

---

## A note on requirement keywords

This guide uses no BCP 14 requirement keywords. Where it reports a draft
requirement, it names the draft section, and the draft's text and keyword
strength govern. Nothing here is a ToIP or DIF rule restated as a JMAP
requirement.

---

## 1. Scope

This guide is non-normative and adds no requirements. Nothing in any JMAP
draft depends on ToIP: no draft references ToIP, the TAS, or the Trust
Spanning Protocol (TSP), and an implementation can conform to every draft
without knowing ToIP exists. The drafts gain no ToIP text from this guide.

ToIP is a trust and governance architecture, not a JMAP wire protocol, so
this mapping lives in a guide. It is an analogy for readers who already
think in ToIP terms. Where a JMAP feature has no ToIP counterpart, or the
reverse, the guide says so rather than stretching the analogy.

ToIP sources, with status as of this writing:

- ToIP Technology Architecture Specification, Public Review Draft 02
  (2 September 2024), https://trustoverip.github.io/TechArch/. This is a
  public review draft. This guide found no published approved V1.0. The
  rendered page tracks the latest draft rather than a fixed version.
- Trust Spanning Protocol specification, labelled "Experimental
  Implementor's Draft Rev 3 with Errata" (protocol version 0.2),
  https://trustoverip.github.io/tswg-tsp-specification/. Not an approved
  ToIP deliverable.
- ToIP Governance Metamodel Specification V1.0 (21 December 2021), a ToIP
  Approved Deliverable,
  https://www.trustoverip.org/permalink/ToIP-Governance-Metamodel-Specification-V1.0-2021-12-21.pdf

`did:webvh` is a DIF specification
(https://identity.foundation/didwebvh/v1.0/), not a ToIP one. It is cited
here as DIF work.

---

## 2. Layer mapping

The TAS defines four layers: Layer 1 Trust Support, Layer 2 Trust
Spanning, Layer 3 Trust Tasks, and Layer 4 Trust Applications. Layer 2 is
the narrow waist of an hourglass. These names replace the 2020-era ToIP
picture (public utilities, DIDComm peer-to-peer, data/credential
exchange, application ecosystems, each paired with a governance stack),
and this guide uses only the current names.

| TAS layer | Closest JMAP corpus features | Where defined |
|---|---|---|
| 1 Trust Support | DID methods: `did:web`, `did:key`, and `did:plc` are mandatory to implement for `urn:ietf:params:jmap:chat:did`; `did:webvh` is recommended. Base JMAP DID mandates no method and advertises `supportedMethods`. Resolver behavior: caching, forced re-resolution on key mismatch, failure handling. Key custody disclosure via `keyCustody`. | `draft-atwood-jmap-chat-did-00` §Mandatory DID Methods, §DID Resolution; `draft-atwood-jmap-did-00` §Capability, §Resolving `did:webvh` |
| 2 Trust Spanning (parallel, not TSP) | Peer signing keys published as Ed25519 JWKs in the Session object's `peerSigningKeys`; RFC 9421 request signatures tagged `jmap-chat-peer`; DNSSEC key binding, recommended where DNSSEC is available; optional mTLS with DANE; identity binding of `ownerUserId` to the authenticated peer domain; DID-bound verification for host-less DIDs. | `draft-atwood-jmap-chat-federation-00` §Peer Authentication Model; `draft-atwood-jmap-chat-did-00` §DID-Bound Verification |
| 3 Trust Tasks | MLS-based end-to-end encryption through the MIMI capability and the E2EE relay topology; blocking (`ChatContact.blocked`); bans (`SpaceBan`); membership and permission checks in `Peer/deliver`. | `draft-atwood-jmap-chat-mimi-00`; `draft-atwood-jmap-chat-00` §End-to-End Encrypted Deployments, §SpaceBan, §Blocked Contacts; `draft-atwood-jmap-chat-federation-00` §New Message Validation |
| 4 Trust Applications | JMAP Chat, JMAP Scene, JMAP VTC, JMAP Graph. | `draft-atwood-jmap-chat-00`, `draft-atwood-jmap-scene-00`, `draft-atwood-jmap-vtc-00`, `draft-atwood-jmap-graph-00` |

### 2.1 Why JMAP federation is parallel to TSP

The TAS calls TSP the only protocol required at Layer 2. JMAP federation
does not implement TSP and is not built on it. It fills a similar role
(authenticated server-to-server exchange), but it is a different kind of
protocol. This is the guide's own reading. The drafts state no position on
TSP.

| Property | TSP | JMAP Chat federation |
|---|---|---|
| Endpoint identity | Verifiable Identifier (VID): a DID or URN with a verifiable trust root | A peer domain. The Session's `ownerUserId` is bound to it by host, or by DID-bound verification when host-less |
| Unit of exchange | Message: envelope (protocol tag, version, sender VID, receiver VID), payload, signature, encoded in CESR | HTTP request to a JMAP endpoint, signed per RFC 9421 over `@method`, `@target-uri`, and `content-digest` |
| Transport | Any transport | HTTPS only |
| Topology | Direct Mode, or Routed Mode through intermediaries with nested messages | Point-to-point between mailbox servers; no intermediaries |
| Interaction model | Message-based relationships formed by mutual VID verification | JMAP client/server calls: the caller fetches a Session, gets a `peer`-role account, and invokes `Peer/*` methods |
| Confidentiality | HPKE-Base or a Libsodium Sealed Box | TLS at transport. Content E2EE, where used, sits above federation (MLS) |
| Trust anchor | The VID's own verifiable trust root | WebPKI and the domain's DNS; DNSSEC when the key binding is published |

JMAP is an HTTP client/server synchronization protocol (RFC 8620), and the
federation draft carries peer traffic as JMAP method calls. It is not a
transport-independent peer messaging substrate, and that is what Layer 2
in the TAS describes. Treating JMAP federation as a TSP binding would
misstate both.

### 2.2 Layer 1 notes

- **Principals.** The non-normative `jmap-did-guide.md` (Section 2.1)
  recommends `did:webvh` for users who must control and rotate their own
  identity, and `did:web` for infrastructure principals only, because the
  host can replace a `did:web` document's keys. The drafts make no such
  split. The Chat DID draft makes `did:web`, `did:key`, and `did:plc`
  mandatory to implement and recommends `did:webvh`.
- **Key custody.** `keyCustody` is `"server"`, `"user"`, or `null` (not
  disclosed). A server that holds an authentication private key cannot
  set `"user"` (`draft-atwood-jmap-did-00` §Account-Level Capability
  Object). ToIP readers asking "who controls this identifier" get an
  answer here, but only as the server reports it.
- **Resolution is local.** Each server resolves cross-server DIDs itself.
  No delegated-resolution protocol exists (`draft-atwood-jmap-chat-did-00`
  §Federation Considerations).

---

## 3. Governance mapping

In ToIP terms, an ecosystem governance framework (EGF) is the governance
framework for a whole trust ecosystem. A governing authority publishes it,
and it sets policies, roles, rules, and assurance requirements for members.
The Governance Metamodel Specification is the template from which
ToIP-compatible governance frameworks are derived. (This summary of EGF is
the guide's paraphrase of ToIP usage, not a quotation of the ToIP
Glossary.)

The JMAP drafts define no governance framework, no governing authority,
and no way to publish or identify one. What they do define are three
mechanisms that a deployment's governance can be expressed through.

| EGF concern | JMAP mechanism | Normative? |
|---|---|---|
| Roles and rules for members of a community | Space roles and permissions: `SpaceRole` with a closed permission vocabulary (`view`, `send`, `pin`, `manage_channels`, `manage_members`, `manage_roles`, `manage_space`, `ban`, `mention_broadcast`, `start_call`), per-channel `ChannelPermission` overrides, a fixed resolution order, role hierarchy enforcement, and `SpaceBan` | Yes: `draft-atwood-jmap-chat-00` §SpaceRole, §ChannelPermission, §Space Permission Resolution, §SpaceBan |
| Which organizations may participate | Operator federation policy: open federation, an allowlist, or a denylist of peer domains | No. The federation draft defines wire mechanics only. Policy options appear in `jmap-chat-federation-guide.md` Section 2 |
| Assurance level of member identifiers | `did:webvh` witness policy | Partly. The drafts defer to DIF `did:webvh`. In employer-managed deployments witnesses are optional and a deployment choice (`draft-atwood-jmap-did-00` §Employer-Managed DID Deployments) |

Limits that matter to a ToIP reviewer:

- **Who governs a Space is deployment-defined.** The Chat draft lists
  three models: a single irreducible controller, no controller (role graph
  only), or delegation to an external IdP or directory
  (`draft-atwood-jmap-chat-00` §Space Governance: Deployment Variation).
- **Governance is not negotiated on the wire.** The same section
  recommends that federated deployments coordinate governance out of band,
  and states that no wire mechanism exists to advertise or negotiate a
  governance policy.
- **Authority does not cross servers.** A receiving server generally has no
  authority over the sender's permission graph. It may re-check
  `mention_broadcast` only if it holds an authoritative or signed copy of
  the Space permission state (`draft-atwood-jmap-chat-federation-00`
  §Broadcast Mention Resolution).
- **Witnesses mitigate split view only.** A witness does not sign two
  children of one parent. Rollback is caught only against a newer
  `versionId` already seen, or by comparing watchers. A first-time resolver
  with no witnesses or watchers has no protection against either; with
  witnesses but no watchers, it still has no rollback protection
  (`draft-atwood-jmap-did-00` §Resolving `did:webvh`). No draft sets a
  witness threshold or selection rule. When a log declares witnesses, DIF
  requires the verifier to enforce the threshold. That is a DIF rule, not
  a JMAP one.

---

## 4. Credentials gap

JMAP DID defines identity binding ("this account belongs to this DID"),
not attribute attestation. Verifiable Credentials issued to the bound DID
can layer on top, and a future JMAP capability could define VC
presentation and verification (`draft-atwood-jmap-did-00` §Relationship to
Verifiable Credentials, `#vc-relationship`). No current draft defines
either.

The patterns below need no draft changes. Each is a **deployment pattern,
not protocol feature**. Peers and clients see only the existing wire
objects, never the credential. `jmap-chat-vc-guide.md` works these
patterns through for Space creation, roles, membership, and revocation.

**4.1 VC-backed Space role.** *Deployment pattern, not protocol feature.*
The Space-hosting operator verifies a VC issued to a user's bound DID out
of band, for example at enrollment, and then sets that member's
`SpaceMember.roleIds`. On the wire this is an ordinary role assignment.
It fits the "delegation to an external IdP or directory" governance model
in `draft-atwood-jmap-chat-00` §Space Governance: Deployment Variation.
Other servers cannot verify that a credential backed the role.

**4.2 VC-informed contact display.** *Deployment pattern, not protocol
feature.* A client resolves the DID in a `ChatContact`'s `did` field and
checks a VC issued to it, then shows a local indicator. The Chat DID draft
defines no "verified" indication. It says a client should not present a
contact as verified solely because a DID is present, and should base any
indicator on its own resolution (`draft-atwood-jmap-chat-did-00` §Optional
`did` field). A VC check is one input a client could use. It is not
interoperable.

**4.3 Trust-registry lookup in peer enrollment.** *Deployment pattern, not
protocol feature.* An operator consults a trust registry, for example
through the ToIP Trust Registry Query Protocol
(github.com/trustoverip/tswg-trust-registry-protocol), before adding a
peer domain to its allowlist. No draft defines a peer-acceptance policy
hook for this, and no draft defines fail-open behavior.
Peer acceptance is entirely operator policy
(`jmap-chat-federation-guide.md` Section 2). How a registry outage is
handled is the operator's call. It is separate from the normative rule
that a DID resolution failure with no usable cache rejects the request
(`draft-atwood-jmap-chat-did-00` §Resolution failure handling).

**4.4 Attribute assurance through a credential oracle.** *Deployment
pattern, not protocol feature.* Where a deployment must restrict a Space,
region, or call by an attribute such as age, the operator asks a
credential oracle (`jmap-chat-vc-guide.md` Section 9) a narrow question,
for example "is this member over 18, and until when?", and maps a yes to
a role or to region access as in Section 4.1. The oracle can answer from
a selective-disclosure credential or a zero-knowledge proof presented by
the holder, so neither the oracle's caller nor any peer learns a
birthdate or an identity record. In ToIP terms the oracle performs a
Layer 3 verification task, and the deployment's ecosystem governance
framework decides which issuers it accepts. Peers and clients see only
the resulting role. What the deployment does when the oracle is
unreachable is an operator setting (`jmap-chat-vc-guide.md` Section 9).

---

## 5. Worked example (hypothetical)

No multi-organization demo exists in this repository. The scenario below
is hypothetical, uses `.example` domains, and relies only on features the
drafts define. Three standards bodies, called IETF, ASF, and W3C here,
each act as their own governance authority. Each appoints one liaison,
and each liaison runs their own mailbox, one peer domain with one
`ownerUserId`: `liaison.ietf.example`, `liaison.asf.example`, and
`liaison.w3c.example`. They want a shared liaison channel. A multi-user
organization server would be a relay deployment, and the federation
draft's Peer/* model covers neither relay-to-relay exchange nor
multi-user peers (`draft-atwood-jmap-chat-federation-00` §Deployment
Context).

**Who governs what.**

- Each body operates its liaison's mailbox. It binds that mailbox's
  account to one DID (`draft-atwood-jmap-did-00` §Account-to-DID Binding)
  and reports `keyCustody`.
- `liaison.ietf.example` hosts the liaison Space and so governs its roles,
  channels, and bans. Space-level operations are enforced by the
  Space-hosting server at delivery time and are not proactively federated
  (`draft-atwood-jmap-chat-federation-00` §Space-Level Federation).
- The three agree out of band on who may hold `manage_members` and `ban`,
  as the Chat draft recommends for federated Spaces. Nothing on the wire
  records that agreement.

**Admission (operator policy).** Each operator adds the other two peer
domains to an allowlist, the corporate pattern in
`jmap-chat-federation-guide.md` Section 2. Each operator is expected to
publish the DNSSEC key binding wherever its zone is DNSSEC-signed.

**A message from the ASF liaison into the liaison channel.**

1. `liaison.asf.example` signs `Peer/deliver` with a key from its
   `peerSigningKeys` (§Request Signing). `liaison.ietf.example` verifies the
   signature, time window, nonce, and content digest. The verified domain
   becomes the authenticated peer domain.
2. `liaison.ietf.example` checks that the peer's `ownerUserId` binds to
   `liaison.asf.example` (§Identity Binding). If that identity were a host-less
   DID, it would instead require DID-bound verification: the DID
   Document's `authentication` relationship has to contain the signing key
   (`draft-atwood-jmap-chat-did-00` §DID-Bound Verification).
3. `liaison.ietf.example` compares the caller-supplied sender against the bound
   peer identity. It then checks current membership, any active
   `SpaceBan`, the channel `send` permission, and slow mode
   (§New Message Validation). If any check fails, nothing is stored.

**What each party is trusting.**

- **IETF trusts ASF** to have bound its liaison's mailbox to the right
  person. The protocol shows that the sender is the authenticated peer
  domain's bound `ownerUserId`, by host match or DID-bound verification,
  not who the human is.
- **ASF and W3C trust IETF** to apply the agreed role and ban policy. A
  receiving server generally cannot re-check the sender's permission
  graph.
- **Identifier assurance for the W3C liaison.** Suppose
  `liaison.w3c.example` binds its owner to a `did:webvh` identity with
  witnesses (a deployment choice). Then other parties that resolve that
  DID get split-view mitigation for it. A first-time resolver still has no rollback protection.
- **Credential claims.** If the liaison role should mean "appointed
  liaison of my organization", each body can issue a VC to its appointee
  and IETF can assign the role after checking it (Section 4.1). Everyone
  else sees only the role.

The trust in this room's membership is therefore bilateral and
out of band. JMAP supplies authenticated peers, bound identifiers, and
server-enforced Space rules. It does not supply a shared ecosystem
governance framework.

---

## 6. Non-goals

- **No DIDComm transport.** Federation is JMAP over HTTPS with RFC 9421
  request signatures. The drafts define no DIDComm or TSP binding.
- **No mandatory Verifiable Credentials.** No draft requires, presents, or
  verifies VCs. The VC relationship section describes a possible future
  capability only.
- **No trust-registry protocol.** No draft defines or requires a trust
  registry, a registry query, or a policy hook for one. Registry use is
  operator policy (Section 4.3).
- **No identity enrollment or age registry.** No draft defines identity
  enrollment, stores a birthdate, or queries a government or other
  registry. Age and attribute assurance is an oracle answer only
  (Section 4.4).

---

## Cross-references

| Topic | Read also |
|---|---|
| DID binding, methods, `keyCustody`, VC relationship | `draft-atwood-jmap-did-00.md`, `jmap-did-guide.md` |
| Mandatory DID methods, DID-bound verification, resolution | `draft-atwood-jmap-chat-did-00.md`, `jmap-chat-did-guide.md` |
| Peer authentication, `Peer/*` methods | `draft-atwood-jmap-chat-federation-00.md` |
| Federation allowlist / denylist policy | `jmap-chat-federation-guide.md` |
| Credential oracle, VC-backed roles, attribute assurance | `jmap-chat-vc-guide.md` |
| Space roles, permissions, governance variation | `draft-atwood-jmap-chat-00.md` |
| MIMI and MLS | `draft-atwood-jmap-chat-mimi-00.md` |
| ToIP Technology Architecture Specification | https://trustoverip.github.io/TechArch/ |
| ToIP Trust Spanning Protocol | https://trustoverip.github.io/tswg-tsp-specification/ |
| ToIP Governance Metamodel Specification V1.0 | https://www.trustoverip.org/permalink/ToIP-Governance-Metamodel-Specification-V1.0-2021-12-21.pdf |
| DIF did:webvh v1.0 | https://identity.foundation/didwebvh/v1.0/ |
| W3C Verifiable Credentials Data Model v2.0 | https://www.w3.org/TR/vc-data-model-2.0/ |
