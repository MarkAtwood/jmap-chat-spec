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
  GLTF:
    title: "glTF 2.0 Specification"
    target: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html
    author:
      org: Khronos Group
    date: 2021

informative:
  RFC6455:
  RFC8887:
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

The specification models spatial *state* — what objects exist, where they are, and who is present — without prescribing the rendering technology or simulation protocol. Actual real-time position synchronization, physics, and rendering travel through a deployment-chosen simulation stack; the JMAP server is a spatial state database, not a rendering engine.

The capability is standalone (`urn:ietf:params:jmap:scene`). When JMAP Chat ({{JMAP-CHAT}}) is also present, SceneRegion objects MAY carry back-references to Chat and Space objects for in-world text communication. When JMAP VTC ({{JMAP-VTC}}) is also present, SceneRegion objects MAY bind to VTCCall objects for in-world voice and video.

--- middle

# Introduction

Three-dimensional spatial environments — virtual worlds, collaborative design spaces, immersive galleries, architectural walkthroughs, data visualizations — are built by many systems, each with its own state model, wire protocol, and asset pipeline. There is no standard API for the spatial state layer: what objects exist in a scene, where they are positioned, who is present, and who may modify what.

This document defines a JMAP capability that models spatial state as JMAP data types with standard get/set/changes/query methods. The server tracks what regions exist, what objects are in them, and which users are present — but the server does not render frames, simulate physics, or negotiate media codecs. A `simulationUri` field on every SceneRegion points to the deployment's real-time simulation layer; JMAP Scene has no opinion on what lives behind that URI.

## Design Philosophy

- **Spatial state, not spatial rendering.** The JMAP server is a spatial state database. It knows a region exists, what objects are in it, and where they are. It does not rasterize triangles, compute lighting, or run physics. Rendering and simulation are the simulation stack's job.
- **Preferred format with extensibility.** glTF 2.0 ({{GLTF}}) is the mandatory-to-implement baseline asset format, chosen for its royalty-free licensing, universal engine support, and matching coordinate conventions. Deployments MAY support additional visual formats (gaussian splats, neural radiance fields, voxels, USD) via the `supportedVisualTypes` capability. As visual formats evolve, the spec does not need amendment.
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

### Preferred Asset Format {#preferred-asset-format}

glTF 2.0 ({{GLTF}}) is the preferred and mandatory-to-implement visual asset format. The IANA media type `model/gltf-binary` (.glb) MUST be listed in `supportedVisualTypes`; servers SHOULD also support `model/gltf+json` (.gltf).

glTF 2.0 is the baseline because:

- It is a royalty-free, open standard maintained by the Khronos Group.
- It uses the same coordinate system as this specification: right-handed, Y-up, meters, quaternion orientation.
- A single .glb file carries meshes, PBR materials, textures, skeletal animations, morph targets, and scene hierarchy — everything needed to represent a SceneObject or SceneAvatar visual.
- It is supported by all major 3D engines (Unity, Unreal, Godot, Three.js, Babylon.js) and content creation tools (Blender, Maya, 3ds Max), making it the closest equivalent to a universal 3D interchange format.
- The binary container (.glb) is a single self-contained file suitable for JMAP blob storage and CDN distribution via `assetUri`.

Clients SHOULD prefer glTF when creating assets intended for cross-deployment portability. Assets in deployment-specific formats (gaussian splats, neural radiance fields, voxels, USD) are valid within deployments that advertise those media types in `supportedVisualTypes`, but are not guaranteed to be portable.

For security considerations related to glTF parsing, see {{asset-uri-untrusted}}.

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
: When `urn:ietf:params:jmap:vtc` is present: the id of an active VTCCall ({{JMAP-VTC}}) bound to this region. `null` if no call is active. When the VTCCall referenced by `activeCallId` transitions to state `"ended"`, the server MUST clear this field to `null` and emit a `StateChange` for the SceneRegion type. This update is a side effect of the VTCCall state transition.

When a SceneRegion has both `chatId` and `activeCallId` set, and the bound Chat also has an `activeCallId` field (per {{JMAP-CHAT}}), the two fields SHOULD reference the same VTCCall. If they diverge (e.g., due to independent client updates), the SceneRegion's `activeCallId` is authoritative for spatial audio binding and the Chat's `activeCallId` is authoritative for the text-channel call banner. Clients SHOULD treat divergence as transient and prefer the SceneRegion's value for in-region UI.

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

Example create request with full JMAP envelope:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAsset/set", {
      "accountId": "acct-001",
      "create": {
        "a0": {
          "blobId": "Gc0f032d390a5d5fa8a35",
          "mediaType": "model/gltf-binary",
          "name": "Sculpture: Convergence",
          "assetUri": "https://cdn.example.com/assets/sculpture-001.glb",
          "sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"
        }
      }
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAsset/set", {
      "accountId": "acct-001",
      "oldState": "state-asset-5",
      "newState": "state-asset-6",
      "created": {
        "a0": {
          "id": "01J5AST0000000000000000001",
          "accountId": "acct-001",
          "size": 2483712,
          "createdAt": "2026-06-06T10:05:00Z"
        }
      },
      "updated": null,
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "a0"]
  ]
}
~~~

Example destroy request:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAsset/set", {
      "accountId": "acct-001",
      "destroy": [
        "01J5AST0000000000000000001"
      ]
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAsset/set", {
      "accountId": "acct-001",
      "oldState": "state-asset-6",
      "newState": "state-asset-7",
      "created": null,
      "updated": null,
      "destroyed": [
        "01J5AST0000000000000000001"
      ],
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "a0"]
  ]
}
~~~

#### SetError Conditions

##### Create Errors

**`invalidArguments`** -- missing or malformed fields:

| Condition | Description |
|---|---|
| `blobId` is missing | `blobId` is required; it references the previously-uploaded blob. |
| `mediaType` is missing | `mediaType` is required. |
| `mediaType` is not in `supportedVisualTypes` and is not a recognized audio/image type | The server must support the declared media type. |

**`notFound`** -- referenced blob does not exist:

| Condition | Description |
|---|---|
| `blobId` does not reference a valid blob | The blob must have been previously uploaded via the JMAP upload endpoint. The server MAY use either `invalidArguments` or `notFound` depending on whether the blobId is syntactically valid but absent (notFound) vs. malformed (invalidArguments). |

**`forbidden`** -- authorization failures:

| Condition | Description |
|---|---|
| Deployment restricts shared-asset creation to administrators | When the deployment policy requires admin privileges to create assets visible to other users, non-admins receive `forbidden`. |

**`overQuota`** -- storage limits:

| Condition | Description |
|---|---|
| Asset exceeds `maxAssetSizeBytes` | The blob's size exceeds the per-asset size limit from the account-level capability. |
| Account asset storage quota exceeded | Deployment-defined total storage limits across all assets. |

##### Update Errors

**`invalidArguments`**:

| Condition | Description |
|---|---|
| Attempting to change immutable fields (`blobId`, `id`, `createdAt`) | These fields are immutable after creation. |

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller does not own the asset and lacks admin privileges | Only the asset owner or an admin may modify asset metadata. |

**`notFound`**:

| Condition | Description |
|---|---|
| The asset id does not exist | No SceneAsset with that id is known. |

##### Destroy Errors

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller does not own the asset and lacks admin privileges | Same permission model as update. |

**`notFound`**:

| Condition | Description |
|---|---|
| The asset id does not exist | No SceneAsset with that id is known. |

### SceneAsset/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

#### Filter Properties

`mediaType` (String, optional):
: Filter by media type.

`name` (String, optional):
: Substring match on asset name. Servers that do not support substring matching MUST return `unsupportedFilter`.

`createdAfter` (UTCDate, optional):
: Assets created at or after this timestamp.

`createdBefore` (UTCDate, optional):
: Assets created before this timestamp.

#### Filter Validation

All filter properties are combined with logical AND. An asset must match every specified filter property to appear in the result set.

If a client provides a filter property the server does not support, the server MUST return an `unsupportedFilter` error.

Sort properties: `createdAt`, `name`, `size`. Default sort: `createdAt` descending.

Example query -- glTF assets named "sculpture" created in a date range:

~~~json
[["SceneAsset/query", {
  "accountId": "account-xyz",
  "filter": {
    "mediaType": "model/gltf-binary",
    "name": "sculpture",
    "createdAfter": "2026-03-01T00:00:00Z",
    "createdBefore": "2026-06-01T00:00:00Z"
  },
  "sort": [{"property": "createdAt", "isAscending": false}],
  "position": 0,
  "limit": 25
}, "0"]]
~~~

### SceneAsset/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SceneRegion Methods

### SceneRegion/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

The server MUST return `notFound` for regions the authenticated user does not have access to (see {{access-control}}).

Example response (abbreviated):

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

Example full request and response -- request two regions by id:

Request:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneRegion/get", {
      "accountId": "acct-001",
      "ids": [
        "01J5ABC0000000000000000001",
        "01J5ABC0000000000000000002"
      ]
    }, "r0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneRegion/get", {
      "accountId": "acct-001",
      "state": "state-region-47",
      "list": [
        {
          "id": "01J5ABC0000000000000000001",
          "accountId": "acct-001",
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
          "simulationUri": "wss://sim.example.com/regions/01J5ABC0001",
          "viewHint": "3d",
          "spawnPosition": [0, 0, 10],
          "spawnOrientation": [0, 0, 0, 1],
          "activeAvatarCount": 3,
          "accessPolicy": "public",
          "createdAt": "2026-06-01T10:00:00Z",
          "updatedAt": "2026-06-05T14:30:00Z",
          "chatId": "01J5CHAT000000000000000099",
          "spaceId": null,
          "channelId": null,
          "activeCallId": null
        },
        {
          "id": "01J5ABC0000000000000000002",
          "accountId": "acct-001",
          "name": "Design Review Room",
          "description": null,
          "bounds": {
            "min": [-20, 0, -20],
            "max": [20, 8, 20]
          },
          "environment": null,
          "simulationUri": "wss://sim.example.com/regions/01J5ABC0002",
          "viewHint": "3d",
          "spawnPosition": [0, 0, 5],
          "spawnOrientation": [0, 0, 0, 1],
          "activeAvatarCount": 0,
          "accessPolicy": "invite",
          "createdAt": "2026-06-02T08:00:00Z",
          "updatedAt": "2026-06-02T08:00:00Z",
          "chatId": null,
          "spaceId": null,
          "channelId": null,
          "activeCallId": null
        }
      ],
      "notFound": []
    }, "r0"]
  ]
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

When `chatId`, `spaceId`, or `channelId` is set to a non-null value, the server MUST verify that the referenced object exists and the caller has access to it. The server MUST return `invalidArguments` if the referenced object does not exist or the caller cannot access it.

Example create request with full JMAP envelope:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "create": {
        "r0": {
          "name": "Main Plaza",
          "description": "Central open-air gathering area",
          "bounds": {
            "min": [-500, -10, -500],
            "max": [500, 200, 500]
          },
          "environment": {
            "skyColor": "#4A90D9",
            "ambientIntensity": 0.8,
            "gravity": 9.81,
            "fogDensity": 0.002
          },
          "simulationUri": "wss://sim.example.com/regions/plaza",
          "viewHint": "3d",
          "spawnPosition": [0, 1, 15],
          "spawnOrientation": [0, 0, 0, 1],
          "accessPolicy": "public"
        }
      }
    }, "r0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "oldState": "state-region-47",
      "newState": "state-region-48",
      "created": {
        "r0": {
          "id": "01J5REG0000000000000000003",
          "accountId": "acct-001",
          "activeAvatarCount": 0,
          "createdAt": "2026-06-06T09:15:00Z",
          "updatedAt": "2026-06-06T09:15:00Z"
        }
      },
      "updated": null,
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "r0"]
  ]
}
~~~

#### Updating a Region

`update` supports patching: `name`, `description`, `bounds`, `environment`, `simulationUri`, `viewHint`, `spawnPosition`, `spawnOrientation`, `accessPolicy`, and the optional binding fields (`chatId`, `spaceId`, `channelId`, `activeCallId`).

When `activeCallId` is set to a non-null value, the server MUST verify that: (a) the referenced VTCCall exists, (b) its state is not `"ended"`, and (c) the caller is a participant in or moderator of the referenced call, or has deployment-defined administrative privileges. The server MUST return `invalidArguments` if the VTCCall does not exist or is ended, and `forbidden` if the caller lacks access.

The server MUST return `forbidden` when the caller is not the region owner and does not have deployment-defined administrative privileges.

Example update -- patch a region's accessPolicy and bounds:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "update": {
        "01J5ABC0000000000000000002": {
          "accessPolicy": "public",
          "bounds": {
            "min": [-40, 0, -40],
            "max": [40, 12, 40]
          }
        }
      }
    }, "r0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "oldState": "state-region-48",
      "newState": "state-region-49",
      "created": null,
      "updated": {
        "01J5ABC0000000000000000002": null
      },
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "r0"]
  ]
}
~~~

#### Destroying a Region

`destroy` removes the SceneRegion and all contained SceneObject and SceneAvatar records. Active avatars are ejected (their `leftAt` is set to the current time). The server MUST return `forbidden` when the caller is not the region owner.

If the destroyed SceneRegion had a non-null `activeCallId`, the server MUST NOT automatically end the referenced VTCCall. The VTCCall continues independently; its participants are unaffected. The server SHOULD deliver a `StateChange` for the VTCCall type so that clients can detect the loss of the spatial context. Deployments that want region destruction to end the call MUST implement this as application logic outside the base protocol.

Example destroy:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "destroy": [
        "01J5REG0000000000000000003"
      ]
    }, "r0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneRegion/set", {
      "accountId": "acct-001",
      "oldState": "state-region-49",
      "newState": "state-region-50",
      "created": null,
      "updated": null,
      "destroyed": [
        "01J5REG0000000000000000003"
      ],
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "r0"]
  ]
}
~~~

#### SetError Conditions

##### Create Errors

**`invalidArguments`** -- missing or malformed required fields:

| Condition | Description |
|---|---|
| `name` is missing or empty | `name` is a required String field. |
| `bounds` is missing | `bounds` (SceneBounds) is required at creation. |
| `bounds.min[i] > bounds.max[i]` for any component | Each component of `min` MUST be less than or equal to the corresponding component of `max`. |
| `bounds.min` or `bounds.max` is not a 3-element numeric array | Position arrays MUST be `[x, y, z]` Numbers. |
| `accessPolicy` is not one of `"public"`, `"invite"`, `"space"` | Unrecognized access-policy values are invalid. |
| `viewHint` contains a value the server considers invalid | While clients tolerate unknown viewHint values, the server MAY reject nonsensical values at write time. Standard values: `"3d"`, `"2d-topdown"`, `"2d-side"`, or reverse-domain extensions. |
| `spawnPosition` is not a valid 3-element numeric array | Position must be `[x, y, z]` finite Numbers. |
| `spawnOrientation` is not a valid unit quaternion | Quaternion must be `[x, y, z, w]` with magnitude within epsilon of 1.0. |
| `environment` contains values the server rejects | When the server validates environment sub-fields (deployment-defined schema), invalid field types or values trigger this error. |
| `accessPolicy` is `"space"` but `urn:ietf:params:jmap:chat` is not in the server's capabilities | `accessPolicy "space" requires urn:ietf:params:jmap:chat`. |
| `accessPolicy` is `"space"` but `spaceId` is null | `accessPolicy "space" requires spaceId`. |

**`forbidden`** -- authorization failures:

| Condition | Description |
|---|---|
| `mayCreateRegion` is `false` | The account-level capability indicates the user may not create regions. |
| Caller lacks deployment-defined administrative privileges required for region creation | Deployment-specific authorization rules deny the operation. |

**`overQuota`** -- resource limits:

| Condition | Description |
|---|---|
| Creating the region would exceed `maxRegionsPerAccount` | The account-level cap on total regions has been reached. |

Example error response -- invalidArguments (bounds validation):

~~~json
[["SceneRegion/set", {
  "accountId": "account-xyz",
  "oldState": "state-41",
  "newState": "state-41",
  "created": null,
  "updated": null,
  "destroyed": null,
  "notCreated": {
    "r0": {
      "type": "invalidArguments",
      "description": "bounds.min[0] (100) is greater than bounds.max[0] (50); each min component must be <= the corresponding max component."
    }
  },
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

##### Update Errors

**`invalidArguments`** -- malformed patch values:

Same field-level validations as create apply: `bounds.min >= bounds.max`, invalid `accessPolicy`, invalid `viewHint`, non-unit `spawnOrientation`, malformed `environment` fields.

**`forbidden`** -- authorization failures:

| Condition | Description |
|---|---|
| Caller is not the region owner and lacks administrative privileges | Only the owner or an admin may modify region properties. |

**`notFound`** -- target does not exist:

| Condition | Description |
|---|---|
| The region id does not exist | No SceneRegion with that id is known to the server. |
| The region exists but the caller lacks access | Per the access-control enumeration rule, the server returns `notFound` rather than `forbidden` to prevent probing. |

##### Destroy Errors

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller is not the region owner | Only the region owner may destroy a region. |

**`notFound`**:

| Condition | Description |
|---|---|
| The region id does not exist or the caller lacks access | Same enumeration-safe rule as update. |

### SceneRegion/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

#### Filter Properties

`name` (String, optional):
: Substring match on region name. Servers that do not support substring matching MUST return `unsupportedFilter`.

`accessPolicy` (String, optional):
: Exact match on access policy. One of `"public"`, `"invite"`, or `"space"`.

`viewHint` (String, optional):
: Exact match on the `viewHint` field. Standard values include `"3d"`, `"2d-topdown"`, and `"2d-side"`; deployment-specific values in reverse-domain notation are also valid filter values.

`hasSimulationUri` (Boolean, optional):
: When `true`, filter to regions where `simulationUri` is non-null. When `false`, filter to regions where `simulationUri` is null.

`hasActiveAvatars` (Boolean, optional):
: When `true`, filter to regions with `activeAvatarCount > 0`. When `false`, filter to empty regions.

`createdAfter` (UTCDate, optional):
: Regions created at or after this timestamp.

`createdBefore` (UTCDate, optional):
: Regions created before this timestamp.

#### Filter Validation

All filter properties are combined with logical AND. A region must match every specified filter property to appear in the result set. The query MUST only return regions the authenticated user has access to (see {{access-control}}).

If a client provides a filter property the server does not support, the server MUST return an `unsupportedFilter` error. If a client provides an invalid value for `accessPolicy` (not one of the defined values), the server MUST return `invalidArguments`.

Sort properties: `createdAt`, `name`, `activeAvatarCount`. Default sort: `createdAt` descending.

Example query -- public regions sorted by population:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneRegion/query", {
      "accountId": "acct-001",
      "filter": {
        "accessPolicy": "public"
      },
      "sort": [
        {"property": "activeAvatarCount", "isAscending": false}
      ],
      "position": 0,
      "limit": 10
    }, "r0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneRegion/query", {
      "accountId": "acct-001",
      "queryState": "qstate-region-12",
      "canCalculateChanges": true,
      "position": 0,
      "ids": [
        "01J5ABC0000000000000000001",
        "01J5ABC0000000000000000002"
      ],
      "total": 2
    }, "r0"]
  ]
}
~~~

