---
title: JMAP Chat Calendars
abbrev: JMAP Chat Calendars
docname: draft-atwood-jmap-chat-calendars-00
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
  RFC5545:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CALENDARS:
    title: JMAP for Calendars
    author:
      fullname: Neil Jenkins
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-calendars-26
    date: 2026
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-calendars/

informative:
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  RFC9073:

--- abstract

This document defines JMAP Chat Calendars, a companion specification to JMAP Chat ({{JMAP-CHAT}}) and JMAP for Calendars ({{JMAP-CALENDARS}}). It specifies how a Space in JMAP Chat may be associated with a JMAP Calendars Calendar, how calendar events are surfaced and acted upon within chat messages, how .ics attachments may be recognized, and how availability information is exposed for in-chat scheduling. The integration is optional: a JMAP Chat deployment without this capability remains fully functional; deployments that advertise it expose richer calendar-aware behavior.

--- middle

# Introduction

{{JMAP-CHAT}} defines Spaces as named containers for channel conversations, members, and roles. {{JMAP-CALENDARS}} defines a JMAP capability for managing calendars and events. This document binds the two: a Space MAY be associated with a Calendar (`calendarId` on Space), calendar events MAY be surfaced as structured chat messages with RSVP semantics, and availability information from JMAP Calendars MAY be exposed for in-chat scheduling decisions.

## Design philosophy

This specification follows three principles consistent with the broader JMAP Chat corpus:

- **Wire contract is minimal.** Few new fields, no new methods (the integration reuses existing JMAP Calendars methods). Deployments compose richer behavior using existing primitives.
- **Permission policy is impl-defined where reasonable.** The closed permission vocabulary of {{JMAP-CHAT}} is unchanged. Deployments MAY map calendar operations to existing permissions or to deployment-internal authorization (per the per-method auth latitude in §Space Permission Resolution of {{JMAP-CHAT}}).
- **Inspiration without lock-in.** The semantics are loosely modeled on patterns from production chat-and-calendar systems (Discord Scheduled Events, Microsoft Teams channel meetings, Slack calendar app integrations), but the spec does not encode any one system's UX as normative.

## Relationship to JMAP Calendars

{{JMAP-CALENDARS}} is the normative source of truth for calendar data types and methods. This document does not redefine Calendar, CalendarEvent, or their methods. It defines only:

- A new optional `calendarId` field on the Space data type (introduced in {{JMAP-CHAT}}).
- The semantics of the existing `urn:jmap:chat:cap:calendar-event` and `urn:jmap:chat:cap:availability` MessageAction / Endpoint type URIs (registered in {{JMAP-CHAT}}) in a JMAP Calendars context.
- Server behavior for optional .ics attachment parsing.
- Privacy considerations specific to in-chat availability lookup.

Implementations of this specification MUST also implement {{JMAP-CALENDARS}}. A deployment that supports JMAP Chat but not JMAP Calendars MUST NOT advertise the capability defined here.

## Relationship to JMAP Chat

This document extends one data type in {{JMAP-CHAT}} (Space) and refines the semantics of two existing MessageAction / Endpoint URN values. No new JMAP Chat methods are defined. The chat-side wire surface remains nearly unchanged.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}}, {{JMAP-CHAT}}, and {{JMAP-CALENDARS}} is used throughout.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:chat:calendars` capability in the JMAP Session object. This capability is meaningful only when both `urn:ietf:params:jmap:chat` and `urn:ietf:params:jmap:calendars` are also advertised on the same account.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:chat:calendars"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:chat:calendars"]` is a JSON object with the following fields:

`mayBindCalendar` (Boolean):
: `true` if accounts on this server may associate a Calendar with a Space, `false` otherwise. Deployments that prohibit Space-Calendar binding (for example, multi-tenant deployments where the chat and calendar accounts are administratively separate) MUST set this to `false`.

`supportsIcsParsing` (Boolean):
: `true` if the server parses `.ics` ({{RFC5545}}) attachments to surface structured CalendarEvent representations; `false` otherwise. See {{ics-parsing}}.

`supportsAvailabilityLookup` (Boolean):
: `true` if the server exposes `Principal/getAvailability` ({{JMAP-CALENDARS}}) in the in-chat context defined here; `false` otherwise. See {{availability}}.

