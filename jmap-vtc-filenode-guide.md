# JMAP VTC — FileNode Integration Guide

For VTC implementers using JMAP FileNode for recordings and file sharing with `draft-atwood-jmap-vtc-00`.

---

## Introduction

`draft-atwood-jmap-vtc-00` defines VTCRecording and VTCLivestream for tracking recording
metadata, and references JMAP blob storage (`blobId`) as the mechanism for accessing
completed recording files. The spec deliberately leaves the storage backend unspecified:
"Where and how recording blobs are stored. The spec only models the metadata and the
resulting `blobId`." (Appendix, Explicit Non-Prescriptions).

JMAP FileNode (`draft-ietf-jmap-filenode`) provides a natural fit for that storage
layer. FileNode gives recordings a first-class object identity, folder organization,
access-control inheritance, and lifecycle management — all through the same JMAP API
that the VTC client already speaks. In-call file sharing (slides, documents, images
shared during a session) fits the same model: files uploaded during a call become
FileNode objects in a call-scoped folder, accessible to participants and cleaned up
when the call ends.

This guide covers the integration points between the two capabilities. It is non-normative.
The VTC draft and the FileNode draft are the sources of truth; where this guide and a
draft disagree, the draft wins.

This guide uses RFC 2119 keywords for clarity in the spirit of implementer guidance, not
as a normative protocol specification. Cite the drafts, not this guide, when claiming
normative authority.

Read `draft-atwood-jmap-vtc-00`, `draft-ietf-jmap-filenode`, and
`jmap-chat-filenode-guide.md` before this guide. This guide does not re-state normative
requirements from those documents.

---

## 1. In-call file sharing

### What the spec leaves open

`draft-atwood-jmap-vtc-00` does not define a file-sharing mechanism. In-call file sharing
is delegated: when JMAP Chat is present, the bound Chat (via `chatId`) provides the
message-and-attachment surface. When FileNode is also present, shared files can be
organized as FileNode objects rather than transient blob attachments.

### What you must decide

- Whether in-call file sharing uses transient JMAP blob uploads (no FileNode, no
  persistence beyond the session) or FileNode objects (persistent, queryable, with
  access control).
- Where in the FileNode tree shared files live: a per-call folder under the Space's
  file tree, a call-scoped ephemeral folder, or a flat per-Space uploads folder.
- Access control scope: who can download files shared during a call — only current
  participants, all Space members, or anyone with the link.
- Cleanup policy: what happens to call-shared files after the call ends.
- Screen share versus file share distinction: screen sharing is a media-layer event
  (represented in VTCMediaState as `screen: true`); file sharing is a separate
  operation that uploads a file for participants to download. These are different
  mechanisms; do not conflate them.

### Considerations

- *Transient blob uploads* (direct to JMAP blob endpoint, no FileNode) are the
  simplest path. The presenter uploads a blob, sends the `blobId` as a Chat message
  attachment (when JMAP Chat is present), and participants download via the blob
  endpoint. The file has no persistent identity; it may be garbage-collected when no
  FileNode references it. Appropriate for lightweight sharing where post-call access
  is not needed.
- *FileNode-backed sharing* gives the file a persistent identity, a visible name, a
  parent folder, and query-ability. The presenter creates a FileNode in a call-scoped
  folder; participants receive the FileNode id (via a Chat message or a VTC-specific
  signal); all call participants can fetch it by id. After the call, the file remains
  accessible under the same FileNode id. Better for slides, reference documents, and
  any content participants may want later.
- *Access control*: the simplest model is to inherit access from the call's bound
  Space or Chat. Current Space members who were call participants can access shared
  files; non-participants cannot. Moderators should be able to remove a shared file
  mid-call.
- *Call-scoped folder* (a FileNode folder created when the call starts, named by the
  call's id or subject, parented to the Space's file tree) keeps per-call artifacts
  organized and makes post-call browsing natural. Create the folder lazily (when the
  first file is shared), not eagerly.
- *Cleanup*: call-shared files SHOULD persist after the call ends unless the deployment
  has a configured retention policy. Auto-deleting call-shared files at call end
  surprises users who want to retrieve a slide deck they received during the session.
  A configurable retention window (default: indefinite, or tied to the Space's
  retention policy) is a better default.
- *Screen share vs file share*: screen sharing is `VTCMediaState.screen: true` on a
  VTCParticipant and is handled entirely by the media layer. File sharing is a JMAP
  blob or FileNode upload operation. Never use VTCMediaState to model file sharing;
  they are orthogonal features.

### Common patterns

