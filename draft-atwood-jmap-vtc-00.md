---
title: JMAP for Video/Voice Teleconferencing
abbrev: JMAP VTC
docname: draft-atwood-jmap-vtc-00
category: std
stream: IETF

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
    title: "Universally Unique Lexicographically Sortable Identifier"
    target: https://github.com/ulid/spec

informative:
  RFC3261:
  RFC3515:
  RFC4733:
  RFC8291:
  RFC8825:
  JMAP-CHAT:
    title: "JMAP for Chat"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-CHAT-PUSH:
    title: "JMAP Chat Push Notifications"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-push-00
    date: 2026
  JMAP-CHAT-WSS:
    title: "JMAP Chat over WebSocket"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026
  JMAP-CHAT-FED:
    title: "JMAP Chat Federation"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-federation-00
    date: 2026
  JMAP-VTC-WSS:
    title: "JMAP VTC over WebSocket"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-wss-00
    date: 2026
  JMAP-CALENDARS:
    title: "JSON Meta Application Protocol (JMAP) for Calendars"
    author:
      - fullname: Neil Jenkins
      - fullname: Michael Douglass
    seriesinfo:
      Internet-Draft: draft-ietf-jmap-calendars-33
    date: 2026-10-13
  JMAP-SCENE:
    title: "JMAP for Scenes"
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-scene-00
    date: 2026
  ITU-E164:
    title: "The international public telecommunication numbering plan"
    author:
      org: International Telecommunication Union
    seriesinfo:
      ITU-T Recommendation: E.164
    date: 2026-02
    target: https://www.itu.int/rec/T-REC-E.164
  ITU-H323:
    title: "Packet-based multimedia communications systems"
    author:
      org: International Telecommunication Union
    seriesinfo:
      ITU-T Recommendation: H.323
    date: 2022-03
    target: https://www.itu.int/rec/T-REC-H.323
  ITU-H245:
    title: "Control protocol for multimedia communication"
    author:
      org: International Telecommunication Union
    seriesinfo:
      ITU-T Recommendation: H.245
    date: 2022-03
    target: https://www.itu.int/rec/T-REC-H.245
  APNS-REQUEST:
    title: "Sending notification requests to APNs"
    author:
      org: Apple Inc.
    target: https://developer.apple.com/documentation/usernotifications/sending-notification-requests-to-apns

--- abstract

This document defines JMAP VTC, a standalone JMAP capability ({{RFC8620}}) for managing video and voice teleconferencing sessions. It defines the `urn:ietf:params:jmap:vtc` capability; four top-level data types (VTCCall, VTCParticipant, VTCRecording, and VTCLivestream) plus the embedded VTCMediaState object; standard JMAP methods for managing calls and participants; push notification payloads for incoming-call ring events; and ephemeral WebSocket events for call state changes.

The specification models call *state* — who is in a call, what state the call is in, and how to join it — without prescribing the media transport. Actual voice and video media travel through a deployment-chosen stack (WebRTC, SIP/RTP, or any other media framework); the JMAP server is a call-state database, not a media server.

The capability is standalone (`urn:ietf:params:jmap:vtc`). When JMAP Chat ({{JMAP-CHAT}}) is also present, VTCCall objects MAY carry optional back-references to Chat and Space objects. When JMAP Calendars ({{JMAP-CALENDARS}}) is also present, VTCCall objects MAY bind to CalendarEvent objects for scheduled meetings.

--- middle

# Introduction

Video and voice calling is a core function of modern communication systems, yet the signaling, state management, and notification layers are typically tightly coupled to a specific media stack or platform. SIP {{RFC3261}} defines a comprehensive session initiation protocol but conflates signaling, registration, routing, and presence into a single system. Production calling products (Jitsi Meet, Zoom, Google Meet, Slack Huddles, Discord Voice Channels, FaceTime) each implement their own call-state model, and none expose it as a generic, standards-based API.

This document defines a JMAP capability that models call state as JMAP data types with standard get/set/changes/query methods. The server tracks who created a call, who is in it, what media types are active, and how to join — but the server does not participate in media negotiation or transport. The `joinUri` field on every VTCCall points to the deployment's media-layer entry point; JMAP VTC has no opinion on what lives behind that URI.

## Design Philosophy {#design-philosophy}

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

Protocol-specific signaling beyond the core lifecycle — call transfer (SIP REFER {{RFC3515}}), DTMF {{RFC4733}}, hold/resume, codec renegotiation, and any future SIP or ITU-T verb — is carried as opaque VTCGatewaySignal events (see {{gateway-signal}}). JMAP VTC provides the pass-through mechanism without defining or constraining individual signals, ensuring that external protocol evolution does not require amendments to this specification.

## Relationship to JMAP Chat (recommended) {#chat-delegation}

Deployments SHOULD advertise `urn:ietf:params:jmap:chat` ({{JMAP-CHAT}}) alongside `urn:ietf:params:jmap:vtc`. When both are present, VTCCall objects carry `chatId`, `spaceId`, and `channelId` back-references, and several collaboration features are delegated to JMAP Chat rather than re-defined in this specification:

- **In-call text chat** is the Chat bound via `chatId`. When `chatId` is supplied, the call is bound to that Chat for in-call text messaging; when omitted, the call has no text backchannel. Messages sent to the bound Chat appear as in-call chat.
- **In-call reactions** (emoji floats) are reactions on Messages in the bound Chat.
- **Live captions and transcription** are delivered as Messages in the bound Chat, with `senderId` providing speaker attribution. Neither this document nor {{JMAP-CHAT-WSS}} defines an ephemeral event type for reactions or captions.
- **Lobby communication** uses the bound Chat: lobby participants can message moderators before being admitted.
- **Scheduled-call invitees** are derived from the Space membership (for Space-bound calls) or the CalendarEvent invitees (for calendar-bound calls).
- **Access control** is inherited from the Space/channel permission model (see {{access-control}}).
- **Active call banner** is signaled via the `activeCallId` field on Chat (see {{JMAP-CHAT}}).

The existing `urn:jmap:chat:cap:vtc` Endpoint and MessageAction type URIs defined in {{JMAP-CHAT}} provide the integration point in the other direction: a chat message can carry a VTC action whose `uri` points to the call's `joinUri`.

Without JMAP Chat, the VTC capability is fully functional as a standalone call-state manager. Calls have no chat binding, and the delegated features above are unavailable.

## Relationship to JMAP Scene (optional)

When `urn:ietf:params:jmap:scene` ({{JMAP-SCENE}}) is also advertised, SceneRegion objects MAY carry an `activeCallId` field referencing a VTCCall. This binding enables spatial audio: the Scene simulation layer can use SceneAvatar positions to spatialize VTC audio for participants in the region. The VTC capability itself is unaware of Scene; the binding is defined entirely in {{JMAP-SCENE}} and is optional. Without JMAP Scene, calls have no spatial component.

## Relationship to JMAP Calendars (optional)

When `urn:ietf:params:jmap:calendars` ({{JMAP-CALENDARS}}) is also advertised, VTCCall objects MAY carry a `calendarEventId` to bind a scheduled call to a CalendarEvent. The event's start time is the call's intended start time. Without JMAP Calendars, scheduled calls use an explicit `scheduledStartAt` field instead of a calendar binding.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout. The term "userId" refers to the authenticated user identity string provided by the transport layer, equivalent to `ChatContact.identity` when {{JMAP-CHAT}} is present. It is not a `ChatContact.id`, which is a server-assigned identifier local to the JMAP Chat server.

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

JMAP VTC defines the call lifecycle (create, join, leave, end) and participant state (media, role, lobby). Protocol-specific signaling beyond this lifecycle — SIP REFER, DTMF, hold/resume, codec renegotiation, H.245 {{ITU-H245}} commands, or any future verb added by an external specification — passes through as opaque data via VTCGatewaySignal events (see {{gateway-signal}}). This design ensures that changes to PSTN, SIP, or ITU-T specifications never require amendments to this document.