# Space Extension: Calendar Binding {#calendar-binding}

The Space data type defined in {{JMAP-CHAT}} is extended with one optional field when the `urn:ietf:params:jmap:chat:calendars` capability is active.

## calendarId {#calendar-id}

`calendarId` (String, optional):
: A JMAP Calendars `Calendar.id` (per {{JMAP-CALENDARS}}) bound to this Space. When set, identifies a Calendar that members of this Space MAY view and, subject to permissions, modify. When absent, this Space has no bound Calendar.

The bound Calendar MUST belong to the same account that owns the Space. Cross-account Calendar references are not supported in this revision of the specification; deployments wishing to support cross-account or cross-server Calendar binding will require future spec work.

## Binding and unbinding

Servers that advertise `mayBindCalendar: true` MUST accept patches to `Space.calendarId` from members holding the `"manage_space"` permission. Setting `calendarId` to a non-null value binds the Space to the identified Calendar; setting it to `null` unbinds the Space.

Servers MAY automatically create a new Calendar when a Space is created and bind it (the deployment-defined "auto-bind on create" pattern). When this is done, the server MUST set `calendarId` to the new Calendar's id at Space creation time.

When a Space is destroyed, the bound Calendar SHOULD NOT be automatically destroyed unless explicitly requested by the destroying member. The Calendar persists independently and may be unbound from the Space without being deleted.

## Authorization

Authorization for operations on a bound Calendar — who may read events, who may create or modify events, who may RSVP — is deployment-defined. Calendar authorization in real-world deployments ranges from "anyone in the Space can do anything" (small team) to deeply layered enterprise models (LDAP/AD groups, OIDC claim mappings, role-based access control, sensitivity labels, delegation models, organizational-unit boundaries, regulatory compliance constraints). This specification does not attempt to prescribe a single model.

The wire contract this specification establishes is:

- The {{JMAP-CALENDARS}} permission model is the authoritative authorization layer for Calendar operations. Servers MUST evaluate JMAP Calendars authorization on every Calendar operation, regardless of how the operation was initiated (directly via JMAP Calendars or indirectly via chat-layer wiring).
- Unauthorized operations MUST receive the appropriate JMAP Calendars error response (typically `forbidden`).
- Servers MAY use {{JMAP-CHAT}} Space membership and the closed Space-permission vocabulary as inputs to that authorization decision (per §Space Permission Resolution of {{JMAP-CHAT}}, which makes such mappings deployment-defined).
- Deployments MUST document the actual authorization model in user-facing API documentation; the wire protocol does not advertise it.

The chat-layer wiring (binding a Calendar to a Space, surfacing CalendarEvents in messages) does not bypass JMAP Calendars authorization. A user lacking JMAP Calendars permission to RSVP to an event cannot RSVP regardless of any chat-side affordance; the underlying `CalendarEvent/set` returns `forbidden`.

`Calendar/set` itself (binding-level changes such as renaming the Calendar or modifying its permissions) is governed by the JMAP Calendars permission model on the account. Deployments MAY layer additional checks tied to {{JMAP-CHAT}} Space permissions (for example, requiring `"manage_space"` to rename the bound Calendar) but the wire contract is the JMAP Calendars model.

# CalendarEvent in Chat Context {#event-in-chat}

A CalendarEvent ({{JMAP-CALENDARS}}) MAY be referenced from a chat message via the `urn:jmap:chat:cap:calendar-event` MessageAction type (registered in {{JMAP-CHAT}} Endpoint and MessageAction type vocabularies).

## MessageAction format

The MessageAction object has the following shape when its `type` is `urn:jmap:chat:cap:calendar-event`:

`type` (String):
: `urn:jmap:chat:cap:calendar-event`.

`uri` (String):
: A URI identifying the referenced CalendarEvent. Two URI forms are recognized:

  - A `jmap:` URI in the form `jmap:calendarevent:<accountId>:<calendarEventId>`, referencing a CalendarEvent on the named account. This form requires the receiving client to have JMAP Calendars access to the named account.
  - A standard `webcal://` or `https://` URL pointing to an iCalendar object retrievable out-of-band.

