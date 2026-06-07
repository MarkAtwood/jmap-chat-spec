# JMAP Scene Simulation Layer -- Implementer's Guide

For server and client implementers of `draft-atwood-jmap-scene-00`. Covers the
real-time simulation layer architecture, state reconciliation, authority models,
and operational decisions that the spec deliberately leaves to implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what the spec *deliberately leaves open* and what implementations must
decide before shipping.

---

## How to read this guide

The JMAP Scene draft defines spatial state at rest: what objects exist, where
they are, who is present. Everything that moves at interactive rates -- avatar
position synchronization, physics, interaction events, spatial audio -- lives
behind the `simulationUri` field on SceneRegion and is explicitly out of scope
for the spec.

This is not a free pass. An implementation that ships `simulationUri` as a bare
WebSocket echo server will deliver a broken product: avatars that teleport,
objects that desync, and a scene that falls apart under ten concurrent users.
The simulation layer is where the real engineering happens. The JMAP layer is
the spatial database; the simulation layer is the spatial runtime.

Each section below follows the same shape:

1. **What the spec leaves open** -- with a draft section citation, so you can
   read the normative text yourself.
2. **What you must decide** -- the concrete deployment choice you cannot avoid.
3. **Considerations** -- the trade-offs that inform the choice.
4. **Common patterns** -- how production spatial systems handle this.
5. **Recommended starting point** -- a defensible default. Not normative; you
   may choose otherwise with good reason.

When two sections interact (for example, authority model choices constrain what
state reconciliation can do), cross-references are explicit.

This guide is non-normative. `draft-atwood-jmap-scene-00` is the source of
truth. Where this guide and the draft disagree, the draft wins.

---

## A note on RFC 2119 keywords

This guide uses RFC 2119 keywords (MUST, SHOULD, MAY, REQUIRED, RECOMMENDED,
etc.) for clarity, but in the spirit of implementer guidance rather than as a
normative protocol specification:

- `draft-atwood-jmap-scene-00` is the normative source of truth. Where this
  guide describes a spec requirement using a keyword, the keyword reflects the
  spec's normativity; if guide and draft disagree, the draft wins.
- Where this guide uses a keyword for an operational practice or deployment
  choice (e.g., "simulation servers SHOULD batch position snapshots"), the
  keyword reflects implementer best-practice. Deviation does not affect
  protocol interoperability.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. The three-path architecture

### What the spec leaves open

The spec defines four JMAP data types (SceneRegion, SceneObject, SceneAvatar,
SceneAsset) with standard get/set/changes/query methods, a `simulationUri`
field pointing to an opaque real-time endpoint, and `assetUri` / `blobId`
fields for visual content. The spec explicitly states that "the JMAP server is
a spatial state database, not a rendering engine" (Introduction, Design
Philosophy). It does not prescribe how these three concerns -- state
management, real-time simulation, and asset delivery -- relate to each other
at the infrastructure level.

### What you must decide

- **How the three paths interact**: what role each path plays, what data flows
  through each, and what happens when one path is unavailable.
- **Which path is authoritative**: when the simulation layer and the JMAP state
  layer disagree on an object's position, which one wins.

### Considerations

A JMAP Scene deployment has three distinct data paths:

```
+------------------------------------------------------------------+
|                         CLIENT                                    |
|                                                                   |
|  +------------------+  +-------------------+  +----------------+  |
|  | JMAP State Layer |  | Simulation Layer  |  | Asset CDN      |  |
|  | (slow, reliable) |  | (fast, real-time) |  | (bulk, cached) |  |
|  +--------+---------+  +---------+---------+  +--------+-------+  |
|           |                      |                     |          |
+-----------+----------------------+---------------------+----------+
            |                      |                     |
            v                      v                     v
    +-------+--------+   +--------+--------+   +--------+-------+
    | JMAP Server     |   | Simulation      |   | CDN / Blob     |
    | (RFC 8620)      |   | Server          |   | Store          |
    | SceneRegion     |   | Position sync   |   | glTF, images,  |
    | SceneObject     |   | Physics         |   | audio          |
    | SceneAvatar     |   | Interactions    |   |                |
    | SceneAsset      |   | Interest mgmt   |   |                |
    +-----------------+   +-----------------+   +----------------+
```

**JMAP state layer.** Slow, reliable, transactional. Handles the questions:
what regions exist? what objects are in them? who is present? This is the
source of truth for object existence, permissions, and metadata. Updates
arrive via standard JMAP request/response or StateChange push. Latency is
measured in hundreds of milliseconds. This is where a client goes to discover
what is in the world before connecting to the simulation layer.

**Simulation layer.** Fast, real-time, ephemeral. Handles the questions: where
is everyone right now? what is moving? what just collided? This is where avatar
positions update at 10-60 Hz, where physics runs, and where interaction events
fire. The endpoint is `simulationUri` on SceneRegion. Latency is measured in
milliseconds. Data on this path is best-effort: a dropped position update is
acceptable because the next one arrives in 50ms.

**Asset CDN.** Bulk, cached, static. Handles the question: what does this
object look like? Asset files (glTF, images, audio) are large and change
infrequently. They are served from `assetUri` on SceneObject/SceneAsset or
from the JMAP blob download endpoint (`blobId`). Latency is measured in
seconds for first load, milliseconds for cache hits. This path is
independent of both the JMAP layer and the simulation layer.

**Why three paths?** Conflating them produces bad outcomes. Pushing 60 Hz
position updates through JMAP request/response would overwhelm the server.
Serving 50 MB glTF files through the simulation WebSocket would starve
position updates. Querying object metadata through the simulation layer would
require reimplementing JMAP's query/filter/sort semantics. Each path has
different latency, bandwidth, reliability, and caching characteristics; keeping
them separate lets you optimize each independently.

**Client connection sequence.** A typical client enters a region in this order:

1. `SceneRegion/get` -- discover the region, its bounds, spawn position,
   `simulationUri`, and optional bindings (chatId, activeCallId).
2. `SceneObject/query` -- fetch objects in the region (possibly filtered by
   spatial proximity to spawn position).
3. Begin asset downloads -- fetch visual assets via `assetUri` or `blobId` for
   each object. These downloads happen in parallel with step 4.
4. Connect to `simulationUri` -- join the simulation layer, receive real-time
   avatar positions, begin sending own position.
5. `SceneAvatar/set create` -- register presence in the region. This may
   happen before or after step 4, depending on whether the deployment wants
   presence to track JMAP state or simulation-layer connection.

### Common patterns

| System | State layer | Simulation layer | Asset delivery |
|---|---|---|---|
| Mozilla Hubs | Reticulum (Phoenix) | Networked A-Frame via WebSocket/WebRTC | CDN (Cloudflare) |
| Second Life | Region server (proprietary) | Region server (UDP) | CDN + local cache |
| Spatial.io | REST API | WebSocket + WebRTC | CDN |
| Game engines (Unity/Unreal) | Database or REST API | Dedicated game server (UDP) | Asset bundles / CDN |

### Recommended starting point

Deploy three separate services: the JMAP server (any RFC 8620 implementation),
a simulation server (see section 6 for architecture options), and a CDN or
static file server for assets. Clients connect to all three. The JMAP server
and simulation server share a database or message bus for state reconciliation
(see section 3). The CDN is fully independent.

