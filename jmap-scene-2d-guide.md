# JMAP Scene -- 2D Implementer's Guide

For client and server implementers building 2D experiences on top of
`draft-atwood-jmap-scene-00`. Covers top-down virtual offices, side-scrolling
games, tile-grid mapping, sprite-based avatars, and integration with JMAP
VTC for spatial video. For board game patterns, see the
[JMAP Scene Board Games Guide](jmap-scene-board-games-guide.md).

Read the draft first. This guide does not re-state normative requirements. It
covers how to use the Scene spec's 3D data model to build compelling 2D
experiences, and what implementation decisions you must make to ship one.

---

## How to read this guide

The JMAP Scene spec defines a 3D coordinate system (right-handed, Y-up,
meters, quaternions) and four data types (SceneRegion, SceneObject,
SceneAvatar, SceneAsset). Every position is `[x, y, z]`, every orientation is
a quaternion `[x, y, z, w]`, every scale is `[sx, sy, sz]`.

None of that changes for 2D. A 2D experience is a proper subset of the 3D
spec: the `viewHint` field on SceneRegion tells the client which projection
to use, and the client ignores one coordinate axis. The server stores the
same data types regardless of view mode. A 2D client and a 3D client can
look at the same SceneRegion simultaneously -- one renders it top-down, the
other in perspective.

Each section below follows the same shape:

1. **What the spec provides** -- the relevant fields and data types.
2. **How to use it for 2D** -- the concrete mapping from spec concepts to
   2D rendering.
3. **JSON examples** -- wire-format illustrations.
4. **Implementation notes** -- edge cases and decisions you must make.

This guide is non-normative. `draft-atwood-jmap-scene-00` is the source of
truth. Where this guide and the draft disagree, the draft wins.

---

## 1. The viewHint field

### What the spec provides

SceneRegion has a `viewHint` field (String|null) that advises the client on
how to render the region. The spec defines four values:

- `"3d"` -- standard 3D perspective rendering.
- `"2d-topdown"` -- top-down 2D view, as in virtual offices or board games.
- `"2d-side"` -- side-scrolling 2D view, as in platformer games.
- `"ar"` -- augmented reality overlay on the physical world (region coordinates anchored via `geoAnchor`).

`null` is treated as `"3d"`. Clients MUST NOT fail on unrecognized values;
they SHOULD fall back to `"3d"`. Deployment-specific hints SHOULD use
reverse-domain notation (e.g., `"com.example.isometric"`).

### How clients should interpret viewHint

**`"2d-topdown"`:** The client renders a bird's-eye view looking down the
Y axis. The X axis maps to screen-right and the Z axis maps to
screen-down (+Z = increasing row index). The Y coordinate is ignored for
rendering purposes (all objects appear on a flat plane). Clients SHOULD
keep `position.y` at `0` for all objects and avatars.

**`"2d-side"`:** The client renders a side view. The X axis maps to
screen-horizontal and the Y axis maps to screen-vertical. The Z coordinate
is ignored for rendering purposes. This is the natural projection for
platformer or side-scrolling experiences.

**Fallback:** If a client encounters an unrecognized `viewHint`, it SHOULD
render the region in 3D perspective. This ensures forward compatibility:
new view hints can be introduced without breaking existing clients.

### Coordinate mapping

```
2d-topdown                          2d-side
==========                          =======

      +------ +X (screen right)              +Y (screen up)
      |                                       |
      |                                       |
     +Z (screen down)                         +------ +X (screen right)
    origin                                  origin

    Y axis is ignored                       Z axis is ignored
    (keep y=0)                              (keep z=0)

    In top-down view the camera looks down the -Y axis.
    +X maps to screen-right and +Z maps to screen-down,
    matching the row-major convention (row 0 at top).
```

### JSON example: a top-down region

```json
{
  "id": "01J5RGN0000000000000000001",
  "name": "Engineering Floor",
  "bounds": {
    "min": [0, 0, 0],
    "max": [30, 0, 20]
  },
  "viewHint": "2d-topdown",
  "spawnPosition": [15, 0, 10],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/regions/01J5RGN",
  "accessPolicy": "space",
  "spaceId": "01J5SPC0000000000000000001",
  "activeAvatarCount": 7,
  "environment": {
    "backgroundColor": "#F5F5F5",
    "gridSize": 1.0
  }
}
```

