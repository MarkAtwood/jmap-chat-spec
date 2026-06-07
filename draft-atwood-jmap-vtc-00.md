---
title: JMAP for Video/Voice Teleconferencing
abbrev: JMAP VTC
docname: draft-atwood-jmap-vtc-00
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
  RFC8030:
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

informative:
  RFC3261:
  RFC8291:
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-PUSH:
    title: JMAP Chat Push Notifications
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-push-00
    date: 2026
  JMAP-CHAT-WSS:
    title: JMAP Chat over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-CHAT-FED:
    title: JMAP Chat Federation
    target: https://datatracker.ietf.org/doc/draft-atwood-jmap-chat-federation/
  JMAP-VTC-WSS:
    title: JMAP VTC over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-wss-00
    date: 2026
  JMAP-CALENDARS:
    title: JMAP for Calendars
    target: https://datatracker.ietf.org/doc/draft-ietf-jmap-calendars/
  JMAP-SCENE:
    title: JMAP Scene
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026

--- abstract

This document defines JMAP VTC, a standalone JMAP capability ({{RFC8620}}) for managing video and voice teleconferencing sessions. It defines the `urn:ietf:params:jmap:vtc` capability; five data types (VTCCall, VTCParticipant, VTCRecording, VTCLivestream, and VTCMediaState); standard JMAP methods for managing calls and participants; push notification payloads for incoming-call ring events; and ephemeral WebSocket events for call state changes.

The specification models call *state* — who is in a call, what state the call is in, and how to join it — without prescribing the media transport. Actual voice and video media travel through a deployment-chosen stack (WebRTC, SIP/RTP, or any other media framework); the JMAP server is a call-state database, not a media server.

The capability is standalone (`urn:ietf:params:jmap:vtc`). When JMAP Chat ({{JMAP-CHAT}}) is also present, VTCCall objects MAY carry optional back-references to Chat and Space objects. When JMAP Calendars ({{JMAP-CALENDARS}}) is also present, VTCCall objects MAY bind to CalendarEvent objects for scheduled meetings.

--- middle

# Introduction

Video and voice calling is a core function of modern communication systems, yet the signaling, state management, and notification layers are typically tightly coupled to a specific media stack or platform. SIP {{RFC3261}} defines a comprehensive session initiation protocol but conflates signaling, registration, routing, and presence into a single system. Production calling products (Jitsi Meet, Zoom, Google Meet, Slack Huddles, Discord Voice Channels, FaceTime) each implement their own call-state model, and none expose it as a generic, standards-based API.

This document defines a JMAP capability that models call state as JMAP data types with standard get/set/changes/query methods. The server tracks who created a call, who is in it, what media types are active, and how to join — but the server does not participate in media negotiation or transport. The `joinUri` field on every VTCCall points to the deployment's media-layer entry point; JMAP VTC has no opinion on what lives behind that URI.

## Design Philosophy

- **Call state, not call media.** The JMAP server is a call-state database. It knows a call exists, who is in it, and whether someone is recording. It does not negotiate codecs, route RTP, or manage ICE candidates. Media signaling is the media stack's job.
- **Three call models, one object.** Ring calls (caller rings, callee answers), room calls (persistent or ephemeral drop-in), and scheduled calls (bound to a calendar event) are three state-machine paths through the same VTCCall type.
- **Ring notification is first-class.** Getting a reliable incoming-call notification to a mobile device within two seconds, with the correct OS-level call UI (CallKit on iOS, ConnectionService on Android), is the hardest and most valuable part of the spec. The push integration is designed around this constraint.
- **Standalone with optional bindings.** The capability stands alone. Chat, calendar, and federation integrations are optional foreign keys, not dependencies.

## Relationship to SIP

SIP {{RFC3261}} defines session initiation, registration, routing, forking, and presence in a single protocol. JMAP VTC takes only the session state model: a call has a lifecycle (creating → ringing → active → ended), participants join and leave, and the initiator can cancel. The remaining SIP functions map to other JMAP capabilities or deployment infrastructure:

- **Registration** → JMAP Session ({{RFC8620}}).
- **Presence** → PresenceStatus in {{JMAP-CHAT}} (optional).
- **Routing/proxy/redirect** → deployment infrastructure.
- **Forking** (ring multiple devices) → push delivery to all registered endpoints.
- **Media negotiation** (SDP offer/answer) → out-of-band, media stack's responsibility.

Protocol-specific signaling beyond the core lifecycle — call transfer (SIP REFER), DTMF, hold/resume, codec renegotiation, and any future SIP or ITU-T verb — is carried as opaque VTCGatewaySignal events (see {{gateway-signal}}). JMAP VTC provides the pass-through mechanism without defining or constraining individual signals, ensuring that external protocol evolution does not require amendments to this specification.

## Relationship to JMAP Chat (recommended) {#chat-delegation}

Deployments SHOULD advertise `urn:ietf:params:jmap:chat` ({{JMAP-CHAT}}) alongside `urn:ietf:params:jmap:vtc`. When both are present, VTCCall objects carry `chatId`, `spaceId`, and `channelId` back-references, and several collaboration features are delegated to JMAP Chat rather than re-defined in this specification:

- **In-call text chat** is the Chat bound via `chatId`. When `chatId` is supplied, the call is bound to that Chat for in-call text messaging; when omitted, the call has no text backchannel. Messages sent to the bound Chat appear as in-call chat.
- **In-call reactions** (emoji floats) are Reaction objects on Messages in the bound Chat, or ephemeral events on the Chat's WebSocket connection ({{JMAP-CHAT-WSS}}).
- **Live captions and transcription** are delivered as ephemeral events on the bound Chat's WebSocket connection, with `senderId` providing speaker attribution.
- **Lobby communication** uses the bound Chat: lobby participants can message moderators before being admitted.
- **Scheduled-call invitees** are derived from the Space membership (for Space-bound calls) or the CalendarEvent invitees (for calendar-bound calls).
- **Access control** is inherited from the Space/channel permission model (see {{access-control}}).
- **Active call banner** is signaled via the `activeCallId` field on Chat (see {{JMAP-CHAT}}).

The existing `urn:jmap:chat:cap:vtc` Endpoint and MessageAction type URIs defined in {{JMAP-CHAT}} provide the integration point in the other direction: a chat message can carry a VTC action whose `uri` points to the call's `joinUri`.

Without JMAP Chat, the VTC capability is fully functional as a standalone call-state manager. Calls have no chat binding, and the delegated features above are unavailable.

## Relationship to JMAP Scene (optional)

When `urn:ietf:params:jmap:scene` ({{JMAP-SCENE}}) is also advertised, SceneRegion objects MAY carry an `activeCallId` field referencing a VTCCall. This binding enables spatial audio: the Scene simulation layer can use SceneAvatar positions to spatialize VTC audio for participants in the region. The VTC capability itself is unaware of Scene; the binding is defined entirely in {{JMAP-SCENE}} and is optional. Without JMAP Scene, calls have no spatial component.

## Relationship to JMAP Calendars (optional)

When `urn:ietf:params:jmap:calendars` ({{JMAP-CALENDARS}}) is also advertised, VTCCall objects MAY carry a `calendarEventId` to bind a scheduled call to a CalendarEvent. The call's `joinUri` becomes active at the event's start time. Without JMAP Calendars, scheduled calls use an explicit `scheduledStartAt` field instead of a calendar binding.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout. The term "userId" refers to the authenticated user identity string provided by the transport layer, equivalent to `ChatContact.id` when {{JMAP-CHAT}} is present.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:vtc` capability in the JMAP Session object.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:vtc"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:vtc"]` is a JSON object with the following fields:

`mayCreateCall` (Boolean):
: `true` if the authenticated user may create new VTCCall objects.

`supportsRingCalls` (Boolean):
: `true` if the server supports the ring-and-answer call model (`callType: "ring"`).

`supportsRoomCalls` (Boolean):
: `true` if the server supports the drop-in room call model (`callType: "room"`).

`supportsScheduledCalls` (Boolean):
: `true` if the server supports scheduled calls (`callType: "scheduled"`).

`supportsRecording` (Boolean):
: `true` if the server supports call recording metadata via VTCRecording.

`supportsLivestream` (Boolean):
: `true` if the server supports livestream metadata via VTCLivestream.

`supportsLobby` (Boolean):
: `true` if the server supports lobby/waiting-room mode on calls.

`supportsBreakoutRooms` (Boolean):
: `true` if the server supports breakout rooms (parent/child VTCCall relationships).

`supportedMediaTypes` (String[]):
: Media types the deployment supports. Values: `"audio"`, `"video"`, `"screen"`. Servers MUST include at least `"audio"`.

`maxParticipantsPerCall` (UnsignedInt|null):
: Maximum participants per call. `null` means no server-imposed limit.

`maxConcurrentCalls` (UnsignedInt|null):
: Maximum concurrent active calls per account. `null` means no limit.

`maxBreakoutRooms` (UnsignedInt|null):
: Maximum breakout rooms per call. `null` means no limit. Absent when `supportsBreakoutRooms` is `false`.

`gateways` (VTCGateway[], optional):
: Protocol gateways available for this deployment. See {{vtc-gateway}}.

# Data Types

Data types are defined in dependency order: embedded sub-types precede the types that reference them.

## VTCGateway {#vtc-gateway}

A VTCGateway advertises a protocol gateway through which external telephony or video-conferencing systems can interoperate with JMAP VTC calls. VTCGateways are advertised in the account-level capability object and are not JMAP objects with their own methods.

JMAP VTC defines the call lifecycle (create, join, leave, end) and participant state (media, role, lobby). Protocol-specific signaling beyond this lifecycle — SIP REFER, DTMF, hold/resume, codec renegotiation, H.245 commands, or any future verb added by an external specification — passes through as opaque data via VTCGatewaySignal events (see {{gateway-signal}}). This design ensures that changes to PSTN, SIP, or ITU-T specifications never require amendments to this document.

`protocol` (String):
: A reverse-domain identifier for the gateway protocol. This specification registers `"pstn"`, `"sip"`, and `"h323"` as short aliases for the three common telephony protocols. Vendor-specific or proprietary protocols SHOULD use reverse-domain notation (e.g., `"com.apple.facetime"`, `"org.signal"`, `"com.zoom.zoomphone"`). Clients MUST ignore unrecognized values.