---

## 2. simulationUri: discovery and connection

### What the spec leaves open

The spec defines `simulationUri` as "the real-time simulation layer endpoint
for this region. Opaque to the JMAP server. May be a WebSocket URL, a WebRTC
signaling endpoint, a custom UDP endpoint, or any other simulation entry point"
(SceneRegion Object Fields, `simulationUri`). The spec requires that clients
MUST NOT connect without explicit user initiation and that servers MUST NOT
fetch or probe the URI (Security Considerations, simulationUri Is Untrusted).
Beyond that, the spec is silent on URI format, authentication, protocol
negotiation, and connection lifecycle.

### What you must decide

- **URI scheme and format**: what `simulationUri` looks like and what protocol
  it implies.
- **Authentication**: how the simulation layer authenticates the connecting
  client.
- **Connection lifecycle**: how the client handles connection, disconnection,
  and reconnection to the simulation layer.
- **Capability negotiation**: how the client and simulation server agree on
  protocol version, tick rate, and supported features.

### Considerations

**URI scheme.** The scheme signals the transport:

- `wss://sim.example.com/regions/{regionId}` -- WebSocket over TLS. The
  simplest choice for browser clients. Universally supported.
- `https://sim.example.com/signal/{regionId}` -- WebRTC signaling endpoint.
  The client performs SDP offer/answer via HTTP, then communicates over
  WebRTC data channels. Lower latency than WebSocket for position updates.
- `quic://sim.example.com:4433/{regionId}` -- QUIC. Low latency, built-in
  encryption, unreliable datagrams via QUIC datagrams extension. Emerging
  support; not yet universally available in browsers.
- `udp://sim.example.com:7777/{regionId}` -- Raw UDP. Lowest latency; standard
  for game servers. Not available in browsers; requires a native client.

For browser-first deployments, use `wss://`. For native-client deployments
targeting low latency, use WebRTC data channels or UDP. The spec is
transport-agnostic; the URI scheme is the deployment's signaling convention.

**Authentication.** The simulation layer must verify that the connecting client
is the user they claim to be and has access to the region. Options:

- *Token in query parameter*: `wss://sim.example.com/regions/{regionId}?token={jwt}`.
  The JMAP server generates a short-lived JWT at region entry time and includes
  it in `simulationUri` or returns it alongside the SceneAvatar create response.
  Simple, widely compatible.
- *Token in first message*: the client connects to `simulationUri` and sends an
  authentication message as the first frame. The simulation server validates the
  token before accepting further messages. Avoids tokens in URLs (which may be
  logged by proxies).
- *Cookie/session*: if the simulation server shares a session store with the
  JMAP server, the same session cookie works. Tight coupling between servers.

A short-lived JWT (expiring in 60-300 seconds, scoped to the specific regionId)
is the recommended approach. The JMAP server mints the token; the simulation
server validates it. This decouples the two servers while preserving
authentication.

**Connection lifecycle.** The client connects to `simulationUri` when the user
enters a region and disconnects when the user leaves. Between those events:

- The simulation server sends position updates for other avatars and dynamic
  objects.
- The client sends its own position updates.
- Either side may send interaction events.

When the connection drops unexpectedly, the client SHOULD attempt reconnection
with exponential backoff. During the reconnection window, the client can fall
back to JMAP-layer SceneAvatar positions (which are stale but better than
nothing). See section 7 for failure mode details.

**Capability negotiation.** On connection, the simulation server SHOULD send
a capabilities message describing:

- Protocol version.
- Tick rate (how often the server sends updates).
- Supported features (physics, interaction events, spatial audio).
- Region bounds (which the client can cross-check against the JMAP SceneRegion).

This lets clients adapt to different simulation server implementations without
hardcoding assumptions.

### Common patterns

| System | simulationUri equivalent | Auth |
|---|---|---|
| Mozilla Hubs | WebSocket URL to Reticulum | Phoenix token |
| Second Life | UDP endpoint (sim IP:port) | Capability token from login |
| Spatial.io | WebSocket + WebRTC signaling | OAuth token |
| Game servers (Unreal) | UDP endpoint | Login token / session key |

### Recommended starting point

Use `wss://sim.example.com/regions/{regionId}` as the URI format. Authenticate
with a short-lived JWT passed in the first WebSocket message (not in the URL).
The JMAP server mints the token when the client creates a SceneAvatar; the
simulation server validates it on connection. Send a capabilities message from
the server as the first frame after authentication succeeds. Implement client
reconnection with exponential backoff (initial delay 500ms, max delay 30s,
jitter).

---

## 3. State reconciliation

### What the spec leaves open

The spec states that "the server SHOULD periodically reconcile the simulation
layer's state with the JMAP Scene state" with avatar positions updated "at a
deployment-defined interval (RECOMMENDED: every 5-30 seconds)" and objects
moved by the simulation layer updated "via the standard JMAP state-change
mechanism" (Real-Time Simulation Layer, State Reconciliation). The spec does
not prescribe the reconciliation mechanism, the conflict resolution strategy,
or the batching approach.

### What you must decide

- **Reconciliation direction**: does the simulation layer push to JMAP, does
  JMAP pull from the simulation layer, or both?
- **Reconciliation frequency**: how often snapshots flow from the simulation
  layer to JMAP state.
- **Conflict resolution**: when a `SceneObject/set` update arrives via JMAP
  while the simulation layer is actively moving that object, which wins?
- **Batching strategy**: how many position updates are batched into a single
  JMAP state change.

### Considerations

**The simulation layer is the real-time authority; JMAP is the persistence
authority.** During active simulation, the simulation layer knows where
everything is right now. JMAP knows where everything was the last time someone
wrote it down. These are complementary, not competing.

The reconciliation flow looks like this:

```
Simulation Layer                JMAP Server
      |                              |
      |--- position updates (60Hz) --|---> clients (real-time)
      |                              |
      |--- snapshot (every 10s) ---->|---> SceneAvatar.position
      |                              |---> SceneObject.position (dynamic)
      |                              |---> StateChange push to clients
      |                              |
      |<--- SceneObject/set ---------|<--- client (place new object)
      |                              |
      |    (simulation layer picks   |
      |     up new object from DB    |
      |     or via message bus)      |
      |                              |
```

**Simulation-to-JMAP reconciliation.** The simulation server periodically
writes authoritative position snapshots to the JMAP data store. This serves
three purposes:

1. *Persistence*: if the simulation server crashes, positions are recoverable
   from the last snapshot.
2. *Late joiners*: a client that connects to JMAP but has not yet connected to
   the simulation layer can see approximately-current positions.
3. *External queries*: an API consumer that queries SceneAvatar positions via
   JMAP (without connecting to the simulation layer) gets recent data.

The snapshot interval is a trade-off between JMAP server load and data
freshness. At 5 seconds, positions are reasonably current but the JMAP server
handles a write every 5 seconds per active region. At 30 seconds, load is low
but positions are stale. The spec recommends 5-30 seconds; 10 seconds is a
reasonable default.

