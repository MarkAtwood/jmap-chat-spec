# JMAP Unified Client Architecture

A technical architecture specification for a desktop application that is a
client for all JMAP protocols: Mail, Chat, VTC, Scene, Push, FileNode,
Calendars, Tasks, DID, and CID.

---

## 1. Design Philosophy

### 1.1 Capability-Driven Identity

The client is not a fixed product. It is a capability-adaptive shell whose
personality is determined entirely by the capabilities advertised in the JMAP
Session object. The same binary connects to different servers and becomes a
different application:

- A server advertising only `urn:ietf:params:jmap:mail` gives you
  Thunderbird.
- A server advertising only `urn:ietf:params:jmap:chat` gives you Slack.
- Add `urn:ietf:params:jmap:vtc` and you get Slack with integrated Zoom.
- Add `urn:ietf:params:jmap:scene` with `viewHint: "2d-topdown"` and you
  get Gather.
- Add `urn:ietf:params:jmap:scene` with `viewHint: "3d"` and you get
  Mozilla Hubs.
- A server advertising `urn:ietf:params:jmap:mail` and
  `urn:ietf:params:jmap:chat` gives you Outlook.
- A server advertising only `urn:ietf:params:jmap:vtc` gives you a
  standalone calling app.
- A server advertising only `urn:ietf:params:jmap:scene` gives you a
  spatial environment browser.

The client never assumes a capability exists. Every UI surface, every
module initialization, every cross-capability binding is gated on the
Session object. If a capability is absent, the corresponding UI surface
does not render, and the corresponding module does not load.

### 1.2 Progressive Enhancement

Each capability adds UI surface. No capability removes it. The presence of
VTC does not change how Chat renders messages; it adds a call banner, a
"start call" button, and a floating call window. The presence of Scene does
not replace the message list; it adds a spatial viewport that can coexist
with text chat.

### 1.3 The Space as Organizing Principle

When Chat is present, the Space is the primary container for user
experience. A Space holds channels (text), voice channels (VTC), file
trees (FileNode), calendars (Calendars), task lists (Tasks), and spatial
regions (Scene). The sidebar organizes around Spaces; within a Space,
the user navigates channels, files, events, and tasks.

When Mail is present (with or without Chat), the Mailbox tree provides a
parallel organizing hierarchy. Mail and Chat coexist as peer views in
the tab bar (Section 6.6).

When Chat is absent, the organizing principle shifts: VTC organizes around
calls; Scene organizes around regions; Mail organizes around mailboxes.

### 1.4 Server Wins

JMAP's state model is authoritative on the server. The client applies
optimistic updates for responsiveness but defers to the server on
conflict. There is no client-side conflict resolution beyond "display
what the server says."


---

## 2. Session Bootstrap and Capability Discovery

### 2.1 The Session Object as Source of Truth

The JMAP Session object (RFC 8620 Section 2) is fetched at startup and
is the single source of truth for what the client can do. The client
reads:

- `capabilities` (top-level): which protocol extensions the server
  supports at all.
- `accounts[].accountCapabilities`: which capabilities are available
  per account, and their configuration.
- `primaryAccounts`: which account to use by default for each
  capability.

```
GET /.well-known/jmap HTTP/1.1
Host: chat.example.com
Authorization: Bearer <token>
```

Example Session object (abbreviated):

```json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": { "maxSizeUpload": 50000000 },
    "urn:ietf:params:jmap:websocket": {
      "url": "wss://chat.example.com/jmap/ws/",
      "supportsPush": true
    },
    "urn:ietf:params:jmap:mail": {},
    "urn:ietf:params:jmap:submission": {},
    "urn:ietf:params:jmap:chat": {},
    "urn:ietf:params:jmap:chat:websocket": {},
    "urn:ietf:params:jmap:chat:push": {},
    "urn:ietf:params:jmap:vtc": {},
    "urn:ietf:params:jmap:vtc:websocket": {},
    "urn:ietf:params:jmap:scene": {},
    "urn:ietf:params:jmap:scene:websocket": {},
    "urn:ietf:params:jmap:cid": {},
    "urn:ietf:params:jmap:chat:did": {},
    "urn:ietf:params:jmap:chat:filenode": {},
    "urn:ietf:params:jmap:chat:calendars": {},
    "urn:ietf:params:jmap:chat:tasks": {}
  },
  "accounts": {
    "u1": {
      "name": "alice@example.com",
      "accountCapabilities": {
        "urn:ietf:params:jmap:mail": {
          "maxMailboxesPerEmail": null,
          "maxSizeAttachmentsPerEmail": 50000000,
          "mayCreateTopLevelMailbox": true
        },
        "urn:ietf:params:jmap:submission": {
          "maxDelayedSend": 44236800
        },
        "urn:ietf:params:jmap:chat": {
          "maxBodyBytes": 65536,
          "maxAttachmentBytes": 26214400,
          "maxAttachmentsPerMessage": 10,
          "supportedBodyTypes": [
            "text/plain",
            "text/markdown",
            "application/jmap-chat-rich"
          ],
          "supportsThreads": true
        },
        "urn:ietf:params:jmap:vtc": {
          "mayCreateCall": true,
          "supportsRingCalls": true,
          "supportsRoomCalls": true,
          "supportsScheduledCalls": true,
          "supportsRecording": true,
          "supportsLivestream": false,
          "supportsLobby": true,
          "supportsBreakoutRooms": true,
          "supportedMediaTypes": ["audio", "video", "screen"],
          "maxParticipantsPerCall": 100,
          "maxConcurrentCalls": 5
        },
        "urn:ietf:params:jmap:scene": {
          "mayCreateRegion": true,
          "mayCreateObject": true,
          "supportedVisualTypes": [
            "model/gltf-binary",
            "image/png"
          ],
          "maxObjectsPerRegion": 10000,
          "maxAvatarsPerRegion": 50
        }
      }
    }
  },
  "primaryAccounts": {
    "urn:ietf:params:jmap:mail": "u1",
    "urn:ietf:params:jmap:submission": "u1",
    "urn:ietf:params:jmap:chat": "u1",
    "urn:ietf:params:jmap:vtc": "u1",
    "urn:ietf:params:jmap:scene": "u1"
  },
  "apiUrl": "https://chat.example.com/jmap/",
  "uploadUrl": "https://chat.example.com/jmap/upload/{accountId}/",
  "downloadUrl": "https://chat.example.com/jmap/download/{accountId}/{blobId}/{name}?accept={type}",
  "state": "abc123"
}
```

### 2.2 Building the UI Skeleton

On receiving the Session object, the client walks the capability list
and initializes modules:

```
for each capability in session.capabilities:
    module = MODULE_REGISTRY[capability]
    if module is not None:
        module.init(session, account_capabilities)
        register module's UI surfaces
        register module's state change handlers
```

The UI skeleton is assembled from the initialized modules. Modules that
are not initialized produce no UI. The layout engine (Section 6) reads
the set of active modules and selects the appropriate layout.

### 2.3 Multi-Account Support

A single client instance may be connected to multiple JMAP servers or
multiple accounts on the same server. Each account has its own capability
set. The client maintains per-account module state:

```
+------------------+     +------------------+
| Account A        |     | Account B        |
| - Mail           |     | - Mail           |
| - Chat           |     | - Chat           |
| - VTC            |     | - VTC            |
| - Scene          |     |   (no Scene)     |
| - FileNode       |     |   (no FileNode)  |
+------------------+     +------------------+
```

Account switching updates the active module set and re-renders the UI
skeleton. A unified notification stream aggregates across accounts.

### 2.4 Reconnection and Capability Change

The Session object may change during a connection's lifetime. The client
detects this via the `session` state change type in a `StateChange`
notification. On receiving a session state change:

1. Re-fetch the Session object.
2. Diff the capability set against the previous session.
3. Initialize newly advertised modules.
4. Tear down modules for removed capabilities.
5. Re-render the UI skeleton.

This handles dynamic capability changes (e.g., an admin enables VTC on
the server mid-session).


---

## 3. Connection Architecture

### 3.1 Transport Topology

The client maintains a small number of connections per account:

```
+------------------------------------------------------------------+
|                         JMAP Server                               |
+------------------------------------------------------------------+
     ^             ^                ^                  ^
     |             |                |                  |
  HTTP POST     WebSocket        Blob Upload       Simulation
  (API calls)  (bidirectional)   (HTTP PUT)        (per-region)
     |             |                |                  |
+------------------------------------------------------------------+
|                      Client Application                          |
+------------------------------------------------------------------+
```

- **HTTP API endpoint** (`apiUrl`): Used for standard JMAP method calls
  when the WebSocket is unavailable or for initial bulk syncs.
- **WebSocket** (`urn:ietf:params:jmap:websocket` URL): Primary
  bidirectional channel carrying method calls, responses, `StateChange`
  push notifications, and all ephemeral events (typing, presence, VTC
  events, Scene events).
- **Blob upload/download**: Separate HTTP connections per RFC 8620
  Section 6.
- **Simulation layer** (`simulationUri` per SceneRegion): Separate
  connection per active region, protocol defined by deployment.
- **Media layer** (`joinUri` per VTCCall): Separate WebRTC or other
  media connection per active call.

### 3.2 Single WebSocket, Multiple Event Streams

The WebSocket connection is the unified real-time channel. A single
connection carries:

| Frame Type | Source Spec | Direction |
|---|---|---|
| `Request` / `Response` | RFC 8887 | Bidirectional |
| `StateChange` | RFC 8887 | Server-to-client |
| `WebSocketPushEnable/Disable` | RFC 8887 | Client-to-server |
| `ChatStreamEnable/Disable` | Chat WSS | Client-to-server |
| `ChatTypingEvent` | Chat WSS | Server-to-client |
| `ChatPresenceEvent` | Chat WSS | Server-to-client |
| `VTCStreamEnable/Disable` | VTC WSS | Client-to-server |
| `VTCRingEvent` | VTC WSS | Server-to-client |
| `VTCCallEndEvent` | VTC WSS | Server-to-client |
| `VTCParticipantEvent` | VTC WSS | Server-to-client |
| `VTCMediaStateEvent` | VTC WSS | Server-to-client |
| `VTCActiveSpeakerEvent` | VTC WSS | Server-to-client |
| `VTCUnmuteRequestEvent` | VTC WSS | Server-to-client |
| `VTCRecordingStateEvent` | VTC WSS | Server-to-client |
| `VTCGatewaySignal` | VTC WSS | Server-to-client |
| `SceneStreamEnable/Disable` | Scene WSS | Client-to-server |
| `SceneAvatarEvent` | Scene WSS | Server-to-client |
| `SceneObjectEvent` | Scene WSS | Server-to-client |
| `SceneInteractionEvent` | Scene WSS | Server-to-client |