Note that `bounds.min.y` and `bounds.max.y` are both `0`. This is a flat
region. The `environment` object is deployment-defined; a 2D client might
use it for background color, grid overlay settings, or tile theme.

### JSON example: a side-scrolling region

```json
{
  "id": "01J5RGN0000000000000000002",
  "name": "Level 3: Crystal Caverns",
  "bounds": {
    "min": [0, 0, 0],
    "max": [200, 30, 0]
  },
  "viewHint": "2d-side",
  "spawnPosition": [2, 1, 0],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/regions/01J5RGN002",
  "accessPolicy": "public",
  "activeAvatarCount": 1,
  "environment": {
    "gravity": 9.81,
    "backgroundColor": "#1a1a2e"
  }
}
```

Here `bounds.max.z` is `0` -- the Z axis is flat. The X axis is the
scrolling axis (200 meters of level), and Y is height (30 meters of
vertical space).

---

## 2. Coordinate conventions for 2D

### Which axes matter

| viewHint | Screen X | Screen Y | Ignored axis | Recommended default |
|---|---|---|---|---|
| `"2d-topdown"` | X | Z | Y | `position.y = 0` |
| `"2d-side"` | X | Y | Z | `position.z = 0` |

### Position

All positions are still `[x, y, z]` in meters, as the spec requires. For
2D experiences, one axis is unused:

- **Top-down:** Set `position[1]` (Y) to `0` for all objects and avatars.
  The server stores it; the client ignores it during rendering.
- **Side-scrolling:** Set `position[2]` (Z) to `0` for all objects and
  avatars.

Servers SHOULD accept non-zero values on the ignored axis without error.
Clients SHOULD clamp the ignored axis to `0` when creating or updating
objects in a 2D region, but MUST NOT fail if the server returns a non-zero
value.

### Orientation simplifies to a single rotation

In 3D, orientation is a full quaternion `[x, y, z, w]` encoding arbitrary
rotation. In 2D, rotation is around a single axis:

- **Top-down:** Rotation around the Y axis (looking down). A quaternion
  encoding a Y-axis rotation of angle `theta` is:
  `[0, sin(theta/2), 0, cos(theta/2)]`.
- **Side-scrolling:** Rotation around the Z axis (into the screen). A
  quaternion encoding a Z-axis rotation of angle `theta` is:
  `[0, 0, sin(theta/2), cos(theta/2)]`.

Clients rendering 2D SHOULD extract the relevant rotation component and
ignore the others. Common facing directions for top-down:

| Facing | Screen direction | Angle (rad) | Quaternion `[x, y, z, w]` |
|---|---|---|---|
| +Z | Down | 0 | `[0, 0, 0, 1]` |
| +X | Right | -pi/2 | `[0, -0.707, 0, 0.707]` |
| -Z | Up | pi | `[0, 1, 0, 0]` |
| -X | Left | pi/2 | `[0, 0.707, 0, 0.707]` |

### Scale

Scale `[sx, sy, sz]` works the same in 2D, but only two components are
visually meaningful in each view mode:

| viewHint | `sx` | `sy` | `sz` |
|---|---|---|---|
| `"2d-topdown"` | screen width (right) | ignored (keep 1) | screen height (down) |
| `"2d-side"` | screen width (right) | screen height (up) | ignored (keep 1) |

For a 2D sprite that is 1 tile wide and 1 tile tall, use `[1, 1, 1]`.
For a 2-tile-wide, 1-tile-tall desk: use `[2, 1, 1]` in top-down
(`sx`=2 tiles wide on screen, `sz`=1 tile tall on screen) or `[2, 1, 1]`
in side-scrolling (`sx`=2 tiles wide, `sy`=1 tile tall). A desk that is
3 tiles wide and 2 tiles tall would be `[3, 1, 2]` in top-down (width
in `sx`, height in `sz`) vs. `[3, 2, 1]` in side-scrolling (width in
`sx`, height in `sy`). Negative scale values mirror the sprite, which is
useful for side-scrolling characters that face left vs. right (negate
`sx` to flip horizontally).

---

## 3. Mapping tile grids to Scene coordinates

