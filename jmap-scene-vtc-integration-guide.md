# JMAP Scene — VTC Integration Guide

For JMAP Scene implementers adding voice/video calling via `draft-atwood-jmap-vtc-00`.

Read the Scene and VTC drafts first. This guide does not re-state normative requirements. It
covers how the two capabilities fit together — the integration surface, the spatial audio
patterns, and the UX patterns that fall out of the design.

This guide is non-normative. `draft-atwood-jmap-scene-00` and `draft-atwood-jmap-vtc-00`
are the source of truth. Where this guide and a draft disagree, the draft wins.

---

## How to read this guide

You know JMAP Scene. You are adding voice and video. The VTC spec is standalone, but it was
designed with Scene in mind: the Scene spec's `activeCallId` field on SceneRegion binds a
VTCCall to a spatial region, and the VTC spec's media-layer abstraction leaves room for
simulation-layer concerns like spatial audio. The integration is a set of optional foreign
keys, a handful of server-side behaviors, and several patterns the specs leave to deployments.

The guide follows a flow that mirrors what a user experiences: enter a region, join the
bound call, hear other avatars in 3D space, walk between breakout sub-regions, see a
screen share on a virtual monitor, and leave.

---

## 1. Capability co-advertisement

### When to advertise both

Advertise `urn:ietf:params:jmap:vtc` alongside `urn:ietf:params:jmap:scene` when your
deployment includes both a call-state manager and a spatial state server on the same
account. The two capabilities are independent — a deployment can run either without the
other — but the integration features described in this guide only activate when both are
present on the same account.

### Session object

A session advertising both capabilities looks like this:

```json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": { "...": "..." },
    "urn:ietf:params:jmap:websocket": {
      "url": "wss://jmap.example.com/ws",
      "supportsPush": true
    },
    "urn:ietf:params:jmap:scene": {},
    "urn:ietf:params:jmap:vtc": {},
    "urn:ietf:params:jmap:scene:websocket": {},
    "urn:ietf:params:jmap:vtc:websocket": {}
  },
  "accounts": {
    "acct1": {
      "name": "alice@example.com",
      "isPersonal": true,
      "accountCapabilities": {
        "urn:ietf:params:jmap:scene": {
          "mayCreateRegion": true,
          "mayCreateObject": true,
          "supportedVisualTypes": ["model/gltf-binary"],
          "maxRegionsPerAccount": null,
          "maxObjectsPerRegion": null,
          "maxAvatarsPerRegion": 50,
          "maxAssetSizeBytes": null
        },
        "urn:ietf:params:jmap:vtc": {
          "mayCreateCall": true,
          "supportsRingCalls": false,
          "supportsRoomCalls": true,
          "supportsScheduledCalls": false,
          "supportsRecording": false,
          "supportsLivestream": false,
          "supportsLobby": false,
          "supportsBreakoutRooms": true,
          "supportedMediaTypes": ["audio", "video", "screen"],
          "maxParticipantsPerCall": 50,
          "maxConcurrentCalls": null,
          "maxBreakoutRooms": 10
        }
      }
    }
  }
}
```

For spatial environments, room calls are the natural model. Ring calls imply a 1:1
interaction that does not map well to entering a shared 3D space. Most Scene+VTC
deployments will have `supportsRoomCalls: true` and may not need ring calls at all.
Breakout room support is valuable for sub-region voice isolation (section 7).

### Feature detection

Clients MUST check for `urn:ietf:params:jmap:vtc` in `accountCapabilities` before
rendering any voice or video UI in the 3D environment. Do not assume VTC is present
because your deployment has it — the same client code must work against deployments
that have not yet deployed calling.

---

## 2. The activeCallId binding

### What activeCallId is

`SceneRegion.activeCallId` is a `String|null` field on the SceneRegion object. It is
the id of a VTCCall that is currently active and bound to this region. The spec says:

> When `urn:ietf:params:jmap:vtc` is present: the id of an active VTCCall bound to
> this region. `null` if no call is active.

This field is the primary integration point between Scene and VTC. When a user enters a
region and sees `activeCallId` set to a VTCCall id, the client knows there is a call to
join. When `activeCallId` is `null`, the region has no associated voice channel.

### Server responsibilities

Your server must implement the normative side, and should implement the binding side:

**When that VTCCall transitions to `"ended"` state (MUST per Scene spec):**
Clear `SceneRegion.activeCallId` to `null`. This triggers a SceneRegion `StateChange`.