| System | In-call file sharing approach |
|---|---|
| Zoom | Files uploaded to in-meeting chat; persisted to cloud or local storage after the meeting |
| Microsoft Teams | Files shared during meetings become SharePoint files in the team's document library |
| Slack Huddles | Files shared in the bound channel; persistent in the channel's message history |
| Google Meet | File-sharing is out-of-band (Google Drive links in chat); no in-call upload mechanism |
| Jitsi Meet | No native file sharing; side-channel approaches (Etherpad, chat links) |

### Recommended starting point

Use **FileNode-backed sharing** when both `urn:ietf:params:jmap:vtc` and
`urn:ietf:params:jmap:filenode` are present. Concrete pattern:

1. When the first file is shared in a call, create a FileNode folder under the bound
   Space's file tree, named `"Call: <subject or callId>"`, parented to the Space root.
   Store the folder's `id` on the call (deployment-side; not a VTC wire field).
2. The presenter uploads the file as a FileNode in that folder. Send the FileNode id
   to other participants via a Chat message attachment (using `filenodeId` per the
   filenode chat spec) or, when Chat is not present, via a deployment-defined
   out-of-band channel.
3. Access control: inherit from the Space's FileNode permission model. Only current
   and past Space members who have `"member"` role or above can read files in the
   call folder.
4. Moderators MAY destroy individual FileNodes during the call to retract a shared
   file.
5. After the call ends, retain the call folder and its contents indefinitely (or per
   the Space's configured file-retention policy). Do not auto-delete.

When FileNode is not present, fall back to transient blob uploads with `blobId` shared
via Chat message. Document the fallback behavior in the deployment's user-facing help.

---

## 2. Recordings as FileNodes

### What the spec leaves open

`draft-atwood-jmap-vtc-00` §5.3 (VTCRecording/set) states that when recording state
transitions from `"stopped"` through `"processing"` to `"available"`, the server sets
`blobId` to a JMAP blob reference for the recording file. The spec does not define how
or where that blob is stored, how it is organized, how access control is enforced beyond
the VTCRecording's own visibility rules, or what metadata is attached to it beyond
`duration`, `size`, and `mediaType`.

### What you must decide

- Whether to expose recordings as raw blobs only (blob id on VTCRecording, no FileNode)
  or as FileNode objects backed by the same blob (richer metadata, folder organization,
  access control through the FileNode permission model).
- Folder structure for recordings: per-call folder, per-date folder, per-Space folder,
  or a flat recordings directory.
- Metadata to surface as FileNode or deployment-defined extension fields: participant
  list, call start and end time, codec, track structure.
- Access control: who can retrieve recordings after the call ends — only original
  participants, all Space members, Space admins only.
- Retention policy: how long recordings are kept, whether they are auto-deleted,
  whether they are archived to cold storage.
- Multi-track recordings: a single composite file (one FileNode, one blob) or separate
  FileNodes per speaker track.

### Considerations

- *Raw blob only* is the minimal implementation: VTCRecording.blobId is set when
  processing completes; any participant who can read the VTCRecording object can
  download the blob via the standard JMAP blob endpoint. Simple; no FileNode
  dependency; no folder organization; no post-call browsing without a query.
- *FileNode-backed recordings* give the recording a browsable location, a persistent
  name, and FileNode-level access control. The VTCRecording.blobId still points to
  the underlying JMAP blob; the FileNode wraps it with a parent folder, a human-readable
  name, and additional metadata fields. Participants can browse past recordings through
  the Space's file tree rather than querying VTCRecording directly.
- *Folder structure*: a per-Space "Recordings" folder at the Space root, with per-call
  subfolders named by the call subject or date, is the most navigable. Flat per-Space
  recordings directories are easier to implement but harder to browse at scale.
- *FileNode metadata for recordings*: the FileNode `name` field can carry a
  human-readable recording name (e.g., `"2026-06-05 Standup.mp4"`). Deployment-defined
  extension fields or a sidecar FileNode (a `.json` or `.txt` in the same folder) can
  carry participant list, call start/end timestamps, codec, and track count. Do not
  attempt to carry this in the recording blob's MIME type or filename alone.
- *Access control*: the simplest correct model for ad-hoc calls (no Space context) is:
  anyone who was a participant in the call (has a VTCParticipant record with `leftAt`
  not null or `joinedAt` not null) can read the recording FileNode. Space admins can
  always read recordings in the Space. Non-participants cannot. Moderators can destroy
  a recording FileNode (which also removes the blob reference, but does not destroy the
  VTCRecording metadata object — that remains as an audit record with `blobId` set to
  null). For Space-bound calls, see §3 for the broader default access model.
- *Retention policy*: recordings are large. Auto-deletion after N days (configurable;
  default 90 days is a reasonable starting point for most deployments — the base
  implementer guide suggests 30 days as a more conservative default; FileNode-backed
  deployments typically use longer retention because structured metadata enables better
  lifecycle management) prevents unbounded storage growth. Archive to cold storage
  (e.g., S3 Glacier or equivalent) before deletion for compliance-sensitive deployments.
  Communicate the retention policy clearly in the user-facing help.
- *Multi-track recordings*: composite (mixed) recordings are one FileNode and one blob.
  Per-speaker tracks are separate FileNodes in the same call folder, named by participant
  display name. Multi-track is operationally more complex; default to composite unless
  the deployment's media stack produces per-speaker tracks natively.
- *E2EE calls*: `draft-atwood-jmap-vtc-00` §7.4 notes that servers SHOULD reject
  VTCRecording creates with `forbidden` when `e2eeEnabled` is `true`. If a deployment
  nonetheless supports client-side encrypted recordings, the recording blob is opaque
  to the server; do not attempt server-side processing or thumbnail generation on it.

### Common patterns

| System | Recording storage approach |
|---|---|
| Zoom | Cloud recordings in per-account cloud drive; organized by meeting date and topic |
| Microsoft Teams | Recordings stored in the initiator's OneDrive or SharePoint team site |
| Google Meet | Recordings saved to the organizer's Google Drive; link shared in the Calendar event |
| Jitsi Meet (JaaS) | Recordings stored in the deployment's S3 bucket; share link sent to participants |
| Slack Huddles | Short clip recordings; stored as file uploads in the bound channel |

### Recommended starting point

Use **FileNode-backed recordings** when both capabilities are present. Concrete pattern:

1. When VTCRecording transitions to `"available"`, the server creates a FileNode in a
   `"Recordings"` folder under the bound Space's file tree (create the folder lazily
   on first recording). The FileNode `name` is
   `"<YYYY-MM-DD> <subject or callId>.<ext>"` where `ext` is derived from
   VTCRecording.mediaType.
