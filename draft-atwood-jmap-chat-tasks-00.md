---
title: JMAP Chat Tasks
abbrev: JMAP Chat Tasks
docname: draft-atwood-jmap-chat-tasks-00
category: std
stream: ietf

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC8174:
  RFC8620:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-TASKS:
    title: JMAP for Tasks
    author:
      fullname: Neil Jenkins
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-tasks-06
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-tasks/

informative:
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  JMAP-CHAT-CALENDARS:
    title: JMAP Chat Calendars
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-calendars-00
    date: 2026

--- abstract

This document defines JMAP Chat Tasks, a companion specification to JMAP Chat ({{JMAP-CHAT}}) and JMAP for Tasks ({{JMAP-TASKS}}). It specifies how a Space in JMAP Chat may be associated with a JMAP Tasks TaskList, how tasks are surfaced and linked from chat messages, and how a Task may carry a back-reference to a discussion Chat. The integration is optional: a JMAP Chat deployment without this capability remains fully functional; deployments that advertise it expose richer task-aware behavior.

--- middle

# Introduction

{{JMAP-CHAT}} defines Spaces as named containers for channel conversations, members, and roles. {{JMAP-TASKS}} defines a JMAP capability for managing task lists and individual tasks. This document binds the two: a Space MAY be associated with a TaskList (`taskListId` on Space), Tasks MAY carry a `chatId` back-reference to a discussion Chat, and chat messages MAY surface Tasks via structured MessageAction entries.

## Design philosophy

This specification follows three principles consistent with the broader JMAP Chat corpus:

- **Wire contract is minimal.** Two new optional fields (one on Space, one on Task). No new methods. All integration uses existing JMAP Tasks methods.
- **Permission policy is impl-defined where reasonable.** The closed permission vocabulary of {{JMAP-CHAT}} is unchanged. Deployments MAY map task operations to existing permissions or to deployment-internal authorization (per the per-method auth latitude in §Space Permission Resolution of {{JMAP-CHAT}}).
- **Inspiration without lock-in.** The semantics are loosely modeled on patterns from production chat-and-task systems (Slack Lists, Microsoft Teams Planner integration, Linear/Jira chat bridges) but do not encode any one system's UX as normative.

## Relationship to JMAP Tasks

{{JMAP-TASKS}} is the normative source of truth for task data types and methods. This document does not redefine Task, TaskList, or their methods. It defines only:

- A new optional `taskListId` field on the Space data type (introduced in {{JMAP-CHAT}}).
- A new optional `chatId` field on the Task data type (introduced in {{JMAP-TASKS}}).
- The semantics of the existing `urn:jmap:chat:cap:task` MessageAction / Endpoint type URI (registered in {{JMAP-CHAT}}) in a JMAP Tasks context.
- Recommended permission mappings (with deployment latitude).

Implementations of this specification MUST also implement {{JMAP-TASKS}}. A deployment that supports JMAP Chat but not JMAP Tasks MUST NOT advertise the capability defined here.

## Relationship to JMAP Chat

This document extends one data type in {{JMAP-CHAT}} (Space) with one optional field and refines the semantics of an existing MessageAction / Endpoint URN value. No new JMAP Chat methods are defined.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{JMAP-CHAT}}, and {{JMAP-TASKS}} is used throughout.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:chat:tasks` capability in the JMAP Session object. This capability is meaningful only when both `urn:ietf:params:jmap:chat` and `urn:ietf:params:jmap:tasks` are also advertised on the same account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:tasks"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:tasks"]` is a JSON object with the following fields:

`mayBindTaskList` (Boolean):
: `true` if accounts on this server may associate a TaskList with a Space, `false` otherwise. Deployments that prohibit Space-TaskList binding (for example, multi-tenant deployments where the chat and task accounts are administratively separate) MUST set this to `false`.