**When a VTCCall bound to a SceneRegion enters `"active"` state (deployment policy):**
Set `SceneRegion.activeCallId` to the VTCCall's id. This triggers a SceneRegion
`StateChange` notification. Note: this direction (call→region binding) is not
normatively required by either spec — it is recommended deployment behavior.

The server should do this atomically with the VTCCall state transition. A VTCCall that has
ended but whose SceneRegion still shows `activeCallId` set is a data integrity bug.

Note: The Scene spec lists `activeCallId` as a client-patchable field on `SceneRegion/set
update` — any authorized client can set or clear it. The atomicity guidance above is
non-normative deployment advice for servers that manage the binding automatically. It is
not a normative requirement from either spec. A deployment may choose server-managed
atomicity (recommended), client-managed binding (the client sets `activeCallId` after
creating a VTCCall), or a hybrid where the server enforces consistency checks on client
patches. The key invariant is that `activeCallId` should not reference a VTCCall in
`"ended"` state, regardless of which component maintains it.

### Persistent vs ephemeral region calls

Two patterns exist for binding calls to regions:

**Persistent call (always-on voice):** Create a room-type VTCCall when the SceneRegion
is created. The call stays active for the lifetime of the region. `activeCallId` is
always set. This is the Mozilla Hubs / Discord voice channel model.

**Ephemeral call (on-demand voice):** The region starts with `activeCallId: null`. A
user explicitly starts a call via `VTCCall/set create` with the region as context. The
call ends when the last participant leaves. This is appropriate for regions that are
sometimes silent (a gallery, a lobby).

For the persistent-call pattern, the server should automatically create a room call when
a new SceneRegion is created, and set `activeCallId` immediately. For the ephemeral
pattern, the client creates the call and the server sets `activeCallId` on the region
via the standard binding mechanism.

---

## 3. Co-location: region entry joins a call

### The pattern

When a SceneRegion has an `activeCallId`, entering the region and joining the call are
two independent operations that the client composes into a single UX gesture: the user
clicks "Enter Room" and the client performs both steps.

### Participant-follows-avatar: entering a region auto-joins the call

This is the most common pattern for immersive environments. When the user enters a
region:

1. Create a SceneAvatar in the region via `SceneAvatar/set create`.
2. Read `activeCallId` from the SceneRegion.
3. If `activeCallId` is non-null, join the call via `VTCParticipant/set create` with
   `callId` set to `activeCallId` and `joinMethod` set to `"webrtc"`.
4. Open the VTCCall's `joinUri` in the media layer to connect audio/video.

Deployments with a simulation layer should also connect to `simulationUri` from the
SceneRegion response (typically after step 2, before or alongside step 4). The
simulation layer connection provides real-time avatar positions needed for spatial
audio and proximity voice. See the
[JMAP Scene Simulation Layer Guide](jmap-scene-simulation-guide.md) for connection
ordering, tick rate selection, and state reconciliation patterns.

```json
[
  ["SceneAvatar/set", {
    "accountId": "acct1",
    "create": {
      "av0": {
        "regionId": "01J5ABC0000000000000000001"
      }
    }
  }, "0"],
  ["SceneRegion/get", {
    "accountId": "acct1",
    "ids": ["01J5ABC0000000000000000001"],
    "properties": ["activeCallId"]
  }, "1"]
]
```

Then, using the `activeCallId` from the response:

```json
[
  ["VTCParticipant/set", {
    "accountId": "acct1",
    "create": {
      "p0": {
        "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
        "joinMethod": "webrtc"
      }
    }
  }, "2"],
  ["VTCCall/get", {
    "accountId": "acct1",
    "ids": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
    "properties": ["joinUri", "joinPassword"]
  }, "3"]
]
```

Do not open `joinUri` automatically. The VTC spec is explicit: clients MUST NOT connect
to `joinUri` without explicit user initiation. The "Enter Room" button is the user
initiation — the client should make clear that entering the room means joining voice.

### Leaving a region leaves the call

When the user leaves a region, the client should leave the VTC call as well:

1. Set `leftAt` on the user's VTCParticipant via `VTCParticipant/set update`.
2. Disconnect from the media layer.
3. Set `leftAt` on the user's SceneAvatar via `SceneAvatar/set update`.

The order matters: leave the call first, then leave the region. This avoids a window
where the user's avatar is gone but their voice is still audible.

### Server-side auto-join (alternative approach)

Instead of the client performing both operations, the server can automatically create a
VTCParticipant when a SceneAvatar is created in a region that has an `activeCallId`.
This is a server-side policy decision:

- The server observes `SceneAvatar/set create` for a region with `activeCallId`.
- The server auto-creates a VTCParticipant record for the user on the bound call.
- The client receives the VTCParticipant id in a `StateChange` and connects to the
  media layer.

