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

*(Stub — fill in.)*

### 1.2 Group-chat bootstrap role assignment

*(Stub — fill in.)*

### 1.3 Additional admin-equivalent principals

*(Stub — fill in.)*

### 1.4 Slow-mode exemption beyond `manage_channels`

*(Stub — fill in.)*

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