`displayName` (String, optional):
: Human-readable label for this gateway (e.g., `"US PSTN"`, `"Corporate SIP Trunk"`, `"H.323 MCU"`).

`supportsDialOut` (Boolean):
: `true` if a moderator may initiate an outbound call through this gateway to bridge an external party into a VTCCall. See {{dial-out}}.

`supportedMediaTypes` (String[]):
: Media types this gateway supports. Subset of the account-level `supportedMediaTypes`. A PSTN gateway typically supports only `["audio"]`; a SIP or H.323 gateway may support `["audio", "video"]`.

`dialInNumbers` (VTCDialInEntry[], optional):
: For gateways that support inbound calls (PSTN, SIP): addresses external callers can use to reach a call. See {{dial-in-entry}}.

`metadata` (Object, optional):
: Protocol-specific gateway configuration. Clients MUST ignore unknown keys. Examples:
  - PSTN: `{}`
  - SIP: `{"registrar": "sip.example.com", "transport": "tls"}`
  - H.323: `{"gatekeeper": "h323gk.example.com"}`

### VTCDialInEntry {#dial-in-entry}

A VTCDialInEntry is an address through which an external caller can reach a VTCCall via a gateway.

`address` (String):
: The dial-in address. Format depends on the parent gateway's protocol:
  - PSTN: E.164 phone number (e.g., `"+14155551234"`)
  - SIP: SIP URI (e.g., `"sip:meet@example.com"`)
  - H.323: H.323 alias or E.164 number

`region` (String, optional):
: A human-readable region label (e.g., `"US West"`, `"EU"`, `"Asia-Pacific"`).

`tollFree` (Boolean):
: `true` if this is a toll-free number. Applicable to PSTN gateways; `false` for other protocols.

## VTCMediaState {#media-state}

VTCMediaState is an embedded object describing a participant's self-reported media state. The server relays this information without verification; it has no media-layer visibility. See {{media-state-accuracy}} for security implications.

`audio` (Boolean):
: `true` if the participant's microphone is active.

`video` (Boolean):
: `true` if the participant's camera is active.

`screen` (Boolean):
: `true` if the participant is sharing their screen.

`raisedHand` (Boolean):
: `true` if the participant has raised their hand.

## VTCCallPolicy {#vtc-call-policy}

VTCCallPolicy is an embedded object on VTCCall containing moderator-controlled media policies for the call. Non-moderators MUST receive `forbidden` when attempting to update these fields via `VTCCall/set`.

`muteOnEntry` (Boolean):
: When `true`, participants join with `mediaState.audio` set to `false`. Default: `false`.

`videoOffOnEntry` (Boolean):
: When `true`, participants join with `mediaState.video` set to `false`. Default: `false`.

`participantsCanUnmute` (Boolean):
: When `false`, non-moderator participants MUST receive `forbidden` when attempting to set their own `mediaState.audio` to `true` via `VTCParticipant/set`. The moderator must use the ask-to-unmute signal (see {{ask-to-unmute}}) or unmute them directly. Default: `true`.

`participantsCanShareScreen` (Boolean):
: When `false`, non-moderator participants MUST receive `forbidden` when attempting to set their own `mediaState.screen` to `true`. Default: `true`.

`participantsCanStartVideo` (Boolean):
: When `false`, non-moderator participants MUST receive `forbidden` when attempting to set their own `mediaState.video` to `true`. Default: `true`.

## VTCParticipant {#vtc-participant}

A VTCParticipant describes one participant in a VTCCall. It is a top-level JMAP data type with its own get/set/changes/query methods.

### VTCParticipant ID Assignment

VTCParticipant IDs are opaque server-assigned identifiers. For authenticated users, the server SHOULD use the userId as the id within a given call; for unauthenticated participants (PSTN callers, anonymous guests), the server assigns a unique id.

### VTCParticipant Object Fields

`id` (String, immutable, server-set):
: Opaque server-assigned identifier.

`callId` (String, immutable):
: The id of the VTCCall this participant belongs to.

`userId` (String|null, immutable, server-set):
: The authenticated user identity string. `null` for unauthenticated participants (PSTN callers, anonymous guests).

`displayName` (String):
: Display name for this participant. For authenticated users, derived from the user profile or ChatContact record when {{JMAP-CHAT}} is present. For PSTN callers, derived from caller ID. For anonymous guests, a server-assigned or self-reported name. The server sets `displayName` at join time from the user profile or `ChatContact` record (or `SpaceMember.nick` when applicable). If the source display name changes during the call, the server is not required to update `displayName` on existing VTCParticipant records. Clients displaying call participants alongside Chat contacts SHOULD prefer the Chat-side display name for consistency.

`role` (String):
: `"moderator"` or `"participant"`. Moderators may perform moderation actions (see {{moderator-actions}}). The call initiator receives `"moderator"` by default; the server MAY grant moderator status to additional participants based on deployment policy. The VTC role (`"moderator"` / `"participant"`) is independent of any Chat or Space role. Deployments MAY choose to auto-promote Space administrators or Chat owners to `"moderator"` at join time, but this is not required by the protocol.

`joinedAt` (UTCDate|null, server-set):
: Time this participant joined the call. `null` if the participant has been invited but has not yet joined (e.g., in a ringing call).

`leftAt` (UTCDate|null):
: Time this participant left the call. `null` if currently in the call.

`callResponse` (String|null):
: For ring-call targets: the participant's response to the ring. Values: `"pending"` (not yet responded), `"accepted"`, `"declined"`. `null` for participants who were not ring targets (room/scheduled call joins, dial-out participants). The server sets this to `"pending"` when creating VTCParticipant records for ring targets, to `"accepted"` when the participant answers, and clients set it to `"declined"` to decline.

`joinMethod` (String):
: How this participant connected. This specification registers `"webrtc"`, `"sip"`, `"pstn"`, and `"h323"` as short aliases. When the participant arrived via a gateway, this value corresponds to the VTCGateway `protocol`. Vendor-specific connection methods SHOULD use reverse-domain notation (e.g., `"com.apple.facetime"`). Clients MUST ignore unrecognized values.

`gatewayData` (Object|null):
: Protocol-specific state for participants who joined via a gateway (`joinMethod` other than `"webrtc"`). Opaque to JMAP VTC; the server stores and relays this object without interpretation. Clients that understand the protocol MAY inspect it; all others MUST ignore it. `null` for WebRTC participants and when no gateway-specific data is available. Examples:
  - PSTN: `{"callerIdNumber": "+14155559876", "callerIdName": "Alice Smith"}`
  - SIP: `{"sipUri": "sip:alice@example.com", "userAgent": "Opal/4.0"}`
  - H.323: `{"alias": "alice", "endpointType": "terminal"}`

`lobbyState` (String|null):
: Present only when the call has `lobbyEnabled: true`. Values: `"waiting"`, `"admitted"`, `"rejected"`. `null` when lobby is not active or the participant bypassed the lobby (moderators).

`mediaState` (VTCMediaState):
: The participant's current self-reported media state.

`speakerTimeMs` (UnsignedInt, server-set):
: Cumulative milliseconds of talk time for this participant, tracked by the server or media layer. Servers that cannot track speaker time MUST set this to `0`.

`kickedBy` (String|null, server-set):
: The userId of the moderator who removed this participant from the call. `null` if the participant left voluntarily or is still in the call.

`e2eeFingerprint` (String|null):
: When the call has `e2eeEnabled: true`: the participant's public key fingerprint for E2EE verification, as reported by the media layer. Clients SHOULD display this to enable out-of-band verification (e.g., reading the fingerprint aloud). The format is media-layer-defined. `null` when E2EE is not active or the participant has not yet completed key exchange.

`unmuteRequested` (Boolean):
: `true` when a moderator has requested that this participant unmute. Cleared to `false` when the participant unmutes or explicitly dismisses the request. Default: `false`.

## VTCCall {#vtc-call}

A VTCCall represents a voice or video call session. It is the primary JMAP data type defined by this specification.

### VTCCall ID Assignment

VTCCall IDs are ULIDs {{ULID}} assigned by the server at the time the call is created.

### VTCCall Object Fields

`id` (String, immutable, server-set):
: A ULID assigned at creation.

`accountId` (String, server-set):
: The account that owns this VTCCall record.

`callType` (String, immutable):
: `"ring"`, `"room"`, or `"scheduled"`. Determines the state machine path (see {{state-machine}}).

`state` (String, server-set):
: Current call state. See {{state-machine}} for valid values and transitions.

`createdAt` (UTCDate, immutable, server-set):
: Time the call was created.

`endedAt` (UTCDate|null, server-set):
: Time the call ended. `null` while the call is active or pending.

`endReason` (String|null, server-set):
: Reason the call ended. Values: `"completed"`, `"missed"`, `"declined"`, `"cancelled"`, `"failed"`, `"timeout"`. `null` while the call has not ended.

`initiatorId` (String, immutable, server-set):
: The userId of the participant who created the call.

`subject` (String|null):
: Optional human-readable call title (e.g., `"Standup"`, `"1:1 with Alice"`).

`joinUri` (String):
: The media-layer entry point for this call. Opaque to the JMAP server. May be a WebRTC room URL, a Jitsi Meet room URL, a SIP URI, or any other media endpoint. Peer-supplied; MUST be treated as untrusted. Clients MUST NOT connect to this URI without explicit user initiation.

`joinPassword` (String|null):
: Optional password or PIN required to join the call at the media layer.

`mediaTypes` (String[]):
: Media types active for this call. Subset of `supportedMediaTypes`. Example: `["audio", "video"]`.

`activeParticipantCount` (UnsignedInt, server-set):
: Number of participants currently in the call (joined and not yet left). Clients use `VTCParticipant/query` with a `callId` filter to retrieve the full participant list.

`lobbyEnabled` (Boolean):
: When `true`, new participants enter a waiting room and must be admitted by a moderator before joining the call. Default: `false`.