This approach reduces client complexity but means the user cannot enter a region without
joining its call. Whether to implement auto-join is a deployment decision. The specs
do not mandate it.

---

## 4. Avatar-follows-participant

### The pattern

When a VTC participant joins or leaves a call that is bound to a SceneRegion, the
server automatically creates or removes a SceneAvatar for that participant.

### Server-side approach

When the server observes a `VTCParticipant/set create` for a call that has a region
binding:

1. Check whether the participant already has a SceneAvatar in the bound region.
2. If not, create a SceneAvatar for the participant at the region's `spawnPosition`.
3. When the VTCParticipant leaves (sets `leftAt`), set `leftAt` on the corresponding
   SceneAvatar.

This approach ensures that every voice participant has a visible avatar, even if their
client does not implement Scene. It is the right pattern when the VTC call is the
primary entry point and the 3D environment is secondary (for example, a spatial audio
overlay on top of a standard video call).

### Client-side approach

The client subscribes to VTCParticipant state changes for the bound call. When a new
participant appears, the client creates a SceneAvatar (or shows a default placeholder)
at the spawn position. When a participant leaves, the client removes or hides the avatar.

This approach is simpler to implement but produces inconsistent results when different
clients have different rendering policies. Prefer the server-side approach for
production deployments.

### Correlating avatars with participants

Both the Scene spec and the VTC spec say the server SHOULD (not MUST) use the userId as
the id within a given context. When the server follows this recommendation, the same
userId appears as both `SceneAvatar.userId` and `VTCParticipant.userId`, providing a
natural join key. The client can correlate avatars with participants by matching on userId.

Because this is a SHOULD, not a MUST, servers are permitted to assign opaque ids that
differ from userId — for example, when a single user has multiple avatars in the same
region or multiple VTC participants in the same call (multi-device). When the id does not
equal userId, the client must fall back to matching on the `userId` field directly rather
than assuming `id == userId`. Clients should always use the `userId` field for correlation,
not the `id` field, to be robust against servers that assign non-userId ids:

```json
{
  "sceneAvatar": {
    "id": "user:alice@example.com",
    "regionId": "01J5ABC0000000000000000001",
    "userId": "user:alice@example.com",
    "position": [5.2, 0, -1.8],
    "displayName": "Alice Chen"
  },
  "vtcParticipant": {
    "id": "user:alice@example.com",
    "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
    "userId": "user:alice@example.com",
    "displayName": "Alice Chen",
    "mediaState": {
      "audio": true,
      "video": true,
      "screen": false,
      "raisedHand": false
    }
  }
}
```

---

## 5. Spatial audio

### What the specs say

The Scene spec defers spatial audio to deployments:

> Spatial audio is a simulation-layer concern. The VTCCall provides the voice channel;
> the simulation layer handles spatialization based on avatar positions.

The VTC spec has no spatial awareness. It provides an audio stream per participant.
Making that audio spatial is entirely a deployment concern at the intersection of the
Scene simulation layer and the VTC media layer.

### Architecture

Spatial audio requires two inputs: the audio stream from each participant (from VTC)
and the 3D position of each participant's avatar (from Scene). These inputs must meet
at a point where spatialization is applied: either on the media server (server-side
mixing) or on the client (client-side mixing).

### Client-side spatial audio

The client receives individual audio streams from the VTC media layer (via WebRTC,
where each participant is a separate audio track). The client also knows the local
user's avatar position and every other avatar's position (from the simulation layer
or from periodic `SceneAvatar/get` calls). The client applies HRTF or stereo panning
based on the relative position of each speaker's avatar to the listener's avatar.

**Implementation steps:**

1. For each VTC participant, look up the corresponding SceneAvatar by userId.
2. Compute the relative vector from the listener's avatar position to the speaker's
   avatar position.
3. Apply distance attenuation: a speaker 50 meters away should be quieter than one
   2 meters away. A common model is inverse-distance with a configurable rolloff
   factor and a maximum audible radius.
4. Apply directional panning: convert the relative vector to azimuth and elevation
   relative to the listener's orientation, then apply stereo panning or HRTF.
5. Update the spatialization parameters as positions change (from the simulation layer
   at 10-20 Hz).

For simulation layer architecture, tick rate selection, and state reconciliation
patterns that affect position update frequency, see the
[JMAP Scene Simulation Layer Guide](jmap-scene-simulation-guide.md).

