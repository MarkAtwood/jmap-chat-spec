# JMAP VTC — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-vtc-00`. Covers the
media stack, lifecycle, policy, and operational decisions that the spec
deliberately leaves to implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what the spec *deliberately leaves open* and what implementations must
decide before shipping.

---

## How to read this guide

The JMAP VTC draft is intentionally silent about media transport, SFU
architecture, E2EE key exchange, recording backends, and many other choices
that a real calling product must make. The draft defines call state as JMAP
data types, the lifecycle state machines, the ring notification payload, and
the in-call ephemeral event set — and stops there. Everything between the
`joinUri` and the actual audio or video reaching a participant's speaker is
deliberately out of scope.

This is not a free pass. An implementation that ignores a deferred decision
will deliver a broken or surprising product: calls that fail behind NAT,
recording files that are never retrievable, E2EE indicators that mean nothing,
or a lobby that can be bypassed with a direct `VTCParticipant/set` create.
Implementations must make each of these decisions explicitly, document them,
and implement them coherently.

Each section below follows the same shape:

1. **What the spec leaves open** — with a draft section citation, so you can
   read the normative text yourself.
2. **What you must decide** — the concrete deployment choice you cannot avoid.
3. **Considerations** — the trade-offs that inform the choice.
4. **Common patterns** — how production calling systems handle this.
5. **Recommended starting point** — a defensible default. Not normative; you
   may choose otherwise with good reason.

When two sections interact (for example, E2EE decisions constrain what the
recording section can do), cross-references are explicit.

This guide is non-normative. `draft-atwood-jmap-vtc-00` is the source of
truth. Where this guide and the draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- `draft-atwood-jmap-vtc-00` is the normative source of truth. Where this
  guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice or deployment
  choice (e.g., "servers SHOULD rate-limit dial-out attempts"), the keyword
  reflects implementer best-practice. Deviation does not affect protocol
  interoperability.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. Media stack selection

### What the spec leaves open

The spec is explicit that it "does not prescribe the media transport"
(§Introduction, Design Philosophy). The `joinUri` field on VTCCall is opaque
to the JMAP server: it is a deployment-supplied URI pointing to wherever media
is actually handled. The spec does not define how `joinUri` is constructed, how
SDP offer/answer is exchanged, how ICE candidates are gathered, or what codec
set is used (§VTCCall Object Fields, `joinUri`). The account-level capability
object advertises `supportedMediaTypes` (`"audio"`, `"video"`, `"screen"`), but
the media protocol used to deliver those types is not part of the JMAP surface.

### What you must decide

- **Which media protocol** your deployment uses: WebRTC, SIP/RTP, H.323, a
  proprietary protocol, or a combination.
- **SFU vs. MCU vs. mesh**: for multi-party calls, whether media is routed
  through a Selective Forwarding Unit, a Multipoint Control Unit, or
  peer-to-peer among participants.
- **How `joinUri` is generated**: what URI scheme, what authority, and what
  room-identifying parameters are encoded in it.
- **STUN/TURN deployment**: how NAT traversal is handled for WebRTC
  participants.
- **Gateway interoperability**: whether PSTN, SIP, or H.323 gateways are
  needed, and how they connect to the media layer.

### Considerations

**WebRTC** is the natural choice for browser and mobile clients. It provides
end-to-end encryption by default (DTLS-SRTP), broad client library support
(libwebrtc on iOS/Android, native in every major browser), and an established
ecosystem of SFU implementations. Its signaling (SDP offer/answer, ICE
candidate exchange) is the deployment's responsibility; JMAP VTC does not relay
SDP or ICE candidates.

**SIP/RTP** is the right choice when interoperability with PSTN gateways,
enterprise PBX systems, or existing SIP infrastructure is required. The
VTCGateway object (§VTCGateway) and the `"sip"` joinMethod exist precisely to
support this path. A SIP-based deployment may use a media server such as
FreeSWITCH, Asterisk, or Kamailio; the JMAP layer sits atop the SIP signaling
layer and tracks state as JMAP objects.

**SFU vs. MCU vs. mesh:**

- An SFU (Selective Forwarding Unit) forwards each participant's media stream
  to all others, with the server performing no transcoding. This scales to
  dozens of participants and preserves individual stream quality, but every
  participant downloads N-1 streams. Mediasoup, Janus, Jitsi Videobridge, and
  LiveKit are SFU implementations.
- An MCU (Multipoint Control Unit) mixes all streams into a single composite
  stream that every participant downloads. This reduces client-side bandwidth at
  the cost of server-side compute and introduces a quality floor from
  transcoding. Suitable for very large calls or when participants have limited
  bandwidth.
- A mesh (peer-to-peer) has no media server: every participant connects
  directly to every other participant. This works well for calls with two to
  four participants and carries zero server-side media cost. It does not scale
  beyond roughly six participants, and NAT traversal failures are more common
  without a TURN relay.

**How `joinUri` connects to the media layer.** The `joinUri` is the
deployment's signaling entry point. For a Jitsi-based deployment it might be
`https://meet.example.com/roomname`. For a mediasoup deployment it might be
`wss://sfu.example.com/room/ULID`. For a SIP deployment it might be
`sip:conference@example.com`. The JMAP server generates this URI at call
creation time (§VTCCall/set, Creating a Ring/Room/Scheduled Call) and stores
it as an opaque field; the client is responsible for connecting to it using the
appropriate media protocol.

**STUN/TURN.** For WebRTC deployments, STUN lets clients discover their public
address. TURN is required for clients behind symmetric NAT or restrictive
corporate firewalls. Plan for TURN from the start: a non-trivial fraction of
enterprise users will fail peer-to-peer connectivity without it. Coturn is the
standard open-source TURN server. TURN credentials are typically short-lived
and HMAC-based; they are generated by the media layer and returned to the
client out-of-band (not via JMAP VTC).