2. Set VTCRecording.blobId to the same blob that backs the FileNode's content.
3. Access control: grant read access to all VTCParticipant records associated with the
   call (resolved at the moment the recording becomes available). Space admins inherit
   write access (including destroy) via the Space's FileNode permission model.
4. Attach a sidecar `<same-name>.json` FileNode in the same call folder, containing
   a deployment-defined JSON object with: `callId`, `startedAt`, `endedAt`,
   `participants` (list of `{userId, displayName, joinedAt, leftAt}`), `duration`,
   `mediaType`. This sidecar carries metadata that does not fit on the FileNode itself.
5. Retention: default 90-day auto-delete (the base implementer guide suggests 30 days
   as a more conservative default; FileNode-backed deployments typically use longer
   retention because structured metadata enables better lifecycle management). Implement
   as a background job that destroys the FileNode (and sidecar) after 90 days from
   `stoppedAt`, preserving the VTCRecording metadata object with `blobId` set to null
   as an audit record.

---

## 3. Shared content references after the call

### What the spec leaves open

`draft-atwood-jmap-vtc-00` does not define a post-call summary mechanism. When a call
ends, the VTCCall object transitions to `"ended"` with `endedAt` and `endReason` set.
The spec does not prescribe how participants access recordings, shared files, or any
auto-generated artifacts (meeting notes, transcripts) after the session.

### What you must decide

- Whether to post a post-call summary message to the bound Chat automatically when the
  call ends.
- What the summary contains: recording links, shared-file links, participant list,
  duration.
- Whether auto-generated artifacts (transcripts, meeting notes) are stored as FileNodes
  or as Chat messages.
- How to surface recording and shared-file links to participants who were not in the
  call but have access to the Space (e.g., a Space member who missed the meeting).
- Content addressing: whether to use JMAP CID (content-addressable identifiers, when
  available from `draft-ietf-jmap-blob`) for stable recording references.

### Considerations

- *Post-call summary message* in the bound Chat is the most discoverable pattern:
  when the call ends, the server automatically posts a Chat message (as the system
  actor, not as the call initiator) summarizing the call. The message includes:
  call subject, duration, participant count, links to the recording FileNode (if any),
  and links to files shared during the call. This message is findable in the channel
  history by anyone with access; no special query is needed.