The Web Audio API provides `PannerNode` with HRTF support, making this straightforward
in browser-based clients. Route each WebRTC audio track through a `PannerNode` whose
position is set to the speaker's avatar coordinates (converted to the listener's
local frame).

**Coordinate mapping:**

Scene uses right-handed Y-up meters. The Web Audio API (`PannerNode` and
`AudioListener`) also uses a right-handed Y-up coordinate system, so axis
orientation matches directly — no axis swapping or sign flipping is needed for
browser clients.

However, the Web Audio API's distance models (`linear`, `inverse`,
`exponential`) use unitless coordinate values, not meters. The API computes
distance from the numeric difference between `PannerNode.positionX/Y/Z` and
`AudioListener.positionX/Y/Z` and feeds that distance into the attenuation
formula with `refDistance`, `maxDistance`, and `rolloffFactor`. If you pass Scene
coordinates (which are in meters) directly, the distance model parameters
(`refDistance`, `maxDistance`) must also be expressed in meters for the
attenuation to be physically correct. This works naturally as long as all values
are in the same unit. Problems arise only if a client scales Scene coordinates
before passing them to Web Audio without also scaling the distance parameters.

For clients using other audio APIs (OpenAL, FMOD, Wwise), convert from Scene's
Y-up right-handed convention to the audio API's convention. OpenAL uses
right-handed Y-up (same as Scene); FMOD and Wwise use left-handed Y-up
(negate Z).

### Server-side spatial audio

A Selective Forwarding Unit (SFU) that has access to avatar positions can apply per-
subscriber spatial mixing. Each subscriber receives a unique mix where each speaker's
audio is attenuated and panned based on the subscriber's avatar position relative to
the speaker's avatar position.

This requires the SFU to know avatar positions. Two approaches:

1. **Scene position feed:** The JMAP server pushes avatar positions to the SFU at a
   deployment-defined interval (5-30 seconds from JMAP state reconciliation, or
   higher-frequency from the simulation layer). The SFU uses these to compute
   per-subscriber mixes.

2. **Simulation-layer integration:** The simulation layer (which has real-time positions
   at 10-20 Hz) feeds positions directly to the SFU. This is the higher-fidelity
   approach and is what Mozilla Hubs used (Janus SFU with Hubs-specific position feed).

Server-side spatial audio scales better for large rooms because each client receives a
single mixed stream rather than N individual streams, but it requires a custom or
spatial-aware SFU.

### Distance attenuation model

A deployment should define:

- **Reference distance** (meters): The distance at which attenuation begins. Typically
  1-2 meters. Audio is at full volume within this radius.
- **Rolloff factor**: How quickly audio attenuates with distance. A value of 1.0 gives
  physically realistic inverse-distance attenuation. Higher values make distant sounds
  quieter faster.
- **Maximum distance** (meters): Beyond this radius, the speaker is inaudible. This is
  the proximity voice cutoff (see section 6).

These parameters are deployment-defined and should be documented for client implementers.
They can be conveyed in the SceneRegion's `environment` object:

```json
{
  "environment": {
    "spatialAudio": {
      "referenceDistance": 1.5,
      "rolloffFactor": 1.0,
      "maxDistance": 25.0,
      "distanceModel": "inverse"
    }
  }
}
```

The `environment` field is opaque to the JMAP Scene spec — the server stores and relays
it without interpretation. Client implementations that understand the `spatialAudio` key
use it; others ignore it.

---

## 6. Proximity-based voice

### The pattern

Only hear participants within a certain radius of your avatar. Participants beyond the
radius are silent. This creates natural conversational clusters in large spaces — small
groups can talk without being overheard by the whole room.

### Client-side implementation

The simplest approach: the client sets the volume of each speaker's audio to zero when
their avatar is beyond the maximum distance. The client already has the position data
(from section 5) and the distance attenuation model. When the computed attenuation
reduces a speaker's volume to zero (or below a threshold), mute that audio track
locally.

This approach works with any SFU — the server sends all audio streams and the client
selectively mutes. The downside is bandwidth: every participant's audio is transmitted
even when not heard.

### Server-side implementation (SFU selective forwarding)

An SFU that knows avatar positions can stop forwarding audio streams that the subscriber
would not hear. This saves bandwidth and reduces client processing.

The SFU maintains, for each subscriber, a set of "audible" speakers — those whose
avatar is within `maxDistance` of the subscriber's avatar. The SFU forwards only
audible speakers' audio tracks. When an avatar moves into range, the SFU starts
forwarding; when it moves out, the SFU stops.

This is the approach used by Mozilla Hubs and Gather. It requires:

1. The SFU receives avatar position updates (from the simulation layer or from JMAP
   state reconciliation).
2. For each pair (subscriber, speaker), the SFU computes distance.
3. The SFU uses standard WebRTC simulcast/SVC selective forwarding to include or
   exclude each speaker's audio track from each subscriber's subscription.

### Hysteresis

Apply hysteresis to the proximity threshold to avoid rapid mute/unmute toggling when an
avatar is near the boundary. A common approach: enter the audible set at `maxDistance`
meters, but do not leave the audible set until `maxDistance + 2` meters. The exact
hysteresis buffer is a deployment tuning parameter.

### SceneAvatar/query for proximity

The Scene spec's `SceneAvatar/query` supports `withinRadius` spatial filtering. A
server can use this to determine which avatars are near a given position. For SFU-side
proximity calculations, the SFU can either maintain its own position state (faster)
or query the JMAP server periodically (simpler but slower).

---

## 7. Breakout rooms as sub-regions

### The pattern

A SceneRegion represents a large space (a conference hall, a virtual office). Within it,
distinct areas (meeting rooms, alcoves, stages) each have their own VTC call. Walking
your avatar from one area to another switches your voice channel.

### Modeling with VTC breakout rooms

The VTC spec supports breakout rooms as parent/child VTCCall relationships. The parent
call's `breakoutRoomIds` lists the child call ids. A moderator can assign participants
to breakout rooms via `VTCParticipant/set update` on the participant's `callId`.

For Scene integration, each breakout room corresponds to a spatial sub-region. There
are two modeling approaches:

**Approach A: Multiple SceneRegions with separate VTCCalls**

Create a separate SceneRegion for each breakout area, each with its own `activeCallId`
pointing to a breakout VTCCall. The parent SceneRegion has `activeCallId` pointing to
the parent VTCCall.

This is clean but requires the user to leave one SceneRegion and enter another, which
means a full region transition (new `SceneAvatar/set` create, new simulation layer
connection). It is appropriate when breakout areas are architecturally distinct (separate
rooms with doors).

**Approach B: Single SceneRegion with spatial zones**

Keep all breakout areas in one SceneRegion. Define spatial zones using SceneObject
records with `customProperties` that reference breakout VTCCall ids. The client detects
when the avatar enters a zone and switches the user's VTC call.

```json
{
  "id": "01J5OBJ0000000000000000010",
  "regionId": "01J5ABC0000000000000000001",
  "name": "Meeting Room A",
  "position": [20, 0, 15],
  "orientation": [0, 0, 0, 1],
  "scale": [1, 1, 1],
  "visualRef": null,
  "visualType": null,
  "physicsMode": "none",
  "interactable": false,
  "visible": false,
  "customProperties": {
    "com.example.zone": {
      "type": "breakout",
      "vtcCallId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "radius": 8.0
    }
  }
}
```

The zone object is invisible (`visible: false`) and has no physics. It serves purely
as a spatial trigger. The client checks the avatar's distance to each zone center; when
the avatar enters a zone's radius, the client switches VTC calls.

This approach avoids region transitions but requires the client to implement zone
detection and call switching. It is appropriate for open-plan spaces where breakout
areas are not architecturally separated (a conference hall with clusters of chairs).

### Switching calls on zone transition

When the avatar crosses from one zone to another:

1. Leave the current VTC call: set `leftAt` on the VTCParticipant.
2. Disconnect from the current media session.
3. Join the new zone's VTC call: create a VTCParticipant on the new call.
4. Connect to the new call's `joinUri`.

The transition should be fast. The client can pre-fetch `joinUri` for nearby zones to
reduce latency. Apply hysteresis to zone boundaries (same as proximity voice in
section 6) to avoid rapid call switching when the avatar is near a boundary.

### Lobby regions

A SceneRegion can serve as a lobby for a VTC call that has `lobbyEnabled: true`. The
user enters the region and sees other waiting avatars. When a moderator admits them
(via `VTCParticipant/set update` setting `lobbyState` to `"admitted"`), the client
connects to the media layer and the avatar transitions from the lobby position to the
main space.

---

## 8. Screen sharing as a SceneObject

### The pattern

A VTC participant's screen share is rendered as a texture on a SceneObject — a virtual
monitor, whiteboard, or projection surface. Other avatars in the region can look at the
screen share by looking at the object.

### Modeling

When a participant starts screen sharing (`mediaState.screen: true`), the client or
server creates a SceneObject representing the shared screen:

```json
{
  "id": "01J5OBJ0000000000000000020",
  "regionId": "01J5ABC0000000000000000001",
  "parentId": null,
  "name": "Screen share: Alice Chen",
  "position": [0, 2.5, -10],
  "orientation": [0, 0, 0, 1],
  "scale": [3.2, 1.8, 0.05],
  "visualRef": null,
  "visualType": null,
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "com.example.screenshare": {
      "participantUserId": "user:alice@example.com",
      "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
      "mediaTrackKind": "screen"
    }
  }
}
```

The SceneObject has no `visualRef` because its visual content comes from a live video
stream, not a static asset. The `customProperties` tell clients which VTC participant's
screen share to render on this surface.

### Client rendering

The client:

1. Receives a `SceneObjectEvent` or `StateChange` for the new screen-share object.
2. Reads `customProperties` and identifies the VTC participant userId and track kind.
3. Maps the VTC participant's screen-share video track to a texture.
4. Renders the texture on the SceneObject's geometry (a flat quad at the specified
   position, orientation, and scale).

The exact rendering is client-defined. The Scene spec has no opinion on how textures
are applied to objects. In Three.js, this is a `VideoTexture` applied to a plane mesh.
In Unity, it is a `RenderTexture` fed from a WebRTC video track.

### Lifecycle

When the participant stops screen sharing (`mediaState.screen: false`), the server or
client should destroy the screen-share SceneObject or set `visible: false`. If the
server creates the object, the server should clean it up. If the client creates it, the
client is responsible.

The server-side approach is preferable: it ensures all clients see consistent state and
prevents orphaned screen-share objects when a client disconnects without cleanup.

### Predefined screen surfaces

Alternatively, a region can have predefined screen surfaces — permanent SceneObjects
that serve as displays. When a participant starts screen sharing, the server or client
assigns the share to an available surface by updating the surface's `customProperties`
with the participant's userId and call id. This avoids dynamic object creation and lets
the region designer control where screen shares appear.

---

## 9. WebSocket interop

### Subscribing to both Scene and VTC events

When a SceneRegion has an `activeCallId`, a client typically needs events from both
the Scene stream (avatar movement, object changes, interactions) and the VTC stream
(participant joins, media state changes, active speaker). Both subscriptions coexist
on the same JMAP WebSocket connection.

Send both subscription messages after connecting:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
```

```json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
  "eventTypes": null
}
```

Both deliver events on the same WebSocket connection. The client distinguishes them by
the `@type` field on each frame.

### Also enable state-change push

In addition to ephemeral events, subscribe to persistent state changes for the data
types you need:

```json
{
  "@type": "WebSocketPushEnable",
  "dataTypes": ["SceneRegion", "SceneObject", "SceneAvatar",
                "VTCCall", "VTCParticipant"]
}
```

This ensures you receive `StateChange` notifications when the SceneRegion's
`activeCallId` changes, when avatar positions are reconciled, or when VTCCall state
transitions occur. Ephemeral events cover real-time signals; `StateChange` covers
persistent state mutations.

### Subscription independence

The Scene WSS spec and VTC WSS spec are explicit: `SceneStreamEnable`/`SceneStreamDisable`
does not affect VTC events, and `VTCStreamEnable`/`VTCStreamDisable` does not affect
Scene events. Disabling one leaves the other active.

When the user leaves a region but stays in the call (for example, transitioning between
sub-regions), update the Scene subscription without touching the VTC subscription:

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000002"],
  "eventTypes": null
}
```

The VTC subscription for the existing call continues uninterrupted.

### Unified subscription via ChatStreamEnable

If the deployment also has `urn:ietf:params:jmap:chat:websocket`, a single
`ChatStreamEnable` can subscribe to all ephemeral event types:

```json
{
  "@type": "ChatStreamEnable",
  "dataTypes": ["typing", "presence", "vtc", "scene"]
}
```

This is the coarse-grained path. It delivers all Scene and VTC events for all regions
and calls in which the user is active. For a client displaying a single region, prefer
the fine-grained `SceneStreamEnable` + `VTCStreamEnable` path to reduce server fan-out
and network traffic.

### Blocked-sender suppression asymmetry

The Scene WSS spec and VTC WSS spec apply blocked-sender suppression at different
scopes. The Scene WSS spec suppresses all event types — `SceneAvatarEvent`,
`SceneObjectEvent`, and `SceneInteractionEvent` — when the originating user is blocked.
The VTC WSS spec suppresses only `VTCRingEvent` and `VTCCallEndEvent` (per-user events)
when the initiator is blocked. Per-call events — `VTCParticipantEvent`,
`VTCMediaStateEvent`, `VTCActiveSpeakerEvent`, `VTCUnmuteRequestEvent`,
`VTCRecordingStateEvent`, and `VTCGatewaySignal` — are not subject to blocked-sender
filtering. They are delivered to any current participant of the call.

