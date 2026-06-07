# JMAP VTC -- Recording Guide

For implementers building call recording with `draft-atwood-jmap-vtc-00`.

Read the spec first, then the recording section of the Implementer's Guide
(section 5, "Recording and storage"). This guide does not restate those decisions. It
covers the full VTCRecording lifecycle in detail: every state transition, every
JSON request, every error condition, and the consent, compliance, and storage
decisions you must make before shipping.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- `draft-atwood-jmap-vtc-00` is the normative source of truth. Where this
  guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and spec disagree, the spec wins.
- Where this guide uses a keyword for an operational practice, UX default, or
  deployment choice, the keyword reflects implementer best practice. Deviation
  does not affect protocol interop.
- Cite the spec, not this guide, when claiming normative authority.

---

## 1. The VTCRecording object

A VTCRecording is a top-level JMAP data type that tracks the metadata of a
recording session within a VTCCall. It does not contain the recording media
itself; it is a state envelope that points to a JMAP blob when the recording
is complete.

### Fields

| Field | Type | Set by | Notes |
|---|---|---|---|
| `id` | String | Server | Immutable. Opaque server-assigned identifier. |
| `callId` | String | Client (create) | Immutable. The VTCCall this recording belongs to. |
| `state` | String | Server / Client | Current lifecycle state (see section 2). |
| `startedAt` | UTCDate | Server | Time recording started. |
| `stoppedAt` | UTCDate or null | Server | Time recording stopped. `null` while active or paused. |
| `initiatedBy` | String | Server | userId of the participant who started the recording. |
| `blobId` | String or null | Server | JMAP blob reference. Available only when `state` is `"available"`. |
| `size` | UnsignedInt or null | Server | File size in bytes. Available only when `state` is `"available"`. |
| `duration` | UnsignedInt or null | Server | Duration in seconds. Available when `state` is `"available"` or `"stopped"`. |
| `mediaType` | String or null | Server | MIME type (e.g., `"video/mp4"`). Available only when `state` is `"available"`. |

The `callId` is the only client-supplied field at creation time. Every other
field is either server-set or patched by the client via `VTCRecording/set`
update (only `state` is patchable, and only for the transitions listed in
section 2).

### Access control

All current and past participants of the associated VTCCall may retrieve
VTCRecording objects via `VTCRecording/get`. The server MUST return `notFound`
for recording ids belonging to calls the authenticated user has no access to.
When the call carries a `chatId`, `spaceId`, or `channelId` binding,
VTCRecording visibility is inherited from the parent VTCCall
(see spec, Access Control).

Only participants with `role: "moderator"` on the associated VTCCall may
create, update, or destroy VTCRecording objects. Non-moderators MUST receive
`forbidden`.

---

## 2. Lifecycle state machine

VTCRecording has six states:

```
                     client
  [create] -----> "recording"
                    |      |
           pause    |      |  stop
                    v      v
                "paused"  "stopped"
                  |   |       |
           resume |   | stop  | (server-initiated)
                  v   v      v
             "recording"  "processing"
                            |        |
                   success  |        |  failure
                            v        v
                      "available"  "failed"
```

### Client-controllable transitions

These transitions are requested by a moderator via `VTCRecording/set`:

| From | To | Trigger | Event delivered |
|---|---|---|---|
| (none) | `"recording"` | `create` | `VTCRecordingStateEvent` with `state: "recording"` |
| `"recording"` | `"paused"` | `update` | `VTCRecordingStateEvent` with `state: "paused"` |
| `"paused"` | `"recording"` | `update` | `VTCRecordingStateEvent` with `state: "recording"` |
| `"recording"` | `"stopped"` | `update` | `VTCRecordingStateEvent` with `state: "stopped"` |
| `"paused"` | `"stopped"` | `update` | `VTCRecordingStateEvent` with `state: "stopped"` |

### Server-initiated transitions

These transitions happen automatically after a recording is stopped. The client
cannot trigger them:

| From | To | Trigger |
|---|---|---|
| `"stopped"` | `"processing"` | Server begins preparing the recording file. |
| `"processing"` | `"available"` | File is ready; `blobId`, `size`, `duration`, `mediaType` are set. |
| `"processing"` | `"failed"` | Processing error (encoding failure, disk full, etc.). |

### Invalid transitions