### 3.3 Ephemeral Stream Subscription

On WebSocket connect, the client sends a single `ChatStreamEnable` to
subscribe to all available ephemeral event categories:

```json
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence", "vtc", "scene"],
  "chatIds": null,
  "contactIds": null
}
```

When Chat WSS is not present but VTC WSS is, the client uses
`VTCStreamEnable` directly:

```json
{
  "@type": "VTCStreamEnable",
  "callIds": null,
  "eventTypes": null
}
```

When Chat WSS is not present but Scene WSS is, the client uses
`SceneStreamEnable` directly:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": null,
  "eventTypes": null
}
```

When multiple standalone WSS capabilities are present without Chat WSS,
the client sends both `VTCStreamEnable` and `SceneStreamEnable`
independently on the same WebSocket.

### 3.4 Push Subscriptions for Background Notifications

For mobile and backgrounded desktop clients, the client registers a
`PushSubscription` (RFC 8620 Section 7.2) targeting the platform push
service (APNs, FCM, Web Push). When `urn:ietf:params:jmap:chat:push`
is available, the subscription includes `chatPush` configuration for
inline message content in push payloads, avoiding the wake-fetch-display
round trip.

### 3.5 Connection State Machine

```
                      +--------+
            +-------->| CLOSED |<--------+
            |         +--------+         |
            |              |             |
         auth fail    user connects   fatal error
            |              |             |
            |              v             |
            |       +--------------+     |
            +-------| CONNECTING   |-----+
                    +--------------+
                           |
                      auth success
                           |
                           v
                    +--------------+
           +------->| ACTIVE       |<------+
           |        +--------------+       |
           |              |                |
      reconnected    conn dropped     re-subscribe
           |              |                |
           |              v                |
           |        +--------------+       |
           +--------| RECONNECTING |-------+
                    +--------------+
```

On entering ACTIVE:
1. Enable state-change push via `WebSocketPushEnable` with `pushState`
   from the last known state (catches up on missed changes).
2. Enable ephemeral push via `ChatStreamEnable` (or standalone
   `VTCStreamEnable`/`SceneStreamEnable`).
3. Issue `/changes` for data types whose state tokens are stale.

On entering RECONNECTING:
1. Exponential backoff with jitter: 1s, 2s, 4s, 8s, ... up to 60s.
2. Preserve last known state tokens.
3. On reconnect success, replay from last `pushState`.
4. Re-subscribe ephemeral streams (ephemeral subscriptions are not
   persisted across connections).


---

## 4. Module Architecture

### 4.1 Module Dependency Graph

```
                    +------+
                    | Core |
                    +------+
                  /  |  |   \
                 / +------+  \
                /  | Mail |   \
               /   +------+   \
              /      |    \    \
           +------+ +-----+ +-------+
           | Chat | | VTC | | Scene |
           +------+ +-----+ +-------+
          /  |  |  \    |       |
         /   |  |   \   |       |
        /    |  |    \  |       |
+------+ +--+--+ +---+-+ +-----+
| File | |Cal. | |Tasks| | CID |
| Node | +-----+ +-----+ +-----+
+------+    |
            |
          +-----+
          | DID |
          +-----+
```

Solid dependency rules:
- **Core** is always present (RFC 8620: session, auth, blobs, push).
- **Mail** depends only on Core (RFC 8621, RFC 8620).
- **Chat**, **VTC**, and **Scene** each depend only on Core.
- **FileNode**, **Calendar**, and **Tasks** each require Chat.
- **DID** requires Chat (extends ChatContact).
- **CID** depends only on Core (extends blob upload).
- **Mail** is independent of Chat; they are peer modules that coexist.
- **VTC** optionally binds to Chat (via chatId/spaceId/channelId).
- **VTC** optionally binds to Calendar (via calendarEventId).
- **Scene** optionally binds to Chat (via chatId/spaceId/channelId).
- **Scene** optionally binds to VTC (via activeCallId).

### 4.2 Module Specifications

#### Core Module (always present)

**Capability:** `urn:ietf:params:jmap:core` (RFC 8620)

**Responsibilities:**
- Session object fetch, parse, and monitoring
- Authentication and credential management
- JMAP HTTP client (method calls, batch requests, result references)
- WebSocket lifecycle management (RFC 8887)
- Blob upload and download
- PushSubscription management
- State token storage and change tracking
- Event router: dispatches `StateChange` and ephemeral events to the
  appropriate module

**Data flow:**

```
  Server
    |
    v
+------------------+
| Core Module      |
| - HTTP Client    |------> request queue, response demux
| - WS Manager     |------> frame parser, event router
| - Auth           |------> credential store, token refresh
| - Blob Manager   |------> upload queue, download cache
| - State Tracker  |------> per-type state tokens
| - Push Manager   |------> PushSubscription CRUD
+------------------+
    |
    +---> dispatch to Mail, Chat, VTC, Scene, etc.