`policy` (VTCCallPolicy):
: Call-level media policies. See {{vtc-call-policy}}. Mutable by moderators only.

`parentCallId` (String|null, immutable):
: For breakout rooms: the id of the parent VTCCall. `null` for top-level calls.

`breakoutRoomIds` (String[], server-set):
: For parent calls: ids of child breakout-room VTCCalls. Empty for non-parent calls and when breakout rooms are not in use.

`scheduledStartAt` (UTCDate|null):
: For scheduled calls: the intended start time. Clients SHOULD display this to invitees. The server transitions the call from `"pending"` to `"active"` when a participant joins at or after this time; the server does not auto-transition based on the clock alone (a scheduled call with no participants stays `"pending"`).

`gatewayPin` (String|null):
: PIN or access code for joining this call via a protocol gateway. Present when a gateway (PSTN, SIP, H.323) requires a PIN to associate an inbound connection with this call. `null` when no gateway requires a PIN or no gateways are configured.

`e2eeEnabled` (Boolean):
: `true` when end-to-end encryption is active for this call's media streams. The mechanism (WebRTC Insertable Streams, SFrame, or other) is media-layer-defined; this field is the state signal for client UI (e.g., displaying a lock icon). Default: `false`. When `true`, features that require server-side media access (recording, livestreaming, gateway participants) are typically unavailable; the server SHOULD reject VTCRecording and VTCLivestream creates with `forbidden` when `e2eeEnabled` is `true`.

### Optional Binding Fields

The following fields are present only when the corresponding companion capability is advertised on the same account:

`chatId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Chat ({{JMAP-CHAT}}) associated with this call. `null` if the call has no chat binding. Same-account only.

`spaceId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Space ({{JMAP-CHAT}}) associated with this call. `null` if the call has no Space context.

