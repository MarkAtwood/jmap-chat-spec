# JMAP Chat — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-chat-00`. Covers the
security, governance, and operational posture decisions that the spec
deliberately leaves to implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what the spec *deliberately leaves open* and what implementations must
decide before shipping.

---

## How to read this guide

The JMAP Chat main draft is intentionally minimal about deployment policy. It
defines wire formats, the closed permission vocabulary, and the receiver-side
authorization model — and stops there. Many things a real chat product needs
(who starts as admin, how long edit history is retained, what counts as
"active" for `@here`, how `@user@host` mentions get resolved, etc.) are
deferred to deployment policy.

This is not a free pass. An implementation that ignores a deferred decision
will not interoperate well, will produce surprising UX, or will leak
information through inconsistent behavior. Implementations must make each of
these decisions explicitly, document them, and implement them coherently.

Each section below follows the same shape:

1. **What the spec leaves open** — with a draft citation, so you can read the
   normative text yourself.
2. **What you must decide** — the concrete deployment choice you cannot avoid.
3. **Considerations** — the trade-offs that inform the choice.
4. **Common patterns** — how production chat systems handle this.
5. **Recommended starting point** — a defensible default. Not normative; you
   may choose otherwise with good reason.

When two sections interact (for example, governance choices feed into
authorization mappings), cross-references are explicit.

This guide is non-normative. The drafts are the source of truth. Where this
guide and a draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of
  truth. Where this guide describes a spec requirement using a keyword, the
  keyword reflects the spec's normativity; if guide and draft disagree, the
  draft wins.
- Where this guide uses a keyword for an operational practice, UX default,
  or deployment choice (e.g., "servers SHOULD log admin actions"), the
  keyword reflects implementer best-practice. Deviation does not affect
  protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. Governance and roles

The main draft defines roles, permissions, and the role-position hierarchy as
wire contract. It does not prescribe **who starts privileged**, **whether
privilege is removable**, or **how it is transferred**. These are deployment
choices that profoundly shape product feel.

### 1.1 Space governance: controller principals

