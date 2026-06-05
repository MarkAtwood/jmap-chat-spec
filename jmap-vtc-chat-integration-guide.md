# JMAP VTC — Chat Integration Guide

For JMAP Chat implementers adding video/voice calling via `draft-atwood-jmap-vtc-00`.

Read the VTC draft first. This guide does not re-state normative requirements. It
covers how the two capabilities fit together — the integration surface, the expected
server behaviors, and the UX patterns that fall out of the design.

This guide is non-normative. `draft-atwood-jmap-vtc-00` and `draft-atwood-jmap-chat-00`
are the source of truth. Where this guide and a draft disagree, the draft wins.

---

## How to read this guide

You know JMAP Chat. You are adding calling. The VTC spec is standalone, but it was
designed with Chat in mind: several collaboration features are explicitly delegated to
Chat rather than re-defined in VTC. The integration is a set of optional foreign keys
and a handful of server-side behaviors you must implement on both sides.

The guide follows a flow that mirrors what a user experiences: the server advertises
capabilities, a user starts a call, the call appears as a banner in chat, participants
join, and the call history ends up searchable in the message thread.

---

## 1. Capability co-advertisement

### When to advertise both

Advertise `urn:ietf:params:jmap:vtc` alongside `urn:ietf:params:jmap:chat` when your
deployment includes both a call-state manager and a chat server on the same account.
The two capabilities are independent — a deployment can run either without the other —
but the integration features described in this guide only activate when both are present
on the same account.

### Session object

A session advertising both capabilities looks like this:

```json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": { ... },
    "urn:ietf:params:jmap:chat": {},
    "urn:ietf:params:jmap:vtc": {}
  },
  "accounts": {
    "acct1": {
      "name": "alice@example.com",
      "isPersonal": true,
      "accountCapabilities": {
        "urn:ietf:params:jmap:chat": { ... },
        "urn:ietf:params:jmap:vtc": {
          "mayCreateCall": true,
          "supportsRingCalls": true,
          "supportsRoomCalls": true,
          "supportsScheduledCalls": false,
          "supportsRecording": false,
          "supportsLivestream": false,
          "supportsLobby": true,
          "supportsBreakoutRooms": false,
          "supportedMediaTypes": ["audio", "video"],
          "maxParticipantsPerCall": null,
          "maxConcurrentCalls": null
        }
      }
    }
  }
}
```

The session-level capability value for `urn:ietf:params:jmap:vtc` is always an empty
object `{}`. The substantive per-account fields live under `accountCapabilities`.

### Account-level capability fields your Chat UI needs

Your Chat client needs to read the following VTC account capability fields before
offering calling features:

- `mayCreateCall` — whether this user can start calls at all. Gate the "call" button
  on this.
- `supportsRingCalls` — whether direct-chat ring calls are available.
- `supportsRoomCalls` — whether drop-in room calls are available for channels.
- `supportedMediaTypes` — which media types the deployment supports. Use this to decide
  whether to offer audio-only, video, or screen-share options.

### Feature detection

Do not assume VTC is present because you know your own deployment. Clients MUST check
for `urn:ietf:params:jmap:vtc` in `accountCapabilities` before rendering any calling
UI. This ensures the same client code works against deployments that have not yet
deployed calling, and against accounts on multi-tenant servers where calling is not
enabled for every account.

---

## 2. Starting calls from chat

### The user clicks "call"

When a user initiates a call from a chat UI, the client calls `VTCCall/set` create. The
server creates the VTCCall, assigns a ULID as its id, generates a `joinUri` pointing to
the deployment's media stack, and returns the new VTCCall. The Chat's `activeCallId`
field is set server-side when the call enters the active state (see section 3).

The client does not set `activeCallId` on the Chat. That field is server-set and
read-only to clients.

### Choosing the call model

Two call models apply from a chat context. Use `callType: "ring"` or `callType: "room"`
based on the chat type:

**Direct chat (two participants) → ring call**

A direct chat has one other participant. Use `callType: "ring"`. Pass the other
participant's userId in `targetParticipantIds`. The server creates VTCParticipant
records for both the initiator and the target, dispatches push ring notifications to
the target's devices, and transitions the call to `"ringing"`.

```json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:vtc"],
  "methodCalls": [
    ["VTCCall/set", {
      "accountId": "acct1",
      "create": {
        "newcall1": {
          "callType": "ring",
          "mediaTypes": ["audio", "video"],
          "targetParticipantIds": ["user-bob"],
          "chatId": "chat-alice-bob",
          "subject": null
        }
      }
    }, "r1"]
  ]
}
```

The `chatId` field binds this call to the Chat. The server uses this binding to set
`Chat.activeCallId` when the call becomes active. Pass the Chat's id here.

**Group chat or channel → room call**

A group chat or Space channel has multiple participants and a continuous membership.
Use `callType: "room"`. Room calls transition directly to `"active"` on creation —
there is no ringing phase. Members join when they choose to.

```json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:vtc"],
  "methodCalls": [
    ["VTCCall/set", {
      "accountId": "acct1",
      "create": {
        "newcall1": {
          "callType": "room",
          "mediaTypes": ["audio", "video"],
          "chatId": "chat-general",
          "spaceId": "space-team",
          "channelId": "chat-general",
          "subject": "Standup"
        }
      }
    }, "r1"]
  ]
}
```

Pass `spaceId` and `channelId` in addition to `chatId` when the chat is a Space
channel. These bindings allow the server to apply Space permission checks and allow
`VTCCall/query` filtering by Space.

### Ring calls vs room calls: which to use when

| Chat context | Call type | Reason |
|---|---|---|
| Direct chat, 1:1 | ring | Notifies the other person; they answer or decline |
| Group DM, small | ring or room | Ring if you want to explicitly invite everyone; room if drop-in is acceptable |
| Space channel | room | Channels are persistent; members join when ready |
| Scheduled meeting | scheduled | Use if your deployment supports it; otherwise room |

When the account capability shows `supportsRingCalls: false`, fall back to room calls
for direct chats as well.

---

## 3. activeCallId and the join banner

### What activeCallId is

`Chat.activeCallId` is a `String|null` field on the Chat object, defined in
`draft-atwood-jmap-chat-00`. It is server-set and read-only to clients. The spec says:

> When `urn:ietf:params:jmap:vtc` is present: the id of a VTCCall that is currently
> active and bound to this Chat via the VTCCall's `chatId` field. The server sets this
> when a bound VTCCall enters the `"active"` state and clears it to `null` when the
> call ends. `null` when no call is active or VTC is not available.

The field exists solely to drive the "call in progress — click to join" banner. It is
not a list of all calls ever associated with this Chat; it is the currently active one.

### Server responsibilities

Your server must implement both sides:

**When a VTCCall with a `chatId` binding enters `"active"` state:**
Set `Chat.activeCallId` to the VTCCall's id. This triggers a Chat `StateChange`
notification, which the client picks up via the standard JMAP push or WebSocket path.

**When that VTCCall transitions to `"ended"` state:**
Clear `Chat.activeCallId` to `null`. Again, this triggers a Chat `StateChange`.

The server must do this atomically with the VTCCall state transition. A VTCCall that
has ended but whose Chat still shows `activeCallId` set is a data integrity bug.

For ring calls: `"active"` state is reached when a target participant answers. Until
then, the call is in `"ringing"` state and `activeCallId` is still `null`. This is
intentional: a ringing call is not yet joinable by other chat members.

For room calls: `"active"` state is immediate on creation. `activeCallId` is set as
soon as the room call exists.

### Client polling vs push