`mayBackReferenceChat` (Boolean):
: `true` if tasks may carry a `chatId` back-reference to a discussion Chat (see {{task-chatid}}), `false` otherwise. A deployment MAY support TaskList binding (`mayBindTaskList: true`) without supporting Task-Chat back-references; for example, a deployment that treats Tasks as fire-and-forget items without discussion threads.

# Space Extension: TaskList Binding {#tasklist-binding}

The Space data type defined in {{JMAP-CHAT}} is extended with one optional field when the `urn:ietf:params:jmap:chat:tasks` capability is active.

## taskListId {#task-list-id}

`taskListId` (String, optional):
: A JMAP Tasks `TaskList.id` (per {{JMAP-TASKS}}) bound to this Space. When set, identifies a TaskList that members of this Space MAY view and, subject to permissions, modify. When absent, this Space has no bound TaskList.

The bound TaskList MUST belong to the same account that owns the Space. Cross-account TaskList references are not supported in this revision; deployments wishing to support cross-account or cross-server TaskList binding will require future spec work.

A Space MAY have at most one bound TaskList in this revision. Deployments wishing to expose multiple TaskLists per Space — for example, separate "open" and "archived" lists, or multiple work-stream lists — may achieve this by using multiple Spaces, or by treating the single bound TaskList as the canonical work list and managing additional lists out-of-band.

## Binding and unbinding

Servers that advertise `mayBindTaskList: true` MUST accept patches to `Space.taskListId` from members holding the `"manage_space"` permission. Setting `taskListId` to a non-null value binds the Space to the identified TaskList; setting it to `null` unbinds the Space.

Servers MAY automatically create a new TaskList when a Space is created and bind it (the deployment-defined "auto-bind on create" pattern). When this is done, the server MUST set `taskListId` to the new TaskList's id at Space creation time.

When a Space is destroyed, the bound TaskList SHOULD NOT be automatically destroyed unless explicitly requested by the destroying member. The TaskList persists independently and may be unbound from the Space without being deleted. Tasks within an unbound TaskList retain their content; only the Space association is removed.

## Authorization

Authorization for operations on a bound TaskList — who may read tasks, who may create or modify tasks, who may close them — is deployment-defined. Task authorization in real-world deployments ranges from "anyone in the Space can do anything" (small team) to deeply layered enterprise models (LDAP/AD groups, project-level ACLs, role-based access control, assignment-based access, organizational-unit boundaries, regulatory or compliance constraints). This specification does not attempt to prescribe a single model.

The wire contract this specification establishes is:

- The {{JMAP-TASKS}} permission model is the authoritative authorization layer for Task operations. Servers MUST evaluate JMAP Tasks authorization on every Task operation, regardless of how the operation was initiated (directly via JMAP Tasks or indirectly via chat-layer wiring).
- Unauthorized operations MUST receive the appropriate JMAP Tasks error response (typically `forbidden`).
- Servers MAY use {{JMAP-CHAT}} Space membership and the closed Space-permission vocabulary as inputs to that authorization decision (per §Space Permission Resolution of {{JMAP-CHAT}}, which makes such mappings deployment-defined).
- Deployments MUST document the actual authorization model in user-facing API documentation; the wire protocol does not advertise it.

The chat-layer wiring (binding a TaskList to a Space, surfacing Tasks in messages, creating discussion Chats for Tasks) does not bypass JMAP Tasks authorization. A user lacking JMAP Tasks permission to create or modify a Task on the bound TaskList cannot do so regardless of any chat-side affordance.

`TaskList/set` itself (binding-level changes such as renaming the TaskList or modifying its permissions) is governed by the JMAP Tasks permission model on the account. Deployments MAY layer additional checks tied to {{JMAP-CHAT}} Space permissions (for example, requiring `"manage_space"` to rename the bound TaskList) but the wire contract is the JMAP Tasks model.

# Task Extension: Chat Back-Reference {#task-chatid}

The Task data type defined in {{JMAP-TASKS}} is extended with one optional field when the `urn:ietf:params:jmap:chat:tasks` capability is active and the server advertises `mayBackReferenceChat: true`.

