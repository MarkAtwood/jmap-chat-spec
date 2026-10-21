# JMAP Chat and Verifiable Credentials — Deployment Guide

For operators and implementers who want Verifiable Credentials (VCs) to
decide who may create a Space, who administers it, and who may join it.
Describes how to do that with the drafts as they stand, and states what
the drafts do not let an operator do.

Read the drafts first. This guide describes them; it does not extend them.

---

## A note on requirement keywords

This guide uses no BCP 14 requirement keywords. Where it reports a draft
requirement, it names the draft section, and the draft's text and keyword
strength govern. Where it reports a credential-format rule, it names that
format's specification. Nothing here is a W3C, IETF, or ToIP rule restated
as a JMAP requirement.

---

## 1. Scope

This guide is non-normative and proposes no draft changes. No JMAP Chat or
federation draft mentions Verifiable Credentials. JMAP DID defines identity
binding ("this account belongs to this DID"), not attribute attestation,
and names VC presentation and verification as a possible future capability
(`draft-atwood-jmap-did-00` §Relationship to Verifiable Credentials). No
current draft defines that capability.

VCs therefore stay outside the wire protocol. Every pattern below is a
**deployment pattern, not protocol feature**: the operator verifies a
credential through its own channel and then acts through ordinary JMAP
methods. Clients and federation peers see only existing wire objects
(`Space`, `SpaceRole`, `SpaceMember`, `SpaceInvite`, `SpaceBan`), never the
credential.

---

## 2. The model

| Party | In this guide |
|---|---|
| Issuer | A body that attests something about a person, for example "appointed liaison" or "member of the secretariat" |
| Holder | The user, identified by the DID bound to their account (`draft-atwood-jmap-did-00` §Account-to-DID Binding), or, for ACDC, by a KERI AID |
| Verifier | The operator's JMAP server, or an enrollment service the operator runs beside it |
| Relying party for the result | The Space-hosting server, which enforces roles and membership |

**Holder binding depends on key custody.** A credential issued to the
user's bound DID proves something about whoever controls that DID's keys.
If the account's `keyCustody` is `"server"`, the server holds at least one
authentication key (`draft-atwood-jmap-did-00` §Account-Level Capability
Object), so a holder proof made with that key shows the server's control,
not the user's. JMAP DID says only user-held or independent-IdP-held keys
give guarantees independent of the server
(`draft-atwood-jmap-did-00` §Server-Generated DID Key Pairs).

**Trust in issuers is operator policy.** None of the three formats in
Section 7 defines how a verifier picks trusted issuers. VC Data Model 2.0
puts it out of scope (§5.1 Trust Model). SD-JWT VC leaves the issuer
authorization check to the verifier without saying how
(draft-ietf-oauth-sd-jwt-vc-19 §7.7). ACDC chains authority through edges
but leaves the choice of root issuer to the verifier (ACDC §Edge Section).
The JMAP drafts define no issuer list, and no wire mechanism exists to
advertise a governance policy (`draft-atwood-jmap-chat-00` §Space
Governance: Deployment Variation). A pinned, version-controlled list of
issuer identifiers is the simplest form this policy can take.

---

## 3. VC-gated Space creation

*Deployment pattern, not protocol feature.*

**What the drafts say.** `Space/set` create takes `name` and, optionally,
`description` and `iconBlobId`. The server adds the caller with a
server-defined bootstrap role that can administer the Space
(`draft-atwood-jmap-chat-00` §Creating a Space). The draft does not say
who may create a Space, and create has no rejection error of its own.
Destroy is different: it names a permission, allows deployment-defined
extra requirements, and returns `forbidden` when they are not met
(§Destroying a Space).

**What the operator can do.**

- **Gate the account, not the method.** Only the mailbox owner may call
  methods on an account; anyone else gets HTTP 401
  (`draft-atwood-jmap-chat-00` §Authorization). An operator that checks a
  credential before provisioning an account has gated every Space that
  account will create. This uses no Space mechanism at all. In the
  mailbox-per-user federation model each mailbox has one owner
  (`draft-atwood-jmap-chat-federation-00` §Deployment Context), so this
  is the natural gate there.