**Security note.** The spec (§joinUri Is Untrusted) requires that clients MUST
NOT connect to `joinUri` without explicit user initiation, and that servers
MUST NOT fetch or probe `joinUri` values. Enforce both: auto-joining is a
privacy violation, and server-side probing exposes the server to SSRF.

### Common patterns

| System | Media stack | Architecture |
|---|---|---|
| Jitsi Meet | WebRTC | SFU (Jitsi Videobridge) |
| Zoom | Proprietary / WebRTC | SFU with MCU fallback for large meetings |
| Google Meet | WebRTC | SFU with transcoding for low-bandwidth paths |
| Slack Huddles | WebRTC | SFU (Slack-operated) |
| Discord | WebRTC | SFU (Discord-operated) |
| Enterprise PBX | SIP/RTP | MCU (conference bridge) |

### Recommended starting point

For a new deployment: WebRTC with an SFU, a TURN server on port 443 (to
traverse restrictive firewalls), and mediasoup or LiveKit as the SFU
implementation. Generate `joinUri` as a stable room URL with the VTCCall ULID
as the room identifier. This combination handles 2-to-50 participant calls
without MCU complexity, gives you E2EE options via Insertable Streams, and
maps naturally to the browser and mobile WebRTC APIs.

For enterprise deployments requiring PSTN interoperability: deploy a SIP trunk
gateway alongside the WebRTC SFU. Ensure the account-level `supportedMediaTypes`
includes `"audio"` and `"video"`. Populate the `gateways` array with your PSTN
and SIP gateway configurations. See the *JMAP VTC Gateway Integration Guide* for
complete gateway lifecycle, signal vocabulary, and security guidance.

---

## 2. VTCParticipant lifecycle

### What the spec leaves open

The spec defines the VTCParticipant object fields, the creation rules for each
call type, and the reconnect behavior (§Reconnecting), but leaves open how
the server should handle edge cases: participants who go silent on the media
layer without calling `VTCParticipant/set` to set `leftAt`, how long to retain
participant records for ended calls, and the precise timing of the
`lobbyState` admission flow relative to media connectivity.

### What you must decide

- **Participant presence detection**: how the server detects that a participant
  has disconnected at the media layer without having sent a `leftAt` update.
- **Ring call vs. room/scheduled call participant creation**: when and how to
  create VTCParticipant records for each call type.
- **Reconnect identity window**: how long after a participant sets `leftAt` the
  server will treat a re-join as a reconnect (updating the existing record)
  rather than a fresh join (creating a new record).
- **Lobby flow timing**: at what point in the media-layer connection sequence
  to place a participant in lobby vs. admitting them to the call.

### Considerations

**Ring calls: pre-created participant records.** When a ring call is created,
the server immediately creates VTCParticipant records for all target
participants (with `joinedAt: null`, `callResponse: "pending"`) and for the
initiator (§Creating a Ring Call). The initiator's record should have
`joinedAt` set when the call transitions to `"active"`. This design lets
clients display who is being called before anyone answers.

**Room and scheduled calls: participant creation on join.** For room and
scheduled calls, VTCParticipant records are created by a `VTCParticipant/set`
`create` operation at the moment the participant joins (§Joining a Room or
Scheduled Call). There are no pre-created records for expected attendees (the
spec has no concept of a room call invitation list at the VTC layer; if you
need invited-attendee tracking for scheduled calls, use the `calendarEventId`
binding and JMAP Calendars, or manage attendee lists in JMAP Chat via the
`chatId` binding).

**Answering flow.** A ring call target answers by updating their existing
VTCParticipant record: they set `joinMethod` via `VTCParticipant/set update`
(§Answering a Ring Call). The server then sets `joinedAt`. This is different
from room/scheduled join, which uses `create`. Keep these paths distinct in
your server implementation.

**Media-layer presence detection.** The spec tracks participant state via
explicit `leftAt` updates, but real participants crash, lose connectivity, or
close their browser without sending a leave signal. Your server SHOULD
implement a heartbeat or media-layer callback to detect silent disconnections.
When the media layer signals that a participant has dropped:

1. Update the VTCParticipant record: set `leftAt` to the current time.
2. If the participant was the last active participant, transition the VTCCall
   to `"ended"` with `endReason: "completed"`.
3. Deliver a `VTCParticipantEvent` with `event: "left"` on the WebSocket.

The heartbeat interval is deployment-defined. A 10–30 second detection window
is typical; shorter windows increase server load, longer windows leave "ghost"
participants visible in the UI.

**Reconnect identity.** The spec requires that a re-joining participant who
already has a VTCParticipant record in the call MUST be reconnected to that
record rather than assigned a new one (§Reconnecting). This preserves role and
speaker time across brief network interruptions. Define your reconnect window:
if a participant's `leftAt` was set within the last N seconds and they re-join
with the same `userId`, treat it as a reconnect. A 5-minute window is
reasonable; beyond that, create a new record. (The spec does not prescribe this
threshold; choose one and document it.)

**Lobby flow.** When `lobbyEnabled` is `true`, the server sets a new
participant's `lobbyState` to `"waiting"` at creation time. The participant
MUST NOT be admitted to the media session until a moderator sets `lobbyState`
to `"admitted"` (§Lobby Bypass, §Admit a lobby participant). The media layer
must enforce this: a waiting participant should be connected to a lobby media
context (or no media context at all) until admitted. How this is implemented
depends on your media stack — Jitsi Meet supports lobby natively; with a raw
SFU you will need to route lobby participants to a separate media room and
move them on admission.