Many 2D experiences use tile-based maps. The Scene spec uses meters as its
unit. The simplest mapping is 1 tile = 1 meter.

### Tile-to-coordinate mapping (top-down)

```
Tile grid (map editor)          Scene coordinates (2d-topdown)

  col 0  col 1  col 2              X=0    X=1    X=2
  +------+------+------+           +------+------+------+
  |      |      |      | row 0     |      |      |      | Z=0
  +------+------+------+           +------+------+------+
  |      |      |      | row 1     |      |      |      | Z=1
  +------+------+------+           +------+------+------+
  |      |      |      | row 2     |      |      |      | Z=2
  +------+------+------+           +------+------+------+

  Tile (col, row) -> position [col, 0, row]
  Tile center: [col + 0.5, 0, row + 0.5]
```

A 30x20 tile map produces a region with `bounds.min = [0, 0, 0]` and
`bounds.max = [30, 0, 20]`.

### Tile-to-coordinate mapping (side-scrolling)

```
Tile grid (level editor)        Scene coordinates (2d-side)

  col 0  col 1  col 2              X=0    X=1    X=2
  +------+------+------+           +------+------+------+
  |      |      |      | row 0     |      |      |      | Y=2
  +------+------+------+           +------+------+------+
  |      |      |      | row 1     |      |      |      | Y=1
  +------+------+------+           +------+------+------+
  |      |      |      | row 2     |      |      |      | Y=0
  +------+------+------+           +------+------+------+

  Tile (col, row) -> position [col, maxRow - row, 0]
  Note: row 0 is at the top in the editor but Y=maxRow in Scene
  coordinates (Y-up).
```

### Wall tiles and floor tiles as SceneObjects

Static map geometry (walls, floors, furniture) can be modeled as
SceneObjects with `physicsMode: "static"` and `interactable: false`. Each
wall tile becomes a SceneObject at the corresponding grid position.

For large maps, this produces many objects. Two optimization strategies:

1. **Merged objects:** Combine adjacent wall tiles into a single SceneObject
   spanning multiple grid cells. A horizontal wall from (2,0) to (5,0)
   becomes one object at `position: [2, 0, 0]` with `scale: [4, 1, 1]`.
2. **Background layer:** Encode the full tile map as a single large image
   (the "background" SceneObject) and only create individual SceneObjects
   for interactive or dynamic elements. The background image handles
   visual rendering; collision data lives in the simulation layer, not in
   individual SceneObjects.

### JSON example: a wall segment

```json
{
  "id": "01J5OBJ0000000000000000010",
  "regionId": "01J5RGN0000000000000000001",
  "parentId": null,
  "name": "North wall",
  "position": [0, 0, 0],
  "orientation": [0, 0, 0, 1],
  "scale": [30, 1, 1],
  "visualRef": "blob-wall-tile-001",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": false,
  "visible": true
}
```

This wall spans 30 tiles along the X axis at row Z=0.

---

## 4. Sprite-based avatars

### Using visualRef with image types

The spec does not restrict `visualType` to 3D model formats. The account
capability `supportedVisualTypes` lists the media types the deployment
supports. For 2D deployments, this list SHOULD include `"image/png"` and
MAY include `"image/svg+xml"`, `"image/webp"`, or `"image/gif"`.

A sprite-based avatar uses `visualRef` pointing to a PNG or SVG blob and
`visualType` set to the corresponding media type. The server stores and
serves it identically to a glTF model reference -- the rendering
interpretation is the client's responsibility.

### JSON example: a sprite avatar

```json
{
  "id": "user:bob@example.com",
  "regionId": "01J5RGN0000000000000000001",
  "userId": "user:bob@example.com",
  "displayName": "Bob Martinez",
  "position": [8, 0, 5],
  "orientation": [0, -0.707, 0, 0.707],
  "visualRef": "blob-sprite-bob-001",
  "visualType": "image/png",
  "joinedAt": "2026-06-06T09:00:00Z",
  "leftAt": null,
  "worldState": {
    "spriteSheet": "blob-spritesheet-bob-001",
    "animationState": "idle",
    "facingDirection": "east"
  }
}
```

### Downloading sprite sheet blobs

A `visualRef` value is a JMAP blob ID. Clients retrieve the image data via
the standard JMAP blob download path (RFC 8620 Section 6.2):