`channelId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a channel Chat within the Space. `null` if the call is Space-wide or has no channel context.

When a Space or channel Chat referenced by `spaceId` or `channelId` is destroyed, the server MUST clear the corresponding field to `null` on all VTCCalls that reference it. This is a side effect of the Space or Chat destruction and MUST emit a `StateChange` for the VTCCall type.

`calendarEventId` (String|null):
: When `urn:ietf:params:jmap:calendars` is present: the id of a CalendarEvent ({{JMAP-CALENDARS}}) bound to this call. Same-account only. `null` if the call has no calendar binding.

### State Machine {#state-machine}

The VTCCall `state` field follows one of three paths depending on `callType`. State transitions are server-enforced; clients request transitions via `VTCCall/set` and the server validates them.

#### Ring Calls (`callType: "ring"`)

Valid states: `"creating"`, `"ringing"`, `"active"`, `"ended"`.

- `"creating"` → `"ringing"`: Server transitions automatically when the call is created and ring notifications have been dispatched.
- `"ringing"` → `"active"`: A target participant answers (see {{answering}}).
- `"ringing"` → `"ended"`: Timeout, all targets decline, or initiator cancels.
- `"active"` → `"ended"`: All participants leave or a moderator ends the call.

#### Room Calls (`callType: "room"`)

Valid states: `"active"`, `"ended"`.

- Room calls transition directly to `"active"` on creation. They are immediately joinable.
- `"active"` → `"ended"`: A moderator explicitly closes the room, or a deployment-defined inactivity timeout fires.

#### Scheduled Calls (`callType: "scheduled"`)

Valid states: `"pending"`, `"active"`, `"ended"`.

- `"pending"`: The call has been created but the scheduled start time has not arrived or no participant has joined yet.
- `"pending"` → `"active"`: A participant joins at or after `scheduledStartAt`.
- `"active"` → `"ended"`: All participants leave or a moderator ends the call.

For all call types, `"ended"` is a terminal state. The server MUST NOT transition a call out of `"ended"`.

## VTCRecording {#vtc-recording}

A VTCRecording tracks the metadata of a recording session within a VTCCall. It is a top-level JMAP data type.

`id` (String, immutable, server-set):
: Opaque server-assigned identifier.

`callId` (String, immutable):
: The VTCCall this recording belongs to.

`state` (String):
: `"recording"`, `"paused"`, `"stopped"`, `"processing"`, `"available"`, or `"failed"`.

`startedAt` (UTCDate, server-set):
: Time recording started.

`stoppedAt` (UTCDate|null, server-set):
: Time recording stopped. `null` while recording is active or paused.

`initiatedBy` (String):
: The userId of the participant who started the recording.

`blobId` (String|null, server-set):
: JMAP blob reference for the recording file. Available only when `state` is `"available"`.

`size` (UnsignedInt|null, server-set):
: Recording file size in bytes. Available only when `state` is `"available"`.

`duration` (UnsignedInt|null, server-set):
: Recording duration in seconds. Available only when `state` is `"available"` or `"stopped"`.

`mediaType` (String|null, server-set):
: MIME type of the recording file (e.g., `"video/webm"`, `"video/mp4"`, `"audio/ogg"`). Available only when `state` is `"available"`.

## VTCLivestream {#vtc-livestream}

A VTCLivestream tracks the metadata of an outbound livestream from a VTCCall to an external platform. It is a top-level JMAP data type.

`id` (String, immutable, server-set):
: Opaque server-assigned identifier.

`callId` (String, immutable):
: The VTCCall this livestream belongs to.

`state` (String):
: `"starting"`, `"live"`, `"stopped"`, or `"failed"`.

`platform` (String|null):
: Target platform identifier (e.g., `"youtube"`, `"twitch"`). Deployment-defined; clients MUST ignore unrecognized values.

`streamUri` (String):
: The RTMP or equivalent ingest endpoint. Treated as sensitive; see {{livestream-key-exposure}}.

`streamKey` (String):
: The stream key or authentication credential for the ingest endpoint. This field is write-only for non-moderators: the server MUST return an empty string in `VTCLivestream/get` responses for participants whose role is not `"moderator"`. See {{livestream-key-exposure}}.

`startedAt` (UTCDate|null, server-set):
: Time the stream went live. `null` before the stream is live.

`stoppedAt` (UTCDate|null, server-set):
: Time the stream stopped. `null` while the stream is active.

# Methods

## VTCCall Methods

### VTCCall/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

Example response (properties abridged):

~~~json
{
  "id": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "callType": "ring",
  "state": "active",
  "createdAt": "2026-06-05T14:30:00Z",
  "endedAt": null,
  "endReason": null,
  "initiatorId": "user:bob@example.com",
  "subject": "Quick sync",
  "joinUri": "https://meet.example.com/room/xkz",
  "joinPassword": null,
  "mediaTypes": ["audio", "video"],
  "activeParticipantCount": 2,
  "lobbyEnabled": false,
  "policy": {
    "muteOnEntry": false,
    "videoOffOnEntry": false,
    "participantsCanUnmute": true,
    "participantsCanShareScreen": true,
    "participantsCanStartVideo": true
  },
  "parentCallId": null,
  "breakoutRoomIds": [],
  "scheduledStartAt": null,
  "gatewayPin": null,
  "e2eeEnabled": false,
  "chatId": "01J4XL2RN5NXWS9QQCFIUJ4BC",
  "spaceId": null,
  "channelId": null,
  "calendarEventId": null
}
~~~

### VTCCall/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCCall/set {#vtc-call-set}

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

VTCCall/set manages call lifecycle: creating calls, ending calls, and updating call-level settings. Participant management (joining, leaving, muting, kicking) is handled by `VTCParticipant/set` (see {{vtc-participant-methods}}).

When creating a VTCCall with a `chatId` that is already referenced by another VTCCall in `"active"` or `"ringing"` state, the server MUST return `invalidArguments` with description "A call is already active on this Chat." This ensures that `Chat.activeCallId` is unambiguous and at most one call is active per Chat at any time.

#### Creating a Ring Call

`create` with `callType: "ring"` accepts:

`targetParticipantIds` (String[], required):
: The userIds of the participants to ring. The server creates a VTCParticipant record for each target (with `joinedAt: null`) and for the initiator, dispatches ring notifications (see {{ring-notification}}) to all targets, and transitions the call to `"ringing"`.

`mediaTypes` (String[], required):
: The media types for this call (e.g., `["audio", "video"]`).

Optional: `subject` (String), `chatId` (String), `spaceId` (String), `channelId` (String).

When `chatId` is supplied, the server SHOULD verify that each `targetParticipantId` corresponds to a member of the bound Chat. The server MAY reject targets that are not Chat members by omitting them from the created VTCParticipant records, or MAY create the records regardless and rely on the join-time membership check.

The server sets `id`, `initiatorId`, `createdAt`, `state`, and `joinUri`. The server generates `joinUri` pointing to the deployment's media stack.

Example create:

~~~json
{
  "create": {
    "c0": {
      "callType": "ring",
      "targetParticipantIds": ["user:alice@example.com"],
      "mediaTypes": ["audio", "video"],
      "subject": "Quick sync"
    }
  }
}
~~~

The server creates VTCParticipant records for each target and for the initiator, dispatches `VTCCallPush` ({{vtc-call-push}}) and `VTCRingEvent` ({{ring-event}}) notifications to all targets, and returns:

~~~json
{
  "created": {
    "c0": {
      "id": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "state": "ringing",
      "initiatorId": "user:bob@example.com",
      "createdAt": "2026-06-05T14:30:00Z",
      "joinUri": "https://meet.example.com/room/xkz",
      "activeParticipantCount": 0
    }
  }
}
~~~

Servers MUST return `invalidArguments` when `targetParticipantIds` is empty. Servers MUST return `invalidArguments` when `mediaTypes` contains values not present in the account-level `supportedMediaTypes`. Servers MUST return `forbidden` when the account's `mayCreateCall` is `false`. Servers MUST return `forbidden` when `supportsRingCalls` is `false`. Servers SHOULD silently drop targets that correspond to blocked contacts ({{ring-notification}}) and proceed with the remaining targets; if all targets are blocked, the server MUST return `forbidden`. When `chatId` is supplied but `urn:ietf:params:jmap:chat` is not present in the server's capabilities, the server MUST return `invalidArguments` with a description indicating the Chat capability is required. When `chatId` is supplied but does not resolve to a Chat object the caller has access to, the server MUST return `invalidArguments`. The caller MUST be a member of the Chat referenced by `chatId`. The server MUST return `forbidden` if the caller is not a member. When the call carries a `spaceId` or `channelId` binding and {{JMAP-CHAT}} is present, the server MUST verify the caller has the `"start_call"` permission in the bound Space or channel; if not, the server MUST return `forbidden`.

#### Creating a Room Call

`create` with `callType: "room"` accepts:

`mediaTypes` (String[], required):
: The media types for this call.

Optional: `subject` (String), `lobbyEnabled` (Boolean), `joinPassword` (String), `chatId` (String), `spaceId` (String), `channelId` (String).

The server sets `id`, `initiatorId`, `createdAt`, `state` (to `"active"`), and `joinUri`.

Example create:

~~~json
{
  "create": {
    "c0": {
      "callType": "room",
      "mediaTypes": ["audio"],
      "subject": "Team Huddle",
      "spaceId": "01J3ABCDEF0000000000000000",
      "channelId": "01J3ABCDEF1111111111111111"
    }
  }
}
~~~

Servers MUST return `forbidden` when `mayCreateCall` is `false` or `supportsRoomCalls` is `false`. When the call carries a `spaceId` or `channelId` binding and {{JMAP-CHAT}} is present, the server MUST verify the caller has the `"start_call"` permission in the target Space or channel; if not, the server MUST return `forbidden`. When `chatId` is supplied but `urn:ietf:params:jmap:chat` is not present in the server's capabilities, the server MUST return `invalidArguments` with a description indicating the Chat capability is required. When `chatId` is supplied but does not resolve to a Chat object the caller has access to, the server MUST return `invalidArguments`. The caller MUST be a member of the Chat referenced by `chatId`. The server MUST return `forbidden` if the caller is not a member.

#### Creating a Scheduled Call

`create` with `callType: "scheduled"` accepts:

`scheduledStartAt` (UTCDate, required):
: The intended start time. MUST be in the future; servers MUST reject past values with `invalidArguments`.

`mediaTypes` (String[], required):
: The media types for this call.

Optional: `subject` (String), `lobbyEnabled` (Boolean), `joinPassword` (String), `chatId` (String), `spaceId` (String), `channelId` (String), `calendarEventId` (String).

The server sets `id`, `initiatorId`, `createdAt`, `state` (to `"pending"`), and `joinUri`.

Example create:

~~~json
{
  "create": {
    "c0": {
      "callType": "scheduled",
      "mediaTypes": ["audio", "video"],
      "scheduledStartAt": "2026-06-06T10:00:00Z",
      "subject": "Sprint Planning",
      "lobbyEnabled": true,
      "spaceId": "01J3ABCDEF0000000000000000"
    }
  }
}
~~~

Servers MUST return `invalidArguments` when `scheduledStartAt` is in the past. Servers MUST return `forbidden` when `mayCreateCall` is `false` or `supportsScheduledCalls` is `false`. When `chatId` is supplied but `urn:ietf:params:jmap:chat` is not present in the server's capabilities, the server MUST return `invalidArguments` with a description indicating the Chat capability is required. When `chatId` is supplied but does not resolve to a Chat object the caller has access to, the server MUST return `invalidArguments`. The caller MUST be a member of the Chat referenced by `chatId`. The server MUST return `forbidden` if the caller is not a member.

#### Ending a Call

A moderator calls `VTCCall/set` to transition `state` to `"ended"`. The server sets `endedAt` and `endReason: "completed"` and dispatches state-change notifications to all participants.

Example update:

~~~json
{
  "update": {
    "01J4XKZQN4MWVT8PPBEHTJ3AB": {
      "state": "ended"
    }
  }
}
~~~

The server MUST return `forbidden` if the caller is not a moderator on this call. The server MUST return `invalidArguments` if the call is already in state `"ended"`. The server MUST return `invalidArguments` for any `state` transition not permitted by the state machine ({{state-machine}}).

#### Moderator Actions on VTCCall {#moderator-actions}

Participants with `role: "moderator"` MAY use `VTCCall/set` to perform the following call-level actions. Non-moderators MUST receive `forbidden` for these operations. Participant-level moderator actions (mute, kick, admit, role changes) are performed via `VTCParticipant/set` (see {{vtc-participant-methods}}).

**Create a breakout room:**
: Create a new VTCCall with `parentCallId` set to the current call's id. The server adds the new call's id to the parent's `breakoutRoomIds`.

**Close a breakout room:**
: Transition a child VTCCall to `"ended"`. The server removes its id from the parent's `breakoutRoomIds`.

**Toggle lobby:**
: Patch `lobbyEnabled` between `true` and `false`.

### VTCCall/query {#vtc-call-query}

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`state` (String, optional):
: Filter by call state. One of `"creating"`, `"ringing"`, `"active"`, `"pending"`, or `"ended"`.

`callType` (String, optional):
: Filter by call type. One of `"ring"`, `"room"`, or `"scheduled"`.

`initiatorId` (String, optional):
: Filter to calls initiated by this userId.

`chatId` (String, optional):
: Filter to calls associated with this Chat. Only valid when `urn:ietf:params:jmap:chat` is present; servers MUST return `unsupportedFilter` otherwise.

`spaceId` (String, optional):
: Filter to calls associated with this Space. Only valid when `urn:ietf:params:jmap:chat` is present; servers MUST return `unsupportedFilter` otherwise.

`hasParticipant` (String, optional):
: Filter to calls in which this userId is or was a participant.

`createdAfter` (UTCDate, optional):
: Calls created at or after this time.

`createdBefore` (UTCDate, optional):
: Calls created before this time.

`endedAfter` (UTCDate, optional):
: Calls ended at or after this time.

`endedBefore` (UTCDate, optional):
: Calls ended before this time.

All filter properties are combined with logical AND. The query MUST only return calls the authenticated user has access to (see {{access-control}}).

Sort properties: `createdAt`, `endedAt`. Default sort: `createdAt` descending.

### VTCCall/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## VTCParticipant Methods {#vtc-participant-methods}

### VTCParticipant/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### VTCParticipant/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCParticipant/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

#### Answering a Ring Call {#answering}

For ring calls, the server creates VTCParticipant records for all targets (with `joinedAt: null`) when the call is created. A target participant answers by calling `VTCParticipant/set` with an `update` on their existing record, setting `joinMethod` to their connection type. The server sets `joinedAt` to the current time.

The server MUST validate that:

1. The call is in state `"ringing"`.
2. The caller's userId matches the VTCParticipant record's userId.

On success, the server transitions the call to `"active"` and dispatches a `VTCCallEndEvent` with `endReason: "answered_elsewhere"` to all other ringing devices (see {{multi-device-forking}}).

Example update (target participant answers):

~~~json
{
  "update": {
    "user:alice@example.com": {
      "joinMethod": "webrtc"
    }
  }
}
~~~

The server MUST return `invalidArguments` if the call is not in state `"ringing"`. The server MUST return `forbidden` if the caller's userId does not match the VTCParticipant record's userId. The server MUST return `notFound` if the VTCParticipant record does not exist.

#### Joining a Room or Scheduled Call

A participant joins by calling `VTCParticipant/set` with a `create`:

`callId` (String, required):
: The VTCCall to join.

`joinMethod` (String, required):
: How this participant is connecting. One of the registered short aliases (`"webrtc"`, `"sip"`, `"pstn"`, `"h323"`) or a reverse-domain identifier.

The server sets `id`, `userId`, `displayName`, `role`, `joinedAt`, `mediaState` (defaults), and `speakerTimeMs` (to `0`).

For **room calls**, the server validates that the call is in state `"active"`.

For **scheduled calls**, the server validates that the call is in state `"pending"` or `"active"`. When a participant joins a `"pending"` scheduled call at or after `scheduledStartAt`, the server transitions the call to `"active"`.

When `lobbyEnabled` is `true`, the server sets the new participant's `lobbyState` to `"waiting"`. The participant does not enter the call until a moderator admits them.

Example create (joining a room call):

~~~json
{
  "create": {
    "p0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "joinMethod": "webrtc"
    }
  }
}
~~~

The server responds with the complete VTCParticipant record:

~~~json
{
  "created": {
    "p0": {
      "id": "user:alice@example.com",
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "userId": "user:alice@example.com",
      "displayName": "Alice Chen",
      "role": "participant",
      "joinedAt": "2026-06-05T14:35:22Z",
      "leftAt": null,
      "callResponse": null,
      "joinMethod": "webrtc",
      "gatewayData": null,
      "lobbyState": null,
      "mediaState": {
        "audio": true,
        "video": true,
        "screen": false,
        "raisedHand": false
      },
      "speakerTimeMs": 0,
      "kickedBy": null,
      "e2eeFingerprint": null,
      "unmuteRequested": false
    }
  }
}
~~~

The server MUST return `notFound` if the `callId` does not exist or the caller does not have access to it. The server MUST return `invalidArguments` if the call is in state `"ended"`. The server MUST return `forbidden` when the call carries a Space binding and the caller does not have `"view"` permission on the target channel. When the VTCCall carries a `chatId` binding and `urn:ietf:params:jmap:chat` is present, the server MUST verify the caller is a member of the bound Chat. The server MUST return `forbidden` if the caller is not a member. If the call's `activeParticipantCount` has reached `maxParticipantsPerCall`, the server MUST return `overQuota`. If the authenticated user already has `maxConcurrentCalls` active calls, the server MUST return `overQuota`.

#### Declining a Ring Call

A target participant whose VTCParticipant record was created by the ring (with `joinedAt: null`) calls `VTCParticipant/set` with an `update` setting `callResponse` to `"declined"`. The server sets `leftAt` to the current time. When all target participants have declined, the server transitions the call to `"ended"` with `endReason: "declined"`.

~~~json
{
  "update": {
    "user:alice@example.com": {
      "callResponse": "declined"
    }
  }
}
~~~

The server MUST return `forbidden` if the caller's userId does not match the VTCParticipant record's userId. The server MUST return `invalidArguments` if the call is not in state `"ringing"` or the participant's `callResponse` is not `"pending"`.

#### Updating Media State

A participant calls `VTCParticipant/set` to `update` their own `mediaState` fields:

~~~json
{
  "update": {
    "participantId123": {
      "mediaState/audio": false
    }
  }
}
~~~

A participant MUST only update their own `mediaState`; updates targeting another participant's `mediaState` are moderator actions (see below). The server MUST return `forbidden` when a non-moderator attempts to set `mediaState/audio` to `true` while `policy.participantsCanUnmute` is `false`, or `mediaState/video` to `true` while `policy.participantsCanStartVideo` is `false`, or `mediaState/screen` to `true` while `policy.participantsCanShareScreen` is `false`.

When `policy.muteOnEntry` is `true`, the server sets `mediaState.audio` to `false` on the VTCParticipant record at join time, regardless of the client's requested value. Similarly for `policy.videoOffOnEntry` and `mediaState.video`.

#### Leaving a Call

A participant calls `VTCParticipant/set` with an `update` setting their own `leftAt` to the current time. When the last active participant leaves, the server transitions the VTCCall to `"ended"` with `endReason: "completed"`.

~~~json
{
  "update": {
    "user:alice@example.com": {
      "leftAt": "2026-06-05T15:02:00Z"
    }
  }
}
~~~

The server MUST return `forbidden` if the caller's userId does not match the VTCParticipant record's userId (leaving on behalf of another participant is a kick and requires moderator role). The server MUST return `invalidArguments` if the participant has already left (`leftAt` is non-null).

#### Reconnecting {#reconnection}

When a participant who has left (has a non-null `leftAt`) re-joins the same call, the server MUST update the existing VTCParticipant record rather than creating a new one: clear `leftAt` to `null`, update `joinedAt` to the current time, and preserve `role` and `speakerTimeMs`. This ensures a single continuous participant identity across disconnections.

#### Participant-Level Moderator Actions

Participants with `role: "moderator"` on the same VTCCall MAY use `VTCParticipant/set` to update other participants' records. Non-moderators MUST receive `forbidden` for updates targeting other participants.

**Mute another participant:**
: Update the target's `mediaState/audio` to `false`. The media layer SHOULD honor this by muting the participant's audio stream; enforcement is media-layer-dependent.

**Ask to unmute:** {#ask-to-unmute}
: When `policy.participantsCanUnmute` is `false`, a moderator may request that a participant unmute by updating the target's `unmuteRequested` to `true`. The server delivers a `VTCUnmuteRequestEvent` (see {{unmute-request-event}}) to the target participant's WebSocket connections. The participant may then choose to unmute (which clears `unmuteRequested`) or ignore the request. This is a soft request, not a forced unmute.

**Kick a participant:**
: Update the target's `leftAt` to the current time. The server sets `kickedBy` to the moderator's userId.

**Admit a lobby participant:**
: Update the target's `lobbyState` from `"waiting"` to `"admitted"`.

**Reject a lobby participant:**
: Update the target's `lobbyState` from `"waiting"` to `"rejected"`.

**Grant or revoke moderator role:**
: Update a participant's `role` between `"moderator"` and `"participant"`. A moderator MUST NOT revoke their own moderator role if they are the last moderator; the server MUST reject this with `forbidden`.

**Assign to a breakout room:**
: Update a participant's `callId` from the parent call to a child breakout-room call. The server sets `leftAt` on the parent-call entry and creates (or reconnects) a VTCParticipant record on the child call. The server MUST return `invalidArguments` if the target `callId` is not a child of the participant's current call.

Example — admitting a lobby participant and muting another:

~~~json
{
  "update": {
    "user:charlie@example.com": {
      "lobbyState": "admitted"
    },
    "user:dave@example.com": {
      "mediaState/audio": false
    }
  }
}
~~~

The server MUST return `invalidArguments` when a lobby-state transition is invalid (e.g., `"admitted"` → `"waiting"` or `"rejected"` → `"admitted"`). The server MUST return `invalidArguments` when a moderator attempts to update a participant who is not in the same call.

#### Gateway Dial-Out {#dial-out}

A moderator may bridge an external party into a call by calling `VTCParticipant/set` with a `create` that specifies a gateway target:

`callId` (String, required):
: The VTCCall to bridge the external party into.

`joinMethod` (String, required):
: The gateway protocol to use. MUST match a VTCGateway `protocol` whose `supportsDialOut` is `true`. One of the registered short aliases (`"pstn"`, `"sip"`, `"h323"`) or a reverse-domain identifier.

`gatewayData` (Object, required):
: Protocol-specific addressing for the outbound call. The server passes this to the gateway without interpretation. Examples:
  - PSTN: `{"dialNumber": "+14155559876"}`
  - SIP: `{"sipUri": "sip:alice@example.com"}`
  - H.323: `{"alias": "alice@mcu.example.com"}`

Optional: `displayName` (String) — a label for the external participant before they are connected.

The server creates a VTCParticipant with `joinedAt: null` and initiates the outbound call via the gateway. When the external party answers, the server sets `joinedAt`. If the outbound call fails or is not answered, the server sets `leftAt` and `endReason` information in `gatewayData`.

Example create (PSTN dial-out):

~~~json
{
  "create": {
    "g0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "joinMethod": "pstn",
      "gatewayData": {"dialNumber": "+14155559876"},
      "displayName": "Alice (mobile)"
    }
  }
}
~~~

Non-moderators MUST receive `forbidden` for dial-out creates. The server MUST return `invalidArguments` when `joinMethod` does not match any VTCGateway `protocol` whose `supportsDialOut` is `true`. The server MUST return `invalidArguments` when the call is in state `"ended"`.

### VTCParticipant/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`callId` (String, optional):
: Filter to participants in this call. Servers SHOULD require this property on every query unless combined with `userId`; a query that omits both `callId` and `userId` MAY be rejected with `unsupportedFilter`.

`userId` (String, optional):
: Filter to a specific user across calls. When used without `callId`, returns the user's participation history.

`role` (String, optional):
: Filter by role. One of `"moderator"` or `"participant"`.

`isActive` (Boolean, optional):
: When `true`, filter to participants with `joinedAt != null` and `leftAt == null`. When `false`, filter to participants who have left.

`lobbyState` (String, optional):
: Filter by lobby state. One of `"waiting"`, `"admitted"`, or `"rejected"`. Only meaningful when the call has `lobbyEnabled: true`.

`joinMethod` (String, optional):
: Filter by connection method (e.g., `"webrtc"`, `"pstn"`).

All filter properties are combined with logical AND.

Sort properties: `joinedAt`, `displayName`. Default sort: `joinedAt` ascending.

### VTCParticipant/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## VTCRecording Methods

### VTCRecording/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

All current and past participants of the associated VTCCall may retrieve VTCRecording objects. The server MUST return `notFound` for recording ids belonging to calls the authenticated user has no access to.

### VTCRecording/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCRecording/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

Only participants with `role: "moderator"` on the associated VTCCall MAY create, update, or destroy VTCRecording objects. Non-moderators MUST receive `forbidden`.

#### Starting a Recording

`create` accepts:

`callId` (String, required):
: The VTCCall to record. The call MUST be in state `"active"`.

The server sets `id`, `state` (to `"recording"`), `startedAt`, and `initiatedBy`.

Example create:

~~~json
{
  "create": {
    "r0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB"
    }
  }
}
~~~

The server MUST return `forbidden` when the caller is not a moderator on the associated call. The server MUST return `forbidden` when the call has `e2eeEnabled: true`, because recording requires server-side media access that is incompatible with end-to-end encryption. The server MUST return `forbidden` when the account-level `supportsRecording` is `false`. The server MUST return `invalidArguments` when the call is not in state `"active"`.

On success, the server MUST deliver a `VTCRecordingStateEvent` ({{vtc-recording-state-event}}) with `state: "recording"` to all participants in the call. This is a mandatory consent signal (see {{recording-consent}}).

#### Pausing, Resuming, and Stopping a Recording

`update` supports patching `state`:

- `"recording"` → `"paused"`: Pause recording. The server MUST deliver a `VTCRecordingStateEvent` with `state: "paused"`.
- `"paused"` → `"recording"`: Resume recording. The server MUST deliver a `VTCRecordingStateEvent` with `state: "recording"`.
- `"recording"` or `"paused"` → `"stopped"`: Stop recording. The server sets `stoppedAt` and delivers a `VTCRecordingStateEvent` with `state: "stopped"`.

Example update (stop recording):

~~~json
{
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "stopped"
    }
  }
}
~~~

The server MUST return `invalidArguments` for any state transition not listed above (e.g., `"stopped"` → `"recording"`, `"available"` → anything). After stopping, the server transitions the recording through `"processing"` (while the recording file is prepared) to `"available"` (when `blobId`, `size`, `duration`, and `mediaType` are set), or to `"failed"` on error. These post-stop transitions are server-initiated and not client-controllable.

#### Destroying a Recording

`destroy` removes the VTCRecording metadata. If the recording is in state `"available"`, the associated blob is subject to the server's standard blob lifecycle (it is not guaranteed to be deleted immediately). The server MUST return `forbidden` when the caller is not a moderator on the associated call.

### VTCRecording/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`callId` (String, optional):
: Filter to recordings for this call.

`state` (String, optional):
: Filter by recording state. One of `"recording"`, `"paused"`, `"stopped"`, `"processing"`, `"available"`, or `"failed"`.

`initiatedBy` (String, optional):
: Filter to recordings started by this userId.

`startedAfter` (UTCDate, optional):
: Recordings started at or after this time.

`startedBefore` (UTCDate, optional):
: Recordings started before this time.

All filter properties are combined with logical AND.

Sort properties: `startedAt`. Default sort: `startedAt` descending.

### VTCRecording/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## VTCLivestream Methods

### VTCLivestream/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

All current and past participants of the associated VTCCall may retrieve VTCLivestream objects. The server MUST return the `streamKey` field as an empty string `""` for participants whose role is not `"moderator"` (see {{livestream-key-exposure}}). The server MUST return `notFound` for livestream ids belonging to calls the authenticated user has no access to.

### VTCLivestream/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCLivestream/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

Only participants with `role: "moderator"` on the associated VTCCall MAY create, update, or destroy VTCLivestream objects. Non-moderators MUST receive `forbidden`.

#### Starting a Livestream

`create` accepts:

`callId` (String, required):
: The VTCCall to stream. The call MUST be in state `"active"`.

`streamUri` (String, required):
: The RTMP or equivalent ingest endpoint.

`streamKey` (String, required):
: The stream key or authentication credential for the ingest endpoint.

Optional: `platform` (String) — a label identifying the target platform (e.g., `"youtube"`, `"twitch"`).

Example create:

~~~json
{
  "create": {
    "s0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "streamUri": "rtmp://live.example.com/ingest",
      "streamKey": "sk_live_abc123def456",
      "platform": "youtube"
    }
  }
}
~~~

The server sets `id` and `state` (to `"starting"`), then transitions to `"live"` when the stream is active, or `"failed"` on error. The `"starting"` → `"live"` and `"starting"` → `"failed"` transitions are server-initiated.

The server MUST return `forbidden` when the caller is not a moderator on the associated call. The server MUST return `forbidden` when the call has `e2eeEnabled: true`, because livestreaming requires server-side media access that is incompatible with end-to-end encryption. The server MUST return `forbidden` when the account-level `supportsLivestream` is `false`. The server MUST return `invalidArguments` when the call is not in state `"active"`.

#### Stopping a Livestream

`update` with `state: "stopped"` ends the stream. The server sets `stoppedAt` to the current time.

~~~json
{
  "update": {
    "01J4XQ7VN0SYWV2TTFHJVW7IJ": {
      "state": "stopped"
    }
  }
}
~~~

The server MUST return `invalidArguments` if the livestream is not in state `"starting"` or `"live"`. Once stopped, the `streamUri`, `streamKey`, and `platform` fields may be updated for a future stream by creating a new VTCLivestream object.

#### Updating Stream Configuration

`update` supports patching `streamUri`, `streamKey`, or `platform` only while the livestream is in state `"stopped"` or `"failed"`. The server MUST return `invalidArguments` for configuration updates while the stream is in state `"starting"` or `"live"`.

#### Destroying a Livestream

`destroy` removes the VTCLivestream metadata. The server MUST return `forbidden` when the caller is not a moderator. The server SHOULD stop an active stream before destroying the record; the server MUST NOT leave an orphaned stream running after the metadata is removed.

### VTCLivestream/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`callId` (String, optional):
: Filter to livestreams for this call.

`state` (String, optional):
: Filter by livestream state. One of `"starting"`, `"live"`, `"stopped"`, or `"failed"`.

`platform` (String, optional):
: Filter by platform identifier.

All filter properties are combined with logical AND.

Sort properties: `startedAt`. Default sort: `startedAt` descending.

### VTCLivestream/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

# Ring Notification {#ring-notification}

Ring notification is the mechanism by which the server alerts a callee that an incoming call is waiting to be answered. This is the most latency-sensitive path in the specification.

## Push: VTCCallPush {#vtc-call-push}

When a ring call is created, the server constructs a `VTCCallPush` payload and delivers it to all registered push endpoints for each target participant. The payload is standalone and sufficient to render an incoming-call UI without a follow-up request.

`@type` (String):
: MUST be `"VTCCallPush"`.

`accountId` (String):
: The account receiving the ring.

`callId` (String):
: The id of the ringing VTCCall.

`callType` (String):
: Always `"ring"` for ring notifications.

`initiatorId` (String):
: The userId of the caller.

`initiatorDisplayName` (String, optional):
: The caller's display name at push-generation time. Snapshot; not authoritative identity.

`subject` (String|null):
: The call subject, if set.

`mediaTypes` (String[]):
: The media types for this call.

`joinUri` (String):
: The media-layer entry point.

`chatId` (String|null):
: The associated Chat id, if any.

`spaceId` (String|null):
: The associated Space id, if any.

Example `VTCCallPush` payload:

~~~json
{
  "@type": "VTCCallPush",
  "accountId": "alice-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "callType": "ring",
  "initiatorId": "user:bob@example.com",
  "initiatorDisplayName": "Bob Martinez",
  "subject": "Quick sync",
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkz",
  "chatId": null,
  "spaceId": null
}
~~~

### Urgency {#ring-urgency}

Ring-call push notifications are time-critical: a ring that arrives five seconds late is a missed call. Servers MUST use the highest available urgency for ring notifications:

- **Web Push ({{RFC8030}}):** `Urgency: high`, `TTL` SHOULD be 30–60 seconds.
- **APNs:** `apns-push-type: voip`. VoIP pushes trigger PushKit, which wakes the app and presents the CallKit incoming-call UI. Servers MUST use the VoIP push type, not the standard alert type, for ring notifications on iOS.
- **FCM:** `priority: high` with a data message. The Android app SHOULD use `ConnectionService` or `TelecomManager` to present the native incoming-call UI.

Room-call and scheduled-call notifications (e.g., "A huddle started in #general") use normal urgency and are not ring-priority. Servers SHOULD deliver these as standard `StateChange` notifications or as `ChatMessagePush` payloads ({{JMAP-CHAT-PUSH}}) rather than `VTCCallPush`.

When a room or scheduled call starts in a Chat that has push subscribers, the server SHOULD suppress the VTCCall `StateChange` for users who will receive the event via `ChatMessagePush` or Chat `StateChange` (for `activeCallId` update). Clients receiving multiple notifications for the same call-start event SHOULD deduplicate by `VTCCall.id` within a 5-second window.

### Blocked-Sender Suppression

Before delivering a `VTCCallPush`, the server MUST check whether the initiator corresponds to a contact whose `blocked` field is `true` on the target participant's contact list (when {{JMAP-CHAT}} is present) or an equivalent deployment-defined block list. If the initiator is blocked, the server MUST silently drop the ring notification. The initiator is not informed.

## WebSocket: VTCRingEvent {#ring-event}

When the target participant has an active WebSocket connection (per {{JMAP-CHAT-WSS}} or {{JMAP-VTC-WSS}}), the server SHOULD deliver a `VTCRingEvent` as an ephemeral event in addition to the push notification.

`@type` (String):
: `"VTCRingEvent"`.

`callId` (String):
: The id of the ringing VTCCall.

`initiatorId` (String):
: The userId of the caller.

`mediaTypes` (String[]):
: The media types for this call.

`joinUri` (String):
: The media-layer entry point.

The same blocked-sender suppression rule applies: the server MUST NOT deliver a `VTCRingEvent` if the initiator is blocked.

## VTCCallEndEvent {#call-end-event}

When a ringing call is answered, declined, cancelled, or times out, the server delivers a `VTCCallEndEvent` to all devices that received the ring. This tells ringing devices to stop ringing.

`@type` (String):
: `"VTCCallEndEvent"`.

`callId` (String):
: The id of the call.

`endReason` (String):
: The reason ringing stopped. Values: `"answered"`, `"answered_elsewhere"`, `"declined"`, `"cancelled"`, `"timeout"`, `"failed"`.

Example — call answered on another device:

~~~json
{
  "@type": "VTCCallEndEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "endReason": "answered_elsewhere"
}
~~~

When a ring call is answered or declined on one device, the server SHOULD deliver a silent push notification (with `urgency: "low"`) to all other devices that received the original `VTCCallPush`, carrying the `endReason` (`"answered_elsewhere"`, `"declined"`, or `"timeout"`). This allows backgrounded devices without an active WebSocket to dismiss the incoming-call UI promptly rather than waiting for the push TTL to expire.

## VTCGatewaySignal {#gateway-signal}

A VTCGatewaySignal carries a protocol-specific signal between a gateway and a call participant. It is an ephemeral WebSocket event — not persisted as a JMAP object. This is the pass-through mechanism for any external protocol verb that does not map to a core VTC lifecycle operation (join, leave, mute, kick, end).

`@type` (String):
: `"VTCGatewaySignal"`.

`callId` (String):
: The VTCCall this signal pertains to.

`participantId` (String):
: The VTCParticipant this signal pertains to (the gateway participant).

`protocol` (String):
: The gateway protocol. Matches VTCGateway `protocol`. One of the registered short aliases (`"pstn"`, `"sip"`, `"h323"`) or a reverse-domain identifier.

`signal` (String):
: The signal type. Opaque to JMAP VTC; defined by the external protocol. Examples:
  - PSTN: `"dtmf"`, `"flash"`
  - SIP: `"refer"`, `"info"`, `"hold"`, `"unhold"`, `"reinvite"`
  - H.323: `"h245cmd"`, `"facilityIndication"`

`data` (Object):
: Signal-specific payload. Opaque to JMAP VTC; the server relays without interpretation. Clients that understand the protocol and signal type MAY act on it; all others MUST ignore the event. Examples:
  - DTMF: `{"digit": "5", "duration": 160}`
  - SIP REFER: `{"referTo": "sip:bob@example.com", "referredBy": "sip:alice@example.com"}`
  - SIP hold: `{"direction": "sendonly"}`

`direction` (String):
: `"inbound"` (from the external party toward the call) or `"outbound"` (from a moderator toward the external party).

`timestamp` (UTCDate):
: Time the signal was generated.

Moderators MAY send outbound VTCGatewaySignals by including them in a VTCParticipant/set update on the target participant's `gatewayData` with a `"pendingSignal"` key. The server extracts the signal, delivers it to the gateway, and removes the key. This allows moderators to send DTMF, initiate transfers (SIP REFER), or perform other protocol-specific operations without JMAP VTC defining a method for each.

Example — inbound DTMF from a PSTN participant:

~~~json
{
  "@type": "VTCGatewaySignal",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "participantId": "gw-pstn-14155559876",
  "protocol": "pstn",
  "signal": "dtmf",
  "data": {"digit": "5", "duration": 160},
  "direction": "inbound",
  "timestamp": "2026-06-05T14:42:17Z"
}
~~~

Example — moderator sends outbound SIP REFER (call transfer) via `VTCParticipant/set`:

~~~json
{
  "update": {
    "gw-sip-alice": {
      "gatewayData/pendingSignal": {
        "signal": "refer",
        "data": {
          "referTo": "sip:bob@example.com",
          "referredBy": "sip:alice@example.com"
        }
      }
    }
  }
}
~~~

## In-Call Ephemeral Events {#in-call-events}

The following ephemeral WebSocket events carry real-time in-call state changes to connected clients. They are delivered over the same WebSocket connection as VTCRingEvent and VTCCallEndEvent. Clients subscribe to these events by including `"vtc"` in the `dataTypes` array of a `ChatStreamEnable` message (when {{JMAP-CHAT-WSS}} is present) or via `VTCStreamEnable` (when {{JMAP-VTC-WSS}} is present). These events are not persisted; they supplement JMAP `StateChange` notifications for VTCParticipant to provide low-latency UI updates.

### VTCParticipantEvent

Delivered when a participant joins, leaves, or is kicked from a call.

`@type` (String):
: `"VTCParticipantEvent"`.

`callId` (String):
: The VTCCall id.

`participantId` (String):
: The VTCParticipant id.

`event` (String):
: `"joined"`, `"left"`, or `"kicked"`.

`displayName` (String):
: The participant's display name at event time.

`role` (String):
: The participant's role at event time.

### VTCMediaStateEvent

Delivered when a participant's media state changes (mute/unmute, camera on/off, screen share start/stop, hand raise).

`@type` (String):
: `"VTCMediaStateEvent"`.

`callId` (String):
: The VTCCall id.

`participantId` (String):
: The VTCParticipant id.

`mediaState` (VTCMediaState):
: The participant's updated media state.

### VTCActiveSpeakerEvent

Delivered when the active (dominant) speaker changes. The server or media layer determines the active speaker based on audio levels.

`@type` (String):
: `"VTCActiveSpeakerEvent"`.

`callId` (String):
: The VTCCall id.

`participantId` (String):
: The VTCParticipant id of the current active speaker.

### VTCUnmuteRequestEvent {#unmute-request-event}

Delivered to a specific participant when a moderator requests that they unmute.

`@type` (String):
: `"VTCUnmuteRequestEvent"`.

`callId` (String):
: The VTCCall id.

`requestedBy` (String):
: The userId of the moderator making the request.

### VTCRecordingStateEvent {#vtc-recording-state-event}

Delivered to all participants when recording state changes. This is a mandatory consent signal.

`@type` (String):
: `"VTCRecordingStateEvent"`.

`callId` (String):
: The VTCCall id.

`recordingId` (String):
: The VTCRecording id.

`state` (String):
: The new recording state (`"recording"`, `"paused"`, `"stopped"`).

`initiatedBy` (String):
: The userId of the participant who initiated the state change.

## Multi-Device Forking {#multi-device-forking}

Ring notifications are delivered to ALL of the target participant's registered push subscriptions and active WebSocket connections. The first device to answer wins: the server accepts the first `VTCCall/set` update transitioning the call to `"active"` and dispatches `VTCCallEndEvent` with `endReason: "answered_elsewhere"` to all other devices of that participant.

When a call targets multiple participants, answering by any one target transitions the call to `"active"`. The server dispatches `VTCCallEndEvent` with `endReason: "answered"` to all devices of all other target participants that have not yet answered.

## Ring Timeout

Servers SHOULD enforce a ring timeout. If no target participant answers within a deployment-defined period (RECOMMENDED: 30 seconds), the server transitions the call to `"ended"` with `endReason: "missed"` and dispatches `VTCCallEndEvent` with `endReason: "timeout"` to all ringing devices.

# Access Control {#access-control}

## Chat-Bound Calls

When a VTCCall carries a `chatId`, `spaceId`, or `channelId` binding and `urn:ietf:params:jmap:chat` is present, access control is inherited from JMAP Chat's permission model:

- **Visibility:** A call is visible to members of the bound Chat or Space channel. Non-members MUST receive `notFound` for `VTCCall/get` and `VTCParticipant/get` requests on the call.
- **Join permission:** In a Space-bound call, the server SHOULD check the `"start_call"` permission (see {{JMAP-CHAT}}) before allowing `VTCParticipant/set create`. Participants who can `"view"` the channel MAY join an existing call even without `"start_call"` permission; `"start_call"` gates call creation, not joining.
- **Recording and livestream access:** VTCRecording and VTCLivestream objects inherit visibility from their parent VTCCall.
- **Space ban enforcement:** When a user is banned from a Space (via SpaceBan creation) and has an active VTCParticipant record in a call bound to that Space (via `spaceId`), the server MUST set `leftAt` on that VTCParticipant record and emit a `VTCParticipantEvent` with event `"left"`. The ban takes effect immediately on active calls.

## Standalone Calls

When no Chat binding is present (the VTC capability is used standalone), the server MUST enforce the following access rules:

- Participants (current or past) may access VTCCall and VTCParticipant records for calls in which they have a VTCParticipant entry.
- The call initiator may access VTCCall records they created.
- Non-participants MUST receive `notFound` for calls they cannot access.
- `VTCCall/query` MUST only return calls the authenticated user has access to.
- Deployment-defined administrative roles MAY have broader access.

# Federation {#federation}

Cross-server VTCCall state synchronization is out of scope for this document.

When used with JMAP Chat federation ({{JMAP-CHAT-FED}}), a call invitation MAY be delivered to remote participants via `Peer/deliver` carrying a MessageAction of type `"urn:jmap:chat:cap:vtc"` with the `joinUri` — using the existing out-of-band action mechanism defined in {{JMAP-CHAT}}. The receiving server MAY auto-create a local VTCCall record from the inbound MessageAction to provide call-state tracking on the remote side.

Federated call state synchronization — maintaining a consistent VTCCall object across multiple servers, with synchronized participant lists and state transitions — is a significantly more complex problem and is deferred to a future companion specification.

# Security Considerations {#security}

## joinUri Is Untrusted

`joinUri` is peer-supplied and opaque to the JMAP server. Clients MUST NOT connect to `joinUri` without explicit user initiation. Auto-joining a call is a privacy violation: it activates the microphone and potentially the camera without consent.

Servers MUST NOT fetch, probe, or validate `joinUri` values. Doing so exposes the server to SSRF attacks.

## Ring as Denial of Service {#ring-dos}

Ring calls cause device-level interruption: vibration, sound, and a full-screen incoming-call UI. Without rate limiting, an attacker could use repeated ring calls to harass a target.

Servers SHOULD enforce:

- Per-caller ring rate limits (e.g., no more than 3 ring calls to the same target within 60 seconds).
- Per-callee ring rate limits (e.g., no more than 10 incoming rings per minute from all callers combined).
- Blocked contacts MUST NOT be able to ring (see {{ring-notification}}).

Servers MAY temporarily suspend ring delivery for a target that is receiving rings at an abusive rate and fall back to silent `StateChange` notifications.

## Recording Consent {#recording-consent}

The server MUST notify all participants when recording state changes (started, paused, resumed, stopped). This notification is a mandatory consent signal. Jurisdictional requirements for recording consent (one-party vs. all-party) are deployment-defined, but the signaling mechanism MUST be present regardless of jurisdiction.

Servers SHOULD provide a mechanism for participants to leave a call when recording starts, as an implicit decline of consent.

## Call Metadata Exposure

Even without access to the media stream, the JMAP server observes: who called whom, when, for how long, from which device, and who else was on the call. This metadata is privacy-sensitive. Deployments requiring metadata privacy SHOULD apply the same mitigations noted in the E2EE section of {{JMAP-CHAT}}: message padding, cover traffic, and other transport-layer techniques outside the scope of this document.

## Media State Accuracy {#media-state-accuracy}

VTCMediaState is client-reported and server-relayed. The server has no media-layer visibility and cannot verify that `audio: false` actually means the microphone is off. A malicious client could report `muted: true` while transmitting audio, or `video: false` while the camera is active.

This is inherent to the media-agnostic design. Clients SHOULD NOT rely on VTCMediaState for security-critical decisions. The media layer is the only authority on what media is actually being transmitted.

## Moderator Privilege Escalation

The server MUST enforce role checks on all moderation actions ({{moderator-actions}}). A participant with `role: "participant"` MUST receive `forbidden` for any `VTCParticipant/set` update targeting another participant, and for any `VTCCall/set` operation restricted to moderators (ending a call, toggling lobby, creating or closing breakout rooms, toggling recording or livestream).

## Lobby Bypass

When `lobbyEnabled` is `true`, the server MUST NOT allow participants to skip the lobby via direct `VTCParticipant/set` manipulation. The lobby admission path — `lobbyState` transitioning from `"waiting"` to `"admitted"` — MUST require a moderator-initiated `VTCParticipant/set` update.

## Livestream Key Exposure {#livestream-key-exposure}

`streamKey` is an authentication credential for an external streaming platform. The server MUST NOT return the actual `streamKey` value in `VTCLivestream/get` responses for non-moderator participants. The server MUST return an empty string for the `streamKey` field for non-moderators.

## Gateway Participant Identity

`displayName` and `gatewayData` for participants who joined via a protocol gateway (`joinMethod` of `"pstn"`, `"sip"`, or `"h323"`) are derived from the external protocol's identity mechanisms (PSTN caller ID, SIP From header, H.323 alias). These identities are trivially spoofable on their respective networks and are not authenticated by the JMAP server. Clients SHOULD visually distinguish gateway participants from authenticated JMAP users to prevent impersonation.

## Gateway Signal Injection

VTCGatewaySignal events ({{gateway-signal}}) carry opaque protocol-specific data. The server MUST NOT interpret or execute gateway signal payloads; it relays them between the gateway infrastructure and the WebSocket connection. Outbound gateway signals (sent by moderators via `VTCParticipant/set`) MUST be restricted to moderators. The gateway infrastructure is responsible for validating that relayed signals are well-formed for its protocol.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:vtc`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Data Type Registration