- **Refuse the create on a multi-account server.** The drafts let
  deployments add authorization policy beyond the named permissions, and
  the wire contract for an unauthorized caller is `forbidden`
  (`draft-atwood-jmap-chat-00` §Space Permission Resolution,
  §Authorization). That text is written about the "Requires X" clauses,
  which create does not have. Returning a `forbidden` SetError for a
  create by an account without a valid credential is this guide's reading
  of that latitude, not a stated rule for create.
- **Shape the bootstrap role.** Because the bootstrap role is
  server-defined (§Creating a Space), a server can give a
  credential-holding creator a broader role than other creators.

**What the operator cannot do.** The caller cannot attach a credential or
presentation to `Space/set` create. The check happens before the call, at
login or enrollment, and the server keeps its own record of the result.

---

## 4. Admins via VC-backed SpaceRole

*Deployment pattern, not protocol feature.*

The enrollment service verifies a credential issued to the user's bound DID
and assigns a `SpaceRole` whose `permissions` include the administrative
names it should confer: `manage_members`, `manage_roles`,
`manage_space`, `ban`, or `manage_channels` (`draft-atwood-jmap-chat-00`
§SpaceRole). On the wire this is an ordinary `updateMembers` or
`addMembers` entry carrying `roleIds` (§Updating a Space). This is the
"delegation to an external IdP or directory" model in §Space Governance:
Deployment Variation, with the credential as the directory.

Notes:

- **Who performs the assignment.** `updateMembers` role changes need
  `manage_roles`; `addMembers` needs `manage_members` (§Updating a
  Space). The enrollment service needs an account holding those, or a
  controller principal, whose designation is deployment-defined (§Space
  Permission Resolution).
- **Hierarchy limits on role assignment are unclear.** Servers enforce
  that a member creates or edits only roles below their own highest
  position, on `addRoles` and `updateRoles` (§Space Permission
  Resolution). The §Space/set rationale says adding a member "checks
  role-hierarchy constraints", but the `addMembers` definition states no
  such rule, and the draft says nothing about role changes through
  `updateMembers`. An operator that wants
  credential-backed roles to be assignable only by the enrollment service
  can add that restriction under the deployment-policy latitude in
  §Space Permission Resolution.
- **One role per credential type.** Mapping each credential type to one
  dedicated `SpaceRole` keeps revocation simple: removing the credential
  means removing that one role id (Section 6).
- **Peers see the role, not the reason.** Other servers and clients see
  `SpaceMember.roleIds` and the role's permissions. They cannot verify
  that a credential backed the role.

---

## 5. Membership via VC

*Deployment pattern, not protocol feature.*

Membership enters a Space in three ways only: `Space/set` `addMembers`,
`Space/join` with an invite code, and `Space/join` by `spaceId` to a public
Space (`draft-atwood-jmap-chat-00` §Updating a Space, §Space/join). The
drafts have no join request, approval step, pending state, or way to
attach a credential to either method.

**Pattern A: verify, then add.** The enrollment service verifies the
credential, then a `manage_members` holder adds the user with
`addMembers`, optionally with `roleIds` taken from the credential's claims.
This is the cleanest pattern because no unverified party ever holds a way
in.

**Pattern B: verify, then hand out a single-use invite.** The enrollment
service verifies the credential, creates a `SpaceInvite` with `maxUses`
of 1 and a short `expiresAt` (§SpaceInvite/set), and delivers the code to
the verified holder over its own channel. Limits:

- An invite code is a bearer token. Whoever redeems it joins. The
  credential check is only as strong as the channel that delivers the
  code.
- `SpaceInvite` has no role property (§SpaceInvite). `Space/join` says the
  joiner gets only `@everyone` "unless the invite specifies otherwise"
  (§Space/join), but an invite has no field to specify otherwise. Roles
  from the credential are assigned afterwards with `updateMembers`.

**What the operator cannot do at join time.** `Space/join` lists its
errors: `invalidArguments` for bad arguments or a bad invite,
`notPermitted` for a non-public Space, and `forbidden` for an active ban
or a closed Space (§Space/join). The draft lists no refusal for a missing
credential. Keeping the Space non-public (`isPublic` false, the default;
§Space) and using Pattern A or B avoids needing one.

