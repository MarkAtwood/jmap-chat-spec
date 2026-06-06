---
title: JMAP for Scenes
abbrev: JMAP Scene
docname: draft-atwood-jmap-scene-00
category: std
stream: ietf

ipr: trust200902

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
  -
    fullname: Mark Atwood
    email: mark@reviewcommit.com

normative:
  RFC2119:
  RFC8174:
  RFC8620:
  ULID:
    title: Universally Unique Lexicographically Sortable Identifier
    target: https://github.com/ulid/spec

informative:
  RFC6455:
  RFC8887:
  GLTF:
    title: "glTF 2.0 Specification"
    target: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html
    author:
      org: Khronos Group
    date: 2021
  JMAP-CHAT:
    title: JMAP for Chat
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-00
    date: 2026
  JMAP-VTC:
    title: JMAP for Video/Voice Teleconferencing
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-vtc-00
    date: 2026
  JMAP-CHAT-WSS:
    title: JMAP Chat over WebSocket
    author:
      fullname: Mark Atwood
    seriesinfo:
      Internet-Draft: draft-atwood-jmap-chat-wss-00
    date: 2026

--- abstract

This document defines JMAP Scene, a JMAP capability ({{RFC8620}}) for managing three-dimensional spatial environments. It defines the `urn:ietf:params:jmap:scene` capability; four data types (SceneRegion, SceneObject, SceneAvatar, and SceneAsset); standard JMAP methods for managing spatial content including spatial query filters; a coordinate system convention; and a visibility contract between server and client.

The specification models spatial *state* — what objects exist, where they are, and who is present — without prescribing the rendering technology, simulation protocol, or asset format. Actual real-time position synchronization, physics, and rendering travel through a deployment-chosen simulation stack; the JMAP server is a spatial state database, not a rendering engine.

The capability is standalone (`urn:ietf:params:jmap:scene`). When JMAP Chat ({{JMAP-CHAT}}) is also present, SceneRegion objects MAY carry back-references to Chat and Space objects for in-world text communication. When JMAP VTC ({{JMAP-VTC}}) is also present, SceneRegion objects MAY bind to VTCCall objects for in-world voice and video.

--- middle

# Introduction

Three-dimensional spatial environments — virtual worlds, collaborative design spaces, immersive galleries, architectural walkthroughs, data visualizations — are built by many systems, each with its own state model, wire protocol, and asset pipeline. There is no standard API for the spatial state layer: what objects exist in a scene, where they are positioned, who is present, and who may modify what.

This document defines a JMAP capability that models spatial state as JMAP data types with standard get/set/changes/query methods. The server tracks what regions exist, what objects are in them, and which users are present — but the server does not render frames, simulate physics, or negotiate media codecs. A `simulationUri` field on every SceneRegion points to the deployment's real-time simulation layer; JMAP Scene has no opinion on what lives behind that URI.

## Design Philosophy

- **Spatial state, not spatial rendering.** The JMAP server is a spatial state database. It knows a region exists, what objects are in it, and where they are. It does not rasterize triangles, compute lighting, or run physics. Rendering and simulation are the simulation stack's job.
- **Format-agnostic.** Visual representations are referenced by media type. The spec requires a mandatory-to-implement baseline format but does not prescribe triangles, voxels, gaussian splats, or any specific representation. As visual formats evolve, the spec does not need amendment.
- **Coordinate system is normative.** Position, orientation, and scale use a single convention: right-handed, Y-up, meters, quaternions. This is the one place the spec is unapologetically opinionated, because interoperability requires it.
- **Standalone with optional bindings.** The capability stands alone. Chat, VTC, and file-system integrations are optional foreign keys, not dependencies.

## Relationship to JMAP Chat (optional) {#chat-binding}

When `urn:ietf:params:jmap:chat` ({{JMAP-CHAT}}) is advertised alongside `urn:ietf:params:jmap:scene`, SceneRegion objects MAY carry `chatId`, `spaceId`, and `channelId` back-references. When present:

- **In-region text chat** is the Chat bound via `chatId`. Messages sent to that Chat appear as in-world text communication.
- **Proximity chat** may be modeled as a Chat per region or as ephemeral events on the Chat's WebSocket connection ({{JMAP-CHAT-WSS}}).
- **Access control** may be inherited from the Space/channel permission model.

Without JMAP Chat, the Scene capability is fully functional as a standalone spatial state manager. Regions have no chat binding, and the text communication features above are unavailable. Clients SHOULD inform the user when a region has multiple avatars but no communication capability is available.

## Relationship to JMAP VTC (optional) {#vtc-binding}

When `urn:ietf:params:jmap:vtc` ({{JMAP-VTC}}) is advertised alongside `urn:ietf:params:jmap:scene`, SceneRegion objects MAY carry an `activeCallId` binding to a VTCCall. When present:

- **In-region voice/video** is the VTCCall bound via `activeCallId`. Joining the region and joining the call are independent actions; the binding provides the association.
- **Spatial audio** is a simulation-layer concern. The VTCCall provides the voice channel; the simulation layer handles spatialization based on avatar positions.

Without JMAP VTC, the Scene capability is fully functional. Regions have no voice binding.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

Terminology from {{RFC8620}} is used throughout.

# Coordinate System {#coordinate-system}

All spatial data in this specification uses a single coordinate convention. Implementations MUST use this convention for all position, orientation, and scale values in JMAP method calls and responses. Conversion to and from other conventions is the client's or simulation layer's responsibility.

## Position

Position is a JSON array of three Numbers: `[x, y, z]`.

