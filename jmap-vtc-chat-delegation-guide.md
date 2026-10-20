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
reactions are carried as Messages in the bound Chat, how lobby communication works,
and how to keep presence state consistent between the VTC and Chat layers.

---

## 2. Binding and creating call Chats

### The chatId binding

`VTCCall` carries an optional `chatId` field
(`draft-atwood-jmap-vtc-00` Section 4, "Optional Binding Fields"). When this
field is set, the Chat identified by that id is the call's dedicated in-call
text channel. Everything that would otherwise require a VTC-native chat
mechanism is handled there instead.

A caller creating a ring call, room call, or scheduled call may supply a
`chatId`. When `chatId` is omitted, the call has no text backchannel
(`draft-atwood-jmap-vtc-00` Section "Relationship to JMAP Chat"); the VTC spec
does not have the server create a Chat on the caller's behalf. To give a call
in-call chat, create or select the Chat first and pass its id as `chatId`. The
caller must be a member of that Chat (`draft-atwood-jmap-vtc-00`, VTCCall/set
create rules).

### Choosing a Chat kind

When creating or selecting a Chat for a call, choose the Chat `kind` based on
the call's context:

- **Ring calls (1:1):** Use the `kind: "direct"` Chat whose `contactId` is
  the `id` of the ChatContact whose `identity` is the target participant's
  userId. If none exists, create one. `Chat/set` create of a direct Chat for a
  contact that already has one fails with `alreadyExists` and an `existingId`
  (`draft-atwood-jmap-chat-00`, Chat/set), so direct chats are never
  duplicated. The call's messages appear alongside the existing conversation
  history.

- **Ring calls (group) and room calls without a Space context:** Create a
  `kind: "group"` Chat whose initial `memberIds` lists the other
  participants: for each, the `id` of the ChatContact whose `identity` is that
  participant's userId. The creator is added automatically and is not listed
  (`draft-atwood-jmap-chat-00`, Chat/set). `name` is required for group
  chats; derive it from the call `subject` when one is set.

- **Room calls and scheduled calls inside a Space:** Use the existing channel
  Chat (`kind: "channel"`) for the Space. Set both `chatId` and `channelId`
  on the VTCCall to that channel's id, plus `spaceId`. `chatId` is what binds
  in-call text and drives `Chat.activeCallId`; `channelId` alone does not. Do
  not create a new Chat; the call is scoped to the channel's existing
  conversation.

- **Ephemeral room calls (huddles, ad-hoc rooms):** Create a `kind: "group"`
  Chat and set `messageExpirySeconds` to a short window (e.g., 86400 seconds)
  if post-call log retention is not desired. This keeps the chat visible during
  and briefly after the call while automatically discarding it.

### Membership sync

When a Chat is created for a call, its initial membership should match the
initial VTCParticipant list. Joining a call that has a `chatId` binding
requires membership in the bound Chat: a non-member gets `notFound`
(`draft-atwood-jmap-vtc-00`, Joining a Room or Scheduled Call). Membership
therefore has to come first:

- Before a user with a `userId` joins, add the `id` of the ChatContact whose
  `identity` is that userId via the `addMembers` patch key on `Chat/set`
  update if they are not already a member. Adding them when `joinedAt` is set
  is too late; the join would already have failed.
- When a VTCParticipant leaves and will not rejoin (i.e., `leftAt` is set and
  the call has ended), the server MAY leave them as a member so they can
  read the post-call chat history, or remove them immediately depending on
  deployment policy.
- Gateway participants (PSTN, SIP, H.323) who have no `userId` cannot be Chat
  members. Do not attempt to add them.
- Lobby participants (discussed in Section 6) are members of the bound Chat
  like anyone else: they need membership to join, and the VTC spec uses the
  bound Chat for lobby communication.

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
Chat client knows which `chatId` corresponds to the current call. The full
`VTCCallPush` ring notification payload already carries `chatId`
(`draft-atwood-jmap-vtc-00` Section on VTCCallPush), so clients receive this
binding at ring time. The content-free form of `VTCCallPush` omits it; in
that case, and at any other time, clients read `VTCCall.chatId` directly.

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

## 4. Live captions

### Captions are Messages

Live captions and transcription are delivered as Messages in the bound Chat,
with `senderId` providing speaker attribution
(`draft-atwood-jmap-vtc-00` Section "Relationship to JMAP Chat"). Neither the
VTC spec nor `draft-atwood-jmap-chat-wss-00` defines an ephemeral event type
for captions. A deployment that wants a separate ephemeral caption stream is
building a deployment-specific extension, not part of the protocol, and
clients of other deployments will not understand it.

The transcription engine itself (Whisper, Vosk, cloud ASR, or other) is
entirely deployment-defined.

### Keeping caption volume manageable

A speech-to-text engine produces high-frequency, rapidly superseded text
fragments. Posting every interim result as a Message would flood the chat
history: hundreds of low-value intermediate results per minute, each requiring
storage, delivery, `StateChange` fan-out, and client-side `Message/changes`
retrieval. Practical patterns:

- Post committed segments (a sentence or a speaker turn), not interim results.
- Coalesce rapid segments from the same speaker on the server before posting.
- Mark caption Messages clearly (e.g., with a `bodyType` or metadata field
  indicating "transcript") so clients can render them in a caption strip and
  distinguish them from participant-typed messages.
- If captions should not outlive the call, plan retention with
  `messageExpirySeconds` at Chat creation time (Section 2).

### Speaker attribution

