# JMAP Chat Federation — Implementer's Guide

For operators of mailbox servers that federate JMAP Chat with peers per
`draft-atwood-jmap-chat-federation-00`. Covers the operational and policy
posture decisions that the federation draft deliberately leaves open.

Read the draft first. This guide does not re-state normative requirements;
it covers what the spec leaves to deployments and offers patterns and
starting points for running federation as a service.

The federation draft itself defines the wire protocol: peer discovery,
the `Peer/*` method set, identity binding, outbox semantics, retry
behavior, and security rules. The main `jmap-chat-implementer-guide.md`
covers federation topics that overlap with core deployment concerns:
outbox durability mechanism (§4.1), edit-history retention (§4.2),
federation cache TTLs (§4.3), the federation `Peer/presence` outbound
rate (§5.2), retry and backoff intervals (§5.4), broadcast-mention
suppression (§6.1), receipt-sharing in federated contexts (§6.2), and
blocked-sender ephemeral suppression (§6.3). This guide focuses on what
those guides do not cover: running federation as an operational system.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, deployment policy, or
  default value choice (e.g., "operators SHOULD log Peer/* method calls"), the keyword
  reflects implementer best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## How to read this guide

`draft-atwood-jmap-chat-federation-00` deliberately specifies a minimal
wire contract (the `Peer/*` method set, discovery via `/.well-known/jmap`,
identity binding, outbox durability outcomes) and defers authentication
mechanism choice, peer-acceptance policy, abuse mitigation, moderation
escalation, and observability to operators. Federation deployment models
range from "open public network, peer-with-anyone" (Mastodon-style
ActivityPub) to "tightly curated allowlist between known organizations"
(corporate trust-domain federation). The spec cannot predict which model
fits any one deployment; this guide helps operators think through the
choices.

Each section follows the same shape as the broader
`jmap-chat-implementer-guide.md` and the other companion guides:

1. **What the spec leaves open** — with a citation.
2. **What you must decide.**
3. **Considerations.**
4. **Common patterns** from production federated systems.
5. **Recommended starting point** — defensible default, not normative.

---

## 1. Peer authentication mechanism choice

### What the spec leaves open