IANA is requested to register the following entries in the "JMAP Data Types" registry:

Type Name: VTCCall
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:vtc`
Specification document: This document.

Type Name: VTCParticipant
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:vtc`
Specification document: This document.

Type Name: VTCRecording
Can reference blobs: Yes
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:vtc`
Specification document: This document.

Type Name: VTCLivestream
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:vtc`
Specification document: This document.

--- back

# Design Influences and Non-Normative Notes

This non-normative section documents the design influences from production calling systems and the explicit non-decisions where deployment latitude was preferred over prescription.

## Influences

- **Jitsi Meet** provided the primary feature-set reference: room calls with lobby/waiting room, moderator controls (mute others, kick, admit), breakout rooms, recording, livestreaming, PSTN dial-in, and hand raise are all Jitsi Meet application features modeled as JMAP data types in this specification.
- **Slack Huddles** inspired the room-call drop-in model. A Huddle is a persistent audio room in a channel that participants join and leave freely; VTCCall with `callType: "room"` captures this pattern.
- **Discord Voice Channels** reinforced the room-call model and informed the Space/channel binding design. Discord voice channels are persistent per-server rooms with the same permission model as text channels.
- **FaceTime** informed the ring-call push design. FaceTime's use of PushKit and CallKit on iOS to deliver a native incoming-call UI within one to two seconds is the benchmark for ring-call latency. The `VTCCallPush` payload and urgency guidance are designed to achieve equivalent behavior.
- **Zoom** informed the scheduled-call model, lobby/waiting room, breakout rooms, recording, and livestreaming features.
- **Google Meet** informed the lobby/waiting room and live-captions patterns. Live captions are delivered as chat messages (or ephemeral events in {{JMAP-CHAT}}) rather than a VTC-native type, following the simplification principle.
- **SIP (RFC 3261)** provided the session-state model: a call has a lifecycle with well-defined states and transitions. The ring/answer/decline/cancel/timeout transitions map directly from SIP INVITE/ACK/BYE/CANCEL semantics.
- **Signal** informed the ring-call E2EE model: minimal server state, push-to-ring, and the principle that the server should know as little as possible about the call beyond its existence.

