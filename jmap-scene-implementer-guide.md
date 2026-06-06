# JMAP Scene — Implementer's Guide

For server and client implementers of `draft-atwood-jmap-scene-00`. Covers
the spatial state model, lifecycle, access control, simulation layer
integration, and operational decisions that the spec deliberately leaves to
implementations.

Read the draft first. This guide does not re-state normative requirements. It
covers what the spec *deliberately leaves open* and what implementations must
decide before shipping.

---

## How to read this guide

The JMAP Scene draft defines spatial state as JMAP data types — regions,
objects, avatars, assets — with standard get/set/changes/query methods. It
deliberately says nothing about rendering engines, simulation protocols,
physics engines, or visual asset pipelines. Everything between the
`simulationUri` and the actual triangles on screen is out of scope.

This is not a free pass. An implementation that ignores a deferred decision
will deliver a broken or surprising product: avatars that pile up because
the one-per-region constraint was not enforced, spatial queries that return
the entire region because `withinRadius` was treated as optional, or objects
with quaternions that drift from unit length because normalization was skipped.
Implementations must make each of these decisions explicitly, document them,
and implement them coherently.

Each section below follows the same shape:

1. **What the spec leaves open** — with a draft section citation, so you can
   read the normative text yourself.
2. **What you must decide** — the concrete deployment choice you cannot avoid.
3. **Considerations** — the trade-offs that inform the choice.
4. **Common patterns** — how production spatial systems handle this.
5. **Recommended starting point** — a defensible default. Not normative; you
   may choose otherwise with good reason.

When two sections interact (for example, access control decisions constrain
what spatial queries return), cross-references are explicit.

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
  choice (e.g., "servers SHOULD rate-limit object creation"), the keyword
  reflects implementer best-practice. Deviation does not affect protocol
  interoperability.
- Cite the spec, not the guide, when claiming normative authority.

---

## 1. Capability advertisement

### What the spec leaves open

The spec requires advertising `urn:ietf:params:jmap:scene` in the JMAP
Session object (section 4). The session-level capability value is an empty
object `{}`. The account-level capability object carries quota limits,
permission flags, and the `supportedVisualTypes` array. The spec does not
prescribe how these values are configured or updated at runtime.

### What you must decide

- **How to populate `supportedVisualTypes`**: which media types your
  deployment accepts for visual assets. The spec requires at least
  `"model/gltf-binary"`.
- **How to configure quota values**: `maxRegionsPerAccount`,
  `maxObjectsPerRegion`, `maxAvatarsPerRegion`, `maxAssetSizeBytes`. Whether
  these are static configuration or per-account overrides.
- **When to set `mayCreateRegion` and `mayCreateObject`**: whether all
  authenticated users can create spatial content, or whether creation
  requires a specific role or subscription tier.

### Considerations

**`supportedVisualTypes` is your format contract.** Every media type listed
here is a promise that the server will accept assets of that type and that
clients should be prepared to render them. Start with `"model/gltf-binary"`
(mandatory) and add types only when your client ecosystem can consume them.
Adding `"model/gltf+json"` is reasonable; adding `"model/usd"` or
`"application/x-gaussian-splat"` signals to clients that they need renderers
for those formats.

**Quota values drive capacity planning.** `maxObjectsPerRegion` directly
affects spatial query performance: a region with 10,000 objects requires
spatial indexing (see section 6). `maxAvatarsPerRegion` drives simulation
layer capacity. Set these conservatively at launch and raise them as you
understand your performance envelope.

**Per-account vs. global quotas.** The spec defines quotas at the account
level, which allows per-tenant configuration in a multi-tenant deployment. A
free tier might set `maxRegionsPerAccount: 3` and `maxObjectsPerRegion: 100`;
a paid tier might set `null` (unlimited) for both. Implement quota lookup as
a function of the account, not a global constant.

### Recommended starting point

Advertise `supportedVisualTypes: ["model/gltf-binary"]`. Set
`maxRegionsPerAccount: 10`, `maxObjectsPerRegion: 500`,
`maxAvatarsPerRegion: 50`, `maxAssetSizeBytes: 52428800` (50 MB). Set
`mayCreateRegion` and `mayCreateObject` to `true` for all authenticated users.
Adjust after observing real usage.

Example session response:

```json
{
  "capabilities": {
    "urn:ietf:params:jmap:scene": {}
  },
  "accounts": {
    "account-xyz": {
      "accountCapabilities": {
        "urn:ietf:params:jmap:scene": {
          "mayCreateRegion": true,
          "mayCreateObject": true,
          "supportedVisualTypes": ["model/gltf-binary"],
          "maxRegionsPerAccount": 10,
          "maxObjectsPerRegion": 500,
          "maxAvatarsPerRegion": 50,
          "maxAssetSizeBytes": 52428800
        }
      }
    }
  }
}
```

---

## 2. SceneRegion lifecycle

### What the spec leaves open

The spec defines SceneRegion fields, creation, update, and destruction
(section 5.3, section 6.2), including the `bounds`, `accessPolicy`,
`viewHint`, `environment`, `simulationUri`, and spawn point fields. It
defines three access policies (`"public"`, `"invite"`, `"space"`) and
advisory view hints (`"3d"`, `"2d-topdown"`, `"2d-side"`). What the spec
leaves open is how to enforce access policies beyond the JMAP layer, how
to manage the `simulationUri` lifecycle, and how `environment` is structured.

### What you must decide

- **Access policy enforcement beyond JMAP**: how `"invite"` invitations are
  issued and tracked, how `"space"` membership is resolved when JMAP Chat is
  co-deployed.
- **`simulationUri` provisioning**: whether the simulation endpoint is
  pre-provisioned, created on demand when the first avatar enters, or
  supplied by the region creator.
- **`environment` schema**: what keys your deployment recognizes (sky,
  lighting, gravity, fog, etc.) and how clients discover the schema.
- **Region destruction cascade**: how to handle active avatars and objects
  when a region is destroyed.
- **`viewHint` handling**: whether your client renders differently for
  `"2d-topdown"` and `"2d-side"` regions, or treats everything as `"3d"`.

### Considerations

**Access policy: `"invite"`.** The spec says only explicitly granted users
may enter, but does not define the invitation mechanism. Common approaches:

- An invite list stored as a server-side ACL (a set of userIds). The region
  owner manages the list via a deployment-defined API or via `customProperties`
  on the region.