**What the spec leaves open.** §Space Governance: Deployment Variation
(`draft-atwood-jmap-chat-00.md`, {#space-deployment} anchor) explicitly
declines to pick a governance model. Two patterns are documented:
single-irreducible-controller (Discord/Slack style — the controller cannot be
removed by other members) and no-controller-role-graph-only (governance lives
entirely in the role hierarchy; if administrative coverage is lost, recovery
is out-of-band). The main draft also mentions, at the §Space Permission
Resolution section, that servers MAY designate one or more controller
principals that bypass the permission graph and have all permissions
implicitly.

**What you must decide.**

- Whether your deployment has a controller principal concept at all.
- If yes: how a controller is designated, how the role transfers, how it is
  revoked (or whether it can be).
- Whether the controller is subject to role-hierarchy enforcement (can be
  demoted by another controller? can be removed from the Space?).
- How your governance model interacts with federation (your controller is not
  recognized as controller on a peer server unless the peer's policy
  coordinates with yours).

**Considerations.**

- *Single-irreducible-controller* is simple and prevents accidental admin
  coverage loss, but creates a single point of governance failure. If the
  controller's account is compromised, lost, or abandoned, recovery requires
  an out-of-band mechanism.
- *Role-graph-only* is the most decentralized model. It maps cleanly to the
  spec's wire contract (no special server-side carve-out) but requires
  out-of-band recovery procedures when no member holds `"manage_space"`.
- *Multi-controller* (N-of-M approval for administrative actions) is a middle
  ground; the spec doesn't preclude it but you implement quorum logic
  server-side outside the wire protocol.
- *Federation:* Your controller has no special standing on a peer server. If
  your deployment federates, document that governance is per-server and that
  controller status does not propagate across the federation boundary.

**Common patterns.**

| System | Pattern |
|---|---|
| Discord | Irreducible server Owner; can transfer but not be removed; not subject to role hierarchy. |
| Slack | Workspace Primary Owner cannot be demoted by others; transfer requires admin action. |
| Mattermost | System Admin role at server level, not Space level; admin-equivalent on all teams. |
| Matrix | Power level system; creator default PL 100; can be demoted by another PL 100 user. |
| IRC | Channel operator status; loose; no irreducible controller. |

**Recommended starting point.**

For consumer or social products: single-irreducible-controller with a
server-managed transfer flow. Familiar UX, minimal lock-out risk.

For enterprise products: a System Admin override above a Space-level Owner
role, plus a transfer flow. The System Admin handles compliance and recovery;
the Space-level Owner handles day-to-day governance.

For decentralized or federated products: role-graph-only with a clearly
documented out-of-band recovery procedure (typically: server operator can
manually grant `"manage_space"` to a member if all admins are lost).

Deployments SHOULD document their governance choice in user-facing security
documentation. Users SHOULD NOT have to read source code to know whether
their account holds an irrevocable governance role.

### 1.2 Group-chat bootstrap role assignment

**What the spec leaves open.** Group chats use a closed two-role enum
(`ChatMember.role`: `"admin" | "member"`). Commit `2fd3cf7` made the
bootstrap-role assignment server-defined: the creator does not *have* to
receive `"admin"` automatically. The wire contract is the role enum and the
admin-action list, not the initial-seat policy.

**What you must decide.**

- Whether the creator of a group chat automatically becomes admin.
- Whether non-creator members start as admin or member.
- What happens to a group chat that loses all its admins (no member holds
  `"admin"` after the last admin leaves or is demoted).

**Considerations.**

- *Creator-becomes-admin* is the simplest UX and matches user expectations
  from Discord, Telegram, WhatsApp. It does mean admin authority is tied to
  the original creator forever unless transferred.
- *Flat peer group (no admins)* makes every member equal. Administrative
  actions either don't exist (the group is fully consensus-driven) or are
  delegated to an out-of-band layer.
- *All-initial-admins* gives every member added at creation time the admin
  role; later additions get member only. Works well for small invited groups
  where trust is symmetric.
- *Promotion-only*: creator is just a member; promotion to admin happens via
  an out-of-band flow (server-level operator, organizational policy). Common
  in enterprise products with directory-driven role assignment.

**Common patterns.**

- WhatsApp / Signal: creator is admin; can promote others; admin-less groups
  are not allowed — the last admin must promote a successor or destroy the
  group.
- Slack: channel creator gets channel-management permissions; admin-equivalent
  actions are gated at workspace level rather than per-channel.
- Discord (DM groups): creator is admin; can add/remove others without group
  consent.
- Matrix: power level system from rooms; creator default PL 100.

**Recommended starting point.**

The creator SHOULD automatically receive the `"admin"` role. Deployments
SHOULD prevent admin-less groups by requiring the last remaining admin to
promote a successor before leaving (or by destroying the group). Deployments
SHOULD provide an out-of-band promotion flow for cases where the creator's
account is lost or abandoned.

For enterprise deployments: layer this with a System Admin override
(see §1.3) so that an organizational admin can recover an admin-less group
without destroying message history.

### 1.3 Additional admin-equivalent principals

**What the spec leaves open.** Commit `ic5s.4` added a paragraph
(`draft-atwood-jmap-chat-00.md` near {#chat-member} anchor) noting that the
`"admin" | "member"` role enum is the *wire-observable* representation, and
that servers MAY designate additional internal principals as having
admin-equivalent authority for the actions admins may perform. The means of
designating such principals is deployment-defined. From a remote peer's
perspective, all such actions appear to originate from a member with the
`"admin"` role on the originating server.

**What you must decide.**

- Whether to support principals not in a chat's member list having
  admin-equivalent authority (server admins, organizational moderators,
  automated systems, IdP-delegated principals).
- How each such principal is designated.
- How the principal's identity is reflected to chat participants when they
  take action.
- How actions are audited beyond the participant-visible record.

**Considerations.**

- *Server admin override* is the most common pattern: a small set of
  operator accounts can take admin actions on any chat without being
  members. Useful for moderation, compliance, content removal.
- *Organizational role-based*: anyone holding role X in your directory
  service automatically has admin-equivalent authority on Spaces tagged
  with a matching org id. Common in enterprise deployments.
- *Automated systems*: bots that pin policy messages, archive old chats,
  enforce retention rules. Treat their actions as admin-equivalent and
  audit them separately.
- *Audit and accountability*: when a non-member acts as admin, the
  participant-visible record (Peer/groupUpdate, Chat/set patches) shows the
  action as if from an admin member. Your audit log should record the
  actual principal so post-hoc review identifies who acted.
- *User trust*: users have a right to know whether non-members can take
  administrative actions in their chats. Document the policy in user-facing
  product copy.

**Common patterns.**

- Slack Enterprise Grid: organization-level admins act on any workspace; the
  action is visible as a system-attributed event.
- Discord: Discord staff can act on any server; the action attribution
  varies depending on the kind.
- Matrix: server-admin role acts on rooms; appears as a regular member in
  the wire view.
- Mattermost: System Admin role acts on all teams; the action is recorded
  in a separate audit log.

**Recommended starting point.**

Deployments SHOULD define a server-admin role with admin-equivalent authority
on all Spaces and group chats. Server-admin actions MUST be audited to a
separate log that is not exposed via the JMAP Chat wire protocol.
Deployments SHOULD NOT expose server-admin status to chat participants
beyond the action attribution.

If you integrate with an external IdP for organizational role-based admin
authority, that integration SHOULD be optional and clearly scoped. The JMAP
Chat wire surface MUST remain identical whether or not the IdP is wired up.

### 1.4 Slow-mode exemption beyond `manage_channels`

**What the spec leaves open.** Commit `ic5s.3` softened `slowModeSeconds`
(main draft, near {#chat} anchor) to require a `rateLimited` error for
non-exempt senders but make the exemption clause itself a SHOULD: servers
SHOULD exempt members holding `"manage_channels"`, and deployments MAY
define additional exempt principals.

**What you must decide.**

- Whether to exempt server admins (§1.3) from slow-mode.
- Whether to exempt dedicated moderator roles, automated systems, or
  IdP-delegated principals.
- Whether to expose exemption status to other chat members.
- How slow-mode interacts with messages arriving via federation
  (Peer/deliver from a peer's local sender).

**Considerations.**

- Moderators and server admins legitimately need to send urgent messages
  during high-activity periods. A strictly-enforced slow-mode that delays
  rule enforcement is worse than the rate limit's intent.
- Exempt principals that appear as ordinary members can create the
  appearance of unfair treatment ("why is X sending so often during
  slow-mode?"). Some products tag exempt senders visibly; others don't.
- Federated peers don't have visibility into your slow-mode setting. The
  rate limit applies at the local-sender level (`Chat/typing` and message
  creation on the originating server); incoming `Peer/deliver` messages
  arrive regardless of your local slow-mode. This is correct: the peer's
  local user is subject to *their* server's slow-mode, not yours.
- Bots and automated systems benefit from exemption when they post
  high-volume notifications (e.g., CI status, monitoring alerts) and from
  rate limiting when they could spam.

**Common patterns.**

- Discord: server staff with Manage Channels exempt; bots with the
  permission exempt. The chat UI does not visually tag exempt senders.
- Slack: channel managers exempt; similar UI behavior.
- Twitch: moderators and channel owner exempt; subscribers may have
  reduced rate limits compared to regular viewers.

**Recommended starting point.**

Deployments SHOULD exempt server admins and dedicated moderator roles from
slow-mode by default. Automated systems that emit important notifications
(CI, on-call alerts) SHOULD be exempt; general-purpose chatbots that could
spam SHOULD be rate-limited normally. Deployments MUST NOT exempt federated
`Peer/deliver` messages at the receiver side; the rate limit applies at the
original sender's server.

Deployments SHOULD NOT visually tag exempt senders unless the product has a
specific trust-and-safety reason to make moderator status visible. The
audit log records the action; the chat UI does not need to.

---

## 2. Authorization policy

The spec defines a closed permission vocabulary (`view`, `send`, `pin`,
`manage_channels`, `manage_members`, `manage_roles`, `manage_space`, `ban`,
`mention_all`) and a receiver-side authorization model. It defers the mapping
from those wire-level permission names to deployment-internal authorization
decisions, and several method-level "Requires X" clauses are explicitly
softened to recommended defaults.

### 2.1 Mapping deployment authorization to the wire permission vocabulary

**What the spec leaves open.** Commit `38055ec` added a paragraph to §Space
Permission Resolution making explicit that "Requires `X`" and "Mutable by
members with `X` permission" clauses describe *recommended* mappings from the
closed permission vocabulary to the actions those permissions gate.
Deployments MAY substitute or augment these mappings with deployment-specific
authorization policy — for example, gating an action on a different
permission, requiring additional authorization beyond merely holding the
named permission, or recognizing an internal capability not present in the
closed vocabulary. The wire-level contract is that unauthorized callers
receive `forbidden`; the permission names in `SpaceRole.permissions` retain
their stated meanings when returned to clients.

**What you must decide.**

- Whether to use the spec's default mappings unchanged or replace them.
- If replacing: what your deployment-specific authorization model looks like
  and how it maps onto the wire-observable permission names that clients
  see.
- Whether to support gradual migration (defaults plus overrides) or
  all-or-nothing replacement.
- How your model handles the carve-out MUST clauses that the latitude
  paragraph explicitly does NOT cover (sender-only constraints on
  `Message/set update` and Reaction operations, the mailbox-owner constraint
  from RFC 8620, and security input-validation requirements).

**Considerations.**

- The closed permission vocabulary is wire-visible. Even if your internal
  authorization differs, what you put in `SpaceRole.permissions` is what
  clients see and may use for pre-flight UX (e.g., greying out "Delete
  channel" if the current user lacks `manage_channels`).
- A deployment that gates "delete channel" on a different permission name
  than `manage_channels` confuses clients that pre-check UX state from the
  permission set.
- Adding *additional* authorization beyond holding the named permission
  (quorum, IdP claim verification, time-of-day policy) is fully
  spec-compliant. The wire contract is the `forbidden` response, not the
  in-server check.
- Document any deviations from spec defaults. Users and auditors need to be
  able to verify behavior without reading server source code.

**Common patterns.**

- *Spec defaults*: follow the per-method "Requires X" clauses as written.
  Predictable for clients; minimal divergence between deployments.
- *Two-key authorization*: certain destructive actions (Space destroy, role
  delete) require approval from two distinct admin members. The wire returns
  `forbidden` until quorum is reached.
- *IdP-driven overrides*: organizational policy determines effective
  authorization; deployments map IdP claims to internal authorization that
  gates the wire methods.
- *Role-graph supplementation*: keep the spec defaults but add additional
  impl-defined internal roles that grant subsets (e.g., a "moderator-lite"
  internal role with `manage_members` but not `manage_roles`).

**Recommended starting point.**

Deployments SHOULD start with the spec defaults. Mappings SHOULD NOT be
substituted without concrete reason — the spec's recommended mapping
captures decades of chat-UX convention.

If richer authorization is required (quorum, IdP, time-limited), it SHOULD
be layered on top of the spec defaults rather than replacing them. The wire
contract is preserved either way; the additional checks just narrow the set
of requests that succeed.

Deviations from spec defaults MUST be documented in the deployment's API
documentation. Users seeing `forbidden` on actions the spec says should work
need to be able to find out why.

### 2.2 Custom emoji authorization

**What the spec leaves open.** Commit `9344aec` removed the normative
`manage_emoji` permission and server-admin emoji authorization. Who may
create, modify, or destroy custom emojis at Space or server scope is
deployment-defined. The `CustomEmoji` data type and the
`CustomEmoji/get|set|query|changes|queryChanges` methods remain wire
contract; only the authorization is server-defined.

**What you must decide.**

- Whether all Space members can add custom emojis or only privileged
  members.
- Whether destruction is reversible (true delete vs archival).
- Whether one member may destroy another member's custom emoji (or only
  their own).
- Server-level emoji policy (the spec defines both Space-scoped and
  server-scoped CustomEmoji records).
- Quotas: maximum emojis per Space, maximum bytes per emoji, total bytes
  across the deployment.

**Considerations.**

- *Open-add* (any Space member): low friction; common in casual contexts;
  risk of abuse (offensive emojis, namespace squatting on names like
  `:approved:` or `:rejected:`).
- *Privileged-add* (specific role required): structured; more friction;
  common in enterprise contexts.
- *Destruction vs archival*: destroying an emoji affects all messages that
  reference it. Archival (mark deprecated, prevent new use, keep existing
  reactions intact) preserves chat history at the cost of indefinite
  storage.
- *Author-preserves-control*: the emoji creator can manage their own
  contributions but not others'. Strong precedent in Discord and Slack;
  matches user intuition.
- *Server-level vs Space-level scope*: server-level emoji carry your
  deployment's name as context; typically reserved for server admins.
  Space-level scope is delegated to Space admins.

**Common patterns.**

- Discord: server members holding "Manage Expressions" can add server
  emojis; members can manage their own additions; archived on deletion
  rather than purged.
- Slack: workspace admins manage workspace emoji; in some plans, all
  members can add; deletion is destructive and replaces reactions with a
  default icon.
- Matrix: no native CustomEmoji; per-room reaction policy is implicit.
- Telegram: sticker packs are user-published globally rather than
  Space-scoped.

**Recommended starting point.**

Space-level emoji: members holding `manage_channels` (or a deployment-internal
"manage_emoji" mapping per §2.1) SHOULD be permitted to add and manage
Space-scoped emoji. Members SHOULD always be able to destroy their own
contributions.

Deployments SHOULD use archival rather than destruction: mark deprecated,
prevent new reactions from using it, preserve existing reactions with a
placeholder icon. The storage cost is bounded by the retention policy.

Quotas: 100 emoji per Space, 256 KB per emoji image. Tune based on storage
budget. The server MUST reject `CustomEmoji/set` with `overQuota` (per
`{{RFC8620}}` §5.3) when the count or size limit is exceeded.

Server-level emoji SHOULD be restricted to a dedicated server-admin role
(§1.3). The server name appears in the emoji context; user-facing
implications are larger than for Space-scoped emoji.

### 2.3 Mention scope predicates (`@here` and `@admins`)

**What the spec leaves open.** Decision D4a of `fig9.1` (broadcast-scope
mention design) makes the predicate definitions deployment-defined:

- `@everyone`: all current members of the Space or Chat — fully deterministic.
- `@here`: "members with active presence" — which presence states qualify is
  server-defined.
- `@admins`: "members with administrative authority" — which permissions or
  roles qualify, and whether controller principals (§1.1) are included, is
  server-defined.

(Note: `fig9.2` implementation of broadcast-scope mentions is pending at the
time of writing. This subsection covers the predicate design once broadcast
mentions land.)

**What you must decide.**

- For `@here`: which `PresenceStatus.presence` values qualify (`"online"`
  only? `"online"` and `"away"`? include `"busy"`?). Exclude `"invisible"`
  and `"offline"`.
- For `@admins`: which permissions in the effective permission union count
  (`manage_space` only? any `manage_*`? a deployment-internal "moderator"
  role?). Whether controller principals from §1.1 are included.
- Whether to expose your predicate definition to clients (so a mention
  preview can render the expected recipient count accurately).
- Whether the predicate is configurable per-Space, per-deployment, or
  hardcoded.

**Considerations.**

- `@here` is a "soft" broadcast — recipients are notified more loudly than
  for a normal message but not as loudly as a direct `@user`. A restrictive
  presence definition reduces accidental over-notification at the cost of
  excluding users who were briefly away.
- `@admins` exists for admin escalation ("hey admins, please look at this").
  Including controller principals (the impl-defined privileged class from
  §1.1) matches user intuition: a controller has admin-equivalent authority.
  Including non-member admin-equivalent principals (§1.3) is also defensible
  but exposes their existence to chat participants more loudly.
- Resolution timing is delivery-time on each receiving server (per `fig9.1`
  D4b). Each server's predicate evaluates against its own local view. This
  means two servers may resolve `@here` to different recipient sets for the
  same message; the spec accepts this.
- Strict predicates (`@here` = online only) make `@here` reliable when it
  fires but cause frequent "I missed it because I was 'away'" complaints.
  Looser predicates (`@here` = online and away) reduce false-negatives at
  the cost of notification volume.

**Common patterns.**

| System | `@here` predicate | `@admins` (or equivalent) predicate |
|---|---|---|
| Slack | Active (excludes "away", DND) | No native scope; use role mention |
| Discord | Online and not DND | No native scope; use role mention |
| Microsoft Teams | All with active presence | Channel owner / team owner |
| Matrix | N/A (PL-based mention) | PL >= threshold |

**Recommended starting point.**

- `@here` predicate: `presence in ("online", "away")`. Generous enough that
  brief away-status periods don't drop users out; strict enough to exclude
  `"offline"`, `"invisible"`, and (debatably) `"busy"`.
- `@admins` predicate: members whose effective permission union includes
  `manage_space` OR who are controller principals on the originating server.
  Non-member admin-equivalent principals (§1.3) SHOULD be excluded from the
  `@admins` recipient set — they receive notifications via separate audit
  channels.
- The predicate MUST be evaluated at delivery time on each receiving server
  using that server's local view of presence and roles (per `fig9.1` D4b).
- Deployments SHOULD document the predicate in user-facing documentation so
  users understand who'll be notified by their `@here` or `@admins`.

---

## 3. Identity and federation

The spec treats `ChatContact.id` as an opaque string from the authentication
layer. Common forms include `user@host`-style identifiers, DID URIs, and
deployment-specific schemes. Federated mention textual forms add a parsing and
resolution layer that the spec leaves to deployment.

### 3.1 Identifier scheme

**What the spec leaves open.** Commit `uy1m.1` softened seven id-field
definitions (`SpaceRole.id`, `Category.id`, `Space.id`, `CustomEmoji.id`,
`SpaceBan.id`, `ReadPosition.id`, `PresenceStatus.id`) to "Opaque
server-assigned JMAP identifier" — no required encoding scheme. `Chat.id`,
`Message.id`, and `senderMsgId` retain explicit ULID requirements because the
spec relies on their lexicographic time-ordering for message retrieval and
unreadCount derivation. `ChatContact.id` is whatever the authentication layer
returns; the spec doesn't constrain its form.

**What you must decide.**

- For non-time-ordered IDs (the seven softened): which scheme (ULID, KSUID,
  Snowflake, opaque random / UUIDv4, prefixed type-tag IDs).
- For `Chat.id` / `Message.id` / `senderMsgId`: still ULID, but you choose
  the implementation.
- For `ChatContact.id`: whatever your auth layer returns; document the
  format so peer servers and clients know what to expect.

**Considerations.**

- *ULID*: 26 chars base32, lexicographic time-ordering, readable, monotonic
  within a millisecond. Default choice for most JMAP implementations.
- *KSUID*: 27 chars base62; similar to ULID; includes more random bits.
- *Snowflake*: 64-bit integer represented as string; requires worker-id
  coordination at scale; common in services with sharded ID generators.
- *UUIDv4 / opaque random*: 36 chars (hex with dashes); no ordering; safe
  default if you don't need time-ordering.
- *Prefixed type tags*: encoding the object type in the ID (e.g.,
  `space:01J3...`) helps debugging and prevents accidental cross-type
  references. Costs a few bytes per ID and ties IDs to type forever.
- *Bytes on the wire*: ULID is 26 chars; UUIDv4 is 36 chars. Across a busy
  chat product, the difference adds up.

**Common patterns.**

| System | ID scheme |
|---|---|
| Slack | Snowflake-style with type prefix (`C12345...` channel, `U12345...` user) |
| Discord | Snowflake; 64-bit numeric as string |
| Matrix | Opaque random with prefix and server suffix (`!ABCDE...:server.example`) |
| Most JMAP servers | ULID for everything |

**Recommended starting point.**

- Time-ordered IDs (`Chat.id`, `Message.id`, `senderMsgId`): MUST be ULIDs
  per the spec (the lexicographic time-ordering requirement is normative).
- Non-time-ordered IDs: SHOULD also be ULIDs, for consistency with the
  time-ordered IDs and ecosystem familiarity. A different scheme MAY be
  used if there is a concrete reason (e.g., integrating with an existing
  ID issuer that already produces KSUIDs).
- `ChatContact.id`: whatever the auth layer returns. Deployments SHOULD
  document the format (regex or example) in their API reference. Peer
  servers MUST validate against this format when checking the
  identity-binding constraint described in the federation draft's Peer
  Authentication Model section.

### 3.2 DID URI handling

**What the spec leaves open.** Commit `48b6a31` (q15f Option B) added a prose
sentence acknowledging that `ChatContact.id` may take any URI form, including
a Decentralized Identifier URI per W3C DID-Core. The core spec stops there:
no DID resolution, no auth integration, no new capability. A future companion
draft (`8sgn`) is filed but not yet written; it would define wire mechanics
for DID-based federation auth, resolution, and method support.

**What you must decide.**

- Whether your deployment accepts DID-shaped identifiers in `ChatContact.id`.
- If accepted: what (if anything) you do with them beyond treating them as
  opaque strings.
- Whether to implement DID resolution (DNS-based `did:web`, blockchain-based
  `did:plc`/`did:ion`, etc.).
- Whether to follow `8sgn` once it lands — and contribute to its design now
  if you have a stake.

**Considerations.**

- *Treat as opaque* (q15f Option B baseline): no special handling. DID URIs
  appear in `ChatContact.id` as long strings; the rest of the protocol
  works unchanged. Lowest implementation cost; no portability benefit.
- *Adding resolution*: lets your server fetch DID Document metadata, verify
  endpoint claims, support identity portability across server moves.
  Substantial implementation effort; most chat products don't bother.
- *`did:web`* is the easiest DID method: DNS-based, low complexity. If you
  decide to engage, start here.
- *`did:plc`* and *`did:ion`* are blockchain-derived; slower resolution,
  larger trust model, more dependencies.
- *Future-proofing*: if you build resolution infrastructure ad-hoc, you may
  end up with a custom flavor that doesn't match `8sgn`'s eventual
  specification. Either follow `8sgn` or accept the rework cost when it
  lands.

**Common patterns.**

- AT Protocol (Bluesky): DID-first identity (`did:plc:xxx`); resolution is
  mandatory.
- Matrix: discussed DID integration; not adopted as primary identity.
- Most JMAP and email-style products: no DID engagement.

**Recommended starting point.**

Deployments SHOULD treat DID URIs as opaque `ChatContact.id` values. No
special handling is required. This matches q15f Option B and keeps the
deployment spec-conformant without committing to DID infrastructure.

If user demand justifies real DID interop, deployments SHOULD follow `8sgn`
rather than rolling custom resolution. The companion draft's design
questions are already enumerated in `8sgn`'s description.

Deployments SHOULD document acceptance criteria in their API reference:
which DID methods (if any) are treated specially, which are accepted
opaquely, and which are rejected.

### 3.3 Federated mention textual form parsing and resolution

**What the spec leaves open.** Commit `5bfb16d` (f13f resolution) added a
paragraph to the Mention section noting that common composer-side textual
forms include `@user@host` (Mastodon style) and DID URI forms (e.g.,
`@did:web:alice.example`). Parsing the textual form into a candidate id, and
resolving that candidate to a known ChatContact, are deployment-defined.
Server-side validation of the resulting Mention (offset/length, ChatContact
existence) is unchanged.

**What you must decide.**

- Whether your client(s) recognize `@user@host` textual forms.
- Whether your client(s) recognize DID URI textual forms.
- Parser rules: what counts as the host part, how to handle subdomains,
  whether to accept Unicode in the user-part (IRI-style) or restrict to
  ASCII (IDN punycode).
- How a textual form resolves to a candidate `ChatContact.id`: local lookup
  first, `/.well-known/jmap` probe, organizational directory, etc.
- Whether to auto-create a `ChatContact` record on a successful resolution
  to a previously-unknown peer user.

**Considerations.**

- Composer-side parsing is purely UX. The wire format is structured Mention
  entries (offset/length + id) regardless of textual form. The receiver
  reads the structured Mention; it does not re-parse the body text.
- *Resolution latency*: typing `@alice@example.com` should ideally show a
  pending state and resolve before the user hits send. If resolution fails
  (peer unreachable, no such user), the client should let the user send
  anyway or downgrade the mention to plaintext.
- *Auto-creation*: a mention to a previously-unknown user implies the
  sender wants to communicate with that user. Auto-creating a `ChatContact`
  record (main draft `:810` permits this) lets the mention land as a real
  mention rather than a broken reference. Be defensive: rate-limit
  auto-creation per sender to prevent enumeration attacks.
- *IDN / IRI*: international domain names should be normalized to punycode
  (`xn--...`) for `ChatContact.id`; the displayed textual form can keep
  Unicode. Treat user-part Unicode carefully; the auth layer's normalization
  rules apply.
- *Anti-abuse*: a malicious sender that mentions hundreds of random users
  to force ChatContact creation is doing reconnaissance. Rate-limit. Cap
  per-Space new-contact-via-mention creation per hour.

**Common patterns.**

- ActivityPub: `@user@host`; Mastodon clients resolve via WebFinger or
  out-of-band; servers auto-create Actor records on first reference.
- Matrix: `@user:server` (Matrix-style; note the colon, not at-sign);
  resolution via `/.well-known/matrix` or homeserver discovery.
- Most JMAP Chat deployments: no formal pattern yet; this guide and `f13f`
  define the conventions.

**Recommended starting point.**

Composer parser SHOULD recognize `@user@host` and `@did:...` forms.
Implementations SHOULD tolerate Unicode in the user-part and normalize IDN
hosts to punycode. Malformed textual forms SHOULD be treated as plaintext
rather than failed mentions (less UX friction).

Resolution SHOULD try local `ChatContact` lookup first (by exact match on
the resolved candidate id). On miss, the implementation SHOULD probe
`/.well-known/jmap` at the host to discover the peer's `ownerUserId` and
validate the resolution. Successful resolutions SHOULD be cached for the
session.

Auto-create policy: a new `ChatContact` record SHOULD be created only on
the sender's explicit send action (not on every keystroke as the textual
form is typed). Auto-creation MUST be rate-limited per sender per Space
per hour. Total ChatContact growth per Space SHOULD be capped.

Deployments SHOULD document the parser and resolution behavior in their
client-side docs so users understand what mentions are valid and how to
type them.

---

## 4. Storage and retention

The spec mandates durability **outcomes** (messages survive local restart
before delivery confirmation; ChatContact records persist) but leaves the
**mechanisms** to deployment. Retention policies for editable content are
similarly impl-defined.

### 4.1 Outbox durability mechanism

**What the spec leaves open.** Commit `uy1m.2` softened the federation
draft's outbox section to express durability as an outcome rather than a
prescribed mechanism: "Outbound messages MUST be durable across local server
restart prior to delivery confirmation; the specific mechanism (persistent
outbox, write-through queue, transactional store with the local message-store
write, replicated in-memory queue backed by an upstream broker with
at-least-once delivery, etc.) is implementation-defined."

**What you must decide.**

- Which durability backing store you use for outbound messages.
- At-most-once vs at-least-once delivery semantics.
- Whether duplicate suppression is sender-side (don't retry once acked) or
  receiver-side (dedup by `senderMsgId`).
- Recovery on restart: replay all pending, or replay only what hasn't been
  acked.

**Considerations.**

- *Persistent queue* (Redis with persistence, SQL queue table): simple,
  well-understood, easy to operate. Most chat servers start here.
- *Transactional DB with message store*: the message-create transaction
  also creates the outbox entry; no separate state machine. Stronger
  consistency; requires the message store to be in a transactional DB.
- *Replicated broker* (Kafka, RabbitMQ with persistence, NATS JetStream):
  scales horizontally; adds operational complexity; useful if you're
  already running one for other purposes.
- *In-memory* queue with snapshot-on-graceful-shutdown: NOT durable across
  crashes. Don't use this for federation outbound — the spec MUST is real.
- *At-least-once* means receivers may see duplicates and must dedup. The
  federation `senderMsgId` is designed for this (federation draft `:209`).
- *At-most-once* end-to-end is hard to guarantee without coordinated acks;
  most chat systems accept at-least-once and rely on receiver dedup.

**Common patterns.**

- Most chat systems: at-least-once delivery with receiver-side dedup via a
  sender-assigned message id (mapped to `senderMsgId` in JMAP Chat
  federation).
- Some enterprise products: transactional DB approach for stronger
  consistency between message persistence and outbox visibility.
- Large-scale deployments: replicated broker with idempotent consumers;
  scales to millions of messages per minute.

**Recommended starting point.**

Deployments SHOULD use at-least-once delivery via `senderMsgId`-based dedup
at the receiver. This matches what the federation draft already specifies
at `:609` ("A message whose senderMsgId is already known for the given chat
at the receiving server MAY be silently discarded by that server").

Backing store: a persistent queue alongside the message database. If the
message store is already in a transactional DB (Postgres, MySQL, SQLite),
implementations SHOULD write the outbox row in the same transaction as the
message row — this gives transactional consistency without a separate state
machine.

A replicated broker SHOULD only be introduced if the deployment is already
operating one for other workloads. Adding Kafka just for chat federation is
operational overkill for most deployments.

### 4.2 Edit-history retention policy

**What the spec leaves open.** Commit `uy1m.5` made the MessageRevision push
on edit conditional on retention. The main draft edit procedure now reads:
"If the server retains edit history, push a MessageRevision onto editHistory
with the current body, bodyType, and current server time as editedAt."
Combined with the main draft `:546` retention paragraph ("Servers MAY limit
the number of retained revisions; if so, older revisions MAY be elided"), a
zero-retention deployment is fully spec-compliant.

**What you must decide.**

- Whether to retain edit history at all.
- If yes: how many revisions per message, how long.
- Whether retention is configurable per-Space, per-account, or fixed
  per-deployment.
- Privacy implications: edit history that survives is searchable evidence;
  users editing sensitive content may not realize it's retained.
- Compliance: some regulated environments require retention; others require
  destruction.

**Considerations.**

- *No retention*: simplest. Messages are mutable; editing leaves only the
  `editedAt` timestamp on the latest version. The `editHistory` field is
  always empty. Lowest storage cost; strongest user privacy.
- *Bounded retention* (N revisions, time-bounded): preserves recent edits
  for audit and "show edit history" UX; drops old ones to bound storage.
  Common balance point.
- *Full retention*: every revision kept forever. Valuable for moderation
  and audit; storage cost grows with edit frequency; raises privacy
  concerns.
- *Privacy*: a user who edits "Meet at 5pm" to "Meet at 6pm" probably
  doesn't care if both versions survive. A user who accidentally pastes a
  password and immediately edits to remove it absolutely does.
- *Compliance*: regulated industries (finance, healthcare, government) may
  legally require retention or destruction; layer your policy on top of
  these external requirements.
- *Client UX*: if you retain history, clients may render an "edited (see
  history)" affordance. If you don't, clients render just "edited" with no
  history.

**Common patterns.**

- Slack: workspace admins configure retention; default is full history for
  Free and Pro plans; Enterprise allows custom retention.
- Discord: edit history retained indefinitely; viewable in audit log for
  server staff.
- Signal: no edit feature historically; if added, ephemeral by design.
- Telegram: limited edit history retained for moderator review on public
  channels.

**Recommended starting point.**

Deployments SHOULD retain the last 5 revisions per message. Older revisions
SHOULD be elided per the spec's `:546` paragraph.

The retention policy MUST be documented prominently in the deployment's
user-facing TOS or privacy notice. Users editing sensitive content need to
know history is kept.

For compliance-driven deployments: additional retention rules SHOULD be
layered on top of the default (longer or shorter as required by
regulation).

Per-Space configuration: deployments MAY allow Space admins to set their
Space's retention within deployment-defined bounds (e.g., zero to
deployment-max). The wire protocol does not expose retention policy
directly; configuration is deployment-side.

If a deployment chooses zero retention, it MUST be explicit about that
choice in product copy. "Edited" without history is unusual for chat
products; users will assume the history is there unless told otherwise.

### 4.3 Federation contact resolution caching

**What the spec leaves open.** Commit `uy1m.3` softened the federation
draft's `receiveTypingIndicators` cache TTL guidance to remove the hardcoded
60-second value. The current wording: "Servers that cache a remote
participant's receiveTypingIndicators value SHOULD use a short TTL; the
specific value is implementation-defined." More broadly, every server that
caches peer-supplied data (Session objects, ChatContact attributes,
preferences) chooses its own freshness vs traffic trade-off.

**What you must decide.**

- TTL for cached remote `receiveTypingIndicators` values.
- TTL for `ChatContact` records derived from peer `/.well-known/jmap`
  Session objects.
- TTL for peer endpoint advertisements (`Session.ownerEndpoints`).
- Cache invalidation strategy: time-based only, or also on explicit refresh
  events.
- Whether to share caches across nodes in your deployment.

**Considerations.**

- Long TTL: less network chatter; staler state. A user who opts out of
  typing indicators may keep receiving them from peers until cache TTL
  expires.
- Short TTL: fresher state; more requests to peer servers. At scale, this
  matters.
- No cache: simplest; most overhead. Not realistic for production
  deployments.
- *Cache invalidation*: there's no federation-wide notification protocol
  for preference changes. A user changes `receiveTypingIndicators` on
  server A; server B has no way to learn this except by re-fetching
  (which is what cache TTL enforces). If you build a notification channel
  for proactive invalidation, document it in deployment-side
  configuration.
- *Cache poisoning*: peer servers are trust boundaries. A hostile peer
  could lie about a user's preferences. Treat peer-supplied cache data as
  advisory, not authoritative.

**Common patterns.**

- HTTP caching norms: `Cache-Control` style policies with revalidation.
  JMAP Chat doesn't define HTTP cache headers for these specific values
  but the same pattern applies.
- Per-conversation cache + on-demand refresh: most chat federation systems.
- Shared cache across nodes: useful at scale; adds invalidation complexity.

**Recommended starting point.**

- `receiveTypingIndicators` cache TTL: **60 seconds**. This matches the
  value previously hardcoded in the federation draft before `uy1m.3`
  softened it; deployments SHOULD treat it as a sensible default and tune
  based on volume and acceptable staleness window.
- `ChatContact` record cache from peer Session: **1 hour** with on-demand
  refresh when the user explicitly initiates a contact interaction.
- `Session.ownerEndpoints` cache: **1 hour**; the spec already treats these
  as ephemeral hints.
- Caches SHOULD be invalidated on explicit user action (e.g.,
  user-initiated "refresh peer info") in addition to time-based expiry.
- Federation caches SHOULD NOT be shared across nodes by default. Per-node
  caches are simpler to reason about; cross-node sharing SHOULD be added
  only when measured traffic justifies the complexity.

---

## 5. Rate limits and timing

The spec specifies a few hardcoded values (3-second typing rate, 10-second
client decay) and leaves others impl-defined (federation outbound presence
rate, receiveTypingIndicators cache TTL, retry/backoff intervals). Even the
hardcoded values are recommended defaults with documented calibration; you may
deviate with reason.

### 5.1 Typing rate and decay calibration

**What the spec leaves open.** Commits `ic5s.1` and `ic5s.2` reframed the
typing rate-limit and client-side decay timer as SHOULD recommendations with
explicit calibration rationale rather than MUST mandates. The main draft now
specifies a 3-second server-side rate-limit window and a 10-second client-side
decay window, with a calibration paragraph explaining why these values pair:
one accepted event per 3 seconds while typing means the receiver's decay timer
sees at least three events before the 10-second window expires.

The two values are calibrated against each other. Changing one without
adjusting the other produces incorrect behavior: a longer rate-limit window
with the same decay would clear the indicator while the sender is still
actively typing; a shorter rate-limit with the same decay would burn
unnecessary network and federation traffic.

**What you must decide.**

- Whether to use the recommended 3s / 10s pair or deviate.
- If deviating: adjust both values to maintain the calibration (the client
  decay should be at least 3x the server rate-limit window).
- Whether to expose the configured values to clients (e.g., via the
  account-level capability object) so multi-client UX stays consistent.
- How aggressively to enforce the rate-limit when senders exceed it
  (silently discard, error response, temporary lockout).

**Considerations.**

- The 3s rate is federation-wide via `Peer/typing` rate limits (federation
  draft `:395`). If you deviate locally, your federated peers still
  rate-limit at 3s; senders crossing the federation boundary will see the
  federated rate even if your local rate is different.
- The 10s decay is *client-side*. Different clients of the same account
  using different decay timers will look inconsistent ("desktop says still
  typing, mobile cleared it"). Pick one value per deployment and document
  it.
- High-latency environments (mobile networks, federated peers, congested
  links) benefit from longer windows. Low-latency (LAN, in-server) tolerate
  shorter.

**Common patterns.**

| System | Typing rate | Decay window |
|---|---|---|
| Slack | ~3 seconds | ~5 seconds (UI cleared on no event) |
| Discord | ~5 seconds | ~10 seconds |
| Signal | Per-message gesture (less rate-based) | ~6 seconds |

**Recommended starting point.**

Deployments SHOULD use the spec's 3s / 10s pair. The calibration is
well-documented in the spec text; deviating without measured reason adds
risk without obvious upside.

If deviation is necessary: deployments MUST maintain the 3x ratio (decay
≥ 3 × rate-limit window). Configured values SHOULD be exposed to clients
via the account capability object so multi-client UX stays consistent.
Deviations MUST be documented in the deployment's API reference.

### 5.2 Federation Peer/presence outbound rate

**What the spec leaves open.** Commit `uy1m.4` removed the hardcoded
30-second value from the federation `Peer/presence` outbound rate guidance.
The current wording: "Servers SHOULD rate-limit outbound `Peer/presence`
calls per subscriber; the specific rate is implementation-defined."

**What you must decide.**

- How frequently to push outbound `Peer/presence` updates per subscriber.
- Whether to batch presence changes (collapse multiple changes within a
  window into a single push).
- Whether the rate is uniform per subscriber or varies by relationship
  (e.g., active conversation partner gets faster updates than dormant
  contact).

**Considerations.**

- Presence is best-effort. Dropped updates are tolerable; the system is not
  cache-coherent.
- Fan-out cost scales with subscriber count. A user with 1,000 active
  presence subscribers triggers 1,000 outbound calls per presence change
  without rate-limiting.
- User experience: 30-second updates feel "reasonably current" for
  online/away/offline transitions; faster (5-10s) feels real-time; slower
  (60s+) feels stale.
- Federation cost is paid by your server; downstream servers benefit from
  your rate-limit without paying for it.

**Common patterns.**

- Most chat federation systems: 30-60 second outbound rate per subscriber.
- High-volume products: aggressive collapsing of frequent transitions
  (online→away→online) into a single update.
- Some products: no proactive presence push; subscribers poll on demand.

**Recommended starting point.**

Deployments SHOULD use **30 seconds per subscriber** — matches the value
previously hardcoded in the federation draft before `uy1m.4` softened it,
and matches the WSS-layer 30s rate at
`draft-atwood-jmap-chat-wss-00.md:253`. Consistent across federation and
WSS layers; subscribers don't see unexpected behavioral differences.

If tuning: implementations SHOULD NOT go below 10 seconds (federation
traffic explodes) and SHOULD NOT go above 5 minutes (presence becomes too
stale to be useful for typical UX).

Implementations SHOULD batch frequent transitions: if a user goes
`online → away → online` within the rate-limit window, only the most
recent state SHOULD be sent at the end of the window.

### 5.3 `receiveTypingIndicators` cache TTL

This topic is covered in §4.3 (Federation contact resolution caching).
The recommended starting point is **60 seconds**.

### 5.4 Retry and backoff intervals

**What the spec leaves open.** The federation draft `:601` says: "The
minimum initial retry interval, maximum retry interval, and total retry
duration are implementation-defined, but implementations SHOULD apply a
jitter factor to avoid synchronized retry storms from multiple servers."

**What you must decide.**

- Initial retry interval (how long after the first failure before retrying).
- Maximum retry interval (the cap as exponential backoff grows).
- Total retry duration (when to declare permanent failure and mark
  `deliveryState: "failed"`).
- Jitter factor (random variation to spread retries from synchronized
  failures).

**Considerations.**

- Initial too short: a temporarily-overloaded peer gets hammered.
- Initial too long: a transient network blip delays delivery longer than
  necessary.
- Maximum too low: backoff doesn't actually back off; peers under sustained
  load see no relief.
- Maximum too high: a recovered peer doesn't see retry traffic for a long
  time; messages sit longer than necessary in the outbox.
- Total too short: messages declared failed before peer recovery is
  reasonable; user-visible delivery failures multiply.
- Total too long: failed messages clog the outbox indefinitely; storage
  cost grows; users wonder why an obviously-gone peer's messages are still
  "pending".
- Jitter is critical at scale. Without jitter, every server in a deployment
  retries at the same exponential schedule when a peer goes down; recovery
  causes a thundering herd.

**Common patterns.**

| System | Initial | Max | Total | Jitter |
|---|---|---|---|---|
| Most email MTAs | 30s | 4 hours | 5 days | 10-20% |
| HTTP retry libs | 1s | 60s | 5 minutes | 10-30% |
| Most chat federation | 5s | 5 minutes | 24-48 hours | 20% |

**Recommended starting point.**

- **Initial retry**: 5 seconds.
- **Maximum interval**: 5 minutes (the backoff cap).
- **Total retry duration**: 24 hours (after which the message MUST be
  marked `deliveryState: "failed"`).
- **Jitter**: ±20% of the computed interval (multiplied by a uniform
  random factor in [0.8, 1.2]). Implementations MUST apply jitter per the
  federation draft `:601`.
- **Backoff schedule**: exponential with base 2, capped at the maximum
  (so 5s → 10s → 20s → 40s → 80s → 160s → 300s [cap] → 300s ...).

Deployments SHOULD tune the total duration based on user expectations: 24
hours is a reasonable balance between "the message has a chance to land"
and "the user shouldn't see it pending for days". Email's 5-day default is
too long for chat product UX.

The retry schedule SHOULD be documented in the deployment's API reference
so operators can predict retry behavior during incidents.

---

## 6. Privacy and suppression

The spec defines several privacy-protective behaviors (blocked-sender
suppression, `receiveTypingIndicators`, receipt-sharing opt-out) and leaves
deployment-level controls open (broadcast-mention suppression preferences,
DND-style escape hatches). Implementations choose how much per-user control
to expose.

### 6.1 Broadcast-mention suppression and `Chat.muted` bypass

**What the spec leaves open.** Decision D5 of `fig9.1` chose the D5-Impl
position: broadcast mentions (`@everyone`, `@here`, `@admins`) bypass
`Chat.muted` for targeted recipients by default; opt-out is deployment-defined
with no new wire field added. Per-scope opt-out (mute `@here` but not
`@everyone`) is explicitly deferred to a future bead.

(Note: `fig9.2` main-draft implementation of broadcast-scope mentions is
pending at the time of writing. This subsection covers the suppression model
once broadcast mentions land.)

**What you must decide.**

- Whether to expose a per-Chat or per-account opt-out from broadcast-mention
  escalation (so users can restore strict-mute behavior).
- Whether Space admins can configure broadcast-mention behavior at the Space
  level.
- How to surface the policy in your client UI (settings page, contextual
  hint when muting a Chat, explicit "always notify on @everyone" toggle).
- Whether to expose the default-bypass behavior to clients via the account
  capability so multi-client UX stays consistent.

**Considerations.**

- *Default bypass* matches user expectations from Slack and Discord. Users
  understand that `@everyone` is "loud" by design.
- *Strict mute as opt-out*: some users want `Chat.muted` to mute
  everything, including `@everyone`. Privacy-conscious users and those who
  have been targets of abuse particularly value this.
- *Workplace policy*: enterprise deployments may need administrative
  controls — e.g., "always allow @here from managers" overriding individual
  mute settings.
- *Wire contract is fixed*: whatever opt-out you build is local
  (deployment-side preference, client-side filter, etc.), not exposed via
  federation. Federated peers don't know your local preferences.
- *Per-scope granularity* (mute `@here` only) is feasible client-side or
  via deployment preferences but not via wire fields — `fig9.1`'s D5-sub
  decision deferred wire-level per-scope to a future bead.

**Common patterns.**

| System | Default | Opt-out |
|---|---|---|
| Slack | `@channel` bypasses DND by default | Per-channel "Suppress @channel notifications" toggle |
| Discord | `@everyone` and `@here` bypass mute | Server-level "Suppress @everyone and @here" toggle |
| Matrix | Mentions respect mute settings | Explicit elevation required for notifications |

**Recommended starting point.**

Default behavior: broadcast mentions to targeted recipients SHOULD bypass
`Chat.muted` and use the configured `mentionUrgency` per the push draft
(`draft-atwood-jmap-chat-push-00.md`). This matches Slack/Discord conventions
and respects the sender's intent.

Deployments SHOULD provide a per-account preference "Always honor mute,
even for @everyone". The default SHOULD be false. The preference SHOULD be
surfaced in the notification settings UI alongside other mute/DND controls.

For workplace deployments: Space-admin configuration MAY layer on top —
"@everyone can override mute for: [all members | members with admin role |
no one]". This MUST NOT be exposed via wire fields; it is deployment policy
expressed through the per-account preference's effective value.

The default-bypass behavior and opt-out controls MUST be documented in the
deployment's privacy notice. Users targeted by broadcast mentions need to
know whether their mute setting protects them.

### 6.2 Receipt-sharing scope and granularity

**What the spec leaves open.** The main draft defines two receipt-sharing
preferences: `PresenceStatus.receiptSharing` (account-level, default `true`)
and `Chat.receiptSharing` (per-chat override). The bidirectional rule
(main draft `:777`) means turning off your sharing also turns off your
visibility into others' read times. Deployments choose finer granularity,
defaults, and UI exposure.

**What you must decide.**

- Default value for new accounts (`true` for useful UX, `false` for stronger
  privacy by default).
- Whether to expose finer granularity (per-sender, per-contact-class,
  time-of-day) on top of account + per-chat.
- How to surface the bidirectional nature to users in your settings UI.
- How to handle conflicting preferences across federation (your user opts
  out; peer server may not propagate the opt-out preference).

**Considerations.**

- *Default true* (WhatsApp / iMessage style): more useful UX; readers see
  delivery confirmations and read receipts; some users feel surveilled.
- *Default false* (Slack / Signal style): more private by default; users
  who want receipt visibility opt in explicitly.
- *Bidirectional rule*: a user who wants to see others' read times but not
  share their own cannot get that — the spec is symmetric. Document this
  clearly.
- *Per-chat overrides*: useful for contexts where receipt sharing is
  expected (work chat) versus where it isn't (sensitive personal contact).
- *Federation*: receipt suppression at the sender's server stops outbound
  `Peer/receipt` calls (federation `:318`); receipt suppression at the
  receiver's server stops inbound `Peer/receipt` from being delivered to
  clients (federation `:329`). Both ends enforce; ` defense-in-depth.

**Common patterns.**

- Slack: read receipts disabled at protocol level; user-visible "seen by N
  members" only for some context types.
- Signal: read receipts opt-in per user.
- WhatsApp: blue ticks on by default; per-account toggle to disable.
- iMessage: read receipts on by default; per-conversation toggle.

**Recommended starting point.**

Default `PresenceStatus.receiptSharing: true` (WhatsApp/iMessage-style).
Deployments SHOULD provide a per-account toggle in settings to disable.

`Chat.receiptSharing` per-chat override SHOULD be exposed in the chat-info
pane. The default SHOULD be absent (inherit account-level); users MAY set
it explicitly for sensitive conversations.

The bidirectional rule MUST be documented in plain language: "Turning off
read receipts means you won't see when others read your messages either."

For federated deployments: implementations MUST trust the federation
suppression rules at both sender and receiver sides; implementations MUST
NOT bypass them with a "show unofficial read time" UI.

### 6.3 Blocked-sender ephemeral-event suppression

**What the spec covers.** Commits `dim0` (WSS) and `87qs` (corpus-wide)
made blocked-sender suppression for typing and presence events normative in
the WSS draft, main draft, and federation draft. The architectural property
is documented in the main draft Security Considerations ({#blocked-contacts}
anchor): typing and presence push events whose sending or referenced
ChatContact has `blocked: true` are dropped server-side before delivery to
any of the owner's clients, on any transport. The sender is not informed.

This is fully spec'd; what's left is implementation choices around how
blocking is surfaced and managed at the UI level.

**What you must decide.**

- How blocking is exposed in client UI (block button location, confirmation
  flow, "blocked users" list).
- Whether to distinguish *block* from *mute* in the UI (different
  affordances, different effects).
- Whether blocking is per-account (block this user globally) or per-Space
  (block this user only in this Space).
- Whether to ever surface "you've been blocked" status to the blocked
  party (almost universally: no — defeats the privacy property).
- How blocking interacts with shared Space membership (block someone you
  share a Space with: do you still see their messages in the Space, or
  does block override membership visibility?).

**Considerations.**

- *Block vs mute distinction*: mute is per-conversation suppression of
  notifications; block is identity-level shunning of incoming events.
  Most chat products distinguish; some collapse them.
- *Per-account vs per-Space block*: per-account is simpler; per-Space lets
  a user block a stranger in one public Space without affecting another.
  The spec models per-account (`ChatContact.blocked`) but doesn't preclude
  deployment-side per-Space lists layered on top.
- *Visibility of blocked status*: surfacing "you've been blocked" to the
  blocked party makes blocking hostile rather than protective. The spec's
  silent-suppression rule depends on the blocked party not knowing.
- *Shared Space membership*: if Alice blocks Bob and they share a Space,
  Alice still sees Space-level activity from Bob (messages in shared
  channels). The block applies to direct messages and ephemeral events,
  not full Space-membership invisibility. (Per the spec's blocked-contacts
  Security Considerations rule.)
- *UI affordance*: a clear "Block" action with a confirmation dialog
  explaining what blocking does. Avoid lossy euphemisms — "Mute" is for
  notifications; "Block" is for the relationship.

**Common patterns.**

- Most chat products: per-account block; silent to the blocked party;
  block-list visible to the blocker in settings.
- Some products: per-Space block for public-Space contexts (Discord
  per-server, Telegram per-group).
- Some products: surface "this user has restricted you" indirectly — a
  message sent to a blocking user appears as "delivered" but never as
  "read".

**Recommended starting point.**

Per-account block via `ChatContact.blocked: true`. Clients SHOULD confirm
via a dialog that explains: blocked users' messages are silently dropped;
blocked users see neither the user's typing indicators nor presence; the
user no longer sees theirs.

Implementations MUST NOT surface block status to the blocked party in any
form. This is the spec's privacy property; respecting it requires not
leaking via UI side channels either.

The block list SHOULD be accessible in user-facing settings (account
settings → "Blocked users") and easy to unblock from.

Shared Space membership: per the spec's rule, blocking does not exit
shared Spaces. The block-confirmation dialog SHOULD make this explicit so
users aren't surprised when they still see the blocked user in a mutual
Space.

Mute and block SHOULD be distinct UI affordances. Mute affects
notifications (per-chat suppression); block affects the identity-level
relationship. Implementations SHOULD NOT collapse them.

---

## Cross-references to existing guides

| If you are implementing... | Read also... |
|---|---|
| The WebSocket transport and ephemeral events | `jmap-chat-wss-guide.md` |
| Inline push payloads (`ChatMessagePush`) | `jmap-chat-push-platform-guide.md` |
| Platform-specific push delivery (FCM, APNs, WNS, etc.) | `jmap-push-platform-guide.md` |
| Federation between mailbox servers | `draft-atwood-jmap-chat-federation-00.md` |

This guide focuses on deployment-policy decisions for the core JMAP Chat
draft. Transport-specific, push-specific, and federation-specific
implementation details live in their dedicated guides.
