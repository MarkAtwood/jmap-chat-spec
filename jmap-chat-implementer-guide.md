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

*(Stub — fill in.)*

### 2.2 Custom emoji authorization

*(Stub — fill in.)*

### 2.3 Mention scope predicates (`@here` and `@admins`)

*(Stub — fill in.)*

---

## 3. Identity and federation

The spec treats `ChatContact.id` as an opaque string from the authentication
layer. Common forms include `user@host`-style identifiers, DID URIs, and
deployment-specific schemes. Federated mention textual forms add a parsing and
resolution layer that the spec leaves to deployment.

### 3.1 Identifier scheme

*(Stub — fill in.)*

### 3.2 DID URI handling

*(Stub — fill in.)*

### 3.3 Federated mention textual form parsing and resolution

*(Stub — fill in.)*

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