- An invitation token: the region owner generates a single-use or multi-use
  URL that grants access. The server records the grant when the token is
  redeemed.
- Integration with an external identity provider or group membership system.

Whichever mechanism you choose, the server must be able to answer "does this
userId have access to this regionId?" in the `SceneAvatar/set create` handler
and in every get/query handler that checks region access.

**Access policy: `"space"`.** This policy only functions when
`urn:ietf:params:jmap:chat` is co-deployed and `spaceId` is set on the
region. The server resolves membership by querying the Space's member list.
If JMAP Chat is not present and a region is created with `accessPolicy:
"space"`, the server SHOULD reject the create with `invalidArguments` (there
is no Space to bind to).

**`simulationUri` provisioning.** Three common patterns:

| Pattern | When to use |
|---|---|
| Pre-provisioned per region | Dedicated simulation infrastructure; regions are long-lived. |
| On-demand at first avatar entry | Cost-sensitive; spin up simulation instances only when needed. |
| Creator-supplied | The region creator runs their own simulation server and provides the URI. |

For on-demand provisioning, the server sets `simulationUri: null` at region
creation, then populates it when the first `SceneAvatar/set create` arrives.
The avatar creation response includes the now-populated `simulationUri` (via
a back-reference or by fetching the region). Clients SHOULD handle a region
with `simulationUri: null` as a static scene (no real-time interaction).

**`environment` is opaque but should be documented.** The spec says
`environment` is deployment-defined and clients must ignore unrecognized keys.
Document your schema so third-party clients can use it. A minimal schema
might include:

```json
{
  "skyColor": "#87CEEB",
  "ambientIntensity": 0.6,
  "gravity": 9.81,
  "fogDensity": 0.0
}
```

**Region destruction cascade.** The spec requires that destroying a region
removes all contained SceneObject and SceneAvatar records and ejects active
avatars (setting `leftAt`). Implement this as a server-side cascade. If the
region has an active `simulationUri`, notify the simulation layer to
disconnect all clients before destroying the JMAP records.

### Common patterns

| System | Region model |
|---|---|
| Mozilla Hubs | Rooms with a URL; anyone with the URL can join; optional pin code. |
| Second Life | Named regions on a grid; land ownership controls entry. |
| Gather | Spaces with rooms; Space membership controls access. |
| VRChat | Worlds with instances; public or invite-only. |

### Recommended starting point

For `"invite"` access: store an invite list as a server-side ACL (a Set of
userIds on the region). Expose add/remove operations as a deployment-defined
extension or through the region's `customProperties`. For `"space"` access:
resolve Space membership via the Chat layer's permission resolver. For
`simulationUri`: start with creator-supplied URIs; add on-demand provisioning
when you have a simulation orchestration layer. For `environment`: define a
minimal schema, document it, and version it. For destruction: cascade
synchronously within the `SceneRegion/set destroy` handler.

Example region creation:

```json
[["SceneRegion/set", {
  "accountId": "account-xyz",
  "create": {
    "r0": {
      "name": "Design Review Room",
      "bounds": {
        "min": [-20, 0, -20],
        "max": [20, 8, 20]
      },
      "spawnPosition": [0, 0, 5],
      "spawnOrientation": [0, 0, 0, 1],
      "accessPolicy": "invite",
      "viewHint": "3d",
      "simulationUri": "wss://sim.example.com/rooms/design-review"
    }
  }
}, "0"]]
```

---

## 3. SceneObject CRUD

### What the spec leaves open

The spec defines SceneObject fields and the standard JMAP set operations
(section 5.2, section 6.3). It specifies the permission model (owner, region
owner, admin), the scene graph hierarchy via `parentId`, physics modes, and
the `customProperties` extension point. What the spec leaves open is how to
handle object creation at scale, how `customProperties` is validated, and how
the simulation layer is notified of JMAP-side object changes.

### What you must decide

- **Object creation validation**: what checks beyond the spec-required ones
  to perform (position within bounds, asset existence, custom property size
  limits).
- **`customProperties` policy**: maximum size, allowed key patterns, whether
  the server validates structure or stores opaquely.
- **Scene graph depth limit**: maximum parent-child nesting depth.
- **Simulation layer synchronization**: how and when the simulation layer
  learns about objects created or modified via JMAP.
- **Concurrent modification**: how to handle two users updating the same
  object simultaneously.

### Considerations

**Position validation.** The spec says objects SHOULD have positions inside
the region's `bounds` and the server MAY clamp or reject out-of-bounds
positions. Choose one strategy and apply it consistently:

- *Clamp*: silently move the position to the nearest point inside bounds.
  Simpler for clients but can produce surprising results (an object intended
  for `[600, 0, 0]` in a region bounded at `[-500, ..., 500]` silently
  appears at `[500, 0, 0]`).
- *Reject*: return `invalidArguments`. Stricter but forces clients to check
  bounds before creating objects.

Clamping is friendlier for interactive building tools; rejection is safer for
automated pipelines.

**`customProperties` limits.** Without limits, a malicious user could store
megabytes of arbitrary JSON in `customProperties`. Set a maximum serialized
size (RECOMMENDED: 64 KB) and enforce it in the `SceneObject/set` handler.
The spec says the server stores and relays without interpretation; you are
not required to validate the structure, but you are responsible for bounding
the size.

**Scene graph depth.** Deep parent-child hierarchies complicate transform
computation and cascade deletion. A depth limit of 16 levels is generous for
any practical scene graph. Enforce this by walking the `parentId` chain on
create and returning `invalidArguments` if the depth would exceed the limit.

**Simulation layer notification.** When an object is created, moved, or
destroyed via JMAP, the simulation layer may need to update its internal
state. Two patterns:

- *Push*: the JMAP server sends a notification (webhook, message queue event)
  to the simulation layer on every SceneObject state change.
- *Poll*: the simulation layer uses `SceneObject/changes` to poll for updates
  at a regular interval.

Push is more responsive; poll is simpler and decouples the two systems. For
static scenes (most objects are `physicsMode: "static"` and rarely change),
poll is sufficient. For dynamic scenes with frequent JMAP-side edits, push
is better.

**Concurrent modification.** JMAP uses `ifInState` for optimistic
concurrency. When two users update the same object, one will receive a
`stateMismatch` error and must retry. This is the standard JMAP pattern;
no Scene-specific handling is needed. For simulation-layer-driven updates
(the simulation layer moving a `"dynamic"` object), the simulation layer
should be the sole writer for position/orientation to avoid conflicts.