**Batching.** Do not write one JMAP update per avatar per tick. Batch all
avatar positions in a region into a single snapshot write. If a region has
20 avatars updating at 20 Hz, that is 400 position updates per second on the
simulation layer -- but only one batched JMAP write every 10 seconds containing
20 position values.

**JMAP-to-simulation reconciliation.** When a client creates or moves an object
via `SceneObject/set`, the JMAP server writes the change to the database. The
simulation layer must pick up this change. Options:

- *Database polling*: the simulation server polls the database for changes.
  Simple but adds latency (up to one poll interval).
- *Message bus*: the JMAP server publishes object changes to a message queue
  (Redis pub/sub, NATS, Kafka). The simulation server subscribes and applies
  changes immediately. Lower latency, more infrastructure.
- *Direct API call*: the JMAP server calls the simulation server's internal
  API to notify it of the change. Tightest coupling, lowest latency.

A message bus is the recommended approach: it decouples the servers while
providing near-immediate propagation.

**Conflict resolution.** The spec does not prescribe a conflict resolution
strategy between the simulation layer and JMAP state. The State
Reconciliation section says the server SHOULD reconcile the two, but the
mechanism and conflict policy are deployment-defined. The following is
**recommended policy**, not a spec requirement:

1. A client moves an object via `SceneObject/set` while the simulation layer
   is also moving it (physics, scripted animation). Recommended resolution:
   the JMAP write takes priority. The simulation layer receives the new
   position via the message bus and snaps the object there. This matches user
   expectations: an explicit "move this object" action should override
   automated movement.

2. The simulation layer writes a position snapshot to JMAP, but the object has
   been deleted via `SceneObject/set destroy` since the last snapshot.
   Recommended resolution: the snapshot write fails (object not found); the
   simulation layer removes the object from its local state.

For both cases, last-write-wins with JMAP as the authority for existence and
the simulation layer as the authority for real-time position is the simplest
correct model. Deployments may choose a different policy (for example, a game
server that is fully authoritative may reverse the priority, with the
simulation layer overriding JMAP writes for physics-controlled objects).

**Throttling snapshot writes.** If a region has no avatars and no dynamic
objects, there is nothing to reconcile. Skip snapshot writes for empty or
static regions. Track a "dirty" flag on the simulation side: set it when any
position changes, clear it after a snapshot write.

### Common patterns

| Pattern | Reconciliation mechanism | Typical latency |
|---|---|---|
| Game server + database | Periodic DB write (5-30s) | Snapshot interval |
| Hubs-style | Phoenix PubSub + PostgreSQL | Sub-second for events; periodic for persistence |
| Cloud-native | Redis Streams or NATS between services | < 100ms for events; periodic for snapshots |

### Recommended starting point

Use a message bus (Redis pub/sub or NATS) for JMAP-to-simulation change
propagation. Write simulation-layer position snapshots to the JMAP data store
every 10 seconds, batched per region. Skip snapshot writes for regions with no
position changes since the last snapshot. Use last-write-wins conflict
resolution with JMAP authoritative for object existence and the simulation
layer authoritative for real-time position. Deliver JMAP `StateChange` push
notifications to connected clients after each snapshot write so they can update
their views.

---

## 4. Authority models

### What the spec leaves open

The spec states that "enforcement is simulation-layer-defined" for physics
(SceneObject, `physicsMode`) and that avatar positions are "client-reported"
with the simulation layer responsible for validation (Security Considerations,
Position Spoofing). The spec does not prescribe whether the simulation layer
runs an authoritative server, trusts clients, or uses a hybrid approach.

### What you must decide

- **Who owns positional truth**: the server, the client, or a negotiated
  combination.
- **How cheating and abuse are handled**: what validation the simulation layer
  performs on client-submitted positions.
- **Physics authority**: who runs physics simulation -- server, client, or both.

### Considerations

There are three authority models, each with distinct trade-offs:

**Authoritative server.** The server runs the simulation. Clients send input
(movement intent: "I am pressing forward"). The server computes the resulting
position, applies physics and collision, and sends the authoritative position
back to all clients. Clients predict their own position locally for
responsiveness (client-side prediction) and correct when the server's
authoritative state arrives (server reconciliation).

```
Client A            Server              Client B
   |                  |                    |
   |--move intent---->|                    |
   |  (predict locally)|                   |
   |                  |---(simulate)----   |
   |<--auth position--|--auth position---->|
   |  (reconcile)     |                    |
```

Pros: cheat-proof; consistent state; server controls physics. Cons: higher
server CPU cost; input latency equals round-trip time unless client-side
prediction is implemented; prediction/reconciliation is complex to implement
correctly.

Best for: competitive environments, games with physics, deployments where
position integrity matters (see spec Security Considerations, Position
Spoofing).

**Client-authoritative.** Each client computes its own position locally and
reports it to the server. The server relays reported positions to other
clients. The server may validate bounds (is the reported position inside the
region's bounds?) and rate-limit movement (is the reported velocity physically
plausible?) but does not run its own simulation.

```
Client A            Server              Client B
   |                  |                    |
   |--my position---->|                    |
   |                  |--A's position----->|
   |                  |                    |
   |                  |<--B's position-----|
   |<--B's position---|                    |
```

Pros: low server CPU cost; no prediction/reconciliation complexity; lower
perceived latency (local movement is instant). Cons: clients can lie about
their position; no server-side physics; inconsistent state between clients
if validation is weak.

Best for: social environments, galleries, virtual offices, collaborative
design -- any context where users are not adversarial about position and the
primary concern is smooth movement, not anti-cheat.

**Hybrid.** Clients are authoritative for their own avatar position (within
validated bounds), but the server is authoritative for physics objects, NPC
movement, and interactions that affect shared state. This splits authority by
object type.

The spec defines `physicsMode` values but states that "enforcement is
simulation-layer-defined" (SceneObject, `physicsMode`). The table below is
**recommended practice**, not a spec requirement -- deployments may assign
authority differently based on their needs:

| Object type | Recommended authority |
|---|---|
| Own avatar position | Client (validated by server) |
| Other avatars | Relayed from owning client |
| Static objects (`physicsMode: "static"`) | JMAP state (immutable position) |
| Dynamic objects (`physicsMode: "dynamic"`) | Server (physics simulation) |
| Kinematic objects (`physicsMode: "kinematic"`) | Server (scripted movement) |
| No-collision objects (`physicsMode: "none"`) | JMAP state (moved via `SceneObject/set`; no simulation involvement) |
| Interaction events | Client-initiated, server-validated |

Pros: good balance of responsiveness and consistency; server CPU is spent only
on physics objects, not avatar movement. Cons: more complex than either pure
model; authority boundaries must be clearly defined.

Best for: most deployments. The hybrid model is a natural fit for JMAP Scene
because the spec already distinguishes between avatar positions (client-
reported, per spec) and object physics modes (which signal intent to the
simulation layer). The spec leaves enforcement to the simulation layer, so
the authority assignments above are a recommended starting point rather than
normative requirements. See the [JMAP Scene Games Implementer
Guide](jmap-scene-board-games-guide.md) for concrete examples of how
different game genres assign simulation authority.

**Server-side validation for client-authoritative positions.** Even in a
client-authoritative model, the server SHOULD validate:

- **Bounds checking**: reject positions outside the region's `bounds`.
- **Speed limiting**: reject position changes that imply movement faster than
  a deployment-defined maximum speed (e.g., 10 m/s for walking, 50 m/s for
  flying). This prevents teleportation exploits.
- **Collision with barriers**: for deployments with walls or restricted zones,
  the server may perform lightweight collision checks against static geometry.

These validations do not make the server authoritative -- the client still
computes its own position -- but they bound the range of possible cheating.

### Common patterns

| System | Authority model |
|---|---|
| Second Life | Server-authoritative (region server runs physics) |
| Mozilla Hubs | Client-authoritative (Networked A-Frame relays positions) |
| Fortnite / competitive games | Server-authoritative with client-side prediction |
| Spatial.io | Hybrid (client-authoritative avatars, server-authoritative objects) |
| VRChat | Client-authoritative with anti-cheat validation |

### Recommended starting point

For a social or collaborative deployment: client-authoritative for avatar
positions with server-side bounds checking and speed limiting. Server-
authoritative for dynamic physics objects (if any). For a competitive or
security-sensitive deployment: authoritative server with client-side prediction.
For most first deployments: the hybrid model described above, starting with
client-authoritative avatars and adding server-authoritative physics later.

---

## 5. Latency, interpolation, and dead reckoning

### What the spec leaves open

The spec mentions "10-20 Hz for social VR, 30-60 Hz for competitive
environments" as typical avatar synchronization rates (Real-Time Simulation
Layer, Simulation Layer Responsibilities) but does not prescribe a tick rate,
interpolation strategy, or prediction scheme. These are simulation-layer
concerns.

### What you must decide

- **Tick rate**: how many position updates per second the simulation layer
  sends.
- **Interpolation**: how clients render smooth movement between discrete
  position updates.
- **Dead reckoning / prediction**: how clients estimate remote avatar
  positions between updates.
- **Jitter buffering**: whether and how to buffer incoming updates to smooth
  out network jitter.

### Considerations

**Tick rate.** The tick rate determines how often the simulation server sends
position updates to each client. Higher tick rates mean smoother remote avatar
movement and more responsive interactions, but consume more bandwidth and
server CPU.

| Context | Tick rate | Bandwidth per avatar |
|---|---|---|
| Virtual office / 2D top-down | 5-10 Hz | ~0.5-1 KB/s |
| Social VR / gallery | 10-20 Hz | ~1-2 KB/s |
| Collaborative design | 15-30 Hz | ~1.5-3 KB/s |
| Competitive game | 30-60 Hz | ~3-6 KB/s |

Bandwidth per avatar assumes a compact binary position update (position: 12
bytes, orientation: 16 bytes, velocity: 12 bytes, timestamp: 4 bytes, header:
~4 bytes = ~48 bytes per update). JSON encoding increases this by 3-5x.

Use binary encoding for position updates. JSON is fine for infrequent messages
(object creation, interaction events); it is wasteful for 20 Hz position
streams.

**Interpolation.** The client receives discrete position snapshots at the tick
rate but must render frames at 60+ FPS. Without interpolation, remote avatars
jump between positions. With interpolation, avatars glide smoothly.

The standard approach is **buffered interpolation**: the client renders remote
avatars at a position slightly in the past (one tick interval behind real-time)
and interpolates between the two most recent received positions. This
introduces one tick interval of additional display latency but guarantees
smooth movement as long as updates arrive reliably.

```
Time --->

Server sends:   P0          P1          P2          P3
                |           |           |           |
Client renders:     lerp(P0,P1)   lerp(P1,P2)   lerp(P2,P3)
                    ^-- one tick behind real-time
```

For orientation, use spherical linear interpolation (slerp) between quaternions.
Linear interpolation of quaternion components produces incorrect results.

**Dead reckoning.** When a position update is late or lost, the client must
decide what to render. Dead reckoning extrapolates the avatar's position based
on its last known velocity (and optionally acceleration):

```
predicted_position = last_position + last_velocity * elapsed_time
```

When the next real update arrives, the client blends from the predicted
position to the actual position over a short duration (100-200ms) to avoid a
visible snap.

Dead reckoning is essential for avatars moving in straight lines at constant
speed (walking, flying). It works poorly for avatars that change direction
frequently. For social VR where avatars mostly stand still or walk slowly,
simple linear dead reckoning is sufficient.

**Jitter buffering.** Network jitter causes position updates to arrive at
irregular intervals even when the server sends them at a constant rate.
A jitter buffer holds incoming updates for a short duration (half to one tick
interval) before processing them, smoothing out arrival times. This adds
latency but eliminates the stutter caused by jitter.

Most implementations combine jitter buffering with interpolation: the
interpolation buffer naturally absorbs one tick interval of jitter. Adding
a separate jitter buffer on top of interpolation is usually unnecessary
unless network conditions are particularly poor.

**Adaptive tick rate.** Some deployments vary the tick rate based on distance
from the observing client (see section 6, interest management). Nearby avatars
update at full rate; distant avatars update at half or quarter rate. This
reduces bandwidth without noticeably affecting visual quality, since distant
avatars are smaller on screen and their movement is less perceptible.

### Common patterns

| System | Tick rate | Interpolation | Dead reckoning |
|---|---|---|---|
| Second Life | ~10 Hz (avatars) | Linear interpolation | Velocity-based |
| Mozilla Hubs | ~15 Hz | Buffered interpolation (Networked A-Frame) | None (relies on interpolation buffer) |
| Fortnite | 30 Hz (server tick) | Client-side prediction + reconciliation | Full prediction with rollback |
| VRChat | ~10-15 Hz | Interpolation + IK smoothing | Velocity extrapolation |

### Recommended starting point

Start with 10 Hz tick rate for social/collaborative deployments, 20 Hz for
anything with fast-moving avatars. Use buffered interpolation with a one-tick
delay. Implement simple linear dead reckoning for missed updates with a
200ms blend-to-correction when the real update arrives. Use binary encoding
for position updates (not JSON). Increase tick rate only after profiling
demonstrates that bandwidth and server CPU are not bottlenecks.

For game-specific tick rate guidance (35 Hz for classic FPS, 60 Hz for arcade,
77 Hz for Quake-style), see the [JMAP Scene Games Implementer
Guide](jmap-scene-board-games-guide.md), which covers tick rate selection
per genre in detail.

---

## 6. Visibility and interest management

### What the spec leaves open