```

#### Mail Module

**Capabilities:** `urn:ietf:params:jmap:mail` (RFC 8621),
`urn:ietf:params:jmap:submission` (RFC 8620 Section 7)

**Data types managed:**
- Mailbox (folder hierarchy: Inbox, Sent, Drafts, Trash, custom)
- Email (message headers, body structure, body parts)
- Thread (conversation grouping by `In-Reply-To` / `References`)
- SearchSnippet (server-generated search result highlights)
- Identity (sender identities for composition)
- EmailSubmission (outbound message lifecycle)

**Ephemeral events consumed:** None (Mail is a store-and-forward
protocol; all updates arrive via `StateChange`).

**State change types:** Mailbox, Email, Thread, Identity,
EmailSubmission

**Key behaviors:**
- **Mailbox tree:** Rendered as a navigable tree in the mail tab's
  sidebar. Standard roles (`inbox`, `sent`, `drafts`, `trash`,
  `junk`, `archive`) are mapped to icons; custom mailboxes render by
  name.
- **Thread view:** Emails grouped by Thread ID. The client displays
  threaded conversations with quoted-text collapsing.
- **Composition:** Rich text (HTML) and plain text composition.
  Attachments via blob upload. Identity selection for the `From`
  header. `EmailSubmission/set` submits the message.
- **Search:** `Email/query` with `FilterCondition` (subject, from, to,
  body text, date range, hasAttachment, inMailbox). Server returns
  matching Email IDs; client fetches `SearchSnippet/get` for result
  highlighting.

**No dependency on Chat.** Mail and Chat are independent modules. A
server may advertise either, both, or neither. When both are present,
the tab bar shows Mail and Chat as peer top-level views.

#### Chat Module

**Capability:** `urn:ietf:params:jmap:chat`

**Data types managed:**
- ChatContact (contacts list, presence, endpoints)
- Chat (direct, group, channel conversations)
- Message (message content, attachments, reactions, threads)
- Space (multi-channel containers with RBAC)
- SpaceRole, SpaceMember, SpaceInvite, SpaceBan
- Category, ChannelPermission
- CustomEmoji
- ReadPosition (per-chat read cursor)
- PresenceStatus (owner's self-reported status)

**Embedded types:** Attachment, Endpoint, MessageAction, Mention,
BroadcastMention, MessageRevision, Reaction, ReadDisposition, ChatMember

**Ephemeral events consumed:** ChatTypingEvent, ChatPresenceEvent

**State change types:** Message, Chat, ChatContact, ReadPosition,
PresenceStatus, Space, CustomEmoji, SpaceBan, SpaceInvite

#### VTC Module

**Capability:** `urn:ietf:params:jmap:vtc`

**Data types managed:**
- VTCCall (call lifecycle, state machine, join URI)
- VTCParticipant (per-call participant state, media state)
- VTCRecording (recording metadata, blob references)
- VTCLivestream (outbound stream metadata)

**Embedded types:** VTCMediaState, VTCCallPolicy, VTCGateway,
VTCDialInEntry

**Ephemeral events consumed:** VTCRingEvent, VTCCallEndEvent,
VTCParticipantEvent, VTCMediaStateEvent, VTCActiveSpeakerEvent,
VTCUnmuteRequestEvent, VTCRecordingStateEvent, VTCGatewaySignal

**State change types:** VTCCall, VTCParticipant, VTCRecording,
VTCLivestream

**Optional bindings (when Chat present):**
- `VTCCall.chatId` -> Chat
- `VTCCall.spaceId` -> Space
- `VTCCall.channelId` -> Chat (channel)
- `Chat.activeCallId` -> VTCCall

**Optional bindings (when Calendar present):**
- `VTCCall.calendarEventId` -> CalendarEvent

#### Scene Module

**Capability:** `urn:ietf:params:jmap:scene`

**Data types managed:**
- SceneRegion (bounded spatial environments)
- SceneObject (positioned entities within regions)
- SceneAvatar (user spatial presence)
- SceneAsset (visual/audio resource metadata)

**Embedded types:** SceneBounds

**Ephemeral events consumed:** SceneAvatarEvent, SceneObjectEvent,
SceneInteractionEvent

**State change types:** SceneRegion, SceneObject, SceneAvatar, SceneAsset

**Optional bindings (when Chat present):**
- `SceneRegion.chatId` -> Chat
- `SceneRegion.spaceId` -> Space
- `SceneRegion.channelId` -> Chat (channel)

**Optional bindings (when VTC present):**
- `SceneRegion.activeCallId` -> VTCCall

#### FileNode Module

**Capability:** `urn:ietf:params:jmap:chat:filenode`

**Responsibilities:**
- Space-scoped file tree management
- FileNode CRUD operations
- Attachment-to-FileNode linking (`Attachment.filenodeId`)

**Requires:** Chat module

#### Calendar Module

**Capability:** `urn:ietf:params:jmap:chat:calendars`

**Responsibilities:**
- Space-scoped calendar binding (`Space.calendarId`)
- Calendar event links in messages (MessageAction type
  `urn:jmap:chat:cap:calendar-event`)
- Availability lookups (MessageAction type
  `urn:jmap:chat:cap:availability`)
- RSVP handling within chat

**Requires:** Chat module

#### Task Module

**Capability:** `urn:ietf:params:jmap:chat:tasks`

**Responsibilities:**
- Space-scoped task list binding (`Space.taskListId`)
- Task-chat back-references (`Task.chatId`)
- Task links in messages (MessageAction type
  `urn:jmap:chat:cap:task`)

**Requires:** Chat module

#### Identity Module (DID)

**Capability:** `urn:ietf:params:jmap:chat:did`

**Responsibilities:**
- DID field extension on ChatContact (`ChatContact.did`)
- DID document resolution
- Verified identity badge display

**Requires:** Chat module

#### CID Module

**Capability:** `urn:ietf:params:jmap:cid`

**Responsibilities:**
- sha256 extension on blob upload responses
- Content-addressed deduplication on upload
- Integrity verification on blob download
- Cache validation using sha256 digests

**Requires:** Core module only


---

## 5. State Management

### 5.1 State Token Architecture

Every JMAP data type has a server-assigned state token. The client
stores the latest known state token per data type per account. State
tokens are opaque strings; the client never parses them.

```
+-------------------+-------------------------------------------+
| Data Type         | State Token Location                      |
+-------------------+-------------------------------------------+
| Mailbox           | per-account, updated on folder change     |
| Email             | per-account, updated on new/edit/delete   |
| Thread            | per-account, updated on thread change     |
| Identity          | per-account, updated on identity change   |
| EmailSubmission   | per-account, updated on submission state  |
| Message           | per-account, updated on new/edit/delete   |
| Chat              | per-account, updated on any chat change   |
| ChatContact       | per-account, updated on contact change    |
| ReadPosition      | per-account, updated on read cursor move  |
| PresenceStatus    | per-account, updated on status change     |
| Space             | per-account, updated on space mutation     |
| VTCCall           | per-account, updated on call state change |
| VTCParticipant    | per-account, updated on participant join/ |
|                   |   leave/media change                      |
| SceneRegion       | per-account, updated on region mutation    |
| SceneObject       | per-account, updated on object change     |
| SceneAvatar       | per-account, updated on avatar enter/exit |
| SceneAsset        | per-account, updated on asset change      |
+-------------------+-------------------------------------------+
```

### 5.2 Local Cache Strategy

The client uses SQLite (desktop) or IndexedDB (web) for persistent
local storage. The schema mirrors JMAP data types:

```
+-----------------+-------------------+------------------------+
| Table           | Cache Strategy    | Rationale              |
+-----------------+-------------------+------------------------+
| Mailbox         | Aggressive        | Folder tree rendering  |
| Email           | Aggressive        | Offline reading, fast  |
|                 |                   | scroll, search         |
| Thread          | Aggressive        | Conversation grouping  |
| Identity        | Aggressive        | Compose-from picker    |
| EmailSubmission | Moderate          | Recent outbox only     |
| Message         | Aggressive        | Offline reading, fast  |
|                 |                   | scroll, search         |
| Chat            | Aggressive        | Sidebar rendering,     |
|                 |                   | unread counts          |
| ChatContact     | Aggressive        | Display names, avatars |
|                 |                   | presence fallback      |
| ReadPosition    | Aggressive        | Unread badge accuracy  |
| Space           | Aggressive        | Sidebar structure      |
| VTCCall         | Moderate          | Recent call history    |
| VTCParticipant  | On-demand         | Only while in a call   |
| VTCRecording    | On-demand         | Only when browsing     |
|                 |                   | recordings             |
| SceneRegion     | Moderate          | Region list rendering  |
| SceneObject     | On-demand         | Only while in a region |
| SceneAvatar     | On-demand         | Only while in a region |
| SceneAsset      | On-demand         | Loaded by renderer     |
+-----------------+-------------------+------------------------+
```

"Aggressive" means the client syncs the full data set on first connect
and keeps it current via `/changes`. "Moderate" means the client syncs
a recent window and fetches older data on demand. "On-demand" means the
client fetches only when the user navigates to the relevant view.

### 5.3 Sync Strategy

**First connect:** Full sync via `Foo/get` for aggressively cached types.
For messages, the client fetches a recent window (e.g., the last N
messages per chat) rather than the full history.

**Incremental sync:** On receiving a `StateChange` for data type `Foo`,
the client calls `Foo/changes` with the last known state token.
If the server returns `cannotCalculateChanges`, the client falls back to
a full `Foo/get`.

**Reconnection:** The client sends `WebSocketPushEnable` with the last
known `pushState`. The server delivers all `StateChange` events since
that `pushState`, and the client processes them in order.

### 5.4 Optimistic Updates

For low-latency UX, the client applies mutations locally before the
server confirms:

1. User sends a message. The client inserts a local Message object with
   a client-generated temporary ID and `deliveryState: "pending"`.
2. The JMAP `Message/set` response returns the server-assigned ID and
   confirms the create. The client replaces the temporary object.
3. If the server returns an error, the client marks the message as
   failed and offers retry.

The same pattern applies to reactions, read position updates, and
Chat/set mutations. The rule is always: server wins on conflict.

### 5.5 StateChange Fan-Out

The Core module's event router receives all `StateChange` notifications
from the WebSocket and dispatches them to the appropriate module:

```
WebSocket frame: {"@type": "StateChange", "changed": {
  "u1": {
    "Email": "state-token-9",
    "Message": "state-token-42",
    "VTCCall": "state-token-17"
  }
}}

Event Router:
  -> Mail Module: Email state changed, token "state-token-9"
  -> Chat Module: Message state changed, token "state-token-42"
  -> VTC Module:  VTCCall state changed, token "state-token-17"
```

Each module independently decides whether to issue a `/changes` call
based on its own last-known state token.


---

## 6. UI Architecture

### 6.1 Adaptive Layout Engine

The layout engine reads the set of active modules and selects a layout
template:

```
+---------------------------------------------------------+
| Capabilities Present  | Layout Template                 |
+---------------------------------------------------------+
| Mail only             | Mailbox tree + Email list +     |
|                       |   Reading pane                  |
| Chat only             | Sidebar + Messages + RightPanel |
| Mail + Chat           | Tab bar: [Mail] [Chat] tabs     |
| Chat + VTC            | Above + Call Banner + FloatCall  |
| Chat + VTC + Scene    | Above + Scene Viewport          |
| Chat + Scene (no VTC) | Above + Scene Viewport (no call)|
| Mail + Chat + Scene   | Tab bar: [Mail] [Chat] [Scene]  |
| VTC only              | Call-centric: call list + call   |
| Scene only            | Spatial-first: region browser    |
| VTC + Scene (no Chat) | Spatial with call overlay        |
+---------------------------------------------------------+
```

Layout template selection:

```
active_tabs = []

if mail_module.active:
    active_tabs.append(MailTab)

if chat_module.active:
    tab = ChatTab(SpaceCentricLayout)
    if vtc_module.active:
        tab.layout.enable(CallBanner, FloatingCallWindow, RingOverlay)
    if scene_module.active:
        tab.layout.enable(SceneViewport)
    if filenode_module.active:
        tab.layout.enable(FileBrowser)
    if calendar_module.active:
        tab.layout.enable(CalendarSidebar, EventLinks)
    if task_module.active:
        tab.layout.enable(TaskSidebar, TaskLinks)
    active_tabs.append(tab)

if scene_module.active and not chat_module.active:
    tab = SceneTab(SpatialFirstLayout)
    if vtc_module.active:
        tab.layout.enable(CallOverlay)
    active_tabs.append(tab)

if vtc_module.active and not chat_module.active and not scene_module.active:
    active_tabs.append(VTCTab(CallCentricLayout))