**Moderator role assignment.** The spec says the call initiator receives
`"moderator"` by default and the server MAY grant moderator status to
additional participants based on deployment policy (§VTCParticipant Object
Fields, `role`). Define your policy: for ring calls, should the callee also
be a moderator? For Space-bound calls, should Space admins automatically
receive moderator role when joining?

### Common patterns

| Pattern | When to use |
|---|---|
| Media-layer webhook sets `leftAt` | SFU emits a "participant left" event; server updates the record immediately. Most reliable. |
| Heartbeat via WebSocket | Server pings active WebSocket connections; no pong within N seconds triggers `leftAt`. Works when media layer has no callback API. |
| Explicit leave signal only | Clients are trusted to send `leftAt`. Simple but leaves ghost participants on crash. Acceptable only for demos. |

### Recommended starting point

For ring calls: create all target VTCParticipant records synchronously when the
call is created. Set the initiator's `joinedAt` when the call moves to
`"active"`. For room and scheduled calls: create records on join. Implement a
media-layer callback for leave detection; fall back to a 30-second heartbeat
if your SFU does not support participant-left webhooks. Set a 5-minute
reconnect identity window. For the first version, grant moderator role only to
the call initiator; extend to Space admins in a follow-up once the Space
permission model is integrated.

---

## 3. VTCCallPolicy enforcement

### What the spec leaves open

The spec defines the VTCCallPolicy fields and their semantics (§VTCCallPolicy):
`muteOnEntry`, `videoOffOnEntry`, `participantsCanUnmute`,
`participantsCanShareScreen`, `participantsCanStartVideo`. It specifies that
non-moderators MUST receive `forbidden` when trying to violate these policies
via `VTCParticipant/set`, and that the server must apply entry policies at join
time. What the spec leaves open is how policy changes mid-call are propagated
to the media layer, and what happens to participants who are already in the
call when policy changes.

### What you must decide

- **Entry policy application**: when exactly to apply `muteOnEntry` and
  `videoOffOnEntry` — at VTCParticipant record creation, or when the media
  layer signals that the participant is connected.
- **Mid-call policy change propagation**: when a moderator changes a policy
  mid-call, which participants are affected retroactively (if any) and how the
  media layer is notified.
- **Media layer enforcement vs. JMAP enforcement**: whether the media layer
  independently enforces policies or whether JMAP state alone controls what
  participants can do.

### Considerations

**Entry policies.** `muteOnEntry: true` means the server MUST set the new
participant's `mediaState.audio` to `false` at join time; the client joins
muted. `videoOffOnEntry: true` similarly forces `mediaState.video: false`.
These are enforced at the JMAP level: the VTCParticipant record is created with
those defaults regardless of what the client requested. The media layer SHOULD
also enforce this independently — a participant whose JMAP record shows
`audio: false` should not have their microphone active in the SFU — but the
JMAP record is the authoritative state signal.

**Mid-call policy changes.** When a moderator patches `policy.muteOnEntry`
or `policy.videoOffOnEntry`, this affects future joiners only; existing
participants are not forcibly muted or video-disabled by the policy change
alone. The spec does not require retroactive enforcement. If you want a
"mute all" function (not just "mute on entry"), that is a moderator action
on each individual participant via `VTCParticipant/set` update on their
`mediaState/audio`.

**`participantsCanUnmute: false`.** This is the "all-muted webinar" mode.
The server must reject any `VTCParticipant/set` update where a non-moderator
tries to set their own `mediaState/audio: true`. The moderator can unblock
a specific participant using the ask-to-unmute flow (§Ask to unmute): set
`unmuteRequested: true` on the target, which delivers a
`VTCUnmuteRequestEvent` to that participant. The participant may then
choose to unmute, which clears `unmuteRequested`. This is explicitly a soft
request — the server MUST NOT force-unmute a participant. If you want to
grant unmute permission to a specific participant in a locked-down call,
the moderator must either patch `policy.participantsCanUnmute` back to `true`
(affecting everyone) or be willing to moderate individual unmute requests.

**`participantsCanShareScreen: false` and `participantsCanStartVideo: false`.**
These follow the same pattern: the server rejects attempts by non-moderators
to set the corresponding `mediaState` field to `true`. The media layer SHOULD
independently gate screen-share and camera activation to match, but the JMAP
rejection is the primary enforcement point.

**Media layer independence.** The spec acknowledges that VTCMediaState is
client-reported and server-relayed: the server has no media-layer visibility
and cannot verify that `audio: false` means the microphone is physically off
(§Media State Accuracy). A deeply adversarial client could lie. For
most deployments this is acceptable — the trust model is that authenticated
JMAP users are not adversarial about their own media state. For high-security
deployments, the SFU should independently enforce mute state by blocking the
participant's audio stream at the media layer, regardless of what JMAP says.

**Policy changes mid-call: notification.** When any policy field changes, the
server SHOULD deliver a `StateChange` notification for the VTCCall object to
all connected participants so their UIs update immediately. The WebSocket
`VTCMediaStateEvent` is not appropriate for policy changes (it covers
individual participant state); a VTCCall `StateChange` is the right mechanism.

### Common patterns

| Pattern | Typical use case |
|---|---|
| All defaults (`true`/`false`) | Informal drop-in call; no restrictions. |
| `muteOnEntry: true`, `participantsCanUnmute: true` | Large meeting starts quiet; participants self-unmute. |
| `muteOnEntry: true`, `participantsCanUnmute: false` | Webinar mode; only moderator unmutes speakers. |
| `videoOffOnEntry: true` | Low-bandwidth context; video off by default. |
| All `Can*` fields `false` | Broadcast/presentation mode; only moderators have media. |

### Recommended starting point