## chatId {#task-chatid-field}

`chatId` (String, optional):
: A JMAP Chat `Chat.id` (per {{JMAP-CHAT}}) referencing a Chat used for discussion of this Task. When set, identifies a Chat (typically of `kind: "channel"` within the same Space whose TaskList contains this Task) that the Task's discussion thread occupies. When absent, this Task has no associated discussion thread.

The referenced Chat MUST exist on the same account as the Task. Cross-account Chat references are not supported. The Chat is not required to be a member of the Space whose TaskList contains the Task; deployments MAY allow a Task to back-reference a Chat outside its bound Space (for example, to thread a task in a general-discussion channel even though the Task belongs to a project Space). In practice, a Chat of `kind: "channel"` within the bound Space's channel set is the most natural pairing.

## Cardinality

Each Task MAY back-reference at most one Chat. A Task wishing multiple discussion threads is expected to keep one canonical thread (via `chatId`) and treat any others as informal references in message bodies, not as structured links.

A Chat MAY be back-referenced by multiple Tasks. The wire model does not constrain this; deployments wishing to enforce "one Task per Chat" or "one Chat per Task" may do so out-of-band.

## Lifecycle

When a Task is destroyed (via `Task/set` destroy), the referenced Chat is NOT automatically destroyed. The Chat persists independently. Deployments wishing to destroy the Chat alongside the Task MUST do so via an explicit follow-up `Chat/set` destroy operation.

When a Chat referenced by one or more Tasks is destroyed, the affected Tasks' `chatId` fields MUST be cleared (set to absent) by the server as part of the destroy operation. Clients reading the updated Tasks observe the absent `chatId` and treat the Tasks as no longer having a discussion thread.

Closing a Task (setting its status to `"completed"` or `"cancelled"` per {{JMAP-TASKS}}) does NOT affect the referenced Chat. Discussion history persists even after the Task is closed.

# Task in Chat Context {#task-in-chat}

A Task ({{JMAP-TASKS}}) MAY be referenced from a chat message via the `urn:jmap:chat:cap:task` MessageAction type (registered in {{JMAP-CHAT}} Endpoint and MessageAction type vocabularies).

## MessageAction format

The MessageAction object has the following shape when its `type` is `urn:jmap:chat:cap:task`:

`type` (String):
: `urn:jmap:chat:cap:task`.

`uri` (String):
: A URI identifying the referenced Task. Two URI forms are recognized:

  - A `jmap:` URI in the form `jmap:task:<accountId>:<taskListId>:<taskId>`, referencing a Task on the named account.
  - A standard `https://` URL pointing to an external task representation (for example, a link to a third-party issue tracker that bridges to JMAP Tasks).