The server MUST return `invalidArguments` for any transition not listed above.
Examples of invalid transitions:

- `"stopped"` back to `"recording"` (a stopped recording cannot be restarted)
- `"available"` to anything (terminal state for the file)
- `"failed"` to anything (terminal state)
- `"processing"` to anything (server-controlled, not client-controllable)

If you need a new recording after stopping one, create a new VTCRecording
object. Each recording session is a separate object.

---

## 3. Starting a recording

### Request

A moderator sends `VTCRecording/set` with a `create` containing only `callId`:

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "create": {
    "r0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB"
    }
  }
}, "0"]]
```

### Server behavior

On success, the server:

1. Assigns `id` (opaque, server-generated).
2. Sets `state` to `"recording"`.
3. Sets `startedAt` to the current time.
4. Sets `initiatedBy` to the authenticated user's userId.
5. Delivers a `VTCRecordingStateEvent` to all participants in the call (see
   section 7).
6. Instructs the recording backend to begin capturing media.

### Response

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "created": {
    "r0": {
      "id": "01J4XP2SN9RXVU1SSEGIUT6GH",
      "state": "recording",
      "startedAt": "2026-06-06T14:30:00Z",
      "initiatedBy": "user:moderator@example.com",
      "stoppedAt": null,
      "blobId": null,
      "size": null,
      "duration": null,
      "mediaType": null
    }
  }
}, "0"]]
```

### Error conditions

The server MUST return an error for the create in each of these cases:

| Condition | Error type | Why |
|---|---|---|
| Caller is not a moderator | `forbidden` | Only moderators may start recordings. |
| `e2eeEnabled: true` on the call | `forbidden` | Recording requires server-side media access, which is incompatible with E2EE. |
| `supportsRecording` is `false` | `forbidden` | The account does not support recording. |
| Call is not in state `"active"` | `invalidArguments` | You cannot record a call that has not started or has already ended. |
| Storage quota exceeded | `overQuota` | Deployment-defined; see section 9. |

Example error response (E2EE conflict):

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "notCreated": {
    "r0": {
      "type": "forbidden",
      "description": "Recording is unavailable when end-to-end encryption is enabled."
    }
  }
}, "0"]]
```

### Pre-create checks for clients

Before sending the create request, the client SHOULD check:

1. The account capability `supportsRecording` is `true`.
2. The call's `e2eeEnabled` is `false`.
3. The call's `state` is `"active"`.
4. The current user's `role` on their VTCParticipant record is `"moderator"`.

If any check fails, do not send the request. Display an appropriate message to
the user explaining why recording is unavailable.

---

## 4. Pausing and resuming

### Use cases

Pausing a recording is useful when:

- A sensitive or off-the-record discussion occurs mid-call.
- The meeting takes a break (coffee, bio, lunch).
- A participant requests that a specific segment not be recorded.
- Legal counsel joins briefly and the conversation should not be captured.

### Pausing

A moderator sends `VTCRecording/set` with an `update` setting `state` to
`"paused"`:

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "paused"
    }
  }
}, "1"]]
```

The server:

1. Transitions the recording state from `"recording"` to `"paused"`.
2. Instructs the recording backend to stop capturing (but keep the session
   open for resumption).
3. Delivers a `VTCRecordingStateEvent` with `state: "paused"` to all
   participants.

### Resuming

A moderator sends `VTCRecording/set` with an `update` setting `state` back to
`"recording"`:

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "recording"
    }
  }
}, "2"]]
```

The server:

1. Transitions the recording state from `"paused"` to `"recording"`.
2. Instructs the recording backend to resume capturing.
3. Delivers a `VTCRecordingStateEvent` with `state: "recording"` to all
   participants.

### Client UI considerations

When a recording is paused:

- Display a "paused" indicator distinct from the active recording indicator.
  A pulsing or dimmed recording icon is common.
- Show a "Resume" button to moderators.
- Continue showing the paused state to non-moderators so they know the
  recording session still exists (and may resume at any time).

The `duration` field on VTCRecording reflects total recorded time, not wall
clock time. Paused intervals are excluded. The server or recording backend
is responsible for tracking this correctly.

---

## 5. Stopping and finalizing

### Stopping

A moderator sends `VTCRecording/set` with an `update` setting `state` to
`"stopped"`:

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "stopped"
    }
  }
}, "3"]]
```