Do not poll `Chat/get` to detect `activeCallId` changes. Subscribe to Chat
`StateChange` notifications via the JMAP push mechanism or the WebSocket connection
(`draft-atwood-jmap-chat-wss-00`). When you receive a `StateChange` with `Chat` in
the changed types, fetch the updated Chat object. If `activeCallId` has changed from
`null` to a value, show the join banner. If it has changed to `null`, hide it.

### The join flow

When the user clicks the join banner:

1. Create a VTCParticipant on the referenced call via `VTCParticipant/set create` with
   `callId` set to `Chat.activeCallId` and `joinMethod` set to the client's connection
   type (typically `"webrtc"`).
2. Read the `joinUri` from the VTCCall object: call `VTCCall/get` using `Chat.activeCallId`
   to retrieve the VTCCall, then read its `joinUri` field. The `joinUri` is a field on
   VTCCall, not on VTCParticipant.
3. Open the `joinUri` in the media layer — only after explicit user action.

Do not open `joinUri` automatically. The VTC spec is explicit: clients MUST NOT connect
to `joinUri` without explicit user initiation.

**Ring call answer vs join:** For ring call targets answering their own ring, use
`VTCParticipant/set update` on the existing VTCParticipant record — the server
pre-created it at ring time. The `VTCParticipant/set create` path described here applies
to joining an already-active call as a new participant (e.g., a room call, or a third
party joining a ring call that has already been answered).

### Displaying participant count and names

The VTCCall object includes `activeParticipantCount` (server-set, UnsignedInt). Display
this in the join banner: "3 people in call — click to join".

For names, call `VTCParticipant/query` with a filter of `{"callId": "...", "isActive": true}`.
This returns ids; fetch VTCParticipant records with `VTCParticipant/get` to get
`displayName` values. For the banner, showing the first two or three names plus an
overflow count is sufficient: "Alice, Bob, and 2 others".

---

## 4. Permissions and access control

### The start_call permission

`draft-atwood-jmap-chat-00` defines a permission named `"start_call"` in the
SpaceRole vocabulary. It is one entry in the `permissions` array of a SpaceRole object:

```json
{
  "id": "role-moderator",
  "name": "Moderator",
  "color": "#5865F2",
  "permissions": ["send_messages", "start_call", "mention_broadcast"],
  "position": 10
}
```

This permission gates VTCCall creation for Space-bound chats. Before sending
`VTCCall/set create` with a `spaceId`, your client can pre-flight check whether the
current user holds a role that includes `"start_call"`. However, the server enforces
this independently — a client that skips the pre-flight check and sends the create will
receive `forbidden` if the user lacks the permission.

### What start_call gates and does not gate

The VTC spec clarifies: `"start_call"` gates call creation, not joining. A member who
can view a channel can join an existing call in that channel even without `"start_call"`
permission. The distinction matters for channel-level access control: a read-only
observer role should be able to join an ongoing meeting without being able to start one.

Implement this server-side: when a `VTCParticipant/set create` arrives for a
Space-bound call, check that the caller is a member of the bound Space with at least
`"view"` permission on the channel. Only check `"start_call"` on `VTCCall/set create`.

### Who holds start_call by default

This is a deployment decision. Common patterns:

- **Everyone can call**: Grant `"start_call"` on the `@everyone` implicit role, which
  every Space member holds. This is appropriate for small teams where any member may
  start a standup or 1:1.
- **Only higher roles can call**: Grant `"start_call"` only on roles with elevated
  position. Appropriate for channels that are primarily broadcast (announcements, support
  queues) where uncontrolled call-starting would be disruptive.
- **Channel-level override**: Use ChannelPermission to deny `"start_call"` on specific
  channels while allowing it globally.

The permission vocabulary is extensible. Companion specifications may register
additional names; servers must ignore unrecognized names. If your deployment needs
finer-grained call control (separate permissions for audio-only vs video calls, for
recording, for guest-link creation), you can register and enforce additional names
without waiting for spec amendments.

### Same-server vs federated deployments