**`visualRef` and `visualType` coupling.** The spec requires that both are
present together or both absent. Enforce this validation before processing the
create or update. An object with `visualRef` but no `visualType` is
unrenderable; the reverse is meaningless.

### Common patterns

| Concern | Pattern |
|---|---|
| Position validation | Reject out-of-bounds; return `invalidArguments` with the violated axis. |
| Custom properties | Store opaquely; enforce 64 KB size limit; no schema validation. |
| Scene graph | Depth limit of 8-16; cascade delete children on parent destroy. |
| Simulation sync | Push via message queue for dynamic scenes; poll for static. |

### Recommended starting point

Reject out-of-bounds positions with `invalidArguments`. Enforce a 64 KB limit
on serialized `customProperties`. Set a scene graph depth limit of 16.
Synchronize the simulation layer via polling (`SceneObject/changes`) for the
first version; add push notifications when latency becomes a problem. Use
`ifInState` for all `SceneObject/set` calls; do not implement custom locking.

Example object creation:

```json
[["SceneObject/set", {
  "accountId": "account-xyz",
  "create": {
    "o0": {
      "regionId": "01J5ABC0000000000000000001",
      "name": "Conference Table",
      "position": [0, 0, 0],
      "orientation": [0, 0, 0, 1],
      "scale": [1, 1, 1],
      "visualRef": "Gc0f032d390a5d5fa8a35",
      "visualType": "model/gltf-binary",
      "physicsMode": "static",
      "interactable": false,
      "customProperties": {
        "material": "walnut",
        "seats": 8
      }
    }
  }
}, "0"]]
```

Example response:

```json
[["SceneObject/set", {
  "accountId": "account-xyz",
  "oldState": "state-001",
  "newState": "state-002",
  "created": {
    "o0": {
      "id": "01J5OBJ0000000000000000042",
      "ownerId": "user:bob@example.com",
      "createdAt": "2026-06-06T10:00:00Z",
      "updatedAt": "2026-06-06T10:00:00Z"
    }
  },
  "notCreated": {}
}, "0"]]
```

Example object update (reposition and change physics mode):

```json
[["SceneObject/set", {
  "accountId": "account-xyz",
  "ifInState": "state-002",
  "update": {
    "01J5OBJ0000000000000000042": {
      "position": [5.0, 0, 3.0],
      "physicsMode": "kinematic"
    }
  }
}, "0"]]
```

---

## 4. SceneAvatar lifecycle

### What the spec leaves open

The spec defines how a user enters a region (`SceneAvatar/set create`), the
one-avatar-per-region constraint, auto-eject from a previous region, avatar
state updates, leaving (setting `leftAt`), and reconnecting (section 5.4,
section 6.4). The spec notes that real-time position synchronization is handled
by the simulation layer, not JMAP. What the spec leaves open is how the server
detects silent disconnections, how long departed avatar records are retained,
and how the simulation layer is coordinated with the JMAP avatar lifecycle.

### What you must decide

- **Silent disconnection detection**: how the server knows a user has left
  without explicitly setting `leftAt`.
- **Avatar record retention**: how long records with non-null `leftAt` are
  kept before cleanup.
- **Reconnect window**: how long after departure a returning user gets their
  existing record back vs. a fresh one.
- **Position update frequency**: how often the server reconciles the
  simulation layer's avatar positions with SceneAvatar records.
- **`displayName` source**: where the display name comes from (user profile,
  ChatContact, client-supplied).

### Considerations

**One avatar per region, one region at a time.** The spec enforces two
constraints:

1. If a user already has an active avatar (`leftAt: null`) in the target
   region, the server returns the existing record in `updated` rather than
   creating a duplicate.
2. If a user has an active avatar in a *different* region, the server sets
   `leftAt` on that avatar before creating the new one.

Both must be enforced atomically. A race condition where two concurrent
`SceneAvatar/set create` calls for the same user in different regions could
leave the user with two active avatars if not serialized. Use a per-user
lock or serializable transaction for avatar creation.

**Silent disconnection.** Like VTC participants, avatars can vanish without
calling `SceneAvatar/set` to set `leftAt` (browser crash, network loss, app
kill). The server SHOULD detect this and set `leftAt` automatically. Two
approaches:

- *Simulation layer callback*: the simulation server notifies the JMAP server
  when a client disconnects from `simulationUri`. This is the most reliable
  signal.
- *Heartbeat/timeout*: the JMAP server expects periodic
  `SceneAvatar/set update` calls (e.g., every 60 seconds with a
  `customProperties` heartbeat). If no update arrives within the timeout
  window, the server sets `leftAt`. Simpler but adds JMAP traffic.

For regions with no simulation layer (`simulationUri: null`), the heartbeat
approach is the only option.

**Reconnect window.** The spec says that when a user who has left re-enters
the same region, the server updates the existing record rather than creating
a new one: clears `leftAt`, updates `joinedAt`, resets position to
`spawnPosition`. This preserves identity continuity. Define a reconnect
window: if the user returns within N minutes, update the existing record.
Beyond that, the existing record may have been cleaned up or the server may
create a fresh one. A 30-minute window is reasonable; align it with your
avatar record retention policy.

**Position reconciliation frequency.** The spec recommends updating
SceneAvatar positions from the simulation layer every 5-30 seconds. These
updates serve clients that are not connected to the simulation layer (e.g.,
a web dashboard showing who is in a region) and provide a stale-but-useful
fallback. Higher frequency means more accurate positions in JMAP queries but
more database writes. For most deployments, 15 seconds is a good balance.

**Entering a region — the full sequence:**

1. Client calls `SceneAvatar/set create` with `regionId`.
2. Server checks access policy (public/invite/space).
3. Server checks `maxAvatarsPerRegion` quota.
4. Server checks for existing active avatar in the same region (return
   existing) or different region (eject from previous).
5. Server creates (or reconnects) the SceneAvatar record with
   `spawnPosition`, `spawnOrientation`, `joinedAt`, `leftAt: null`.
6. Server increments `activeAvatarCount` on the SceneRegion.
7. Server returns the created record.
8. Client reads `simulationUri` from the region and connects to the
   simulation layer independently.

Steps 2-6 should be atomic.

**Leaving a region — the full sequence:**