```
GET /jmap/download/{accountId}/{blobId}/{name}
```

The same mechanism applies when `worldState` references a secondary
sprite sheet blob (e.g., `"spriteSheet": "blob-spritesheet-bob-001"`): the
client issues a separate blob download request for that blob ID. The server
treats all blob references identically regardless of whether they come from
`visualRef` or from inside `worldState`.

For sprite sheets (texture atlases), the downloaded image contains multiple
animation frames packed into a single file. `worldState` carries the
metadata needed to extract individual frames: the pixel dimensions of each
frame, how many columns and rows are in the sheet, and which frame indices
belong to each animation. See the recommended schema below.

Clients MAY cache blob downloads by blob ID. Blob IDs are content-addressed:
the same ID always refers to the same bytes. When a deployment changes an
avatar's sprite sheet, it uploads a new blob and updates the avatar's
`worldState` to reference the new blob ID.

### Sprite sheets and animation

The spec's `visualRef` points to a single asset. For animated sprites
(walk cycles, idle animations), two patterns are available:

1. **Single sprite sheet in visualRef:** The `visualRef` points to a sprite
   sheet image. The client uses `worldState` to determine which frame
   to display. The sprite sheet layout (frame size, frame count, animation
   names) is deployment-defined.
2. **Multiple SceneAssets:** Upload individual frames or animation-specific
   sprite sheets as separate SceneAssets. Use `worldState` to
   reference the active animation's asset ID.

Pattern 1 is simpler and requires fewer asset uploads. Pattern 2 allows
per-animation asset swapping, which is useful when avatars have many
costume options.

### Recommended sprite sheet metadata schema

The spec's `worldState` is opaque, so sprite sheet layout is
deployment-defined. The following schema is a recommended convention
for interoperability between clients. Deployments MAY use a different
schema, but clients that encounter these keys SHOULD interpret them as
described here.

**On the SceneAvatar or SceneObject that references the sprite sheet:**

```json
{
  "worldState": {
    "spriteSheet": "blob-spritesheet-bob-001",
    "spriteSheetMeta": {
      "frameWidth": 32,
      "frameHeight": 32,
      "columns": 4,
      "rows": 4,
      "animations": {
        "idle":  { "frames": [0, 1, 2, 3], "fps": 4, "loop": true },
        "walk":  { "frames": [4, 5, 6, 7, 8, 9, 10, 11], "fps": 8, "loop": true },
        "interact": { "frames": [12, 13, 14, 15], "fps": 6, "loop": false }
      }
    },
    "animationState": "idle",
    "facingDirection": "east"
  }
}
```

**Field definitions for `spriteSheetMeta`:**

| Field | Type | Description |
|---|---|---|
| `frameWidth` | Number | Width of one frame in pixels. |
| `frameHeight` | Number | Height of one frame in pixels. |
| `columns` | Number | Number of frame columns in the sheet image. |
| `rows` | Number | Number of frame rows in the sheet image. |
| `animations` | Object | Map of animation name to animation descriptor. |

**Animation descriptor fields:**

| Field | Type | Description |
|---|---|---|
| `frames` | Number[] | Ordered list of frame indices (0-based, left-to-right then top-to-bottom). |
| `fps` | Number | Playback rate in frames per second. |
| `loop` | Boolean | `true` to loop; `false` to play once and hold the last frame. |

Frame index `0` is the top-left cell of the sheet. Index `n` maps to
column `n % columns`, row `floor(n / columns)`. The pixel region for
frame `n` is `(col * frameWidth, row * frameHeight, frameWidth,
frameHeight)`.

When `spriteSheetMeta` is absent, the client falls back to
deployment-specific conventions or treats `visualRef` as a static image.

### Orientation as facing direction

In a top-down 2D view, the avatar's quaternion orientation encodes facing
direction. Clients SHOULD extract the Y-axis rotation from the quaternion
to determine which sprite frame to display (north-facing, south-facing,
etc.). For a 4-direction sprite sheet:

```
Quaternion Y rotation   ->   Sprite direction
[0, 0, 0, 1]                +Z facing (down on screen)
[0, -0.707, 0, 0.707]       +X facing (right on screen)
[0, 1, 0, 0]                -Z facing (up on screen)
[0, 0.707, 0, 0.707]        -X facing (left on screen)
```