**Removing a member** uses `removeMembers` and needs `manage_members`.
Servers are expected to refuse a removal that would leave no member
holding `manage_members` or `manage_space`, and that refusal is an
`invalidProperties` SetError (§Updating a Space). A ban (`SpaceBan`) keeps
a removed member from rejoining (§SpaceBan, §Space/join).

---

## 6. Expiry and revocation

*Deployment pattern, not protocol feature.*

No draft says anything about credential status. JMAP DID's lifecycle
covers suspension and revocation of the DID binding itself
(`draft-atwood-jmap-did-00` §Suspension, §Revocation). A revoked
credential and a revoked DID are separate events, and the drafts connect
neither to Space roles.

**Status mechanisms by format.**

| Format | Status mechanism | States |
|---|---|---|
| W3C VC 2.0 | `credentialStatus` entries; the examples use `BitstringStatusListEntry` (VC Data Model 2.0 §4.10) | Bitstring Status List v1.0 purposes: `revocation`, `suspension`, `refresh`, `message` |
| SD-JWT VC | `status` claim, defined by the Token Status List draft (draft-ietf-oauth-sd-jwt-vc-19 §2.2.2.3) | `VALID`, `INVALID` (revoked), `SUSPENDED` (draft-ietf-oauth-status-list-21 §7.1) |
| ACDC | Transaction Event Log (TEL) registry named by `rd`, anchored in the issuer's KEL (ACDC §Registry SAID Field) | Issued or revoked; with no registry, issued only |

Validity windows: VC 2.0 has optional `validFrom` and `validUntil` (§4.9
Validity Period). SD-JWT VC has optional `nbf` and `exp`, which cannot be
selectively disclosed (§2.2.2.3). ACDC has no dedicated expiry field; any
expiry is defined by the credential's schema (ACDC §Reserved field
labels).

**What happens to roles.** Nothing happens unless the operator acts. The
server granted the role from its own check, so it has to re-check on its own
schedule: at each session start, periodically, or both. When a check
fails, the operator removes the credential-backed role with
`updateMembers`, or removes the member with `removeMembers`
(`draft-atwood-jmap-chat-00` §Updating a Space). If the role being
removed is the Space's last administrator, the last-administrator refusal
on `removeMembers` applies (§Updating a Space). The draft does not say
whether the same protection applies to a role change through
`updateMembers`. A Space whose last `manage_space` holder leaves while
others remain is closed (§Updating a Space, §Space).

Suspension maps naturally to removing the role and restoring it later.
Revocation maps to removing the role, and, if the person should not
rejoin by other means, to a `SpaceBan`.

**What federation peers see.** Space-level changes are enforced by the
Space-hosting server at delivery time and are not proactively federated
(`draft-atwood-jmap-chat-federation-00` §Space-Level Federation). A remote
member whose role or membership was withdrawn finds out through a rejected
delivery or a later `Space/get`. If they were removed, their next
`Peer/deliver` into the Space is rejected at the membership check (step 3;
the draft names no error code for it). If they remain a member but lost
the role, it fails with `forbidden` at the `send` check (step 5). The
silent drop for a ban (step 4) applies only to a sender who is still a
member (§New Message Validation). Peers never learn
that a credential was the reason.

The core draft says `addMembers` triggers `Peer/groupUpdate` in federated
deployments (`draft-atwood-jmap-chat-00` §Space/set), but `Peer/groupUpdate`
is defined only for group chats, and the federation draft says Space-level
operations are not proactively federated
(`draft-atwood-jmap-chat-federation-00` §Peer/groupUpdate, §Space-Level
Federation). The two drafts disagree. This guide relies on the federation
draft's statement.

---

## 7. Credential formats compared