`label` (String, optional):
: Display label (typically the Task's title). Servers SHOULD set this to the Task's `title` when constructing MessageActions from Task records they have access to.

`metadata` (Object, optional):
: MAY include the following type-specific keys:

  - `status` (String) — the Task's current status (e.g., `"needs-action"`, `"in-process"`, `"completed"`, `"cancelled"`).
  - `due` (UTCDate, optional) — the Task's due date.
  - `assigneeIds` (String[], optional) — ChatContact.ids of assigned users.
  - `priority` (Integer, optional) — priority value per {{JMAP-TASKS}}.
  Servers MAY include other type-specific keys; clients MUST ignore unknown keys.

Servers MUST treat the `uri` and `metadata` values as peer-supplied and untrusted; clients MUST validate any URI they intend to dereference per the standard MessageAction handling rules in {{JMAP-CHAT}}.

## Rendering hints

Clients receiving a MessageAction of this type SHOULD render it as a structured task card showing the task title, status, due date (when present), and assignees, with an action affordance to open the task in a task client. Clients that do not support structured rendering MAY fall back to displaying the `label` as a hyperlink.

When the referenced Task is on the same account and JMAP Tasks is available, clients SHOULD fetch the current Task record via `Task/get` rather than relying solely on the MessageAction metadata, which is a snapshot at the time of message composition and may be stale.

## Updates after the message is sent

If the underlying Task is modified or destroyed after the chat message is sent, the message itself is not automatically updated. Clients SHOULD treat the MessageAction metadata as a hint and revalidate against the live Task at display time.

## Task-Chat round-trip

When a Task has both a `chatId` back-reference and a chat message references the Task via MessageAction, clients and servers SHOULD treat the back-reference as the canonical discussion thread. UX flows that "open the task" from a chat message SHOULD navigate to the back-referenced Chat when one exists, falling back to the task-client view otherwise.

# Operations {#operations}

This specification does not define new JMAP methods. All task operations use existing {{JMAP-TASKS}} methods. The integration adds two semantic conveniences:

## Creating a Task from a chat thread

When a user wishes to create a Task whose discussion will live in the current Chat, the recommended pattern is:

1. The client calls `Task/set` (per {{JMAP-TASKS}}) with the desired Task fields and the current Chat's id as the new Task's `chatId`.
2. The server creates the Task on the bound TaskList (or the TaskList specified by the client) and stores the back-reference.
3. The client may then post a confirmation message in the Chat carrying a MessageAction of type `urn:jmap:chat:cap:task` pointing to the new Task; this is OPTIONAL and useful for surfacing the Task in the chat history.

The Task creation MAY succeed even if the current Chat is not bound to the same Space as the target TaskList; back-references are loose by design (per {{task-chatid}}).

## Creating a discussion Chat for an existing Task

When a Task already exists and a user wishes to start a discussion thread for it, the recommended pattern is:

1. The client calls `Chat/set` (per {{JMAP-CHAT}}) to create a new channel Chat in the Space whose TaskList contains the Task. The Chat's name SHOULD reflect the Task (the Task's title is a sensible default).
2. The client calls `Task/set` to update the Task's `chatId` to the new Chat's id.
3. The new Chat begins normal operation; messages posted there are the Task's discussion thread.

The two operations MAY be performed in either order; the server MUST tolerate transient inconsistency (a Task with `chatId` set to a Chat that does not yet exist on this server is a brief inconsistency, resolved when the Chat is created or the chatId is cleared).

# Federation {#federation}

Cross-server federation of TaskList binding is **not in scope** for this revision. A Space's `taskListId` MUST reference a TaskList on the same server. TaskList federation across JMAP servers is a separate problem that may be addressed in a future revision.

Cross-server federation of Task-Chat back-references is similarly out of scope. A Task's `chatId` MUST reference a Chat on the same server.

Cross-server federation of `urn:jmap:chat:cap:task` MessageActions works in the limited sense described in {{task-in-chat}}: the MessageAction carries a URI that may or may not be dereferenceable from the receiving server; the receiving client adapts. A `jmap:` URI may resolve only on the originating account; an external `https://` URL may resolve anywhere.

# Security Considerations {#security}

## Authorization is deployment-defined

The authorization model for Task operations on a bound TaskList is deployment-defined (see {{tasklist-binding}}). This specification does not prescribe who may perform which operations; that decision belongs to the deployment.

What this specification does require is that the {{JMAP-TASKS}} permission model is the authoritative layer: servers MUST evaluate JMAP Tasks authorization on every Task operation, and unauthorized requests MUST receive a JMAP Tasks error response. The chat-layer wiring does not bypass JMAP Tasks authorization. A user lacking JMAP Tasks permissions on the bound TaskList cannot perform Task operations regardless of any chat-side affordance.

## Task content exposure

A Task's content (title, description, status, attachments) is governed by the JMAP Tasks authorization model on the bound TaskList. A Task referenced by a chat MessageAction is not automatically readable by chat members who lack Task access; the MessageAction reveals only the metadata included in the action itself.