For 8-direction sprite sheets, use 45-degree increments. Clients MAY also
use the `worldState.facingDirection` string if the deployment prefers
explicit direction labels over quaternion math.

In a side-scrolling view, facing is simpler: the character faces left or
right. Use `scale[0]` as a mirror flag: positive for right-facing, negative
for left-facing. Alternatively, encode facing in `worldState`.

---

## 5. Gather-like virtual offices

This section covers building a Gather-style 2D virtual office using the
Scene spec.

### Architecture overview

```
+--------------------------------------------------+
|  SceneRegion: "Engineering Floor"                |
|  viewHint: "2d-topdown"                          |
|  bounds: [0,0,0] to [30,0,20]                   |
|                                                  |
|  +--------+  +--------+  +--------+             |
|  | Desk A |  | Desk B |  | Desk C |  <- SceneObjects
|  | (5,0,3)|  |(10,0,3)|  |(15,0,3)|             |
|  +--------+  +--------+  +--------+             |
|                                                  |
|     @          @                     <- SceneAvatars
|   Alice       Bob                                |
|  (6,0,5)    (11,0,4)                            |
|                                                  |
|  +------------------+                            |
|  |   Whiteboard     |  <- SceneObject            |
|  |   (20,0,10)      |     interactable: true     |
|  +------------------+                            |
|                                                  |
|  +----------+                                    |
|  | Meeting  |  <- SceneRegion (child)            |
|  | Room     |     or zone within this region     |
|  +----------+                                    |
+--------------------------------------------------+
```

### Modeling rooms

Each floor or area is a SceneRegion with `viewHint: "2d-topdown"`. For a
multi-room office:

- **One region per floor:** All desks, meeting rooms, and common areas are
  SceneObjects within a single region. Meeting "rooms" are visual zones,
  not separate SceneRegions. Proximity is computed within one coordinate
  space.
- **One region per room:** Each meeting room is a separate SceneRegion.
  Moving between rooms requires leaving one region and entering another
  (a `SceneAvatar/set` update (setting `leftAt`) followed by a
  `SceneAvatar/set` create in the new region). This provides stronger
  isolation: avatars in different regions cannot see each other.

For most virtual offices, one region per floor is simpler. Use separate
regions only when you need access control isolation (e.g., a private
meeting room that non-members cannot observe).

### Furniture as SceneObjects

Desks, chairs, whiteboards, plants, and other office furniture are
SceneObjects positioned on the tile grid.

```json
{
  "id": "01J5OBJ0000000000000000020",
  "regionId": "01J5RGN0000000000000000001",
  "name": "Desk A",
  "position": [5, 0, 3],
  "orientation": [0, 0, 0, 1],
  "scale": [2, 1, 1],
  "visualRef": "blob-desk-sprite-001",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "worldState": {
    "assignedTo": "user:alice@example.com",
    "label": "Alice's Desk"
  }
}
```

Interactive furniture (whiteboards, TVs, vending machines) uses
`interactable: true`. The interaction behavior (opening a whiteboard app,
starting a video) is handled by the simulation layer or by
application-level logic triggered by a `SceneInteractionEvent`.

### Proximity-based interactions

The core Gather experience is proximity-based: when two avatars are close
enough, their video/audio activates. The Scene spec supports this through
spatial queries.

**Detecting proximity on the server:** Use `SceneAvatar/query` with
`withinRadius` to find avatars near a given position:

```json
[["SceneAvatar/query", {
  "accountId": "account-xyz",
  "filter": {
    "regionId": "01J5RGN0000000000000000001",
    "isActive": true,
    "withinRadius": {
      "center": [6, 0, 5],
      "radius": 3
    }
  }
}, "0"]]
```

This returns all active avatars within 3 meters of Alice's position. The
simulation layer uses this (or its own real-time equivalent) to determine
which video tiles to show and which audio streams to mix.

**Detecting proximity on the client:** If the simulation layer pushes
real-time avatar positions to connected clients, the client can compute
proximity locally. This avoids a JMAP round-trip for every position update.
The JMAP spatial query is a fallback for clients not connected to the
simulation layer.

### Proximity zones