1. Client calls `SceneAvatar/set update` with `leftAt` set to current time.
2. Server verifies the caller owns this avatar record.
3. Server sets `leftAt`.
4. Server decrements `activeAvatarCount` on the SceneRegion.
5. Client disconnects from the simulation layer independently.

### Common patterns

| Concern | Pattern |
|---|---|
| Silent disconnect detection | Simulation layer callback (preferred) or JMAP heartbeat timeout (fallback). |
| Reconnect window | 5-30 minutes; same record reused with cleared `leftAt`. |
| Position reconciliation | Every 10-30 seconds; simulation layer pushes batch updates. |
| Avatar record retention | Keep departed records for 24 hours for query/history; purge after. |

### Recommended starting point

Detect silent disconnections via simulation layer callbacks. Set a 30-minute
reconnect window. Reconcile avatar positions every 15 seconds. Retain
departed avatar records for 24 hours. Serialize avatar creation per user
to prevent race conditions. Source `displayName` from the user profile;
override from ChatContact when JMAP Chat is co-deployed.

Example — entering a region:

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "create": {
    "av0": {
      "regionId": "01J5ABC0000000000000000001",
      "visualRef": "blob-avatar-alice-001",
      "visualType": "model/gltf-binary",
      "customProperties": {
        "animation": "idle",
        "nametag": true
      }
    }
  }
}, "0"]]
```

Example response:

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "oldState": "state-010",
  "newState": "state-011",
  "created": {
    "av0": {
      "id": "user:alice@example.com",
      "regionId": "01J5ABC0000000000000000001",
      "userId": "user:alice@example.com",
      "displayName": "Alice Chen",
      "position": [0, 0, 10],
      "orientation": [0, 0, 0, 1],
      "joinedAt": "2026-06-06T10:05:00Z",
      "leftAt": null
    }
  },
  "notCreated": {}
}, "0"]]
```