When a chat MessageAction includes Task metadata (per {{task-in-chat}}), the sending client/server is choosing to expose that metadata to chat-message recipients. Implementations SHOULD NOT include sensitive fields (Task description, assignee notes, attachments) in MessageAction metadata; the metadata is intended as a hint for the rendered card, not a substitute for proper authorization.

## Chat back-reference leakage

A Task's `chatId` is a reference to a Chat that may itself be private (a member-restricted channel or a closed group chat). Exposing `chatId` to users who can read the Task but not the Chat reveals only that the discussion exists, not its content. Implementations MUST NOT use the `chatId` field as a side channel to disclose Chat membership or content.

Deployments MAY hide the `chatId` field from Task readers who lack access to the referenced Chat. The wire contract permits this (it is equivalent to the Task not exposing the field to that reader), and it reduces information leakage in mixed-access deployments.

## Cross-account references

This document explicitly excludes cross-account `taskListId` and `chatId` references. Implementations MUST validate that referenced TaskLists and Chats belong to the same account as the referencing Space or Task before persisting the binding. Cross-account references are a privilege-escalation risk: an attacker with `"manage_space"` on Space A could otherwise bind Space A to a TaskList on account B that they do not control, gaining a foothold for further attacks.

## Federation surface

Cross-server federation of TaskList binding, Task-Chat back-references, and authorization is explicitly out of scope. Implementations MUST NOT extend the wire protocol to support cross-server queries without explicit user consent flows; this would otherwise expose Task content via federation without the corresponding JMAP Tasks federation specification (which does not yet exist).

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:tasks`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## MessageAction Type URI

The `urn:jmap:chat:cap:task` URI value used in MessageAction and Endpoint contexts is registered by {{JMAP-CHAT}}. This document refines its semantics in a JMAP Tasks context but does not request a new URI registration.

--- back

# Design Influences and Non-Normative Notes

This non-normative section documents the design influences from production chat-and-task systems and the explicit non-decisions where deployment latitude was preferred over prescription.

## Influences

- **Slack Lists** inspired the per-Space single-TaskList model. Slack's Lists feature treats lists as per-channel content with simple membership-driven access. This specification scales that to per-Space single-TaskList (a Space contains multiple channels but shares one task list across them), preserving the simplicity.
- **Microsoft Teams + Microsoft Planner** inspired the rich task metadata in MessageAction. Teams renders Planner cards inline with status, due date, and assignees visible. This specification permits equivalent metadata in `urn:jmap:chat:cap:task` MessageActions.
- **Linear / Jira / GitHub Issues chat integrations** inspired the Task-to-Chat back-reference. Most third-party trackers maintain a per-issue discussion thread linked back from the issue; this specification provides a native equivalent via `Task.chatId`.
- **Slack /todo and todoist bots** inspired the "task creation from a chat thread" pattern documented in {{operations}}. These bots create tasks whose context is the conversation; the specification's `Task.chatId` field gives the same back-reference natively.

## Explicit non-prescriptions

The following design choices were left to deployments rather than prescribed:

- **Multiple TaskLists per Space.** Out of scope for v0; a Space wishing multiple lists uses multiple Spaces or manages additional lists out-of-band.
- **Auto-bind-on-create** (creating a new TaskList when a Space is created). Permitted but not mandated.
- **Task status state machine** (which transitions are valid). Out of scope; the underlying {{JMAP-TASKS}} defines status semantics.
- **Notification policy** (when a Task assignment or status change generates a chat notification). Deployment-defined; not part of the wire contract.
- **Federated TaskList binding.** Explicitly out of scope.
- **Multi-Chat task threading** (one Task with multiple back-referenced Chats). Out of scope; the single `chatId` field is by design.

# Acknowledgements

The author thanks the authors of {{JMAP-CHAT}} for the chat protocol this specification extends, the authors of {{JMAP-TASKS}} for the task protocol this specification integrates with, and the design teams of Slack, Microsoft Teams + Planner, Linear, and Jira for prior art in chat-task integration that informed this work.