Some virtual offices define named zones (the "kitchen", the "lounge") where
being present triggers specific behavior (background music, group video).
Model these as invisible SceneObjects with `visible: false` and
`physicsMode: "none"`:

```json
{
  "id": "01J5OBJ0000000000000000030",
  "regionId": "01J5RGN0000000000000000001",
  "name": "Kitchen zone",
  "position": [25, 0, 15],
  "orientation": [0, 0, 0, 1],
  "scale": [5, 1, 5],
  "visualRef": null,
  "visualType": null,
  "physicsMode": "none",
  "interactable": false,
  "visible": false,
  "worldState": {
    "zoneType": "social",
    "backgroundAudio": "blob-audio-cafe-ambience"
  }
}
```

The client or simulation layer checks whether an avatar's position falls
within the zone's bounding box (`position +/- scale/2`) and applies the
zone-specific behavior.

---

## 6. Video game patterns

### Single-player games

A single-player game uses one SceneAvatar (the player) and many
SceneObjects (enemies, items, terrain, triggers). The simulation layer runs
game logic: physics, AI, collision detection, scoring.

**Game input:** The player interacts with the world through
`SceneInteractionEvent` actions delivered via the JMAP WebSocket
(SceneStreamEnable). The Scene WSS spec (draft-atwood-jmap-scene-wss-00)
registers click, grab, release, and activate as standard actions, plus
extensible custom actions. A game client maps keyboard/gamepad input to
these events:

- Arrow keys / WASD -> avatar position updates (via simulation layer)
- Space bar -> `activate` on the nearest interactable object
- E key -> `grab` / `release` for inventory pickup
- Mouse click -> `click` on a targeted object

**Authoritative simulation:** Even for single-player, run an authoritative
simulation server if cheating matters (leaderboards, achievements). The
client sends input events; the server computes outcomes and updates
SceneObject positions. Clients predict locally and reconcile with server
state.

### Multiplayer games

Multiple SceneAvatars in the same region. The simulation layer handles:

- **Authoritative position:** The server validates avatar movement. Clients
  send movement intent; the server computes the resulting position and
  broadcasts it. `SceneAvatar.position` in the JMAP store reflects the
  server-authoritative position, updated periodically.
- **Object ownership:** Use `SceneObject.ownerId` to track which player
  controls a dynamic object. The simulation layer enforces that only the
  owner can move or interact with their objects.
- **Visibility/fog of war:** Use the spec's visibility contract
  (deployment-defined visibility filtering) to restrict which objects each
  player can see. A competitive game might hide enemy positions by
  filtering `SceneObject/get` responses based on line-of-sight from the
  player's avatar.

### Side-scrolling games (`"2d-side"`)

Side-scrolling games use `viewHint: "2d-side"`. Key differences from
top-down:

- **Gravity matters.** The `environment.gravity` field (deployment-defined)
  tells the simulation layer to apply downward acceleration. Objects with
  `physicsMode: "dynamic"` fall. Platforms are `physicsMode: "static"`.
- **Jump physics.** The simulation layer handles jump arcs. The client sends
  a jump intent; the server applies an upward impulse and lets gravity
  bring the avatar back down.
- **Horizontal scrolling.** The client viewport follows the player avatar
  along the X axis. The region's `bounds.max.x` defines the level length.

### JSON example: a side-scrolling platform

```json
{
  "id": "01J5OBJ0000000000000000040",
  "regionId": "01J5RGN0000000000000000002",
  "name": "Floating platform",
  "position": [12, 5, 0],
  "orientation": [0, 0, 0, 1],
  "scale": [4, 0.5, 1],
  "visualRef": "blob-platform-stone-001",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": false,
  "visible": true
}
```

This platform is 4 meters wide, 0.5 meters tall, positioned at X=12, Y=5
(5 meters above the ground). The Z coordinate is 0 (ignored in
side-scrolling).

### JSON example: a collectible item

```json
{
  "id": "01J5OBJ0000000000000000041",
  "regionId": "01J5RGN0000000000000000002",
  "name": "Gold coin",
  "position": [14, 7, 0],
  "orientation": [0, 0, 0, 1],
  "scale": [0.5, 0.5, 1],
  "visualRef": "blob-coin-sprite-001",
  "visualType": "image/png",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "worldState": {
    "itemType": "coin",
    "value": 10
  }
}
```