Example — leaving a region:

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "update": {
    "user:alice@example.com": {
      "leftAt": "2026-06-06T11:30:00Z"
    }
  }
}, "0"]]
```

Example — auto-eject scenario (user moves from one region to another):

```json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "create": {
    "av1": {
      "regionId": "01J5ABC0000000000000000002"
    }
  }
}, "0"]]
```

If Alice was active in region `01J5ABC0000000000000000001`, the server
automatically sets `leftAt` on her avatar there before creating the new
avatar in region `01J5ABC0000000000000000002`. The response does not
surface the ejection explicitly; the client observes it via
`SceneAvatar/changes` on the previous region.

---

## 5. Coordinate system

### What the spec leaves open

The spec is prescriptive about the coordinate convention: right-handed, Y-up,
meters, quaternion orientation `[x, y, z, w]` (section 3). This matches
glTF 2.0. The spec requires servers to normalize quaternions to unit length
and permits servers to reject non-unit quaternions. What the spec leaves open
is the tolerance for quaternion normalization, how to handle coordinate
conversions at the simulation layer boundary, and how clients in different
rendering engines should adapt.

### What you must decide

- **Quaternion epsilon**: the maximum deviation from unit length that the
  server accepts before rejecting with `invalidArguments`.
- **Coordinate conversion responsibility**: whether the JMAP server, the
  simulation layer, or the client is responsible for converting between
  the spec's convention and engine-native conventions.
- **Scale semantics for negative values**: whether your deployment permits
  negative scale (mirroring) or restricts to positive values.

### Considerations

**Right-handed Y-up is glTF 2.0.** This convention was chosen because glTF
is the mandatory-to-implement asset format. If your simulation layer uses a
different convention (Unity is left-handed Y-up; Unreal is left-handed Z-up),
conversion happens at the boundary. The JMAP wire format is always
right-handed Y-up.

**Quaternion normalization.** Floating-point arithmetic means quaternions
will drift from unit length. The server must normalize before storage. An
epsilon of `1e-4` (0.0001) is reasonable for rejecting inputs that are clearly
not unit quaternions (e.g., `[0, 0, 0, 0]` or `[10, 10, 10, 10]`). Inputs
within the epsilon should be silently normalized. Inputs outside the epsilon
should be rejected with `invalidArguments`.

**Meters are absolute.** Position values are in meters. An object at
`[100, 0, 50]` is 100 meters from the origin along X. This has implications
for `withinRadius` queries: a radius of `20` means 20 meters. Clients that
display positions in other units (feet, centimeters) must convert locally.

**Quaternion component order: `[x, y, z, w]`.** This matches glTF 2.0 and
most game engines. Some math libraries use `[w, x, y, z]` order. Clients
using those libraries must reorder components at the serialization boundary.
Getting this wrong produces rotations that look correct for small angles but
are wildly wrong for large ones. Test with a 90-degree rotation around Y:
the correct value is `[0, 0.707, 0, 0.707]` in `[x, y, z, w]` order.

**Coordinate conversion cheat sheet:**

| Engine | Handedness | Up | Conversion from spec |
|---|---|---|---|
| glTF 2.0 | Right | Y | None (native match). |
| Three.js | Right | Y | None (native match). |
| Unity | Left | Y | Negate Z for position; negate Z and W for quaternion. |
| Unreal | Left | Z | Swap Y and Z for position; swap and negate for quaternion. |
| Godot | Right | Y | None (native match). |
| Babylon.js | Left | Y | Negate Z for position; negate Z and W for quaternion. |

Conversion is the client's or simulation adapter's responsibility. The JMAP
server stores and returns coordinates in the spec's convention unconditionally.

### Recommended starting point

Set quaternion epsilon to `1e-4`. Normalize all quaternions to unit length
on write. Reject quaternions with magnitude deviation greater than `1e-4`
with `invalidArguments`. Permit negative scale values (mirroring). Perform
no coordinate conversion on the server; document the convention clearly for
client implementers.

---

## 6. Spatial query filters

### What the spec leaves open

The spec defines two spatial filter properties — `withinRadius` and
`withinBounds` — and declares them mandatory-to-implement for both
`SceneObject/query` and `SceneAvatar/query` (section 6.3.5, section 6.4.5).
The spec does not prescribe the spatial indexing algorithm, how to handle
large result sets, or whether the server should support additional spatial
filters beyond the two required ones.

### What you must decide

- **Spatial indexing strategy**: how to make `withinRadius` and
  `withinBounds` queries fast as object counts grow.
- **Interaction with visibility filtering**: whether spatial queries apply
  before or after the visibility contract (section 7.3).
- **Result set limits**: how to handle queries that match thousands of
  objects.
- **Additional spatial filters**: whether to support deployment-specific
  filters beyond the two required ones.

### Considerations

**`withinRadius` and `withinBounds` are mandatory.** Servers MUST support
both. Returning `unsupportedFilter` for either is a spec violation. These
are the primary mechanism clients use to load objects near the avatar and
implement level-of-detail loading.

**`withinRadius` semantics.** The filter matches objects whose *position*
(a single point) falls within the sphere. It does not consider the object's
bounding box or visual extent. A large object whose origin is outside the
radius will not match, even if part of its visual representation is inside.
This is a deliberate simplification; bounding-box intersection is expensive
and the point-in-sphere test is sufficient for interest management.

**`withinBounds` semantics.** Same principle: the filter matches objects whose
position falls within the axis-aligned bounding box. The AABB is defined by
`min` and `max` corners. Point-in-AABB is a cheap test.

**Combining spatial and non-spatial filters.** All filter properties are
combined with AND. A query with `withinRadius`, `interactable: true`, and
`visualType: "model/gltf-binary"` returns only objects that satisfy all three
conditions. Apply spatial filters first (they are typically the most
selective) and then apply non-spatial filters on the spatial result set.

**Spatial indexing.** For small regions (under ~1000 objects), a linear scan
is acceptable. For larger regions, use a spatial index:

| Strategy | Trade-offs |
|---|---|
| R-tree | Good for AABB queries; well-supported in most databases (PostGIS, SQLite R*tree module). |
| Octree/Quadtree | Good for radius queries; in-memory; needs rebuilding on object moves. |
| Grid (spatial hash) | Simple; O(1) lookup for bounded regions; wastes memory for sparse regions. |
| Database-native spatial | PostgreSQL `earth_distance` or `PostGIS`; handles both radius and AABB natively. |

If your objects are stored in PostgreSQL, use PostGIS or the built-in
`cube`/`earthdistance` extensions. If objects are in-memory, an R-tree or
grid provides sub-millisecond queries.

**Result set limits.** JMAP `/query` supports `position` and `limit` for
pagination. A `withinRadius` query with a 1000-meter radius in a dense region
could return thousands of objects. Enforce the `limit` parameter (default to
a sane value like 100 if the client does not set one) and let clients paginate.
Do not silently truncate without honoring the JMAP pagination contract.

**Visibility filtering interaction.** The visibility contract (spec section
7.3) says the server SHOULD apply visibility filtering. If your deployment
implements visibility filtering, apply it as an additional filter on query
results. The order is: access control check (MUST), then spatial filter,
then visibility filter (SHOULD), then non-spatial filters.

Example — query objects within 20 meters of the avatar:

```json
[["SceneObject/query", {
  "accountId": "account-xyz",
  "filter": {
    "regionId": "01J5ABC0000000000000000001",
    "withinRadius": {
      "center": [5.0, 1.5, -2.0],
      "radius": 20
    }
  },
  "sort": [{"property": "name", "isAscending": true}],
  "limit": 50
}, "0"]]
```

Example — query objects within a bounding box (useful for loading a specific
area of a large region):

```json
[["SceneObject/query", {
  "accountId": "account-xyz",
  "filter": {
    "regionId": "01J5ABC0000000000000000001",
    "withinBounds": {
      "min": [-10, 0, -10],
      "max": [10, 5, 10]
    },
    "visible": true
  },
  "limit": 100
}, "0"]]
```

Example — query avatars near a point (for proximity features):

```json
[["SceneAvatar/query", {
  "accountId": "account-xyz",
  "filter": {
    "regionId": "01J5ABC0000000000000000001",
    "isActive": true,
    "withinRadius": {
      "center": [5.0, 0, -2.0],
      "radius": 10
    }
  }
}, "0"]]
```

### Recommended starting point

Use your database's spatial indexing if available (PostGIS for PostgreSQL,
R*tree for SQLite). If objects are in-memory, use a grid/spatial hash for
regions with known bounds. Default query `limit` to 100. Apply filters in
order: access control, spatial, visibility, non-spatial. Do not implement
additional spatial filters beyond `withinRadius` and `withinBounds` in the
first version.

---

## 7. SceneAsset management

### What the spec leaves open

The spec defines SceneAsset as metadata about a JMAP blob: `blobId`,
`mediaType`, `name`, `size`, `sha256`, and an optional `assetUri` CDN
endpoint (section 5.1). Assets are uploaded via the standard JMAP upload
mechanism and then registered via `SceneAsset/set create`. The spec does
not prescribe how assets are validated, how CDN distribution works, or how
asset deduplication is handled.

### What you must decide

- **Asset validation on upload**: whether the server validates that the
  uploaded blob matches the declared `mediaType` (e.g., actually parses the
  glTF header) or trusts the client.
- **CDN integration**: how `assetUri` is populated — server-generated from
  the blob store, client-supplied, or absent.
- **Asset deduplication**: whether two SceneAsset objects pointing to the
  same `blobId` are permitted, and whether identical uploads are deduplicated
  at the blob level.
- **Asset lifecycle**: whether SceneAsset records are cleaned up when no
  SceneObject references them.

### Considerations

**`model/gltf-binary` is mandatory to accept.** The server MUST accept assets
with `mediaType: "model/gltf-binary"`. The spec does not require the server
to parse or validate the file contents, but doing so is strongly recommended.
A malformed glTF file that passes through the server will crash or exploit
clients. At minimum, validate the glTF magic number (`0x46546C67`) and version
field. Full validation (schema conformance, buffer bounds, texture references)
is better but more expensive.

**`assetUri` is a CDN optimization.** When present, clients SHOULD prefer
`assetUri` over the JMAP blob download path. The server can populate
`assetUri` automatically by mapping `blobId` to a CDN URL. If your blob
store is backed by S3 or a similar service, generate a CDN URL at asset
creation time and store it. If the client supplies `assetUri`, treat it as
untrusted (spec section 8.2): clients fetching it must validate the scheme
and should verify `sha256` when available.

**`sha256` for integrity.** The spec defines `sha256` as optional but clients
MAY use it for cache validation. If the server computes the hash at upload
time (which it should, since it has the blob), always populate this field.
Clients that implement hash verification get cache-safe deduplication and
tamper detection for free.

**Storage quota.** The `maxAssetSizeBytes` limit applies per asset file. The
server must check this at `SceneAsset/set create` time by reading the blob's
size. Return `overQuota` if the blob exceeds the limit. Note that the limit
is per-asset, not per-account total; for total storage limits, use a
deployment-defined quota on the blob store.

### Common patterns

| Concern | Pattern |
|---|---|
| Asset validation | Validate magic number and header on upload; full parse on a background job. |
| CDN | S3 + CloudFront; server generates `assetUri` at creation time. |
| Deduplication | Allow multiple SceneAsset records per `blobId`; deduplicate at the blob layer. |
| Cleanup | Garbage-collect SceneAsset records with no referencing SceneObject after 30 days. |

### Recommended starting point

Validate glTF magic number at `SceneAsset/set create` time. Compute and
store `sha256` at upload time. Generate `assetUri` from your CDN
configuration if available; leave `null` otherwise. Allow multiple
SceneAsset records per `blobId`. Implement a background job that identifies
unreferenced assets older than 30 days and deletes them.

Example — uploading and registering an asset:

```json
[["SceneAsset/set", {
  "accountId": "account-xyz",
  "create": {
    "a0": {
      "blobId": "Gc0f032d390a5d5fa8a35",
      "mediaType": "model/gltf-binary",
      "name": "Office Chair"
    }
  }
}, "0"]]
```

Example response:

```json
[["SceneAsset/set", {
  "accountId": "account-xyz",
  "oldState": "state-020",
  "newState": "state-021",
  "created": {
    "a0": {
      "id": "01J5AST0000000000000000007",
      "accountId": "account-xyz",
      "size": 2457600,
      "sha256": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4e5f6a1b2",
      "assetUri": "https://cdn.example.com/assets/01J5AST0000000000000000007.glb",
      "createdAt": "2026-06-06T09:00:00Z"
    }
  },
  "notCreated": {}
}, "0"]]
```

---

## 8. Access control patterns

### What the spec leaves open

The spec defines region-level access control via `accessPolicy` (section 9),
object-level permissions via `ownerId` / region owner / admin (section 9.2),
and avatar permissions (users modify only their own avatar, section 9.3). It
requires returning `notFound` rather than `forbidden` for unauthorized
get/query requests to prevent enumeration. It permits region owners and
admins to eject avatars. What the spec leaves open is how administrative
privileges are defined, how the invitation mechanism works for `"invite"`
regions, and how to handle cross-region visibility.

### What you must decide

- **Administrative privilege model**: who counts as an "admin" for object
  and avatar permissions.
- **Invitation management for `"invite"` regions**: how invitations are
  created, accepted, and revoked.
- **`notFound` vs. `forbidden` consistency**: ensuring that every code path
  that checks access returns `notFound` for unauthorized users, never
  `forbidden` (to prevent enumeration).
- **Ejection policy**: under what circumstances admins and region owners
  may eject avatars, and whether ejected users can re-enter immediately.

### Considerations

**`notFound` masking is a security requirement.** The spec is explicit: for
`SceneRegion/get`, `SceneObject/get`, and `SceneAvatar/get`, unauthorized
users receive `notFound`, not `forbidden`. This prevents a user from
discovering whether a region exists by observing the error type. Apply this
consistently across all code paths, including query results (unauthorized
regions simply do not appear in results).

The exception is `SceneAvatar/set create` (entering a region): the spec
returns `notFound` when the region does not exist or the caller does not have
access, and `forbidden` when the access policy specifically rejects the user
(e.g., `"invite"` without an invitation). This is a deliberate distinction:
the user already knows the region exists (they have the regionId) and the
error helps them understand why entry failed.

**Object permissions are a three-tier model:**

1. **Owner** (`ownerId`): the user who created the object. Can always update
   or destroy it.
2. **Region owner**: the user whose account owns the region. Can update or
   destroy any object in their region.
3. **Admin**: deployment-defined. Can update or destroy any object.

For a simple deployment, "admin" can be a boolean flag on the account. For
a complex deployment, consider role-based access control with region-specific
roles (e.g., "editor" for a specific region, "global admin" for all regions).

**Avatar ejection.** The spec says region owners and admins may eject avatars
by setting `leftAt` on another user's SceneAvatar. Implement a cooldown
after ejection: if a user is ejected, prevent them from re-entering the same
region for a deployment-defined period (RECOMMENDED: 5 minutes). Without a
cooldown, an ejected user can immediately re-enter, making ejection useless.
The spec does not define this cooldown; it is a deployment decision.

**Cross-account visibility.** In a multi-account deployment, a user in
account A should not see regions, objects, or avatars in account B unless
explicitly shared. The JMAP account model handles this naturally (data types
are scoped to accounts), but verify that your spatial query implementation
does not leak cross-account data through shared spatial indexes.

### Recommended starting point

Define "admin" as a boolean flag on the account capability or user profile.
For `"invite"` regions, maintain a server-side ACL (Set of userIds) per
region. Implement a 5-minute re-entry cooldown after ejection. Audit every
get/query code path to ensure `notFound` masking is applied. For
`SceneObject/set`, check permissions in order: is caller the owner? Is
caller the region owner? Is caller an admin? If none, return `forbidden`.

---

## 9. Quota enforcement

### What the spec leaves open

The spec defines four quota fields in the account-level capability:
`maxRegionsPerAccount`, `maxObjectsPerRegion`, `maxAvatarsPerRegion`, and
`maxAssetSizeBytes` (section 4.2). It requires `overQuota` errors when
limits are exceeded. What the spec leaves open is when and how to check
these limits, how to handle near-quota warnings, and whether rate limits
(as distinct from capacity limits) are needed.

### What you must decide

- **Check timing**: whether to check quotas at the start of the set
  operation (before any creates) or per-create within a batch.
- **Near-quota signals**: whether to warn clients approaching a limit.
- **Rate limits**: whether to impose per-user creation rate limits beyond
  the capacity quotas.
- **`null` (unlimited) handling**: how to handle accounts with no
  server-imposed limit — whether there is a hard ceiling somewhere else
  (database, memory).

### Considerations

**Per-create vs. batch checking.** A `SceneObject/set` call with 10 creates
might push the region from 495 to 505 objects when the limit is 500. Two
strategies:

- *All-or-nothing*: reject the entire batch if it would exceed the quota.
  Simpler but frustrating for clients sending large batches.
- *Per-create*: process creates sequentially; return `overQuota` for
  individual creates that would exceed the limit; succeed the rest. More
  granular and consistent with the JMAP set model (per-id errors in
  `notCreated`).

The per-create approach matches standard JMAP semantics better.

**Rate limits.** Capacity quotas prevent one user from filling a region with
too many objects, but they do not prevent rapid create-delete churn. A user
could create 500 objects, delete them, create 500 more, etc. If this creates
a performance problem (e.g., generating excessive `StateChange` notifications),
add a per-user rate limit on `SceneObject/set create` (RECOMMENDED: 60 creates
per minute per region).

**Interaction event rate limits.** When the Scene WebSocket capability is
deployed, the server SHOULD enforce a rate limit of 5 `SceneInteractionEvent`
events per user per region per second, and 2 `SceneObjectEvent` updates per
object per second. These limits are defined in {{JMAP-SCENE-WSS}} and apply
to outbound event delivery. Events that exceed the limit are silently dropped;
the most recent state is delivered when the rate window reopens.

**`null` means no server-imposed limit, not infinite.** Even when quota fields
are `null`, physical limits exist (database size, memory, file descriptors).
Document your deployment's practical limits even when the JMAP quota is
`null`.

**Object density as DoS.** The spec explicitly calls out object density as a
denial-of-service vector (section 8.6). `maxObjectsPerRegion` is the primary
defense. Enforce it strictly.

### Recommended starting point

Check quotas per-create within a batch; return `overQuota` in `notCreated`
for individual creates that exceed the limit. Enforce a rate limit of 60
object creates per minute per user per region. When quota fields are `null`,
enforce a deployment-level hard ceiling (e.g., 10,000 objects per region) to
prevent unbounded growth. Log near-quota events (>90% utilization) for
operator alerting.

---

## 10. Error conditions

### What the spec leaves open

The spec references standard JMAP SetError types (`invalidArguments`,
`forbidden`, `notFound`, `overQuota`, `stateMismatch`) throughout the method
definitions. What the spec leaves open is how to structure error descriptions,
whether to include additional deployment-defined error types, and how to
handle compound errors (e.g., a batch operation where some items succeed and
some fail).

### What you must decide

- **Error description detail**: how much information to include in the
  `description` field of SetError objects.
- **Deployment-defined error types**: whether to introduce additional error
  types beyond the spec-defined ones.
- **Compound error handling**: how to report partial failures in batch
  operations.

### Considerations

**Common error conditions and their types:**

| Condition | SetError type | When it occurs |
|---|---|---|
| Missing required field | `invalidArguments` | Create without `regionId`, `position`, etc. |
| `visualRef` without `visualType` | `invalidArguments` | Create or update with mismatched pair. |
| `visualType` not in `supportedVisualTypes` | `invalidArguments` | Create with unsupported media type. |
| `bounds.min` >= `bounds.max` on any axis | `invalidArguments` | Region create with degenerate bounds. |
| Quaternion magnitude outside epsilon | `invalidArguments` | Object or avatar create/update. |
| `parentId` in different region | `invalidArguments` | Object create with cross-region parent. |
| Position outside region bounds | `invalidArguments` | Object create/update (if rejecting, not clamping). |
| `leftAt` already set on avatar | `invalidArguments` | Attempt to leave when already left. |
| Region does not exist or no access | `notFound` | Object create referencing inaccessible region. |
| Object/avatar in inaccessible region | `notFound` | Get/query for unauthorized data. |
| User not object owner, region owner, or admin | `forbidden` | Object update/destroy by unauthorized user. |
| Modifying another user's avatar | `forbidden` | Avatar update with mismatched userId. |
| `mayCreateRegion` is `false` | `forbidden` | Region create by unauthorized user. |
| `mayCreateObject` is `false` | `forbidden` | Object create by unauthorized user. |
| Access policy denies entry | `forbidden` | Avatar create in invite-only region without invitation. |
| `maxRegionsPerAccount` exceeded | `overQuota` | Region create. |
| `maxObjectsPerRegion` exceeded | `overQuota` | Object create. |
| `maxAvatarsPerRegion` exceeded | `overQuota` | Avatar create (entering a full region). |
| `maxAssetSizeBytes` exceeded | `overQuota` | Asset create with oversized blob. |
| Concurrent modification | `stateMismatch` | Set with stale `ifInState`. |

**Error descriptions should be actionable.** Include the specific field name
and the constraint that was violated. For example:

```json
{
  "type": "invalidArguments",
  "description": "position[0] is 600.0, which is outside region bounds [-500, 500] on the X axis"
}
```

Not:

```json
{
  "type": "invalidArguments",
  "description": "Invalid position"
}
```

**Compound errors follow JMAP semantics.** In a batch `SceneObject/set` with
multiple creates, each create succeeds or fails independently. Successful
creates appear in `created`; failed creates appear in `notCreated` with
per-id SetError objects. The overall response always returns HTTP 200; errors
are in the JMAP response body.

---

## 11. The simulationUri pattern

### What the spec leaves open

The spec defines `simulationUri` as an opaque, deployment-supplied URI on
SceneRegion that points to the real-time simulation layer (section 5.3,
section 7). The spec explicitly states that the JMAP server is a spatial
state database, not a rendering engine, and that the simulation protocol is
deployment-defined. It calls out security requirements: clients MUST NOT
auto-connect, and servers MUST NOT probe the URI.

### What you must decide

- **URI scheme and format**: what your `simulationUri` looks like.
- **Authentication**: how the simulation layer authenticates users arriving
  from the JMAP client.
- **State reconciliation**: how frequently and by what mechanism the
  simulation layer writes back to the JMAP state.
- **Failure handling**: what happens when the simulation layer is unreachable.

### Considerations

**`simulationUri` is the bridge between JMAP and real-time.** JMAP Scene
manages state at rest: what exists and where. The simulation layer manages
state in motion: real-time avatar positions, physics, interactions. The two
must be loosely coupled. The `simulationUri` is the only point of contact
visible in the JMAP data model.

**URI format depends on your simulation stack.**

| Simulation stack | Typical `simulationUri` |
|---|---|
| WebSocket-based (custom, Colyseus) | `wss://sim.example.com/regions/{regionId}` |
| WebRTC data channel | `https://sim.example.com/signaling/{regionId}` |
| LiveKit (spatial) | `wss://livekit.example.com/room/{regionId}` |
| UDP/QUIC (custom game server) | `udp://sim.example.com:7777?region={regionId}` |
| None (static scene) | `null` |