| | W3C VC 2.0 | SD-JWT VC | ACDC |
|---|---|---|---|
| Holder binding | `credentialSubject.id` can be a DID (§4.8). The data model leaves proof of holder control to the securing mechanism (§4.13) | Issuer puts the holder key in `cnf`; holder signs a Key Binding JWT with `aud`, `nonce`, `iat`, `sd_hash` (RFC 9901 §4.1.2, §4.3). `sub` is optional and need not match `cnf` (sd-jwt-vc-19 §2.2.2.3) | Issuee AID in the attribute section's `i` field (ACDC §Attribute Section Variants); control of the AID proves holding |
| Selective disclosure | None in the data model; comes from the securing mechanism, for example `ecdsa-sd-2023` or SD-JWT (§8.9) | Salted Disclosures with digests in `_sd`; decoys supported (RFC 9901 §4.2) | Graduated disclosure: compact, partial, selective, and others; blinding with `u` (ACDC §Aggregate Section) |
| Status | `credentialStatus`; Bitstring Status List (§4.10) | Token Status List (sd-jwt-vc-19 §2.2.2.3) | TEL registry (ACDC §Registry SAID Field) |
| Presentation | Verifiable Presentation, short-lived and bound to a verifier challenge; how depends on the securing mechanism and transport (§4.13) | Issuer-signed JWT plus chosen Disclosures plus KB-JWT, joined by `~` (RFC 9901 §4.3) | IPEX `exn` messages: apply, offer, agree, grant, admit, spurn (ACDC §IPEX, marked non-normative) |
| Spec status | W3C Recommendation, 15 May 2025 | draft-ietf-oauth-sd-jwt-vc-19, submitted to the IESG, not an RFC. Builds on RFC 9901 | ToIP KSWG specification v1.1 |

Notes:

- **W3C VC 2.0.** Two securing families: enveloping (VC-JOSE-COSE, a W3C
  Recommendation) and embedded (VC Data Integrity 1.0, a W3C
  Recommendation of 15 May 2025) (§4.12). The EdDSA and ECDSA
  cryptosuites are Recommendations. The BBS cryptosuite, which offers
  unlinkable selective disclosure, is still a Candidate Recommendation
  Draft. VC Data Model v2.1 is a Working Draft (30 September 2026). This
  guide did not survey how each securing mechanism proves holder control.
- **SD-JWT VC.** Media type `application/dc+sd-jwt` (§2.1). This differs
  from the `vc+sd-jwt` type VC-JOSE-COSE uses for JSON-LD credentials.
  Issuer keys come from JWT VC Issuer Metadata or an `x5c` chain (§2.5).
  SD-JWT VC itself defines no DID-based holder binding or issuer key
  mechanism; its changelog says a DID-based issuer mechanism is possible
  via profile or extension, and §2.5 lets other specifications or
  ecosystem rules add key mechanisms. OpenID4VCI 1.0, which profiles
  SD-JWT VC as `dc+sd-jwt` (Appendix A.3), lets an issuer bind the
  credential at issuance to a key named by a DID URL (the `jwt` key
  proof's `kid`). How that key appears in `cnf`, for example as a `jwk`
  copied from the DID document or as a `kid`, is left to the profile.
- **ACDC.** ACDCs are not signed directly. They are anchored to the
  issuer's key state, so issuer key rotation does not force revocation
  (ACDC §Binding to Key State at Time of ACDC State Change). Issuer and
  issuee are KERI AIDs, not arbitrary DIDs. `did:webs` carries an AID as
  a DID (did:webs §Method-Specific Identifier), so a JMAP account bound to
  a `did:webs` DID can be an ACDC issuee. No JMAP draft lists `did:webs`
  among its mandatory methods (`draft-atwood-jmap-chat-did-00` §Mandatory
  DID Methods); a server would advertise it in `supportedMethods`
  (`draft-atwood-jmap-did-00` §Session-Level Capability Object).

For a JMAP deployment, the format choice matters at one point only: what
the enrollment service can verify. Nothing about the choice reaches the
wire.

---

## 8. Worked example: the liaison room (hypothetical)

This continues the scenario in `jmap-toip-mapping-guide.md` Section 5.
Three standards bodies, called IETF, ASF, and W3C here, each appoint one
liaison. Each liaison runs their own mailbox, one peer domain with one
`ownerUserId`: `liaison.ietf.example`, `liaison.asf.example`, and
`liaison.w3c.example`. `liaison.ietf.example` hosts the liaison Space.

**Agreed out of band.** The three bodies agree on one credential format,
on each body's issuer identifier, and on two credential types: "appointed
liaison" and "room administrator". Nothing on the wire records the
agreement (`draft-atwood-jmap-chat-00` §Space Governance: Deployment
Variation).

**Issuance.** Each body issues an "appointed liaison" credential to the
DID bound to its liaison's mailbox account (`draft-atwood-jmap-did-00`
§Account-to-DID Binding). Each mailbox reports `keyCustody` in its own
Session (`draft-atwood-jmap-did-00` §Account-Level Capability Object). The
other two bodies learn it only if the operator shares it out of band or
exposes it in a peer-role Session, which no draft defines; with it, they
can treat a `"server"` value as weaker holder proof (Section 2).