When the player avatar overlaps this object, the simulation layer fires a
collision or interaction event. The application logic awards points, the
SceneObject is destroyed (`SceneObject/set destroy`), and the client plays
a collection animation.

---

## 7. Board game patterns

Board games map naturally to the Scene spec: the board is a SceneRegion
with `viewHint: "2d-topdown"`, pieces are SceneObjects, and players are
SceneAvatars. The coordinate conventions from sections 2-3 apply
directly -- each board square is one meter, and piece positions map to
grid coordinates.

For comprehensive coverage of board game implementation patterns
including chess, Go, checkers, card games, dice, hidden information,
turn enforcement, and game state tracking, see
**`jmap-scene-board-games-guide.md`**. That guide covers:

- **Board region setup** -- bounds, environment, and access policy.
- **Piece modeling** -- SceneObjects with `interactable: true` and
  `physicsMode: "none"`, with piece identity in `worldState`.
- **Move mechanics** -- click-to-select and drag-and-drop (grab/release)
  patterns with server-side validation.
- **Game state tracking** -- invisible SceneObject (`visible: false`)
  holding turn, phase, and score data in `worldState`.
- **Hidden information** -- visibility filtering for card games
  (server replaces `visualRef` and redacts `worldState` for
  non-owners).
- **Dice** -- `activate` interaction triggering server-side random
  result with `StateChange` broadcast.
- **Turn enforcement** -- simulation layer rejecting out-of-turn
  interaction events.
- **Chat integration** -- game events posted to the bound Chat via
  `chatId`.
- **Worked examples** -- Tic-Tac-Toe, Go, Checkers, Chess, Sorry!,
  Monopoly, Battleship, Poker, Mahjong, and more.

---

## 8. Integration with VTC for spatial video

For WebSocket connection management, SceneStreamEnable setup, reconnection,
and event handling, see the
[JMAP Scene WebSocket Guide](jmap-scene-wss-guide.md).

### What the spec provides

When `urn:ietf:params:jmap:vtc` is co-deployed, a SceneRegion MAY carry
an `activeCallId` binding to a VTCCall. The spec states that spatial audio
is a simulation-layer concern: the VTCCall provides the voice channel; the
simulation layer handles spatialization based on avatar positions.

### Proximity-based video tiles in a 2D office

The classic Gather pattern: when two avatars are within N meters of each
other, their webcam video tiles appear. When they move apart, the tiles
fade and disappear.

Implementation:

1. **Region setup:** The SceneRegion has `activeCallId` set to a VTCCall.
   All users in the region are participants in that VTCCall.
2. **Proximity detection:** The simulation layer (or client) computes
   pairwise distances between avatars.
3. **Video tile visibility:** When distance < threshold, the client renders
   the remote participant's video tile. When distance > threshold, the
   client hides it.
4. **Audio attenuation:** The simulation layer applies distance-based audio
   attenuation to VTC audio streams. Nearby avatars are loud; distant
   avatars are quiet or silent.

```
+---------------------------------------------+
|  2D Office Floor (SceneRegion)              |
|  activeCallId: "01J5CALL001"                |
|                                              |
|     @Alice ---- 2m ---- @Bob                |
|     [video]             [video]              |
|                                              |
|                    @Carol                    |
|                    (15m away, no video)       |
+---------------------------------------------+
```

### Linking avatar position to spatial audio

The VTCCall provides audio streams for all participants. The simulation
layer uses SceneAvatar positions to spatialize those streams:

1. For each pair of participants, compute the vector from listener to
   speaker using their SceneAvatar positions.
2. Apply distance attenuation (inverse-square or linear falloff, deployment
   choice).
3. Apply stereo panning based on the horizontal angle.
4. For top-down 2D, the "horizontal" angle is computed in the XZ plane.

This is entirely a simulation-layer concern. The JMAP Scene spec provides
the positions; the JMAP VTC spec provides the audio streams; the simulation
layer connects them.

### Entering a region with VTC

When a user enters a region that has `activeCallId` set:

1. Create a SceneAvatar in the region (`SceneAvatar/set create`).
2. Join the VTCCall (`VTCParticipant/set create` with the `activeCallId`).