- **Handedness:** Right-handed.
- **Up axis:** Y (positive Y points up).
- **Units:** Meters.
- **Origin:** Region-local. Each SceneRegion has its own origin at `[0, 0, 0]`. The region's `bounds` defines the extent of valid coordinates.

This convention matches glTF 2.0 ({{GLTF}}).

## Orientation

Orientation is a JSON array of four Numbers: `[x, y, z, w]`, representing a unit quaternion.

The identity orientation (no rotation) is `[0, 0, 0, 1]`.

Quaternions are used because they are the minimal non-degenerate parameterization of 3D rotation, avoid gimbal lock, interpolate smoothly (slerp), and match glTF 2.0 and all major physics engines. The spec does not use Euler angles or axis-angle representations on the wire.

Servers MUST normalize quaternions to unit length before storage. Clients SHOULD send unit quaternions; servers MAY reject quaternions whose magnitude deviates from 1.0 by more than a deployment-defined epsilon with `invalidArguments`.

## Scale

Scale is a JSON array of three Numbers: `[sx, sy, sz]`, representing scale factors along each axis. `[1, 1, 1]` is the identity (unscaled). Negative values are permitted and indicate mirroring along that axis.

# Capability {#capability}

Servers supporting this specification MUST advertise the `urn:ietf:params:jmap:scene` capability in the JMAP Session object.

## Session-Level Capability Object

The value of `capabilities["urn:ietf:params:jmap:scene"]` at the session level is an empty JSON object `{}`.

## Account-Level Capability Object

The value of `accountCapabilities["urn:ietf:params:jmap:scene"]` is a JSON object with the following fields:

`mayCreateRegion` (Boolean):
: `true` if the authenticated user may create new SceneRegion objects.

`mayCreateObject` (Boolean):
: `true` if the authenticated user may create new SceneObject objects.

`supportedVisualTypes` (String[]):
: Media types the deployment supports for visual representations. Servers MUST include at least `"model/gltf-binary"`.

`maxRegionsPerAccount` (UnsignedInt|null):
: Maximum regions per account. `null` means no server-imposed limit.

`maxObjectsPerRegion` (UnsignedInt|null):
: Maximum objects per region. `null` means no limit.

`maxAvatarsPerRegion` (UnsignedInt|null):
: Maximum concurrent avatars per region. `null` means no limit.

`maxAssetSizeBytes` (UnsignedInt|null):
: Maximum asset file size in bytes. `null` means no limit.

# Data Types

Data types are defined in dependency order: embedded sub-types and referenced types precede the types that reference them.

## SceneBounds {#scene-bounds}

SceneBounds is an embedded object describing an axis-aligned bounding box. It is not a top-level JMAP data type.

`min` (Number[3]):
: The minimum corner of the bounding box: `[x, y, z]` in region-local coordinates.

`max` (Number[3]):
: The maximum corner of the bounding box: `[x, y, z]` in region-local coordinates. Each component MUST be greater than or equal to the corresponding component of `min`.

Example:

~~~json
{
  "min": [-500, -10, -500],
  "max": [500, 200, 500]
}
~~~

This describes a 1000m x 210m x 1000m region.

## SceneAsset {#scene-asset}

A SceneAsset is metadata about a visual or audio resource stored as a JMAP blob. It is a top-level JMAP data type.

SceneAsset provides queryable metadata for assets referenced by SceneObject and SceneAvatar. Deployments that do not need asset management beyond raw blob references MAY omit support for SceneAsset methods; in that case, SceneObject and SceneAvatar fields reference blobIds directly.

`id` (String, immutable, server-set):
: A ULID {{ULID}} assigned at creation.

`accountId` (String, server-set):
: The account that owns this asset.

`blobId` (String, immutable):
: The JMAP blob reference for the asset file. The blob is uploaded via the standard JMAP upload mechanism ({{RFC8620}} Section 6.1).

`assetUri` (String|null):
: An optional deployment-provided URI for retrieving the asset, typically a CDN endpoint. When present, clients SHOULD prefer `assetUri` over the JMAP blob download path for performance. Peer-supplied; MUST be treated as untrusted (see {{asset-uri-untrusted}}).

`mediaType` (String):
: The media type of the asset (e.g., `"model/gltf-binary"`, `"image/png"`, `"audio/mpeg"`).

`name` (String|null):
: Human-readable label for the asset.

`size` (UnsignedInt, server-set):
: File size in bytes.

`sha256` (String|null):
: SHA-256 hash of the asset file, lowercase hexadecimal. Clients MAY use this for cache validation.

`createdAt` (UTCDate, immutable, server-set):
: Time the asset was created.

## SceneRegion {#scene-region}

A SceneRegion represents a discrete bounded spatial environment that users can enter and explore. It is the primary container for SceneObject and SceneAvatar records. It is a top-level JMAP data type.

### SceneRegion Object Fields

`id` (String, immutable, server-set):
: A ULID assigned at creation.

`accountId` (String, server-set):
: The account that owns this region.

`name` (String):
: Human-readable name for the region (e.g., `"Gallery East Wing"`, `"Main Plaza"`).

`description` (String|null):
: Optional description of the region.

`bounds` (SceneBounds):
: The spatial extent of the region. Objects and avatars within this region SHOULD have positions inside these bounds. Servers MAY clamp positions that fall outside bounds or MAY reject them with `invalidArguments`.

`environment` (Object|null):
: Deployment-defined environment settings for the region (e.g., sky, lighting, gravity, fog). Opaque to the JMAP Scene specification; clients that understand the deployment's environment schema MAY use it. Clients MUST ignore unrecognized keys.