Apply `muteOnEntry` and `videoOffOnEntry` at VTCParticipant creation time as
JMAP field defaults. Enforce `participantsCanUnmute`, `participantsCanShareScreen`,
and `participantsCanStartVideo` server-side in the `VTCParticipant/set`
handler: check policy before processing any `mediaState` field update from a
non-moderator. Do not apply entry policies retroactively when they change
mid-call; add a "mute all" convenience action at the application layer if
needed. Make the SFU honor the JMAP media state as a secondary enforcement
layer, but treat the JMAP rejection as the primary gate.

---

## 4. E2EE deployment

### What the spec leaves open

The spec defines two fields: `e2eeEnabled` on VTCCall (Boolean, default
`false`) and `e2eeFingerprint` on VTCParticipant (the participant's public key
fingerprint). It specifies that features requiring server-side media access
(recording, livestreaming, gateway participants) are typically unavailable when
`e2eeEnabled` is `true`, and that the server SHOULD reject VTCRecording and
VTCLivestream creates with `forbidden` in that case (§VTCCall Object Fields,
`e2eeEnabled`). The key exchange mechanism, the encryption protocol, and the
fingerprint format are explicitly out of scope for the spec.

### What you must decide

- **Whether to support E2EE at all** in the first version.
- **Which E2EE mechanism** to use at the media layer: WebRTC Insertable
  Streams / Encoded Transform, SFrame (RFC 9605), or a proprietary approach.
- **Key agreement protocol**: how participants derive a shared key — Static
  Diffie-Hellman, MLS (RFC 9420), Signal Double Ratchet, or a simpler
  symmetric key distribution by the server.
- **Fingerprint format and display**: what the `e2eeFingerprint` string
  contains, and how clients present it for out-of-band verification.
- **Trust model**: trust-on-first-use (TOFU), PKI with a CA, or manual
  verification.

### Considerations

**WebRTC Insertable Streams / Encoded Transform** is the primary mechanism for
E2EE in a WebRTC + SFU deployment. The SFU forwards encrypted frames without
decrypting them; clients encrypt before sending and decrypt after receiving.
The server's JMAP layer never sees plaintext media, but the SFU still sees
encrypted frames and can perform routing. This is the approach used by Jitsi
Meet's E2EE mode and several other systems. It requires a compatible SFU (not
all SFUs support it — verify your choice).

**SFrame (RFC 9605)** is a standards-track frame encryption scheme for
real-time media. It provides a cleaner separation between the encryption layer
and the transport layer than Insertable Streams and is more portable across
transport protocols. SFrame requires a symmetric key distribution mechanism
(the spec does not provide one).

**MLS (Messaging Layer Security, RFC 9420)** is the IETF standard for
scalable group key agreement. MLS is designed for group communication where
membership changes — exactly the VTC use case. An MLS group tracks the same
membership as your VTCCall's active participants; when a participant joins or
leaves, the group performs a commit that updates the encryption epoch. This
is the right long-term choice for deployments that need forward secrecy and
post-compromise security. MLS implementation complexity is non-trivial;
evaluate OpenMLS or mls-rs as reference implementations.

**Fingerprint display.** The spec says clients SHOULD display `e2eeFingerprint`
to enable out-of-band verification (§VTCParticipant Object Fields,
`e2eeFingerprint`). The format is media-layer-defined. Store the full
hex-encoded SHA-256 of the public key in the field for interoperability, but
display it as 4-6 words from a standardized word list (e.g., BIP-39 or PGP
word list) — VTC has a built-in voice channel, so "read your code aloud" is
the natural verification gesture, and words are far easier to speak than hex.
Clients SHOULD show these in the participant list when `e2eeEnabled` is `true`,
with a "verified" indicator if the user has confirmed the fingerprint
out-of-band. See the E2EE Guide (§8. Fingerprint verification) for format
comparison and derivation details.

**TOFU vs. PKI.** Trust-on-first-use: the client records the fingerprint it
first sees for a userId and warns on change. Simple to implement, effective
against mass surveillance, vulnerable to a first-contact MITM. PKI: the server
or a CA vouches for binding between userId and public key. More robust but
requires a key distribution and revocation infrastructure. For most
deployments, TOFU with UI warnings on fingerprint change is the right starting
point.

**Constraints imposed by E2EE.** When `e2eeEnabled: true`, the server SHOULD
reject recording and livestreaming creates. Gateway participants
(PSTN, SIP, H.323) also cannot participate in an E2EE call without key material
— the gateway has no way to decrypt the media. Clients SHOULD make these
limitations clear in the UI before the user enables E2EE, and the server SHOULD
return a descriptive `forbidden` error if a moderator attempts to start a
recording on an E2EE call.

### Common patterns

| System | E2EE mechanism | Key agreement |
|---|---|---|
| Signal (desktop calling) | SRTP-DTLS + Signal Double Ratchet | Signal Protocol |
| Jitsi Meet (E2EE mode) | WebRTC Insertable Streams | DTLS-SRTP + OlmKit / custom |
| Zoom E2EE | Proprietary | Zoom-managed key tree |
| Element Calls (Matrix) | WebRTC Insertable Streams | MLS via Matrix |

### Recommended starting point

For the first version, do not implement E2EE unless it is a product
requirement. Set `e2eeEnabled: false` as the default and leave the field
read-only for participants. This avoids the complexity of key management and
the recording/gateway constraints.

When adding E2EE: use WebRTC Insertable Streams on the media side and MLS for
key agreement. Store the full hex-encoded SHA-256 of the public key in
`e2eeFingerprint`; display it as 4-6 words from a standardized word list
for voice verification. Implement TOFU fingerprint verification in the client.
Enforce the recording/livestream prohibition in the `VTCRecording/set` and
`VTCLivestream/set` handlers. Make it clear to the user before enabling E2EE
that PSTN dial-in and recording will be unavailable.