## Explicit Non-Prescriptions

The following design choices were left to deployments rather than prescribed:

- **Media stack selection.** WebRTC, SIP/RTP, proprietary SFU, or any other media transport. The spec is media-agnostic by design.
- **SFU/MCU architecture.** Whether the deployment uses a Selective Forwarding Unit, a Multipoint Control Unit, peer-to-peer mesh, or a hybrid. The JMAP server does not participate in media routing.
- **WebRTC signaling.** SDP offer/answer and ICE candidate exchange are the media stack's responsibility. A future companion specification may define JMAP-based signaling relay, but this document does not.
- **STUN/TURN server deployment.** NAT traversal infrastructure is deployment-defined.
- **Recording storage backend.** Where and how recording blobs are stored. The spec only models the metadata and the resulting `blobId`.
- **Livestream ingest protocol.** RTMP is assumed but not prescribed; deployments may use SRT, HLS ingest, or any other protocol.
- **Transcription engine.** Live captions and transcription are delivered as chat messages or ephemeral events via {{JMAP-CHAT}}, not as VTC-native types. The transcription engine (Whisper, Vosk, cloud ASR, or other) is entirely deployment-defined.
- **Gateway carrier interconnection.** How PSTN, SIP, or H.323 gateways connect to their respective networks is deployment infrastructure.
- **Protocol-specific signaling.** Call transfer (SIP REFER), DTMF relay, hold/resume, codec renegotiation, H.245 commands, and any future verb added by an external specification are carried as opaque VTCGatewaySignal events (see {{gateway-signal}}). JMAP VTC does not define or constrain protocol-specific signals; it provides the pass-through mechanism.
- **Voicemail and IVR.** Out of scope. May be modeled as gateway-specific signal flows in deployments that support them.
- **Call queuing (contact center).** Out of scope. May be modeled as gateway-specific signal flows.
- **Virtual backgrounds, noise suppression, layout selection.** Client-side rendering concerns with no protocol surface.