**Authentication at the simulation layer.** The JMAP client authenticates
to the JMAP server via standard JMAP auth. When the client connects to
`simulationUri`, the simulation layer needs to verify the user's identity.
Common patterns:

- *Token relay*: the JMAP server issues a short-lived token (JWT or opaque)
  for the simulation layer. The client passes this token when connecting to
  `simulationUri`. The simulation layer validates the token against the JMAP
  server or a shared secret.
- *Session cookie*: the simulation layer shares an auth domain with the JMAP
  server and validates the same session.
- *No auth*: the simulation layer is open. Suitable only for public,
  non-sensitive scenes. Not recommended for production.

Token relay is the safest and most portable approach.

**State reconciliation.** The simulation layer holds authoritative real-time
state (avatar positions updated 10-60 Hz). The JMAP server holds persistent
state (positions updated every 5-30 seconds). Reconciliation flows one
direction: simulation layer to JMAP. The simulation layer periodically
writes avatar positions and dynamic object positions back to the JMAP
server via internal API calls (not via JMAP method calls from the client).

If the simulation layer crashes and restarts, it should bootstrap its state
from the JMAP server: load all SceneObject and active SceneAvatar records
for the region.

**Failure handling.** When the simulation layer behind `simulationUri` is
unreachable:

- Clients should display the scene in a read-only or static mode (objects
  visible, no real-time interaction).