This means a user who has blocked another user will not see that user's avatar appear
(SceneAvatarEvent is suppressed) but will still receive VTCParticipantEvent when the
blocked user joins the same call, and will still receive VTCMediaStateEvent and
VTCActiveSpeakerEvent for the blocked user's media activity. The client must handle this
gap: if the deployment wants full blocked-user hiding in a Scene+VTC context, the client
should filter VTC per-call events client-side by checking the participant's userId against
the local block list. The server does not do this filtering for per-call VTC events because
the VTC WSS spec scopes block filtering to per-user ring events only.

### Correlating Scene and VTC events

When a `SceneAvatarEvent` with `event: "entered"` arrives, and the region has an
`activeCallId`, the client may soon receive a `VTCParticipantEvent` with
`event: "joined"` for the same userId. The two events arrive independently — they are
not guaranteed to arrive in any particular order. The client should handle both
gracefully:

- If the avatar event arrives first, show the avatar at the spawn position. When the
  participant event arrives, connect the avatar to the audio track.
- If the participant event arrives first, show the participant in the VTC participant
  list. When the avatar event arrives, place the avatar in the 3D scene and attach
  the audio.

---

## 10. Putting it together: a Mozilla Hubs-like deployment

This section walks through a complete flow for a deployment modeled after Mozilla Hubs:
persistent rooms with spatial audio, avatar tracking, and screen sharing.

### Region setup (server/admin)

Create a SceneRegion and a persistent room call:

```json
[
  ["VTCCall/set", {
    "accountId": "acct1",
    "create": {
      "c0": {
        "callType": "room",
        "mediaTypes": ["audio", "video", "screen"],
        "subject": "Design Studio"
      }
    }
  }, "0"],
  ["SceneRegion/set", {
    "accountId": "acct1",
    "create": {
      "r0": {
        "name": "Design Studio",
        "bounds": {
          "min": [-30, 0, -30],
          "max": [30, 10, 30]
        },
        "spawnPosition": [0, 0, 8],
        "spawnOrientation": [0, 0, 0, 1],
        "accessPolicy": "public",
        "environment": {
          "skyColor": "#1a1a2e",
          "ambientIntensity": 0.4,
          "spatialAudio": {
            "referenceDistance": 1.5,
            "rolloffFactor": 1.0,
            "maxDistance": 20.0,
            "distanceModel": "inverse"
          }
        }
      }
    }
  }, "1"],
  ["SceneRegion/set", {
    "accountId": "acct1",
    "update": {
      "#r0": {
        "activeCallId": "#c0"
      }
    }
  }, "2"]
]
```

Note: `activeCallId` is not a valid create-time field on `SceneRegion/set` — it is
only accepted on `update`. The example uses a two-step approach: create the region,
then immediately patch `activeCallId` via a back-reference to the just-created call.
The room call is immediately active and the region is ready for visitors.

### User enters the room

Alice clicks "Enter Room" in the client.

**Step 1: Create avatar and get region details.**

```json
[
  ["SceneAvatar/set", {
    "accountId": "acct1",
    "create": {
      "av0": {
        "regionId": "01J5ABC0000000000000000001",
        "visualRef": "blob-avatar-alice-001",
        "visualType": "model/gltf-binary"
      }
    }
  }, "0"],
  ["SceneRegion/get", {
    "accountId": "acct1",
    "ids": ["01J5ABC0000000000000000001"],
    "properties": ["activeCallId", "simulationUri", "environment"]
  }, "1"]
]
```

**Step 2: Join the VTC call.**

```json
[
  ["VTCParticipant/set", {
    "accountId": "acct1",
    "create": {
      "p0": {
        "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
        "joinMethod": "webrtc"
      }
    }
  }, "2"],
  ["VTCCall/get", {
    "accountId": "acct1",
    "ids": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
    "properties": ["joinUri", "joinPassword"]
  }, "3"]
]
```

**Step 3: Connect to the simulation layer and media layer.**

The client connects to `simulationUri` for real-time avatar position sync (10-20 Hz)
and to `joinUri` for WebRTC audio/video.

**Step 4: Subscribe to events.**

```json
{
  "@type": "SceneStreamEnable",
  "regionIds": ["01J5ABC0000000000000000001"],
  "eventTypes": null
}
```

```json
{
  "@type": "VTCStreamEnable",
  "callIds": ["01J4XKZQN4MWVT8PPBEHTJ3AB"],
  "eventTypes": null
}
```