- *Chat message format*: the JMAP Chat Message body carries the summary as human-readable
  text. FileNode references go in the `attachments` array using `filenodeId` per the
  filenode chat spec, giving clients the information needed to render inline previews
  and download links. Do not embed raw blob URLs in the message body; use `filenodeId`
  references so access control is enforced on download.
- *Transcripts and meeting notes*: if the deployment generates a transcript or
  AI-generated notes, store them as FileNodes in the call's folder (alongside the
  recording and shared files). Reference them in the post-call summary message. This
  keeps all post-call artifacts in one browsable location.
- *Non-participant access*: Space members who were not in the call can see the call
  folder under the Space's file tree (subject to the Space's FileNode permission model)
  and can read the post-call summary message in the Chat. Recording FileNodes SHOULD
  be accessible to all Space members by default (not restricted to original
  participants only), unless the call was marked private or confidential at creation
  time. For Space-bound calls, the default broadens from the participant-only model
  in §2 to all Space members, since the call occurred in a shared context.
- *JMAP CID*: when `draft-ietf-jmap-blob` content-addressable identifiers are available,
  use CIDs for recording references in addition to FileNode ids. A CID is stable
  across blob moves and storage migration; a FileNode id is stable for the lifetime
  of the FileNode object. Both are useful; CIDs add robustness for long-lived
  archive references.
- *Calendar integration*: when `calendarEventId` is set on the VTCCall, the post-call
  summary can be linked from the CalendarEvent (via a deployment-defined extension or
  by updating the event's description field with the recording link). Attendees who
  accepted the calendar invite can retrieve the recording through the calendar entry
  without needing to find the Chat channel.

### Common patterns

| System | Post-call content references |
|---|---|
| Zoom | Auto-email to all participants with recording link; link also available in the Zoom web portal |
| Microsoft Teams | Meeting summary posted to the Teams channel; recording in the meeting chat and SharePoint |
| Google Meet | Recording link emailed to the organizer; link in the Calendar event |
| Slack Huddles | Clip link posted to the channel where the Huddle was started |
| Jitsi Meet | No built-in post-call summary; deployment-defined |

### Recommended starting point

When both VTC and JMAP Chat are present:

1. When VTCCall transitions to `"ended"`, post a system-actor Chat message to the
   bound Chat. Message body (plain text): `"Call ended — [duration]. [N] participants."`.
   If a recording is available (VTCRecording.state is `"available"` or `"processing"`),
   append: `"Recording: [FileNode name]."`. List files shared during the call (from
   the call's FileNode folder) as `filenodeId` attachments.
2. If VTCRecording is still in `"processing"` when the call ends, delay the summary
   message until processing completes (or set a 30-minute timeout after which the
   summary is sent without the recording link, with a note that processing is in
   progress).
3. All files in the call's FileNode folder (recordings, shared files, transcripts, notes)
   are readable by all Space members with `"member"` role or above by default. If the
   call had `lobbyEnabled: true` and some participants were rejected, those users do
   not gain access to call files simply by being Space members — enforce based on
   actual admission status.
4. When `calendarEventId` is present, update the CalendarEvent's description field
   with a plain-text link to the recording FileNode after processing completes.
5. Do not send post-call emails outside the JMAP surface; the Chat summary message is
   the canonical artifact. External notification (email, push) should reference the
   Chat message, not carry a separate copy of the links.

---

## Cross-references

| Topic | See also |
|---|---|
| VTC recording lifecycle | `draft-atwood-jmap-vtc-00` §5.3 (VTCRecording/set) |
| VTCRecording data type fields | `draft-atwood-jmap-vtc-00` §4.5 |
| Recording consent notification | `draft-atwood-jmap-vtc-00` §7.3 |
| E2EE and recording incompatibility | `draft-atwood-jmap-vtc-00` §4.2 (VTCCall, `e2eeEnabled`) |
| JMAP FileNode data model | `draft-ietf-jmap-filenode` |
| FileNode storage backend choice | `jmap-chat-filenode-guide.md` §1 |
| FileNode quota management | `jmap-chat-filenode-guide.md` §4 |
| FileNode retention after Space destruction | `jmap-chat-filenode-guide.md` §6 |
| Chat message attachment with `filenodeId` | `draft-atwood-jmap-chat-filenode-00` |
| Space role to FileNode permission mapping | `draft-atwood-jmap-chat-filenode-00` {#permissions} |
| VTC capability overview and call lifecycle | `draft-atwood-jmap-vtc-00` §3–§6 |
| JMAP blob content-addressable identifiers | `draft-ietf-jmap-blob` |