Example query -- public regions with a simulation layer, created after a date:

~~~json
[["SceneRegion/query", {
  "accountId": "account-xyz",
  "filter": {
    "accessPolicy": "public",
    "hasSimulationUri": true,
    "createdAfter": "2026-01-01T00:00:00Z"
  },
  "sort": [{"property": "createdAt", "isAscending": false}],
  "position": 0,
  "limit": 50
}, "0"]]
~~~

### SceneRegion/queryChanges

Standard JMAP `/queryChanges` ({{RFC8620}} Section 5.6).

## SceneObject Methods

### SceneObject/get

Standard JMAP `/get` ({{RFC8620}} Section 5.1).

The server MUST return `notFound` for objects in regions the authenticated user does not have access to.

Example request and response -- request two objects by id:

Request:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/get", {
      "accountId": "acct-001",
      "ids": [
        "01J5OBJ0000000000000000001",
        "01J5OBJ0000000000000000002"
      ]
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/get", {
      "accountId": "acct-001",
      "state": "state-obj-33",
      "list": [
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
        },
        {
          "id": "01J5OBJ0000000000000000002",
          "regionId": "01J5ABC0000000000000000001",
          "parentId": null,
          "name": "Info Kiosk",
          "position": [0, 0, 8],
          "orientation": [0, 0, 0, 1],
          "scale": [0.8, 0.8, 0.8],
          "visualRef": "blob-gltf-kiosk-001",
          "visualType": "model/gltf-binary",
          "assetUri": "https://cdn.example.com/assets/kiosk-001.glb",
          "physicsMode": "static",
          "interactable": true,
          "visible": true,
          "ownerId": "user:curator@example.com",
          "createdAt": "2026-06-01T12:30:00Z",
          "updatedAt": "2026-06-03T09:15:00Z",
          "customProperties": null
        }
      ],
      "notFound": []
    }, "o0"]
  ]
}
~~~

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

Example create request with full JMAP envelope:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "create": {
        "o0": {
          "regionId": "01J5ABC0000000000000000001",
          "parentId": null,
          "name": "Pedestal A",
          "position": [8.0, 0, -6.5],
          "orientation": [0, 0.383, 0, 0.924],
          "scale": [1.2, 1.0, 1.2],
          "visualRef": "blob-gltf-pedestal-001",
          "visualType": "model/gltf-binary",
          "assetUri": "https://cdn.example.com/assets/pedestal-001.glb",
          "physicsMode": "static",
          "interactable": false,
          "visible": true,
          "customProperties": {
            "material": "marble",
            "label": "Pedestal for rotating exhibit"
          }
        }
      }
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "oldState": "state-obj-33",
      "newState": "state-obj-34",
      "created": {
        "o0": {
          "id": "01J5OBJ0000000000000000003",
          "ownerId": "user:curator@example.com",
          "createdAt": "2026-06-06T10:00:00Z",
          "updatedAt": "2026-06-06T10:00:00Z"
        }
      },
      "updated": null,
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "o0"]
  ]
}
~~~

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

Example update request with full JMAP envelope -- move an object:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "update": {
        "01J5OBJ0000000000000000001": {
          "position": [15.0, 0, -5.0]
        }
      }
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "oldState": "state-obj-34",
      "newState": "state-obj-35",
      "created": null,
      "updated": {
        "01J5OBJ0000000000000000001": null
      },
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "o0"]
  ]
}
~~~

#### Destroying an Object

`destroy` removes the SceneObject and all its children (objects whose `parentId` references this object, recursively). The server MUST return `forbidden` when the caller is not the object's owner and does not have deployment-defined edit privileges.

Example destroy -- children are also destroyed recursively:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "destroy": [
        "01J5OBJ0000000000000000003"
      ]
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/set", {
      "accountId": "acct-001",
      "oldState": "state-obj-35",
      "newState": "state-obj-36",
      "created": null,
      "updated": null,
      "destroyed": [
        "01J5OBJ0000000000000000003"
      ],
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "o0"]
  ]
}
~~~

#### SetError Conditions

##### Create Errors

**`invalidArguments`** -- missing or malformed required fields:

| Condition | Description |
|---|---|
| `regionId` is missing | `regionId` is required; it identifies the containing SceneRegion. |
| `position` is missing | `position` is required at creation. |
| `position` contains NaN or Infinity | Position components MUST be finite Numbers. |
| `orientation` is not a unit quaternion | Magnitude must be within epsilon of 1.0. NaN or Infinity components are rejected. |
| `scale` contains non-finite Numbers | Scale factors must be finite. |
| `physicsMode` is not one of `"static"`, `"dynamic"`, `"kinematic"`, `"none"` | Unrecognized physics-mode values are invalid. |
| `visualType` is not in `supportedVisualTypes` | The media type must appear in the account-level capability. |
| `visualRef` is present but `visualType` is absent (or vice versa) | Both must be present together or both absent. |
| `parentId` references a SceneObject in a different region | Parent-child relationships are region-scoped. |
| `parentId` creates a circular reference | Setting `parentId` such that the object is its own ancestor (directly or transitively) is invalid. |
| `parentId` would cause scene-graph depth to exceed deployment limit | Servers MAY impose a maximum hierarchy depth. |

**`forbidden`** -- authorization failures:

| Condition | Description |
|---|---|
| `mayCreateObject` is `false` | The account-level capability denies object creation. |
| Caller is not a member of the target region | The user must have access to the region to place objects in it. For `"invite"` regions, this means explicit invitation; for `"space"` regions, Space membership. |

**`notFound`** -- referenced entities do not exist:

| Condition | Description |
|---|---|
| `regionId` does not exist or caller lacks access | The server returns `notFound` (not `forbidden`) per the enumeration-safe access-control rule. |
| `parentId` does not reference an existing SceneObject | The parent object must exist (within the same region). |
| `visualRef` does not reference a valid blob | The blobId must have been previously uploaded via the JMAP upload endpoint. |

**`overQuota`** -- resource limits:

| Condition | Description |
|---|---|
| Creating the object would exceed `maxObjectsPerRegion` | The per-region object cap has been reached. |

Example error response -- notFound (regionId does not exist):