`simulationUri` (String|null):
: The real-time simulation layer endpoint for this region. Opaque to the JMAP server. May be a WebSocket URL, a WebRTC signaling endpoint, a custom UDP endpoint, or any other simulation entry point. Peer-supplied; MUST be treated as untrusted (see {{simulation-uri-untrusted}}). `null` when the region has no real-time simulation layer (static scene, offline viewing).

`spawnPosition` (Number[3]):
: Default position where new avatars appear when entering the region. Default: `[0, 0, 0]`.

`spawnOrientation` (Number[4]):
: Default orientation for new avatars. Default: `[0, 0, 0, 1]` (identity).

`activeAvatarCount` (UnsignedInt, server-set):
: Number of avatars currently present in the region.

`viewHint` (String|null):
: Advisory rendering mode for clients. Values defined by this specification: `"3d"` (standard 3D perspective), `"2d-topdown"` (top-down 2D view, as in virtual offices or board games), `"2d-side"` (side-scrolling 2D view, as in platformer games). `null` is treated as `"3d"`. This field is advisory; clients MAY ignore it and render the region in any mode they support. Clients MUST NOT fail when encountering an unrecognized value; they SHOULD fall back to `"3d"`. Deployment-specific view hints SHOULD use reverse-domain notation (e.g., `"com.example.isometric"`).

`accessPolicy` (String):
: Who may enter this region. Values: `"public"` (any authenticated user), `"invite"` (only users explicitly granted access), `"space"` (members of the bound Space, when {{JMAP-CHAT}} is present). Default: `"public"`.

`createdAt` (UTCDate, immutable, server-set):
: Time the region was created.

`updatedAt` (UTCDate, server-set):
: Time the region was last modified.

### Optional Binding Fields

The following fields are present only when the corresponding companion capability is advertised on the same account:

`chatId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Chat ({{JMAP-CHAT}}) associated with this region. `null` if the region has no chat binding.

`spaceId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a Space ({{JMAP-CHAT}}) associated with this region. `null` if the region has no Space context.

`channelId` (String|null):
: When `urn:ietf:params:jmap:chat` is present: the id of a channel Chat within the Space. `null` if the region has no channel context.

`activeCallId` (String|null):
: When `urn:ietf:params:jmap:vtc` is present: the id of an active VTCCall ({{JMAP-VTC}}) bound to this region. `null` if no call is active.

### Example

~~~json
{
  "id": "01J5ABC0000000000000000001",
  "accountId": "account-xyz",
  "name": "Gallery East Wing",
  "description": "Contemporary sculpture exhibition",
  "bounds": {
    "min": [-50, 0, -50],
    "max": [50, 10, 50]
  },
  "environment": {
    "skyColor": "#87CEEB",
    "ambientIntensity": 0.6,
    "gravity": 9.81
  },
  "simulationUri": "wss://sim.example.com/regions/01J5ABC",
  "viewHint": "3d",
  "spawnPosition": [0, 0, 10],
  "spawnOrientation": [0, 0, 0, 1],
  "activeAvatarCount": 3,
  "accessPolicy": "public",
  "createdAt": "2026-06-01T10:00:00Z",
  "updatedAt": "2026-06-05T14:30:00Z",
  "chatId": "01J5ABC0000000000000000099",
  "spaceId": null,
  "channelId": null,
  "activeCallId": null
}
~~~

## SceneObject {#scene-object}

A SceneObject represents a positioned entity within a SceneRegion. It is a top-level JMAP data type.

SceneObjects form a scene graph: each object has a position, orientation, and scale relative to its parent (or relative to the region origin if it has no parent).

### SceneObject Object Fields

`id` (String, immutable, server-set):
: A ULID assigned at creation.

`regionId` (String, immutable):
: The SceneRegion this object belongs to.

`parentId` (String|null):
: The id of the parent SceneObject for scene graph hierarchy. `null` for root-level objects. When non-null, `position`, `orientation`, and `scale` are relative to the parent's transform.

`name` (String|null):
: Human-readable label for the object.

`position` (Number[3]):
: Position as `[x, y, z]` in meters, region-local (or parent-relative when `parentId` is non-null). See {{coordinate-system}}.

`orientation` (Number[4]):
: Orientation as a unit quaternion `[x, y, z, w]`. See {{coordinate-system}}.

`scale` (Number[3]):
: Scale factors `[sx, sy, sz]`. Default: `[1, 1, 1]`.

`visualRef` (String|null):
: Blob reference for the visual representation. When a SceneAsset exists for this blob, this matches the SceneAsset's `blobId`. `null` for invisible objects (triggers, spawn points, spatial audio sources).

`visualType` (String|null):
: Media type of the visual representation (e.g., `"model/gltf-binary"`). MUST be present when `visualRef` is non-null. MUST be null when `visualRef` is null.

`assetUri` (String|null):
: CDN or static-server URI for the visual asset. When present, clients SHOULD prefer this over the JMAP blob download path.

`physicsMode` (String):
: How the simulation layer should treat this object's physics. Values: `"static"` (immovable collider), `"dynamic"` (physics-simulated), `"kinematic"` (moves via script/server, not physics), `"none"` (no collision). Default: `"static"`. Enforcement is simulation-layer-defined.

`interactable` (Boolean):
: `true` if users may interact with this object (click, grab, activate). The interaction mechanism is simulation-layer-defined. Default: `false`.

`visible` (Boolean):
: `true` if the object should be rendered. Default: `true`. Objects with `visible: false` may still participate in physics (collision) depending on `physicsMode`.

`ownerId` (String|null, server-set):
: The userId of the user who created this object. `null` for system-placed objects.

`createdAt` (UTCDate, immutable, server-set):
: Time the object was created.

`updatedAt` (UTCDate, server-set):
: Time the object was last modified.