- The JMAP server continues to function normally: clients can query objects,
  read avatar records, and even create/update objects. The scene is
  "frozen" — no real-time position updates, no physics, no interactions —
  but the state database is intact.
- Clients SHOULD inform the user that real-time features are unavailable.

**Security recap.** Two non-negotiable rules from the spec:

1. **Clients MUST NOT auto-connect to `simulationUri`.** The user must
   initiate the connection (e.g., by clicking "Enter Region"). Auto-connecting
   on region load exposes the client to malicious simulation servers.
2. **Servers MUST NOT fetch or probe `simulationUri`.** This prevents SSRF.
   The server stores the URI as an opaque string and never dereferences it.

### Recommended starting point

Use a WebSocket-based simulation protocol with `simulationUri` formatted as
`wss://sim.example.com/regions/{regionId}`. Issue a short-lived JWT at
`SceneAvatar/set create` time (returned alongside or after the avatar
creation response, via a deployment-defined mechanism) for the client to
present when connecting to the simulation layer. Reconcile positions from the
simulation layer to JMAP every 15 seconds via an internal batch update API.
When the simulation layer is unavailable, render the scene in static mode
from JMAP data.

A more detailed treatment of the simulation layer integration — including
protocol design, interest management, physics synchronization, and scaling —
is covered in a separate simulation integration guide.