~~~json
[["SceneObject/set", {
  "accountId": "account-xyz",
  "oldState": "state-77",
  "newState": "state-77",
  "created": null,
  "updated": null,
  "destroyed": null,
  "notCreated": {
    "o0": {
      "type": "notFound",
      "description": "regionId does not reference an accessible SceneRegion."
    }
  },
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

##### Update Errors

**`invalidArguments`**:

Same field-level validations as create: non-finite position/orientation, invalid `physicsMode`, `visualRef`/`visualType` mismatch, circular `parentId`, excess hierarchy depth.

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller is not the object owner, region owner, or admin | Only the object's `ownerId`, the region owner, or a deployment-admin may modify another user's object. |

**`notFound`**:

| Condition | Description |
|---|---|
| The object id does not exist or caller lacks access to the containing region | Same enumeration-safe rule. |

Example error response -- forbidden (non-owner modifying another user's object):

~~~json
[["SceneObject/set", {
  "accountId": "account-xyz",
  "oldState": "state-78",
  "newState": "state-78",
  "created": null,
  "updated": null,
  "destroyed": null,
  "notCreated": null,
  "notUpdated": {
    "01J5OBJ0000000000000000001": {
      "type": "forbidden",
      "description": "Caller is not the object owner, region owner, or an administrator."
    }
  },
  "notDestroyed": null
}, "0"]]
~~~

##### Destroy Errors

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller is not the object owner, region owner, or admin | Same permission model as update. Destroy cascades to children. |

**`notFound`**:

| Condition | Description |
|---|---|
| The object id does not exist or caller lacks access | Same enumeration-safe rule. |

### SceneObject/query {#scene-object-query}

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

This method supports spatial query filters that enable clients to retrieve objects based on position.

#### Filter Properties

`regionId` (Id, required):
: The region to query. Servers MUST return `invalidArguments` when this property is absent.

`name` (String, optional):
: Full-text search over object name. Servers that do not support full-text search MUST return `unsupportedFilter`.

`visualType` (String, optional):
: Exact match on the visual media type (e.g., `"model/gltf-binary"`).

`ownerId` (Id, optional):
: Filter to objects owned by this userId.

`interactable` (Boolean, optional):
: When `true`, filter to objects with `interactable: true`. When `false`, filter to non-interactable objects.

`visible` (Boolean, optional):
: Filter to visible or invisible objects.

`physicsMode` (String, optional):
: Filter by physics mode. One of `"static"`, `"dynamic"`, `"kinematic"`, or `"none"`.

`parentId` (Id|null, optional):
: Filter by parent object. When set to a valid Id, returns only objects whose `parentId` matches. When set to `null`, returns only root-level objects (objects with no parent).

#### Spatial Filters {#spatial-filters}

Servers MUST support the following spatial filter properties. These are the mandatory-to-implement baseline for spatial queries.

`withinRadius` (Object, optional):
: Spatial proximity filter. Filter to objects whose position falls within a sphere. Properties:
  - `center` (Number[3], required): Center point `[x, y, z]` in region-local coordinates.
  - `radius` (Number, required): Radius in meters. MUST be positive.

`withinBounds` (Object, optional):
: Spatial bounding-box filter. Filter to objects whose position falls within an axis-aligned bounding box. Properties:
  - `min` (Number[3], required): Minimum corner `[x, y, z]`.
  - `max` (Number[3], required): Maximum corner `[x, y, z]`. Each component MUST be greater than or equal to the corresponding component of `min`.

#### Filter Validation

All filter properties are combined with logical AND. An object must match every specified filter property to appear in the result set.

If `regionId` is absent, the server MUST return `invalidArguments`. If `physicsMode` contains a value not in the defined set, the server MUST return `invalidArguments`. If `withinRadius.radius` is zero or negative, the server MUST return `invalidArguments`.

Sort properties: `createdAt`, `name`. Default sort: `createdAt` ascending.

Example query -- all interactable objects within 20 meters of a point:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/query", {
      "accountId": "acct-001",
      "filter": {
        "regionId": "01J5ABC0000000000000000001",
        "interactable": true,
        "withinRadius": {
          "center": [5.0, 0, -2.0],
          "radius": 20
        }
      },
      "sort": [
        {"property": "name", "isAscending": true}
      ],
      "position": 0,
      "limit": 25
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/query", {
      "accountId": "acct-001",
      "queryState": "qstate-obj-18",
      "canCalculateChanges": true,
      "position": 0,
      "ids": [
        "01J5OBJ0000000000000000002",
        "01J5OBJ0000000000000000001"
      ],
      "total": 2
    }, "o0"]
  ]
}
~~~

Example query -- objects within an axis-aligned bounding box:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneObject/query", {
      "accountId": "acct-001",
      "filter": {
        "regionId": "01J5ABC0000000000000000001",
        "withinBounds": {
          "min": [-15, 0, -15],
          "max": [15, 10, 15]
        }
      },
      "sort": [
        {"property": "createdAt", "isAscending": true}
      ],
      "position": 0,
      "limit": 50
    }, "o0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneObject/query", {
      "accountId": "acct-001",
      "queryState": "qstate-obj-18",
      "canCalculateChanges": true,
      "position": 0,
      "ids": [
        "01J5OBJ0000000000000000001",
        "01J5OBJ0000000000000000002"
      ],
      "total": 2
    }, "o0"]
  ]
}
~~~

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

Example create request with full JMAP envelope:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "create": {
        "av0": {
          "regionId": "01J5ABC0000000000000000001",
          "displayName": "Alice Chen",
          "visualRef": "blob-avatar-alice-001",
          "visualType": "model/gltf-binary"
        }
      }
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "oldState": "state-avatar-20",
      "newState": "state-avatar-21",
      "created": {
        "av0": {
          "id": "user:alice@example.com",
          "regionId": "01J5ABC0000000000000000001",
          "userId": "user:alice@example.com",
          "displayName": "Alice Chen",
          "position": [0, 0, 10],
          "orientation": [0, 0, 0, 1],
          "visualRef": "blob-avatar-alice-001",
          "visualType": "model/gltf-binary",
          "joinedAt": "2026-06-06T10:30:00Z",
          "leftAt": null,
          "customProperties": null
        }
      },
      "updated": null,
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "a0"]
  ]
}
~~~

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

Example update -- teleport to a bookmark position:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "update": {
        "user:alice@example.com": {
          "position": [25.0, 0, -12.3],
          "orientation": [0, 0.924, 0, 0.383]
        }
      }
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "oldState": "state-avatar-21",
      "newState": "state-avatar-22",
      "created": null,
      "updated": {
        "user:alice@example.com": null
      },
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "a0"]
  ]
}
~~~

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

Example leave request with full JMAP envelope:

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "update": {
        "user:alice@example.com": {
          "leftAt": "2026-06-06T11:45:00Z"
        }
      }
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAvatar/set", {
      "accountId": "acct-001",
      "oldState": "state-avatar-22",
      "newState": "state-avatar-23",
      "created": null,
      "updated": {
        "user:alice@example.com": null
      },
      "destroyed": null,
      "notCreated": null,
      "notUpdated": null,
      "notDestroyed": null
    }, "a0"]
  ]
}
~~~

#### Reconnecting

When a user who has left (non-null `leftAt`) re-enters the same region, the server MUST update the existing SceneAvatar record rather than creating a new one: clear `leftAt` to `null`, update `joinedAt` to the current time, and set `position` to `spawnPosition`. This ensures a single continuous identity.

#### SetError Conditions

##### Create Errors (Entering a Region)

**`invalidArguments`** -- missing or malformed fields:

| Condition | Description |
|---|---|
| `regionId` is missing | `regionId` is required to specify which region to enter. |
| `position` contains non-finite values (if explicitly supplied) | Position, when supplied by the client (e.g., for teleport-on-entry), must contain finite Numbers. Normally the server sets position from `spawnPosition`. |
| `visualRef` is present but `visualType` is absent (or vice versa) | Both must be present together or both absent. |

**`forbidden`** -- access denied:

| Condition | Description |
|---|---|
| Region `accessPolicy` is `"invite"` and user has not been granted access | The invitation mechanism is deployment-defined; without an invitation, entry is denied. |
| Region `accessPolicy` is `"space"` and user is not a member of the bound Space | Space membership is checked when JMAP Chat is present with `spaceId` set. |
| User is banned or ejected with a cooldown period still active | Deployment-defined ban/eject policies may temporarily or permanently deny entry. |
| Caller is creating an avatar record for a different user | Users MUST only create their own avatar. The server sets `userId` from the authenticated identity. |

**`notFound`** -- region does not exist:

| Condition | Description |
|---|---|
| `regionId` does not exist or caller lacks access | Same enumeration-safe access-control rule. |

**`overQuota`** -- region capacity:

| Condition | Description |
|---|---|
| `maxAvatarsPerRegion` would be exceeded | The region has reached its concurrent-avatar capacity. |

Example error response -- forbidden (invite-only region):