---

## 5. Recording and storage

### What the spec leaves open

The spec defines the VTCRecording object, its lifecycle state machine
(`recording` → `paused` → `stopped` → `processing` → `available` or
`failed`), and the consent notification requirement (§Recording Consent,
§VTCRecordingStateEvent). It does not specify: the recording storage backend,
the audio/video format, whether to produce a composite or per-track recording,
how long recordings are retained, or how the `blobId` is generated once
`processing` completes.

### What you must decide

- **Recording backend**: where recordings are stored (local filesystem, object
  storage such as S3, a dedicated media archive service).
- **Recording format**: `"video/mp4"`, `"video/webm"`, `"audio/ogg"`, or
  another MIME type. This is reported in the `mediaType` field when the
  recording is `"available"`.
- **Composite vs. multi-track**: a single mixed-down file vs. one file per
  participant.
- **Retention policy**: how long `"available"` recordings persist, and what
  happens when the VTCCall record is cleaned up.
- **Blob ID generation**: how the recording file becomes a JMAP blob and how
  clients download it.

### Considerations

**Consent signaling is mandatory.** The spec is unambiguous: the server MUST
deliver a `VTCRecordingStateEvent` to all participants whenever recording state
changes (started, paused, resumed, stopped) (§VTCRecording/set). This is not
optional. Implement this event delivery before shipping recording to production.
Jurisdictions vary on consent requirements (one-party vs. all-party consent),
but the notification mechanism must be present regardless. The spec also
recommends providing a mechanism for participants to leave when recording
starts (§Recording Consent); implement this as a UI affordance, not a server
enforcement.

**Composite vs. multi-track.** A composite recording mixes all participant
audio and video into a single file. It is simpler, smaller, and immediately
usable, but loses the ability to attribute speech to speakers in post-processing.
Multi-track recording produces one file per participant (or per media type),
preserving per-speaker attribution for transcription, compliance, or editing
workflows. The `VTCRecording` object models a single recording per `create`
operation; if you produce multiple files for a multi-track recording, decide
whether to create multiple `VTCRecording` objects (one per track) or a single
object pointing to a container blob.

**State machine timing.** The `"processing"` state exists to model the gap
between when recording stops and when the file is available for download. For
composite recordings from a cloud recorder, this may be seconds. For
multi-track recordings requiring a mux step, it may be minutes. Set `stoppedAt`
when the recording session ends, transition to `"processing"` immediately,
deliver `VTCRecordingStateEvent` with `state: "stopped"`, then transition to
`"available"` (setting `blobId`, `size`, `duration`, `mediaType`) when
the file is ready. If the processing step fails, transition to `"failed"` and
log the error for operator review.

**JMAP blob integration.** The `blobId` (§VTCRecording, `blobId`) is a
standard JMAP blob reference (RFC 8620 §6). Once the recording file is stored,
register it in the JMAP blob store and set `blobId` on the VTCRecording object.
Clients retrieve the file via `JMAP/download` with `blobId`. If your recording
backend is an external object store (S3, GCS), the JMAP blob store MAY proxy
the download URL or generate a pre-signed URL; either approach is compatible
with the spec.

**Retention policy.** The spec does not define how long recordings persist.
Decide and document this. Typical choices:
- Retain until explicitly deleted by a moderator.
- Retain for N days after the call ends.
- Retain until storage quota is exceeded, then purge oldest first.

When a recording is deleted (either by moderator or by retention policy), set
`blobId: null` and `state: "failed"` (or a deployment-defined `"deleted"` state,
noting that this is a non-spec extension). Do not destroy the VTCRecording
object itself if the VTCCall record still exists; the object is a historical
record of the recording event.

**E2EE constraint.** See section 4. When `e2eeEnabled: true`, the server SHOULD
reject `VTCRecording/set create` with `forbidden`.

### Common patterns

| Pattern | Storage backend | Format | Track model |
|---|---|---|---|
| Cloud-native | S3 / GCS | MP4 (H.264/AAC) | Composite |
| Self-hosted SFU with recording | Local disk / NFS | WebM (VP8/Opus) | Composite |
| Compliance / transcription | Object store + STT pipeline | Multi-track MP4 or OGG | Per-participant |
| Audio-only | Object store | OGG/Opus or MP3 | Composite |

### Recommended starting point

Composite recording to object storage (S3-compatible), in MP4 format (H.264
video, AAC audio, or Opus if your encoder supports it). Retain recordings for
30 days after the call ends, then delete the blob and mark `blobId: null`.
Register blobs in the JMAP blob store; serve downloads via `JMAP/download`.
Deliver `VTCRecordingStateEvent` synchronously (before returning the
`VTCRecording/set` response) so the notification is guaranteed. Wire up
the `"processing"` → `"available"` transition as a callback from your
recording pipeline rather than polling.

---

## 6. Breakout rooms

### What the spec leaves open

The spec defines the breakout room structure: a child VTCCall with
`parentCallId` set, and the parent's `breakoutRoomIds` list (§VTCCall Object
Fields). It defines the mechanism for creating and closing breakout rooms
(moderator actions via `VTCCall/set`, §Moderator Actions on VTCCall) and
for moving participants between rooms (moderator patches a participant's
`callId` via `VTCParticipant/set`, §Assign to a breakout room). The spec
gates breakout room support behind the `supportsBreakoutRooms` capability
flag. What the spec leaves open: the media layer implications of room
movement, timer-based return, and the UX flow for participant-initiated
return to the main room.

### What you must decide

- **Media layer room model**: how breakout rooms map to your SFU/media
  architecture (separate SFU rooms, separate WebRTC sessions, or a
  sub-conference on the same media server).
