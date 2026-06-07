# JMAP Chat Calendars — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-chat-calendars-00`.
Covers the deployment-defined posture decisions the spec deliberately leaves
open.

Read the draft and `draft-ietf-jmap-calendars` first. This guide does not
re-state normative requirements; it covers what the spec leaves to
implementations and offers patterns and starting points.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED, etc.) for
clarity, but in the spirit of implementer guidance rather than as a normative protocol
specification:

- The drafts (`draft-atwood-jmap-chat-*.md`) are the normative source of truth. Where
  this guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice, UX default, or deployment
  choice (e.g., "servers SHOULD log admin actions"), the keyword reflects implementer
  best-practice. Deviation does not affect protocol interop.
- Cite the spec, not the guide, when claiming normative authority.

---

## How to read this guide

`draft-atwood-jmap-chat-calendars-00` deliberately specifies a minimal wire surface
(one new optional field on Space, one new capability with three configuration properties) and defers authorization, privacy
policy, and operational defaults to implementations. Calendar authorization in
production deployments ranges from "any team member can do anything" (small
collaborative groups) to deeply layered enterprise models with LDAP/AD groups,
sensitivity labels, delegation chains, and compliance overlays. The spec cannot
predict which model fits any one deployment; this guide helps implementers think
through the choices.

Each section follows the same shape as the broader `jmap-chat-implementer-guide.md`:

1. **What the spec leaves open** — with a citation.
2. **What you must decide.**
3. **Considerations.**
4. **Common patterns** from production chat-and-calendar systems.
5. **Recommended starting point** — defensible default, not normative.

---

## 1. Calendar binding policy

### What the spec leaves open