~~~json
[["SceneAvatar/set", {
  "accountId": "account-xyz",
  "oldState": "state-12",
  "newState": "state-12",
  "created": null,
  "updated": null,
  "destroyed": null,
  "notCreated": {
    "av0": {
      "type": "forbidden",
      "description": "Region accessPolicy is \"invite\" and the caller has not been granted access."
    }
  },
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

**One-avatar-per-region constraint:**

When a user already has an active SceneAvatar (`leftAt: null`) in the same region, the server MUST NOT create a duplicate. Instead, the server MUST return the existing record in the `updated` map. This is not a SetError -- the server silently handles the idempotency by acknowledging the existing presence. Clients receive a successful response with the existing avatar's current state.

When the user has an active avatar in a different region, the server MUST set `leftAt` on the previous avatar (auto-eject) before creating the new one, enforcing the one-region-at-a-time constraint. This is also not a SetError; it is implicit auto-eject behavior.

Servers that prefer to reject rather than auto-eject MAY return an `invalidArguments` SetError with a description indicating the user must leave their current region first. The spec recommends the auto-eject approach.

##### Update Errors

**`invalidArguments`**:

| Condition | Description |
|---|---|
| Setting `leftAt` on an avatar that has already left (`leftAt` is non-null) | The user has already departed; double-leave is invalid. |
| `position` or `orientation` contains non-finite values | Must be finite Numbers. |

**`forbidden`**:

| Condition | Description |
|---|---|
| Caller's userId does not match the SceneAvatar's userId | Users MUST NOT update other users' avatars. Exception: region owners and administrators may set `leftAt` on another user's avatar to eject them. |

**`notFound`**:

| Condition | Description |
|---|---|
| The avatar id does not exist or caller lacks access to the containing region | Same enumeration-safe rule. |

##### Destroy Errors

SceneAvatar records are not destroyed via `/set destroy`. Departure is modeled as an update that sets `leftAt`. Servers MUST return `forbidden` for destroy operations on SceneAvatar.

### SceneAvatar/query

Standard JMAP `/query` ({{RFC8620}} Section 5.5).

#### Filter Properties

`regionId` (Id, required):
: The region to query. Servers MUST return `invalidArguments` when this property is absent.

`userId` (Id, optional):
: Filter to avatars belonging to this user account.

`isActive` (Boolean, optional):
: When `true`, filter to avatars with `leftAt == null` (currently present in the region). When `false`, filter to avatars who have left (`leftAt` is non-null).

Spatial filters:

`withinRadius` (Object, optional):
: Spatial proximity filter. Same syntax as SceneObject/query ({{spatial-filters}}): filter to avatars whose last known position falls within a sphere. Properties:
  - `center` (Number[3], required): Center point `[x, y, z]` in region-local coordinates.
  - `radius` (Number, required): Radius in meters. MUST be positive.

`withinBounds` (Object, optional):
: Same syntax as SceneObject/query ({{spatial-filters}}).

#### Filter Validation

All filter properties are combined with logical AND. An avatar must match every specified filter property to appear in the result set.

If `regionId` is absent, the server MUST return `invalidArguments`. If `withinRadius.radius` is zero or negative, the server MUST return `invalidArguments`. The query MUST only return avatars in regions the authenticated user has access to.

Sort properties: `joinedAt`, `displayName`. Default sort: `joinedAt` ascending.

Example query -- active avatars within 30 meters of a point (proximity query):

~~~json
{
  "using": ["urn:ietf:params:jmap:core", "urn:ietf:params:jmap:scene"],
  "methodCalls": [
    ["SceneAvatar/query", {
      "accountId": "acct-001",
      "filter": {
        "regionId": "01J5ABC0000000000000000001",
        "isActive": true,
        "withinRadius": {
          "center": [10.0, 0, -5.0],
          "radius": 30
        }
      },
      "sort": [
        {"property": "joinedAt", "isAscending": true}
      ],
      "position": 0,
      "limit": 50
    }, "a0"]
  ]
}
~~~

Response:

~~~json
{
  "methodResponses": [
    ["SceneAvatar/query", {
      "accountId": "acct-001",
      "queryState": "qstate-avatar-9",
      "canCalculateChanges": true,
      "position": 0,
      "ids": [
        "user:bob@example.com",
        "user:carol@example.com"
      ],
      "total": 2
    }, "a0"]
  ]
}
~~~

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

When a client attempts to interact with an object that is no longer within the client's visibility set (e.g., the object moved out of range, or the avatar moved away), the server MAY return `notFound` for JMAP method calls (`SceneObject/set`, `SceneObject/get`) targeting that object, or MAY silently discard the interaction with no error response. Both behaviors are valid. The server MUST NOT return `forbidden`, as that would confirm the object still exists. For interactions delivered via the simulation layer, the simulation layer MAY silently ignore stale targets without generating any JMAP-level error.

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

When a SceneRegion's `activeCallId` references a VTCCall with `e2eeEnabled` set to `true`, the server SHOULD suppress `SceneAvatarEvent` delivery (enter, leave, and update events) for that region to subscribers who are not participants in the bound VTCCall. This prevents the Scene WebSocket channel from leaking real-time call participant presence metadata that E2EE is designed to protect. Servers that cannot implement this suppression MUST document the limitation.

## Object Density as Denial of Service

Without limits, a malicious user could create thousands of objects in a region, overwhelming clients that attempt to render them all. Servers MUST enforce `maxObjectsPerRegion`. Servers SHOULD impose per-user object creation rate limits.

## Visibility and Information Leakage

SceneObject data returned to a client reveals information about the scene (object names, positions, custom properties). Deployments where scene content is sensitive (e.g., a competitive game with hidden objects) SHOULD use the visibility filtering described in {{visibility-contract}} to limit what data reaches each client.

## Blocked-Contact Spatial Presence {#blocked-contact-spatial}

When `urn:ietf:params:jmap:chat` ({{JMAP-CHAT}}) is co-deployed with `urn:ietf:params:jmap:scene`, the server has access to the blocking user's ChatContact records and their `blocked` field. The server SHOULD use this information to suppress spatial presence of blocked users:

- SceneAvatar records belonging to a blocked ChatContact SHOULD be excluded from `SceneAvatar/get` responses, `SceneAvatar/query` results, and `SceneAvatar/queryChanges` notifications delivered to the blocking user. The blocked user's avatar still exists in the region; it is simply not visible to the blocking user's client.

- The blocked user MUST NOT learn that they have been blocked from the absence of spatial data. The server MUST NOT return an error, a filtered-result indicator, or any other signal that distinguishes "filtered because blocked" from "not present." From the blocked user's perspective, the blocking user's query behavior is indistinguishable from normal operation.

- Blocked-contact filtering is applied AFTER the visibility filtering described in {{visibility-contract}}. The server first computes the visibility set for the requesting user's position and access level, then removes any SceneAvatar records whose `userId` corresponds to a ChatContact with `blocked: true` on the requesting user's contact list.

- The `activeAvatarCount` field on SceneRegion is a server-set aggregate. Servers MAY choose to return the true count (which includes blocked users) or a filtered count that excludes blocked users. Either approach leaks some information: a true count reveals the presence of unseen users; a filtered count that changes when a block is applied reveals the act of blocking. Deployments SHOULD document which behavior they implement and SHOULD prefer the true count, since it is consistent for all observers and does not leak per-user blocking decisions.

Without `urn:ietf:params:jmap:chat`, there is no blocked-contact list and this filtering does not apply. The Scene capability alone has no concept of blocked users.

## Content Moderation for User-Created Objects {#content-moderation}

SceneObject visual assets (`visualRef`) and SceneAvatar visual assets (`visualRef`) are user-supplied and may contain inappropriate content: offensive 3D models, textures with prohibited imagery, or audio assets with harmful material. Servers SHOULD validate visual assets against deployment-defined content policies before making those assets visible to other users.

Servers MAY defer the visibility of newly created SceneObject records until asset scanning completes. During the scanning interval, the object exists in the JMAP state (the creator can retrieve it via `SceneObject/get`), but the server SHOULD exclude it from `SceneObject/query` results and `SceneObject/queryChanges` notifications delivered to other users. Once scanning completes and the asset passes the content policy, the object becomes visible through normal query and change mechanisms. If the asset fails the content policy, the server SHOULD destroy the object and SHOULD notify the creator via a `SetError` of type `contentPolicy` on a subsequent state-change notification or, if the deployment supports it, via an out-of-band moderation channel.

Text fields on SceneObject — `name`, `customProperties`, and any string values within `customProperties` — are user-controlled and may contain spam, offensive language, or phishing content. Servers SHOULD apply the same content-policy validation to text fields as to visual assets. Servers MAY reject `SceneObject/set` create or update requests that violate text content policies with a `SetError` of type `contentPolicy`.

Rapid `SceneObject/set` create calls can be used as a spam or denial-of-service vector, flooding a region with objects faster than content scanning can process them. Servers SHOULD enforce per-user rate limits on `SceneObject/set` create operations, independent of the `maxObjectsPerRegion` cap. When a rate limit is exceeded, the server SHOULD return a `SetError` of type `rateLimit` and SHOULD include a `retryAfter` property indicating the number of seconds before the client may retry.

## Avatar Identity and Impersonation {#avatar-impersonation}

`SceneAvatar.displayName` is user-controlled and may be set to impersonate other users, system entities, or administrative roles (e.g., "System Administrator", "Moderator", or another user's real name). This creates a social-engineering attack surface in multi-user spatial environments.

Servers SHOULD enforce at least one of the following mitigations to prevent display-name impersonation:

- **Uniqueness constraints.** The server MAY enforce that no two active SceneAvatar records within the same SceneRegion share the same `displayName`. When a collision is detected, the server SHOULD reject the later `SceneAvatar/set` create with a `SetError` of type `invalidArguments` and a description indicating the name conflict. Alternatively, the server MAY append a disambiguating suffix (e.g., a numeric tag) to the duplicate name.

- **Visual differentiation.** The server MAY provide system-assigned badges, name colors, or other visual metadata (via `customProperties` or a deployment-defined mechanism) that distinguish authenticated identity from user-chosen display names. Clients SHOULD render these differentiators prominently.

- **Reserved name lists.** The server SHOULD maintain a list of reserved display names that correspond to system roles or administrative functions and MUST reject `SceneAvatar/set` operations that attempt to use a reserved name with a `SetError` of type `forbidden`.

In deployments where `urn:ietf:params:jmap:chat` ({{JMAP-CHAT}}) is co-deployed, the server MAY enforce that `SceneAvatar.displayName` matches the user's ChatContact display name. This provides a single authoritative source for display names across chat and spatial contexts, reducing the impersonation surface. When this enforcement is active, `SceneAvatar/set` operations that attempt to set a `displayName` different from the ChatContact display name SHOULD be rejected with a `SetError` of type `invalidArguments`.

Avatar visual assets (`visualRef`) present a separate impersonation vector: a user may upload a 3D model designed to visually replicate another user's avatar. Servers MAY restrict avatar visual references to a deployment-approved set of assets, or MAY apply perceptual similarity checks against other users' registered avatar assets. These mitigations are deployment-defined; this specification does not prescribe a specific visual-similarity algorithm.

# Push Notifications {#push}

Servers MUST support the EventSource mechanism defined in {{RFC8620}} Section 7.3.

Servers SHOULD also support the push subscription mechanism defined in {{RFC8620}} Section 7.2 for deployments requiring offline and mobile push delivery.

Servers that deliver push subscriptions via Web Push SHOULD also advertise the `urn:ietf:params:jmap:webpush-vapid` capability and authenticate their Web Push messages with a VAPID-signed JWT, consistent with the recommendation in {{JMAP-CHAT}} when that capability is co-deployed.

## State-Change Events {#push-state-change}

All four Scene data types -- SceneRegion, SceneObject, SceneAvatar, and SceneAsset -- participate in the RFC 8620 state-change mechanism ({{RFC8620}} Section 7.1). The IANA registrations in {{iana}} declare "Can use for state change: Yes" for each type.

The server MUST include a state string for each Scene data type in the `StateChange` object delivered to subscribed clients whenever the corresponding data changes. State strings are opaque to clients; a state string changes on any mutation (create, update, or destroy) to any record of that type within the account. Clients MUST NOT parse or interpret state string values; they are meaningful only as arguments to the corresponding `/changes` method.

Example `StateChange` event via EventSource:

~~~
event: state
data: {"@type":"StateChange","changed":{"account-xyz":{"SceneRegion":"r41a","SceneObject":"o88f2","SceneAvatar":"av03c","SceneAsset":"as7e1"}}}
~~~

A single `StateChange` MAY include state strings for a subset of the four types. The server MUST include a type's state string only when that type has changed since the last event delivered to that subscriber. If only SceneAvatar records changed (a user entered or left a region), the server MAY omit the other three types from the payload:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "account-xyz": {
      "SceneAvatar": "av04d"
    }
  }
}
~~~

When JMAP Chat ({{JMAP-CHAT}}) is co-deployed, a single `StateChange` MAY carry state strings for both Chat and Scene data types:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "account-xyz": {
      "Message": "m99x",
      "Chat": "c12y",
      "SceneRegion": "r41a",
      "SceneAvatar": "av04d"
    }
  }
}
~~~