`customProperties` (Object|null):
: Deployment-defined extension properties. Opaque to the JMAP Scene specification; the server stores and relays this object without interpretation. Clients MUST ignore unrecognized keys.

### Example

~~~json
{
  "id": "01J5OBJ0000000000000000001",
  "regionId": "01J5ABC0000000000000000001",
  "parentId": null,
  "name": "Sculpture: Convergence",
  "position": [12.5, 0, -3.2],
  "orientation": [0, 0.707, 0, 0.707],
  "scale": [1, 1, 1],
  "visualRef": "blob-gltf-sculpture-001",
  "visualType": "model/gltf-binary",
  "assetUri": "https://cdn.example.com/assets/sculpture-001.glb",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "ownerId": "user:curator@example.com",
  "createdAt": "2026-06-01T12:00:00Z",
  "updatedAt": "2026-06-01T12:00:00Z",
  "customProperties": {
    "artist": "Elena Voss",
    "year": 2025,
    "medium": "Bronze"
  }
}
~~~

## SceneAvatar {#scene-avatar}

A SceneAvatar represents a user's spatial presence within a SceneRegion. It is a top-level JMAP data type.

SceneAvatar is the Scene analog of VTCParticipant: it tracks who is in a region, when they entered, and where they are. Real-time avatar position synchronization (10+ Hz updates) is handled by the simulation layer behind `simulationUri`, not by JMAP methods. The SceneAvatar record reflects the last known position and is updated periodically by the server.

### SceneAvatar Object Fields

`id` (String, immutable, server-set):
: Opaque server-assigned identifier. For authenticated users, the server SHOULD use the userId as the id within a given region.

`regionId` (String, immutable):
: The SceneRegion this avatar is in.

`userId` (String, server-set):
: The authenticated user identity string.

`displayName` (String):
: Display name for this avatar.

`position` (Number[3]):
: Last known position as `[x, y, z]`. Updated periodically by the server from the simulation layer. Not a real-time value; clients in the simulation layer receive real-time positions through the simulation protocol.

`orientation` (Number[4]):
: Last known orientation as a unit quaternion `[x, y, z, w]`.

`visualRef` (String|null):
: Blob reference for the avatar's visual representation. `null` when the avatar has no custom visual (deployment provides a default).

`visualType` (String|null):
: Media type of the avatar visual. MUST be present when `visualRef` is non-null.

`joinedAt` (UTCDate, server-set):
: Time this avatar entered the region.

`leftAt` (UTCDate|null):
: Time this avatar left the region. `null` if currently present.

`customProperties` (Object|null):
: Deployment-defined avatar state (e.g., animation, equipped items, status). Opaque to the spec; the server stores and relays without interpretation.

### Example

~~~json
{
  "id": "user:alice@example.com",
  "regionId": "01J5ABC0000000000000000001",
  "userId": "user:alice@example.com",
  "displayName": "Alice Chen",
  "position": [5.2, 0, -1.8],
  "orientation": [0, 0.383, 0, 0.924],
  "visualRef": "blob-avatar-alice-001",
  "visualType": "model/gltf-binary",
  "joinedAt": "2026-06-05T14:35:00Z",
  "leftAt": null,
  "customProperties": {
    "animation": "idle",
    "nametag": true
  }
}
~~~

# Methods

## SceneAsset Methods

### SceneAsset/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

### SceneAsset/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### SceneAsset/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

`create` accepts:

`blobId` (String, required):
: The blob reference for the asset file, previously uploaded via the JMAP upload endpoint.

`mediaType` (String, required):
: The media type of the asset. MUST be a value present in the account-level `supportedVisualTypes` or a recognized audio/image type.

Optional: `name` (String), `assetUri` (String), `sha256` (String).

The server sets `id`, `accountId`, `size`, and `createdAt`.

Example create:

~~~json
{
  "create": {
    "a0": {
      "blobId": "Gc0f032d390a5d5fa8a35",
      "mediaType": "model/gltf-binary",
      "name": "Sculpture: Convergence"
    }
  }
}
~~~

The server MUST return `invalidArguments` if `blobId` does not reference a valid blob. The server MUST return `overQuota` if the asset exceeds `maxAssetSizeBytes`.

### SceneAsset/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`mediaType` (String, optional):
: Filter by media type.

`name` (String, optional):
: Full-text search over asset name. Servers that do not support full-text search MUST return `unsupportedFilter`.

`createdAfter` (UTCDate, optional):
: Assets created at or after this time.

`createdBefore` (UTCDate, optional):
: Assets created before this time.

All filter properties are combined with logical AND.

Sort properties: `createdAt`, `name`, `size`. Default sort: `createdAt` descending.

### SceneAsset/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SceneRegion Methods

### SceneRegion/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

The server MUST return `notFound` for regions the authenticated user does not have access to (see {{access-control}}).

Example response:

~~~json
{
  "id": "01J5ABC0000000000000000001",
  "name": "Gallery East Wing",
  "bounds": {
    "min": [-50, 0, -50],
    "max": [50, 10, 50]
  },
  "simulationUri": "wss://sim.example.com/regions/01J5ABC",
  "spawnPosition": [0, 0, 10],
  "activeAvatarCount": 3,
  "accessPolicy": "public"
}
~~~

### SceneRegion/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### SceneRegion/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

#### Creating a Region

`create` accepts:

`name` (String, required):
: Display name for the region.

`bounds` (SceneBounds, required):
: The spatial extent.

Optional: `description` (String), `environment` (Object), `simulationUri` (String), `viewHint` (String), `spawnPosition` (Number[3]), `spawnOrientation` (Number[4]), `accessPolicy` (String), `chatId` (String), `spaceId` (String), `channelId` (String).