---

## Appendix: Decision checklist

Before deploying JMAP Scene to production, verify that your implementation has
made and documented each of the following decisions:

**Capability advertisement (section 1)**
- [ ] `supportedVisualTypes` populated (at least `"model/gltf-binary"`)
- [ ] Quota values configured (`maxRegionsPerAccount`, `maxObjectsPerRegion`,
  `maxAvatarsPerRegion`, `maxAssetSizeBytes`)
- [ ] `mayCreateRegion` and `mayCreateObject` permission logic implemented

**SceneRegion lifecycle (section 2)**
- [ ] `"invite"` access policy: invitation mechanism implemented
- [ ] `"space"` access policy: Space membership resolution implemented (if
  JMAP Chat is co-deployed)
- [ ] `simulationUri` provisioning strategy chosen
- [ ] `environment` schema documented
- [ ] Region destruction cascade implemented (objects, avatars, simulation
  layer notification)

**SceneObject CRUD (section 3)**
- [ ] Position bounds validation strategy chosen (clamp vs. reject)
- [ ] `customProperties` size limit enforced
- [ ] Scene graph depth limit enforced
- [ ] `visualRef`/`visualType` coupling validated on create and update
- [ ] Simulation layer synchronization mechanism implemented

**SceneAvatar lifecycle (section 4)**
- [ ] One-avatar-per-region constraint enforced atomically
- [ ] Auto-eject from previous region on enter implemented
- [ ] Silent disconnection detection implemented (simulation callback or
  heartbeat)
- [ ] Reconnect window defined
- [ ] `activeAvatarCount` incremented/decremented atomically

**Coordinate system (section 5)**
- [ ] Quaternion normalization implemented on write
- [ ] Quaternion epsilon defined and documented
- [ ] Coordinate convention documented for client implementers
- [ ] 90-degree rotation test vector verified: `[0, 0.707, 0, 0.707]`

**Spatial query filters (section 6)**
- [ ] `withinRadius` implemented (mandatory)
- [ ] `withinBounds` implemented (mandatory)
- [ ] Spatial indexing strategy chosen for expected object counts
- [ ] Query result pagination working with `limit` and `position`

**SceneAsset management (section 7)**
- [ ] `model/gltf-binary` accepted
- [ ] Asset validation (at least magic number check) implemented
- [ ] `sha256` computed and stored at upload time
- [ ] `assetUri` generation strategy chosen
- [ ] `maxAssetSizeBytes` enforced at create time

**Access control (section 8)**
- [ ] `notFound` returned (not `forbidden`) for unauthorized get/query
- [ ] Object permission checks: owner, region owner, admin
- [ ] Avatar permission checks: users modify only their own
- [ ] Admin privilege model defined
- [ ] Ejection cooldown implemented

**Quota enforcement (section 9)**
- [ ] Per-create quota checking in batch operations
- [ ] Rate limits on object creation implemented
- [ ] `null` quota fields backed by a deployment-level hard ceiling

**Error conditions (section 10)**
- [ ] All SetError types from the error table implemented
- [ ] Error descriptions include specific field names and constraints
- [ ] Compound errors reported per-id in `notCreated`/`notUpdated`/`notDestroyed`

**simulationUri (section 11)**
- [ ] URI scheme and format documented
- [ ] Simulation layer authentication mechanism implemented
- [ ] State reconciliation frequency configured
- [ ] Failure mode (static scene fallback) implemented
- [ ] Client auto-connect prevention verified
- [ ] Server SSRF prevention verified (no probing of `simulationUri`)