- **Participant movement mechanics**: what happens to a participant's media
  session when they are moved from one VTCCall to another.
- **Timer-based return**: whether breakout rooms have a countdown timer after
  which participants are returned to the parent call.
- **Participant-initiated return**: whether participants can leave a breakout
  room and return to the parent call themselves, or whether this requires
  moderator action.
- **Maximum depth**: whether nested breakout rooms (breakout rooms within
  breakout rooms) are permitted.

### Considerations

**JMAP model for participant movement.** Moving a participant from the parent
call to a breakout room requires two JMAP operations (or one if the server
handles them atomically):

1. Set `leftAt` on the participant's VTCParticipant record in the parent call.
2. Create (or reconnect, per §Reconnecting) a VTCParticipant record in the
   child call with the same `userId`.

The spec describes this as a single moderator action ("Assign to a breakout
room", §VTCParticipant/set) that patches the participant's `callId`. The
server executes both operations atomically: it sets `leftAt` on the old
record and creates or reconnects the participant in the new call. Implement
this as a server-side transaction, not two independent client requests.

**Media layer room movement.** When a participant is moved to a breakout room,
their media session must change. For a WebRTC SFU: the participant receives a
new `joinUri` (the breakout room's `joinUri`) and must reconnect. The client
observes the `callId` change on their VTCParticipant record via `StateChange`
and reconnects to the new call's `joinUri`. Do not attempt to migrate an
existing WebRTC session transparently; a new RTCPeerConnection for the new
room is simpler and more reliable.

**Breakout room `joinUri` generation.** Each breakout room is a full VTCCall
with its own `joinUri`. Generate breakout room `joinUri` values the same way
as parent calls — the ULID of the child VTCCall is the room identifier. The
media layer creates a new room for each breakout VTCCall.

**Timer-based return.** The spec does not define a timer mechanism; this is
deployment-defined. A common pattern: when creating a breakout room, include
a deployment-defined `scheduledEndAt` in the `subject` or in the response
metadata. When the timer expires, the server transitions the child VTCCall to
`"ended"` and moves all remaining participants back to the parent call. Notify
participants in advance (e.g., via a chat message on the bound Chat or a custom
ephemeral event) so they are not surprised by the reconnection.

**Participant-initiated return.** The spec does not define this; implement it
as follows: a participant in a breakout room may call `VTCParticipant/set` to
set their own `leftAt` (§Leaving a Call). If the client then joins the parent
call, the server creates a new VTCParticipant record there (or reconnects the
existing one). This is the same as any other room join; no special mechanism
is needed. Optionally, include the parent call's `joinUri` in the breakout
room's VTCCall object via the `parentCallId` field, so the client can display
a "Return to main room" button.

**Depth limit.** Do not support nested breakout rooms (breakout rooms within
breakout rooms). The spec does not prohibit this, but the participant movement
logic becomes significantly more complex and the UX is confusing. Enforce a
maximum `parentCallId` depth of one at the server level.

### Common patterns

| System | Breakout model |
|---|---|
| Zoom | Moderator assigns participants; timer optional; participant can request to return. |
| Jitsi Meet | Breakout rooms are separate Jitsi rooms; participants move via UI; no JMAP equivalent in native Jitsi. |
| Google Meet | Separate media rooms; moderator controls; timer optional. |
| Microsoft Teams | Separate media sessions; timer with countdown; participants can return before timer. |

### Recommended starting point

Implement breakout rooms as separate SFU rooms, each with their own `joinUri`.
Implement participant movement as a server-side atomic operation. Set a
maximum of `maxBreakoutRooms` per call (advertise this in the capability
object). Do not implement timers in the first version — use manual moderator
close. Allow participants to self-return by leaving the breakout call and
joining the parent call. Enforce single-level depth. Add timer support as a
follow-on feature.

---

## 7. Chat integration patterns

### What the spec leaves open

The spec defines optional binding fields on VTCCall (`chatId`, `spaceId`,
`channelId`, §Optional Binding Fields) and specifies the delegation pattern:
when JMAP Chat is co-deployed, several features are handled by Chat rather than
VTC (§Relationship to JMAP Chat). The spec does not prescribe when to
auto-create a Chat, what the Chat type should be, how captions are formatted,
or how the `activeCallId` field on Chat is kept in sync.

### What you must decide

- **Auto-create Chat on call creation**: whether the server automatically
  creates a Chat and sets `chatId` when a new VTCCall is created, or whether
  the caller must supply an existing `chatId`.
- **Chat type for in-call text**: a direct chat between participants (for ring
  calls), a group chat, or a channel chat within a Space.
- **Caption delivery mechanism**: whether live captions are delivered as regular
  Chat messages or as ephemeral events on the Chat WebSocket.
- **`activeCallId` lifecycle**: when to set and clear `activeCallId` on the
  Chat (and any Space/channel), and who is responsible for keeping it current.

### Considerations

**`chatId` binding semantics.** A VTCCall's `chatId` is the id of a Chat
object (from `draft-atwood-jmap-chat-00`) associated with the call. Messages
sent to that Chat appear in the in-call chat UI. For ring calls between two
users, the natural binding is the existing direct Chat between them; set
`chatId` to that Chat's id at call creation. For Space-bound calls (`spaceId`
set), bind to the Space's channel Chat (`channelId`). For room calls with no
Space context, auto-create a group Chat for the call participants.

**Auto-creating Chat.** If the caller does not supply a `chatId` at call
creation, the server MAY auto-create a Chat with the call participants as
members and set `chatId` on the VTCCall. This produces an in-call text chat
automatically, but adds write side effects to the `VTCCall/set create` path
that callers may not expect. Document this behavior explicitly. An alternative:
require callers to supply `chatId` explicitly, and return `invalidArguments`
if it is absent when the Chat capability is present.

**`activeCallId` on Chat.** The `activeCallId` field on Chat signals to Chat
clients that there is an active call associated with this Chat, enabling a
"join call" banner in the chat UI. The VTC server SHOULD set `activeCallId`
on the bound Chat when the VTCCall transitions to `"active"` or `"ringing"`,
and clear it (set to `null`) when the call transitions to `"ended"`. This
requires a cross-type update within the same JMAP request processing path;
implement it as an internal server-side side effect, not a client operation.

**Live captions.** The spec delegates live captions to the Chat's WebSocket
connection (§Relationship to JMAP Chat): captions are delivered as ephemeral
events with `senderId` for speaker attribution. Two delivery patterns:

- *Ephemeral events only*: captions arrive on the WebSocket and are displayed
  in the call UI in real time, but are not persisted as Chat messages. Lower
  storage cost; captions are unavailable after the call.
- *Persisted as messages*: captions are stored as Chat messages in the bound
  Chat, optionally with a special `messageType` to distinguish them from
  regular messages. Enables post-call transcript retrieval.

Choose one. The spec does not require persistence of captions; ephemeral-only
is the simpler starting point.

**Reactions.** The spec mentions in-call reactions (emoji floats) as either
Reaction objects on Messages in the bound Chat, or ephemeral events on the
Chat WebSocket (§Relationship to JMAP Chat). Ephemeral floating reactions
(the animated emoji that floats across the screen during a call) are typically
delivered as ephemeral events rather than persisted; they are UI effects, not
communication content. Persistent reactions on specific messages are Chat-layer
objects. Implement both: ephemeral events for the floating reaction animation,
persistent Reaction objects if users react to a specific in-call chat message.

**Lobby communication.** The spec mentions that lobby participants can
message moderators via the bound Chat before being admitted (§Relationship
to JMAP Chat). If you implement this, create the Chat binding and add the
lobby participant as a member before they are admitted to the call. This
requires that the `chatId` Chat membership is decoupled from the call
`lobbyState` — a participant can be a Chat member without being admitted to
the media call.

**Access control inheritance.** When `chatId`, `spaceId`, or `channelId` is
set, call visibility is inherited from the Chat/Space permission model
(§Chat-Bound Calls): non-members receive `notFound` for VTCCall/get. This
means the VTC server must consult the Chat permission model for every
VTCCall/get and VTCParticipant/get request on a Chat-bound call. Implement
this as a permission check in the VTC layer, not by re-implementing Chat ACLs.
A direct call into the Chat permission resolver is the right coupling.

### Common patterns

| System | In-call chat model |
|---|---|
| Zoom | Separate in-meeting chat; persisted until meeting ends; optionally saved. |
| Jitsi Meet | Persistent chat messages in the call; displayed in sidebar; saved with recording. |
| Slack Huddles | Huddle message thread in the channel; persists after call ends. |
| Google Meet | In-meeting chat; ephemeral by default; optionally saved to Drive. |
| Microsoft Teams | Meeting chat thread in the channel; persists; associated with meeting recording. |

### Recommended starting point

For ring calls: set `chatId` to the existing direct Chat between the two
participants if one exists; create one if not. For Space/channel calls: set
`chatId` to the channel's Chat, `spaceId` and `channelId` to the Space and
channel. Set `activeCallId` on the bound Chat when the call becomes active;
clear it when the call ends. Deliver captions as ephemeral WebSocket events
only in the first version; add persistence as a configurable option later.
Deliver floating reactions as ephemeral events. Require `chatId` to be
supplied by the caller for standalone calls; auto-create for Space-bound calls.

---

## 8. Call retention and cleanup

### What the spec leaves open

The spec defines the `"ended"` terminal state and the `endReason` field but
does not specify how long ended VTCCall records or their associated
VTCParticipant, VTCRecording, and VTCLivestream records should be retained.
It also does not define how the server should handle stale active calls
(calls that are technically `"active"` but have had no media activity for
a long time) or what triggers automatic cleanup.

### What you must decide

- **Ended call retention period**: how long VTCCall records in `"ended"` state
  are kept before being purged.
- **Participant record retention**: whether VTCParticipant records for ended
  calls are retained for the same period as the call or purged earlier.
- **Stale active call cleanup**: what inactivity threshold triggers
  transitioning a technically-active call to `"ended"`.
- **State change notifications on cleanup**: what notifications to deliver
  when the server purges records.

### Considerations

**Ending a call correctly.** The four primary end paths are:

1. Moderator calls `VTCCall/set` to transition `state` to `"ended"`.
2. All participants set `leftAt` (the server auto-ends when the last active
   participant leaves).
3. Ring timeout: the server transitions to `"ended"` with
   `endReason: "missed"` (§Ring Timeout).
4. All ring targets decline: the server transitions to `"ended"` with
   `endReason: "declined"` (§Declining a Ring Call).

Ensure all four paths are implemented and deliver `VTCCallEndEvent` (for ring
calls) and a VTCCall `StateChange` notification to all connected participants
on every transition to `"ended"`.

**Stale active call detection.** A room call that all participants have
abandoned (via media-layer disconnect, not an explicit `leftAt`) may remain
in state `"active"` indefinitely if the media-layer disconnect detection
(see section 2) fails. Implement a scheduled cleanup job that checks for
VTCCalls in `"active"` state where `activeParticipantCount` is `0` and
`endedAt` is `null`, and transitions them to `"ended"` with
`endReason: "completed"`. Run this job every few minutes. Log cleanup events
for operator review.

**Retention period.** Call history is valuable to users (call log, recording
retrieval, audit). Retain ended VTCCall records for at least as long as their
associated recording blobs exist. Typical policies:

- *Short retention (7–30 days)*: low storage cost; limited historical call log;
  acceptable for most consumer products.
- *Long retention (1–7 years)*: required for compliance (financial services,
  healthcare, legal). Requires storage planning and legal hold support.
- *User-controlled deletion*: users may delete their own call records from the
  call history. The spec does not define a "soft delete" pattern; implement
  deletion as a `VTCCall/set` `destroy` operation gated to call participants.

**Cascade behavior on VTCCall deletion.** When a VTCCall record is deleted
(via `VTCCall/set destroy` or by retention policy):

1. Delete all associated VTCParticipant records for that call.
2. Delete (or orphan) associated VTCRecording and VTCLivestream records.
3. Delete the recording blob if it is no longer referenced.

Implement this as a server-side cascade, not client-driven cleanup.

**State change notifications.** When the server auto-transitions a call to
`"ended"` (timeout, last participant left, stale cleanup), MUST deliver a
`StateChange` notification for the VTCCall object to all accounts that have
access to the call. This ensures clients update their call-history UI without
polling. For ring calls that time out, also deliver `VTCCallEndEvent` with
`endReason: "timeout"` to all devices that received the ring notification
(§Ring Timeout, §VTCCallEndEvent).

**Breakout room cleanup.** When a parent call ends, all child breakout-room
VTCCalls MUST also be ended (transitioning them to `"ended"` and setting
`endedAt`). The spec does not state this explicitly, but leaving breakout
rooms in `"active"` state after the parent has ended would produce inconsistent
state. Implement parent-call end as a cascade that ends all child calls first.

**Per-account limits.** The account-level capability advertises
`maxConcurrentCalls` (§Account-Level Capability Object). Enforce this limit in
the `VTCCall/set create` handler: if the account already has the maximum number
of active calls, return `forbidden` (or a deployment-defined `overQuota` error).
Enforce the limit against calls in all non-ended states
(`"creating"`, `"ringing"`, `"active"`, `"pending"`).

### Common patterns

| Scenario | Retention | Notes |
|---|---|---|
| Consumer calling app | 30 days | Short call history; users rarely need older records. |
| Enterprise with compliance | 7 years | Legal hold; encrypted at rest; audit log. |
| Ephemeral/anonymous calls | 24 hours | No persistent call history by design. |
| Hospitality / support | 90 days | Long enough for dispute resolution; not indefinite. |

### Recommended starting point

Retain ended VTCCall records for 90 days. Retain VTCParticipant records for
the same period. Purge recording blobs at 30 days (configurable) with a
separate blob-retention lifecycle policy. Run a stale-call cleanup job every
5 minutes targeting active calls with zero active participants. End parent
calls before child breakout rooms in all cascade paths. Deliver `StateChange`
notifications for every server-initiated `"ended"` transition. Log all
automatic end events with the `endReason` for operator observability.

---

## Appendix: Decision checklist

Before deploying JMAP VTC to production, verify that your implementation has
made and documented each of the following decisions:

**Media stack (section 1)**
- [ ] Media protocol selected (WebRTC / SIP / other)
- [ ] SFU or MCU or mesh; specific implementation chosen
- [ ] `joinUri` format defined and documented
- [ ] STUN/TURN deployed; credential rotation configured
- [ ] Gateway protocols listed in `gateways` capability if applicable

**VTCParticipant lifecycle (section 2)**
- [ ] Media-layer presence detection implemented (webhook or heartbeat)
- [ ] Reconnect identity window defined
- [ ] Lobby flow: waiting participants routed to lobby media context
- [ ] Moderator role assignment policy documented

**Policy enforcement (section 3)**
- [ ] Entry policies (`muteOnEntry`, `videoOffOnEntry`) applied at join time
- [ ] `participantsCanUnmute / ShareScreen / StartVideo` enforced in
  `VTCParticipant/set` handler
- [ ] Webinar mode (all `Can*` false) tested end-to-end
- [ ] Ask-to-unmute flow (`unmuteRequested` + `VTCUnmuteRequestEvent`) implemented

**E2EE (section 4)**
- [ ] E2EE decision made: supported or explicitly deferred
- [ ] If supported: mechanism chosen; key exchange documented
- [ ] Fingerprint format defined and displayed in client UI
- [ ] Recording/livestream prohibition enforced when `e2eeEnabled: true`

**Recording (section 5)**
- [ ] `VTCRecordingStateEvent` delivered on every state change
- [ ] Recording backend and format documented
- [ ] `"processing"` → `"available"` transition wired to recording pipeline
- [ ] Blob retention period configured
- [ ] E2EE call recording prohibition enforced

**Breakout rooms (section 6)**
- [ ] Breakout room feature flag (`supportsBreakoutRooms`) set correctly
- [ ] Participant movement implemented as server-side atomic operation
- [ ] Media layer creates separate room for each breakout VTCCall
- [ ] `maxBreakoutRooms` enforced
- [ ] Nested breakout depth limited to one

**Chat integration (section 7)**
- [ ] `chatId` binding strategy documented (auto-create vs. caller-supplied)
- [ ] `activeCallId` set and cleared on call state transitions
- [ ] Caption delivery mechanism chosen (ephemeral vs. persisted)
- [ ] Access control inheritance implemented for Chat-bound calls

**Retention and cleanup (section 8)**
- [ ] Ended call retention period configured
- [ ] Stale active call cleanup job scheduled
- [ ] Cascade delete implemented (VTCCall → VTCParticipant, VTCRecording)
- [ ] `StateChange` notifications delivered on server-initiated end transitions
- [ ] Breakout room cascade end-ordering implemented
- [ ] Per-account `maxConcurrentCalls` limit enforced