This works from either `"recording"` or `"paused"` state. The server:

1. Sets `stoppedAt` to the current time.
2. Delivers a `VTCRecordingStateEvent` with `state: "stopped"` to all
   participants.
3. Instructs the recording backend to finalize the capture.

### Auto-stop on call end

When a VTCCall transitions to `"ended"`, the server MUST stop all active
recordings for that call. Any VTCRecording in `"recording"` or `"paused"`
state is transitioned to `"stopped"`, with `stoppedAt` set and
`VTCRecordingStateEvent` delivered. The server then proceeds with the normal
processing pipeline.

This is a server-side cascade, not a client operation. Clients should not
need to explicitly stop recordings before ending a call.

### Processing

After a recording is stopped, the server transitions it to `"processing"`.
This state models the gap between media capture ending and the downloadable
file being ready. During processing:

- The recording backend muxes, encodes, or transcodes the raw capture into the
  final format.
- For composite recordings, this means mixing all participant streams into a
  single file.
- For multi-track recordings, this may mean packaging individual tracks into a
  container or producing separate files.

The processing duration depends on the recording backend:

| Backend type | Typical processing time |
|---|---|
| Cloud recorder with real-time mux | Seconds |
| SFU-side recording (e.g., Janus, mediasoup) | Seconds to minutes |
| Post-capture multi-track mux | Minutes |
| Transcription + mux pipeline | Minutes to tens of minutes |

### Available

When processing completes successfully, the server transitions the recording to
`"available"` and sets:

- `blobId`: a JMAP blob reference (RFC 8620, section 6) for the recording file.
- `size`: file size in bytes.
- `duration`: total recorded duration in seconds (excluding paused intervals).
- `mediaType`: MIME type of the file (e.g., `"video/mp4"`, `"video/webm"`,
  `"audio/ogg"`).

The server delivers a `StateChange` notification for VTCRecording so clients
know the file is ready.

Example of a completed VTCRecording object:

```json
{
  "id": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "state": "available",
  "startedAt": "2026-06-06T14:30:00Z",
  "stoppedAt": "2026-06-06T15:02:17Z",
  "initiatedBy": "user:moderator@example.com",
  "blobId": "G4c5b8a0e1f2d3c4b5a6978",
  "size": 157286400,
  "duration": 1832,
  "mediaType": "video/mp4"
}
```

### Downloading

Clients retrieve the recording file via the standard JMAP blob download
mechanism:

```
GET /jmap/download/{accountId}/{blobId}/{name}?accept={type}
```

For example:

```
GET /jmap/download/moderator-account/G4c5b8a0e1f2d3c4b5a6978/recording.mp4?accept=video/mp4
```

The `blobId` is a standard JMAP blob reference. If your recording backend uses
an external object store (S3, GCS), the JMAP blob store MAY proxy the download
or generate a pre-signed URL; either approach is compatible with the spec.

### Failed

If processing fails (encoding error, disk full, I/O failure), the server
transitions the recording to `"failed"`. The `blobId` remains `null`. This is a
terminal state. See section 10 for error handling patterns.

---

## 6. Destroying recordings

### Request

A moderator sends `VTCRecording/set` with a `destroy`:

```json
[["VTCRecording/set", {
  "accountId": "moderator-account",
  "destroy": ["01J4XP2SN9RXVU1SSEGIUT6GH"]
}, "4"]]
```

### What happens

1. The VTCRecording metadata object is removed.
2. If the recording was in state `"available"`, the associated blob is subject
   to the server's standard blob lifecycle. The spec does not guarantee
   immediate deletion of the blob; the server MAY defer blob cleanup to a
   garbage-collection cycle.
3. If the recording was in `"recording"` or `"paused"` state, the server SHOULD
   stop the active capture before removing the metadata. Deliver a
   `VTCRecordingStateEvent` with `state: "stopped"` before the destroy
   completes.

### Permission requirements

The server MUST return `forbidden` when the caller is not a moderator on the
associated call. Non-moderators cannot destroy recordings.

### Blob lifecycle after destroy

The blob referenced by `blobId` may still be referenced by other objects (for
example, a FileNode object linking to the same blob). The server MUST NOT delete
the blob if other references exist. When no references remain, the server
applies its standard blob retention policy.

