# JMAP VTC — Chat Delegation Guide

For VTC implementers leveraging JMAP Chat for in-call features with `draft-atwood-jmap-vtc-00`.

---

## How to read this guide

`draft-atwood-jmap-vtc-00` (the VTC spec) recommends that deployments co-advertise
`urn:ietf:params:jmap:chat` alongside `urn:ietf:params:jmap:vtc`. When both
capabilities are present, several in-call collaboration features — text chat,
reactions, live captions, lobby communication — are delegated to JMAP Chat's
existing infrastructure rather than re-invented inside the VTC layer.

This guide explains how to implement that delegation. It is written from the
perspective of a VTC implementer who knows the VTC spec and is learning how to
wire up the Chat side. It does not re-state normative requirements from either
spec. Where a spec requirement is referenced, a citation is given so you can
read the normative text yourself.

This guide is non-normative. The drafts are the source of truth. Where this
guide and a draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords for clarity in the spirit of implementer
guidance. Where a keyword describes a spec requirement, it reflects the spec's
normativity. Where it describes an operational practice or deployment choice,
it reflects best practice. Deviation from a best-practice recommendation does
not affect protocol interoperability. Cite the spec, not the guide, when
claiming normative authority.

---

## 1. Introduction: why delegate?

Building text chat, reactions, and caption delivery from scratch inside a VTC
system duplicates engineering that JMAP Chat has already solved: delivery
guarantees, read tracking, membership management, WebSocket fan-out,
accessibility, and federation. Every feature you build from scratch is also a
feature you must maintain, document, and debug separately.

The VTC spec takes a delegation approach. `VTCCall` carries three Chat
back-reference fields — `chatId`, `spaceId`, and `channelId` — that bind a
call to an existing Chat infrastructure object. Once that binding exists,
in-call text, reactions, and captions flow through the Chat layer with no new
protocol surface on the VTC side.

The sections below walk through each delegated feature: how to create and
manage the Chat binding, how to use it for text chat, how captions and
reactions attach to the Chat's WebSocket stream, how lobby communication works,
and how to keep presence state consistent between the VTC and Chat layers.

---

## 2. Auto-creating call Chats

### The chatId binding

`VTCCall` carries an optional `chatId` field
(`draft-atwood-jmap-vtc-00` Section 4, "Optional Binding Fields"). When this
field is set, the Chat identified by that id is the call's dedicated in-call
text channel. Everything that would otherwise require a VTC-native chat
mechanism is handled there instead.

A caller creating a ring call, room call, or scheduled call MAY supply an
existing `chatId`. When no `chatId` is supplied, the server SHOULD auto-create
a new Chat and record the resulting id on the VTCCall. This is the recommended
path for most call types: it ensures every call has a chat channel ready before
the first participant joins.

### Choosing a Chat kind

When auto-creating a Chat for a call, choose the Chat `kind` based on the
call's context:

- **Ring calls (1:1):** Create a `kind: "direct"` Chat between the two
  participants. This is a Chat-layer operation: the server looks up whether a
  `kind: "direct"` Chat already exists with `contactId` matching the target
  participant's userId, and if so MUST reuse it (the Chat spec requires
  deduplication of direct chats). The call's messages appear alongside the
  existing conversation history.

- **Ring calls (group) and room calls without a Space context:** Create a
  `kind: "group"` Chat with initial `memberIds` matching the VTCParticipant
  list. Give the group chat a `name` derived from the call `subject` if one is
  set.

- **Room calls and scheduled calls inside a Space:** Use the existing channel
  Chat (`kind: "channel"`) for the Space. Set `channelId` on the VTCCall to
  that channel's id. Do not create a new Chat; the call is scoped to the
  channel's existing conversation.

- **Ephemeral room calls (huddles, ad-hoc rooms):** Create a `kind: "group"`
  Chat and set `messageExpirySeconds` to a short window (e.g., 86400 seconds)
  if post-call log retention is not desired. This keeps the chat visible during
  and briefly after the call while automatically discarding it.

### Membership sync

When the Chat is auto-created for a call, its initial membership should match
the initial VTCParticipant list. Ongoing, the server SHOULD keep membership in
sync as participants join and leave:

- When a VTCParticipant with a `userId` joins the call (i.e., `joinedAt` is
  set), add that userId to the Chat's `members` list via `Chat/set` if they
  are not already a member.
- When a VTCParticipant leaves and will not rejoin (i.e., `leftAt` is set and
  the call has ended), the server MAY leave them as a member so they can
  read the post-call chat history, or remove them immediately depending on
  deployment policy.
- Gateway participants (PSTN, SIP, H.323) who have no `userId` cannot be Chat
  members. Do not attempt to add them.
- Lobby participants (discussed in Section 6) require special handling; do not
  add them to the main call Chat until they are admitted.

### Chat lifecycle and the call lifecycle

A Chat created for a call is a regular Chat object. Its lifecycle is
independent of the call's lifecycle: the call ends, the Chat persists. This is
usually what you want — participants can continue the conversation or read the
transcript after the call.

If your deployment wants to discard the chat on call end, the cleanest
mechanism is `messageExpirySeconds` set at Chat creation time, not programmatic
deletion. Deleting a Chat retroactively removes the history for all
participants and should only be done deliberately.

The Chat spec defines `Chat.activeCallId` (Chat spec Section on the Chat object)
as a server-set field that reflects when a bound VTCCall is in the `"active"`
state. Your server MUST set this field when the call enters `"active"` and
clear it to `null` when the call ends. Clients use `activeCallId` to display a
"join call" banner on the Chat. This is the primary call-awareness signal for
Chat clients; do not omit it.

### When to bind an existing Chat vs. create a new one

Bind an existing Chat (i.e., the caller supplies a `chatId`) when:

- The call is being started from within a channel or group chat UI — the user
  expects messages to appear in the conversation they are already looking at.
- A scheduled call was created from a Space channel; that channel's Chat is
  the natural host.

Create a new Chat when:

- No Chat context exists (ad-hoc ring call, spontaneous room call).
- The call participants form a set that does not correspond to any existing
  Chat.

Do not create a new Chat when one already exists for the same participants in
the same context. The Chat spec's deduplication rule for direct Chats handles
the 1:1 case automatically. For group Chats, your call-creation logic should
check for an existing group Chat covering the same participant set before
creating a new one.

---

## 3. In-call text chat

### The bound Chat as text channel

Once a VTCCall has a `chatId` binding, the bound Chat is the in-call text
channel. Participants send Messages to that Chat using the standard
`Message/set` create method, and receive them via `StateChange` notifications
and `Message/changes` calls. No new VTC methods are required.

The VTC layer's only responsibility here is ensuring that each participant's
Chat client knows which `chatId` corresponds to the current call. The
`VTCCallPush` ring notification payload already carries `chatId`
(`draft-atwood-jmap-vtc-00` Section on VTCCallPush), so clients receive this
binding at ring time. Clients can also read `VTCCall.chatId` directly.

### Client UI considerations

In-call chat panels typically show messages alongside the video grid. The
client queries `Message/query` filtered to the bound `chatId`, sorted by
`receivedAt`, to populate the panel. The `Chat.unreadCount` field provides the
unread badge count. `ReadPosition` should be updated as the participant reads
messages, using the standard `ReadPosition/set` mechanism, so the unread count
stays accurate.

During an active call, chat notifications for the bound Chat should be
displayed inline in the call UI rather than as system notifications. Suppress
the Chat's push notifications on the device while the in-call UI is
foregrounded — the user is looking at the chat panel already. See Section 7
for presence and notification coordination.

### Post-call message persistence

Messages in the bound Chat persist after the call ends unless a
`messageExpirySeconds` policy causes them to expire. This is the default and
usually the right behavior. Post-call transcripts and chat logs are valuable.