When Chat and VTC run on the same server, access control is straightforward: the server
has full visibility into both Space membership and VTCCall state, and enforces
permissions in a single transaction.

When Chat and VTC are on different servers, or when call participants come from federated
Chat deployments, access control becomes significantly more complex. The VTC spec
defers cross-server call state synchronization to a future companion specification.
For now:

- A call's `chatId`, `spaceId`, and `channelId` are same-account references. The server
  can only enforce Chat permissions for local accounts.
- Remote participants in a federated call join via a call invitation delivered through
  the federation channel (see section 5). Their access to the VTCCall object itself
  depends on whether the remote server creates a local VTCCall shadow record.
- For the initial implementation, confine Space-bound calling to same-server members.
  Cross-server calling through federation is an advanced case and should be deferred
  until the federation companion spec exists.

---

## 5. Call invitations via MessageAction

### Passive invitation vs active ring

A ring call actively notifies specific participants via push. A MessageAction in a chat
message is a passive invitation: the message appears in the channel, and participants
join when they see it and choose to. These are two different UX patterns and they compose
naturally.

For a group channel call, send a Message with a MessageAction after creating the room
call. This serves as the visible record of "a call started" and provides a join entry
point for members who were offline when the call started or who join the chat later.

### Constructing the MessageAction

A MessageAction for a VTC call uses `type: "urn:jmap:chat:cap:vtc"`. The `uri` field
carries the `joinUri` from the VTCCall. The `metadata` field carries the `callId` and
optionally a `joinPassword`:

```json
{
  "type": "urn:jmap:chat:cap:vtc",
  "uri": "https://meet.example.com/room/standup-2026",
  "label": "Join standup call",
  "metadata": {
    "protocol": "webrtc",
    "callId": "01JQZK8MRVT4W9N0P3BXHE5FY2",
    "joinPassword": null
  }
}
```

Include this in the `actions` array when creating the Message via `Message/set`.

The `callId` in metadata allows a client that recognizes VTC to fetch the live
VTCCall object and display current participant count alongside the message — a
richer experience than a static "join call" button. Clients that do not recognize
VTC fall back to treating `uri` as a plain link, which is correct graceful degradation.

### Rich link preview showing call status

A client that receives a Message with a `urn:jmap:chat:cap:vtc` MessageAction and
has VTC capability can render a live status card:

1. Extract `callId` from `actions[0].metadata`.
2. Call `VTCCall/get` with that id.
3. If the call is `"active"`, show participant count and a "Join" button.
4. If the call is `"ended"`, show call duration (derived from `createdAt` and
   `endedAt`) and a "Call ended" label. Do not show a join button.
5. If the call is `"pending"` (scheduled), show the `scheduledStartAt` time and a
   "Join when ready" button.

Poll or subscribe to `VTCCall` state changes to keep the card current while the user
is looking at the message.

### How this differs from ring calls

Ring calls push an OS-level notification to specific devices. MessageAction invitations
are passive, in-channel, and visible to all channel members including those who join
later. They are complementary:

- For a 1:1 direct call, use ring. The MessageAction is automatically unnecessary
  because the ring handles notification.
- For a group channel call, create a room call, then send a Message with a MessageAction
  so the call is discoverable in the channel history.
- For a scheduled meeting, send the MessageAction at scheduling time (when the call is
  in `"pending"` state) so members have a persistent join link in the channel.

Do not send a ring-type push for group channel calls. Room calls are drop-in; forcing
a phone-call-style interruption on every channel member when someone starts a standup
is the wrong UX. The join banner (via `activeCallId`) and the MessageAction together
provide sufficient notification without device-level interruption.

---

## 6. Call history in chat

### Why call history matters

A user who was offline when a call happened should see that the call occurred. A user
who missed a ring call should be able to tell it was missed. Call events should be
searchable in the channel's message history alongside regular messages.

The VTC spec does not define a call history mechanism — it tracks call state, not
messaging history. Call history is a Chat concern: it is represented as system messages
in the chat thread.