`label` (String, optional):
: Display label (for example, the event's summary). Servers SHOULD set this to the event's `title` when constructing MessageActions from CalendarEvent records they have access to.

`metadata` (Object, optional):
: MAY include the following type-specific keys:
  - `startsAt` (UTCDate) — event start time.
  - `endsAt` (UTCDate) — event end time.
  - `location` (String) — event location.
  - `participantCount` (UnsignedInt) — count of participants on the underlying CalendarEvent.
  Servers MAY include other type-specific metadata keys; clients MUST ignore unknown keys.

Servers MUST treat the `uri` and `metadata` values as peer-supplied and untrusted; clients MUST validate any URI they intend to dereference per the standard MessageAction handling rules in {{JMAP-CHAT}}.

## Rendering hints

Clients receiving a MessageAction of this type SHOULD render it as a structured event card showing the event title, time, and (when present) location, with an action affordance to RSVP (see {{rsvp}}) and an action affordance to open the event in a calendar client. Clients that do not support structured rendering MAY fall back to displaying the `label` as a hyperlink.

When the referenced CalendarEvent is on the same account and JMAP Calendars is available, clients SHOULD fetch the current CalendarEvent record via `CalendarEvent/get` rather than relying solely on the MessageAction metadata, which is a snapshot at the time of message composition and may be stale.

## Updates after the message is sent

If the underlying CalendarEvent is modified or destroyed after the chat message is sent, the message itself is not automatically updated. Clients SHOULD treat the MessageAction metadata as a hint and revalidate against the live CalendarEvent at display time. Servers MAY define deployment-specific mechanisms to update or invalidate messages whose referenced CalendarEvent has changed; this is out of scope for the wire protocol.

# RSVP Handling {#rsvp}

This specification does not define a new method for RSVPing to events. RSVPs are performed using {{JMAP-CALENDARS}}'s existing `CalendarEvent/set` method to patch the `participants` map on the target event. The chat layer's contribution is solely the UX wiring: a message carrying a `urn:jmap:chat:cap:calendar-event` MessageAction can render RSVP buttons that, when activated, trigger the appropriate JMAP Calendars call.

## Recommended client flow

A client receiving a chat message with a calendar-event MessageAction and wishing to expose an RSVP UI SHOULD:

1. Identify the recipient user's `Principal.id` on the relevant account.
2. Compose a `CalendarEvent/set` request patching the `participants/{principalId}/participationStatus` field on the referenced CalendarEvent to one of `"accepted"`, `"declined"`, `"tentative"`, or `"delegated"` per {{JMAP-CALENDARS}}.
3. Submit the request via the standard JMAP transport.
4. On success, the chat client SHOULD update its local view of the message (re-fetch the CalendarEvent or update the cached participant state).

The chat server is not involved in the RSVP transaction beyond having delivered the original message. The RSVP is a JMAP Calendars transaction; the chat layer is a launchpad.

## Federation considerations

When a calendar-event MessageAction is received via {{JMAP-CHAT-FED}} `Peer/deliver`, the referenced CalendarEvent likely resides on a different server than the receiving chat user. Three patterns are valid:

1. The CalendarEvent has been replicated to the receiving user's JMAP Calendars account (via existing JMAP Calendars sharing or scheduling mechanisms). The RSVP flows over the local JMAP Calendars account.
2. The URI is an out-of-band `webcal://` or `https://` URL. The client opens it externally; RSVP happens outside JMAP entirely.
3. The receiving user has no access to the CalendarEvent. The client renders the message with the metadata snapshot only; no RSVP affordance is exposed.

This document does not define a federated RSVP protocol; cross-server RSVP coordination is out of scope.

# ICS Attachment Handling {#ics-parsing}

Servers MAY parse `.ics` ({{RFC5545}}) attachments dropped into chat messages and surface them as structured calendar information. When this capability is offered, servers MUST advertise `supportsIcsParsing: true` in the account capability object.

## Server-side parsing

When a Message is created with an attachment whose `contentType` is `text/calendar` and the server has `supportsIcsParsing: true`, the server MAY perform one or more of the following actions:

- Parse the iCalendar object and surface a derived CalendarEvent on the account's default Calendar (or a deployment-defined target Calendar).
- Synthesize a MessageAction of type `urn:jmap:chat:cap:calendar-event` referencing the newly created or matched CalendarEvent.
- Decline to parse (return the message with the raw attachment intact), in which case the client may parse client-side or render the attachment as a generic file.

The specific behavior is deployment-defined. Deployments MUST NOT auto-RSVP on the recipient's behalf as a result of `.ics` parsing; RSVPs require explicit user action.

## Auto-creation safety

When server-side parsing creates a new CalendarEvent on a recipient's calendar, the server SHOULD store the event as a tentative state (`participationStatus: "needs-action"` for the recipient) and SHOULD NOT mark it as accepted. The recipient retains the choice of whether to RSVP.

Servers SHOULD rate-limit auto-creation of CalendarEvents from `.ics` attachments to prevent a malicious sender from flooding a recipient's calendar. A reasonable default is no more than 10 new CalendarEvents auto-created per sender per hour; deployments tune this.

Servers MUST reject `.ics` attachments that exceed a deployment-defined size limit (a few hundred kilobytes is typical); large iCalendar objects can be denial-of-service vectors.

## Client-side parsing alternative

Servers MAY choose not to parse `.ics` attachments server-side at all. In that case, clients MAY perform parsing themselves and present RSVP affordances based on the parsed content; the resulting CalendarEvent creation (if any) is a normal `CalendarEvent/set` initiated by the client. Server-side parsing is an optimization for clients that prefer not to ship a full iCalendar parser, not a replacement for client-side support.

# Availability Lookup {#availability}

The `urn:jmap:chat:cap:availability` MessageAction / Endpoint type URI (registered in {{JMAP-CHAT}}) MAY surface a hyperlink to a Principal whose free/busy availability can be queried via {{JMAP-CALENDARS}} `Principal/getAvailability`.

## MessageAction format

When `type` is `urn:jmap:chat:cap:availability`:

`uri` (String):
: A URI identifying the Principal. Two forms are recognized:

  - A `jmap:` URI in the form `jmap:principal:<accountId>:<principalId>`.
  - A standard scheduling URL.

`label` (String, optional):
: Display label (typically the Principal's name).

`metadata` (Object, optional):
: MAY include `lookupWindow` (Object, with `startsAt` and `endsAt` fields) hinting at the time range the sender expects the recipient to query.

## Privacy

`Principal/getAvailability` exposes occupancy information (whether time slots are busy or free) — not event details, but still privacy-sensitive. The policy governing in-chat availability lookup is deployment-defined.

Real-world deployments choose along axes including: same-Space membership; same-account-not-same-Space; cross-account (federated); same-organizational-unit (in enterprise contexts with directory integration); explicit per-principal consent; time-of-day or working-hours windows; sensitivity tags on the queried calendar; regulatory or compliance overlays. This specification does not prescribe values along these axes.

The wire contract this specification establishes is:

- Servers MAY expose `Principal/getAvailability` in response to chat-context queries.
- Servers MUST reject (with the appropriate JMAP Calendars error) any query that violates their configured policy.
- Deployments MUST document the policy in user-facing product documentation; users SHOULD NOT have to read source code to learn who can see their free/busy state.

## Granularity

Implementations SHOULD expose availability in slot-based form (busy / free for each contiguous interval), not event-detail form. The chat context is for "find a time"; event titles, locations, and participant lists belong to JMAP Calendars proper and require richer authorization.

# Federation {#federation}

Cross-server federation of calendar binding is **not in scope** for this revision. A Space's `calendarId` MUST reference a Calendar on the same server. Calendar federation across JMAP servers (where two servers' Calendars are synchronized or one references the other) is a separate problem that may be addressed in a future revision of this or a different specification.

Cross-server federation of calendar-event MessageActions and ICS attachments works in the limited sense described in {{event-in-chat}}: the MessageAction carries a URI that may or may not be dereferenceable from the receiving server; the receiving client adapts.

Cross-server availability lookup is out of scope for this specification. Deployments wishing to enable it MUST do so via mechanisms outside this specification; the wire contract here does not extend across the federation boundary.

# Security Considerations {#security}

## RSVP authorization

RSVPs are performed via {{JMAP-CALENDARS}} `CalendarEvent/set`, which has its own authorization model. The chat layer does not bypass that model. A user who lacks JMAP Calendars permissions to modify their own participation status cannot RSVP regardless of chat-layer permissions.

## Availability privacy

`Principal/getAvailability` exposes occupancy patterns. Even without event details, occupancy data can reveal behavioral patterns (work hours, lunch routines, periodic meetings). Deployments MUST treat availability data as privacy-sensitive and apply access controls per {{availability}}. Implementations MUST NOT expose availability data to clients that lack the corresponding JMAP Calendars privileges, even when the JMAP Chat layer would otherwise allow the query.

## ICS parsing safety

iCalendar parsers have historically been a source of denial-of-service and injection vulnerabilities. Implementations of optional `.ics` parsing MUST:

- Enforce a strict size limit on iCalendar object bytes.
- Limit recursion depth in parsed `VCALENDAR`, `VEVENT`, `VTODO`, etc. structures.
- Reject calendar objects whose effective recurrence patterns would produce an unreasonable number of expanded instances (a calendar event recurring every second for ten years).
- Sanitize text fields (`SUMMARY`, `DESCRIPTION`, `LOCATION`) for control characters and URL injection before surfacing them in chat UI.
- Never auto-accept invitations on the recipient's behalf (per {{ics-parsing}}).

## Calendar binding integrity

A Space's `calendarId` is set by members holding the `"manage_space"` permission. A compromised admin account could bind a malicious Calendar to a Space or unbind the legitimate one. Deployments SHOULD audit `Space/set` patches that change `calendarId` to a separate log; users of the affected Space SHOULD be notified when the binding changes. The chat wire protocol does not require this notification; it is a deployment-side UX choice.

## Federation surface

This document explicitly excludes cross-server calendar binding and availability lookup. Implementations MUST NOT extend the wire protocol to support cross-server queries without explicit user consent flows. A future revision MAY define federated calendar mechanics; until then, the conservative default is "calendar binding is server-local".

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:chat:calendars`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## MessageAction Type URI

The `urn:jmap:chat:cap:calendar-event` and `urn:jmap:chat:cap:availability` URI values used in MessageAction and Endpoint contexts are registered by {{JMAP-CHAT}}. This document refines their semantics in a JMAP Calendars context but does not request new URI registrations.

--- back

# Design Influences and Non-Normative Notes

This non-normative section documents the design influences from production chat-and-calendar systems and the explicit non-decisions where deployment latitude was preferred over prescription.

## Influences

- **Discord Scheduled Events** inspired the single-Calendar-per-Space binding model. Discord scopes events to a server (analogous to a Space); members can opt in but there is no per-channel events surface. This kept the wire surface small.
- **Microsoft Teams channel meetings** inspired the rich participation status round-trip. Teams treats RSVPs as first-class state that propagates between Outlook calendars and Teams chat. This specification uses {{JMAP-CALENDARS}} participation status as the equivalent first-class representation.
- **Slack calendar app integrations** inspired the optional `.ics` parsing pattern. Slack apps surface `.ics` shares as structured event cards; this specification permits but does not mandate equivalent server-side parsing.
- **Microsoft Teams Scheduling Assistant** inspired the in-chat availability lookup. Teams renders free/busy slots inline during scheduling conversations. This specification exposes the underlying mechanism (`Principal/getAvailability`) without prescribing a UX.

## Explicit non-prescriptions

The following design choices were left to deployments rather than prescribed:

- **Per-channel calendars** (multiple Calendars per Space, one per channel). Out of scope; a Space wishing per-channel granularity uses multiple Spaces.
- **Auto-bind-on-create** (creating a new Calendar when a Space is created). Permitted but not mandated.
- **Reactions as RSVPs.** Explicitly not adopted; reactions stay for true emoji reactions, RSVPs use {{JMAP-CALENDARS}} participation status.
- **Federated calendar binding.** Explicitly out of scope.
- **Cross-Space availability disclosure defaults.** Deployment-defined.
- **Auto-RSVP on `.ics` parsing.** Explicitly forbidden; recipients must consent.

# Acknowledgements

The author thanks the authors of {{JMAP-CHAT}} for the chat protocol this specification extends, the authors of {{JMAP-CALENDARS}} for the calendar protocol this specification integrates with, and the design teams of Discord, Microsoft Teams, and Slack for prior art in chat-calendar integration that informed this work.