The federation draft (`draft-atwood-jmap-chat-federation-00.md`,
{#peer-authentication}) requires that a remote server authenticate before
invoking any `Peer/*` method, and requires the local server to verify
that the authenticated identity equals the `ownerUserId` advertised in
the remote's own `/.well-known/jmap` Session object. The specific
authentication *mechanism* is deployment-defined. The draft enumerates
acceptable choices — mutual TLS, OAuth 2.0 client credentials, bearer
tokens from out-of-band enrollment, overlay-network membership
credentials (Tailscale node identity, etc.) — but does not pick one.

### What you must decide

- Which authentication mechanism your deployment supports (one, or
  several with peer-specific choice at enrollment).
- How peer credentials are provisioned (mTLS certificate enrollment,
  OAuth client registration, bearer-token issuance, overlay-network
  join).
- Rotation and revocation procedures for each supported mechanism.
- How the local authentication layer derives the stable identity string
  used as the peer's `ChatContact.id` (the federation draft requires
  this binding; the mechanism by which the auth layer surfaces the
  string is internal).
- Whether peers must support a specific mechanism (mandatory-to-implement
  in your deployment) or whether you negotiate per-peer.

### Considerations

- *Mutual TLS* is the strongest mechanism for inter-server authentication
  and the easiest to scale to many peers, but operationally heavy:
  certificate issuance, expiry monitoring, and revocation infrastructure
  (CRL or OCSP, or a private CA) are non-trivial. Best fit when peers
  are organizations that already operate a PKI.
- *OAuth 2.0 client credentials* is well-understood, library support is
  ubiquitous, and rotation is straightforward (issue new client secret,
  retire old). Adds a dependency on an authorization server. Best fit
  when the deployment already runs an OAuth AS for end users.
- *Bearer tokens (out-of-band enrollment)* is the lowest-friction mechanism
  for small deployments: an admin issues a token, the peer presents it,
  done. Rotation is manual; revocation requires server-side token-state
  tracking. Acceptable for small, stable peer sets; does not scale to
  open federation.
- *Overlay-network membership* (Tailscale, WireGuard with mesh
  management, Nebula, ZeroTier, etc.) delegates authentication to the
  overlay. The peer's network identity *is* the credential; the overlay
  enforces it. Operationally simple; ties the deployment to the overlay
  provider's trust model and availability. Best fit when peers are
  organizational siblings already sharing infrastructure.
- *Multiple mechanisms* allow peer-specific choice: an enterprise peer
  uses mTLS, a small partner uses OAuth, an internal lab uses Tailscale.
  Increases code complexity in the auth layer; offers operational
  flexibility.
- *Identity binding* (federation draft, {#peer-authentication}) is the
  same requirement regardless of mechanism: whatever stable identity
  string the auth layer returns MUST equal the peer's advertised
  `ownerUserId`. The mechanism affects how the identity is verified, not
  what the binding rule is.

### Common patterns

| System | Authentication mechanism |
|---|---|
| ActivityPub (Mastodon, etc.) | HTTP Signatures over Linked Data; per-actor key pairs |
| Matrix federation | Mutual TLS at the transport layer; server keys at the protocol layer |
| XMPP server-to-server | TLS with SASL EXTERNAL or DIALBACK |
| Email (SMTP) | Variable: opportunistic TLS, DKIM at the message layer, SPF/DMARC at the policy layer |
| Tailscale-meshed services | Network-layer identity; service-layer trust derived from node identity |

### Recommended starting point

For most deployments: **OAuth 2.0 client credentials** with a per-peer
client registration flow. Library support is broad, rotation is
straightforward, and the authorization-server dependency is usually
already present for end-user authentication. The federation draft's
{#peer-authentication} examples list OAuth 2.0 first for this reason.

The bearer token presented by the peer SHOULD carry (or be associated
with, server-side) a `client_id` or claim that maps to a stable identity
string. The local server's auth layer surfaces that string to the JMAP
Chat layer; the JMAP Chat layer compares it against the peer's
advertised `ownerUserId` before granting peer-role access.

For deployments with strong corporate trust-domain models (between
named organizations with existing PKI): **mutual TLS** with a private CA
or a federated CA. The peer's certificate Subject (CN or SAN) maps to
the stable identity string; the local server's auth layer validates the
certificate chain.

For very small deployments (under 10 peers, all known): **bearer tokens
issued out-of-band** are acceptable. Track each token's binding to a
peer identity in a server-side table. Rotate annually at minimum.

Document the supported mechanisms, the enrollment procedure, the
rotation schedule, and the revocation procedure in deployment operator
documentation. Federation peers need this information to integrate.

---

## 2. Federation allowlist / denylist policy

### What the spec leaves open

The federation draft does not require any specific peer-acceptance
policy. A deployment MAY federate with any peer that successfully
completes peer authentication, or it MAY restrict federation to a known
set of peers via an allowlist, or it MAY deny specific peers via a
denylist. The draft defines the wire mechanics but does not constrain
which peers a deployment accepts.

### What you must decide

- Whether your deployment operates in *open federation* (any peer that
  authenticates is accepted), *invite-only / allowlist* (only enrolled
  peers can federate), or *denylist* (open by default, with named
  exclusions).
- How peer-acceptance policy is expressed (configuration file, admin
  database, runtime admin API).
- Who has authority to update the policy (single operator, multi-admin
  approval, time-locked changes).
- Onboarding procedure for accepting a new peer.
- Offboarding procedure for removing a peer (graceful drain vs immediate
  cutoff; see also §5 hostile-peer mitigation).
- Whether the policy is uniform across the deployment or scoped (e.g.,
  certain JMAP accounts are open-federation, others are allowlist-only).

### Considerations

- *Open federation* maximizes reach but exposes the deployment to abuse.
  Every malicious or compromised peer in the wider network can attempt
  delivery. The federation draft's blocked-sender rules
  ({#blocked-contacts}) and rate-limit requirements
  ({#peer-typing}, {#peer-presence}) bound the damage; they do not
  eliminate it.
- *Allowlist* is the safest model and the easiest to reason about: only
  enrolled peers can deliver. The cost is operational: every new peer
  requires enrollment; users cannot federate with the wider network
  spontaneously. Best for corporate or organizational deployments.
- *Denylist* gives reach with selective exclusion: useful for established
  open-federation deployments that need to defederate specific bad
  actors without abandoning open federation entirely.
- *Hybrid scoping* — e.g., open federation for personal accounts,
  allowlist-only for accounts in a regulated business unit — adds
  complexity but is realistic for multi-tenant deployments.
- *Policy expression*: a configuration file is simple and version-controlled
  but slow to update; an admin database with a runtime API is flexible
  but requires more careful access control. Most deployments use a
  combination: a baseline config file for the bulk of the policy, with
  an admin API for emergency additions and removals.
- *Authority*: a denylist with single-operator authority is a key risk
  surface — a compromised admin account can defederate critical peers
  unilaterally. Larger deployments SHOULD require multi-admin approval
  for permanent additions to the denylist; emergency one-admin actions
  can be permitted with a follow-up review window.
- *Offboarding*: an immediate cutoff is necessary for hostile peers but
  loses in-flight messages. A graceful drain (stop accepting new
  deliveries; finish processing the outbox; then cut off) is appropriate
  for non-hostile offboarding (e.g., a peer is winding down operations).
- *Discovery interaction*: a peer that is not on your allowlist (or is
  on your denylist) should not be able to learn that fact via probing.
  The federation draft requires the identity-binding check to precede
  any `Peer/*` work, but discovery probes (`GET /.well-known/jmap`) are
  public. Treat the discovery endpoint as public; enforce policy at the
  authentication layer.

### Common patterns

| System | Default policy |
|---|---|
| ActivityPub (Mastodon) | Open federation; per-instance denylists ("defederation") |
| Matrix (default) | Open federation; per-server allow/deny rules via policy modules |
| XMPP server-to-server | Variable; many production servers operate allowlist-style |
| Enterprise Slack / Teams external connections | Allowlist (admin-approved per external organization) |
| Email | Open federation with reputation-based filtering (SPF/DKIM/DMARC + RBL) |

### Recommended starting point

For new deployments without a specific federation model in mind:
**allowlist-only**, with explicit per-peer enrollment. This is the
lowest-risk default and avoids the abuse-mitigation burden of open
federation. Promote to denylist mode if and when there is demonstrated
demand for broader reach.

For deployments deliberately joining a wider open-federation network
(e.g., a public Mastodon-style social product on the JMAP Chat
substrate): **denylist with conservative defaults**. Start with a
denylist seeded from community-maintained block lists for known bad
actors; review and curate over time.

For corporate / multi-organization deployments: **allowlist**, scoped
to organizations with active business relationships. New peers are
added via the onboarding procedure described in §4.

Express the policy in a version-controlled configuration file for the
baseline, with an admin API for runtime additions and removals. Require
multi-admin approval for permanent denylist additions in larger
deployments. Log every policy change to an immutable audit trail.

---

## 3. /.well-known/jmap discovery hygiene

### What the spec leaves open

The federation draft ({#peer-discovery}) requires servers to cache
discovered session data and to re-run discovery when delivery fails,
but does not specify cache TTLs, refresh triggers beyond
delivery-failure, or how to handle peers that are persistently
unreachable. The main `jmap-chat-implementer-guide.md` §4.3 covers
federation cache TTLs for specific peer-supplied values
(`receiveTypingIndicators`, `ChatContact` records, endpoint
advertisements); this section covers the operational view of running
the discovery process itself.

### What you must decide

- How frequently to refresh `/.well-known/jmap` for known peers
  (proactive refresh) versus refreshing only on delivery failure
  (reactive refresh).
- How to distinguish transient peer unreachability (network blip, peer
  restart) from persistent unreachability (peer has gone away).
- How long to retain cached discovery data for a peer that has become
  unreachable.
- Whether to pre-cache (warm) discovery data for newly enrolled peers
  before the first message exchange.
- Concurrency: how many simultaneous discovery requests to allow against
  a single peer.
- Failure mode: if discovery fails entirely, do you queue outbound
  messages or fail them?

### Considerations

- *Proactive refresh* on a fixed schedule (e.g., every 1-6 hours) keeps
  cache fresh at the cost of background traffic. Useful when peer
  metadata changes (capabilities, endpoint hints) need to propagate
  without waiting for a delivery failure.
- *Reactive refresh* on delivery failure minimizes traffic but means
  stale metadata can cause repeated delivery failures before a refresh
  uncovers an updated session URL or `ownerUserId`. The federation draft
  requires this mode at minimum; proactive refresh is additive.
- *Transient vs persistent unreachability*: a peer that returns 503 or
  times out is probably transient; a peer that returns 404 or DNS NXDOMAIN
  for `/.well-known/jmap` is probably gone. Track consecutive failure
  count and time-since-last-success; declare persistent after a threshold
  (e.g., 7 days of consecutive failures).
- *Stale cache retention*: when a peer is unreachable, you have a choice
  between (a) keeping the cached discovery data so that recovery is fast
  if the peer returns, and (b) evicting it so that fresh discovery is
  forced on next contact. Most deployments keep the cache and let TTL
  expiry handle the rest.
- *Pre-cache warming* of newly enrolled peers (proactive discovery at
  enrollment time) lets the first message exchange skip discovery
  latency. Recommended for allowlist deployments where the peer set
  changes infrequently.
- *Concurrency*: a single peer that has 1000 messages queued for outbound
  delivery should not generate 1000 concurrent discovery requests on a
  cache miss. Coalesce: one discovery request in flight per peer at a
  time; queued senders wait for the result.
- *Discovery failure handling*: queue outbound messages and let them
  retry per the federation draft's retry/backoff rules (covered in
  main `jmap-chat-implementer-guide.md` §5.4). Do not fail messages
  immediately on a single discovery failure; the peer may be
  transiently unreachable.

### Common patterns

- HTTP-style caching with TTL + revalidation on near-expiry: most
  reasonable production behavior.
- Coalesced refresh per peer: one in-flight discovery per peer; queued
  callers share the result.
- Background refresh thread / scheduler: periodic walks over the active
  peer set, refreshing entries approaching TTL expiry.
- Reachability watchdog: tracks consecutive failures per peer; emits
  alerts when a peer crosses the persistent-unreachable threshold.

### Recommended starting point

- **Cache TTL**: 1 hour for `/.well-known/jmap` Session objects. Long
  enough to avoid hammering peers; short enough that capability and
  endpoint changes propagate within a workday.
- **Refresh trigger**: TTL expiry, plus immediate refresh on any
  delivery failure where the peer's session URL or `ownerUserId` might
  have changed.
- **Persistent-unreachable threshold**: 7 days of consecutive discovery
  failures; emit an operator alert at that point. Continue retrying at
  a slow rate (e.g., once per day) until recovery or explicit
  defederation.
- **Pre-cache warming**: enroll new peers with a synchronous discovery
  request at enrollment time. Fail the enrollment if discovery fails;
  the peer cannot federate yet.
- **Concurrency**: at most one in-flight discovery per peer; queue
  additional callers and serve them from the result.
- **Discovery failure**: do not fail messages immediately. Queue per the
  retry/backoff rules. Emit operator metrics for discovery success rate
  per peer.

Document the cache TTL and refresh schedule in the operator handbook so
peer operators can predict when their published changes propagate.

---

## 4. Onboarding a new peer

### What the spec leaves open

The federation draft describes the wire mechanics of peer authentication
and identity binding but does not specify the onboarding workflow: how
two organizations move from "we agreed to federate" to "the federation
is active". The spec also does not constrain how operators learn about
each other's capabilities (which `Peer/*` methods, which extensions).

### What you must decide

- The first-contact workflow: out-of-band agreement, exchange of
  credentials, discovery probe, first test message.
- Identity binding verification: confirming that the peer's authenticated
  identity equals their advertised `ownerUserId` before opening the
  federation gate.
- Capability negotiation: cataloging which `Peer/*` methods and which
  capability extensions the peer supports (versus what your deployment
  expects).
- Health-check procedure: a low-stakes test exchange before federation
  is declared "open" for end-user traffic.
- Documentation expectations: what does the peer operator need to
  publish for your operators to enroll them (and vice versa).

### Considerations

- *Out-of-band agreement* is necessary regardless of mechanism: the two
  operators must agree on which authentication mechanism to use, what
  identity strings to expect, and what scope of federation is intended
  (all users, named accounts only, etc.). This is a business and
  operational conversation, not a protocol step.
- *Credential exchange* depends on mechanism: mTLS certificate (or CA
  chain) exchange, OAuth client registration, bearer token issuance,
  or overlay-network mutual enrollment.
- *Discovery probe* validates that the peer's `/.well-known/jmap` is
  reachable and well-formed. This SHOULD happen before declaring
  federation active.
- *Identity binding verification* is the federation draft's
  {#peer-authentication} correspondence check, applied at first contact:
  authenticate the peer, fetch their `/.well-known/jmap`, confirm the
  authenticated identity equals the discovered `ownerUserId`. If this
  check fails at onboarding time, the federation cannot proceed; the
  peer's configuration is broken.
- *Capability negotiation* is reading the peer's Session object and
  understanding which capabilities they advertise. JMAP capabilities are
  self-describing; the peer either advertises
  `urn:ietf:params:jmap:chat` (with `role: "peer"` available) or they
  don't federate at all. Extension capabilities (push, WSS, filenode,
  calendars, tasks, etc.) are advertised in the same Session object;
  operators should know which extensions the peer supports.
- *Health-check exchange*: a test message in a designated test Space,
  with a real (not synthetic) recipient, validates end-to-end delivery,
  including outbox, retry, and read-receipt round-trip. This is
  operational practice, not protocol.
- *Documentation*: peer operators should publish at minimum: their
  supported authentication mechanisms; the format of their
  `ownerUserId` (regex or example); their `serverUrl` and discovery
  endpoint; which `urn:ietf:params:jmap:chat:*` companion capabilities
  they implement; their rate-limit policies; their abuse-report contact.

### Common patterns

- Email-style domain-level federation: largely automated; per-domain
  policies in MTA configuration; no per-relationship onboarding.
- XMPP server-to-server federation: out-of-band agreement on dialback or
  TLS-EXTERNAL; certificate exchange or DNS-anchored trust; manual
  configuration on both ends.
- Matrix homeserver federation: open by default; "trust-on-first-use"
  with reactive defederation; minimal per-peer onboarding.
- Enterprise Slack Connect / Teams external connection: explicit
  per-organization enrollment via admin UI; cross-organizational
  agreement required.
- AT Protocol / Bluesky: DID-anchored identity with discovery via DID
  resolution; less per-peer enrollment, more identity-system bootstrap.

### Recommended starting point

For allowlist-mode deployments (recommended in §2 for most new
operators): implement a five-step onboarding procedure documented in
the operator handbook.

1. **Out-of-band agreement** between operators. Agree on
   authentication mechanism, scope of federation, and abuse-reporting
   contacts.
2. **Credential exchange**. Each operator provisions the credentials
   their auth layer requires for the chosen mechanism.
3. **Discovery probe** from each side: each operator fetches the
   other's `/.well-known/jmap` and validates that it returns a
   well-formed Session object with the expected `ownerUserId`.
4. **Identity binding verification**: the federation draft's
   correspondence check applied at first contact. A failure here
   indicates a configuration error; federation cannot proceed.
5. **Health-check exchange**: a designated test account exchanges a
   small batch of messages, reactions, and typing events to validate
   the full `Peer/*` surface area before federation is declared
   active for end-user traffic.

Record each onboarded peer in a federation registry (peer identity, the
authenticated `ownerUserId`, the agreed authentication mechanism, the
contact for abuse reports, the onboarding date, the last successful
discovery refresh). This registry is the source of truth for the
operator's view of who they federate with.

Open-federation deployments (denylist mode) may skip the explicit
onboarding procedure for routine peers, but SHOULD apply the same
correspondence check at first contact and SHOULD record the peer in
the federation registry for observability.

---

## 5. Hostile-peer detection and mitigation

### What the spec leaves open

The federation draft requires rate-limiting on specific `Peer/*`
methods (`Peer/typing` rate is hardcoded; `Peer/presence` outbound rate
is implementation-defined per main `jmap-chat-implementer-guide.md`
§5.2) and requires blocked-sender suppression ({#blocked-contacts}).
The draft does not specify how to detect hostile-peer behavior in
aggregate, how to escalate from rate-limiting to defederation, or how
to coordinate abuse signals across multiple local accounts and Spaces.

### What you must decide

- Which behaviors qualify as hostile (excessive `Peer/deliver` rate,
  malformed messages, repeated SSRF attempts, identity-binding failures,
  spam volume).
- Per-peer rate limits for `Peer/deliver` (the draft does not specify
  one; the only hardcoded rate is `Peer/typing` for ephemeral signals).
- Abuse-signal aggregation across multiple recipients: a peer that
  spams 100 separate recipients is hostile even if each individual
  recipient's rate-limit was not exceeded.
- Quarantine vs full defederation: temporary suspension with operator
  review vs immediate hard cut-off.
- Defederation procedure: emergency one-admin action vs multi-admin
  approval; grace period for in-flight messages; user notification.
- How to handle peer credentials after defederation (revoke immediately,
  or let them expire).

### Considerations

- *`Peer/deliver` rate*: the federation draft does not specify a rate
  limit for `Peer/deliver`. Deployments MUST set one. A reasonable
  baseline is "high enough that normal traffic is not affected, low
  enough that a runaway sender is bounded" — e.g., 1000 messages per
  peer per minute as a coarse upper bound; lower for small deployments.
- *Per-recipient vs per-peer aggregation*: a hostile peer can stay
  under any per-recipient limit by spreading traffic across many
  recipients. Aggregate signals at the peer level: total `Peer/deliver`
  rate, total blocked-message rate (messages dropped under
  {#blocked-contacts}), identity-binding failure rate, malformed-payload
  rate.
- *Identity-binding failures* are a strong signal: a peer whose
  authenticated identity does not match their advertised `ownerUserId`
  is either misconfigured or attempting forgery. A burst of these
  failures from one peer should escalate immediately.
- *SSRF attempts*: blob-fetch URLs that target loopback, link-local,
  or private-network addresses (federation draft {#ssrf}). A peer
  consistently supplying such URLs is hostile; log and escalate.
- *Spam volume*: even without policy violations, a peer that delivers
  far more than typical traffic patterns is suspicious. Compare against
  baseline per-peer rates and flag outliers.
- *Quarantine* (temporary suspension) preserves the option of recovery
  if the peer was compromised and is now restored. *Defederation*
  (permanent removal) is appropriate for confirmed-hostile peers or
  for peers that fail to recover after operator outreach.
- *Defederation procedure* trades safety against speed: a multi-admin
  approval requirement is safer (one compromised admin can't defederate
  alone) but slower (an emergency takes longer to act on). A reasonable
  compromise: any one admin can quarantine immediately; permanent
  defederation requires multi-admin approval within 24 hours, or the
  quarantine is automatically lifted.
- *Grace period for in-flight messages*: on graceful defederation,
  drain the outbox and complete in-flight delivery before cutting off
  authentication. On emergency defederation (active abuse), cut
  authentication immediately; in-flight outbound messages from the
  hostile peer are abandoned.
- *User notification*: defederation affects users who had relationships
  with users on the defederated peer. Some deployments notify affected
  users explicitly ("your contact X is no longer reachable because their
  server has been defederated"); others suppress the cause. The
  spec-level blocked-contacts rule {#blocked-contacts} does not extend
  to defederation; this is operator UX policy.

### Common patterns

| System | Typical mitigation |
|---|---|
| Mastodon / ActivityPub | Per-instance silence/suspend; community blocklists |
| Matrix | Server ACLs; per-room moderation policies |
| XMPP | Server-level blacklist; sometimes per-user blocked-domain lists |
| Email | Reputation-based filtering, RBLs, greylisting, full domain blocking |
| Enterprise federation | Manual operator review and explicit defederation |

### Recommended starting point

- **Per-peer `Peer/deliver` rate limit**: 1000 messages per minute as a
  starting cap. Tune based on observed traffic. A peer hitting this
  limit gets responses delayed (not errored) to slow them; a peer
  sustaining the limit for hours triggers an operator alert.
- **Peer-level abuse aggregation**: track total `Peer/deliver` rate,
  blocked-message rate, identity-binding failure rate, malformed-payload
  rate, and SSRF-attempt rate per peer over rolling windows (last 5
  minutes, last hour, last 24 hours).
- **Identity-binding failure burst threshold**: more than 10 failures
  in any 5-minute window from a single peer triggers an immediate
  operator alert and an automatic 1-hour authentication quarantine for
  that peer.
- **Spam-volume threshold**: a peer whose delivery rate exceeds 5x
  their 7-day baseline triggers an alert and a precautionary rate-limit
  tightening.
- **Quarantine action**: pause acceptance of new `Peer/*` requests from
  the peer; continue serving previously-delivered content normally so
  affected end users are not surprised. Notify the peer operator out
  of band (via the abuse-report contact from onboarding) before
  permanent defederation.
- **Defederation procedure**: any one admin can quarantine for up to
  24 hours; permanent defederation requires multi-admin approval (at
  least two admins) for production-scale deployments. Smaller
  deployments MAY allow single-admin defederation but SHOULD log to an
  immutable audit trail with operator name and reason.
- **Grace period**: graceful defederation drains the outbox over 24
  hours; emergency defederation is immediate.
- **User notification**: deployments SHOULD inform affected users that
  the peer is no longer reachable, without disclosing the reason if the
  reason is sensitive (active abuse investigation, legal hold, etc.).
- **Credential handling post-defederation**: revoke the peer's
  authentication credentials immediately on permanent defederation
  (mTLS certificate revocation, OAuth client deletion, bearer token
  revocation, overlay-network removal).

Document the abuse-handling procedure in the operator handbook and the
deployment's privacy notice. Users need to understand that federation
mistakes can affect their reachable contact set.

---

## 6. Cross-server moderation

### What the spec leaves open

The federation draft addresses identity-level abuse via blocked-sender
suppression ({#blocked-contacts}) and addresses peer-level abuse via
the rate-limit and identity-binding mechanisms. It does not address the
operational reality of cross-server moderation: a user on peer A
misbehaves in a Space hosted on local server B, the Space admin on B
wants to moderate, but the offender's account is administered by A.

### What you must decide

- How a local Space admin moderates a remote user (kick from Space, ban
  from Space, report to peer operator).
- How a local end user reports abuse from a remote user (in-product
  report; out-of-band complaint).
- How peer abuse reports are exchanged between operators (out-of-band
  email, structured report format, dedicated reporting endpoint).
- Whether local moderation actions against remote users are visible to
  the remote user (block is silent; ban from Space typically is not).
- Escalation path: local Space-level action → local account-level
  block → peer operator notification → defederation.

### Considerations

- *Local Space-level action*: a Space admin on server B can kick or ban
  a user from peer A from their Space using the normal admin tools
  (`Chat/set` removing the member, etc.). The remote user observes
  this as a `Peer/groupUpdate` message indicating their removal. This
  is normal protocol behavior; nothing special is needed for federation.
- *Local account-level block*: a local user on B blocks a user on peer
  A via `ChatContact.blocked: true`. The spec's blocked-contacts rule
  ({#blocked-contacts}) silently suppresses incoming events from that
  user. This is per-account; it does not affect other local users on B.
- *Peer abuse reports*: there is no JMAP Chat protocol for sending
  abuse reports between operators. This is intentional: abuse reports
  are sensitive and benefit from human review, which a wire protocol
  cannot replace. Operators exchange abuse reports out of band, via
  email or web forms documented at the peer's `/.well-known/jmap`
  endpoint (in a `feedbackUri` or similar deployment-defined field) or
  in the onboarding registry.
- *Visibility*: blocked-contact suppression is silent (the blocked party
  does not see they are blocked). Ban-from-Space is visible (the banned
  user sees they have been removed). This asymmetry is deliberate: the
  block is privacy-protective for the blocker; the ban is
  Space-governance-visible for the ban subject.
- *Escalation*: most moderation actions stay local. Escalation to the
  peer operator is reserved for serious patterns of abuse (repeated
  incidents from multiple users on the peer, identity-binding
  irregularities, attempted SSRF or other security-relevant abuse). The
  ultimate escalation step is defederation (§5).
- *Cross-server policy mismatch*: peer A and peer B may have different
  acceptable-use policies. A user on A doing something acceptable by A's
  policy may be unacceptable by B's policy. Local moderation enforces
  B's policy on B's territory; the peer operator on A is not obligated
  to mirror B's enforcement.
- *Anti-retaliation*: a hostile user reporting non-hostile users to
  retaliate is a known pattern. Peer abuse reports SHOULD be
  investigated by the receiving operator, not blindly acted upon.

### Common patterns

- ActivityPub / Mastodon: per-instance moderation; in-product
  reporting; per-user blocks; per-instance defederation as escalation.
- Matrix: per-room moderation via ACLs; per-server bans; protocol-level
  redaction events.
- Email: per-message reporting (RFC 5965 ARF, or proprietary "report
  spam" buttons feeding back to reputation services); per-domain
  blocking via RBLs as escalation.
- Enterprise Slack Connect / Teams: in-product reporting; admin actions
  visible across organizations via the connection metadata.

### Recommended starting point

For most deployments:

- **Local Space-level moderation**: use the normal admin tools
  (`Chat/set` membership patches, `SpaceBan` records). No federation
  special-casing required. The remote user receives the `Peer/groupUpdate`
  reflecting their removal.
- **Local account-level block**: standard `ChatContact.blocked: true`.
  Per the spec's blocked-contacts rule, all subsequent ephemeral and
  delivery events from that user are silently suppressed.
- **In-product abuse reporting**: clients SHOULD provide a "Report user"
  affordance that routes to the deployment's abuse-handling team. The
  report includes the offending user's ChatContact id, the Space (if
  any), the offending message id, and the reporter's description.
  Local team triages and decides on actions: local-only (account block,
  Space ban) versus escalate to peer operator.
- **Peer-operator notification format**: when escalating to the peer
  operator, send an email or web-form submission with: the offending
  user identity (their `ownerUserId`), the offending behavior summary,
  evidence (message ids, timestamps), and the local action taken. The
  peer operator decides whether to act on their end.
- **Defederation as last resort** (§5 covers details). Defederation
  affects all users on the peer, not just the offender; reserve for
  cases where the peer operator is unable or unwilling to act on
  serious abuse.
- **Investigation discipline**: peer abuse reports received from other
  operators SHOULD be investigated before action. Treat them as input,
  not commands. Anti-retaliation rule applies.

Document the abuse-reporting workflow, the escalation criteria, and
the typical timelines in the deployment's user-facing trust-and-safety
documentation. Users should understand what reports do and what to
expect.

---

## 7. Federation observability

### What the spec leaves open

The federation draft does not require any specific logging, metrics, or
monitoring. Observability is entirely a deployment concern.
Practically, no production federated deployment can operate without it:
debugging delivery failures, capacity planning, and abuse investigation
all require visibility into the federation surface.

### What you must decide

- Which `Peer/*` method calls to log, at what level of detail.
- Which metrics to expose (delivery success rate, retry depth, queue
  depth, per-peer round-trip time, identity-binding failure rate, etc.).
- Retention period for federation logs (operational debugging vs
  compliance vs privacy).
- How to distinguish "peer down" from "local bug" from "policy
  rejection" in observability data.
- Dashboards and alerting thresholds.
- Tracing: whether to propagate trace context across the federation
  boundary (interesting for distributed debugging; raises privacy
  concerns if trace data leaks).

### Considerations

- *Logging level*: full request-and-response logging at every `Peer/*`
  call is operationally expensive and a privacy concern (logs contain
  message bodies). Most production deployments log metadata (method,
  peer identity, timing, success/failure) at the per-call level and
  full payloads only at error level or behind a debug flag.
- *Metrics to expose*: at minimum, per-peer counters and histograms for
  inbound `Peer/*` call rate, outbound `Peer/*` call rate, delivery
  success rate, retry depth distribution, round-trip time, outbox queue
  depth, blocked-message count, identity-binding failure count,
  rate-limit hit count. These metrics drive operator dashboards and
  alerts.
- *Distinguishing failure types*:
  - *Peer down*: network errors, timeouts, 5xx responses from peer.
    Action: backoff and retry; alert if persistent.
  - *Local bug*: 5xx responses from your own server (you should see
    these in local error logs simultaneously). Action: fix the bug.
  - *Policy rejection*: 4xx responses from peer (peer rejecting your
    request) or your own server returning 4xx (you rejecting peer's
    request). Action: investigate policy mismatch; not a transient
    failure.
- *Retention*: federation logs are operational data, not user data, but
  they contain identifying metadata (peer identities, account ids,
  Space ids, message ids). Retain for as long as operational debugging
  and compliance require; not longer. 30 days is a reasonable default
  for raw logs; aggregated metrics retain longer (1 year is common).
- *Tracing*: distributed tracing (OpenTelemetry-style) across federation
  is operationally useful when both peers cooperate. It is not part of
  the JMAP Chat federation protocol; if used, do so via standard HTTP
  trace propagation headers and ensure that trace data does not include
  message bodies or sensitive metadata.
- *Privacy hygiene*: log peer identity (the `ownerUserId`) and Chat /
  Space / Message ids; do NOT log message bodies, attachment contents,
  or `ChatContact` profile data routinely. Behind a debug flag, payload
  inclusion is acceptable for short-term incident response.

### Common patterns

- Prometheus + Grafana for metrics; structured JSON logs to a central
  collection system; per-peer dashboards.
- Per-peer SLO tracking (e.g., 99% of `Peer/deliver` succeed within 5
  seconds when peer is reachable; 99.9% of messages eventually deliver
  within 24 hours).
- Alerting on: persistent peer unreachability, identity-binding failure
  burst, rate-limit hit burst, outbox queue depth above threshold,
  delivery success rate below threshold per peer.
- Operator dashboards: per-peer health summary, federation-wide
  overview, top-N peers by traffic, top-N peers by failure rate.

### Recommended starting point

- **Log every `Peer/*` call** at metadata level: timestamp, method,
  peer `ownerUserId`, target accountId, chatId (if applicable), success
  or failure, response code, latency. NOT message body, NOT attachment
  content.
- **Metrics**: expose per-peer counters and histograms via the
  deployment's standard metrics endpoint (Prometheus-style or
  equivalent). At minimum: `peer_deliver_requests_total{peer, status}`,
  `peer_deliver_latency_seconds{peer}`, `peer_outbox_depth{peer}`,
  `peer_identity_binding_failures_total{peer}`,
  `peer_blocked_messages_total{peer}`,
  `peer_rate_limit_hits_total{peer, method}`.
- **Log retention**: 30 days for raw `Peer/*` request logs; 1 year for
  aggregated metrics; permanent (audit log) for federation policy
  changes (peer add/remove, defederation actions).
- **Alerting**:
  - Per-peer delivery success rate drops below 95% over 1 hour.
  - Per-peer identity-binding failures exceed 10 in 5 minutes.
  - Outbox depth for a peer exceeds 10,000 messages.
  - Discovery failures for a peer over 7 consecutive days
    (persistent-unreachable threshold from §3).
- **Dashboards**: maintain a federation overview dashboard (all peers,
  one row each, key metrics in a single view) and per-peer drill-down
  dashboards for incident investigation.
- **Privacy**: never log message bodies in routine federation logs.
  Behind a debug flag (off by default; on only for active incident
  response), payload logging may be enabled temporarily for a specific
  peer or chatId. Document any such temporary collection in an incident
  log.
- **Tracing**: optional. If implemented, propagate W3C Trace Context
  (`traceparent`, `tracestate`) headers on `Peer/*` calls; ensure that
  span attributes do not contain message bodies.

Document the metrics exposed, the dashboard locations, and the alerting
thresholds in the operator handbook. Federation is a live system;
ongoing operational visibility is essential.

---

## Cross-references

| Topic | See also |
|---|---|
| Underlying JMAP Chat protocol | `draft-atwood-jmap-chat-00.md` |
| The JMAP Chat federation spec | `draft-atwood-jmap-chat-federation-00.md` |
| Outbox durability mechanism | `jmap-chat-implementer-guide.md` §4.1 |
| Edit-history retention | `jmap-chat-implementer-guide.md` §4.2 |
| Federation cache TTLs (`receiveTypingIndicators`, ChatContact, endpoints) | `jmap-chat-implementer-guide.md` §4.3 |
| Federation `Peer/presence` outbound rate | `jmap-chat-implementer-guide.md` §5.2 |
| Retry and backoff intervals | `jmap-chat-implementer-guide.md` §5.4 |
| Broadcast-mention suppression in federated contexts | `jmap-chat-implementer-guide.md` §6.1 |
| Receipt-sharing in federated contexts | `jmap-chat-implementer-guide.md` §6.2 |
| Blocked-sender ephemeral-event suppression | `jmap-chat-implementer-guide.md` §6.3 |
| Identifier scheme (incl. `ChatContact.id` format) | `jmap-chat-implementer-guide.md` §3.1 |
| DID URI handling in `ChatContact.id` | `jmap-chat-implementer-guide.md` §3.2 |
| Federated mention textual form parsing | `jmap-chat-implementer-guide.md` §3.3 |
| WebSocket transport for ephemeral events | `jmap-chat-wss-guide.md` |
| Inline push payloads | `jmap-chat-push-platform-guide.md` |
| Platform-specific push delivery | `jmap-push-platform-guide.md` |
| Calendar integration | `jmap-chat-calendars-guide.md` |
| Task list integration | `jmap-chat-tasks-guide.md` |