The server sets `id`, `accountId`, `activeAvatarCount` (to `0`), `createdAt`, and `updatedAt`.

Example create:

~~~json
{
  "create": {
    "r0": {
      "name": "Design Review Room",
      "bounds": {
        "min": [-20, 0, -20],
        "max": [20, 8, 20]
      },
      "spawnPosition": [0, 0, 5],
      "accessPolicy": "invite"
    }
  }
}
~~~

The server MUST return `forbidden` when `mayCreateRegion` is `false`. The server MUST return `overQuota` when `maxRegionsPerAccount` would be exceeded. The server MUST return `invalidArguments` when `bounds.min` components are not less than or equal to corresponding `bounds.max` components.

#### Updating a Region

`update` supports patching: `name`, `description`, `bounds`, `environment`, `simulationUri`, `viewHint`, `spawnPosition`, `spawnOrientation`, `accessPolicy`, and the optional binding fields (`chatId`, `spaceId`, `channelId`, `activeCallId`).

The server MUST return `forbidden` when the caller is not the region owner and does not have deployment-defined administrative privileges.

#### Destroying a Region

`destroy` removes the SceneRegion and all contained SceneObject and SceneAvatar records. Active avatars are ejected (their `leftAt` is set to the current time). The server MUST return `forbidden` when the caller is not the region owner.

### SceneRegion/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`name` (String, optional):
: Full-text search over region name. Servers that do not support full-text search MUST return `unsupportedFilter`.

`accessPolicy` (String, optional):
: Filter by access policy.

`hasActiveAvatars` (Boolean, optional):
: When `true`, filter to regions with `activeAvatarCount > 0`. When `false`, filter to empty regions.

`createdAfter` (UTCDate, optional):
: Regions created at or after this time.

`createdBefore` (UTCDate, optional):
: Regions created before this time.

All filter properties are combined with logical AND. The query MUST only return regions the authenticated user has access to (see {{access-control}}).

Sort properties: `createdAt`, `name`, `activeAvatarCount`. Default sort: `createdAt` descending.

### SceneRegion/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SceneObject Methods

### SceneObject/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

The server MUST return `notFound` for objects in regions the authenticated user does not have access to.

### SceneObject/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### SceneObject/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

#### Creating an Object

`create` accepts:

`regionId` (String, required):
: The SceneRegion to place this object in.

`position` (Number[3], required):
: Initial position.

`visualRef` (String), `visualType` (String):
: Visual representation. Both MUST be present together or both absent.

Optional: `parentId` (String), `name` (String), `orientation` (Number[4]), `scale` (Number[3]), `assetUri` (String), `physicsMode` (String), `interactable` (Boolean), `visible` (Boolean), `customProperties` (Object).

The server sets `id`, `ownerId`, `createdAt`, and `updatedAt`.

Example create:

~~~json
{
  "create": {
    "o0": {
      "regionId": "01J5ABC0000000000000000001",
      "position": [12.5, 0, -3.2],
      "orientation": [0, 0.707, 0, 0.707],
      "visualRef": "blob-gltf-sculpture-001",
      "visualType": "model/gltf-binary",
      "name": "Sculpture: Convergence",
      "physicsMode": "static",
      "interactable": true
    }
  }
}
~~~

The server MUST return `forbidden` when `mayCreateObject` is `false`. The server MUST return `overQuota` when `maxObjectsPerRegion` would be exceeded. The server MUST return `notFound` when `regionId` does not exist or the caller does not have access. The server MUST return `invalidArguments` when `visualType` is not in `supportedVisualTypes`. The server MUST return `invalidArguments` when `visualRef` is present but `visualType` is absent, or vice versa. The server MUST return `invalidArguments` when `parentId` references a SceneObject in a different region.

#### Updating an Object

`update` supports patching all mutable fields: `parentId`, `name`, `position`, `orientation`, `scale`, `visualRef`, `visualType`, `assetUri`, `physicsMode`, `interactable`, `visible`, `customProperties`.

Example update (move an object):

~~~json
{
  "update": {
    "01J5OBJ0000000000000000001": {
      "position": [15.0, 0, -5.0],
      "orientation": [0, 1, 0, 0]
    }
  }
}
~~~

The server MUST return `forbidden` when the caller is not the object's owner and does not have deployment-defined edit privileges for the region.

#### Destroying an Object

`destroy` removes the SceneObject and all its children (objects whose `parentId` references this object, recursively). The server MUST return `forbidden` when the caller is not the object's owner and does not have deployment-defined edit privileges.

### SceneObject/query {#scene-object-query}

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

This method supports spatial query filters that enable clients to retrieve objects based on position.

#### Filter Properties

`regionId` (String, required):
: The region to query. Servers MUST return `invalidArguments` when this is absent.

`name` (String, optional):
: Full-text search over object name. Servers that do not support full-text search MUST return `unsupportedFilter`.

`visualType` (String, optional):
: Filter by visual media type.

`ownerId` (String, optional):
: Filter to objects owned by this userId.

`interactable` (Boolean, optional):
: Filter to interactable or non-interactable objects.

`visible` (Boolean, optional):
: Filter to visible or invisible objects.

`physicsMode` (String, optional):
: Filter by physics mode.

#### Spatial Filters {#spatial-filters}

Servers MUST support the following spatial filter properties. These are the mandatory-to-implement baseline for spatial queries.

`withinRadius` (Object, optional):
: Filter to objects whose position falls within a sphere. Properties:
  - `center` (Number[3], required): Center point `[x, y, z]` in region-local coordinates.
  - `radius` (Number, required): Radius in meters. MUST be positive.