Clients SHOULD call the corresponding `/changes` method for each type whose state string differs from the locally cached value. On `cannotCalculateChanges`, fall back to `/get`.

### State-Change Frequency

Scene data types differ significantly in mutation frequency:

- **SceneAvatar** changes frequently: avatar enter, leave, and periodic position reconciliation from the simulation layer all produce mutations. The server SHOULD coalesce rapid SceneAvatar mutations within a deployment-defined window (RECOMMENDED: 1-5 seconds) before emitting a `StateChange`, to avoid overwhelming subscribers with per-avatar state changes during high-traffic periods.

- **SceneObject** changes at moderate frequency: object creation, destruction, and position/property updates. The server SHOULD coalesce rapid SceneObject mutations similarly.

- **SceneRegion** and **SceneAsset** change infrequently: region creation, configuration updates, and asset uploads are low-frequency operations. Coalescing is generally unnecessary but remains at the server's discretion.

Real-time avatar and object position synchronization at simulation-layer rates (10+ Hz) MUST NOT generate `StateChange` events at that frequency. The periodic position reconciliation described in {{simulation-layer}} produces JMAP-layer mutations at a much lower rate (every 5-30 seconds); only those reconciliation writes produce state changes. Clients connected to the simulation layer receive real-time positions through the simulation protocol; `StateChange` events for SceneAvatar and SceneObject serve clients that are not connected to the simulation layer or that need to detect structural changes (new avatars, removed objects).

When Scene and Chat are co-deployed, the server SHOULD coalesce `StateChange` events across both capabilities to avoid overwhelming Chat-only clients. A `StateChange` containing only SceneAvatar or SceneObject state tokens SHOULD be coalesced within a 1-5 second window. Clients that do not use Scene types SHOULD ignore state tokens for types they do not track, avoiding unnecessary `/changes` calls.

## Push Urgency {#push-urgency}

When delivering `StateChange` notifications via Web Push, the server SHOULD set the `Urgency` header based on the nature of the change. The following urgency assignments are RECOMMENDED:

`high`:
: Region destruction (`SceneRegion/set destroy`) when the receiving user has SceneObject records in the destroyed region. The user's objects have been removed as a side effect and the user should be informed promptly.

`normal`:
: Avatar enter and leave events. A user MAY want to know when someone enters or leaves a region they are present in, particularly for invite-only or low-population regions. Region creation and configuration changes also warrant `normal` urgency when the user is an active participant in the region.