Message `senderId` is a local id: the `id` of the ChatContact whose
`identity` is the speaker's `VTCParticipant.userId`, or the accountId when the
speaker is the account owner. Use that ChatContact's `displayName` or `login`
field to label the speaker in the caption UI.

For speakers who joined via gateway (no `userId`, so no ChatContact), the
deployment decides how to attribute the caption Message, for example posting
it from a system or bot ChatContact and including the
`VTCParticipant.displayName` in the text.

### The "vtc" dataType is not a caption stream

`draft-atwood-jmap-chat-wss-00` (Section "Enabling Ephemeral Push") defines
`"vtc"` as a companion `dataTypes` value in `ChatStreamEnable`. It is
recognized only when the server advertises `urn:ietf:params:jmap:vtc:websocket`
(and `urn:ietf:params:jmap:chat:websocket`); otherwise it is silently ignored.
It subscribes the connection to the VTC in-call events defined by
`draft-atwood-jmap-vtc-wss-00` (participant, media, and call-state events), not
to captions. Caption Messages arrive through the normal Message path.

### Accessibility considerations

Live captions serve participants who are deaf or hard of hearing, who are in
noisy environments, or who are consuming the call in a language that benefits
from a text parallel. Treat captions as a first-class feature, not an
afterthought. Because captions are Messages in the bound Chat, they are
delivered to every member regardless of whether the participant has enabled
captions in the UI; the client decides whether to render them. They also form
a searchable transcript after the call without any extra step.

---

## 5. In-call reactions

In-call emoji reactions — the "raise hand", "clap", "heart" buttons in
Zoom/Meet — are reactions on Messages in the bound Chat
(`draft-atwood-jmap-vtc-00` Section "Relationship to JMAP Chat"). No draft
defines an ephemeral reaction event.

### Adding a reaction

A participant adds a reaction with a `Message/set` update on the target
Message that patches `reactions/<senderReactionId>` with an `emoji` and a
`sentAt` timestamp; the client supplies a ULID as the `senderReactionId`
(`draft-atwood-jmap-chat-00`, Adding and Removing Reactions). This produces a
permanent record of who reacted to what, visible in the post-call chat
history, and rendered by any JMAP Chat client without VTC-specific logic.

### Float reactions

"Applause" reactions and emoji floats — the visual effect where emojis drift
up the screen when many participants react at once — are not tied to any
particular participant-typed Message. The VTC spec still carries them as
Message reactions; which Message they attach to (for example, the
call-started system message) is a deployment choice. The client renders a
float animation when a new reaction arrives on that Message during the call.

A deployment that wants unpersisted float reactions on the WebSocket is
building a deployment-specific extension, not part of the protocol.

If your deployment wants a summary of reaction activity for post-call
analytics (e.g., "42 clap reactions at 14:23"), derive it from the reaction
records rather than adding new records.

---

## 6. Lobby chat

### The lobby scenario

When `VTCCall.lobbyEnabled` is `true`, participants who join the call enter a
waiting room. Their `VTCParticipant.lobbyState` is `"waiting"` and `joinedAt`
stays `null` until a moderator admits them; rejection sets `leftAt`
(`draft-atwood-jmap-vtc-00`, Joining a Room or Scheduled Call). During this
wait, they may need to communicate with the host — to announce themselves,
clarify their name, or ask when the call will start.

### The bound Chat is the lobby channel

Lobby communication uses the bound Chat: lobby participants can message
moderators before being admitted (`draft-atwood-jmap-vtc-00` Section
"Relationship to JMAP Chat"). No separate lobby Chat is needed. A waiting
participant is already a member of the bound Chat, because joining a
Chat-bound call requires it (Section 2).

A waiting participant is not counted in `activeParticipantCount` and does not
receive VTC in-call events. Chat delivery is unaffected: they see the bound
Chat's Messages like any other member.

The bound Chat's history is visible to waiting participants as Chat members.
If that is not acceptable for a given call, the choice is made when the Chat is
selected or created (Section 2), for example by binding a Chat created for the
call rather than a long-lived one. A separate lobby Chat or per-lobby access
restriction is a deployment-specific extension, not part of the protocol.

### Host broadcast to waiting participants

A moderator wanting to address waiting participants (e.g., "We'll start in 5
minutes") sends a Message to the bound Chat. `BroadcastMention` with
`scope: "everyone"` notifies all members of the Chat, admitted or not.

### Transitioning on admission

When a moderator admits a waiting participant (`lobbyState` transitions from
`"waiting"` to `"admitted"`), the server sets `joinedAt`. No Chat membership
change is needed; the participant was already a member of the bound Chat. The
client switches from its lobby view to the in-call view and keeps showing the
same chat panel.

The `VTCParticipant.lobbyState` transition is the authoritative signal.

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
do-not-disturb intent. A deployment can take this into account when deciding whether
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
| Creating a call Chat | VTCCall/set create (`chatId`) | Chat/set group create; Space/set `addChannels` for channels |
| activeCallId banner | Section 4, Chat binding fields | Chat object, `activeCallId` field |
| In-call text messages | Section 3, Chat Delegation | Message/set |
| Read positions and unread counts | (via Chat) | ReadPosition, Chat.unreadCount |
| Live captions | Relationship to JMAP Chat | Message/set in the bound Chat |
| In-call reactions | Relationship to JMAP Chat | Reaction, Message.reactions map |
| Lobby communication | Relationship to JMAP Chat; VTCParticipant.lobbyState | The bound Chat |
| Presence on join/leave | (server policy) | PresenceStatus/set |
| Notification muting | (server policy) | Chat.muted or client-side |
| Call banner signal | VTCCall.state → active | Chat.activeCallId (server-set) |