`withinBounds` (Object, optional):
: Filter to objects whose position falls within an axis-aligned bounding box. Properties:
  - `min` (Number[3], required): Minimum corner `[x, y, z]`.
  - `max` (Number[3], required): Maximum corner `[x, y, z]`.

When both spatial filters are present, they are combined with logical AND (the object must satisfy both).

Example query — all interactable objects within 20 meters of a point:

~~~json
[["SceneObject/query", {
  "accountId": "account-xyz",
  "filter": {
    "regionId": "01J5ABC0000000000000000001",
    "interactable": true,
    "withinRadius": {
      "center": [5.0, 0, -2.0],
      "radius": 20
    }
  },
  "sort": [{"property": "name", "isAscending": true}]
}, "0"]]
~~~

All non-spatial filter properties are combined with spatial filters using logical AND.

Sort properties: `createdAt`, `name`. Default sort: `createdAt` ascending.

### SceneObject/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SceneAvatar Methods

### SceneAvatar/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

The server MUST return `notFound` for avatars in regions the authenticated user does not have access to.

### SceneAvatar/changes

Standard JMAP `/changes` ({{RFC8620}} Section 5.2).

### SceneAvatar/set

Standard JMAP `/set` ({{RFC8620}} Section 5.3).

#### Entering a Region

A user enters a region by calling `SceneAvatar/set` with a `create`:

`regionId` (String, required):
: The SceneRegion to enter.

Optional: `visualRef` (String), `visualType` (String), `customProperties` (Object).

The server sets `id`, `userId`, `displayName` (from the user profile or ChatContact when {{JMAP-CHAT}} is present), `position` (to the region's `spawnPosition`), `orientation` (to the region's `spawnOrientation`), `joinedAt`, and `leftAt` (to `null`).

Example create:

~~~json
{
  "create": {
    "av0": {
      "regionId": "01J5ABC0000000000000000001",
      "visualRef": "blob-avatar-alice-001",
      "visualType": "model/gltf-binary"
    }
  }
}
~~~

The server responds:

~~~json
{
  "created": {
    "av0": {
      "id": "user:alice@example.com",
      "regionId": "01J5ABC0000000000000000001",
      "userId": "user:alice@example.com",
      "displayName": "Alice Chen",
      "position": [0, 0, 10],
      "orientation": [0, 0, 0, 1],
      "joinedAt": "2026-06-05T14:35:00Z",
      "leftAt": null
    }
  }
}
~~~

The server MUST return `notFound` when `regionId` does not exist or the caller does not have access. The server MUST return `forbidden` when the region's `accessPolicy` is `"invite"` and the caller has not been granted access, or `"space"` and the caller is not a member of the bound Space. The server MUST return `overQuota` when `maxAvatarsPerRegion` would be exceeded.

When the user already has an active SceneAvatar (with `leftAt: null`) in the same region, the server MUST return the existing record in `updated` rather than creating a duplicate. When the user has an active SceneAvatar in a different region, the server MUST set `leftAt` on the previous avatar before creating the new one (a user is present in at most one region at a time).

#### Updating Avatar State

A user calls `SceneAvatar/set` to update their own avatar:

~~~json
{
  "update": {
    "user:alice@example.com": {
      "visualRef": "blob-avatar-alice-formal",
      "visualType": "model/gltf-binary",
      "customProperties": {
        "animation": "waving"
      }
    }
  }
}
~~~

The server MUST return `forbidden` if the caller's userId does not match the SceneAvatar record's userId. Users MUST NOT update other users' avatars.

Position and orientation updates via `SceneAvatar/set` are permitted but are expected to be infrequent (e.g., teleporting to a bookmark). Continuous position synchronization is handled by the simulation layer.

#### Leaving a Region

A user leaves by calling `SceneAvatar/set` with an update setting `leftAt`:

~~~json
{
  "update": {
    "user:alice@example.com": {
      "leftAt": "2026-06-05T15:45:00Z"
    }
  }
}
~~~

The server MUST return `forbidden` if the caller's userId does not match the SceneAvatar record's userId. The server MUST return `invalidArguments` if the avatar has already left (`leftAt` is non-null).

#### Reconnecting

When a user who has left (non-null `leftAt`) re-enters the same region, the server MUST update the existing SceneAvatar record rather than creating a new one: clear `leftAt` to `null`, update `joinedAt` to the current time, and set `position` to `spawnPosition`. This ensures a single continuous identity.

### SceneAvatar/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

Filter properties:

`regionId` (String, optional):
: Filter to avatars in this region. Servers SHOULD require this property unless combined with `userId`.

`userId` (String, optional):
: Filter to a specific user across regions.

`isActive` (Boolean, optional):
: When `true`, filter to avatars with `leftAt == null`. When `false`, filter to avatars who have left.

Spatial filters:

`withinRadius` (Object, optional):
: Same syntax as SceneObject/query ({{spatial-filters}}).

`withinBounds` (Object, optional):
: Same syntax as SceneObject/query ({{spatial-filters}}).

All filter properties are combined with logical AND.

Sort properties: `joinedAt`, `displayName`. Default sort: `joinedAt` ascending.

### SceneAvatar/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

# Real-Time Simulation Layer {#simulation-layer}

The JMAP Scene specification defines spatial state at rest: what objects exist, where they are, and who is present. Real-time spatial state in motion — avatar position synchronization at 10+ Hz, physics simulation, spatial audio mixing, interaction events — is handled by the simulation layer behind `simulationUri`.

The simulation protocol is deployment-defined. This specification does not prescribe WebRTC data channels, UDP, QUIC, WebSocket, or any specific transport. Deployments choose their simulation stack based on their latency, scalability, and feature requirements.

## Simulation Layer Responsibilities

The simulation layer is responsible for:

- **Avatar position/orientation synchronization** at a rate appropriate for the experience (typically 10-20 Hz for social VR, 30-60 Hz for competitive environments).
- **Interest management**: determining which updates to send to which client based on spatial proximity, visibility, or other criteria. The specific algorithm (radius-based, frustum-based, occlusion-based) is deployment-defined.
- **Object state synchronization** for dynamic objects (`physicsMode: "dynamic"` or `"kinematic"`).
- **Interaction events** (click, grab, activate) for interactable objects.
- **Spatial audio mixing** when voice (via {{JMAP-VTC}}) is combined with avatar positions.

## State Reconciliation

The server SHOULD periodically reconcile the simulation layer's state with the JMAP Scene state:

- Avatar positions in SceneAvatar records SHOULD be updated from the simulation layer at a deployment-defined interval (RECOMMENDED: every 5-30 seconds). These updates are not real-time; they serve as a fallback for clients that do not have a simulation layer connection.
- Objects moved by the simulation layer (dynamic physics, scripted movement) SHOULD have their SceneObject position/orientation updated via the standard JMAP state-change mechanism.

Clients connected to the simulation layer SHOULD use simulation-layer positions for rendering and SHOULD treat SceneAvatar and SceneObject positions as stale fallbacks.

## Visibility Contract {#visibility-contract}

The server MUST NOT include a SceneObject in a `SceneObject/get` response or `SceneObject/query` result if the authenticated user does not have access to the object's containing SceneRegion.

The server SHOULD apply visibility filtering to limit the objects returned based on the authenticated user's avatar position within the region. The specific visibility algorithm is deployment-defined:

- A simple deployment MAY return all objects in the region.
- A security-conscious deployment MAY return only objects within a radius of the avatar's last known position.
- A high-security deployment MAY perform server-side occlusion culling, returning only objects in the avatar's potential visibility set.

The spec does not dictate the algorithm. It requires that the access-control check is enforced (MUST) and that visibility optimization is available as a deployment choice (SHOULD).

# Access Control {#access-control}

## Region Access

`accessPolicy` on SceneRegion determines who may enter:

`"public"`:
: Any authenticated user may enter and view objects. The server MAY impose rate limits on entry.

`"invite"`:
: Only users explicitly granted access may enter. The server MUST return `forbidden` for `SceneAvatar/set` create from unauthorized users. The mechanism for granting invitations is deployment-defined.

`"space"`:
: When `urn:ietf:params:jmap:chat` is present and `spaceId` is set: only members of the bound Space may enter. The server MUST return `forbidden` for non-members.

Regardless of `accessPolicy`, the server MUST return `notFound` (not `forbidden`) for `SceneRegion/get`, `SceneObject/get`, and `SceneAvatar/get` requests from users who do not have access. This prevents access-policy enumeration.

## Object Permissions

SceneObject modifications (update, destroy) are restricted to:

1. The object's owner (`ownerId`).
2. The region owner.
3. Users with deployment-defined administrative privileges.

All other users MUST receive `forbidden` for `SceneObject/set` updates or destroys targeting objects they do not own.

## Avatar Permissions

Users may only modify their own SceneAvatar record. The server MUST return `forbidden` for `SceneAvatar/set` updates targeting another user's avatar.

Region owners and administrators MAY eject avatars by setting `leftAt` on another user's SceneAvatar. This is the Scene equivalent of kicking a participant from a VTCCall.

# Security Considerations {#security}

## simulationUri Is Untrusted {#simulation-uri-untrusted}

`simulationUri` is peer-supplied and opaque to the JMAP server. Clients MUST NOT connect to `simulationUri` without explicit user initiation. Auto-connecting to a simulation endpoint exposes the client to malicious servers.

Servers MUST NOT fetch, probe, or validate `simulationUri` values. Doing so exposes the server to SSRF attacks.

## assetUri Is Untrusted {#asset-uri-untrusted}

`assetUri` on SceneObject and SceneAsset is peer-supplied. Clients MUST validate that `assetUri` uses a permitted scheme (`https:`) before fetching. Clients MUST NOT follow redirects to non-HTTPS endpoints. Fetching a malicious `assetUri` could expose the client to tracking (via unique URLs per user), content injection, or exploit payloads disguised as 3D assets.

Clients SHOULD verify asset integrity using the `sha256` field on SceneAsset when available.

## Asset Parsing

3D asset files (glTF, images, audio) are complex formats with large attack surfaces. Clients MUST parse assets in a sandboxed context. A malformed glTF file MUST NOT be able to escape the rendering sandbox to access local files, network resources, or other browser/OS capabilities.

## Position Spoofing

Avatar positions reported by the client (via `SceneAvatar/set` or the simulation layer) are self-reported and unverifiable by the server. A malicious client could report a position it is not actually at, potentially bypassing proximity-based access controls or eavesdropping on proximity chat.

Deployments that require position integrity SHOULD implement server-side position validation in the simulation layer (e.g., maximum movement speed, collision with barriers). This is a simulation-layer concern, not a JMAP Scene concern.

## Spatial Metadata Exposure

The JMAP server observes who enters which regions, when, for how long, and (at periodic intervals) where they are. This metadata is privacy-sensitive. Deployments requiring spatial privacy SHOULD minimize the frequency of position reconciliation between the simulation layer and the JMAP server.

## Object Density as Denial of Service

Without limits, a malicious user could create thousands of objects in a region, overwhelming clients that attempt to render them all. Servers MUST enforce `maxObjectsPerRegion`. Servers SHOULD impose per-user object creation rate limits.

## Visibility and Information Leakage