If your deployment requires chat logs to be discarded after the call, use
`messageExpirySeconds` on the Chat, set at call/Chat creation time. A common
pattern is to offer a deployment-level policy (e.g., "ephemeral calls discard
chat after 24 hours") rather than per-call user choice.

There is no mechanism to retroactively remove chat history from an existing
Chat without deleting individual messages. Plan your retention policy at Chat
creation time.

---

## 4. Live captions via ephemeral events

### Why ephemeral, not messages

Live captions from a speech-to-text engine produce high-frequency, rapidly
superseded text fragments. Storing each caption fragment as a `Message` would
flood the chat history with noise: hundreds of low-value intermediate
transcription results per minute, each requiring storage, delivery,
`StateChange` fan-out, and client-side `Message/changes` retrieval. The chat
log would be unreadable.

The correct delivery path is ephemeral WebSocket events. The
`draft-atwood-jmap-chat-wss-00` spec defines a `ChatStreamEnable` message that
accepts a `dataTypes` array. The `"vtc"` value is registered by the VTC spec
for in-call events (`draft-atwood-jmap-chat-wss-00` Section on ChatStreamEnable,
`draft-atwood-jmap-vtc-00` Section on In-Call Ephemeral Events). Caption events
travel on this channel: they are delivered immediately, not persisted, and
silently dropped on reconnect.

### Enabling the caption stream

The client sends a `ChatStreamEnable` message on the same WebSocket connection
it uses for chat:

```json
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence", "vtc"],
  "chatIds": null,
  "contactIds": null
}
```

Including `"vtc"` in `dataTypes` opts the connection into VTC-scoped ephemeral
events for calls the authenticated user is participating in, including caption
events.

### Caption event format

The VTC spec does not define a rigid caption event schema beyond the common
ephemeral event envelope. A caption event SHOULD carry at minimum:

- `@type`: a deployment-defined string, e.g., `"VTCCaptionEvent"`
- `callId`: the VTCCall id
- `chatId`: the bound Chat id (allows the client to scope display)
- `senderId`: the VTCParticipant.userId of the speaker (for attribution)
- `text`: the caption fragment as UTF-8 text
- `isFinal`: Boolean — `false` for interim results, `true` for a committed
  segment
- `language` (optional): BCP 47 language tag if the ASR engine provides it

Interim results (`isFinal: false`) should replace the previous caption for the
same `senderId` in the UI. Final results (`isFinal: true`) may be appended to
a running caption display or used to synthesize a post-call transcript stored
as a Message in the bound Chat.

### Speaker attribution

`senderId` maps to a `VTCParticipant.userId`, which maps to a
`ChatContact.id`. Use the ChatContact's `displayName` or `login` field to label
the speaker in the caption UI.

For speakers who joined via gateway (no `userId`), use the
`VTCParticipant.displayName` field instead.

### Accessibility considerations

Live captions serve participants who are deaf or hard of hearing, who are in
noisy environments, or who are consuming the call in a language that benefits
from a text parallel. Treat captions as a first-class feature, not an
afterthought. Recommendations:

- Caption events SHOULD be delivered on the WebSocket regardless of whether
  the participant has explicitly enabled captions in the UI. Let the client
  decide whether to render them. Server-side suppression removes the option.
- Final caption segments SHOULD be stored as Messages in the bound Chat after
  the call ends, attributed to the speaker, to provide a searchable transcript.
  Mark them clearly (e.g., with a `bodyType` or metadata field indicating
  "transcript") so they are distinguishable from participant-typed messages.
- Rate-limit caption delivery to avoid flooding the WebSocket. Coalesce rapid
  interim results on the server before fan-out. One event per speaker per
  300–500 ms is a practical floor for readable captions.

---

## 5. In-call reactions

In-call emoji reactions — the "raise hand", "clap", "heart" buttons in
Zoom/Meet — have two distinct use cases with different persistence
requirements. Use different mechanisms for each.

### Persistent reactions: Reaction objects on Messages

A participant reacting to a specific Message in the in-call chat should use
the standard Reaction mechanism: a `Message/set` update that patches the
`reactions` map on the target Message. The Reaction has an `emoji` field, a
`senderId`, and a `sentAt` timestamp. This produces a permanent record of who
reacted to what, visible in the post-call chat history, and rendered by any
JMAP Chat client without VTC-specific logic.

This is the right path for reactions that are contextually tied to a message
(thumbs-up on a question, checkmark on a decision).

### Ephemeral float reactions: ephemeral events

"Applause" reactions and emoji floats — the visual effect where emojis drift
up the screen when many participants react simultaneously — are not tied to any
specific Message. They are audience sentiment signals with a lifespan of a few
seconds. Storing each one as a Reaction on a synthetic Message would pollute
the chat history with hundreds of individually meaningless records.

Use ephemeral events instead. Deliver float reactions as VTC-scoped ephemeral
events on the bound Chat's WebSocket stream, alongside caption events:

```json
{
  "@type": "VTCReactionEvent",
  "callId": "01J3XKZ...",
  "chatId": "01J3XKY...",
  "senderId": "user:alice@example.com",
  "emoji": "\ud83d\udc4f",
  "sentAt": "2026-06-05T14:23:01Z"
}
```

These events are not persisted. If the client misses them (disconnect,
reconnect), that is acceptable — a missed applause animation is not a
functional loss.

### Recommendation

Use **ephemeral events** for float reactions and visual audience sentiment
signals. Use **Reaction objects** for per-message reactions in the chat panel.
Do not conflate the two: a Reaction on a Message implies semantic attachment to
that message's content; a float reaction is ambient sentiment that happens to
occur at a moment in time.

If your deployment wants to retain a summary of reaction activity for post-call
analytics (e.g., "42 clap reactions at timestamp 14:23"), synthesize a single
summary Message or metadata record at call end, not one record per reaction
event.

---

## 6. Lobby chat

### The lobby scenario

When `VTCCall.lobbyEnabled` is `true`, participants who join the call enter a
waiting room. Their `VTCParticipant.lobbyState` is `"waiting"` until a
moderator admits them. During this wait, they may need to communicate with the
host — to announce themselves, clarify their name, or ask when the call will
start.

The main call Chat is not appropriate for lobby participants: they have not
been admitted to the call yet, and full visibility into the call's chat
history before admission may not be desired.

### Separate lobby Chat vs. restricted access

Two approaches are practical:

**Option A: Separate lobby Chat.** Create a second `kind: "group"` Chat for
lobby participants. When a moderator creates the call, also create a lobby Chat
and record its id in `VTCCall` metadata (e.g., via JMAP Metadata if available,
or in a deployment-defined field). Add lobby participants to the lobby Chat
when their `lobbyState` is set to `"waiting"`. When they are admitted, add
them to the main call Chat. They now see both conversations in their client.

The lobby Chat's membership is: waiting participants + moderators. This allows
moderators to communicate with waiting participants and allows waiting
participants to see each other (or not, depending on whether you want
participant-to-participant lobby chat). To prevent waiting participants from
messaging each other — allowing them to message moderators only — use a channel
`kind: "channel"` with a Space permission model that restricts `"send"` to
moderators and removes it from other members.

**Option B: Restricted view of the main call Chat.** Do not create a separate
Chat. Instead, structure the main call Chat as a Space channel and use
`permissionOverrides` to restrict what waiting participants can see. This is
more complex to implement correctly and is not recommended unless your
deployment already uses Space channels as the primary call chat structure.

For most deployments, Option A is simpler and more correct. The lobby Chat is
ephemeral infrastructure; set `messageExpirySeconds` to a short window so it
clears automatically.

### Host broadcast to waiting participants

A moderator wanting to broadcast a message to all waiting participants (e.g.,
"We'll start in 5 minutes") sends a Message to the lobby Chat. If the lobby
Chat is a `kind: "group"`, this is a standard group message. If it is a
channel with a `"send"` restriction on participants, the moderator's SpaceRole
permits sending while participants can only read.

`BroadcastMention` with `scope: "everyone"` is available for group and channel
Chats and will notify all current members of the lobby Chat. This is the
appropriate tool for time-sensitive broadcast messages to waiting participants.

### Transitioning on admission

When a moderator admits a waiting participant (`lobbyState` transitions from
`"waiting"` to `"admitted"`):

1. Add the participant's userId to the main call Chat's `members` list via
   `Chat/set`.
2. Optionally remove them from the lobby Chat, or leave them in both.
3. The client should switch the UI focus from the lobby chat panel to the main
   call chat panel.

The `VTCParticipant.lobbyState` transition is the authoritative signal; the
Chat membership change follows from it.

---

## 7. Presence and notification coordination

### Reflecting call state in presence

When a user is actively in a call, their presence should say so. Participants
looking at the user's presence indicator — in a contact list, a DM panel, or
a channel member list — should see that the user is busy.

The mechanism is `PresenceStatus`. Each account has exactly one `PresenceStatus`
record. When a participant joins a call (their `VTCParticipant.joinedAt` is
set), the server SHOULD update the participant's `PresenceStatus` to:

```json
{
  "presence": "busy",
  "statusText": "In a call",
  "statusEmoji": null,
  "expiresAt": null
}
```

When the participant leaves the call (`VTCParticipant.leftAt` is set), the
server SHOULD restore their previous presence state, or reset to
`"online"` if no prior state was captured.

The `PresenceStatus.expiresAt` field can be used as a backstop: set it to
the call's expected end time if known (e.g., for scheduled calls), so that
presence auto-clears even if the participant's client disconnects abruptly
without a clean leave event.

`ChatContact.presence` on the remote side will reflect this via
`ChatPresenceEvent` delivery to contacts who have subscribed to presence for
this user. This is the correct propagation path; do not write `ChatContact`
records directly.

### Notification muting during calls

Chat notification sounds — the ping when a new message arrives — are
disruptive during a call. A user who is actively in an audio or video call
should not have their in-call audio interrupted by notification sounds.

The mechanism is `Chat.muted`. When a participant joins a call, the server MAY
set `muted: true` on the Chats the participant is not actively using (i.e., all
Chats except the bound call Chat). Restore `muted: false` on those Chats when
the call ends.

A softer alternative is to handle this entirely on the client: detect that the
user is in an active call (via `Chat.activeCallId` being non-null on the call's
Chat) and suppress notification sounds in the client UI without updating the
server-side `muted` flag. This avoids the risk of incorrectly leaving `muted`
set if the call ends without a clean state transition.

The recommended approach is client-side suppression. Server-side `muted`
mutation is an option for deployments where the VTC server has authority over
the Chat account, but it introduces state that must be reliably cleaned up.

### Do-not-disturb coordination

A user who has set `PresenceStatus.presence` to `"busy"` is signaling
do-not-disturb intent. The VTC server SHOULD respect this when deciding whether
to deliver ring notifications: the blocked-sender suppression rule
(`draft-atwood-jmap-vtc-00` Section on Blocked-Sender Suppression) applies to
blocked contacts; a `"busy"` presence does not itself block rings unless your
deployment policy adds that rule.

However, your deployment MAY allow users to set `presence: "busy"` as a
"do not ring me" signal. If you implement this, do so consistently: suppress
ring notifications server-side (by not dispatching `VTCCallPush` for targets
whose `PresenceStatus.presence` is `"busy"`) and document the behavior to
callers.

If a participant is already in an active call (`Chat.activeCallId` is non-null
on one of their bound Chats), the server SHOULD apply the ring-rate-limiting
rules from `draft-atwood-jmap-vtc-00` conservatively: a participant who is
already in one call is generally a poor target for a ring notification from a
second call. Surface the second call as a `VTCRingEvent` on their WebSocket if
connected, but consider suppressing the device-level push that would interrupt
the first call with a full-screen incoming-call UI.

---

## Cross-reference summary

| Feature | VTC spec reference | Chat spec reference |
|---|---|---|
| `chatId` binding on VTCCall | Section 4, Optional Binding Fields | Chat object, `activeCallId` field |
| Auto-creating Chat | VTCCall/set, room call creation | Chat/set, Creating a Group Chat |
| activeCallId banner | Section 4, Chat binding fields | Chat object, `activeCallId` field |
| In-call text messages | Section 3, Chat Delegation | Message/set |
| Read positions and unread counts | (via Chat) | ReadPosition, Chat.unreadCount |
| Caption/reaction ephemeral events | In-Call Ephemeral Events section | ChatStreamEnable, `dataTypes: ["vtc"]` |
| Persistent reactions | (via Chat) | Reaction, Message.reactions map |
| Lobby Chat | VTCParticipant.lobbyState | Chat/set group or channel |
| Presence on join/leave | (server policy) | PresenceStatus/set |
| Notification muting | (server policy) | Chat.muted or client-side |
| Call banner signal | VTCCall.state → active | Chat.activeCallId (server-set) |
