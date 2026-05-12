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

Document your choice in your deployment's user-facing security documentation.
Users should not have to read your source code to know whether their account
holds an irrevocable governance role.

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

Creator auto-receives the `"admin"` role. Block admin-less groups (the last
admin must promote a successor before leaving, or the group is destroyed).
Provide an out-of-band promotion flow for cases where the creator's account
is lost or abandoned.

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

Define a server-admin role with admin-equivalent authority on all Spaces and
group chats. Audit all server-admin actions to a separate log that is not
exposed via the JMAP Chat wire protocol. Do not expose server-admin status
to chat participants beyond the action attribution.

If you integrate with an external IdP for organizational role-based admin
authority, make that integration optional and clearly scoped. Test that the
JMAP Chat wire surface remains identical whether or not the IdP is wired up.

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

Exempt server admins and dedicated moderator roles from slow-mode by
default. Selectively exempt automated systems that emit important
notifications (CI, on-call alerts) and rate-limit those that could spam
(general-purpose chatbots). Do not exempt federated `Peer/deliver`
messages at the receiver side; the rate limit applies at the original
sender's server.

Do not visually tag exempt senders unless your product has a specific
trust-and-safety reason to make moderator status visible. The audit log
records the action; the chat UI does not need to.

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

Start with the spec defaults. Don't substitute mappings unless you have a
concrete reason — the spec's recommended mapping captures decades of chat-UX
convention.

If you need richer authorization (quorum, IdP, time-limited), layer it on top
of the spec defaults rather than replacing them. The wire contract is
preserved either way; the additional checks just narrow the set of requests
that succeed.

Document deviations in your deployment's API documentation. Users seeing
`forbidden` on actions the spec says should work need to be able to find out
why.

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
"manage_emoji" mapping per §2.1) can add and manage Space-scoped emoji.
Members can always destroy their own contributions.

Use archival rather than destruction: mark deprecated, prevent new reactions
from using it, preserve existing reactions with a placeholder icon. The
storage cost is bounded by your retention policy.

Quotas: 100 emoji per Space, 256 KB per emoji image. Tune based on storage
budget. Reject `CustomEmoji/set` with `overQuota` (per `{{RFC8620}}` §5.3)
when the count or size limit is exceeded.

Server-level emoji: restrict to a dedicated server-admin role (§1.3). The
server name appears in the emoji context; user-facing implications are
larger than for Space-scoped emoji.

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
  Exclude non-member admin-equivalent principals (§1.3) from the `@admins`
  recipient set — they receive notifications via separate audit channels.
- Evaluate at delivery time on each receiving server using that server's
  local view of presence and roles.
- Document the predicate in your deployment's user-facing documentation so
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

- Time-ordered IDs (`Chat.id`, `Message.id`, `senderMsgId`): ULID.
- Non-time-ordered IDs: ULID, for consistency with time-ordered IDs and
  ecosystem familiarity. Use a different scheme only if you have a concrete
  reason (e.g., integrating with an existing ID issuer that already produces
  KSUIDs).
- `ChatContact.id`: whatever your auth layer returns. Document the format
  (regex or example) in your deployment's API reference. Peer servers
  validate against this format when checking the identity-binding
  constraint at §`{{peer-authentication}}` of the federation draft.

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

Treat DID URIs as opaque `ChatContact.id` values. No special handling. This
matches q15f Option B and keeps you spec-conformant without committing to
DID infrastructure.

If your user base demands real DID interop, follow `8sgn` rather than
rolling custom resolution. The companion draft's design questions are
already enumerated in `8sgn`'s description.

Document acceptance criteria in your deployment's API reference: which DID
methods (if any) you treat specially, which you accept opaquely, which you
reject.

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

Composer parser: recognize `@user@host` and `@did:...` forms. Tolerate
Unicode in the user-part; normalize IDN hosts to punycode. Treat
malformed textual forms as plaintext rather than failed mentions (less UX
friction).

Resolution: try local `ChatContact` lookup first (by exact match on the
resolved candidate id). On miss, probe `/.well-known/jmap` at the host to
discover the peer's `ownerUserId` and validate the resolution. Cache
successful resolutions for the session.

Auto-create policy: create a new `ChatContact` record only on the sender's
explicit send action (not on every keystroke as the textual form is typed).
Rate-limit auto-creation per sender per Space per hour. Cap total
ChatContact growth per Space.

Document the parser and resolution behavior in your deployment's
client-side docs so users understand what mentions are valid and how to
type them.

---

## 4. Storage and retention

The spec mandates durability **outcomes** (messages survive local restart
before delivery confirmation; ChatContact records persist) but leaves the
**mechanisms** to deployment. Retention policies for editable content are
similarly impl-defined.

### 4.1 Outbox durability mechanism

*(Stub — fill in.)*

### 4.2 Edit-history retention policy

*(Stub — fill in.)*

### 4.3 Federation contact resolution caching

*(Stub — fill in.)*

---

## 5. Rate limits and timing

The spec specifies a few hardcoded values (3-second typing rate, 10-second
client decay) and leaves others impl-defined (federation outbound presence
rate, receiveTypingIndicators cache TTL, retry/backoff intervals). Even the
hardcoded values are recommended defaults with documented calibration; you may
deviate with reason.

### 5.1 Typing rate and decay calibration

*(Stub — fill in.)*

### 5.2 Federation Peer/presence outbound rate

*(Stub — fill in.)*

### 5.3 `receiveTypingIndicators` cache TTL

*(Stub — fill in.)*

### 5.4 Retry and backoff intervals

*(Stub — fill in.)*

---

## 6. Privacy and suppression

The spec defines several privacy-protective behaviors (blocked-sender
suppression, `receiveTypingIndicators`, receipt-sharing opt-out) and leaves
deployment-level controls open (broadcast-mention suppression preferences,
DND-style escape hatches). Implementations choose how much per-user control
to expose.

### 6.1 Broadcast-mention suppression and `Chat.muted` bypass

*(Stub — fill in.)*

### 6.2 Receipt-sharing scope and granularity

*(Stub — fill in.)*

### 6.3 Blocked-sender ephemeral-event suppression

*(Stub — fill in.)*

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