**Space creation.** `liaison.ietf.example` has one owner, and only that
owner can call methods on it (`draft-atwood-jmap-chat-00`
§Authorization). IETF gated that account when it appointed the liaison.
There is no separate create gate to configure (Section 3).

**Roles.** The IETF liaison creates two roles: `liaison` (`view`, `send`)
and `room-admin` (`manage_members`, `ban`). IETF's enrollment service
checks each liaison's presentation against the agreed issuer list and the
credential's status, and the IETF liaison assigns `liaison` with
`addMembers` (Section 5, Pattern A). If ASF's credential set includes
"room administrator", the ASF liaison also gets `room-admin` with
`updateMembers` (Section 4).

**What ASF and W3C see.** Through `Space/get` as members they see the
role names, permissions, and member list (`draft-atwood-jmap-chat-00`
§Space/get). They do not see the credentials and cannot re-verify that
IETF checked them. They trust IETF to, just as they trust IETF to apply
role and ban policy (`jmap-toip-mapping-guide.md` Section 5).

**Revocation.** ASF replaces its liaison and marks the old credential
revoked in its status list. Nothing tells `liaison.ietf.example`. Either
IETF's next status re-check catches it, or ASF tells IETF directly. The
IETF liaison then removes the old member and adds the new one. The old
liaison's next `Peer/deliver` into the channel fails the membership check
(`draft-atwood-jmap-chat-federation-00` §New Message Validation).

**What JMAP supplies here.** Authenticated peers, bound identifiers, and
server-enforced Space rules. The credential check, the issuer list, and
the status re-check all live with IETF as the hosting operator.

---

## 9. Non-goals

- **No wire-level VC presentation.** No draft defines a method, argument,
  or property that carries a credential or presentation. That is the
  possible future capability in `draft-atwood-jmap-did-00`
  §Relationship to Verifiable Credentials, not something this guide adds.
- **No mandated format.** Section 7 compares formats; it does not choose
  one.
- **No trust-registry protocol.** Issuer trust is operator policy. Using a
  registry is the operator's choice (`jmap-toip-mapping-guide.md`
  Section 4.3).
- **No cross-server credential verification.** A receiving server cannot
  check that a credential backed a remote member's role. The drafts
  define no signed Space permission state that could carry such a claim
  (`draft-atwood-jmap-chat-federation-00` §Broadcast Mention Resolution).

---

## Cross-references

| Topic | Read also |
|---|---|
| DID binding, `keyCustody`, VC relationship, DID lifecycle | `draft-atwood-jmap-did-00.md`, `jmap-did-guide.md` |
| Mandatory DID methods, `ChatContact.did` | `draft-atwood-jmap-chat-did-00.md`, `jmap-chat-did-guide.md` |
| Space roles, members, invites, bans, governance variation | `draft-atwood-jmap-chat-00.md` |
| Space-level federation, message validation | `draft-atwood-jmap-chat-federation-00.md` |
| Federation allowlist policy | `jmap-chat-federation-guide.md` |
| ToIP mapping and the liaison scenario | `jmap-toip-mapping-guide.md` |
| W3C Verifiable Credentials Data Model v2.0 | https://www.w3.org/TR/vc-data-model-2.0/ |
| W3C Bitstring Status List v1.0 | https://www.w3.org/TR/2025/REC-vc-bitstring-status-list-20250515/ |
| W3C VC-JOSE-COSE | https://www.w3.org/TR/vc-jose-cose/ |
| W3C VC Data Integrity 1.0 | https://www.w3.org/TR/2025/REC-vc-data-integrity-20250515/ |
| SD-JWT (RFC 9901) | https://www.rfc-editor.org/rfc/rfc9901 |
| SD-JWT VC | https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/ |
| Token Status List | https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/ |
| ToIP ACDC specification v1.1 | https://doi.org/10.5281/zenodo.18792085 |