These are two independent JMAP operations. The spec does not auto-join the
call when entering the region, and does not auto-enter the region when
joining the call. The client is responsible for coordinating both.

A deployment MAY automate this: when the server processes a
`SceneAvatar/set create` for a region with a non-null `activeCallId`, the
server MAY automatically create a VTCParticipant record for the user in
that call. This is a deployment behavior, not a spec requirement.

---

## 9. SceneObject as a sprite: putting it all together

A complete 2D SceneObject representing a desk in a virtual office, with
all fields shown:

```json
{
  "id": "01J5OBJ0000000000000000080",
  "regionId": "01J5RGN0000000000000000001",
  "parentId": null,
  "name": "Standing Desk B2",
  "position": [10, 0, 3],
  "orientation": [0, 0, 0, 1],
  "scale": [2, 1, 1],
  "visualRef": "blob-desk-standing-001",
  "visualType": "image/png",
  "assetUri": "https://cdn.example.com/tiles/desk-standing.png",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "ownerId": null,
  "createdAt": "2026-06-01T08:00:00Z",
  "updatedAt": "2026-06-01T08:00:00Z",
  "worldState": {
    "assignedTo": "user:bob@example.com",
    "tileWidth": 2,
    "tileHeight": 1,
    "interactionType": "status-display",
    "statusText": "Focusing until 2pm"
  }
}
```

A complete 2D SceneAvatar representing a user with a sprite:

```json
{
  "id": "user:alice@example.com",
  "regionId": "01J5RGN0000000000000000001",
  "userId": "user:alice@example.com",
  "displayName": "Alice Chen",
  "position": [6, 0, 5],
  "orientation": [0, -0.707, 0, 0.707],
  "visualRef": "blob-avatar-alice-pixel-001",
  "visualType": "image/png",
  "joinedAt": "2026-06-06T09:15:00Z",
  "leftAt": null,
  "worldState": {
    "spriteSheet": "blob-spritesheet-alice-001",
    "animationState": "walking",
    "facingDirection": "east",
    "statusEmoji": "headphones",
    "awayState": false
  }
}
```

---

## Appendix: Decision checklist

Before shipping a 2D experience on JMAP Scene, verify that your
implementation has made and documented each of the following decisions:

**View mode (section 1)**
- [ ] `viewHint` value chosen (`"2d-topdown"` or `"2d-side"`)
- [ ] Client handles unknown `viewHint` values with `"3d"` fallback
- [ ] Ignored axis documented (Y for top-down, Z for side-scrolling)

**Coordinate conventions (sections 2-3)**
- [ ] Tile-to-meter ratio defined (recommended: 1 tile = 1 meter)
- [ ] Tile grid origin mapped to Scene coordinates
- [ ] Bounds set correctly for the map dimensions

**Visual assets (section 4)**
- [ ] `supportedVisualTypes` includes `"image/png"` (and optionally
  `"image/svg+xml"`, `"image/webp"`)
- [ ] Sprite sheet format documented (frame size, animation layout)
- [ ] Orientation-to-facing-direction mapping defined

**Virtual office (section 5)**
- [ ] Region-per-floor vs. region-per-room decision made
- [ ] Proximity threshold defined for interactions
- [ ] Zone model defined (invisible SceneObjects or application logic)
- [ ] Furniture objects created with correct physicsMode and interactable

**Video games (section 6)**
- [ ] Simulation authority model chosen (client-authoritative vs.
  server-authoritative)
- [ ] Input-to-interaction-event mapping defined
- [ ] For side-scrolling: gravity and jump physics implemented in
  simulation layer

**Board games (section 7; see `jmap-scene-board-games-guide.md`)**
- [ ] Piece interaction model defined (grab/release vs. click-to-move)
- [ ] Turn enforcement implemented in simulation layer or application
  logic
- [ ] Hidden information handled via visibility filtering
- [ ] Game state tracking mechanism chosen (invisible object vs.
  external state)

**VTC integration (section 8)**
- [ ] Proximity threshold for video tile visibility defined
- [ ] Audio attenuation curve defined (linear, inverse-square, or
  stepped)
- [ ] Region entry and VTCCall join coordinated in client
- [ ] Spatial audio panning implemented in simulation layer