`protocol` (String):
: A reverse-domain identifier for the gateway protocol. This specification defines `"pstn"`, `"sip"`, and `"h323"` as short aliases for the three common telephony protocols: the PSTN, SIP {{RFC3261}}, and H.323 {{ITU-H323}}. Vendor-specific or proprietary protocols SHOULD use reverse-domain notation (e.g., `"com.apple.facetime"`, `"org.signal"`, `"com.zoom.zoomphone"`). Clients MUST ignore unrecognized values.

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
  - PSTN: E.164 {{ITU-E164}} phone number (e.g., `"+14155551234"`)
  - SIP: SIP URI ({{RFC3261, Section 19.1}}) (e.g., `"sip:meet@example.com"`)
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

VTCCallPolicy is an embedded object on VTCCall containing moderator-controlled media policies for the call. Non-moderators SHOULD receive `forbidden` when attempting to update these fields via `VTCCall/set`.

`muteOnEntry` (Boolean):
: When `true`, participants join with `mediaState.audio` set to `false`. Default: `false`.

`videoOffOnEntry` (Boolean):
: When `true`, participants join with `mediaState.video` set to `false`. Default: `false`.

`participantsCanUnmute` (Boolean):
: When `false`, non-moderator participants SHOULD receive `forbidden` when attempting to set their own `mediaState.audio` to `true` via `VTCParticipant/set`, unless their `unmuteRequested` is `true`. A moderator invites a participant to unmute with the ask-to-unmute signal (see {{ask-to-unmute}}). Default: `true`.

`participantsCanShareScreen` (Boolean):
: When `false`, non-moderator participants SHOULD receive `forbidden` when attempting to set their own `mediaState.screen` to `true`. Default: `true`.

`participantsCanStartVideo` (Boolean):
: When `false`, non-moderator participants SHOULD receive `forbidden` when attempting to set their own `mediaState.video` to `true`. Default: `true`.

## VTCParticipant {#vtc-participant}

A VTCParticipant describes one participant in a VTCCall. It is a top-level JMAP data type with its own get/set/changes/query methods.

### VTCParticipant ID Assignment

VTCParticipant ids are ordinary server-assigned JMAP ids, unique within the call's hosting account ({{hosting-account}}). An authenticated user has at most one VTCParticipant record per call, which is reused when the user rejoins ({{reconnection}}). Clients find a user's record in a call with `VTCParticipant/query` filtering on `callId` and `userId`.

### VTCParticipant Object Fields

`id` (String, immutable, server-set):
: Opaque server-assigned identifier.

`callId` (String, immutable except for the breakout-room assignment in {{participant-moderator-actions}}):
: The id of the VTCCall this participant belongs to.

`userId` (String|null, immutable, server-set):
: The authenticated user identity string. `null` for unauthenticated participants (PSTN callers, anonymous guests).

`displayName` (String):
: Display name for this participant. For authenticated users, derived from the user profile or ChatContact record when {{JMAP-CHAT}} is present. For PSTN callers, derived from caller ID. For anonymous guests, a server-assigned or self-reported name. The server sets `displayName` at join time from the user profile or `ChatContact` record (or `SpaceMember.nick` when applicable). If the source display name changes during the call, the server is not required to update `displayName` on existing VTCParticipant records. Clients displaying call participants alongside Chat contacts may prefer the Chat-side display name for consistency.

`role` (String):
: `"moderator"` or `"participant"`. Moderators may perform moderation actions (see {{moderator-actions}}). The call initiator receives `"moderator"` by default; the server MAY grant moderator status to additional participants based on deployment policy. The VTC role (`"moderator"` / `"participant"`) is independent of any Chat or Space role. Deployments MAY choose to auto-promote Space administrators or Chat owners to `"moderator"` at join time, but this is not required by the protocol.

`joinedAt` (UTCDate|null, server-set):
: Time this participant joined the call. `null` if the participant has been invited but has not yet joined (e.g., in a ringing call).

`leftAt` (UTCDate|null):
: Time this participant left the call. `null` if currently in the call.

`callResponse` (String|null):
: For ring-call targets: the participant's response to the ring. Values: `"pending"` (not yet responded), `"accepted"`, `"declined"`. `null` for participants who were not ring targets (room/scheduled call joins, dial-out participants). The server sets this to `"pending"` when creating VTCParticipant records for ring targets, to `"accepted"` when the participant answers, and clients set it to `"declined"` to decline.

`joinMethod` (String):
: How this participant connected. This specification defines `"webrtc"` (WebRTC {{RFC8825}}), `"sip"`, `"pstn"`, and `"h323"` as short aliases. When the participant arrived via a gateway, this value corresponds to the VTCGateway `protocol`. Vendor-specific connection methods SHOULD use reverse-domain notation (e.g., `"com.apple.facetime"`). Clients MUST ignore unrecognized values.

`gatewayData` (Object|null):
: Protocol-specific state for participants who joined via a gateway (`joinMethod` other than `"webrtc"`). The server stores and relays this object without interpretation, with one exception: the server MUST recognize and process the reserved `"pendingSignal"` key for outbound gateway signals (see {{gateway-signal}}). Clients that understand the protocol MAY inspect it; all others MUST ignore it. `null` for WebRTC participants and when no gateway-specific data is available. Examples:
  - PSTN: `{"callerIdNumber": "+14155559876", "callerIdName": "Alice Smith"}`
  - SIP: `{"sipUri": "sip:alice@example.com", "userAgent": "Opal/4.0"}`
  - H.323: `{"alias": "alice", "endpointType": "terminal"}`

`lobbyState` (String|null):
: Present only when the call has `lobbyEnabled: true`. Values: `"waiting"`, `"admitted"`, `"rejected"`. `null` when lobby is not active or the participant bypassed the lobby (moderators).

`mediaState` (VTCMediaState):
: The participant's current self-reported media state.

`speakerTimeMs` (UnsignedInt, server-set):
: Cumulative milliseconds of talk time for this participant, tracked by the server or media layer. Servers that cannot track speaker time MUST set this to `0`. For calls with `e2eeEnabled: true`, see {{e2ee-active-speaker}}.

`kickedBy` (String|null, server-set):
: The userId of the moderator who removed this participant from the call. `null` if the participant left voluntarily or is still in the call.

`e2eeFingerprint` (String|null):
: When the call has `e2eeEnabled: true`: the participant's public key fingerprint for E2EE verification, as reported by the media layer. Clients can display this for out-of-band verification (e.g., reading the fingerprint aloud). The format is media-layer-defined. `null` when E2EE is not active or the participant has not yet completed key exchange.

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
: The call's hosting account ({{hosting-account}}).

`callType` (String, immutable):
: `"ring"`, `"room"`, or `"scheduled"`. Determines the state machine path (see {{state-machine}}).

`state` (String):
: Current call state. See {{state-machine}} for valid values and transitions. The server sets this field, except that a moderator may set it to `"ended"` (see {{ending-a-call}}).

`createdAt` (UTCDate, immutable, server-set):
: Time the call was created.

`endedAt` (UTCDate|null, server-set):
: Time the call ended. `null` while the call is active or pending.

`endReason` (String|null, server-set):
: Reason the call ended. `null` while the call has not ended. Values:
  - `"completed"`: An active call was ended normally (moderator ended or all participants left).
  - `"missed"`: A ring call timed out with no answer.
  - `"declined"`: All ring-call targets declined.
  - `"cancelled"`: The initiator withdrew a ringing or pending call before anyone answered/joined.
  - `"failed"`: A server or media-layer error prevented the call from proceeding.
  - `"timeout"`: A deployment-defined inactivity timeout fired on an active call with no participants.

`initiatorId` (String, immutable, server-set):
: The userId of the participant who created the call.

`subject` (String|null):
: Optional human-readable call title (e.g., `"Standup"`, `"1:1 with Alice"`).

`joinUri` (String):
: The media-layer entry point for this call. Opaque to the JMAP server. May be a WebRTC room URL, a Jitsi Meet room URL, a SIP URI, or any other media endpoint. Generated by the server for local calls; for a call created from a federated invitation ({{federation}}) it comes from the remote peer. Clients treat it as untrusted; see {{joinuri-untrusted}}.

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
: For scheduled calls: the intended start time. Clients typically display this to invitees. The time is advisory: the server transitions the call from `"pending"` to `"active"` when the first participant joins, whether before or after this time, and does not auto-transition based on the clock alone (a scheduled call with no participants stays `"pending"`).