The spec defines a two-tier visibility contract (Real-Time Simulation
Layer, Visibility Contract): first, a MUST-level access-control rule ("The
server MUST NOT include a SceneObject in a `SceneObject/get` response or
`SceneObject/query` result if the authenticated user does not have access to
the object's containing SceneRegion"), and second, a SHOULD-level filtering
rule ("The server SHOULD apply visibility filtering to limit the objects
returned based on the authenticated user's avatar position within the
region"). It describes three levels of implementation for the filtering rule
-- return all objects, a radius filter, or server-side occlusion culling --
and leaves the algorithm to the deployment. The spec also lists interest
management as a simulation layer responsibility (Real-Time Simulation Layer,
Simulation Layer Responsibilities).

### What you must decide

- **JMAP-level visibility filtering**: what spatial filtering the JMAP server
  applies to `SceneObject/get` and `SceneObject/query` responses.
- **Simulation-level interest management**: what filtering the simulation layer
  applies to real-time position updates.
- **Whether these two levels share an algorithm** or operate independently.

### Considerations

Visibility filtering operates at two layers, and they serve different purposes:

**JMAP-level filtering** controls what objects the client learns about at all.
A client that never receives a SceneObject record does not know the object
exists. This matters for security (hiding competitive game objects) and for
bandwidth (avoiding thousands of object records for a large region). The spec's
spatial query filters (`withinRadius`, `withinBounds` on SceneObject/query) are
the client-facing mechanism for this.

**Simulation-level interest management** controls which real-time updates the
client receives from the simulation layer. A client that is subscribed to
updates for 500 avatars across a large region wastes bandwidth on avatars it
cannot see. Interest management reduces the update set to what the client
actually needs.

These two levels can operate independently. A simple deployment might return
all objects from JMAP (no filtering) but filter simulation updates by radius.
A security-conscious deployment might filter at both levels.

**Radius-based culling.** The simplest interest management algorithm: send
updates only for entities within a radius R of the observing avatar.

```
+--------------------------------------+
|              Region                   |
|                                       |
|     B                                 |
|      \                                |
|       \  R                            |
|   C----A----D        E                |
|       /                               |
|      /                                |
|     F                                 |
|                                       |
+--------------------------------------+

Avatar A receives updates for B, C, D, F (within radius R).
Avatar A does NOT receive updates for E (outside radius R).
```

Pros: simple to implement; O(n) per avatar per tick; works well for
uniform-density scenes. Cons: avatars just outside the radius pop in/out
abruptly; does not account for line-of-sight; the radius must be tuned per
deployment.

Mitigate pop-in with a **hysteresis band**: an avatar enters the interest set
at radius R but does not leave until radius R + margin (e.g., R + 10%). This
prevents oscillation at the boundary.

**Frustum culling.** Send updates only for entities within the client's view
frustum (the camera's field of view extended into the scene). This is more
aggressive than radius-based culling -- the client receives no updates for
entities behind or beside the camera -- but requires the client to report its
camera orientation to the simulation server (not just position).

Frustum culling is standard in 3D rendering engines but less common in
simulation-layer interest management because the camera can rotate quickly,
causing entities to pop in when the user turns. Combine with a generous radius
as a fallback: frustum-cull within the radius, but keep all entities within a
smaller "always-send" radius regardless of frustum.

**Area-of-interest (AOI) partitioning.** Divide the region into spatial
partitions (a grid, a quadtree, or an octree). Each partition maintains a list
of entities. The simulation server subscribes each client to partitions near
their avatar. When an avatar moves across a partition boundary, the server
updates the subscription.

```
+--------+--------+--------+--------+
|        |        |        |        |
|   P1   |   P2   |   P3   |   P4   |
|        |   A    |        |        |
+--------+--------+--------+--------+
|        |        |        |        |
|   P5   |   P6   |   P7   |   P8   |
|        |        |        |        |
+--------+--------+--------+--------+

Avatar A is in partition P2.
A receives updates from P1, P2, P3, P5, P6, P7 (3x3 neighborhood).
A does NOT receive updates from P4 or P8.
```

Pros: O(1) per avatar per tick (just check partition membership); scales well;
naturally supports sharding (see section 8). Cons: partition boundaries cause
discrete subscription changes; objects near a boundary may pop in/out.

**Combining JMAP and simulation filtering.** For most deployments, JMAP-level
filtering should be loose (return all objects in the region, or all within a
generous radius) and simulation-level filtering should be tight (radius or AOI
with hysteresis). The JMAP response tells the client what exists; the
simulation layer tells it what is moving nearby. Tight JMAP-level filtering is
needed only when object existence itself is sensitive (competitive games,
hidden content).

### Common patterns

| System | Interest management |
|---|---|
| Second Life | Region-grid based; 256m x 256m regions; objects beyond draw distance not sent |
| Mozilla Hubs | No interest management (rooms are small, < 25 users) |
| Fortnite | Relevancy-based: per-actor update frequency based on distance and importance |
| World of Warcraft | Grid-based AOI; entities enter/leave update set as player moves between cells |

### Recommended starting point

For small regions (< 50 users, < 1000 objects): skip interest management
entirely; send all updates to all clients. For medium regions (50-500 users):
implement radius-based culling with hysteresis on the simulation layer; return
all objects from JMAP queries. For large regions (500+ users): implement
grid-based AOI partitioning on the simulation layer; consider JMAP-level
spatial filtering for `SceneObject/get` responses. Start without frustum
culling; add it only if bandwidth measurements show it is needed.

---

## 7. Failure modes and recovery

### What the spec leaves open

The spec states that `simulationUri` is opaque and may be unreachable. It
defines SceneAvatar `leftAt` for tracking departures and the reconnect behavior
(SceneAvatar/set, Reconnecting). It does not specify what clients should do
when the simulation layer fails, how to detect state divergence, or how to
recover from a simulation server crash.

### What you must decide

- **Client behavior when simulationUri is unreachable**: what the client
  renders and what functionality remains.
- **Reconnection strategy**: how the client reconnects to the simulation layer
  after a disconnect.
- **State divergence detection**: how to detect when the simulation layer and
  JMAP state have drifted apart.
- **Simulation server crash recovery**: how the simulation layer recovers its
  state after a restart.

### Considerations

**simulationUri unreachable.** When the client cannot connect to `simulationUri`
(DNS failure, server down, network partition), the client should degrade
gracefully:

1. *Render the scene from JMAP state.* SceneObject positions from
   `SceneObject/get` are available; render a static scene. Avatar positions
   from `SceneAvatar/get` are stale but present; render other avatars at their
   last known positions.
2. *Disable real-time features.* No avatar movement updates, no interaction
   events, no physics. The user can view the scene but not participate in
   real-time.
3. *Display connection status.* Show the user that the simulation layer is
   unavailable and that the scene is in "view only" mode. Do not silently
   degrade.
4. *Continue reconnection attempts.* Use exponential backoff with jitter.

This degradation is natural because the spec separates spatial state (JMAP)
from spatial runtime (simulation). A client with only JMAP access has a
snapshot of the world; it just cannot see it move.

**Regions with no simulation layer.** The spec allows `simulationUri: null`
for "static scene, offline viewing" (SceneRegion Object Fields,
`simulationUri`). Clients must handle this case: render the scene from JMAP
state alone. No avatar position synchronization occurs; other avatars are
visible at their last JMAP-reported position. This is the right mode for
architectural walkthroughs, museums, or any scene where real-time interaction
is not needed.

**Reconnection.** When the simulation layer connection drops:

```
Client                          Simulation Server
  |                                   |
  |--- connection established ------->|
  |<-- position updates --------------|
  |                                   |
  |    *** connection drops ***       |
  |                                   |
  |--- reconnect attempt 1 (500ms) -->| (may fail)
  |--- reconnect attempt 2 (1s) ----->| (may fail)
  |--- reconnect attempt 3 (2s) ----->| (succeeds)
  |                                   |
  |--- auth + last known state ------>|
  |<-- full state snapshot -----------|
  |<-- resume position updates -------|
```

On reconnection, the client should send its last known simulation state
(sequence number, timestamp) so the server can determine what updates the
client missed. The server may respond with a full state snapshot (positions of
all nearby entities) or a delta from the client's last known state.

**State divergence detection.** The JMAP state and simulation state can diverge
in several ways:

- An object was created via `SceneObject/set` but the simulation layer never
  picked it up (message bus failure).
- The simulation layer moved an object, but the snapshot write to JMAP failed
  (database unavailable).
- An avatar's SceneAvatar record shows `leftAt: null` but the simulation layer
  has no active connection for that user (stale JMAP state after a simulation
  server crash).

Detection strategies:

- *Periodic full reconciliation*: every N minutes, the simulation server
  compares its entity list against the JMAP database and resolves differences.
  Heavyweight but thorough.
- *Checksum comparison*: the simulation server computes a hash of its entity
  positions; the JMAP server computes a hash of its stored positions. If they
  diverge beyond the expected snapshot staleness, trigger a full reconciliation.
- *Avatar presence audit*: the simulation server periodically reports its list
  of connected users. The JMAP server compares this against SceneAvatar records
  with `leftAt: null` and marks disconnected avatars as departed.

The avatar presence audit is the most important: stale avatar records (showing
a user as present when they have disconnected) are the most visible divergence
to other users.

**Simulation server crash recovery.** When the simulation server restarts:

1. Load the last known state from the JMAP database (or from its own persistent
   store, if it has one).
2. Resume accepting client connections.
3. Clients reconnect (they will have been attempting reconnection with backoff).
4. Run a full reconciliation against JMAP state.

The recovery time depends on how much state the simulation server needs to
rebuild. A stateless relay server (client-authoritative, no physics) recovers
instantly: it just starts accepting connections again. A server-authoritative
server with physics state needs to reload the physics world from the JMAP
database, which may take seconds.

### Common patterns

| Failure mode | Recovery pattern |
|---|---|
| Simulation server down | Client shows static scene from JMAP; reconnects with backoff |
| Network partition (client) | Client-side dead reckoning for local avatar; stale display for remote avatars |
| Database unavailable | Simulation server continues in-memory; queues snapshot writes; retries |
| Message bus down | JMAP changes queue locally; simulation server polls DB as fallback |

### Recommended starting point

Implement client reconnection with exponential backoff (500ms initial, 30s max,
jitter). On reconnect, send a full state snapshot from the simulation server.
Run an avatar presence audit every 60 seconds: compare the simulation server's
connected user list against SceneAvatar records and set `leftAt` on any stale
records. Handle `simulationUri: null` as a static scene render mode (no
simulation connection attempt). Display connection status to the user at all
times.

---

## 8. Transport options

### What the spec leaves open

The spec states it "does not prescribe WebRTC data channels, UDP, QUIC,
WebSocket, or any specific transport" (Real-Time Simulation Layer). The
`simulationUri` scheme is the only hint to the client about what transport
to expect.

### What you must decide

- **Primary transport protocol**: what the simulation layer uses for real-time
  position updates.
- **Reliability requirements**: whether position updates need reliable delivery
  or can be best-effort (unreliable).
- **Browser compatibility**: whether the simulation layer must work in browsers
  without plugins.

### Considerations

**WebSocket (WSS).** Reliable, ordered, TCP-based. The simplest option for
browser clients.

Pros: universal browser support; simple to implement; works through firewalls
and proxies; TLS built in. Cons: TCP head-of-line blocking adds latency under
packet loss; reliable delivery wastes bandwidth on stale position updates that
are superseded before arrival.

For social environments with < 50 users and 10 Hz tick rates, WebSocket is
adequate. Head-of-line blocking is noticeable only under significant packet
loss (> 2-3%).

**WebRTC data channels.** Unreliable and unordered mode available. NAT
traversal built in (ICE).

Pros: unreliable/unordered mode eliminates head-of-line blocking; NAT traversal
handles peer-to-peer and client-to-server connectivity; DTLS encryption
built in; browser-native. Cons: more complex setup (signaling, ICE, DTLS
handshake); TURN servers needed for fallback; not a simple listen-on-port
model.

For deployments where latency matters (VR, fast-moving scenes), WebRTC data
channels in unreliable mode are the right choice. Use a signaling endpoint at
`simulationUri` (HTTPS) to exchange SDP offers/answers, then communicate over
the data channel.

**QUIC.** UDP-based, multiplexed streams, built-in encryption. Supports
unreliable datagrams (RFC 9221).

Pros: no head-of-line blocking between streams; unreliable datagrams available;
0-RTT connection establishment; built-in congestion control. Cons: limited
browser support for raw QUIC (WebTransport is the browser API, still maturing);
requires native client or WebTransport-capable browser; fewer production
libraries than WebSocket or WebRTC.

QUIC via WebTransport is a strong future choice. For current deployments, it
is viable only with native clients or browsers that support WebTransport.

**Raw UDP.** The game server standard. Unreliable, unordered, minimal overhead.

Pros: lowest possible latency; complete control over framing and congestion;
mature ecosystem (ENet, GameNetworkingSockets). Cons: not available in browsers;
must implement encryption (DTLS or custom); must implement congestion control;
firewall traversal is harder.

For native-client-only deployments (VR headsets, desktop game clients), raw UDP
with a library like ENet or Valve's GameNetworkingSockets is the standard
choice.

**Summary matrix:**

| Transport | Browser | Unreliable mode | NAT traversal | Complexity |
|---|---|---|---|---|
| WebSocket | Yes | No | N/A (TCP) | Low |
| WebRTC data channel | Yes | Yes | Built-in (ICE) | Medium |
| QUIC (native) | No | Yes (RFC 9221 datagrams) | N/A | Medium-High |
| WebTransport | Partial (Chrome, Edge; not yet Safari/Firefox) | Yes (datagrams + unidirectional streams) | N/A | Medium |
| Raw UDP | No | Yes | Manual (STUN/TURN) | High |

**WebTransport note.** WebTransport is a browser API layered on HTTP/3 (QUIC).
It provides both reliable streams and unreliable datagrams from browser
JavaScript -- something WebSocket and WebRTC data channels cannot both offer
cleanly. Browser support is partial: Chromium-based browsers (Chrome, Edge)
ship WebTransport; Safari and Firefox have it behind flags or in development
as of mid-2026. For deployments that can require Chromium or that serve both
browser and native clients, WebTransport is a strong middle ground between
WebSocket (reliable-only, browser-universal) and raw QUIC (full control,
native-only).

### Recommended starting point

For browser-first deployments: WebSocket for the first version. Move to WebRTC
data channels (unreliable mode) when latency measurements show WebSocket is
insufficient. For native-client deployments: WebRTC data channels or raw UDP
depending on team expertise. For future-proofing: design the simulation
protocol as transport-agnostic (a message framing layer that can sit on any
of these transports) so you can swap transports without rewriting the
simulation logic.

---

## 9. Example architectures

### What the spec leaves open

The spec is architecture-agnostic. It defines the data model and the
simulation layer's responsibilities but not the system topology. This section
presents three reference architectures at increasing complexity.

### Architecture A: WebSocket relay (simplest)

For small deployments: virtual offices, galleries, small meeting spaces.
Up to ~50 concurrent users per region.

```
+----------+     +----------+     +----------+
| Client A |     | Client B |     | Client C |
+-----+----+     +-----+----+     +-----+----+
      |                |                |
      |    WSS         |    WSS         |    WSS
      |                |                |
+-----+----------------+----------------+----+
|              WebSocket Relay Server         |
|                                             |
|  - Receives position from each client       |
|  - Broadcasts to all other clients          |
|  - Validates bounds + speed                 |
|  - Writes snapshots to DB every 10s         |
|  - No physics, no authority                 |
+---------------------+----------------------+
                      |
                      | DB writes
                      v
              +-------+--------+
              |  JMAP Server   |
              |  (SceneRegion, |
              |   SceneObject, |
              |   SceneAvatar) |
              +----------------+
```

Authority model: client-authoritative. Each client reports its own position;
the relay broadcasts it. The relay validates bounds and speed but does not
simulate.

Pros: simple to build; a single Node.js or Go process handles the relay; low
server CPU. Cons: no physics; no anti-cheat beyond bounds checking; does not
scale beyond ~50 users without sharding.

Implementation notes:
- The relay server is a WebSocket server that maintains a set of connected
  clients per region.
- On receiving a position update from a client, validate it and broadcast
  to all other clients in the same region.
- Every 10 seconds, batch-write all current positions to the JMAP database.
- Use binary framing for position updates (see section 5).

### Architecture B: authoritative game server

For competitive or physics-heavy deployments: games, training simulations,
physics-based collaboration. Up to ~100 users per region with physics.

```
+----------+     +----------+     +----------+
| Client A |     | Client B |     | Client C |
+-----+----+     +-----+----+     +-----+----+
      |                |                |
      |    UDP/WebRTC  |    UDP/WebRTC  |    UDP/WebRTC
      |                |                |
+-----+----------------+----------------+----+
|           Authoritative Game Server         |
|                                             |
|  - Receives input from clients              |
|  - Runs physics simulation (Rapier/PhysX)   |
|  - Computes authoritative positions         |
|  - Sends state updates to clients           |
|  - Runs interaction event logic             |
|  - Writes snapshots to DB every 10s         |
+---------------------+----------------------+
                      |
            DB writes + message bus
                      v
              +-------+--------+
              |  JMAP Server   |
              +-------+--------+
                      |
                      v
              +-------+--------+
              |  Message Bus   |
              | (Redis/NATS)   |
              +----------------+
```

Authority model: server-authoritative. Clients send input; the server computes
state. Clients run prediction locally and reconcile with server state.

Pros: consistent physics; anti-cheat; server controls all shared state. Cons:
higher server CPU; requires client-side prediction implementation; more complex
to build and operate.

Implementation notes:
- The game server runs a physics engine (Rapier for Rust, PhysX for C++,
  Cannon.js for Node.js) at 30-60 Hz.
- Static objects (`physicsMode: "static"`) are loaded from JMAP state at
  startup and placed as static colliders.
- Dynamic objects (`physicsMode: "dynamic"`) are simulated by the physics
  engine.
- Avatar movement: clients send movement intent (direction + speed); the
  server applies it as a kinematic force and checks collision.
- State snapshots are written to the JMAP database every 10 seconds via the
  message bus.

### Architecture C: peer-to-peer mesh (small rooms)

For very small deployments: 2-6 user collaboration, pair design reviews,
intimate social spaces. No server-side simulation at all.

```
+----------+           +----------+
| Client A +-----------+ Client B |
+-----+----+           +----+-----+
      |   \           /     |
      |    \         /      |
      |     \       /       |
      |      +-----+        |
      |      |Client|       |
      |      |  C   |       |
      |      +------+       |
      |                     |
      |  JMAP for state     |
      +----------+----------+
                 |
         +-------+--------+
         |  JMAP Server   |
         +----------------+
```

Authority model: client-authoritative with direct peer exchange. Each client
sends its position directly to every other client via WebRTC data channels.
No simulation server exists; `simulationUri` points to a lightweight signaling
server that helps clients discover each other and exchange SDP offers.

Pros: zero simulation server cost; lowest latency (direct peer connection);
simple for tiny groups. Cons: scales as O(n^2) connections; no server-side
validation; NAT traversal failures are more likely; no physics beyond what each
client computes locally.

Implementation notes:
- `simulationUri` is a WebRTC signaling endpoint.
- The signaling server maintains a list of peers per region and relays SDP
  offers/answers.
- Once peers are connected, position updates flow directly peer-to-peer.
- Each client writes its own SceneAvatar position to the JMAP server
  periodically (every 10-30 seconds) as a persistence snapshot.
- If the signaling server is down, clients cannot discover new peers but
  existing peer connections continue to function.

### Choosing an architecture

| Factor | A: WebSocket relay | B: Game server | C: Peer-to-peer |
|---|---|---|---|
| Max users per region | ~50 | ~100 (with physics) | ~6 |
| Server CPU cost | Low | High | None |
| Physics | None | Full | Client-only |
| Anti-cheat | Bounds only | Server-authoritative | None |
| Browser support | Yes (WSS) | Yes (WebRTC) or No (UDP) | Yes (WebRTC) |
| Complexity to build | Low | High | Medium |
| Best for | Social, galleries | Games, training | Pair/small-group collab |

---

## 10. Scaling: sharding and region handoff

### What the spec leaves open

The spec defines SceneRegion as a bounded spatial container with its own origin
and coordinate system (SceneRegion Object Fields). It does not address how
regions map to simulation servers, how to handle regions too large for a single
server, or what happens when an avatar moves between regions.

### What you must decide

- **Region-to-server mapping**: how regions are assigned to simulation servers.
- **Sub-region sharding**: whether a single large region can be split across
  multiple simulation servers.
- **Region handoff**: what happens when an avatar transitions from one region
  (and one simulation server) to another.
- **Cross-region visibility**: whether entities in adjacent regions are visible
  to each other.

### Considerations

**Region-to-server mapping.** The simplest model: one simulation server per
region. The `simulationUri` on each SceneRegion points to the server assigned
to that region. A load balancer or orchestrator assigns regions to servers and
updates `simulationUri` when a server is added or removed.

```
+----------------+     +----------------+     +----------------+
| Sim Server 1   |     | Sim Server 2   |     | Sim Server 3   |
| Region A       |     | Region B       |     | Region C       |
| Region D       |     | Region E       |     | Region F       |
+----------------+     +----------------+     +----------------+
```

For regions with low user counts, multiple regions can share a simulation
server. For regions with high user counts, a dedicated server per region.
Use a consistent hashing scheme to distribute regions across servers, with
rebalancing when servers are added or removed.

**Sub-region sharding.** When a single region is too large or too populated
for one simulation server (hundreds of users, millions of square meters),
split the region's spatial volume across multiple simulation servers. Each
server handles a spatial partition (a shard) of the region.

```
+--------------------------------------+
|           SceneRegion "World"        |
|                                      |
|  +----------+  +----------+         |
|  | Shard NW |  | Shard NE |         |
|  | Server 1 |  | Server 2 |         |
|  +----------+  +----------+         |
|  +----------+  +----------+         |
|  | Shard SW |  | Shard SE |         |
|  | Server 3 |  | Server 4 |         |
|  +----------+  +----------+         |
|                                      |
+--------------------------------------+
```

From the JMAP perspective, this is still one SceneRegion with one `simulationUri`.
The `simulationUri` points to a routing layer that assigns the client to the
correct shard server based on the client's avatar position. When the avatar
moves across a shard boundary, the routing layer hands the client off to the
new shard server.

Sub-region sharding is complex. It requires:
- A routing layer that tracks avatar positions and assigns shard membership.
- A boundary overlap zone where both shard servers replicate entity state (so
  avatars near the boundary can see entities on the other side).
- A handoff protocol that migrates the client's simulation connection from one
  server to another without visible interruption.

Do not implement sub-region sharding unless you need it. Most deployments can
avoid it by keeping regions reasonably sized (the spec's SceneBounds is
region-local, so regions can be any size -- but practical regions are typically
100m-1000m on a side).

**Region handoff.** When an avatar moves from Region A to Region B, the client
must:

1. Set `leftAt` on the SceneAvatar in Region A (or the server does this
   automatically per the spec: "the server MUST set `leftAt` on the previous
   avatar before creating the new one" when a user enters a new region,
   SceneAvatar/set, Entering a Region).
2. Create a new SceneAvatar in Region B via `SceneAvatar/set create`.
3. Disconnect from Region A's `simulationUri`.
4. Connect to Region B's `simulationUri`.
5. Begin receiving simulation updates for Region B.

This is a visible transition: the client disconnects from one simulation server
and connects to another. For a seamless experience, the client should:
- Pre-fetch Region B's objects and assets before the transition (if the
  destination is known in advance, e.g., the avatar is walking toward a portal).
- Connect to Region B's simulation server before disconnecting from Region A's,
  overlapping the connections briefly.
- Render Region B's scene from JMAP state while the simulation connection is
  being established.

**Cross-region visibility.** The spec defines regions as independent coordinate
spaces with no built-in cross-region references. If adjacent regions need to
be visually connected (e.g., looking through a window from one region into
another), the simulation layer must handle this outside the JMAP model:
- Subscribe the client to updates from the adjacent region's simulation server
  (limited to entities near the boundary).
- Transform positions from the adjacent region's coordinate space into the
  local region's coordinate space.

This is niche. Most deployments treat regions as independent spaces with
explicit transitions (portals, doors, teleportation).

**Instance management.** For regions that need multiple independent copies
(e.g., a game lobby that spawns a fresh instance per match), the deployment
creates multiple SceneRegion objects with the same content but different ids
and `simulationUri` values. The JMAP model supports this naturally: each
instance is a separate SceneRegion. A matchmaking layer assigns users to
instances.

### Common patterns

| System | Scaling model |
|---|---|
| Second Life | Fixed 256m x 256m regions, one region server each, handoff on crossing |
| EVE Online | Solar systems = regions; each on one server; player-driven load spikes handled by "time dilation" |
| Fortnite | One game server per match instance (100 players); no spatial sharding |
| World of Warcraft | Continental regions sharded into zones; seamless handoff with overlap zones |
| Mozilla Hubs | One room = one region; no handoff; rooms are independent |

### Recommended starting point

Start with one simulation server per region. Use a deployment orchestrator
(Kubernetes, Nomad) to assign regions to servers and scale the server pool.
Point `simulationUri` at the assigned server's endpoint. Implement region
handoff as a clean disconnect/reconnect sequence (steps 1-5 above) with
asset pre-fetching for the destination region. Do not implement sub-region
sharding until a single region demonstrably cannot fit on one server. Do not
implement cross-region visibility unless adjacent-region views are a product
requirement.

---

## Appendix: Decision checklist

Before deploying JMAP Scene with a simulation layer to production, verify that
your implementation has made and documented each of the following decisions:

**Three-path architecture (section 1)**
- [ ] JMAP server, simulation server, and asset delivery are deployed and
  independently reachable
- [ ] Client connection sequence documented (JMAP first, then assets + simulation)
- [ ] Graceful degradation defined for each path being unavailable

**simulationUri (section 2)**
- [ ] URI scheme and format defined and documented
- [ ] Authentication mechanism chosen (JWT, first-message token, or session)
- [ ] Token lifetime and scope configured
- [ ] Client reconnection strategy implemented (exponential backoff with jitter)
- [ ] Capability negotiation message defined

**State reconciliation (section 3)**
- [ ] Reconciliation direction defined (simulation-to-JMAP and JMAP-to-simulation)
- [ ] Snapshot write interval configured (recommended: 10s)
- [ ] Message bus or polling mechanism for JMAP-to-simulation propagation
- [ ] Conflict resolution policy documented (last-write-wins or otherwise)
- [ ] Snapshot writes skipped for idle regions

**Authority model (section 4)**
- [ ] Authority model chosen (server-authoritative, client-authoritative, hybrid)
- [ ] Server-side validation rules documented (bounds, speed, collision)
- [ ] Per-object authority defined for each `physicsMode` value
- [ ] Client-side prediction implemented (if server-authoritative)

**Latency and interpolation (section 5)**
- [ ] Tick rate chosen and documented
- [ ] Position update encoding format defined (binary recommended)
- [ ] Interpolation strategy implemented (buffered interpolation recommended)
- [ ] Dead reckoning implemented for missed updates
- [ ] Adaptive tick rate considered (distance-based)

**Visibility and interest management (section 6)**
- [ ] JMAP-level spatial filtering policy documented
- [ ] Simulation-level interest management algorithm chosen (radius, AOI, or none)
- [ ] Hysteresis margin configured for interest set boundaries
- [ ] Interest management disabled for small regions (< 50 users)

**Failure modes (section 7)**
- [ ] Client behavior defined for unreachable `simulationUri`
- [ ] Static scene render mode implemented for `simulationUri: null`
- [ ] Reconnection with full state snapshot implemented
- [ ] Avatar presence audit scheduled (recommended: every 60s)
- [ ] State divergence detection implemented (at minimum: avatar presence audit)

**Transport (section 8)**
- [ ] Transport protocol chosen (WebSocket, WebRTC, QUIC, UDP)
- [ ] Browser compatibility verified if browser clients are required
- [ ] Unreliable delivery mode evaluated for position updates
- [ ] Protocol designed to be transport-swappable (recommended)

**Architecture (section 9)**
- [ ] Architecture chosen (relay, authoritative, peer-to-peer, or custom)
- [ ] Maximum users per region validated by load testing
- [ ] Physics engine integrated (if applicable)

**Scaling (section 10)**
- [ ] Region-to-server mapping strategy defined
- [ ] Orchestrator configured for server pool scaling
- [ ] Region handoff sequence implemented (disconnect/reconnect with pre-fetch)
- [ ] Sub-region sharding decision made (implement or explicitly defer)
- [ ] Instance management strategy defined (if applicable)