For deployments using FileNode integration (see section 9), destroying the
VTCRecording metadata does not automatically destroy the FileNode object. These
are independent references to the same blob. To fully remove a recording, a
moderator must destroy both the VTCRecording and the associated FileNode.

---

## 7. Consent signals and VTCRecordingStateEvent

### The event

Every recording state change MUST produce a `VTCRecordingStateEvent` delivered
to all participants in the call via their WebSocket connections. This is a
mandatory consent signal.

```json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "recording",
  "initiatedBy": "user:moderator@example.com"
}
```

The `state` field in the event carries one of three values:

| Value | Meaning |
|---|---|
| `"recording"` | Recording has started or resumed. |
| `"paused"` | Recording is paused. |
| `"stopped"` | Recording has stopped. |

The post-stop states (`"processing"`, `"available"`, `"failed"`) are not
delivered as `VTCRecordingStateEvent`. Clients learn about those transitions
through standard JMAP `StateChange` notifications for the VTCRecording type
and re-fetch via `VTCRecording/get`.

### Delivery timing

The server MUST deliver the `VTCRecordingStateEvent` before returning the
`VTCRecording/set` response. This ensures that all participants are notified
of the recording state change at least as early as the moderator receives
confirmation that the operation succeeded. Do not defer event delivery to a
background queue.

### Client UI on receiving a VTCRecordingStateEvent

When a client receives a `VTCRecordingStateEvent`:

**`state: "recording"` (started or resumed):**

1. Display a prominent recording indicator. A red dot or "REC" badge in the
   call UI is the industry convention.
2. If this is the first `"recording"` event for this `recordingId` (a new
   recording, not a resume), display a modal or banner informing the user
   that the call is now being recorded. Include the name of the user who
   started the recording (resolve `initiatedBy` to a display name).
3. Offer a "Leave call" affordance. The spec recommends providing a mechanism
   for participants to leave when recording starts, as an implicit decline of
   consent.

**`state: "paused"`:**

1. Update the recording indicator to show a paused state (dimmed dot, "PAUSED"
   label).
2. No modal is necessary; the indicator change is sufficient.

**`state: "stopped"`:**

1. Remove the recording indicator.
2. Optionally display a brief notification that recording has ended.

### Legal and compliance implications

The `VTCRecordingStateEvent` is the technical mechanism; the legal framework
around it is deployment-defined. Key considerations:

**One-party consent jurisdictions** (e.g., most US states, England and Wales):
Only one participant needs to consent to the recording. The notification is
informational. The recording may proceed regardless of whether other
participants acknowledge it.

**All-party consent jurisdictions** (e.g., California, Illinois, Germany, many
EU member states): All participants must consent to the recording. The spec does
not define a consent-acknowledgment mechanism (there is no `consentGiven` field
on VTCParticipant). Deployments operating in all-party consent jurisdictions
SHOULD implement one of:

1. **Explicit consent flow:** When a `VTCRecordingStateEvent` with
   `state: "recording"` arrives, the client displays a consent dialog. The
   participant must affirmatively accept before continuing. If they decline,
   the client automatically leaves the call. This consent decision is logged
   by the client or communicated to the server via a deployment-defined
   extension.