`gatewayPin` (String|null):
: PIN or access code for joining this call via a protocol gateway. Present when a gateway (PSTN, SIP, H.323) requires a PIN to associate an inbound connection with this call. `null` when no gateway requires a PIN or no gateways are configured.

`e2eeEnabled` (Boolean):
: `true` when end-to-end encryption is active for this call's media streams. The mechanism (WebRTC Encoded Transform, SFrame, or other) is media-layer-defined; this field is the state signal for client UI (e.g., displaying a lock icon). Default: `false`. The call creator MAY set this field at create time, and a moderator MAY change it from `false` to `true` via update. The server SHOULD return `forbidden` if a non-moderator attempts to update `e2eeEnabled`. The server SHOULD reject an update from `true` to `false` with `invalidProperties` naming `e2eeEnabled`, so that participants never have to detect a downgrade. When `true`, the server SHOULD NOT access plaintext media itself. Recording, livestreaming, and gateway participants are available only through a disclosed key-holding participant, and requests that would need server-side media access are rejected ({{e2ee-media-participants}}). See {{e2ee-calls}}.

### Optional Binding Fields

The following fields are present only when the corresponding companion capability is advertised on the same account:

`chatId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Chat ({{JMAP-CHAT}}) associated with this call. `null` if the call has no chat binding. Same-account only.

`spaceId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Space ({{JMAP-CHAT}}) associated with this call. `null` if the call has no Space context.

