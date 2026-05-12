# JMAP Chat Tasks — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-chat-tasks-00`.
Covers the deployment-defined posture decisions the spec deliberately leaves
open.

Read the draft and `draft-ietf-jmap-tasks` first. This guide does not re-state
normative requirements; it covers what the spec leaves to implementations
and offers patterns and starting points.

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

`draft-atwood-jmap-chat-tasks-00` specifies a minimal wire surface (two new
optional fields, one new capability) and defers authorization, workflow
policy, and operational defaults to implementations. Task authorization in
production deployments ranges from "any team member can manage any task"
(small collaborative groups) to deeply layered enterprise models with
project ACLs, assignment-based access, role-based controls, and regulatory
overlays. The spec cannot predict which model fits any one deployment; this
guide helps implementers think through the choices.

Each section follows the same shape as the broader `jmap-chat-implementer-guide.md`:

1. **What the spec leaves open** — with a citation.
2. **What you must decide.**
3. **Considerations.**
4. **Common patterns** from production chat-and-task systems.
5. **Recommended starting point** — defensible default, not normative.

---

## 1. TaskList binding policy

### What the spec leaves open

`Space.taskListId` (per `draft-atwood-jmap-chat-tasks-00.md` {#tasklist-binding})
is an optional field. The spec says deployments advertise
`mayBindTaskList: true` if they support binding, and the binding can be set
or cleared by members holding `"manage_space"`. The spec does not say *when*
binding happens, *who* initiates it, *whether* a new TaskList is created at
Space creation time, or what happens at Space destruction.

### What you must decide

- Whether your deployment supports TaskList binding at all
  (`mayBindTaskList: true/false`).
- When a new TaskList is created: at Space creation (auto-bind), on first
  task-related action (lazy-bind), or only on explicit admin request
  (manual-bind).
- Whether a Space may be re-bound to a different TaskList after the initial
  bind, and what happens to existing Tasks in the old TaskList on rebind.
- Whether destroying a Space destroys its bound TaskList, leaves it
  orphaned, or un-binds it without destruction.
- Whether multiple "logical" task lists are supported through some
  deployment-specific mechanism (the spec permits only one bound TaskList
  per Space in this revision).

### Considerations

- *Auto-bind-on-create* is the lowest friction: every Space gets a TaskList
  automatically. Cost: storage for TaskLists that may never be used; UX
  cluttered with empty "Tasks" tabs in Spaces that don't need them.
- *Lazy-bind* delays TaskList creation until something actually needs it.
  Lower baseline storage; introduces a "create task list?" UX moment when
  the user first tries to add a task.
- *Manual-bind* requires admin intent. Highest friction; clearest mental
  model; right for enterprise deployments where TaskList provisioning may
  involve approval flows or project management policy.
- *Single TaskList per Space* is the spec's wire constraint. Deployments
  wanting multiple lists per Space (e.g., "Open Bugs" + "Feature Requests")
  achieve this either by using multiple Spaces (one per list), by using
  Task `category` or tag fields within a single TaskList to partition, or
  by managing additional TaskLists out-of-band (not bound to the Space but
  still accessible via JMAP Tasks).
- *Rebind* is messy: existing message references to Tasks in the old list
  may no longer resolve cleanly; clients caching MessageAction metadata
  see stale data. Some deployments allow rebind but flag it as
  consequential; others disallow rebind and require a new Space.
- *Cascade-destroy* (Space destroy → TaskList destroy) loses task history.
  Most deployments prefer un-bind-on-destroy: the TaskList persists for
  archival access.

### Common patterns

| System | Pattern |
|---|---|
| Slack Lists (per channel) | Each channel has at most one list; created on demand when the user first opens the "Lists" tab; persists across channel archive. |
| Microsoft Teams + Planner | Each team auto-gets a Planner board; multiple buckets within the board partition work. |
| Linear | Each project has one issue list; projects are the unit of binding. |
| Jira | Per-project boards; teams may have multiple boards mapped to a single project. |
| Trello-style | One list per board per channel; channels can have multiple boards. |

### Recommended starting point

For consumer/social deployments: **auto-bind-on-create** with the option
to disable per-Space. Users get a working task list without thinking about
it.

For enterprise deployments: **manual-bind**. The TaskList may belong to a
different administrative domain (Planner, Jira, custom tracker) and
binding is a deliberate setup step.

For unknown product context: **lazy-bind**. Postpone TaskList creation
until justified by use.

Re-bind: deployments SHOULD allow re-binding but SHOULD log it to a server
admin audit trail and SHOULD notify Space members when the binding changes.
The wire protocol does not require this; it is deployment-side UX.

Un-bind on Space destroy: deployments SHOULD un-bind rather than
cascade-destroy. The TaskList persists for archival; users wanting to
delete the TaskList do so via `TaskList/set` separately.

For multiple-list scenarios: prefer `Task` `category` or tag fields to
partition within one bound TaskList. Use multiple Spaces if the lists are
truly independent (separate project teams, separate organizational scopes).

---

## 2. Authorization for task operations

### What the spec leaves open

The new tasks spec section "Authorization" explicitly defers the
authorization model to deployments. The wire contract is:

- JMAP Tasks authorization is authoritative for Task operations.
- Servers MUST evaluate JMAP Tasks authorization on every Task operation.
- Servers MAY use Space membership and the closed Space-permission
  vocabulary as inputs to that decision (per the per-method auth
  latitude in `draft-atwood-jmap-chat-00.md`).
- Deployments MUST document the actual authorization model.

### What you must decide

- How JMAP Tasks permissions on the bound TaskList relate to Space
  membership and the Space's closed permission vocabulary.
- Whether tasks have per-task ACLs (a task assigned to specific people is
  only visible/editable by them) or follow the TaskList's authorization
  uniformly.
- Whether the task creator retains special privileges (edit/close their
  own tasks) regardless of the broader permission model.
- Whether assigned users have implicit edit privileges on their own
  assignments.
- How enterprise authorization (LDAP/AD groups, project membership, RBAC)
  interacts with the Space-side and Task-side models.

### Considerations

- *Mirror-Space pattern*: read = Space membership; create/update-own =
  `send`; modify-others = `manage_channels`. Simple; matches chat-product
  intuition. Insufficient for enterprise contexts with project-level ACLs.
- *Mirror-TaskList pattern*: defer entirely to JMAP Tasks' TaskList
  permissions. Most flexible; requires Space members to have JMAP Tasks
  privileges on the bound TaskList.
- *Per-task ACL pattern*: tasks may have additional access controls beyond
  the TaskList (assignment-based, sensitivity-based, project-based).
  Common in enterprise issue trackers.
- *Hybrid (creator-preserves-control) pattern*: anyone with `send` can
  create tasks; only the creator can edit them by default; assignees can
  update status; `manage_channels` can override. Matches user intuition
  about "my task" vs "the team's task".
- *External authorization*: a policy engine decides every Task operation
  based on organizational structure. Used in tightly regulated
  environments.
- *Assignment-based privileges*: assignees can update status, add
  comments, attach files to their assigned tasks regardless of other
  permissions. Most fielded systems support this; the spec is silent
  on it as a default.

### Common patterns

| System | Authorization pattern |
|---|---|
| Slack Lists | Channel members can view; list creator and channel admins can manage; assignees can update their own items. |
| Microsoft Teams + Planner | Team members can view and edit; assigned users have implicit edit on their assignments. |
| Linear | Project members can view; specific roles (admin/lead) can manage; assignees can update status. |
| Jira | Per-project ACLs; reporter/assignee have special privileges; project leads have full management. |
| GitHub Issues | Repo members can view (public) or per-permission (private); assignees can edit; maintainers manage. |

### Recommended starting point

**Hybrid creator-preserves-control + assignment privileges** as the
baseline:

- Read tasks: Space membership.
- Create tasks: Space members with `send` permission.
- Update own task (created by the current user): the creator can edit any
  field.
- Update assigned task (created by someone else but assigned to the
  current user): the assignee can update status, due date, attachments,
  comments. The assignee cannot reassign or delete.
- Update others' task: `manage_channels`.
- Destroy task: creator OR `manage_channels`.
- Close task (move to `"completed"` or `"cancelled"`): creator OR assignee
  OR `manage_channels`.

For enterprise deployments: add an external authorization layer above this.
Define per-project, per-assignee, or per-sensitivity restrictions that
override the Space-side defaults. The wire contract stays the same; the
authorization model becomes richer.

For small/casual deployments: even the hybrid pattern may be overkill.
Read/write everything by any Space member is acceptable.

Document the chosen model. A user seeing `forbidden` on a task action
expected to work needs to be able to find out why.

---

## 3. Task-Chat back-reference UX

### What the spec leaves open

`Task.chatId` (per `draft-atwood-jmap-chat-tasks-00.md` {#task-chatid}) is
the optional back-reference from a Task to a discussion Chat. The spec
permits at most one Chat per Task. It does not say *when* the back-reference
is created, *who* creates the discussion Chat, *what kind* of Chat (channel
or group), or *how* the UX surfaces the linkage.

### What you must decide

- When a discussion Chat is created for a Task: automatically on Task
  creation, on first comment-style interaction, on explicit user action?
- What kind of Chat: channel Chat in the Space (visible to all members)
  vs group Chat (visible to a subset, e.g., assignees only) vs special-
  purpose Chat kind?
- Whether all Tasks have a discussion Chat (uniform) or only some do
  (opt-in)?
- How the linkage appears in UX: a "discussion" tab in the task view, a
  pinned message in the channel, a separate section?
- Lifecycle: what happens to the discussion Chat when the Task is closed?
  When it is destroyed?

### Considerations

- *Always create* (every Task gets a discussion Chat at creation time)
  produces clutter for routine tasks that need no discussion. Some
  deployments accept this; others find it overwhelming.
- *Create on first interaction* defers Chat creation until someone actually
  posts a comment-style message. Lower clutter; latency in surfacing the
  thread when the first reply arrives.
- *Explicit user action* requires "start discussion" as a UX affordance.
  Lowest clutter; lowest discoverability — users may not realize the
  feature exists.
- *Channel Chat in the bound Space* makes the discussion visible to all
  Space members. Good for transparency; bad for sensitive tasks.
- *Group Chat with assignees only* limits visibility. Better for sensitive
  tasks; worse for collaboration with bystanders who might contribute.
- *Special-purpose Chat kind*: introducing a new `kind` value (e.g.,
  `"task-discussion"`) is out of scope for the spec but a deployment could
  conceptually do this internally. The spec's wire constraint is
  `kind: "channel" | "group" | "direct"`; "task-discussion" would not be
  wire-compatible.
- *Close-Task → Chat lifecycle*: when a Task closes, the discussion Chat
  may stay open (preserves history; new comments still possible), be
  archived (read-only), or be locked (no new participants). Most users
  expect "still readable, may comment with caveat".
- *Destroy-Task → Chat lifecycle*: per the spec, destroying a Task does
  NOT automatically destroy the Chat. Deployments may offer "destroy
  task and discussion together" as an explicit two-step operation in the
  UX.

### Common patterns

| System | Task-Discussion pattern |
|---|---|
| Slack Lists | Per-item conversation threads; created on first comment; visible to channel members. |
| Linear | Per-issue discussion thread; visible to project members; created on issue creation. |
| Jira | Per-issue comments + linked Confluence pages; threading varies. |
| Microsoft Teams + Planner | Task-level conversations; created on first comment; persists after task closes. |
| GitHub Issues | Issue comments; threaded; visible per repo permissions. |

### Recommended starting point

**Create discussion Chat on first interaction** (the first comment-style
message posted to the task). Use `kind: "channel"` within the bound Space.
This means:

1. Task created: no Chat exists; `Task.chatId` is absent.
2. User posts the first comment via UX affordance ("Comment on this
   task"). The chat server creates a new channel Chat with `name` set
   to the task title (or "Task: <title>" for clarity). `Task.chatId`
   is updated to the new Chat's id.
3. Subsequent comments are normal messages in the channel.

The discussion Chat is visible to all Space members. Sensitive tasks
requiring restricted discussion: use a group Chat (with `kind: "group"`)
and a manually-curated member list. The wire format supports this; the UX
surfaces it as "private discussion" affordance for sensitive task types.

Close-Task → Chat: leave the discussion Chat open and readable; post a
system message noting the task closure for context.

Destroy-Task → Chat: provide a "destroy task and discussion together" UX
that performs both operations atomically (the server-side flow is two
sequential operations, but the user sees one action).

Surface the linkage prominently in both directions: the task view shows
"View discussion" linking to the Chat; the Chat header shows "About this
task" linking to the Task.

---

## 4. Notification policy

### What the spec leaves open

When a task is created, assigned, updated, or closed, *should* a chat-level
notification fire? The spec does not say. Notification policy in chat-
plus-task systems is a major UX area with significant per-deployment
variation.

### What you must decide

- Which task events generate chat notifications: creation, assignment,
  status change, due-date change, comment, all of these?
- Where notifications appear: in the bound Space's general channel, in
  the task's discussion Chat, in a private DM to assignees, all of the
  above?
- Notification urgency: regular notification, mention-equivalent
  elevated urgency, silent badge update?
- Aggregation: one notification per change, batched daily digest,
  threshold-based?
- Per-user preferences: opt-in/opt-out, granularity (all tasks vs my
  tasks vs assigned tasks)?

### Considerations

- *Every change generates a notification* is overwhelming. Users tune out
  or disable notifications entirely.
- *Selective notification* (only "high-signal" events: assignment to me,
  mention in task comment, status change on my task, approaching due
  date) is the usable middle ground.
- *Notification location*: posting to the bound Space's general channel
  clutters the channel; posting to the task discussion Chat is contextual
  but easy to miss; DM to assignee is high-signal but easy to overlook in
  a notification flood.
- *Urgency*: task assignments may warrant elevated notification (similar
  to a direct mention); routine status updates do not.
- *Aggregation*: daily digest is useful for low-priority signals; live
  notifications for high-priority. Most products combine.
- *Per-user preferences* are essential. Users in different roles want
  different notification surfaces.

### Common patterns

| System | Notification pattern |
|---|---|
| Slack | Per-user preferences; default-DM for personal assignments; channel post for general updates. |
| Microsoft Teams + Planner | Activity feed shows task changes; assignment generates a notification. |
| Linear | Per-user notification preferences; @-mentions in task comments elevated; status changes silent by default. |
| Jira | Highly configurable; per-project per-event preferences; smart filtering. |
| GitHub Issues | Per-repo subscription model; thread participants get notifications; opt-in for new issues. |

### Recommended starting point

**Selective per-user-relevant notifications** as the baseline:

- Task assignment to me: notification in the task discussion Chat (or DM if
  no Chat yet). Elevated urgency (mention-equivalent).
- Task I created updated by someone else: notification in the task
  discussion Chat. Normal urgency.
- Task I'm assigned status changed by someone else: notification in the
  task discussion Chat. Normal urgency.
- Task I'm not assigned, not created, etc., changed: no notification.
- Mention in task comment: notification per the chat-layer's mention
  rules.
- Approaching due date (24 hours before, configurable): DM to assignees.

Per-user preferences: surface notification settings in user settings.
Default to the above; let users tune granularity.

Avoid the "general channel firehose" pattern (every task event posted to
the bound Space's general channel). It clutters chat with low-signal
noise.

For digest-style aggregation: offer "daily summary of my tasks" as an
opt-in.

---

## 5. Task rendering in chat

### What the spec leaves open

The spec says `urn:jmap:chat:cap:task` MessageActions carry task metadata
and clients SHOULD render them as task cards. It does not prescribe the
card layout, which fields appear, or how the rendering responds to
metadata updates.

### What you must decide

- Which task fields are surfaced in the card: title, status, due date,
  assignees, priority, description excerpt, attachment indicator?
- Whether the card refreshes from JMAP Tasks (live data) or uses the
  cached MessageAction metadata (snapshot at composition time)?
- Action affordances embedded in the card: open task, mark complete,
  reassign, comment, none of these?
- Compact vs expanded rendering: small inline indicator vs full card?
- How tasks with no due date or unassigned tasks render differently from
  fully-specified tasks?

### Considerations

- *Live data refresh* keeps the card accurate but adds JMAP Tasks API
  calls on every render. Caching with periodic refresh is the practical
  middle ground.
- *Snapshot rendering* shows what the sender saw at composition time. May
  go stale; useful for historical context ("at the time I sent this, the
  task was assigned to X").
- *Inline action affordances* (mark complete, reassign) accelerate
  workflow but require careful authorization checks. Most products limit
  inline actions to high-confidence ones (status toggle) and link out for
  edits.
- *Compact rendering* fits chat-flow better; expanded rendering surfaces
  more context. Most products auto-select based on screen size or user
  preference.
- *Empty-field rendering*: a task with no due date or assignee should
  render gracefully — "No due date" placeholder, "Unassigned" placeholder.
  Not as missing fields in the card.

### Common patterns

| System | Card content |
|---|---|
| Slack Lists shared in chat | Title, status, due date, assignee avatars; compact inline rendering with click-to-expand. |
| Microsoft Teams + Planner | Title, status pill, due date, assignee avatars; full card with action buttons. |
| Linear shared in chat (Linear-Slack integration) | Title, status, priority, assignee; compact with click-to-open. |
| Jira shared in chat | Title, status, priority, assignee, issue type; varies by integration. |

### Recommended starting point

**Compact inline card** showing: task title, status pill, due date (if
present), assignee avatars (up to 3, then "+N"). Click expands to full
card or opens in task UI.

**Refresh from JMAP Tasks on display** with short cache (5-15 seconds for
status changes; longer for less-volatile fields). Show the cached
metadata immediately and update the card when refresh completes.

**Inline action affordances**: "Mark complete" button visible to assignees
and creator. "Open task" button visible to all viewers. "Reassign" and
other edit operations link out to the task UI rather than being inline.

**Empty fields**: show as placeholders, not omissions. "No due date" is
clearer than the absence of a due-date row.

---

## 6. Workflow patterns

### What the spec leaves open

The spec describes Task data shape and back-reference but does not
prescribe workflow models. Production deployments use a wide range:
kanban boards, simple lists, assignment-driven flows, deadline-driven
flows, dependency-driven flows.

### What you must decide

- Which workflow model your product UX presents: list (linear), kanban
  (columns by status), board (grouped by assignee or category), calendar
  (sorted by due date), or hybrid?
- Whether tasks have inherent ordering (priority, manual ordering) or are
  unordered (status + due date determines display)?
- Whether sub-tasks (parent/child task relationships) are supported in
  your UX?
- Whether dependencies between tasks are tracked and surfaced?
- Whether recurring tasks are supported, and how?

### Considerations

- *List view* is the simplest. Slack Lists, most casual products. Works
  well for personal task tracking and small-team todo lists.
- *Kanban* (columns by status) suits team workflows where tasks flow
  through stages. Adopted by Linear, Jira, GitHub Projects, Trello.
- *Calendar view* suits deadline-driven work (sales, event planning,
  publishing).
- *Sub-tasks* add hierarchy. Some users love them; some find them
  bewildering. The JMAP Tasks spec defines them; deployments choose
  whether to surface them.
- *Dependencies* (Task A blocks Task B) are an enterprise feature. Most
  casual products skip them.
- *Recurring tasks* (daily standup checklist, weekly review) are common
  in personal-productivity contexts. JMAP Tasks supports recurrence.

### Common patterns

| System | Primary workflow model |
|---|---|
| Slack Lists | Linear list with status columns; per-item attributes. |
| Microsoft Planner | Buckets (columns) with cards. Assignment-driven flow. |
| Linear | Kanban + list; cycle-based (sprint-like time-boxing). |
| Jira | Highly configurable; kanban, scrum, or custom. |
| GitHub Projects | Kanban with custom fields. |
| Asana | Multiple views (list, board, calendar, timeline); same underlying tasks. |
| Trello | Strict kanban (columns and cards). |

### Recommended starting point

For **casual/social deployments**: simple list view with status indicator
per task. Per-task fields visible: title, assignee, due date, status. No
sub-tasks, no dependencies.

For **team productivity deployments**: kanban view with configurable
columns (default: "To Do" / "In Progress" / "Done"). Per-task fields
visible: title, assignee, due date, status, priority. Sub-tasks supported.

For **enterprise/project-management deployments**: configurable view (list,
kanban, calendar, dependency graph). Full feature set: sub-tasks,
dependencies, recurrence, custom fields. Multiple view modes per user.

Implement at most one view model in v1; add more based on user demand.
The JMAP Tasks wire model supports all of the above; the choice is purely
UX.

---

## 7. External task tracker integration

### What the spec leaves open

The spec assumes JMAP Tasks. Many production deployments need to bridge to
external trackers (Linear, Jira, GitHub Issues, Asana, custom corporate
trackers). The spec does not prescribe how this bridging works.

### What you must decide

- Whether your deployment proxies external trackers through JMAP Tasks
  (so users see one unified API regardless of source)?
- Whether your deployment supports mixed-mode (some Spaces bound to
  JMAP-native TaskLists, others to proxied external lists)?
- How identity matching works between JMAP Chat ChatContact identities
  and external tracker user accounts?
- How task-ID mapping works between JMAP Tasks Task.id and external
  tracker IDs (e.g., Linear's issue IDs, Jira's issue keys)?
- How task changes round-trip: a user updates a task in chat, the chat
  server calls JMAP Tasks, the JMAP Tasks adapter calls the external
  tracker's API.

### Considerations

- *JMAP Tasks adapter pattern* (proxy external trackers through JMAP
  Tasks) is the cleanest architecturally. The chat layer sees JMAP Tasks
  uniformly; adapter complexity lives in JMAP Tasks.
- *Mixed-mode* deployments are common: some teams use native JMAP Tasks
  for casual work; others use Jira for engineering. The chat layer
  treats them uniformly via JMAP Tasks adapter.
- *Identity matching*: SSO (OIDC/SAML) provides deterministic linkage in
  enterprise contexts. Consumer deployments may use per-user OAuth flows
  ("connect your Linear account") with the linkage stored in the
  adapter.
- *Task-ID mapping*: a JMAP Tasks Task.id maps to (and from) an external
  tracker's ID. The adapter stores this mapping. Bidirectional
  round-trip requires the adapter to receive change events from the
  external tracker (webhook, polling, or push notification).
- *Round-trip latency and failure*: external trackers may be slow or
  unavailable. Optimistic UX with eventual consistency; clear error
  surfacing on failure.

### Common patterns

| System | External integration |
|---|---|
| Slack | Per-channel app installations (Linear, Jira, Asana, GitHub); webhooks for inbound updates; slash commands for outbound. |
| Microsoft Teams | Power Automate connectors; Power Platform integration patterns. |
| Linear | Slack and Discord integrations via Linear's own bridges. |
| Mattermost | Per-team app/integration; webhook-based. |
| Discord | Bot-based integrations; no native task tracker support. |

### Recommended starting point

If your deployment serves both JMAP-native users and external-tracker
users, build a **JMAP Tasks adapter** that proxies the external tracker
through JMAP Tasks before introducing this chat integration.

For identity matching: use SSO where possible. For consumer deployments
without SSO: per-user OAuth ("connect your Linear account") with linkage
stored in the JMAP Tasks adapter.

For task-ID mapping: the adapter stores a deterministic mapping table.
The chat layer references only `Task.id`; the mapping is invisible above
the adapter.

For round-trip updates: render optimistically ("status updated") with
asynchronous confirmation. On failure, surface a clear error and let the
user retry.

For change ingestion from the external tracker: webhook-based ingestion
where possible; polling fallback. The JMAP Tasks adapter is responsible
for translating external-tracker events into JMAP Tasks state changes,
which then propagate to the chat layer via standard JMAP push.

---

## 8. Federation considerations

### What the spec leaves open

Cross-server TaskList binding and Task-Chat back-references are out of
scope for the current spec revision. The spec acknowledges that future
work may address federated task mechanics; until then, task binding is
server-local.

### What you must decide

- Whether your deployment makes any attempt at federated task features,
  or stays strictly server-local.
- If attempting federation: how cross-server `Task` references in chat
  messages are handled when the receiving user does not have access to
  the originating server's TaskList.
- How federated chat clients display `urn:jmap:chat:cap:task` MessageActions
  whose URIs they cannot resolve.

### Considerations

- *Strictly server-local* is the spec-conformant default and the
  lowest-risk path.
- *Best-effort cross-server* attempts to render task MessageActions when
  possible and falls back to text-only rendering otherwise.
- *Cross-server task state changes*: the spec does not define how a Task
  state change on one server propagates to a chat message on another
  server. Out of scope.

### Common patterns

Most chat federation systems do not federate task features. Cross-org
task federation (e.g., Linear's cross-workspace features) is configured
at the task layer, not the chat layer.

### Recommended starting point

**Stay server-local for task features.** Render cross-server task
MessageActions as text-only (the `label` field) if the underlying Task is
not accessible; do not surface inline action affordances unless the Task
is locally accessible.

When JMAP Tasks itself defines federation, revisit this guide and the
underlying spec.

---

## Cross-references

| Topic | See also |
|---|---|
| Underlying JMAP Tasks protocol | `draft-ietf-jmap-tasks` |
| Underlying JMAP Chat protocol | `draft-atwood-jmap-chat-00.md` |
| The JMAP Chat Tasks spec | `draft-atwood-jmap-chat-tasks-00.md` |
| Main draft deployment topics not specific to tasks | `jmap-chat-implementer-guide.md` |
| Calendar-and-tasks shared patterns | `jmap-chat-calendars-guide.md` |
| Push notification rendering of task MessageActions | `jmap-chat-push-platform-guide.md` |