# Complete Lifecycle Examples {#lifecycle-examples}

## Ring Call: Create, Answer, Record, End

This example shows a complete ring-call flow between Bob (initiator) and Alice (target).

### Step 1: Bob creates a ring call

Bob's client sends `VTCCall/set`:

~~~json
[["VTCCall/set", {
  "accountId": "bob-account",
  "create": {
    "c0": {
      "callType": "ring",
      "targetParticipantIds": ["user:alice@example.com"],
      "mediaTypes": ["audio", "video"],
      "subject": "Quick sync"
    }
  }
}, "0"]]
~~~

Server responds:

~~~json
[["VTCCall/set", {
  "accountId": "bob-account",
  "created": {
    "c0": {
      "id": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "state": "ringing",
      "initiatorId": "user:bob@example.com",
      "createdAt": "2026-06-05T14:30:00Z",
      "joinUri": "https://meet.example.com/room/xkz",
      "activeParticipantCount": 0
    }
  }
}, "0"]]
~~~

The server creates two VTCParticipant records (Bob and Alice, both with `joinedAt: null`), delivers a `VTCCallPush` via APNs VoIP push to Alice's iOS device, and delivers a `VTCRingEvent` to Alice's active WebSocket connection.

### Step 2: Alice's devices receive the ring

Alice's iOS device receives a VoIP push with the `VTCCallPush` payload and presents the CallKit incoming-call UI. Simultaneously, Alice's desktop client receives via WebSocket:

~~~json
{
  "@type": "VTCRingEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "initiatorId": "user:bob@example.com",
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkz"
}
~~~

### Step 3: Alice answers on her desktop

Alice's desktop client sends `VTCParticipant/set`:

~~~json
[["VTCParticipant/set", {
  "accountId": "alice-account",
  "update": {
    "user:alice@example.com": {
      "joinMethod": "webrtc"
    }
  }
}, "1"]]
~~~

The server sets `joinedAt`, transitions the VTCCall to `"active"`, and delivers a `VTCCallEndEvent` to Alice's iOS device:

~~~json
{
  "@type": "VTCCallEndEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "endReason": "answered_elsewhere"
}
~~~

Alice's iOS dismisses the CallKit UI.

### Step 4: Bob starts recording

Bob's client sends `VTCRecording/set`:

~~~json
[["VTCRecording/set", {
  "accountId": "bob-account",
  "create": {
    "r0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB"
    }
  }
}, "2"]]
~~~

Both participants receive a `VTCRecordingStateEvent` via WebSocket:

~~~json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "recording",
  "initiatedBy": "user:bob@example.com"
}
~~~

Alice's client displays a recording indicator.

### Step 5: Alice mutes

Alice's client sends `VTCParticipant/set`:

~~~json
[["VTCParticipant/set", {
  "accountId": "alice-account",
  "update": {
    "user:alice@example.com": {
      "mediaState/audio": false
    }
  }
}, "3"]]
~~~

Bob receives via WebSocket:

~~~json
{
  "@type": "VTCMediaStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "participantId": "user:alice@example.com",
  "mediaState": {
    "audio": false,
    "video": true,
    "screen": false,
    "raisedHand": false
  }
}
~~~

### Step 6: Bob ends the call

Bob's client sends `VTCCall/set`:

~~~json
[["VTCCall/set", {
  "accountId": "bob-account",
  "update": {
    "01J4XKZQN4MWVT8PPBEHTJ3AB": {
      "state": "ended"
    }
  }
}, "4"]]
~~~

The server sets `endedAt` and `endReason: "completed"`, stops the recording (transitioning it through `"stopped"` → `"processing"` → `"available"`), and delivers a `StateChange` to Alice's connection:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "alice-account": {
      "VTCCall": "s201",
      "VTCParticipant": "s202"
    }
  }
}
~~~

Alice's client fetches the updated VTCCall via `VTCCall/get` and sees `state: "ended"`.

## Room Call with Lobby: Create, Admit, Join, Leave

This example shows a room call with lobby mode in a Space channel.

### Step 1: Moderator creates a room call

~~~json
[["VTCCall/set", {
  "accountId": "mod-account",
  "create": {
    "c0": {
      "callType": "room",
      "mediaTypes": ["audio", "video"],
      "subject": "Office Hours",
      "lobbyEnabled": true,
      "spaceId": "01J3ABCDEF0000000000000000",
      "channelId": "01J3ABCDEF1111111111111111"
    }
  }
}, "0"]]
~~~

The call is immediately `"active"`. The Space's `activeCallId` is set to the new call's id (when {{JMAP-CHAT}} is present).

### Step 2: Guest joins and enters lobby

A channel member joins:

~~~json
[["VTCParticipant/set", {
  "accountId": "guest-account",
  "create": {
    "p0": {
      "callId": "01J4XL8QN5OWXU9RRBEIUS5CD",
      "joinMethod": "webrtc"
    }
  }
}, "1"]]
~~~

Because `lobbyEnabled` is `true`, the server sets `lobbyState: "waiting"`. The guest is not yet in the call. The moderator receives a `VTCParticipantEvent` (or `StateChange` for VTCParticipant) indicating a new lobby participant.

### Step 3: Moderator admits the guest

~~~json
[["VTCParticipant/set", {
  "accountId": "mod-account",
  "update": {
    "user:guest@example.com": {
      "lobbyState": "admitted"
    }
  }
}, "2"]]
~~~

The guest's `lobbyState` transitions to `"admitted"` and they can now see and hear other participants. The guest receives a `StateChange` for their VTCParticipant record.

### Step 4: Guest leaves

~~~json
[["VTCParticipant/set", {
  "accountId": "guest-account",
  "update": {
    "user:guest@example.com": {
      "leftAt": "2026-06-05T15:45:00Z"
    }
  }
}, "3"]]
~~~

The moderator receives a `VTCParticipantEvent`:

~~~json
{
  "@type": "VTCParticipantEvent",
  "callId": "01J4XL8QN5OWXU9RRBEIUS5CD",
  "participantId": "user:guest@example.com",
  "event": "left",
  "displayName": "Guest User",
  "role": "participant"
}
~~~

# Acknowledgements

The author thanks the Jitsi Meet project for the comprehensive open-source reference implementation that informed this specification's feature set, the authors of {{JMAP-CHAT}} for the chat protocol whose design patterns this specification follows, and the authors of SIP {{RFC3261}} for the session-state model that underpins the VTCCall lifecycle.