### When to create system messages

Create a system Message in the bound Chat for the following VTCCall lifecycle events:

**Call started:**
When a VTCCall with a `chatId` binding enters `"active"` state, create a Message in
that Chat. The body text should be something like "Alice started a call." Include a
`urn:jmap:chat:cap:vtc` MessageAction so the system message doubles as a join link
while the call is active.

**Call ended:**
When the VTCCall transitions to `"ended"`, create a Message with the call duration:
"Call ended — 5m 23s." Derive duration from `endedAt - createdAt` (or from the
first participant's `joinedAt` if you want to exclude ringing time). Include
`endReason` in the text when it is informative to the user: "Missed call" for
`endReason: "missed"`, "Call declined" for `endReason: "declined"`.

**Missed ring call:**
When a ring call ends with `endReason: "missed"`, create a Message visible to the
call target: "Missed call from Alice." This is the standard missed-call notification
pattern from mobile messaging apps.

### Message representation

Use a consistent `bodyType` for system messages so clients can render them differently
from user messages. A plain-text representation is the most interoperable choice:
`"text/plain"` with a server-generated body. Clients that want richer rendering can
inspect the `actions` array for the VTC MessageAction and render a card.

There is no dedicated system-message type in `draft-atwood-jmap-chat-00`. Use a
server-assigned `senderId` that identifies your system account (for example, a
well-known userId like `"system@example.com"` or a bot account). Clients can
distinguish system messages from user messages by the `senderId`.

### Call duration

Compute duration from the VTCCall fields:

- For a completed call: `endedAt - createdAt`. This includes ringing time for ring
  calls. If you want media-only duration, use the time from the minimum `joinedAt`
  across all VTCParticipant records for this call to `endedAt` (there is no
  call-level firstJoinedAt field; derive it by querying VTCParticipant records).
- For a missed call: duration is not meaningful; omit it.

Format duration as minutes and seconds for calls under an hour, hours and minutes for
longer calls. "0m 4s" for very short calls; "1h 12m" for long ones.

### Searchability

Because call history is represented as Chat Messages, it is automatically searchable
via `Message/query` using the standard `text` filter. A user searching for "standup
call" will find both the system messages and any user messages that mentioned the call.

For finer-grained call history queries — "show me all calls in this channel in the
last week" — use `VTCCall/query` with a `chatId` filter and a `createdAfter` date
filter. This queries the VTC object store directly rather than the message history.

---

## Quick reference: server checklist

When implementing the Chat-VTC integration, your server must:

- Set `Chat.activeCallId` when a bound VTCCall enters `"active"` state.
- Clear `Chat.activeCallId` to `null` when the bound VTCCall enters `"ended"` state.
- Enforce `"start_call"` permission on `VTCCall/set create` for Space-bound calls.
- Allow `VTCParticipant/set create` (join) for Space channel members with at least
  `"view"` permission, regardless of `"start_call"`.
- Return `notFound` for `VTCCall/get` and `VTCParticipant/get` to users who are not
  members of the bound Chat or Space.
- Create system Messages in the bound Chat for call-started, call-ended, and missed-call
  events.
- Advertise both `urn:ietf:params:jmap:chat` and `urn:ietf:params:jmap:vtc` under
  `accountCapabilities` for accounts that have both capabilities.

When implementing on the client side, your client must:

- Check `urn:ietf:params:jmap:vtc` in `accountCapabilities` before showing calling UI.
- Check `mayCreateCall` before enabling the call button.
- Respond to Chat `StateChange` notifications to show or hide the join banner based
  on `activeCallId`.
- Only open `joinUri` after explicit user action — never automatically.
- Use `callType: "ring"` for 1:1 direct chats and `callType: "room"` for group
  channels (when both are supported; see section 2 for the fallback).
- Include a `urn:jmap:chat:cap:vtc` MessageAction in the Message created when a room
  call starts in a channel.