`channelId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a channel Chat within the Space. `null` if the call is Space-wide or has no channel context.

When a Space or channel Chat referenced by `spaceId` or `channelId` is destroyed, the server MUST clear the corresponding field to `null` on all VTCCalls that reference it. This is a side effect of the Space or Chat destruction and changes VTCCall state ({{RFC8620}} Section 7).

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

- `"pending"`: The call has been created and no participant has joined yet.
- `"pending"` → `"active"`: The first participant joins.
- `"pending"` → `"ended"`: A moderator cancels the scheduled call before it starts.
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

`initiatedBy` (String, server-set):
: The userId of the participant who started the recording.

`blobId` (String|null, server-set):
: JMAP blob reference for the recording file. Available only when `state` is `"available"`.

`size` (UnsignedInt|null, server-set):
: Recording file size in bytes. Available only when `state` is `"available"`.

`duration` (UnsignedInt|null, server-set):
: Recording duration in seconds. Available only when `state` is `"stopped"`, `"processing"`, or `"available"`.

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
: The stream key or authentication credential for the ingest endpoint. This field is write-only for non-moderators: the server SHOULD return an empty string in `VTCLivestream/get` responses for participants whose role is not `"moderator"`. See {{livestream-key-exposure}}.

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
  "id": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
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
  "chatId": "01J4XL2RN5NXWS9QQCFJVJ4BC0",
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

When creating a VTCCall with a `chatId` that is already referenced by another VTCCall in `"active"` or `"ringing"` state, the server MUST return an `invalidProperties` SetError naming `chatId`, with description "A call is already active on this Chat." This ensures that `Chat.activeCallId` is unambiguous and at most one call is active per Chat at any time.

The following checks apply to every `create`. When `chatId` is supplied but `urn:ietf:params:jmap:chat` is not present in the server's capabilities, the server MUST return `invalidProperties` naming `chatId` with a description indicating the Chat capability is required. When `chatId` does not resolve to a Chat object, the server MUST return `invalidProperties` naming `chatId`. The server SHOULD return the same error when the caller is not a member of that Chat, so that the response does not reveal that the Chat exists. When the call carries a `spaceId` or `channelId` binding and {{JMAP-CHAT}} is present, the server SHOULD return `forbidden` if the caller lacks the `"start_call"` permission in the bound Space or channel.

#### Creating a Ring Call

`create` with `callType: "ring"` accepts:

`targetParticipantIds` (String[], required):
: The userIds of the participants to ring. The server creates a VTCParticipant record for each target (with `joinedAt: null` and `callResponse: "pending"`) and one for the initiator (with `joinedAt` set to the creation time, since the initiator is in the call from the start), dispatches ring notifications (see {{ring-notification}}) to all targets, and transitions the call to `"ringing"`.

`mediaTypes` (String[], required):
: The media types for this call (e.g., `["audio", "video"]`).

Optional: `subject` (String), `e2eeEnabled` (Boolean), `chatId` (String), `spaceId` (String), `channelId` (String).

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
      "id": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
      "state": "ringing",
      "initiatorId": "user:bob@example.com",
      "createdAt": "2026-06-05T14:30:00Z",
      "joinUri": "https://meet.example.com/room/xkz",
      "activeParticipantCount": 1
    }
  }
}
~~~

Servers MUST return `invalidProperties` naming `targetParticipantIds` when `targetParticipantIds` is empty. Servers MUST return `invalidProperties` naming `mediaTypes` when `mediaTypes` contains values not present in the account-level `supportedMediaTypes`. Servers MUST return `forbidden` when the account's `mayCreateCall` is `false`. Servers MUST return `forbidden` when `supportsRingCalls` is `false`. Servers SHOULD silently drop targets that correspond to blocked contacts ({{ring-notification}}) and proceed with the remaining targets. If all targets are blocked, the server creates the call as usual and lets it end by ring timeout, so that the initiator cannot tell a block from an unanswered call. The common `chatId` and `"start_call"` checks in {{vtc-call-set}} also apply.

#### Creating a Room Call

`create` with `callType: "room"` accepts:

`mediaTypes` (String[], required):
: The media types for this call.

Optional: `subject` (String), `lobbyEnabled` (Boolean), `joinPassword` (String), `e2eeEnabled` (Boolean), `chatId` (String), `spaceId` (String), `channelId` (String).

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

Servers MUST return `forbidden` when `mayCreateCall` is `false` or `supportsRoomCalls` is `false`. The common `chatId` and `"start_call"` checks in {{vtc-call-set}} also apply.

#### Creating a Scheduled Call

`create` with `callType: "scheduled"` accepts:

`scheduledStartAt` (UTCDate, required):
: The intended start time, normally in the future. Servers MAY reject past values with an `invalidProperties` SetError naming `scheduledStartAt`.

`mediaTypes` (String[], required):
: The media types for this call.

Optional: `subject` (String), `lobbyEnabled` (Boolean), `joinPassword` (String), `e2eeEnabled` (Boolean), `chatId` (String), `spaceId` (String), `channelId` (String), `calendarEventId` (String).

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

Servers MUST return `forbidden` when `mayCreateCall` is `false` or `supportsScheduledCalls` is `false`. The common `chatId` and `"start_call"` checks in {{vtc-call-set}} also apply.

#### Ending a Call {#ending-a-call}

A moderator (or the initiator of a ring call) calls `VTCCall/set` to transition `state` to `"ended"`. The server sets `endedAt` and dispatches state-change notifications to all participants. The server sets `endReason` based on context:

- `"completed"`: The call was in state `"active"` when ended.
- `"cancelled"`: The call was in state `"ringing"` (the initiator withdrew the ring before anyone answered) or `"pending"` (a moderator cancelled a scheduled call before anyone joined).

When a call ends, the server stops every VTCRecording for the call in state `"recording"` or `"paused"` and every VTCLivestream in state `"starting"` or `"live"`, as if a moderator had set each to `"stopped"`.

Example update:

~~~json
{
  "update": {
    "01J4XKZQN4MWVT8PPBEHTJ3AB0": {
      "state": "ended"
    }
  }
}
~~~

The server SHOULD return `forbidden` if the caller is not a moderator on this call. The server MUST return `invalidProperties` naming `state` if the call is already in state `"ended"`. The server MUST return `invalidProperties` naming `state` for any `state` transition not permitted by the state machine ({{state-machine}}).

#### Moderator Actions on VTCCall {#moderator-actions}

Participants with `role: "moderator"` MAY use `VTCCall/set` to perform the following call-level actions. Non-moderators SHOULD receive `forbidden` for these operations. Participant-level moderator actions (mute, kick, admit, role changes) are performed via `VTCParticipant/set` (see {{vtc-participant-methods}}).

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

All filter properties are combined with logical AND. The query SHOULD return only calls the authenticated user has access to (see {{access-control}}).

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

For ring calls, the server creates VTCParticipant records for all targets (with `joinedAt: null`) when the call is created. A target participant answers by calling `VTCParticipant/set` in the call's hosting account ({{hosting-account}}) with an `update` on their existing record, setting `joinMethod` to their connection type. The target finds that record with `VTCParticipant/query` filtering on `callId` and `userId`. The server sets `joinedAt` to the current time and `callResponse` to `"accepted"`.

The server MUST validate that:

1. The call is in state `"ringing"` or `"active"`.
2. The caller's userId matches the VTCParticipant record's userId.
3. The record's `callResponse` is `"pending"`.

On success, if the call is `"ringing"`, the server transitions it to `"active"`. In either case the server dispatches a `VTCCallEndEvent` with `endReason: "answered_elsewhere"` to the answering user's other ringing devices (see {{multi-device-forking}}). Accepting answers while the call is `"active"` lets every target of a multi-target ring call join, not only the first to answer.

Example update (target participant answers):

~~~json
{
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "joinMethod": "webrtc"
    }
  }
}
~~~

The server MUST return `invalidProperties` naming `joinMethod` if the call is not in state `"ringing"` or `"active"`, or if the record's `callResponse` is not `"pending"`. The server SHOULD return `forbidden` if the caller's userId does not match the VTCParticipant record's userId. An unknown VTCParticipant id yields `notFound` ({{RFC8620}} Section 5.3).

#### Joining a Room or Scheduled Call {#joining}

A participant joins by calling `VTCParticipant/set` with a `create`:

`callId` (String, required):
: The VTCCall to join.

`joinMethod` (String, required):
: How this participant is connecting. One of the registered short aliases (`"webrtc"`, `"sip"`, `"pstn"`, `"h323"`) or a reverse-domain identifier.

The server sets `id`, `userId`, `displayName`, `role`, `joinedAt`, `mediaState` (defaults), and `speakerTimeMs` (to `0`).

For **room calls**, the server validates that the call is in state `"active"`.

For **scheduled calls**, the server validates that the call is in state `"pending"` or `"active"`. When a participant joins a `"pending"` scheduled call, the server transitions the call to `"active"`.

When `lobbyEnabled` is `true`, the server sets the new participant's `lobbyState` to `"waiting"` and leaves `joinedAt` `null`. The participant does not enter the call until a moderator admits them: admission sets `joinedAt`, and rejection sets `leftAt`. A waiting participant is therefore not counted in `activeParticipantCount` and does not receive in-call events.

Example create (joining a room call):

~~~json
{
  "create": {
    "p0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
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
      "id": "01J4XMVQK7NRYS9PPCEHTK4CD0",
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
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

The server SHOULD return `notFound` if the `callId` does not exist or the caller does not have access to it. When `urn:ietf:params:jmap:chat` is present, callers without access include a caller who lacks `"view"` permission on the channel of a Space-bound call, a caller who is not a member of the Chat bound by `chatId`, and a caller whom the call initiator has blocked (`ChatContact.blocked` is `true` on the initiator's contact record for the caller). Using `notFound` for all of these keeps the response from revealing that the call exists or that the caller is blocked. The server MUST return `invalidProperties` naming `callId` if the call is in state `"ended"`. If the call's `activeParticipantCount` has reached `maxParticipantsPerCall`, the server MUST return `overQuota`. If the account already has `maxConcurrentCalls` active calls, the server MUST return `overQuota`.

#### Declining a Ring Call

A target participant whose VTCParticipant record was created by the ring (with `joinedAt: null`) calls `VTCParticipant/set` with an `update` setting `callResponse` to `"declined"`. The server sets `leftAt` to the current time. When all target participants have declined and the call is still `"ringing"`, the server transitions the call to `"ended"` with `endReason: "declined"`.

~~~json
{
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "callResponse": "declined"
    }
  }
}
~~~

The server SHOULD return `forbidden` if the caller's userId does not match the VTCParticipant record's userId. The server MUST return `invalidProperties` naming `callResponse` if the call is not in state `"ringing"` or `"active"`, or the participant's `callResponse` is not `"pending"`. Other targets of a multi-target ring may still decline after the call becomes `"active"` ({{multi-device-forking}}).

#### Updating Media State

A participant calls `VTCParticipant/set` to `update` their own `mediaState` fields:

~~~json
{
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "mediaState/audio": false
    }
  }
}
~~~

Updates targeting another participant's `mediaState` are moderator actions ({{participant-moderator-actions}}). The call policy ({{vtc-call-policy}}) limits which of their own `mediaState` fields a non-moderator can set to `true`.

When `policy.muteOnEntry` is `true`, the server sets `mediaState.audio` to `false` on the VTCParticipant record at join time, regardless of the client's requested value. Similarly for `policy.videoOffOnEntry` and `mediaState.video`.

#### Leaving a Call

A participant calls `VTCParticipant/set` with an `update` setting their own `leftAt`. The server records its own current time as `leftAt`; the value the client supplies only signals the intent to leave.

When the last active participant leaves a ring or scheduled call, the server transitions the VTCCall to `"ended"` with `endReason: "completed"`, or `"cancelled"` if the call was still `"ringing"`. A room call stays `"active"` when it empties; it ends only as described in {{state-machine}}.

When the last moderator leaves a call that still has active participants, the server SHOULD grant `"moderator"` to the remaining authenticated participant (non-null `userId`) who joined earliest, so that someone can still admit lobby participants and end the call.

~~~json
{
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "leftAt": "2026-06-05T15:02:00Z"
    }
  }
}
~~~

The server SHOULD return `forbidden` if a caller who is not a moderator on the call sets `leftAt` on a VTCParticipant record whose userId does not match their own. Setting another participant's `leftAt` is a kick ({{participant-moderator-actions}}). The server MUST return `invalidProperties` naming `leftAt` if the participant has already left (`leftAt` is non-null).

#### Reconnecting {#reconnection}

A participant who has left (has a non-null `leftAt`) re-joins the same call by updating their existing VTCParticipant record to set `leftAt` to `null`. The server sets `joinedAt` to the current time, clears `kickedBy`, and preserves `role` and `speakerTimeMs`; the access checks, errors, and lobby rules for joining ({{joining}}) apply. A `create` for a user who already has a record in the call fails with an `alreadyExists` SetError ({{RFC8620}} Section 5.4) whose `existingId` names that record. Reusing the record keeps a single continuous participant identity across disconnections.

#### Participant-Level Moderator Actions {#participant-moderator-actions}

Participants with `role: "moderator"` on the same VTCCall MAY use `VTCParticipant/set` to update other participants' records. Non-moderators SHOULD receive `forbidden` for updates targeting other participants.

**Mute another participant:**
: Update the target's `mediaState/audio` to `false`. The media layer SHOULD honor this by muting the participant's audio stream; enforcement is media-layer-dependent.

**Ask to unmute:**
: When `policy.participantsCanUnmute` is `false`, a moderator may request that a participant unmute by updating the target's `unmuteRequested` to `true`. The server delivers a `VTCUnmuteRequestEvent` (see {{unmute-request-event}}) to the target participant's WebSocket connections. The participant may then choose to unmute (which clears `unmuteRequested`) or ignore the request. This is a soft request, not a forced unmute.
  {: #ask-to-unmute}

**Kick a participant:**
: Update the target's `leftAt`. The server records its own current time as `leftAt` and sets `kickedBy` to the moderator's userId.

**Admit a lobby participant:**
: Update the target's `lobbyState` from `"waiting"` to `"admitted"`. The server sets the target's `joinedAt` to the current time.

**Reject a lobby participant:**
: Update the target's `lobbyState` from `"waiting"` to `"rejected"`. The server sets the target's `leftAt` to the current time.

**Grant or revoke moderator role:**
: Update a participant's `role` between `"moderator"` and `"participant"`. A moderator MUST NOT revoke their own moderator role if they are the last moderator; the server MUST reject this with `forbidden`.

**Assign to a breakout room:**
: Update a participant's `callId` from the parent call to a child breakout-room call. This is the only update that `callId` accepts. The server does not move the record: it sets `leftAt` on the parent-call entry and creates (or reconnects) a VTCParticipant record on the child call. The server MUST return `invalidProperties` naming `callId` if the target `callId` is not a child of the participant's current call.

Example — admitting a lobby participant and muting another:

~~~json
{
  "update": {
    "01J4XQ3TP0SYWW2TTFHKWX7JK0": {
      "lobbyState": "admitted"
    },
    "01J4XR4VQ1TZXX3VVGJMXY8KM0": {
      "mediaState/audio": false
    }
  }
}
~~~

The server MUST return `invalidProperties` naming `lobbyState` when a lobby-state transition is invalid (e.g., `"admitted"` → `"waiting"` or `"rejected"` → `"admitted"`).

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

The server creates a VTCParticipant with `joinedAt: null` and initiates the outbound call via the gateway. When the external party answers, the server sets `joinedAt`. If the outbound call fails or is not answered, the server sets `leftAt` and records the gateway-specific failure reason in `gatewayData` (e.g., `{"failureReason": "busy"}`).

Example create (PSTN dial-out):

~~~json
{
  "create": {
    "g0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
      "joinMethod": "pstn",
      "gatewayData": {"dialNumber": "+14155559876"},
      "displayName": "Alice (mobile)"
    }
  }
}
~~~

Non-moderators SHOULD receive `forbidden` for dial-out creates. The server MUST return `invalidProperties` naming `joinMethod` when `joinMethod` does not match any VTCGateway `protocol` whose `supportsDialOut` is `true`. The server MUST return `invalidProperties` naming `callId` when the call is in state `"ended"`. When the call has `e2eeEnabled: true`, a dial-out whose gateway cannot join as a disclosed key-holding participant is rejected with `forbidden`, as {{e2ee-media-participants}} requires.

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

All current and past participants of the associated VTCCall may retrieve VTCRecording objects. The server SHOULD return `notFound` for recording ids belonging to calls the authenticated user has no access to.

### VTCRecording/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCRecording/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

Only participants with `role: "moderator"` on the associated VTCCall MAY create, update, or destroy VTCRecording objects. Non-moderators SHOULD receive `forbidden`.

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
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0"
    }
  }
}
~~~

When the call has `e2eeEnabled: true`, a recording that would require server-side media access is rejected as {{e2ee-media-participants}} describes. The server MAY accept the create for such a call when the recording is performed by a key-holding recorder that joins the call as a participant ({{e2ee-media-participants}}). The server MUST return `forbidden` when the account-level `supportsRecording` is `false`. The server MUST return `invalidProperties` naming `callId` when the call is not in state `"active"`.

On success, the server SHOULD deliver a `VTCRecordingStateEvent` ({{vtc-recording-state-event}}) with `state: "recording"` to all participants in the call. This is the consent signal (see {{recording-consent}}). The event reaches only clients with an active WebSocket subscription, and a participant who joins mid-recording receives no event, so on joining a call a client SHOULD check `VTCRecording/query` for recordings of the call in state `"recording"` or `"paused"` and display them.

#### Pausing, Resuming, and Stopping a Recording

`update` supports patching `state`:

- `"recording"` → `"paused"`: Pause recording. The server delivers a `VTCRecordingStateEvent` with `state: "paused"`.
- `"paused"` → `"recording"`: Resume recording. The server delivers a `VTCRecordingStateEvent` with `state: "recording"`.
- `"recording"` or `"paused"` → `"stopped"`: Stop recording. The server sets `stoppedAt` and delivers a `VTCRecordingStateEvent` with `state: "stopped"`.

Example update (stop recording):

~~~json
{
  "update": {
    "01J4XP2SN9RXVV1SSEGJVT6GH0": {
      "state": "stopped"
    }
  }
}
~~~

The server MUST return `invalidProperties` naming `state` for any state transition not listed above (e.g., `"stopped"` → `"recording"`, `"available"` → anything). After stopping, the server transitions the recording through `"processing"` (while the recording file is prepared) to `"available"` (when `blobId`, `size`, `duration`, and `mediaType` are set), or to `"failed"` on error. These post-stop transitions are server-initiated and not client-controllable.

#### Destroying a Recording

`destroy` removes the VTCRecording metadata. If the recording is in state `"recording"` or `"paused"`, the server first stops it and delivers the `VTCRecordingStateEvent` with `state: "stopped"`. If the recording is in state `"available"`, the associated blob is subject to the server's standard blob lifecycle (it is not guaranteed to be deleted immediately).

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

All current and past participants of the associated VTCCall may retrieve VTCLivestream objects. Non-moderators receive `streamKey` as an empty string (see `streamKey` in {{vtc-livestream}}). The server SHOULD return `notFound` for livestream ids belonging to calls the authenticated user has no access to.

### VTCLivestream/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### VTCLivestream/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

Only participants with `role: "moderator"` on the associated VTCCall MAY create, update, or destroy VTCLivestream objects. Non-moderators SHOULD receive `forbidden`.

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
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
      "streamUri": "rtmp://live.example.com/ingest",
      "streamKey": "sk_live_abc123def456",
      "platform": "youtube"
    }
  }
}
~~~

The server sets `id` and `state` (to `"starting"`), then transitions to `"live"` when the stream is active, or `"failed"` on error. The `"starting"` → `"live"` and `"starting"` → `"failed"` transitions are server-initiated.

When the call has `e2eeEnabled: true`, a livestream that would require server-side media access is rejected as {{e2ee-media-participants}} describes. The server MAY accept the create for such a call when the stream is produced by a key-holding livestream bridge that joins the call as a participant ({{e2ee-media-participants}}). The server MUST return `forbidden` when the account-level `supportsLivestream` is `false`. The server MUST return `invalidProperties` naming `callId` when the call is not in state `"active"`.

#### Stopping a Livestream

`update` with `state: "stopped"` ends the stream. The server sets `stoppedAt` to the current time.

~~~json
{
  "update": {
    "01J4XQ7VN0SYWV2TTFHJVW7JJ0": {
      "state": "stopped"
    }
  }
}
~~~

The server MUST return `invalidProperties` naming `state` if the livestream is not in state `"starting"` or `"live"`. Once stopped, the `streamUri`, `streamKey`, and `platform` fields may be updated for a future stream (see {{updating-stream-configuration}}) or by creating a new VTCLivestream object.

#### Updating Stream Configuration {#updating-stream-configuration}

`update` supports patching `streamUri`, `streamKey`, or `platform` only while the livestream is in state `"stopped"` or `"failed"`. The server MUST return `invalidProperties` naming each patched configuration property for configuration updates while the stream is in state `"starting"` or `"live"`.

#### Destroying a Livestream

`destroy` removes the VTCLivestream metadata. If the livestream is in state `"starting"` or `"live"`, the server first stops it, so that no orphaned stream keeps running after the metadata is removed.

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

When a ring call is created, the server constructs a `VTCCallPush` payload and delivers it to all registered push endpoints for each target participant. The full payload is standalone and sufficient to render an incoming-call UI without a follow-up request. A server MAY instead send the content-free form defined in {{ring-push-content-free}}, which omits every field below except `@type`, `accountId`, `callId`, and `callType`.

`@type` (String):
: MUST be `"VTCCallPush"`.

`accountId` (String):
: The call's hosting account ({{hosting-account}}). This is usually not the target's personal account.

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

`channelId` (String|null):
: The associated channel id within the Space, if any.

Example `VTCCallPush` payload:

~~~json
{
  "@type": "VTCCallPush",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "callType": "ring",
  "initiatorId": "user:bob@example.com",
  "initiatorDisplayName": "Bob Martinez",
  "subject": "Quick sync",
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkz",
  "chatId": null,
  "spaceId": null,
  "channelId": null
}
~~~

### Content-Free Form {#ring-push-content-free}

Payloads sent through APNs, FCM, or another platform push service are visible to the platform operator unless the server encrypts them at the application layer. Web Push payloads are encrypted to the client per {{RFC8291}}; other platforms provide no equivalent protection. A full `VTCCallPush` therefore tells the platform operator who is calling whom, with what subject, and in which Chat or Space.

A content-free `VTCCallPush` carries only `@type`, `accountId`, `callId`, and `callType`. Servers SHOULD send the content-free form for calls with `e2eeEnabled: true` when delivering through a platform push service other than Web Push. Servers MAY send it for any call. The urgency and blocked-sender rules below apply to both forms.

A client that receives a `VTCCallPush` without `initiatorId` SHOULD fetch the call details after waking, by calling `VTCCall/get` for `callId` and, if it needs the caller's display name, `VTCParticipant/get` for the call. Where the platform requires the incoming-call UI to be presented before any network request completes (as iOS PushKit does), the client can present it with a generic caller label and update it once the fetch returns.

The follow-up fetch adds at least one network round trip, plus connection setup if the client has no open connection, between wake and a fully populated incoming-call UI. On a slow or cold connection this can push ring presentation past the two-second target in {{design-philosophy}}. Deployments that choose the content-free form trade that latency for not disclosing call metadata to the platform operator.

Example content-free `VTCCallPush` payload:

~~~json
{
  "@type": "VTCCallPush",
  "accountId": "bob-account",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "callType": "ring"
}
~~~

### Urgency {#ring-urgency}

Ring-call push notifications are time-critical: a ring that arrives five seconds late is a missed call. Servers SHOULD use the highest available urgency for ring notifications:

- **Web Push ({{RFC8030}}):** `Urgency: high`, `TTL` SHOULD be 30–60 seconds.
- **APNs:** `apns-push-type: voip` {{APNS-REQUEST}}. VoIP pushes trigger PushKit, which wakes the app and presents the CallKit incoming-call UI. Servers MUST use the VoIP push type, not the standard alert type, for ring notifications on iOS.
- **FCM:** `priority: high` with a data message. Android apps typically use `ConnectionService` or `TelecomManager` to present the native incoming-call UI.

Room-call and scheduled-call notifications (e.g., "A huddle started in #general") use normal urgency and are not ring-priority. Servers SHOULD deliver these as standard `StateChange` notifications or as `ChatMessagePush` payloads ({{JMAP-CHAT-PUSH}}) rather than `VTCCallPush`.

When a room or scheduled call starts in a Chat that has push subscribers, the server SHOULD suppress the VTCCall `StateChange` for users who will receive the event via `ChatMessagePush` or Chat `StateChange` (for `activeCallId` update). Clients receiving multiple notifications for the same call-start event can deduplicate them by `VTCCall.id` within a 5-second window.

### Blocked-Sender Suppression

Before delivering a `VTCCallPush`, the server SHOULD check whether the initiator corresponds to a contact whose `blocked` field is `true` on the target participant's contact list (when {{JMAP-CHAT}} is present) or an equivalent deployment-defined block list. If the initiator is blocked, the server SHOULD silently drop the ring notification. The initiator is not informed.

## WebSocket: VTCRingEvent {#ring-event}

When the target participant has a WebSocket connection with an active VTC subscription that covers `VTCRingEvent` (`VTCStreamEnable` per {{JMAP-VTC-WSS}}, or `ChatStreamEnable` with the `"vtc"` dataType per {{JMAP-CHAT-WSS}}), the server SHOULD deliver a `VTCRingEvent` on that connection as an ephemeral event in addition to the push notification.

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

The same blocked-sender suppression rule applies: the server SHOULD NOT deliver a `VTCRingEvent` if the initiator is blocked.

## VTCCallEndEvent {#call-end-event}

When a ringing call is answered, declined, cancelled, or times out, the server delivers a `VTCCallEndEvent` to all devices that received the ring. This tells ringing devices to stop ringing.

`@type` (String):
: `"VTCCallEndEvent"`.

`callId` (String):
: The id of the call.

`endReason` (String):
: The reason ringing stopped for this device. Values: `"answered_elsewhere"` (the same user answered on a different device), `"declined"`, `"cancelled"` (initiator withdrew the ring), `"timeout"`, `"failed"`. Note: this is a per-device ring-disposition enum, distinct from VTCCall.endReason which records the call-level outcome.

Example — call answered on another device:

~~~json
{
  "@type": "VTCCallEndEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
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
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "participantId": "01J4XS5WR2VAYY4WWHKNYZ9MN0",
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
    "01J4XT6XS3WBZZ5XXJMPZ0ANP0": {
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

The following ephemeral WebSocket events carry real-time in-call state changes to connected clients. They are delivered over the same WebSocket connection as VTCRingEvent and VTCCallEndEvent. Clients subscribe to these events via `VTCStreamEnable` ({{JMAP-VTC-WSS}}), or by including `"vtc"` in the `dataTypes` array of a `ChatStreamEnable` message ({{JMAP-CHAT-WSS}}), which requires the server to advertise both `urn:ietf:params:jmap:chat:websocket` and `urn:ietf:params:jmap:vtc:websocket`. These events are not persisted; they supplement JMAP `StateChange` notifications for VTCParticipant to provide low-latency UI updates.

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

Delivered when the active (dominant) speaker changes. The server or media layer determines the active speaker based on audio levels. For calls with `e2eeEnabled: true`, the server SHOULD NOT deliver this event and clients determine the active speaker locally ({{e2ee-active-speaker}}).

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

`participantId` (String):
: The VTCParticipant id of the target participant being asked to unmute.

`requestedBy` (String):
: The userId of the moderator making the request.

### VTCRecordingStateEvent {#vtc-recording-state-event}

Delivered to all participants when recording state changes. This is the consent signal ({{recording-consent}}).

`@type` (String):
: `"VTCRecordingStateEvent"`.

`callId` (String):
: The VTCCall id.

`recordingId` (String, optional):
: The VTCRecording id. Absent when the event reports a livestream bridge.

`livestreamId` (String, optional):
: The VTCLivestream id. Present only when the event reports a key-holding livestream bridge ({{e2ee-media-participants}}). Exactly one of `recordingId` and `livestreamId` is present.

`state` (String):
: The new recording state (`"recording"`, `"paused"`, `"stopped"`).

`initiatedBy` (String):
: The userId of the participant who initiated the state change.

## Multi-Device Forking {#multi-device-forking}

Ring notifications are delivered to ALL of the target participant's registered push subscriptions and to each WebSocket connection subscribed to `VTCRingEvent` ({{ring-event}}). The first device to answer wins: the server accepts the first `VTCParticipant/set` update (setting `joinMethod`), then dispatches `VTCCallEndEvent` with `endReason: "answered_elsewhere"` to all other devices of that participant.

When a call targets multiple participants, the first answer by any target transitions the call to `"active"`. The other targets keep ringing: each may still answer (and join) or decline until the ring timeout expires.

## Ring Timeout

Servers SHOULD enforce a ring timeout. If no target participant answers within a deployment-defined period (RECOMMENDED: 30 seconds), the server transitions the call to `"ended"` with `endReason: "missed"` and dispatches `VTCCallEndEvent` with `endReason: "timeout"` to all ringing devices. If the call is already `"active"` when the period expires, the call continues; the server dispatches `VTCCallEndEvent` with `endReason: "timeout"` to the devices of targets whose `callResponse` is still `"pending"`.

# End-to-End Encrypted Calls {#e2ee-calls}

This section applies to calls with `e2eeEnabled: true`.

## Key-Holding Media Participants {#e2ee-media-participants}

A recorder, livestream bridge, or protocol gateway needs plaintext media. In an E2EE call it MAY obtain plaintext only by joining the call as a disclosed key-holding participant:

- It SHOULD have a VTCParticipant record in the call for as long as it receives media, and the server SHOULD include that record in the participant list and announce its join and departure with `VTCParticipantEvent` like any other participant.
- It SHOULD obtain media keys only through the media layer's key agreement, on the same terms as other participants. Its `e2eeFingerprint` is populated like any other participant's.
- Its `displayName` SHOULD state its function (for example, "Recorder" or "Livestream"). Gateway participants carry the `joinMethod` of their gateway protocol.
- When a key-holding recorder or livestream bridge starts, pauses, resumes, or stops capturing media, the server SHOULD deliver `VTCRecordingStateEvent` ({{vtc-recording-state-event}}) to all participants. For a livestream bridge, the event carries `livestreamId` instead of `recordingId`. This is the same consent signal described in {{recording-consent}}.

The server SHOULD NOT use any other path to plaintext media in an E2EE call, such as server-side media taps, SFU-side decryption, or delivery of media keys to the server or to any party that does not appear in the participant list. The server SHOULD reject with `forbidden` a recording or livestream request that can only be met by such a path.

Clients can visually distinguish key-holding recorders, bridges, and gateways from human participants, so that users can see who outside the conversation holds plaintext.

## Active Speaker and Speaker Time {#e2ee-active-speaker}

The participant list, `speakerTimeMs`, and the active speaker are visible to the server when the server populates them. Speaking patterns can reveal content (who answered whom, and for how long), so E2EE calls limit what the server learns:

- The server SHOULD NOT map media-layer streams (for example, SFU stream or RTP SSRC identifiers) to VTCParticipant ids.
- A server that does not map streams cannot attribute audio, so it SHOULD NOT deliver `VTCActiveSpeakerEvent` for the call, and it sets `speakerTimeMs` to `0` as that field's definition requires.
- Clients MAY determine the active speaker locally from decrypted audio levels, using the stream-to-participant mapping learned inside the E2EE layer. Clients SHOULD prefer local detection over `VTCActiveSpeakerEvent` when both are available.

## Participant Visibility {#e2ee-participant-visibility}

The server SHOULD NOT expose VTCParticipant records or VTCParticipant state changes for the call to users who do not have a VTCParticipant record in it, even where {{access-control}} would otherwise make the call visible to them (for example, members of a bound Chat). For such users the server SHOULD return `notFound` for `VTCParticipant/get` and exclude the records from `VTCParticipant/query`. `VTCParticipant/changes` keeps reporting changes for the account as a whole, as {{RFC8620}} defines; it reveals only ids, whose contents stay hidden behind `notFound`. This matches the rule in {{JMAP-SCENE}} that suppresses SceneAvatarEvent delivery for a region bound to an E2EE call to subscribers who are not participants. A server that does not implement this suppression exposes the call's participant list to those users.

# Access Control {#access-control}

## Hosting Account {#hosting-account}

A VTCCall, together with the VTCParticipant, VTCRecording, and VTCLivestream records that belong to it, lives in one JMAP account: the account in which the call was created, called the call's hosting account. A breakout room lives in the hosting account of its parent call.

Other users reach these records by accessing the hosting account as a shared account ({{RFC8620}} Section 1.6.2): the account appears in the user's Session with `isPersonal` set to `false`. The server gives a user access to the hosting account when the user is a ring target of a call in it or may join a call in it. Access to the account does not by itself grant access to every call in it; the per-call rules in {{chat-bound-calls}} and {{standalone-calls}} apply to each call.

The `accountId` of every method call on a VTC data type, of `VTCCallPush` ({{vtc-call-push}}), and of every WebSocket event ({{JMAP-VTC-WSS}}) that concerns a call is the id of that call's hosting account. A participant who answers, mutes, or leaves therefore sends `VTCParticipant/set` to the hosting account, not to their personal account. A user's calls can be spread across several hosting accounts; a client that lists them queries each account the user can access.

## Chat-Bound Calls {#chat-bound-calls}

When a VTCCall carries a `chatId`, `spaceId`, or `channelId` binding and `urn:ietf:params:jmap:chat` is present, access control is inherited from JMAP Chat's permission model:

- **Visibility:** A call is visible to members of the bound Chat or Space channel. Non-members SHOULD receive `notFound` for `VTCCall/get` and `VTCParticipant/get` requests on the call. For calls with `e2eeEnabled: true`, participant records are further restricted per {{e2ee-participant-visibility}}.
- **Join permission:** In a Space-bound call, participants who can `"view"` the channel MAY join an existing call even without the `"start_call"` permission (see {{JMAP-CHAT}}); `"start_call"` gates call creation, not joining.
- **Recording and livestream access:** VTCRecording and VTCLivestream objects inherit visibility from their parent VTCCall.
- **Space ban enforcement:** When a user is banned from a Space (via SpaceBan creation) and has an active VTCParticipant record in a call bound to that Space (via `spaceId`), the server SHOULD set `leftAt` on that VTCParticipant record and emit a `VTCParticipantEvent` with event `"left"`. The ban takes effect immediately on active calls.

## Standalone Calls {#standalone-calls}

When no Chat binding is present (the VTC capability is used standalone), the server SHOULD enforce the following access rules:

- Participants (current or past) may access VTCCall and VTCParticipant records for calls in which they have a VTCParticipant entry.
- The call initiator may access VTCCall records they created.
- Non-participants receive `notFound` for calls they cannot access.
- `VTCCall/query` returns only calls the authenticated user has access to ({{vtc-call-query}}).
- Deployment-defined administrative roles MAY have broader access.

# Federation {#federation}

Cross-server VTCCall state synchronization is out of scope for this document.

When used with JMAP Chat federation ({{JMAP-CHAT-FED}}), a call invitation MAY be delivered to remote participants via `Peer/deliver` carrying a MessageAction of type `"urn:jmap:chat:cap:vtc"` with the `joinUri` — using the existing out-of-band action mechanism defined in {{JMAP-CHAT}}. The receiving server MAY auto-create a local VTCCall record from the inbound MessageAction to provide call-state tracking on the remote side.

Federated call state synchronization — maintaining a consistent VTCCall object across multiple servers, with synchronized participant lists and state transitions — is a significantly more complex problem and is deferred to a future companion specification.

# Security Considerations {#security}

## joinUri Is Untrusted {#joinuri-untrusted}

`joinUri` is opaque to the JMAP server and, for a call created from a federated invitation ({{federation}}), supplied by the remote peer. Clients SHOULD NOT connect to `joinUri` without explicit user initiation. Auto-joining a call is a privacy violation: it activates the microphone and potentially the camera without consent.

Servers SHOULD NOT fetch, probe, or validate `joinUri` values. Doing so exposes the server to SSRF attacks.

## Ring as Denial of Service {#ring-dos}

Ring calls cause device-level interruption: vibration, sound, and a full-screen incoming-call UI. Without rate limiting, an attacker could use repeated ring calls to harass a target.

Servers SHOULD enforce:

- Per-caller ring rate limits (e.g., no more than 3 ring calls to the same target within 60 seconds).
- Per-callee ring rate limits (e.g., no more than 10 incoming rings per minute from all callers combined).
- Blocked-sender suppression of rings ({{ring-notification}}).

Servers MAY temporarily suspend ring delivery for a target that is receiving rings at an abusive rate and fall back to silent `StateChange` notifications.

## Recording Consent {#recording-consent}

The server SHOULD notify all participants when recording state changes (started, paused, resumed, stopped), using `VTCRecordingStateEvent` ({{vtc-recording-state-event}}). This notification is the consent signal: it gives every participant notice that recording state changed. Whether a recording needs one-party or all-party consent is a matter of law, not of this protocol; deployments subject to all-party consent laws rely on this signal.

A participant can leave a call at any time by setting their own `leftAt` (see Leaving a Call in {{vtc-participant-methods}}), which serves as an implicit decline of consent when recording starts.

## Call Metadata Exposure

Even without access to the media stream, the JMAP server observes: who called whom, when, for how long, from which device, and who else was on the call. This metadata is privacy-sensitive. Deployments requiring metadata privacy SHOULD apply the same mitigations noted in the E2EE section of {{JMAP-CHAT}}: message padding, cover traffic, and other transport-layer techniques outside the scope of this document.

`e2eeEnabled: true` protects media content, not call metadata. The server still knows the participant list, join and leave times, and `activeParticipantCount`. If it maps media streams to participants it also learns `speakerTimeMs` and the active speaker, which can reveal the shape of a conversation. {{e2ee-active-speaker}} and {{e2ee-participant-visibility}} limit this exposure. A full `VTCCallPush` also discloses call metadata to the push platform operator; see {{ring-push-content-free}}. A key-holding recorder, bridge, or gateway ({{e2ee-media-participants}}) holds plaintext media by design; its presence in the participant list is what keeps the E2EE claim honest.

## Media State Accuracy {#media-state-accuracy}

VTCMediaState is client-reported and server-relayed. The server has no media-layer visibility and cannot verify that `audio: false` actually means the microphone is off. A malicious client could report `audio: false` while transmitting audio, or `video: false` while the camera is active.

This is inherent to the media-agnostic design. Clients SHOULD NOT rely on VTCMediaState for security-critical decisions. The media layer is the only authority on what media is actually being transmitted.

## Moderator Privilege Escalation

Each moderation action carries its own role check: call-level actions in {{moderator-actions}} and {{ending-a-call}}, participant-level actions in {{participant-moderator-actions}}, and recording and livestream control in `VTCRecording/set` and `VTCLivestream/set`. A participant who could pass these checks without the moderator role could end calls, kick or mute others, admit lobby participants, or start recording or broadcasting.

## Lobby Bypass

When `lobbyEnabled` is `true`, the server SHOULD NOT allow participants to skip the lobby via direct `VTCParticipant/set` manipulation. The lobby admission path — `lobbyState` transitioning from `"waiting"` to `"admitted"` — SHOULD require a moderator-initiated `VTCParticipant/set` update.

## Livestream Key Exposure {#livestream-key-exposure}

`streamKey` is an authentication credential for an external streaming platform, which is why non-moderators receive it as an empty string (see `streamKey` in {{vtc-livestream}}). The `streamUri` is also sensitive (it often contains account-identifying path components); the server SHOULD return `streamUri` as an empty string for non-moderator participants, as for `streamKey`.

## Gateway Participant Identity

`displayName` and `gatewayData` for participants who joined via a protocol gateway (`joinMethod` of `"pstn"`, `"sip"`, or `"h323"`) are derived from the external protocol's identity mechanisms (PSTN caller ID, SIP From header, H.323 alias). These identities are trivially spoofable on their respective networks and are not authenticated by the JMAP server. Clients can visually distinguish gateway participants from authenticated JMAP users to limit impersonation.

## Gateway Signal Injection

VTCGatewaySignal events ({{gateway-signal}}) carry opaque protocol-specific data. The server SHOULD NOT interpret or execute gateway signal payloads; it relays them between the gateway infrastructure and the WebSocket connection. Outbound gateway signals are sent through a `VTCParticipant/set` update on another participant's record, which only moderators may make ({{participant-moderator-actions}}). The gateway infrastructure is responsible for validating that relayed signals are well-formed for its protocol.

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
- **Google Meet** informed the lobby/waiting room and live-captions patterns. Live captions are delivered as chat messages rather than a VTC-native type, following the simplification principle.
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
- **Transcription engine.** Live captions and transcription are delivered as chat messages via {{JMAP-CHAT}}, not as VTC-native types. The transcription engine (Whisper, Vosk, cloud ASR, or other) is entirely deployment-defined.
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
      "id": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
      "state": "ringing",
      "initiatorId": "user:bob@example.com",
      "createdAt": "2026-06-05T14:30:00Z",
      "joinUri": "https://meet.example.com/room/xkz",
      "activeParticipantCount": 1
    }
  }
}, "0"]]
~~~

The server creates two VTCParticipant records (Bob, joined at creation, and Alice, with `joinedAt: null` and `callResponse: "pending"`), delivers a `VTCCallPush` via APNs VoIP push to Alice's iOS device, and delivers a `VTCRingEvent` to Alice's active WebSocket connection.

### Step 2: Alice's devices receive the ring

Alice's iOS device receives a VoIP push with the `VTCCallPush` payload and presents the CallKit incoming-call UI. Simultaneously, Alice's desktop client receives via WebSocket:

~~~json
{
  "@type": "VTCRingEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "initiatorId": "user:bob@example.com",
  "mediaTypes": ["audio", "video"],
  "joinUri": "https://meet.example.com/room/xkz"
}
~~~

### Step 3: Alice answers on her desktop

The call lives in Bob's account, which Alice's Session lists as a shared account ({{hosting-account}}). Alice's desktop client finds her VTCParticipant record:

~~~json
[["VTCParticipant/query", {
  "accountId": "bob-account",
  "filter": {
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
    "userId": "user:alice@example.com"
  }
}, "1a"]]
~~~

The response's `ids` is `["01J4XMVQK7NRYS9PPCEHTK4CD0"]`. The client then sends `VTCParticipant/set`:

~~~json
[["VTCParticipant/set", {
  "accountId": "bob-account",
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "joinMethod": "webrtc"
    }
  }
}, "1"]]
~~~

The server sets `joinedAt`, transitions the VTCCall to `"active"`, and delivers a `VTCCallEndEvent` to Alice's iOS device:

~~~json
{
  "@type": "VTCCallEndEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
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
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0"
    }
  }
}, "2"]]
~~~

Both participants receive a `VTCRecordingStateEvent` via WebSocket:

~~~json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "recordingId": "01J4XP2SN9RXVV1SSEGJVT6GH0",
  "state": "recording",
  "initiatedBy": "user:bob@example.com"
}
~~~

Alice's client displays a recording indicator.

### Step 5: Alice mutes

Alice's client sends `VTCParticipant/set`:

~~~json
[["VTCParticipant/set", {
  "accountId": "bob-account",
  "update": {
    "01J4XMVQK7NRYS9PPCEHTK4CD0": {
      "mediaState/audio": false
    }
  }
}, "3"]]
~~~

Bob receives via WebSocket:

~~~json
{
  "@type": "VTCMediaStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB0",
  "participantId": "01J4XMVQK7NRYS9PPCEHTK4CD0",
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
    "01J4XKZQN4MWVT8PPBEHTJ3AB0": {
      "state": "ended"
    }
  }
}, "4"]]
~~~

The server sets `endedAt` and `endReason: "completed"`, stops the recording (transitioning it through `"stopped"` → `"processing"` → `"available"`), and delivers a `StateChange` for the hosting account to Alice's connection:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "bob-account": {
      "VTCCall": "s201",
      "VTCParticipant": "s202",
      "VTCRecording": "s203"
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

The call is immediately `"active"`. Because it is bound by `spaceId` and `channelId` rather than `chatId`, no Chat's `activeCallId` changes; a call created with a `chatId` would also set that Chat's `activeCallId` ({{JMAP-CHAT}}).

### Step 2: Guest joins and enters lobby

A channel member joins. The call lives in the moderator's account, which the member reaches as a shared account ({{hosting-account}}):

~~~json
[["VTCParticipant/set", {
  "accountId": "mod-account",
  "create": {
    "p0": {
      "callId": "01J4XL8QN50WXV9RRBEJVS5CD0",
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
    "01J4XV7YT4XCA06YYKNQ01BQR0": {
      "lobbyState": "admitted"
    }
  }
}, "2"]]
~~~

The guest's `lobbyState` transitions to `"admitted"` and they can now see and hear other participants. The guest receives a `StateChange` for their VTCParticipant record.

### Step 4: Guest leaves

~~~json
[["VTCParticipant/set", {
  "accountId": "mod-account",
  "update": {
    "01J4XV7YT4XCA06YYKNQ01BQR0": {
      "leftAt": "2026-06-05T15:45:00Z"
    }
  }
}, "3"]]
~~~

The moderator receives a `VTCParticipantEvent`:

~~~json
{
  "@type": "VTCParticipantEvent",
  "callId": "01J4XL8QN50WXV9RRBEJVS5CD0",
  "participantId": "01J4XV7YT4XCA06YYKNQ01BQR0",
  "event": "left",
  "displayName": "Guest User",
  "role": "participant"
}
~~~

# Acknowledgements

The author thanks the Jitsi Meet project for the comprehensive open-source reference implementation that informed this specification's feature set, the authors of {{JMAP-CHAT}} for the chat protocol whose design patterns this specification follows, and the authors of SIP {{RFC3261}} for the session-state model that underpins the VTCCall lifecycle.