# Each Scene region can also spawn its own tab (Section 6.6)
layout = TabBarLayout(active_tabs) if len(active_tabs) > 1 else active_tabs[0].layout
```

### 6.2 Mail-Centric Navigation (when Mail present, no Chat)

When Mail is the only high-level module, the layout is a classic
three-pane email client:

```
+------------------+---------------------+-----------------------+
|                  |                     |                       |
| Mailbox Tree     | Email List          | Reading Pane          |
|                  |                     |                       |
| [Inbox (3)]      | From: Alice         | Subject: Re: Project  |
| [Drafts]         | Subject: Re: Proj.. | From: Alice <a@ex>    |
| [Sent]           | 10:42 AM            | Date: 2026-06-06      |
| [Trash]          |                     |                       |
| [Archive]        | From: Bob           | Hi Mark,              |
| [Custom...]      | Subject: Meeting    |                       |
|                  | 9:15 AM             | Here are the notes... |
|                  |                     |                       |
| ----             | From: Carol         | [Attachment: spec.pdf]|
| [Account 2]     | Subject: Review     |                       |
|                  | Yesterday           |                       |
+------------------+---------------------+-----------------------+
| [Compose]                                                      |
+----------------------------------------------------------------+
```

**Mailbox tree** (left):
- Hierarchical mailbox tree from `Mailbox/get`
- Standard role mailboxes at the top (Inbox, Drafts, Sent, Trash, Junk)
- Custom mailboxes below
- Unread count badges from `Mailbox.unreadEmails`
- Multi-account: account switcher or unified inbox

**Email list** (center-left):
- `Email/query` with `inMailbox` filter, sorted by `receivedAt`
- Thread grouping: collapsed threads show latest message, expand inline
- Search bar triggers `Email/query` with text/header filters;
  `SearchSnippet/get` for highlighted results
- Swipe/keyboard actions: archive, delete, mark read/unread, move

**Reading pane** (right):
- Full email rendering: HTML body (sanitized), plain text fallback
- Attachment list with download via JMAP blob endpoint
- Reply/Forward/Reply-All compose inline
- Thread view: full conversation with quoted-text collapsing

**Composition:**
- Full-window or split-pane compose
- Identity selection (`Identity/get` for available From addresses)
- Rich text (HTML) and plain text modes
- Attachments via blob upload
- Submit via `EmailSubmission/set`
- Drafts auto-saved via `Email/set` to Drafts mailbox

### 6.3 Space-Centric Navigation (when Chat present)

```
+--------+--------------------+-----------+
|        |                    |           |
| Space  |   Channel View     |  Right    |
| Side-  |                    |  Panel    |
| bar    | +----------------+ |           |
|        | | Message List   | | Members   |
| [S1]   | |                | | Files     |
| [S2]   | |                | | Pinned    |
| [S3]   | |                | | Search    |
|        | |                | |           |
| ----   | +----------------+ |           |
| DMs    | | Compose Bar    | |           |
|        | +----------------+ |           |
| [D1]   |                    |           |
| [D2]   | [Call Banner]      |           |
+--------+--------------------+-----------+
```

**Space sidebar** (left):
- List of joined Spaces, each expandable to show:
  - Text channels (grouped by Category)
  - Voice channels (VTC rooms bound to channels)
  - File browser entry (FileNode)
  - Calendar entry (Calendar)
  - Task list entry (Tasks)
  - Scene regions (Scene)
- Below Spaces: direct messages list
- Below DMs: account switcher

**Channel view** (center):
- Message list with virtualized scrolling
- Compose bar with rich body support, mentions, attachments
- Thread panel (slide-out) when `supportsThreads` is true
- Call banner at top when `Chat.activeCallId` is set

**Right panel** (right):
- Member list with presence indicators and roles
- Pinned messages
- File attachments for current channel
- Search results

### 6.4 Spatial Viewport

The Scene module renders a spatial viewport. The viewport's rendering
mode is determined by the SceneRegion's `viewHint`:

```
+---------------------------------------------------+
| viewHint      | Renderer   | Camera    | Controls |
+---------------------------------------------------+
| "3d"          | Three.js   | Perspec-  | Mouse +  |
|               | or wgpu    | tive      | WASD     |
| "2d-topdown"  | PixiJS or  | Ortho-    | WASD /   |
|               | Canvas2D   | graphic   | arrows   |
| "2d-side"     | PixiJS or  | Ortho-    | Arrows + |
|               | Canvas2D   | graphic   | jump     |
| null          | Three.js   | Perspec-  | Mouse +  |
|               | (fallback) | tive      | WASD     |
| unrecognized  | Three.js   | Perspec-  | Mouse +  |
|               | (fallback) | tive      | WASD     |
+---------------------------------------------------+
```

**Avatar representation:** The local user's avatar visual is loaded from
`SceneAvatar.visualRef` (blob) or the blob referenced by
`SceneAvatar.visualType`. The media type determines the loader:
`model/gltf-binary` uses the glTF loader (mandatory), other types
per `supportedVisualTypes`.

**Object rendering:** Each SceneObject's `visualRef` or `assetUri`
is fetched, decoded per `visualType`, and placed at `[position]` with
`[orientation]` and `[scale]` in the region's coordinate space
(right-handed, Y-up, meters).

**Interaction:** SceneObjects with `interactable: true` respond to
input. The client maps input device events to interaction types:

| Input | Interaction |
|---|---|
| Left click | `"click"` |
| Click + drag | `"grab"` / `"release"` |
| Right click / E key | `"activate"` |
| Custom (reverse-domain) | Plugin-defined |

**Simulation layer:** When `SceneRegion.simulationUri` is non-null, the
client connects to the simulation endpoint for high-frequency position
updates (10+ Hz). Between JMAP snapshots (`SceneAvatar.position`
updates), the client interpolates using simulation data. When
`simulationUri` is null, the client falls back to polling
`SceneAvatar/get` at a lower rate.

### 6.5 Call UI

The VTC module provides call UI surfaces that integrate with the
layout:

**Ring notification (incoming call):**
- OS-native call UI where available (CallKit on macOS/iOS,
  ConnectionService on Android)
- In-app ring overlay with accept/decline buttons
- Caller identity from ChatContact (when Chat present) or
  VTCParticipant.displayName

**In-call view:**

```
+------------------------------------------+
|  [Mute] [Camera] [Screen] [Leave]        |
|                                          |
|  +--------+  +--------+  +--------+     |
|  | Video  |  | Video  |  | Video  |     |
|  | Alice  |  | Bob    |  | Carol  |     |
|  +--------+  +--------+  +--------+     |
|                                          |
|  +--------+                              |
|  | Screen |                              |
|  | Share  |                              |
|  +--------+                              |
|                                          |
|  Participants: 3  |  Recording: ON       |
+------------------------------------------+
```

**Floating picture-in-picture:** When the user navigates away from the
call view (e.g., switches to a different channel), the call UI collapses
to a draggable floating window showing the active speaker or a
participant count.

**Call + Scene integration:** When the user is in a SceneRegion with an
`activeCallId`, the call audio is spatialized based on avatar positions
(see Section 7). The video grid may be replaced or supplemented by
avatar-mounted video textures in the 3D scene.

**Recording/livestream indicators:** When `VTCRecording` or
`VTCLivestream` objects exist for the active call, the UI displays
persistent indicators ("Recording" / "Live").

### 6.6 Tab System and Multi-View

When more than one top-level module is active, the client renders a
**tab bar** across the top of the window. Each tab is an independent
view context with its own layout, state, and rendering pipeline.

```
+------------------------------------------------------------------+
| [Mail]  [Chat]  [Chess Game]  [Doom Arena]         [+]           |
+------------------------------------------------------------------+
|                                                                  |
|   (active tab's content fills the remaining window area)         |
|                                                                  |
+------------------------------------------------------------------+
```

**Tab types:**

| Tab Type | Source | Content |
|---|---|---|
| Mail | Mail module | Mailbox tree + email list + reading pane |
| Chat | Chat module | Space sidebar + channel view + right panel |
| Scene (standalone) | Scene module | Spatial viewport (full-window) |
| Scene (in-Space) | Chat + Scene | Scene viewport within Space context |
| VTC (standalone) | VTC module | Call-centric layout |

**Scene regions as tabs:** Any SceneRegion can be opened in its own
tab via a "Pop Out" action. This allows the user to have a chess game
(`viewHint: "2d-topdown"`) in one tab while playing Doom
(`viewHint: "3d"`) in another, while reading email in a third.

**Tab lifecycle:**
- **Static tabs** (Mail, Chat) are created at startup based on the
  Session capabilities and persist for the session lifetime.
- **Dynamic tabs** (Scene regions, standalone calls) are created on
  user action ("Open in new tab", "Pop out") and closed when the user
  closes them or the underlying object is destroyed.
- Each tab maintains its own scroll position, selection state, and
  (for Scene tabs) its own renderer instance.
- Only the active tab's Scene renderer runs its frame loop. Background
  Scene tabs suspend rendering but maintain WebSocket event processing
  so state stays current.

**Cross-tab awareness:**
- VTC call banners and floating PiP windows render above the tab bar,
  visible regardless of which tab is active.
- OS notifications (new email, new chat message, incoming call) are
  tab-independent.
- Badge counts appear on tab labels: `[Mail (3)]` for unread email,
  `[Chat (5)]` for unread messages.

**Example: Email + Chess + Doom**

A user connected to a server advertising `urn:ietf:params:jmap:mail`,
`urn:ietf:params:jmap:chat`, and `urn:ietf:params:jmap:scene`:

```
Tab bar: [Mail (2)]  [Chat]  [Chess - Board Room]  [Doom - Arena]

Mail tab (active):
+------------------+---------------------+-----------------------+
| Mailbox Tree     | Email List          | Reading Pane          |
| [Inbox (2)]      | From: Alice ...     | (email content)       |
| [Sent]           | From: Bob ...       |                       |
+------------------+---------------------+-----------------------+

Chess tab (background, suspended renderer):
  SceneRegion "Board Room", viewHint: "2d-topdown"
  PixiJS renderer, orthographic camera
  8x8 grid with chess pieces as SceneObjects
  Two SceneAvatars (players) with turn-based interaction

Doom tab (background, suspended renderer):
  SceneRegion "Arena", viewHint: "3d"
  Three.js renderer, perspective camera, WASD controls
  SceneObjects for weapons, items, architecture
  SceneAvatars for other players
  Simulation layer at simulationUri for physics/hit detection
```

### 6.7 Notification System

**OS-native notifications:**
- New email: sender name, subject line snippet
- New chat message: sender name, chat name, body snippet
- Incoming call: VTCRingEvent triggers OS call UI
- Avatar enters region: SceneAvatarEvent with `event: "entered"`
- Mention: elevated urgency for @mentions and broadcast mentions
- Task updates: when task status changes via Task module

**Badge counts:**
- Unread email: sum of `Mailbox.unreadEmails` (or Inbox only)
- Unread chat messages: sum of `Chat.unreadCount` across all chats
- Missed calls: VTCCall objects with `endReason: "missed"`
- Per-Space unread: sum of unread across channels in a Space
- Tab badges: each tab shows its own unread count

**Do-not-disturb:**
- Respects `Chat.muted` and `Chat.muteUntil` per chat
- Global DND mode suppresses all non-critical notifications
- Broadcast mentions override per-chat mute (per spec)
- Ring calls may override DND based on user preference

**Background push (Chat Push spec):**
- `ChatMessagePush` payloads delivered via `PushSubscription`
- Inline body snippet eliminates wake-fetch round trip
- Urgency-aware: `"high"` for mentions, `"normal"` for regular
  messages


---

## 7. Media Pipeline

### 7.1 WebRTC Stack for VTC

The VTC module does not handle media transport directly; the JMAP server
is a call-state database, not a media server. The client connects to the
media layer via `VTCCall.joinUri`. The typical stack:

```
+-----------------------+
| VTC Module (JMAP)     |  -- call state: who, when, mute/unmute
+-----------------------+
         |
    joinUri
         |
         v
+-----------------------+
| Media Layer           |  -- WebRTC, SIP, or other
| - getUserMedia        |
| - RTCPeerConnection   |
| - MediaStream tracks  |
+-----------------------+
         |
         v
+-----------------------+
| Audio/Video Output    |
+-----------------------+
```

The client reports media state back to the JMAP server via
`VTCParticipant/set` (updating `mediaState.audio`, `mediaState.video`,
`mediaState.screen`). The server relays this to other clients via
`VTCMediaStateEvent`.

### 7.2 Audio Graph

The audio pipeline handles two modes:

**Non-spatial (standard call):**
```
Participant A audio --\
Participant B audio ---+--> Stereo Mix --> Speakers
Participant C audio --/
```
Standard stereo mixing. All participants at equal volume (modulo
individual volume controls).

**Spatial (Scene + VTC):**
```
Participant A audio --> PannerNode(posA) --\
Participant B audio --> PannerNode(posB) ---+--> AudioContext --> Speakers
Participant C audio --> PannerNode(posC) --/
```

When the user is in a SceneRegion with an `activeCallId`:
1. For each VTC participant, look up the corresponding SceneAvatar by
   `userId`.
2. Set the PannerNode position to the SceneAvatar's `[x, y, z]`
   position.
3. Set the AudioListener position to the local user's SceneAvatar
   position and orientation.
4. Distance attenuation model derived from `SceneRegion.environment`
   (deployment-defined; the client applies a reasonable default if the
   environment schema is unrecognized).

The Web Audio API's PannerNode handles HRTF-based spatialization.
Position updates come from the simulation layer at 10+ Hz; the audio
graph updates at the same rate.

### 7.3 Video Rendering

**Non-spatial:** Video elements in a CSS grid layout. Active speaker
detection via `VTCActiveSpeakerEvent` drives layout priority (active
speaker gets the largest tile).

**Spatial:** Video frames rendered as textures on SceneAvatar objects in
the 3D scene. Each participant's video track is mapped to a quad or
billboard positioned at the avatar's location. This requires the 3D
renderer to accept video textures (Three.js `VideoTexture` or
equivalent).

### 7.4 Screen Sharing

**Non-spatial:** Screen share is a video track in the WebRTC connection,
rendered as a large tile in the call grid or a separate full-window
view.

**Spatial (Scene + VTC):** Screen share can be rendered as a SceneObject
texture -- a virtual "screen" placed in the 3D scene. The SceneObject's
`visualRef` points to a dynamically-updated texture fed by the screen
share track.

### 7.5 Recording Playback

`VTCRecording` objects in the `"available"` state have a `blobId`.
The client downloads the blob via the JMAP download endpoint and plays
it in a standard media player. The `mediaType` field (`"video/webm"`,
`"video/mp4"`, etc.) determines the codec.


---

## 8. Rendering Pipeline (Scene)

### 8.1 Asset Loading

```
SceneObject.visualRef (blobId)
    |
    +--> Is assetUri present?
    |       |
    |      YES --> Fetch from CDN (assetUri)
    |       |         |
    |      NO  --> Fetch from JMAP blob download endpoint
    |                 |
    +------+----------+
           |
           v
    CID verification (if urn:ietf:params:jmap:cid present):
      compute sha256 of downloaded content
      compare with SceneAsset.sha256 or Attachment.sha256
      reject on mismatch
           |
           v
    Decode by mediaType:
      "model/gltf-binary" --> glTF loader (mandatory)
      "image/png"         --> texture loader
      other               --> per supportedVisualTypes
           |
           v
    Insert into scene graph at [position, orientation, scale]
```

### 8.2 Renderer Selection

The client selects a renderer based on the SceneRegion's `viewHint`:

```
match viewHint:
    "3d"          -> Three.js (or Babylon.js, or native wgpu)
    "2d-topdown"  -> PixiJS (or Canvas 2D)
    "2d-side"     -> PixiJS (or Canvas 2D)
    null          -> Three.js (default)
    unrecognized  -> Three.js (fallback, with warning log)
```

**3D renderer (Three.js / Babylon.js / wgpu):**
- Perspective camera
- glTF 2.0 as the mandatory visual format
- PBR materials, skeletal animation
- Scene graph hierarchy matching SceneObject parent-child relationships

**2D renderer (PixiJS / Canvas 2D):**
- Orthographic camera (top-down) or side-scrolling camera
- Sprites or simplified meshes from visual assets
- Grid-based or free-form positioning depending on deployment

### 8.3 Simulation Client

When `SceneRegion.simulationUri` is non-null:

```
+--------------------------------------------------+
|  JMAP Server                                     |
|  (SceneObject/get, SceneAvatar/get -- snapshots) |
+--------------------------------------------------+
      |                              ^
      | state snapshots              | position commits
      | (1-5 Hz)                     | (periodic)
      v                              |
+--------------------------------------------------+
|  Client Scene Module                             |
|  - Interpolates between snapshots                |
|  - Applies simulation updates in real-time       |
+--------------------------------------------------+
      |                              ^
      | simulation frames            | local input
      | (10-60 Hz)                   | (movement, actions)
      v                              |
+--------------------------------------------------+
|  Simulation Layer (simulationUri)                |
|  - Real-time position sync                       |
|  - Physics (if applicable)                       |
|  - Protocol: deployment-defined                  |
+--------------------------------------------------+
```

The client maintains two data streams:
1. **JMAP snapshots:** Low-frequency, authoritative position data from
   `SceneAvatar/get` and `SceneObject/get`. Used for reconciliation.
2. **Simulation stream:** High-frequency position updates from the
   simulation layer. Used for rendering.

Between simulation frames, the renderer interpolates. When a JMAP
snapshot arrives and disagrees with the simulation state, the simulation
state is authoritative for rendering (it is more recent), but the JMAP
state is authoritative for persistence.

### 8.4 Visibility and Spatial Queries

The server handles the visibility contract: the client requests objects
via `SceneObject/query` with spatial filters (e.g., `withinRadius`),
and the server returns only the objects the client is authorized to see
within the query bounds. The client does not need to implement
server-side visibility culling.

Client-side culling (frustum culling, occlusion culling) is a
rendering optimization and is the client's responsibility.

### 8.5 Level of Detail

Level-of-detail is a client-side rendering concern:
- Reduce polygon count for distant objects
- Use sprite impostors for very distant avatars
- Skip animation updates for off-screen objects
- Adaptive quality based on frame rate

### 8.6 Frame Budget

Scene rendering must not starve the rest of the UI. The rendering loop
should target a frame budget (e.g., 16ms for 60 FPS) and yield to the
main thread between frames. Chat message rendering, VTC control
updates, and notification handling take priority over scene frame
rendering when the frame budget is exceeded.

On web-based runtimes (Electron, Tauri webview), use
`requestAnimationFrame` and avoid blocking the main thread. Heavy
asset decoding (glTF parsing, texture decompression) should run in Web
Workers or off-main-thread.


---

## 9. Offline and Background Behavior

### 9.1 Offline Cache

When the network is unavailable, the client operates from the local
cache:

- **Email:** Read cached emails. Compose and queue drafts for send on
  reconnect. Mailbox tree navigable from cache.
- **Chat messages:** Read cached messages. Compose queued for send on
  reconnect.
- **Contacts:** Display cached names, avatars, and last-known presence.
- **Read positions:** Apply locally. Sync to server on reconnect.
- **Spaces and channels:** Navigate cached structure. New channels
  appear on reconnect.

Data that is not cached (VTCParticipant, SceneObject) is unavailable
offline. The UI shows placeholder states ("Reconnecting...").

### 9.2 Background Sync

When the app is backgrounded (desktop: minimized; mobile: suspended):

1. **PushSubscription** keeps the server informed of the push endpoint.
2. **ChatMessagePush** payloads arrive via the platform push service,
   containing inline message content for immediate notification display.
3. **VTCRingEvent** arrives via push with `urgency: "high"`, triggering
   OS-level call UI (CallKit/ConnectionService).
4. The WebSocket connection may be dropped by the OS. On foregrounding,
   the client reconnects and replays from `pushState`.

### 9.3 Incoming Call While Backgrounded

The most latency-sensitive scenario:

```
Server                   Push Service          Client (background)
  |                          |                       |
  |-- VTCRingEvent push ---->|                       |
  |                          |-- OS push delivery -->|
  |                          |                       |-- OS call UI
  |                          |                       |   (ring, accept/
  |                          |                       |    decline)
  |                          |                       |
  |                          |                 [User accepts]
  |                          |                       |
  |<----- VTCParticipant/set (callResponse: "accepted") -----|
  |                          |                       |
  |                          |                       |-- Connect to
  |                          |                       |   joinUri
```

The push payload must reach the device within 2 seconds for acceptable
UX. The `urgency: "high"` header on the Web Push message ensures
priority delivery.

### 9.4 Reconnection Protocol

On reconnect after a disconnect:

1. Re-authenticate (refresh token if expired).
2. Fetch Session object to check for capability changes.
3. Open WebSocket.
4. Send `WebSocketPushEnable` with last `pushState`.
5. Process queued `StateChange` events.
6. Send `ChatStreamEnable` to re-subscribe ephemeral streams.
7. Flush queued local mutations (pending messages, read positions).

Exponential backoff: 1s, 2s, 4s, 8s, 16s, 32s, 60s max. Add random
jitter of 0-1s to avoid thundering herd on server recovery.

### 9.5 Scene Offline

When the simulation layer is disconnected:
- Show the last-known region state statically (frozen avatars, static
  objects).
- Display a "Reconnecting to simulation..." indicator.
- On reconnect, re-establish the simulation connection and snap to
  current state.
- JMAP-level scene data (regions, objects) remains available from cache
  even when the simulation is down.


---

## 10. Cross-Capability Integration Points

This section enumerates every place where two or more capabilities
interact in the client UI.

### 10.1 Mail + Chat

| Integration Point | Mechanism |
|---|---|
| Unified notification stream | Both modules feed into the same notification system; badge counts aggregate across Mail and Chat |
| Contact correlation | ChatContact display names and avatars are used when rendering email from/to addresses that match a known ChatContact |
| Shared blob storage | Email attachments and Chat attachments share the same JMAP blob infrastructure; CID deduplication applies across both |
| Tab coexistence | Mail and Chat occupy peer tabs in the tab bar; switching tabs preserves independent state |
| Link sharing | An email can contain a link to a Chat channel or Space (deployment-defined URI scheme); the client can deep-link from the Mail tab to the Chat tab |

### 10.2 Chat + VTC

| Integration Point | Mechanism |
|---|---|
| Active call banner in channel | `Chat.activeCallId` is non-null; client renders "Join Call" banner above message list |
| "Start call" button | Channel UI includes call button when VTC is present and user has `"start_call"` permission; creates VTCCall with `chatId`/`spaceId`/`channelId` set |
| Call history in chat | VTCCall objects with matching `chatId` are displayed as system messages in the channel timeline |
| In-call chat | The Chat bound via `VTCCall.chatId` serves as the in-call text chat; messages sent to that Chat appear in the call's chat sidebar |
| In-call reactions | Reactions on Messages in the bound Chat render as emoji float animations in the call view |
| Ring notification context | Incoming VTCRingEvent displays caller's ChatContact.displayName and avatar |
| VTC endpoints on contacts | ChatContact.endpoints with `type: "urn:jmap:chat:cap:vtc"` provide direct-call shortcuts on the contact profile |

### 10.3 Chat + Scene

| Integration Point | Mechanism |
|---|---|
| Scene viewport in Space | SceneRegion with `spaceId` set renders a viewport tab or panel within the Space navigation |
| Chat overlay in Scene | When in a SceneRegion with `chatId`, the chat for that region is rendered as a semi-transparent overlay on the scene viewport |
| Avatar display name | SceneAvatar.displayName falls back to ChatContact.displayName for the same userId |
| Presence in Scene | ChatContact.presence is displayed on SceneAvatar nametags |
| Proximity chat | Messages in the region-bound Chat appear as speech bubbles above avatars in 2D views, or as floating text in 3D views |

### 10.4 VTC + Scene

| Integration Point | Mechanism |
|---|---|
| Spatial audio | When SceneRegion.activeCallId is set, VTC audio is spatialized using SceneAvatar positions as PannerNode coordinates |
| Screen share as SceneObject | A screen-sharing participant's video track can be rendered as a texture on a SceneObject (a virtual screen in the 3D scene) |
| Participant-avatar binding | VTCParticipant.userId matches SceneAvatar.userId; the call participant list and avatar list are correlated |
| Avatar-follows-participant | When a VTC participant joins/leaves, the corresponding SceneAvatar enters/exits the region |
| Video on avatar | Participant video tracks rendered as textures on avatar model billboards |

### 10.5 Chat + FileNode

| Integration Point | Mechanism |
|---|---|
| File browser in Space sidebar | FileNode tree for the Space's file storage namespace renders as a sidebar panel |
| Drag-and-drop upload | Dropping a file on the compose bar uploads a blob, creates a FileNode entry, and attaches it to the message with `Attachment.filenodeId` set |
| File links in messages | Attachments with `filenodeId` render as clickable links to the file in the Space's file browser |
| File search | Search within a Space searches both messages and FileNode entries |

### 10.6 Chat + Calendar

| Integration Point | Mechanism |
|---|---|
| Event links in messages | MessageAction with `type: "urn:jmap:chat:cap:calendar-event"` renders as a rich calendar event card in the message |
| "Join meeting" action | Calendar events bound to VTCCalls (via `calendarEventId`) render a "Join" button |
| Upcoming events in sidebar | When Calendar is present, the Space sidebar shows upcoming events from `Space.calendarId` |
| RSVP in chat | Calendar event MessageActions include RSVP buttons rendered inline |
| Availability lookup | MessageAction with `type: "urn:jmap:chat:cap:availability"` renders a scheduling widget |

### 10.7 Chat + Tasks

| Integration Point | Mechanism |
|---|---|
| Task links in messages | MessageAction with `type: "urn:jmap:chat:cap:task"` renders as a task card with status, assignee, and due date |
| Task list in sidebar | When Tasks is present, the Space sidebar shows the task list from `Space.taskListId` |
| Task status updates | Changes to task status can generate system messages in the bound Chat (`Task.chatId`) |
| Create task from message | Right-click context menu on a message offers "Create Task" which creates a Task with `chatId` set to the current Chat |

### 10.8 VTC + Calendar

| Integration Point | Mechanism |
|---|---|
| Scheduled call from event | VTCCall with `calendarEventId` set shows the event title and time in the call card |
| Join button on calendar | Calendar event cards with a bound VTCCall render a "Join" button that connects to `joinUri` |
| Upcoming calls | The call list shows scheduled VTCCalls sorted by `scheduledStartAt`, cross-referenced with CalendarEvent details |

### 10.9 Scene + FileNode

| Integration Point | Mechanism |
|---|---|
| Asset storage | SceneAsset blobs stored as FileNode entries in the Space's file namespace |
| sha256 dedup | Assets with identical sha256 hashes share a single blob (CID integration) |
| Asset browser | The file browser shows scene assets alongside other files |

### 10.10 CID Everywhere

| Integration Point | Mechanism |
|---|---|
| Upload dedup | On blob upload, if the server returns a sha256 matching an existing blob, the client can skip the upload |
| Download verification | On blob download, the client computes sha256 and compares with the stored digest; rejects on mismatch |
| Cache validation | Local blob cache is keyed by sha256; cache hits avoid re-download |
| Scene asset integrity | SceneAsset.sha256 verified on asset load |
| Attachment integrity | Attachment.sha256 verified on download |

### 10.11 DID + Contacts

| Integration Point | Mechanism |
|---|---|
| Verified identity badge | ChatContacts with a `did` field display a verified badge on their display name |
| DID resolution | The client resolves the DID document to verify the association between the ChatContact identity and the DID |
| Federation identity | In federated deployments, DID provides cryptographic identity verification across server boundaries |


---

## 11. Tech Stack Evaluation

### Option A: Electron + Web Stack

**Architecture:** Chromium process for rendering, Node.js for backend
logic, IPC bridge between renderer and main process.

**Rendering stack:**
- Three.js for 3D Scene rendering
- PixiJS for 2D Scene rendering
- WebRTC built into Chromium
- Web Audio API for spatial audio

**Pros:**
- Fastest path to a working prototype
- Largest ecosystem of web libraries
- Web developer skills transfer directly
- Single codebase for desktop and (with adaptation) web deployment
- WebRTC is battle-tested in Chromium

**Cons:**
- Memory-heavy: 200-400 MB baseline before application code
- Large binary: 150-250 MB distribution
- Chromium security surface area
- Two process model (main + renderer) adds IPC complexity
- Update cadence tied to Chromium releases

### Option B: Tauri + Web Frontend

**Architecture:** System webview (WebKit on macOS/Linux, WebView2 on
Windows) for rendering, Rust backend for network, storage, and
computation.

**Rendering stack:**
- Same web stack as Electron (Three.js, PixiJS, WebRTC, Web Audio)
- Rust backend handles JMAP HTTP, WebSocket, SQLite, blob cache

**Pros:**
- Binary size: 5-15 MB (vs 150-250 MB for Electron)
- Memory: 50-100 MB baseline (system webview, no bundled Chromium)
- Rust backend: memory-safe, fast, good for crypto and network
- IPC via Tauri commands is ergonomic
- Active development community

**Cons:**
- WebView inconsistencies across platforms (WebKit vs WebView2 vs
  WebKitGTK)
- WebRTC availability varies by webview (WebView2 has it, WebKitGTK
  may not)
- Less mature than Electron for production desktop apps
- Debugging across Rust/web boundary is harder

### Option C: Native + Embedded Renderer

**Architecture:** Qt or GTK for UI shell, embedded wgpu or Vulkan
context for Scene rendering, native WebRTC library (libwebrtc or
pion-based).

**Rendering stack:**
- Native 3D: wgpu with custom glTF loader
- Native 2D: Skia or Cairo
- Native WebRTC: libwebrtc or GStreamer WebRTC
- Native audio: platform audio APIs + custom spatial mixer

**Pros:**
- Best performance ceiling
- Smallest memory footprint
- Best OS integration (native notifications, system tray, accessibility)
- No web engine dependency

**Cons:**
- Highest development cost (3-5x vs web-based)
- Platform-specific code for each OS
- Smaller talent pool (C++/Rust UI developers)
- Must build or integrate a glTF loader, text rendering, layout engine
- WebRTC without a browser is operationally complex

### Option D: Flutter + Custom Renderers

**Architecture:** Dart/Flutter for UI, platform channels to native
code for WebRTC and Scene rendering.

**Rendering stack:**
- Flutter's Skia-based renderer for 2D UI
- Platform channels to native 3D renderer for Scene
- Platform channels to native WebRTC library for VTC
- Custom audio pipeline via platform channels

**Pros:**
- Single codebase for desktop and mobile
- Good rendering performance (60 FPS UI)
- Hot reload for rapid development
- Material Design and Cupertino widget libraries

**Cons:**
- WebRTC via platform channels adds latency and complexity
- 3D rendering requires a custom engine integration (no Three.js)
- Platform channels are a serialization bottleneck for real-time data
- Flutter desktop is less mature than Flutter mobile
- Limited web ecosystem access

### Recommendation: Tauri + Web Frontend with Optional Native Scene Renderer

```
+---------------------------------------------------------------+
|                       Tauri Shell                             |
|  +----------------------------------------------------------+|
|  |                   Rust Backend                            ||
|  |  +----------+  +-----------+  +----------+  +----------+ ||
|  |  | JMAP     |  | WebSocket |  | SQLite   |  | Blob     | ||
|  |  | HTTP     |  | Manager   |  | Cache    |  | Cache    | ||
|  |  | Client   |  |           |  |          |  |          | ||
|  |  +----------+  +-----------+  +----------+  +----------+ ||
|  |  +----------+  +-----------+  +----------+               ||
|  |  | Simula-  |  | Crypto    |  | Push     |               ||
|  |  | tion     |  | (sha256,  |  | Manager  |               ||
|  |  | Client   |  |  DID)     |  |          |               ||
|  |  +----------+  +-----------+  +----------+               ||
|  +----------------------------------------------------------+||
|  +----------------------------------------------------------+||
|  |                   Web Frontend (Webview)                  ||
|  |  +----------+  +-----------+  +----------+  +----------+ ||
|  |  | React/   |  | Three.js  |  | PixiJS   |  | WebRTC   | ||
|  |  | Solid/   |  | (3D       |  | (2D      |  | (Media   | ||
|  |  | Svelte   |  |  Scene)   |  |  Scene)  |  |  Layer)  | ||
|  |  +----------+  +-----------+  +----------+  +----------+ ||
|  |  +----------+  +-----------+  +----------+               ||
|  |  | Web      |  | State     |  | UI       |               ||
|  |  | Audio    |  | Manage-   |  | Layout   |               ||
|  |  | API      |  | ment      |  | Engine   |               ||
|  |  +----------+  +-----------+  +----------+               ||
|  +----------------------------------------------------------+||
+---------------------------------------------------------------+
```

**Division of responsibility:**

| Layer | Responsibilities |
|---|---|
| Rust backend | JMAP HTTP client, WebSocket management, authentication, SQLite local cache, blob cache with sha256 verification, simulation layer client (connecting to `simulationUri`), DID resolution, push subscription management |
| Web frontend | UI rendering (Mail, Chat, VTC controls, Calendar, Tasks, Files), Three.js for 3D Scene, PixiJS for 2D Scene, WebRTC via browser APIs, Web Audio for spatial audio, tab management, state management and event routing |
| IPC (Tauri commands) | Structured commands for JMAP method calls, blob operations, cache queries, state token management |

**Why Tauri over Electron:**
- 10-30x smaller binary
- 3-5x lower baseline memory
- Rust backend is a natural fit for the JMAP client (HTTP, WebSocket,
  crypto, cache) and avoids Node.js's single-threaded limitations
- WebView2 on Windows and WebKit on macOS provide adequate WebRTC and
  Web Audio support for the media pipeline

**Optional native Scene renderer:**
For game-oriented deployments where Scene is the primary capability and
performance matters, the Rust backend can host a wgpu-based renderer
in a separate native window or embedded surface, bypassing the webview
for scene rendering while keeping the chat/VTC UI in the webview. This
is an optimization, not a requirement.


---

## 12. Security Architecture

### 12.1 Transport Security

All connections use TLS:
- JMAP API endpoint: HTTPS
- WebSocket: WSS
- Blob upload/download: HTTPS
- `simulationUri`: WSS or other TLS-protected transport
- `assetUri` (CDN): HTTPS
- `joinUri` (media): SRTP/DTLS (WebRTC) or TLS (other)

The client rejects non-TLS connections. Certificate validation follows
platform defaults. Certificate pinning is a deployment option, not a
client requirement.

### 12.2 Credential Storage

| Platform | Storage |
|---|---|
| macOS | Keychain Services |
| Windows | Windows Credential Manager |
| Linux | Secret Service API (via libsecret) |

The Rust backend handles credential storage via platform-native APIs.
Credentials never touch the webview's localStorage or IndexedDB.

### 12.3 Session Token Handling

- Tokens are held in Rust backend memory only.
- Never logged (not in debug logs, not in crash reports).
- Never written to disk in plaintext.
- Never passed to the webview except as needed for blob download URLs
  (and those are short-lived, scoped URLs).
- Token refresh handled by the Rust backend; the webview is not aware
  of credential rotation.

### 12.4 Blob Integrity

When `urn:ietf:params:jmap:cid` is present:
1. On upload, the server returns `sha256` in the upload response. The
   client stores this alongside the `blobId`.
2. On download, the client computes sha256 of the received content and
   compares with the stored digest.
3. On mismatch, the client rejects the content and alerts the user.
4. Scene assets, message attachments, and FileNode entries all
   participate in this verification.

### 12.5 Content Sandboxing

**Scene assets:** Visual assets (glTF, images, audio) are parsed in the
renderer's sandboxed context. No script execution from scene data.
glTF extensions that reference executable content are ignored. The glTF
loader strips or ignores `KHR_techniques_webgl` and similar extensions
that could execute shader code from untrusted sources.

**Rich body content:** Messages with `bodyType:
"application/jmap-chat-rich"` are sanitized before rendering. The
rich body format defines a fixed set of span types; unknown span types
are rendered as plain text. HTML injection is not possible because the
format is not HTML.

**Custom properties:** `SceneObject.customProperties` and
`SceneAvatar.customProperties` are opaque JSON. The client renders known
keys for the deployment's schema and ignores unknown keys. No key value
is interpreted as executable.

### 12.6 Input Validation

- All URIs from the server (`joinUri`, `simulationUri`, `assetUri`,
  Endpoint URIs, MessageAction URIs) are treated as untrusted.
- The client validates URI scheme (must be `https:`, `wss:`, or other
  expected scheme) before connecting.
- No automatic navigation to URIs without user action.
- MessageAction types: clients do not act on MessageActions
  automatically; all OOB actions require explicit user initiation.

### 12.7 Blocked Contacts

Client-side enforcement as defense-in-depth:
- Messages from blocked contacts (`ChatContact.blocked: true`) are
  filtered from the message list even if the server delivers them.
- Typing events from blocked contacts are dropped.
- Presence events from blocked contacts are dropped.
- The server is expected to filter these server-side as well; the
  client filtering is a belt-and-suspenders measure.


---

## 13. Accessibility

### 13.1 Screen Reader Support

All text-based UI surfaces (Chat, Calendar, Tasks, File Browser,
Settings) are built with standard web accessibility patterns:

- Semantic HTML (or framework equivalents): headings, lists, buttons,
  landmarks
- ARIA roles and labels for custom widgets (message list, compose bar,
  emoji picker)
- Live regions (`aria-live`) for incoming messages, typing indicators,
  and notifications
- Focus management: new message notification moves focus to the message
  list only when the user has configured this preference

### 13.2 Keyboard Navigation

**Tab bar:**
- Ctrl+1..9 switches to tab by position
- Ctrl+Tab / Ctrl+Shift+Tab cycles tabs
- Ctrl+W closes dynamic tabs (Scene regions, calls)

**Mail UI:**
- Tab/Shift-Tab navigates between mailbox tree, email list, and
  reading pane
- Arrow keys navigate within the email list
- Enter opens an email in the reading pane
- R/A/F for Reply/Reply-All/Forward
- Delete/Backspace moves to Trash
- N for compose new email

**Chat UI:**
- Tab/Shift-Tab navigates between sidebar, message list, compose bar,
  and right panel
- Arrow keys navigate within lists (Space list, channel list, message
  list)
- Enter opens/activates the focused item
- Escape closes modals, thread panel, and right panel

**Spatial views:**
- Arrow keys or WASD move the avatar in 2D views
- Arrow keys rotate the camera in 3D views
- Tab cycles between interactive objects
- Enter/Space activates the focused object
- Escape exits the scene viewport back to the chat view

**Call UI:**
- Keyboard shortcuts for mute (M), camera toggle (V), screen share (S),
  leave call (L), raise hand (H)
- Tab navigation through participant list and controls

### 13.3 Audio Descriptions for Scene Events

When a screen reader is active, the Scene module generates audio
descriptions for spatial events:

- "Alice entered the region" (SceneAvatarEvent, event: "entered")
- "Bob left the region" (SceneAvatarEvent, event: "left")
- "Object 'Whiteboard' was placed" (SceneObjectEvent, event: "created")
- "You interacted with 'Door'" (SceneInteractionEvent)

These descriptions are delivered via ARIA live regions and are also
available as synthesized speech.

### 13.4 Visual Accessibility

- **High contrast mode:** 2D spatial views use high-contrast color
  schemes for avatars and objects. 3D views adjust material colors for
  contrast.
- **Reduced motion:** Disable avatar animations, simplify scene
  transitions, freeze particle effects. Respects the OS-level
  `prefers-reduced-motion` media query.
- **Text scaling:** All text UI respects the system font size setting.
  Scene nametags scale with accessibility font size.
- **Color-blind safe:** Role colors and presence indicators use patterns
  or shapes in addition to color (e.g., presence dots use shapes:
  circle for online, minus for away, X for busy).

### 13.5 Caption Support for VTC

When captions are available via the Chat bound to a VTCCall, the client
renders a caption overlay:

- Captions positioned at the bottom of the call view (configurable)
- Speaker attribution from `senderId`
- Font size adjustable independently of system font size
- Background contrast for readability


---

## 14. Extension Points

### 14.1 Custom MessageAction Types

MessageAction uses an extensible URI namespace for `type`. Third-party
integrations can define custom types using reverse-domain notation:

```json
{
  "type": "com.example.poll",
  "uri": "https://polls.example.com/vote/abc123",
  "label": "Vote in poll",
  "metadata": {
    "question": "What should we have for lunch?",
    "options": ["Pizza", "Sushi", "Tacos"]
  }
}
```

The client's plugin system can register renderers for custom
MessageAction types. Unrecognized types fall back to a generic
link-with-label rendering.

### 14.2 Custom SceneInteractionEvent Actions

SceneInteractionEvent actions use reverse-domain extensibility:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "...",
  "objectId": "...",
  "action": "com.example.quiz.answer",
  "userId": "...",
  "data": { "answer": "B" }
}
```

Plugins register handlers for custom action types. Unrecognized actions
are logged and ignored.

### 14.3 Custom viewHint Renderers

Deployment-specific `viewHint` values (e.g.,
`"com.example.isometric"`) can be handled by pluggable renderers:

```
RendererRegistry.register("com.example.isometric", IsometricRenderer)
```

The renderer receives the SceneRegion configuration and a canvas/context
to render into. It must implement the standard renderer interface
(initialize, update, render, dispose).

### 14.4 Custom Environment Schema

`SceneRegion.environment` is opaque JSON. Plugins can register
interpreters for deployment-specific environment schemas:

```
EnvironmentRegistry.register("com.example.weather", WeatherPlugin)
```

The plugin receives the environment object and modifies the renderer
state (e.g., adding rain particles, adjusting lighting).

### 14.5 Theme System

The client supports deployment-specific theming:

- CSS custom properties for colors, typography, spacing
- Dark/light mode toggle
- Deployment-provided theme manifest (JSON with color tokens, logo
  URLs, font specifications)
- W3C Design Tokens Community Group format for structured color
  values, consistent with the color representation convention used
  in SpaceRole colors

### 14.6 Deployment Branding

Per-deployment customization:
- Logo (Space icon, login screen, about dialog)
- Application name
- Accent colors
- Custom emoji packs pre-loaded
- Default Space configuration
- Feature flags (e.g., disable Scene for a chat-only deployment)

Branding is delivered as a JSON manifest fetched alongside or embedded
in the Session object (deployment-defined extension to the Session
capabilities).


---

## Appendix A: Complete Capability Inventory

| Spec | Capability URI | Primary Data Types |
|---|---|---|
| Mail | `urn:ietf:params:jmap:mail` | Mailbox, Email, Thread, SearchSnippet |
| Mail Submission | `urn:ietf:params:jmap:submission` | Identity, EmailSubmission |
| Chat | `urn:ietf:params:jmap:chat` | ChatContact, Chat, Message, Space, SpaceRole, SpaceMember, SpaceInvite, SpaceBan, CustomEmoji, ReadPosition, PresenceStatus |
| Chat WSS | `urn:ietf:params:jmap:chat:websocket` | (events: ChatTypingEvent, ChatPresenceEvent) |
| Chat Push | `urn:ietf:params:jmap:chat:push` | ChatPushConfig, ChatMessagePush, ChatMessageEntry |
| Chat DID | `urn:ietf:params:jmap:chat:did` | (extends ChatContact with `did` field) |
| Chat FileNode | `urn:ietf:params:jmap:chat:filenode` | FileNode (Space-scoped) |
| Chat Tasks | `urn:ietf:params:jmap:chat:tasks` | (bindings: Space.taskListId, Task.chatId) |
| Chat Calendars | `urn:ietf:params:jmap:chat:calendars` | (bindings: Space.calendarId, MessageAction calendar types) |
| VTC | `urn:ietf:params:jmap:vtc` | VTCCall, VTCParticipant, VTCRecording, VTCLivestream |
| VTC WSS | `urn:ietf:params:jmap:vtc:websocket` | (events: VTCRingEvent, VTCCallEndEvent, VTCParticipantEvent, VTCMediaStateEvent, VTCActiveSpeakerEvent, VTCUnmuteRequestEvent, VTCRecordingStateEvent, VTCGatewaySignal) |
| Scene | `urn:ietf:params:jmap:scene` | SceneRegion, SceneObject, SceneAvatar, SceneAsset |
| Scene WSS | `urn:ietf:params:jmap:scene:websocket` | (events: SceneAvatarEvent, SceneObjectEvent, SceneInteractionEvent) |
| CID | `urn:ietf:params:jmap:cid` | (extends blob upload response with sha256) |

## Appendix B: Cross-Capability Binding Reference

```
Chat.activeCallId -----------------> VTCCall.id
VTCCall.chatId --------------------> Chat.id
VTCCall.spaceId -------------------> Space.id
VTCCall.channelId -----------------> Chat.id (channel)
VTCCall.calendarEventId -----------> CalendarEvent.id
SceneRegion.chatId ----------------> Chat.id
SceneRegion.spaceId ---------------> Space.id
SceneRegion.channelId -------------> Chat.id (channel)
SceneRegion.activeCallId ----------> VTCCall.id
Space.taskListId ------------------> TaskList.id
Space.calendarId ------------------> Calendar.id
Attachment.filenodeId -------------> FileNode.id
ChatContact.did -------------------> DID Document
SceneAsset.sha256 -----------------> CID verification
Attachment.sha256 -----------------> CID verification
```

Direction of binding (which spec defines the field):
- `Chat.activeCallId`: defined in Chat spec, references VTC
- `VTCCall.chatId/spaceId/channelId`: defined in VTC spec, references Chat
- `VTCCall.calendarEventId`: defined in VTC spec, references Calendars
- `SceneRegion.chatId/spaceId/channelId`: defined in Scene spec, references Chat
- `SceneRegion.activeCallId`: defined in Scene spec, references VTC
- `Space.taskListId`: defined in Chat Tasks spec
- `Space.calendarId`: defined in Chat Calendars spec
- `Attachment.filenodeId`: defined in Chat FileNode spec
- `ChatContact.did`: defined in Chat DID spec

## Appendix C: WebSocket Frame Routing Table

The Core module's event router dispatches incoming WebSocket frames
by `@type`:

```
+---------------------------+------------------+---------------------+
| @type                     | Target Module    | Handler             |
+---------------------------+------------------+---------------------+
| StateChange               | Core (fan-out)   | route by data type  |
|   Mailbox/Email/Thread    |   -> Mail        | mailbox/email sync  |
|   Identity/EmailSubmission|   -> Mail        | outbox sync         |
| Response                  | Core             | match to request id |
| RequestError              | Core             | error handling      |
| ChatTypingEvent           | Chat             | typing indicator UI |
| ChatPresenceEvent         | Chat             | presence badge      |
| VTCRingEvent              | VTC              | ring notification   |
| VTCCallEndEvent           | VTC              | call ended cleanup  |
| VTCParticipantEvent       | VTC              | participant list    |
| VTCMediaStateEvent        | VTC              | media state badges  |
| VTCActiveSpeakerEvent     | VTC              | speaker highlight   |
| VTCUnmuteRequestEvent     | VTC              | unmute prompt       |
| VTCRecordingStateEvent    | VTC              | recording indicator |
| VTCGatewaySignal          | VTC              | gateway passthrough |
| SceneAvatarEvent          | Scene            | avatar enter/exit   |
| SceneObjectEvent          | Scene            | object CRUD notify  |
| SceneInteractionEvent     | Scene            | interaction handler |
| (unrecognized)            | (ignored)        | log + discard       |
+---------------------------+------------------+---------------------+
```

## Appendix D: Data Flow Diagram -- Message Send

End-to-end flow for sending a chat message, showing interaction
between layers:

```
User types message and presses Enter
         |
         v
+------------------+
| Web Frontend     |
| Compose Bar      |
+------------------+
         |
    Tauri command: jmap_method_call
         |
         v
+------------------+
| Rust Backend     |
| JMAP HTTP Client |
+------------------+
         |
    1. Optimistic: insert local Message with temp ID
    2. Upload attachments (if any) via blob upload endpoint
    3. If CID present, record sha256 from upload response
    4. Send Message/set create via WebSocket
         |
         v
+------------------+
| JMAP Server      |
+------------------+
         |
    Response: server-assigned ID, state token
         |
         v
+------------------+
| Rust Backend     |
+------------------+
         |
    1. Replace temp ID with server ID
    2. Update Message state token
    3. Notify web frontend via event
         |
         v
+------------------+
| Web Frontend     |
| Message List     |
+------------------+
         |
    Update message: remove "pending" indicator
```

## Appendix E: Data Flow Diagram -- Incoming Call

```
+------------------+
| JMAP Server      |
+------------------+
         |
    VTCRingEvent (ephemeral, via WebSocket)
    + VTCCall StateChange (persistent)
         |
         v
+------------------+
| Rust Backend     |
| WS Manager       |
+------------------+
         |
    Route VTCRingEvent to VTC module
    Route StateChange to state tracker
         |
         v
+------------------+         +------------------+
| VTC Module       |         | OS Integration   |
| (Web Frontend)   |-------->| (Rust Backend)   |
+------------------+         +------------------+
         |                           |
    Render in-app ring       Trigger OS call UI
    overlay (if focused)     (if backgrounded)
         |                           |
    [User accepts]                   |
         |                           |
         v                           |
    VTCParticipant/set               |
    callResponse: "accepted"         |
         |                           |
         v                           |
    Connect to joinUri               |
    (WebRTC peer connection)         |
```

## Appendix F: Layout Template -- Mail + Chat + VTC + Scene (Tabbed)

Full layout when all four primary capabilities are present, with
tabbed multi-view:

```
+------------------------------------------------------------------+
| [Mail (2)]  [Chat (1)]  [Chess - Board Room]  [Doom - Arena]     |
+------------------------------------------------------------------+
| [PiP Call Window - always visible across tabs]                   |
+--------+---------------------+------------------+-----------+
|        |                     | Scene Viewport   |           |
| Space  |   Channel View      | (or collapsed)   |  Right    |
| Side-  |                     |                  |  Panel    |
| bar    | +----------------+  | +-------------+  |           |
|        | | Message List   |  | | 3D/2D Scene |  | Members   |
| [S1]   | |                |  | |             |  | Files     |
| [S2]   | |                |  | | [Avatars]   |  | Pinned    |
| [S3]   | |                |  | | [Objects]   |  |           |
|        | |                |  | |             |  |           |
| ----   | +----------------+  | +-------------+  |           |
| DMs    | | Compose Bar    |  |                  |           |
|        | +----------------+  | [Scene Chat      |           |
| [D1]   |                     |  Overlay]         |           |
| [D2]   | [Call Banner]       |                  |           |
+--------+---------------------+------------------+-----------+
```

This shows the Chat tab active. Switching to the Mail tab replaces
everything below the tab bar with the three-pane mail layout (Section
6.2). Switching to a Scene tab (Chess or Doom) replaces everything
with a full-window spatial viewport.

The PiP call window floats above the tab bar and is visible in all
tabs. The Scene viewport within the Chat tab is resizable and can be
expanded to full width (hiding the message list) or collapsed to a
thumbnail. The chat overlay in the scene viewport shows messages
from the region-bound Chat as semi-transparent text.