**Step 5: Set up spatial audio.**

The client reads the `spatialAudio` config from the region's `environment`. For each
VTC participant (fetched via `VTCParticipant/query`), the client:

1. Looks up the corresponding SceneAvatar by userId.
2. Creates a Web Audio `PannerNode` with the configured distance model and parameters.
3. Routes the participant's WebRTC audio track through the `PannerNode`.
4. Updates the `PannerNode` position from the simulation layer as the avatar moves.

### Bob enters

The client receives:

```json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:bob@example.com",
  "event": "entered",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez"
}
```

Shortly after:

```json
{
  "@type": "VTCParticipantEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "participantId": "user:bob@example.com",
  "event": "joined",
  "role": "participant",
  "displayName": "Bob Martinez"
}
```

The client:

1. Renders Bob's avatar at the spawn position.
2. Creates a `PannerNode` for Bob's audio, positioned at the spawn point.
3. As Bob's avatar moves (via the simulation layer), updates the `PannerNode` position.
4. Bob's voice is now spatialized — Alice hears him from the direction of his avatar.

### Bob walks away

As Bob's avatar moves from position `[2, 0, 3]` to `[18, 0, -5]`, Alice hears his
voice gradually shift in the stereo field and attenuate with distance. At 20 meters
(the configured `maxDistance`), Bob's voice fades to silence. The audio continues to
be transmitted (unless SFU selective forwarding is enabled), but the client's spatial
audio processing reduces it to zero gain.

### Alice shares her screen

Alice starts screen sharing via the media layer. The server (or Alice's client) creates
a screen-share SceneObject:

```json
{
  "@type": "SceneObjectEvent",
  "regionId": "01J5ABC0000000000000000001",
  "objectId": "01J5OBJ0000000000000000020",
  "event": "created",
  "updatedBy": "user:alice@example.com",
  "position": [0, 2.5, -10]
}
```

Other clients fetch the object, read its `customProperties`, and render Alice's
screen-share video track on the virtual monitor at position `[0, 2.5, -10]`.

### Alice leaves

1. Alice's client sets `leftAt` on her VTCParticipant.
2. Alice's client disconnects from the media layer.
3. Alice's client sets `leftAt` on her SceneAvatar.

Other clients receive:

```json
{
  "@type": "VTCParticipantEvent",
  "callId": "01J4XKZQN4MWVT8PPBEHTJ3AB",
  "participantId": "user:alice@example.com",
  "event": "left",
  "role": "participant",
  "displayName": "Alice Chen"
}
```

```json
{
  "@type": "SceneAvatarEvent",
  "regionId": "01J5ABC0000000000000000001",
  "avatarId": "user:alice@example.com",
  "event": "left",
  "userId": "user:alice@example.com",
  "displayName": "Alice Chen"
}
```

The screen-share object is destroyed (server-side cleanup). Bob's client removes
Alice's avatar and her audio `PannerNode`.

---

## Quick reference: server checklist

When implementing the Scene-VTC integration, your server must:

- Set `SceneRegion.activeCallId` when a bound VTCCall enters `"active"` state.
- Clear `SceneRegion.activeCallId` to `null` when the bound VTCCall enters `"ended"` state.
- For persistent-call deployments: auto-create a room VTCCall when a SceneRegion is
  created (deployment policy).
- Feed avatar positions from the simulation layer to the SFU for server-side spatial
  audio (if applicable).
- Clean up screen-share SceneObjects when a participant stops screen sharing or leaves
  (if the server creates them).
- Return `notFound` for `VTCCall/get` and `VTCParticipant/get` to users who are not
  present in the bound SceneRegion (when region access control should gate call access).
- Advertise both `urn:ietf:params:jmap:scene` and `urn:ietf:params:jmap:vtc` under
  `accountCapabilities` for accounts that have both capabilities.

When implementing on the client side, your client must:

- Check `urn:ietf:params:jmap:vtc` in `accountCapabilities` before showing voice/video
  UI in the 3D environment.
- Check `activeCallId` on SceneRegion to determine whether voice is available.
- Only open `joinUri` after explicit user action — never automatically.
- Subscribe to both `SceneStreamEnable` and `VTCStreamEnable` for the active region
  and call.
- Correlate SceneAvatar and VTCParticipant records by userId for spatial audio.
- Apply spatial audio using avatar positions and the region's `environment` config.
- Handle Scene and VTC events arriving in any order for the same userId.
- Apply hysteresis to proximity and zone boundaries to avoid rapid audio toggling or
  call switching.