`low`:
: Object destruction by an administrator (region owner or deployment admin destroying another user's object). This is informational: the owner of the destroyed object should eventually learn that their object was removed, but the notification is not time-sensitive.

`very-low`:
: SceneAsset changes. Asset metadata updates are administrative bookkeeping and rarely require user attention. SceneObject position and property updates that result from periodic simulation-layer reconciliation also fall into this category; the user is typically connected to the simulation layer and already aware of these changes.

Servers MUST NOT assign `high` urgency to routine position reconciliation updates. A deployment that reconciles avatar positions every 5 seconds and assigns `high` urgency to each reconciliation would rapidly exhaust push infrastructure quotas and trigger OS-level throttling on mobile clients.

When a single `StateChange` event covers multiple data types with different recommended urgencies, the server SHOULD use the highest applicable urgency for the combined payload.

## PushSubscription Filtering {#push-filtering}

Clients create push subscriptions using `PushSubscription/set` as defined in {{RFC8620}} Section 7.2. The `types` property on `PushSubscription` controls which data types generate push notifications for that subscription.

A client that wishes to receive push notifications only for Scene data types SHOULD set `types` to include the desired subset:

~~~json
[["PushSubscription/set", {
  "create": {
    "ps0": {
      "url": "https://push.example.com/endpoint/scene-abc123",
      "types": ["SceneRegion", "SceneAvatar"],
      "keys": {
        "p256dh": "BLc4xRW...",
        "auth": "mQ5_Kg..."
      }
    }
  }
}, "0"]]
~~~

In this example, the subscription receives `StateChange` notifications only when SceneRegion or SceneAvatar records change. SceneObject and SceneAsset changes are excluded. This is useful for a client that monitors region occupancy without tracking individual object mutations.

A client that wishes to receive notifications for all Scene types sets `types` to `["SceneRegion", "SceneObject", "SceneAvatar", "SceneAsset"]`, or omits `types` entirely (which subscribes to all types the server supports, including non-Scene types if other capabilities are present).

When both Scene and Chat capabilities are present, a single `PushSubscription` MAY include types from both capabilities:

~~~json
[["PushSubscription/set", {
  "create": {
    "ps1": {
      "url": "https://push.example.com/endpoint/unified-abc456",
      "types": [
        "Message", "Chat",
        "SceneRegion", "SceneAvatar", "SceneObject"
      ],
      "keys": {
        "p256dh": "BNpR3x...",
        "auth": "kP7_Rw..."
      }
    }
  }
}, "0"]]
~~~

## Interaction with Chat Push {#push-chat-interaction}

When `urn:ietf:params:jmap:scene` and `urn:ietf:params:jmap:chat` are both advertised on the same account, certain user-visible events may be expressible as both a Scene state change and a Chat state change. The server SHOULD deduplicate cross-capability push notifications for the same logical event to avoid redundant notifications to the user.

### Avatar Enter and Chat Member Join

When a SceneRegion is bound to a Chat via the `chatId` field (see {{chat-binding}}), a user entering the region (SceneAvatar creation) and the corresponding "user joined" event in the bound Chat represent the same logical event: a user has become present. The server MUST still update state for both data types (SceneAvatar and Chat/Message), because clients tracking each type independently require accurate state strings. However, the server SHOULD NOT generate user-facing push notifications for both events when they are consequences of the same user action.

Specifically:

- If the Chat capability generates a push notification for a "user joined" system message in the bound Chat, the server SHOULD suppress any additional user-facing notification derived from the corresponding SceneAvatar creation. The `StateChange` for SceneAvatar MUST still be delivered (it is a state-tracking mechanism, not a user-facing notification), but the server SHOULD NOT set `Urgency: normal` or higher on the SceneAvatar `StateChange` when a Chat push for the same event has already been delivered or is being delivered concurrently.

- If the Chat capability does not generate a notification for the join event (because the Chat is muted, or the Chat push subscription does not cover that Chat), the Scene `StateChange` for SceneAvatar proceeds with its normal urgency.

### Avatar Leave and Chat Member Departure

The same deduplication logic applies to avatar departure. When a user leaves a region (SceneAvatar `leftAt` set to non-null) and a corresponding "user left" system message is posted to the bound Chat, the server SHOULD treat these as a single logical event for push deduplication purposes.

### Independent Events

Not all Scene events have Chat analogs. The following Scene events have no Chat counterpart and MUST generate push notifications independently:

- SceneObject creation, modification, or destruction (no Chat equivalent).
- SceneRegion creation, configuration changes, or destruction (no Chat equivalent unless a system message is posted to the bound Chat, which is deployment-defined).
- SceneAsset changes (no Chat equivalent).
- SceneAvatar position reconciliation updates (no Chat equivalent; these are `very-low` urgency regardless).

### Server Implementation

The deduplication requirement is a SHOULD, not a MUST, because the server is in the best position to determine whether two state changes represent the same logical event. Servers that do not implement cross-capability deduplication will produce functionally correct but potentially redundant notifications. Clients SHOULD be prepared to receive notifications from both capabilities for the same logical event and SHOULD deduplicate on the client side when possible (for example, by suppressing a "user entered region" notification if a "user joined chat" notification for the same user and bound Chat was received within the preceding 5 seconds).

## Scene Push Does Not Carry Inline Content

Unlike JMAP Chat Push, which defines a `ChatMessagePush` payload carrying inline message content to avoid a mobile round-trip, Scene push notifications use the standard `StateChange` mechanism exclusively. This design choice reflects a difference in operational constraints:

- Chat messages are time-sensitive user communications that mobile platforms must display immediately. The round-trip cost of `StateChange` followed by `/changes` followed by `/get` routinely exceeds mobile background execution budgets.

- Scene state changes are structural metadata (who is present, what objects exist, where they are). Clients that need real-time spatial awareness are connected to the simulation layer and do not rely on push notifications for that data. Clients that are not connected to the simulation layer (background or idle clients) can tolerate the `/changes` round-trip because there is no mobile-platform penalty for a brief delay in learning that a scene object moved.

If a future deployment identifies a need for inline Scene push payloads (for example, a notification that says "Alice entered Gallery East Wing" without a round-trip), a companion specification may define a `SceneEventPush` payload. This specification does not define one.

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
- **Additional visual asset formats.** glTF 2.0 is mandatory-to-implement (see {{preferred-asset-format}}), but deployments may support gaussian splats, neural radiance fields, voxels, USD, or any future format via the `visualType` field and `supportedVisualTypes` capability.
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
- **Color and color space.** The `environment` object is opaque; any color values within it are deployment-defined. Visual assets referenced by `visualRef` carry their own color data in whatever format and color space the asset format specifies. Where color values appear in deployment-defined fields (e.g., `environment.skyColor`, `environment.fogColor`), deployments SHOULD follow the color representation convention defined in {{JMAP-CHAT}} ({{color-convention}}): W3C Design Tokens format preferred, CAM16 as fallback, sRGB hex as baseline.

# Complete Lifecycle Examples {#lifecycle-examples}

## Avatar Session: Enter, Interact, Move, Leave

This example shows a complete avatar session where Alice discovers the Scene capability, finds a region, enters it, interacts with an object, moves through the space, and leaves.

### Step 1: Client discovers Scene capability

Alice's client fetches the JMAP Session object and finds the Scene capability advertised at both the session and account levels:

~~~json
{
  "capabilities": {
    "urn:ietf:params:jmap:core": {},
    "urn:ietf:params:jmap:scene": {}
  },
  "accounts": {
    "alice-account": {
      "name": "alice@example.com",
      "accountCapabilities": {
        "urn:ietf:params:jmap:scene": {
          "mayCreateRegion": false,
          "mayCreateObject": true,
          "supportedVisualTypes": [
            "model/gltf-binary",
            "image/png"
          ],
          "maxRegionsPerAccount": 10,
          "maxObjectsPerRegion": 5000,
          "maxAvatarsPerRegion": 100,
          "maxAssetSizeBytes": 52428800
        }
      }
    }
  },
  "apiUrl": "https://jmap.example.com/api/",
  "uploadUrl": "https://jmap.example.com/upload/{accountId}/",
  "downloadUrl": "https://jmap.example.com/download/{accountId}/{blobId}/{name}"
}
~~~

Alice's client sees `urn:ietf:params:jmap:scene` is present. She can create objects but not regions. The server supports glTF binary and PNG assets, and allows up to 100 concurrent avatars per region.

### Step 2: Client queries available regions

Alice's client sends `SceneRegion/query` to find public regions with active users, then fetches the results with `SceneRegion/get` using a back-reference:

~~~json
[["SceneRegion/query", {
  "accountId": "alice-account",
  "filter": {
    "accessPolicy": "public",
    "hasActiveAvatars": true
  },
  "sort": [{"property": "activeAvatarCount", "isAscending": false}],
  "limit": 10
}, "0"],
["SceneRegion/get", {
  "accountId": "alice-account",
  "#ids": {
    "resultOf": "0",
    "name": "SceneRegion/query",
    "path": "/ids"
  }
}, "1"]]
~~~

Server responds:

~~~json
[["SceneRegion/query", {
  "accountId": "alice-account",
  "queryState": "s100",
  "canCalculateChanges": true,
  "position": 0,
  "total": 2,
  "ids": [
    "01JXKR5M0G3QVTA8N2BWFP7Y01",
    "01JXKR5M0G3QVTA8N2BWFP7Y02"
  ]
}, "0"],
["SceneRegion/get", {
  "accountId": "alice-account",
  "state": "s101",
  "list": [
    {
      "id": "01JXKR5M0G3QVTA8N2BWFP7Y01",
      "accountId": "alice-account",
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
      "simulationUri": "wss://sim.example.com/regions/01JXKR5M",
      "viewHint": "3d",
      "spawnPosition": [0, 0, 10],
      "spawnOrientation": [0, 0, 0, 1],
      "activeAvatarCount": 3,
      "accessPolicy": "public",
      "createdAt": "2026-05-01T10:00:00Z",
      "updatedAt": "2026-06-05T14:30:00Z",
      "chatId": null,
      "spaceId": null,
      "channelId": null,
      "activeCallId": null
    },
    {
      "id": "01JXKR5M0G3QVTA8N2BWFP7Y02",
      "accountId": "alice-account",
      "name": "Rooftop Lounge",
      "description": "Social hangout with city skyline",
      "bounds": {
        "min": [-30, 0, -30],
        "max": [30, 5, 30]
      },
      "environment": {
        "skyColor": "#1a1a2e",
        "ambientIntensity": 0.3,
        "gravity": 9.81
      },
      "simulationUri": "wss://sim.example.com/regions/01JXKR5N",
      "viewHint": "3d",
      "spawnPosition": [0, 0, 0],
      "spawnOrientation": [0, 0, 0, 1],
      "activeAvatarCount": 1,
      "accessPolicy": "public",
      "createdAt": "2026-05-15T08:00:00Z",
      "updatedAt": "2026-06-04T22:10:00Z",
      "chatId": null,
      "spaceId": null,
      "channelId": null,
      "activeCallId": null
    }
  ],
  "notFound": []
}, "1"]]
~~~

Alice sees "Gallery East Wing" has 3 active avatars and decides to enter it.

### Step 3: Client enters a region

Alice's client sends `SceneAvatar/set` with a `create` to enter the Gallery East Wing region:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "create": {
    "av0": {
      "regionId": "01JXKR5M0G3QVTA8N2BWFP7Y01",
      "visualRef": "blob-avatar-alice-001",
      "visualType": "model/gltf-binary",
      "customProperties": {
        "animation": "idle",
        "nametag": true
      }
    }
  }
}, "0"]]
~~~

Server responds with the created avatar. The server assigns the id from Alice's userId, sets `displayName` from her user profile, sets `position` and `orientation` from the region's spawn point, and sets `joinedAt` to the current time:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "oldState": "s200",
  "newState": "s201",
  "created": {
    "av0": {
      "id": "user:alice@example.com",
      "regionId": "01JXKR5M0G3QVTA8N2BWFP7Y01",
      "userId": "user:alice@example.com",
      "displayName": "Alice Chen",
      "position": [0, 0, 10],
      "orientation": [0, 0, 0, 1],
      "visualRef": "blob-avatar-alice-001",
      "visualType": "model/gltf-binary",
      "joinedAt": "2026-06-06T09:15:00Z",
      "leftAt": null,
      "customProperties": {
        "animation": "idle",
        "nametag": true
      }
    }
  },
  "updated": null,
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

The server increments the region's `activeAvatarCount` to 4. Alice's client connects to `simulationUri` (`wss://sim.example.com/regions/01JXKR5M`) to begin receiving real-time position updates from other avatars.

### Step 4: Client interacts with an object

Alice walks up to an interactable sculpture and clicks on it. Her client sends a `SceneInteractionEvent` over the WebSocket connection:

~~~json
{
  "@type": "SceneInteractionEvent",
  "regionId": "01JXKR5M0G3QVTA8N2BWFP7Y01",
  "objectId": "01JXKR6P4HNTQW29DCEAGM8V05",
  "userId": "user:alice@example.com",
  "action": "click",
  "data": null
}
~~~

The server fans out this event to all other avatars in the region whose visibility set includes the sculpture. Other clients may display a click animation or info panel. The interaction is purely ephemeral and is not persisted.

### Step 5: Client updates avatar position

Alice teleports to a bookmark position near the back of the gallery. Her client sends `SceneAvatar/set` with a position patch:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "update": {
    "user:alice@example.com": {
      "position": [35.0, 0, -22.5],
      "orientation": [0, -0.383, 0, 0.924]
    }
  }
}, "0"]]
~~~

Server responds:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "oldState": "s201",
  "newState": "s202",
  "created": null,
  "updated": {
    "user:alice@example.com": null
  },
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