SceneObject data returned to a client reveals information about the scene (object names, positions, custom properties). Deployments where scene content is sensitive (e.g., a competitive game with hidden objects) SHOULD use the visibility filtering described in {{visibility-contract}} to limit what data reaches each client.

# IANA Considerations {#iana}

## JMAP Capability Registration

IANA is requested to register the following entry in the "JMAP Capabilities" registry:

Capability Name:
: `urn:ietf:params:jmap:scene`

Intended Use:
: common

Change Controller:
: IETF

Specification document:
: This document.

Security and privacy considerations:
: See {{security}} of this document.

## JMAP Data Type Registration

IANA is requested to register the following entries in the "JMAP Data Types" registry:

Type Name: SceneRegion
Can reference blobs: No
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:scene`
Specification document: This document.

Type Name: SceneObject
Can reference blobs: Yes
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:scene`
Specification document: This document.

Type Name: SceneAvatar
Can reference blobs: Yes
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:scene`
Specification document: This document.

Type Name: SceneAsset
Can reference blobs: Yes
Can use for state change: Yes
Capability: `urn:ietf:params:jmap:scene`
Specification document: This document.

--- back

# Design Influences and Non-Normative Notes

## Influences

- **Mozilla Hubs** provided the primary reference for a browser-based social 3D environment: WebRTC-based voice with spatial audio, room-based entry, avatar presence, and media sharing. The SceneRegion/SceneAvatar model mirrors Hubs' room/participant model.
- **Second Life** informed the persistent-world model: user-created objects with permissions, land ownership (modeled as region access control), and the separation between the world server and the voice server (Vivox, later WebRTC). The SceneObject permission model (owner, region owner, admin) follows SL's pattern.
- **glTF 2.0** ({{GLTF}}) provided the coordinate system convention (right-handed, Y-up, meters, quaternion orientation) and the mandatory-to-implement visual asset format.
- **A-Frame / Three.js** informed the scene graph model: objects with position/orientation/scale, parent-child hierarchy, and component-like custom properties.
- **Gather** (and similar 2D spatial-video products like Amazon Chime's experimental spatial view) demonstrated that top-down 2D worlds with proximity-based video and spatial audio are a compelling subset of 3D environments, particularly for virtual offices and social spaces. The `viewHint` field on SceneRegion was designed to support this use case natively without requiring a separate specification.
- **Video games** — both single-player and multiplayer — informed the decision to keep Scene independent of Chat and VTC. A game world needs spatial objects, avatars, physics modes, and interaction events but may not need chat channels or video calling. The `SceneInteractionEvent` action vocabulary (click, grab, release, activate, plus extensible custom actions) follows game input conventions.
- **Board games and tabletop simulators** (Tabletop Simulator, Board Game Arena) influenced the SceneObject permission model and the 2D top-down view hint. A board game is a scene region with a fixed set of objects (pieces, cards, dice) that players interact with via grab/release/activate actions, rendered top-down.
- **Game engine conventions** (Unity, Unreal, Godot) informed the physicsMode and interactable properties. The separation between physics modes (static, dynamic, kinematic, none) is standard across all major engines.

## Explicit Non-Prescriptions

The following design choices were left to deployments rather than prescribed:

- **Simulation protocol.** WebRTC data channels, UDP, QUIC, WebSocket, or any other transport for real-time position synchronization. The spec is simulation-agnostic by design.
- **Visual asset format.** `model/gltf-binary` is mandatory-to-implement, but deployments may support gaussian splats, neural radiance fields, voxels, USD, or any future format via the `visualType` field.
- **Rendering engine.** Three.js, Babylon.js, Unity, Unreal, native Vulkan/Metal, or any other renderer. The spec has no rendering-layer surface.
- **Interest management algorithm.** Radius-based, frustum-based, occlusion-based, portal-based, or none. Deployment-defined.
- **Physics engine.** Bullet, PhysX, Rapier, Cannon.js, or none. The `physicsMode` field signals intent; the simulation layer implements it.
- **Avatar system.** Full-body tracking, head-and-hands, 2D sprites, abstract shapes. The `visualRef` points to whatever the deployment uses.
- **Authentication for region entry.** OAuth, JMAP auth, tickets, or any other mechanism. The spec defines access policies; the auth mechanism is deployment infrastructure.
- **Spatial audio.** How voice from {{JMAP-VTC}} is spatialized based on avatar positions. This is a simulation-layer concern at the intersection of VTC and Scene.
- **Scripting and behaviors.** Object behaviors, triggers, animations, and interaction logic. Out of scope. May be modeled via `customProperties` or a future companion specification.
- **Economy and inventory.** Virtual currencies, item trading, user inventories. Out of scope.
- **Terrain and heightmaps.** Terrain is a SceneObject with a visual representation. The spec does not define a terrain-specific data type.
- **Building and editing tools.** Client-side concerns. Objects are created and modified via standard `SceneObject/set`; the UI for doing so is client-defined.
- **2D rendering style.** For regions with `viewHint` of `"2d-topdown"` or `"2d-side"`, the rendering style (pixel art, vector, isometric projection, sprite sheets) is client-defined. The spec provides spatial coordinates and advisory view orientation; visual style is a client concern.
- **Game rules and turn logic.** For board-game or game use cases, rules enforcement, turn order, scoring, and win conditions are application logic outside the spec. Scene provides the spatial state layer; game logic runs above it.

# Acknowledgements

The author thanks the Mozilla Hubs project for the open-source reference implementation that demonstrated browser-based social 3D environments are viable, the Second Life team for two decades of persistent-world operation that informed the permission and region models, the Khronos Group for glTF 2.0 whose coordinate conventions this specification adopts, and the Gather team for demonstrating that 2D spatial environments are a compelling and practical subset of 3D worlds.