2. **Pre-call consent:** The call invitation or join flow includes a consent
   notice (e.g., "This call may be recorded. By joining you consent to
   recording."). Joining the call constitutes consent.

3. **Auto-recording with notice:** The VTCCall is configured to start
   recording automatically when the call becomes active. The join flow
   includes a mandatory consent acknowledgment. Participants who do not
   acknowledge cannot join.

The spec's position: the signaling mechanism (the event) MUST be present
regardless of jurisdiction. What the deployment does with the signal is
deployment-defined.

---

## 8. Multiple simultaneous recordings

### Is it supported?

The spec does not prohibit multiple simultaneous VTCRecording objects for the
same VTCCall. Each `VTCRecording/set create` produces an independent recording
object. Whether the server supports this is deployment-defined.

### Use cases

- **Composite plus multi-track:** One recording captures a mixed composite
  (for immediate playback); another captures per-participant tracks (for
  transcription or post-production editing).
- **Different media types:** One recording captures video+audio (`"video/mp4"`);
  another captures audio-only (`"audio/ogg"`) for a lighter-weight archive.
- **Compliance dual-recording:** A primary recording for business use and a
  separate tamper-evident recording for regulatory compliance.

### Server-side mixing versus per-participant tracks

The spec models a single VTCRecording per `create` operation. It does not define
whether the recording is composite or per-track. This is a backend decision:

**Composite recording** (one mixed file): The recording backend mixes all
participant audio and video into a single stream. The VTCRecording object
points to one blob. This is the simpler model and the recommended starting
point.

**Per-participant tracks** (one file per participant): The recording backend
captures each participant's media separately. You have two options for
modeling this:

1. **One VTCRecording, one container blob.** The backend produces a
   multi-track container (e.g., a Matroska file with labeled tracks, or a zip
   archive). The VTCRecording points to one blob; the client must understand the
   container format to extract individual tracks.

2. **Multiple VTCRecording objects, one per track.** The server creates a
   separate VTCRecording for each participant's track. Each has its own
   `blobId`, `size`, `duration`, and `mediaType`. Clients query
   `VTCRecording/query` with `callId` to discover all tracks. This is more
   flexible but produces more objects.

### Recommended approach

Start with composite recording. Add multi-track support as a follow-on if
your use case requires per-speaker attribution, transcription, or
post-production editing. If you implement multi-track, prefer one
VTCRecording per track over a container blob -- it works naturally with the
existing JMAP query and blob download mechanisms.

---

## 9. Storage integration

### JMAP blob storage

The recording file is stored as a JMAP blob (RFC 8620, section 6). The
`blobId` on VTCRecording is a standard blob reference. Clients download it
via the `JMAP/download` endpoint. This is the minimum integration required
by the spec.

### External object stores

If your recording backend writes to an external object store (S3, GCS, Azure
Blob Storage), you have two options:

1. **Proxy through JMAP blob store.** Register the recording as a blob in the
   JMAP store; the blob store proxies downloads from the external store. The
   client sees a normal `blobId` and uses `JMAP/download`. This is the simpler
   client experience.

2. **Pre-signed URL redirect.** The JMAP blob store generates a time-limited
   pre-signed URL for the external object and returns a redirect. The client
   follows the redirect. This avoids double-hop bandwidth costs for large
   recordings but requires client support for redirects on blob downloads.

Either approach is compatible with the spec.

### FileNode integration

When `draft-ietf-jmap-filenode` is available, recordings can be organized as
FileNode objects in the file tree. This gives recordings folder organization,
access-control inheritance, and lifecycle management through the standard JMAP
FileNode API.

The integration pattern (see the FileNode Integration Guide for details):

1. When a VTCRecording transitions to `"available"`, the server creates a
   FileNode object referencing the same `blobId`.
2. The FileNode is placed in a call-scoped folder (e.g.,
   `/{space}/recordings/{callId}/`).
3. The FileNode inherits access control from its parent folder (typically the
   Space or channel permissions).
4. The VTCRecording's `blobId` and the FileNode's `blobId` reference the same
   underlying blob.

This means recordings are discoverable through two paths: `VTCRecording/query`
(from the VTC API) and `FileNode/query` (from the file tree). Both are valid;
the VTC path is call-centric, the FileNode path is folder-centric.

### Quota considerations

Recording files are large. A one-hour 720p video call at typical bitrates
produces roughly 500 MB to 1 GB. Deployments MUST plan for this:

- **Account-level quota.** Track recording blob usage against the account's
  JMAP storage quota. Return `overQuota` on `VTCRecording/set create` if the
  account cannot accommodate a new recording.
- **Per-recording size limits.** Optionally enforce a maximum recording duration
  or file size. Stop the recording automatically when the limit is reached,
  transitioning to `"stopped"` and delivering the appropriate event.
- **Retention policy.** Define how long `"available"` recordings persist. Purge
  recording blobs after a configurable period (30, 60, 90 days). When purging,
  set `blobId` to `null` on the VTCRecording and transition to `"failed"` (or
  a deployment-defined `"expired"` extension state). Do not destroy the
  VTCRecording metadata itself if the VTCCall record still exists; the object
  is a historical record of the recording event.

---

## 10. Error handling

### Mid-recording failures

Recording can fail while active. Common causes:

- **Disk full or storage quota exceeded.** The recording backend cannot write
  more data.
- **Media pipeline crash.** The SFU or recording process crashes or is
  restarted.
- **Network partition.** The recording backend loses connectivity to the media
  stream.
- **Encoding error.** A codec error or corrupt frame causes the encoder to
  abort.

When a recording fails mid-capture:

1. The server transitions the recording from `"recording"` (or `"paused"`) to
   `"stopped"`, setting `stoppedAt`.
2. The server delivers a `VTCRecordingStateEvent` with `state: "stopped"` to
   all participants so the recording indicator is removed.
3. The server attempts to salvage whatever media was captured. If a partial
   file is recoverable, it proceeds through `"processing"` to `"available"`
   with a partial recording. If nothing is recoverable, it transitions to
   `"failed"`.

### Processing failures

After a recording is stopped, the processing step (muxing, encoding) can fail:

1. The server transitions the recording from `"processing"` to `"failed"`.
2. The `blobId` remains `null`.
3. The server delivers a `StateChange` notification for VTCRecording so
   clients know the recording is not available.

Clients that detect a `"failed"` state SHOULD display a message indicating that
the recording is unavailable and suggesting that the moderator contact the
system administrator.

### Server-side observability

Log every recording state transition with:

- The recording id and call id.
- The transition (from state, to state).
- The userId who triggered the transition (or "server" for automatic
  transitions).
- For failures: the error message and category (storage, encoding, network).

This log is the primary diagnostic tool for recording failures. Expose it
through your operational monitoring system.

---

## 11. WebSocket event delivery

### Event types for recording

Two notification mechanisms cover the VTCRecording lifecycle:

**`VTCRecordingStateEvent`** (ephemeral WebSocket event): Delivered for
client-visible state changes (`"recording"`, `"paused"`, `"stopped"`). These
are consent signals and must reach all participants immediately.

**`StateChange`** (standard JMAP push): Delivered for all state changes,
including server-initiated transitions (`"processing"`, `"available"`,
`"failed"`). Clients use this to know when to re-fetch VTCRecording objects.

### Client implementation pattern

```
on VTCRecordingStateEvent:
  if event.state == "recording":
    show recording indicator
    if new recording (not seen this recordingId before):
      show consent banner
  else if event.state == "paused":
    show paused indicator
  else if event.state == "stopped":
    hide recording indicator

on StateChange for VTCRecording:
  fetch updated VTCRecording via VTCRecording/get
  if recording.state == "available":
    enable "Download recording" button
  else if recording.state == "failed":
    show recording failure message
```

### Handling late-joining participants

A participant who joins a call where recording is already active will not
receive the original `VTCRecordingStateEvent`. The client MUST query
`VTCRecording/query` with `callId` and `state: "recording"` (or
`state: "paused"`) on join to discover active recordings and display the
appropriate indicator.

```json
[["VTCRecording/query", {
  "accountId": "participant-account",
  "filter": {
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
    "state": "recording"
  }
}, "0"],
["VTCRecording/get", {
  "accountId": "participant-account",
  "#ids": {
    "resultOf": "0",
    "name": "VTCRecording/query",
    "path": "/ids"
  }
}, "1"]]
```

If results are returned, show the recording indicator immediately. Also query
for `state: "paused"` to detect paused recordings.

---

## 12. Compliance patterns

### Auto-recording

Some organizations require that all calls be recorded. Implement this as
follows:

1. Configure the server to automatically create a VTCRecording when a VTCCall
   transitions to `"active"`.
2. The auto-created recording uses a system userId as `initiatedBy` (or the
   call initiator's userId, depending on deployment policy).
3. The `VTCRecordingStateEvent` is delivered normally, so all participants are
   notified.
4. The join flow includes a mandatory consent acknowledgment. The client
   displays a notice before the user can join: "This call will be recorded.
   By joining, you consent to recording." The server MAY enforce this by
   rejecting `VTCParticipant/set create` until a consent flag is set (this is
   a deployment-defined extension, not part of the spec).

### Recording retention policies

| Policy | Retention | Use case |
|---|---|---|
| Short-term | 7--30 days | Consumer calling; users rarely need old recordings. |
| Medium-term | 90 days | Enterprise; long enough for dispute resolution. |
| Long-term | 1--7 years | Financial services, healthcare, legal compliance. |
| Indefinite | Until explicitly deleted | Regulated industries with legal hold requirements. |

Implement retention as a server-side background job that:

1. Queries for VTCRecording objects with `state: "available"` and `stoppedAt`
   older than the retention period.
2. Deletes the blob.
3. Sets `blobId: null` and transitions the VTCRecording state to `"failed"`
   (or a deployment-defined `"expired"` extension state).
4. Delivers a `StateChange` notification.

### Audit trails

For compliance deployments, maintain an immutable audit log of all recording
events:

- Who started the recording, when, on which call.
- Every pause, resume, and stop event with timestamps and actor userIds.
- When the recording became available (with blob hash for integrity
  verification).
- When and by whom the recording was accessed (downloaded).
- When and by whom the recording was destroyed.

This audit log is separate from the VTCRecording object itself. The
VTCRecording object may be destroyed; the audit log must not be.

### Legal hold

When a recording is subject to a legal hold, the retention policy must not
delete it. Implement legal hold as a server-side flag on the blob (or on the
VTCRecording, as a deployment-defined extension field). The retention job skips
recordings under legal hold.

---

## 13. Querying recordings

### VTCRecording/query

Use `VTCRecording/query` to find recordings. The available filter properties:

| Property | Type | Description |
|---|---|---|
| `callId` | String | Recordings for this call. |
| `state` | String | Filter by state. |
| `initiatedBy` | String | Recordings started by this userId. |
| `startedAfter` | UTCDate | Recordings started at or after this time. |
| `startedBefore` | UTCDate | Recordings started before this time. |

All filter properties are combined with logical AND. Default sort:
`startedAt` descending.

### Example: find all available recordings for a call

```json
[["VTCRecording/query", {
  "accountId": "user-account",
  "filter": {
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
    "state": "available"
  }
}, "0"],
["VTCRecording/get", {
  "accountId": "user-account",
  "#ids": {
    "resultOf": "0",
    "name": "VTCRecording/query",
    "path": "/ids"
  }
}, "1"]]
```

### Example: find all recordings by a specific user in a date range

```json
[["VTCRecording/query", {
  "accountId": "admin-account",
  "filter": {
    "initiatedBy": "user:moderator@example.com",
    "startedAfter": "2026-06-01T00:00:00Z",
    "startedBefore": "2026-06-30T23:59:59Z"
  },
  "sort": [{"property": "startedAt", "isAscending": true}]
}, "0"],
["VTCRecording/get", {
  "accountId": "admin-account",
  "#ids": {
    "resultOf": "0",
    "name": "VTCRecording/query",
    "path": "/ids"
  }
}, "1"]]
```

---

## Appendix: Complete lifecycle walkthrough

This walkthrough shows the full lifecycle of a recording from start to download
in a two-participant call between Alice (moderator) and Bob (participant).

### 1. Alice starts recording

Request:

```json
[["VTCRecording/set", {
  "accountId": "alice-account",
  "create": {
    "r0": {
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB"
    }
  }
}, "0"]]
```

Both Alice and Bob receive via WebSocket:

```json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "recording",
  "initiatedBy": "user:alice@example.com"
}
```

Bob's client displays a recording indicator and a consent banner.

### 2. Alice pauses for a break

Request:

```json
[["VTCRecording/set", {
  "accountId": "alice-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "paused"
    }
  }
}, "1"]]
```

Both receive:

```json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "paused",
  "initiatedBy": "user:alice@example.com"
}
```

Bob's client updates the indicator to show "paused".

### 3. Alice resumes after break

Request:

```json
[["VTCRecording/set", {
  "accountId": "alice-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "recording"
    }
  }
}, "2"]]
```

Both receive:

```json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "recording",
  "initiatedBy": "user:alice@example.com"
}
```

Bob's client shows the active recording indicator again.

### 4. Alice stops recording

Request:

```json
[["VTCRecording/set", {
  "accountId": "alice-account",
  "update": {
    "01J4XP2SN9RXVU1SSEGIUT6GH": {
      "state": "stopped"
    }
  }
}, "3"]]
```

Both receive:

```json
{
  "@type": "VTCRecordingStateEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "recordingId": "01J4XP2SN9RXVU1SSEGIUT6GH",
  "state": "stopped",
  "initiatedBy": "user:alice@example.com"
}
```

Bob's client removes the recording indicator.

### 5. Server processes and finalizes

The server transitions the recording through `"processing"` to `"available"`.
Both Alice and Bob receive a `StateChange` notification:

```json
{
  "@type": "StateChange",
  "changed": {
    "alice-account": {
      "VTCRecording": "s301"
    }
  }
}
```

Alice's client fetches the updated recording:

```json
[["VTCRecording/get", {
  "accountId": "alice-account",
  "ids": ["01J4XP2SN9RXVU1SSEGIUT6GH"]
}, "4"]]
```

Response:

```json
[["VTCRecording/get", {
  "accountId": "alice-account",
  "list": [{
    "id": "01J4XP2SN9RXVU1SSEGIUT6GH",
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
    "state": "available",
    "startedAt": "2026-06-06T14:30:00Z",
    "stoppedAt": "2026-06-06T15:02:17Z",
    "initiatedBy": "user:alice@example.com",
    "blobId": "G4c5b8a0e1f2d3c4b5a6978",
    "size": 157286400,
    "duration": 1832,
    "mediaType": "video/mp4"
  }]
}, "4"]]
```

### 6. Bob downloads the recording

Bob's client uses the `blobId` to download:

```
GET /jmap/download/alice-account/G4c5b8a0e1f2d3c4b5a6978/meeting-recording.mp4?accept=video/mp4
```

The server returns the recording file.

> **Cross-account blob access.** JMAP blob downloads are scoped to the account
> that owns the blob (RFC 8620 §6.2). The `downloadUrl` template includes
> `accountId` and the server MUST verify the authenticated user has access to
> that account's blobs. In this walkthrough the recording blob lives in
> `alice-account` (the moderator who created it), so Bob must reference
> `alice-account` in the download URL — not his own account.
>
> For multi-account deployments where call participants span separate accounts,
> the server must provide one of these mechanisms so all authorized participants
> can retrieve recording blobs:
>
> (a) **Copy-on-create.** Store the recording blob in each participant's
>     account when the recording becomes available. Each participant downloads
>     from their own account. This is the simplest client experience but
>     multiplies storage costs.
>
> (b) **Cross-account blob access via JMAP Sharing (RFC 8620 §1.6.2).** Grant
>     participants read access to the recording-owning account's blobs. Bob
>     authenticates as himself but downloads from `alice-account` because
>     sharing grants permit it.
>
> (c) **Deployment-specific mechanism.** Use pre-signed URLs, a shared blob
>     store, or a proxy endpoint that validates VTCRecording access control
>     independently of JMAP account boundaries.
>
> The VTC spec's access control rule — all current and past participants may
> retrieve recordings (see section 1, Access control) — implicitly requires one
> of these mechanisms when participants span accounts. Implementers MUST
> document which approach their deployment uses.

---

## Appendix: Decision checklist

Before deploying recording to production, verify that your implementation has
made and documented each of the following decisions:

**Recording backend**
- [ ] Storage backend selected (object store, local disk, NFS)
- [ ] Recording format selected (MP4, WebM, OGG)
- [ ] Composite vs. multi-track decision made
- [ ] Recording pipeline tested for the processing to available transition

**Consent**
- [ ] `VTCRecordingStateEvent` delivered on every state change
- [ ] Client displays recording indicator on receiving the event
- [ ] Client offers leave-call affordance when recording starts
- [ ] Consent model documented for target jurisdictions (one-party vs. all-party)

**Permissions**
- [ ] Non-moderator create/update/destroy returns `forbidden`
- [ ] E2EE call recording create returns `forbidden`
- [ ] `supportsRecording: false` create returns `forbidden`
- [ ] Non-active call create returns `invalidArguments`

**Storage and lifecycle**
- [ ] `blobId` registered in JMAP blob store when recording becomes available
- [ ] Recording download tested via `JMAP/download`
- [ ] Storage quota enforcement implemented
- [ ] Retention policy configured and background purge job scheduled

**Error handling**
- [ ] Mid-recording failure path tested (stops recording, delivers event)
- [ ] Processing failure path tested (transitions to failed)
- [ ] Client handles failed state with appropriate messaging

**Auto-stop**
- [ ] Call end cascades stop to all active recordings
- [ ] `VTCRecordingStateEvent` delivered for auto-stopped recordings

**Late join**
- [ ] Client queries for active recordings on join
- [ ] Recording indicator displayed for in-progress recordings

**Compliance (if applicable)**
- [ ] Auto-recording implemented with pre-join consent
- [ ] Audit log captures all recording lifecycle events
- [ ] Legal hold prevents retention policy from purging recordings
- [ ] Retention period documented and enforced