This is a JMAP-level teleport. Continuous position updates during normal movement are handled by the simulation layer at 10-20 Hz, not by JMAP method calls.

### Step 6: Client leaves the region

Alice is done browsing the gallery. Her client sends `SceneAvatar/set` to set `leftAt`:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "update": {
    "user:alice@example.com": {
      "leftAt": "2026-06-06T09:47:30Z"
    }
  }
}, "0"]]
~~~

Server responds:

~~~json
[["SceneAvatar/set", {
  "accountId": "alice-account",
  "oldState": "s202",
  "newState": "s203",
  "created": null,
  "updated": {
    "user:alice@example.com": null
  },
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

The server decrements the region's `activeAvatarCount` to 3. Other clients in the region receive a `StateChange` for `SceneAvatar`:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "alice-account": {
      "SceneAvatar": "s203"
    }
  }
}
~~~

Alice's client disconnects from the simulation layer WebSocket.

## Region Administration: Create, Populate, Restrict, Eject

This example shows an administrator creating a region, populating it with objects, changing the access policy, and ejecting an avatar.

### Step 1: Admin creates a region

The admin creates a new region for a product launch event:

~~~json
[["SceneRegion/set", {
  "accountId": "admin-account",
  "create": {
    "r0": {
      "name": "Product Launch Hall",
      "description": "Annual product launch event venue",
      "bounds": {
        "min": [-100, 0, -60],
        "max": [100, 20, 60]
      },
      "viewHint": "3d",
      "accessPolicy": "public",
      "spawnPosition": [0, 0, 50],
      "spawnOrientation": [0, 0, 0, 1],
      "simulationUri": "wss://sim.example.com/regions/launch-hall",
      "environment": {
        "skyColor": "#000022",
        "ambientIntensity": 0.4,
        "gravity": 9.81,
        "fogDensity": 0.01,
        "fogColor": "#111133"
      }
    }
  }
}, "0"]]
~~~

Server responds with the created region:

~~~json
[["SceneRegion/set", {
  "accountId": "admin-account",
  "oldState": "s300",
  "newState": "s301",
  "created": {
    "r0": {
      "id": "01JXKRBN7KWMPS46GFHDJT9R10",
      "accountId": "admin-account",
      "name": "Product Launch Hall",
      "description": "Annual product launch event venue",
      "bounds": {
        "min": [-100, 0, -60],
        "max": [100, 20, 60]
      },
      "viewHint": "3d",
      "accessPolicy": "public",
      "spawnPosition": [0, 0, 50],
      "spawnOrientation": [0, 0, 0, 1],
      "simulationUri": "wss://sim.example.com/regions/launch-hall",
      "environment": {
        "skyColor": "#000022",
        "ambientIntensity": 0.4,
        "gravity": 9.81,
        "fogDensity": 0.01,
        "fogColor": "#111133"
      },
      "activeAvatarCount": 0,
      "createdAt": "2026-06-06T08:00:00Z",
      "updatedAt": "2026-06-06T08:00:00Z",
      "chatId": null,
      "spaceId": null,
      "channelId": null,
      "activeCallId": null
    }
  },
  "updated": null,
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

The region is now live and publicly accessible, but empty.

### Step 2: Admin populates region with objects

The admin places three objects in a single batch: a static back wall, an interactable entrance door, and a dynamic physics-enabled demo cube. All three are created in one `SceneObject/set` call:

~~~json
[["SceneObject/set", {
  "accountId": "admin-account",
  "create": {
    "wall": {
      "regionId": "01JXKRBN7KWMPS46GFHDJT9R10",
      "name": "Back Wall",
      "position": [0, 5, -59],
      "orientation": [0, 0, 0, 1],
      "scale": [200, 10, 1],
      "visualRef": "blob-gltf-wall-panel-001",
      "visualType": "model/gltf-binary",
      "assetUri": "https://cdn.example.com/assets/wall-panel.glb",
      "physicsMode": "static",
      "interactable": false,
      "visible": true,
      "customProperties": {
        "material": "concrete",
        "tiling": [10, 2]
      }
    },
    "door": {
      "regionId": "01JXKRBN7KWMPS46GFHDJT9R10",
      "name": "Entrance Door",
      "position": [0, 1.5, 55],
      "orientation": [0, 0, 0, 1],
      "scale": [2, 3, 0.2],
      "visualRef": "blob-gltf-door-001",
      "visualType": "model/gltf-binary",
      "assetUri": "https://cdn.example.com/assets/door-sliding.glb",
      "physicsMode": "kinematic",
      "interactable": true,
      "visible": true,
      "customProperties": {
        "animationOnActivate": "slide-open",
        "autoCloseSeconds": 5
      }
    },
    "cube": {
      "regionId": "01JXKRBN7KWMPS46GFHDJT9R10",
      "name": "Demo Physics Cube",
      "position": [15, 3, 0],
      "orientation": [0.1, 0.2, 0.05, 0.974],
      "scale": [1, 1, 1],
      "visualRef": "blob-gltf-cube-001",
      "visualType": "model/gltf-binary",
      "assetUri": "https://cdn.example.com/assets/demo-cube.glb",
      "physicsMode": "dynamic",
      "interactable": true,
      "visible": true,
      "customProperties": {
        "mass": 5.0,
        "restitution": 0.7
      }
    }
  }
}, "0"]]
~~~

Server responds with all three objects created:

~~~json
[["SceneObject/set", {
  "accountId": "admin-account",
  "oldState": "s400",
  "newState": "s401",
  "created": {
    "wall": {
      "id": "01JXKRC47NTHRZ35KBMPQW6A20",
      "ownerId": "user:admin@example.com",
      "createdAt": "2026-06-06T08:05:00Z",
      "updatedAt": "2026-06-06T08:05:00Z"
    },
    "door": {
      "id": "01JXKRC47NTHRZ35KBMPQW6A21",
      "ownerId": "user:admin@example.com",
      "createdAt": "2026-06-06T08:05:00Z",
      "updatedAt": "2026-06-06T08:05:00Z"
    },
    "cube": {
      "id": "01JXKRC47NTHRZ35KBMPQW6A22",
      "ownerId": "user:admin@example.com",
      "createdAt": "2026-06-06T08:05:00Z",
      "updatedAt": "2026-06-06T08:05:00Z"
    }
  },
  "updated": null,
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

The region now contains three objects: the wall acts as an immovable collider, the door responds to user interaction and is moved by server scripts (kinematic), and the cube is fully physics-simulated and can be pushed or thrown by avatars.

### Step 3: Admin updates access policy

The event is about to start. The admin restricts the region from public to invite-only so that only approved attendees can enter:

~~~json
[["SceneRegion/set", {
  "accountId": "admin-account",
  "update": {
    "01JXKRBN7KWMPS46GFHDJT9R10": {
      "accessPolicy": "invite",
      "description": "Annual product launch event venue (invite only)"
    }
  }
}, "0"]]
~~~

Server responds:

~~~json
[["SceneRegion/set", {
  "accountId": "admin-account",
  "oldState": "s301",
  "newState": "s302",
  "created": null,
  "updated": {
    "01JXKRBN7KWMPS46GFHDJT9R10": null
  },
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

From this point forward, users without an explicit invitation receive `forbidden` when attempting `SceneAvatar/set` create for this region. Users already present are not ejected by the policy change; they remain until they leave or are explicitly ejected.

### Step 4: Admin ejects an avatar

A disruptive user (`user:troll@example.com`) is in the region. The admin ejects them by setting `leftAt` on their SceneAvatar record, with a reason in `customProperties`:

~~~json
[["SceneAvatar/set", {
  "accountId": "admin-account",
  "update": {
    "user:troll@example.com": {
      "leftAt": "2026-06-06T10:22:15Z",
      "customProperties": {
        "ejectReason": "Disruptive behavior",
        "ejectedBy": "user:admin@example.com"
      }
    }
  }
}, "0"]]
~~~

Server responds:

~~~json
[["SceneAvatar/set", {
  "accountId": "admin-account",
  "oldState": "s210",
  "newState": "s211",
  "created": null,
  "updated": {
    "user:troll@example.com": null
  },
  "destroyed": null,
  "notCreated": null,
  "notUpdated": null,
  "notDestroyed": null
}, "0"]]
~~~

The server decrements the region's `activeAvatarCount` and delivers a `StateChange` to the ejected user's connection:

~~~json
{
  "@type": "StateChange",
  "changed": {
    "troll-account": {
      "SceneAvatar": "s211"
    }
  }
}
~~~

The ejected user's client fetches the updated SceneAvatar via `SceneAvatar/get`, sees `leftAt` is non-null, and disconnects from the simulation layer. Because the region's `accessPolicy` is now `"invite"`, the ejected user cannot re-enter without a new invitation.

# Acknowledgements

The author thanks the Mozilla Hubs project for the open-source reference implementation that demonstrated browser-based social 3D environments are viable, the Second Life team for two decades of persistent-world operation that informed the permission and region models, the Khronos Group for glTF 2.0 whose coordinate conventions this specification adopts, and the Gather team for demonstrating that 2D spatial environments are a compelling and practical subset of 3D worlds.