`Space.calendarId` (per `draft-atwood-jmap-chat-calendars-00.md`, {#calendar-binding})
is an optional field. The spec says deployments advertise `mayBindCalendar: true` if
they support binding at all, and the binding can be set or cleared by members holding
`"manage_space"`. The spec does not say *when* binding happens, *who* initiates it, or
*whether* a new Calendar is automatically created at Space creation time.

### What you must decide

- Whether your deployment supports calendar binding at all (`mayBindCalendar:
  true/false`).
- When a new Calendar is created: at Space creation (auto-bind), on first
  calendar-related action (lazy-bind), or only on explicit admin request
  (manual-bind).
- Whether a Space may be re-bound to a different Calendar after the initial bind,
  and what happens to existing CalendarEvents in the old Calendar on rebind.
- Whether destroying a Space destroys its bound Calendar, leaves it orphaned, or
  un-binds it without destruction.
- Whether a Space MUST have a bound Calendar (calendar-first deployments) or MAY
  remain unbound (calendar-as-feature deployments).

### Considerations

- *Auto-bind-on-create* is the lowest friction: every Space gets a Calendar
  automatically; users never think about binding. Cost: storage for Calendars
  that may never be used. Risk: bound Calendar count grows with Space count
  even when calendar features are dormant.
- *Lazy-bind* delays Calendar creation until something actually needs it. Lower
  baseline storage; introduces a "set up calendar?" UX moment.
- *Manual-bind* is the most explicit: admins choose when calendar features
  appear in a Space. Highest friction; clearest mental model; right for
  enterprise deployments where Calendar provisioning may have approval flows.
- *Rebind* is messy: existing event references in chat messages point at the
  old Calendar's CalendarEvents; clients caching MessageAction metadata see
  stale data. Some deployments allow rebind but flag the operation as
  consequential; others disallow rebind and require a new Space.
- *Cascade-destroy* (destroying the Space destroys the Calendar) is convenient
  but loses event history. Most deployments prefer un-bind-on-destroy: the
  Calendar persists for archival access; binding is just severed.

### Common patterns

| System | Pattern |
|---|---|
| Discord | Server creation does not auto-create a calendar; Scheduled Events are added per-event. |
| Microsoft Teams | Team creation auto-creates a calendar (the team calendar in SharePoint); admins do not configure binding. |
| Slack | No native calendar binding; external calendar apps create per-channel integrations. |
| Mattermost | Calendar feature requires plugin/integration; binding is explicit per-channel. |

### Recommended starting point

For consumer/social deployments: **auto-bind-on-create** with explicit
un-binding available. Users expect features to "just work" without
configuration.

For enterprise deployments: **manual-bind** with administrative approval. The
Calendar may belong to a different administrative domain than the Space
(corporate calendar service vs team chat service); binding is a deliberate
act.

For unknown product context: **lazy-bind on first calendar-related action**.
Postpones the Calendar creation until justified by use.

Re-bind: deployments SHOULD allow re-binding but SHOULD log it to a server
admin audit trail and SHOULD notify Space members when the binding changes
(via a system message in any channel of the Space, or an out-of-band
notification). The wire protocol does not require this notification; it is a
deployment-side UX choice.

Un-bind on Space destroy: deployments SHOULD un-bind rather than cascade-
destroy. The Calendar persists for archival access; users who want to delete
the Calendar do so via `Calendar/set` separately.

---

## 2. Authorization for calendar operations

### What the spec leaves open

The new calendars spec section "Authorization" explicitly defers the
authorization model to deployments. The wire contract is:

- JMAP Calendars authorization is authoritative for Calendar operations.
- Servers MUST evaluate JMAP Calendars authorization on every Calendar
  operation.
- Servers MAY use Space membership and the closed Space-permission
  vocabulary as inputs to that decision (per §Space Permission Resolution
  of `draft-atwood-jmap-chat-00.md`, the per-method auth latitude paragraph
  from commit 38055ec).
- Deployments MUST document the actual authorization model.

### What you must decide

- How JMAP Calendars permissions on a bound Calendar relate to Space
  membership and the Space's closed permission vocabulary
  (`view`, `send`, `manage_channels`, `manage_space`, etc.).
- Whether Space-level permissions are *inputs* to the authorization decision
  (your server combines them with JMAP Calendars permissions), or *replaced
  by* it (Space permissions are advisory; Calendar permissions are
  authoritative), or *bypassed* by it (Space permissions decide; Calendar
  permissions are not consulted).
- Whether enterprise authorization (LDAP/AD groups, OIDC claims, SAML
  attributes, sensitivity labels, delegation) overrides or augments the
  built-in permission models.
- Default behavior for newly-bound Calendars: who can RSVP, create events,
  modify others' events?

### Considerations

- *Mirror-Space pattern*: read = Space membership; create = `manage_channels`;
  RSVP = Space membership; modify-others = `manage_channels`. Simple and
  predictable; matches user intuition from chat-product UX. Insufficient for
  enterprise where calendar authorization is finer-grained than chat
  authorization.
- *Mirror-Calendar pattern*: defer entirely to JMAP Calendars' Calendar
  permissions. Most flexible; requires Space members to also have JMAP
  Calendars privileges on the bound Calendar. Useful when Calendars are
  shared across multiple Spaces or used independently.
- *Hybrid pattern*: Space membership gates discovery (you must be a Space
  member to *see* the bound Calendar at all); JMAP Calendars permissions
  gate operations. Practical default for many deployments.
- *External authorization*: an external policy engine (Open Policy Agent,
  custom enterprise auth service) decides every operation. Used when the
  deployment's authorization model doesn't fit either closed vocabulary.
  Highest implementation cost; best fit for complex regulated environments.
- *Sensitivity labels* (Microsoft Information Protection or similar): some
  CalendarEvents may carry confidentiality classifications that override
  membership-based access. A user who is a Space member may still not be
  permitted to read certain CalendarEvents.
- *Delegation*: an executive assistant managing a manager's Calendar acts on
  the manager's behalf. The Space membership rules don't see the delegation;
  JMAP Calendars does.

### Common patterns

| System | Authorization pattern |
|---|---|
| Discord | Server-level "Manage Events" permission for create/modify; visibility tied to channel membership. Simple two-tier. |
| Microsoft Teams | Channel membership + Outlook permissions; sensitivity labels overlay; delegation respected. Complex. |
| Slack | External app permissions; calendar app's ACL is authoritative. Slack-side membership is advisory. |
| Google Calendar with chat | Per-calendar sharing settings; Google Workspace IAM overrides. |
| Linear (analogous tasks system) | Per-project ACL; team membership grants project visibility; specific roles grant write. |

### Recommended starting point

**Hybrid pattern** as the baseline: Space membership grants the right to
*see* that a Calendar is bound to the Space and to receive `CalendarEvent`
state-change notifications via JMAP Chat. JMAP Calendars permissions on the
bound Calendar gate operations (read details, RSVP, create, modify).

For enterprise deployments: add an external authorization layer above both.
Define which JMAP Calendars operations require which directory groups, claim
combinations, or sensitivity-label conditions. The Space-side wiring
becomes one of several inputs.

For small/casual deployments: mirror-Space pattern is sufficient. Read = any
member, create/RSVP = any member, modify-others = `manage_channels`. Treat
RSVP as automatically permitted to any member.

Document the chosen model in user-facing API documentation. A user seeing
`forbidden` on a calendar action expected to work needs to be able to find
out why without reading server source code.

---

## 3. RSVP UX flow patterns

### What the spec leaves open

The spec says RSVPs are performed via `CalendarEvent/set` (no new method).
The chat layer's contribution is the UX wiring: a message with a
`urn:jmap:chat:cap:calendar-event` MessageAction can render RSVP buttons.
The spec does not prescribe the button set, layout, or interaction model.

### What you must decide

- Button set: tri-state (Accept / Decline / Tentative), single-state
  (Interested), or richer (Accept / Decline / Tentative / Delegated)?
- Where RSVP appears: inline in the message (action buttons on the event
  card) vs. opening a separate calendar UI for RSVP?
- What happens when RSVP succeeds: in-line confirmation, refreshing the
  event card, posting a follow-up message?
- Whether to surface other participants' RSVPs in the chat view, or only in
  the calendar app?
- Whether to allow RSVP comments (per JMAP Calendars participation comment),
  and where to surface them?

### Considerations

- *Tri-state RSVP* matches calendar-app conventions (Outlook, Google
  Calendar, Apple Calendar). Best fit for productivity contexts where the
  RSVP carries scheduling weight.
- *Single-state "Interested"* is Discord's approach. Best fit for
  casual/social events where formal RSVPs feel heavy-handed.
- *Inline RSVP* is the dominant pattern (Teams, Slack). One-click
  participation; no context-switch to a separate app.
- *Calendar-app RSVP* is the minimal pattern (Telegram, many older chat
  products). The chat just links to the event; RSVPing happens elsewhere.
- *Participant visibility* is privacy-sensitive: showing who RSVP'd what
  reveals organizational and social information. Some deployments hide
  RSVPs except from the organizer; others surface them to all participants.
- *RSVP comments* (JMAP Calendars `participationComment`) are a useful UX
  affordance but require a richer UI (text input alongside the RSVP buttons).

### Common patterns

| System | RSVP pattern |
|---|---|
| Microsoft Teams channel meetings | Tri-state (Accept / Decline / Tentative); inline buttons on meeting card; RSVPs visible to organizer and (optionally) attendees. |
| Discord Scheduled Events | Single-state "Interested" / "Not Interested"; participants visible to all. |
| Slack | External integrations vary; most surface tri-state via app cards. |
| Telegram | Polls used informally for RSVP; no native calendar RSVP. |
| Apple iMessage | Calendar invites surface as iMessage interactive cards (iOS); tri-state RSVP available. |

### Recommended starting point

**Tri-state RSVP with inline buttons** on the calendar-event MessageAction
card. Map to JMAP Calendars participation status as:

| Button | JMAP Calendars `participationStatus` |
|---|---|
| Accept | `"accepted"` |
| Decline | `"declined"` |
| Tentative | `"tentative"` |

Surface RSVP results to the organizer always; surface to other participants
based on a deployment-defined policy (default: visible to all participants
in non-sensitive contexts, organizer-only in sensitive contexts).

For RSVP comments: enable a "RSVP with comment" affordance when the user
holds the button or selects "More options". Keep the default RSVP path
single-click.

For Discord-style casual contexts (gaming communities, friend groups):
single-state "Interested" is acceptable. Map to JMAP Calendars
`participationStatus: "accepted"` and treat as a softer commitment.

---

## 4. ICS attachment parsing decisions

### What the spec leaves open

The spec says servers MAY parse `.ics` attachments and surface CalendarEvent
representations (per `draft-atwood-jmap-chat-calendars-00.md` {#ics-parsing}).
It does not say *when* to parse, *what* to do with the result, or *how
aggressively* to rate-limit. It does prohibit auto-RSVP on the recipient's
behalf and requires DoS-resistant parsing (size limits, recursion limits,
sanitization).

### What you must decide

- Whether your server parses `.ics` attachments at all
  (`supportsIcsParsing: true/false`).
- If parsing: when (always on message store, on user click, on background
  job)?
- Whether parsing creates a draft CalendarEvent on the recipient's Calendar
  (yes / opt-in only / never)?
- Sanitization rules: which fields are preserved (SUMMARY, DESCRIPTION,
  LOCATION), which are stripped (URL, ATTENDEE, ATTACH), which are
  rewritten?
- Rate limits for auto-creation per sender, per recipient, per hour.
- Failure mode for malformed iCalendar: reject the message, accept
  message-without-parse, return error to sender?

### Considerations

- *Always parse* is convenient but consumes parser resources for every
  attachment. Useful when the server has spare cycles.
- *Parse on click* defers cost until the user actually wants the
  CalendarEvent. Lower baseline cost; one-click latency to surface.
- *Background parse* runs after message storage but before display. Good
  middle ground.
- *No parse* (just attach the `.ics` as a file) is the minimum-effort
  approach. Users download and import manually. Acceptable when JMAP
  Calendars is a separate product or not in scope.
- *Auto-creation on recipient's Calendar* is privacy-sensitive: the
  sender's attachment now lives in the recipient's calendar database. The
  spec requires participation status to be `"needs-action"` (no auto-RSVP),
  but the event itself is on the recipient's calendar.
- *Sanitization*: iCalendar `URL`, `ATTACH`, and `ATTENDEE` fields carry
  peer-supplied URIs and email addresses. Treating them as untrusted is
  essential. Sanitization should strip or quarantine these fields.
- *Rate limiting* prevents a malicious sender from flooding the recipient's
  calendar with junk CalendarEvents. A reasonable default is no more than
  a few new CalendarEvents auto-created per sender per recipient per hour.

### Common patterns

| System | ICS handling |
|---|---|
| Microsoft Teams | Server-side parse; integrates with Outlook calendar; recipient sees the event with RSVP buttons inline. |
| Apple iMessage | Server-side parse; "Maybe" / "Yes" / "No" buttons appear inline. |
| Slack with calendar app | App-specific parse; surfaces event card; click to add to user's calendar. |
| Gmail | Server-side parse; renders event preview with RSVP buttons; "Maybe" / "Yes" / "No". |

### Recommended starting point

**Parse on background after message storage**, before display. This balances
parser cost (one parse per `.ics`) with UX (event card visible immediately
when message is opened).

**Do not auto-create CalendarEvents on the recipient's Calendar** without
explicit user action. Render the event card with an "Add to calendar"
button that the recipient clicks to create the CalendarEvent. Auto-creation
introduces too much privacy risk and surprise; the click is a small UX cost.

Sanitization SHOULD strip:
- `URL` fields containing javascript: or data: URIs.
- `ATTACH` fields containing non-https URIs or peer-supplied attachments not
  separately verified.
- Control characters and HTML tags in `SUMMARY`, `DESCRIPTION`, `LOCATION`.

Rate-limit auto-creation to 10 new CalendarEvents per sender per recipient
per hour. Reject `.ics` attachments larger than 256 KB or with more than
~50 expanded recurrence instances per year (DoS guard).

For deployments where JMAP Calendars is not present or is a separate
account: do not parse server-side; surface `.ics` as a regular file
attachment.

---

## 5. Availability lookup policy

### What the spec leaves open

The spec says `Principal/getAvailability` may be exposed in chat context but
the policy is deployment-defined (per `draft-atwood-jmap-chat-calendars-00.md`
{#availability}). The wire contract is: servers MAY expose, MUST reject
violations, MUST document the policy. The spec enumerates policy axes
(same-Space membership, cross-account, federation, organizational unit,
explicit consent, time windows, sensitivity tags, regulatory overlays) but
does not prescribe values.

### What you must decide

- Who can query whose availability?
- Whether the policy is uniform per-deployment or configurable per-Space
  or per-principal?
- What time window is exposed (next 24 hours, next week, no limit)?
- Granularity: free/busy slots only, or also tentative/private/working-
  elsewhere distinctions?
- How federation is handled: cross-server queries allowed, allowed with
  consent, or never?
- Whether the policy is dynamic (changes based on context) or static
  (fixed by deployment configuration).

### Considerations

- *Default-open within a Space* (any member can query any other member's
  availability for events within the Space's time horizon) is the friction-
  free choice. Best for collaborative environments where scheduling is
  routine.
- *Default-closed* requires explicit opt-in. Better for privacy-conscious
  contexts.
- *Time-window restrictions* (only show next 7 days; only working hours)
  reduce information leakage while preserving scheduling utility.
- *Granularity*: showing only "busy / free" reveals less than "busy: working
  on Project X" or "busy: working from coffee shop". The spec recommends
  slot-based granularity (busy/free intervals); event details belong to
  JMAP Calendars proper with richer authorization.
- *Federation*: cross-server availability is high-risk (the remote server
  is a separate trust boundary). Most deployments either disallow entirely
  or require explicit consent flows.
- *Enterprise context*: directory groups define visibility scope. A user
  may see availability of their team but not other teams; of their
  organizational unit but not other units.

### Common patterns

| System | Availability pattern |
|---|---|
| Microsoft Teams | Default-open within an organization; cross-org via federation requires explicit configuration. Slot-based; respects calendar permissions. |
| Discord | No availability lookup; users self-report status. |
| Google Workspace | Per-calendar sharing settings; "free/busy" tier is most common; org-level defaults configurable. |
| Slack | External calendar integrations may expose availability; native Slack does not. |

### Recommended starting point

For **internal/single-organization deployments**: default-open within a
Space. Members of the same Space can query each other's availability for
the next 14 days. Slot-based granularity. Sensitivity-marked CalendarEvents
are excluded from results.

For **multi-tenant deployments** (e.g., chat host serving many independent
organizations): default-closed across tenant boundaries. Within a tenant,
default-open within a Space.

For **federated deployments**: cross-server availability lookup is
disallowed by default. Deployments wishing to enable it MUST establish a
consent flow outside this specification.

For **privacy-conscious deployments** (regulated industries, personal-data
contexts): default-closed; require explicit per-principal opt-in. Document
the consent mechanism in user-facing settings.

Document the chosen policy prominently. Users SHOULD know who can see their
free/busy state without reading source code.

---

## 6. External calendar provider integration

### What the spec leaves open

The spec assumes a JMAP Calendars-native model. Real-world deployments
often need to bridge to external calendar providers (Google Calendar,
Outlook/Exchange, Apple Calendar via CalDAV). The spec does not prescribe
how this bridging works.

### What you must decide

- Whether your deployment proxies external calendar providers through JMAP
  Calendars (so users see one unified calendar API regardless of source)?
- Whether your deployment supports mixed-mode (some Spaces bound to
  JMAP-native Calendars, others to proxied external Calendars)?
- How identity matching works: when a Space member has both a JMAP Chat
  identity and an external calendar account, how is the linkage
  established?
- How RSVPs round-trip: a member RSVPs in chat, the chat server calls JMAP
  Calendars, the JMAP Calendars adapter calls the external provider's API
  to update the event. Failure modes at each step.
- How event-ID mapping works: external providers have their own event ID
  schemes; the JMAP Calendars CalendarEvent.id must map to/from them.

### Considerations

- *JMAP Calendars adapter pattern* (proxy external providers through JMAP
  Calendars) is the cleanest architecturally. The chat layer sees JMAP
  Calendars and doesn't care about the underlying provider. Adapter
  complexity lives in the JMAP Calendars implementation.
- *Mixed-mode* deployments are common in enterprise contexts: some teams
  have native Calendars, others use Outlook via Exchange. The chat layer
  treats them uniformly via JMAP Calendars; the adapter handles the
  diversity.
- *Identity matching* is the hard part. JMAP Chat `ChatContact.id` is one
  identifier; the external calendar account is another. Linking them
  requires either user self-service ("connect your Google account") or
  enterprise SSO with deterministic mapping.
- *RSVP round-trip latency and failure*: an external provider may be slow
  or unavailable. The RSVP UX must handle pending state and eventual
  consistency.
- *Event-ID mapping*: a JMAP Calendars CalendarEvent.id maps to (and from)
  an external provider's event ID. The JMAP Calendars adapter MUST maintain
  this mapping deterministically; otherwise message references break.

### Common patterns

| System | External integration |
|---|---|
| Slack | Per-user "Connect Google Calendar" or "Connect Outlook" via OAuth; events surface as bot messages. |
| Microsoft Teams | Native Exchange integration; uses Outlook as the underlying calendar; no separate "provider". |
| Discord | No external calendar integration; Scheduled Events are Discord-native. |
| Mattermost | Per-team calendar plugin; OAuth-style external provider connections. |

### Recommended starting point

If your deployment serves both JMAP-native users and users with external
calendars, build an **adapter that surfaces external calendars through JMAP
Calendars** before introducing this chat integration. The chat layer should
not care which calendar provider is in use.

For identity matching: use SSO (OIDC / SAML) where possible to deterministically
link a JMAP Chat identity to an external calendar account. For consumer
deployments without SSO: a per-user OAuth flow ("connect your Google
account") with the linkage stored in the JMAP Calendars adapter.

For RSVP round-trip: render the chat RSVP buttons as optimistic
("you're in"), then asynchronously confirm with the external provider.
On failure (rate-limited, provider down, ACL changed), surface a clear
error and let the user retry. Do not silently lose the RSVP.

For event-ID mapping: the JMAP Calendars adapter stores a deterministic
mapping table from `CalendarEvent.id` to external-provider IDs. The chat
layer references only `CalendarEvent.id`; the mapping is invisible above
the adapter.

---

## 7. Federation considerations

### What the spec leaves open

The calendars spec explicitly puts cross-server calendar binding **out of
scope** for this revision. Cross-server availability lookup is similarly
out of scope. The spec acknowledges that future revisions may address
federated calendar mechanics; until then, calendar binding is server-local.

### What you must decide

- Whether your deployment makes any attempt at federated calendar features,
  or stays strictly server-local.
- If attempting federation: how cross-server `CalendarEvent` references in
  chat messages are handled when the receiving user does not have access
  to the originating server's Calendar.
- Whether your `.ics` parsing applies to attachments arriving via
  federation (`Peer/deliver`) or only to locally-composed messages.
- How federated chat clients display calendar-event MessageActions whose
  URIs they cannot resolve.

### Considerations

- *Strictly server-local* is the spec-conformant default and the lowest-
  risk path. Cross-server calendar features are not supported; users on
  different servers communicate via chat messages but cannot share
  CalendarEvents directly.
- *Best-effort cross-server* attempts to render calendar-event
  MessageActions when possible (the receiving user happens to have access
  to the underlying CalendarEvent via a shared external account) and falls
  back to text-only rendering otherwise.
- *`.ics` parsing on federation*: parsing `.ics` attachments arriving from
  remote peers is higher-risk than parsing locally-composed attachments,
  because the remote sender is not directly known to the receiving server's
  authentication layer. Stronger sanitization and rate limits apply.

### Common patterns

Most chat federation systems do not federate calendar features. Cross-org
calendar federation (when it exists, e.g., Microsoft Outlook federation
between two Exchange organizations) is configured at the calendar layer,
not the chat layer.

### Recommended starting point

**Stay server-local for calendar features.** Render cross-server calendar-
event MessageActions as text-only (the `label` field) if the underlying
CalendarEvent is not accessible; do not surface RSVP affordances unless
the CalendarEvent is locally accessible.

For `.ics` attachments arriving via federation: either decline to parse
(treat as raw attachment) or apply stricter sanitization and stricter rate
limits (e.g., 5 auto-creations per remote sender per recipient per hour).

When JMAP Calendars itself defines federation, revisit this guide and the
underlying spec to integrate.

---

## Cross-references

| Topic | See also |
|---|---|
| Underlying JMAP Calendars protocol | `draft-ietf-jmap-calendars` |
| Underlying JMAP Chat protocol | `draft-atwood-jmap-chat-00.md` |
| The JMAP Chat Calendars spec | `draft-atwood-jmap-chat-calendars-00.md` |
| Main draft deployment topics not specific to calendars | `jmap-chat-implementer-guide.md` |
| Push notification rendering of calendar-event MessageActions | `jmap-chat-push-platform-guide.md` (planned; not yet covered in the push platform guide) |
