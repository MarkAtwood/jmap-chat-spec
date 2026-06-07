# JMAP Scene Games Implementer Guide

How to implement classic board games, card games, and video games
using the JMAP Scene protocol. Each game maps onto the same Scene
primitives: a SceneRegion defines the play area, SceneObjects
represent pieces, tokens, projectiles, and terrain, SceneAvatars
represent players, and an invisible game-state SceneObject tracks
turns, scores, and phase. The simulation layer (reached via
`simulationUri`) enforces rules; the Scene spec provides only the
spatial state layer.

Games in this guide span all three `viewHint` values:

- **`"2d-topdown"`** -- board games, card games, top-down arcade
  (Asteroids)
- **`"2d-side"`** -- platformers, side-scrollers (Pitfall)
- **`"3d"`** -- first-person games (Battlezone, Doom, Quake, Descent,
  Flight Combat)

---

## Common Patterns

> **Fields omitted for brevity.** JSON examples throughout this guide
> omit SceneObject and SceneRegion fields that are unchanged from their
> defaults or are server-set. Omitted fields include: `orientation`
> (defaults to `[0, 0, 0, 1]`), `scale` (defaults to `[1, 1, 1]`),
> `parentId` (defaults to `null`), `assetUri`, `accountId`,
> `createdAt`, `updatedAt`, `description`, `spawnOrientation`,
> `activeAvatarCount`, and optional binding fields (`chatId`,
> `spaceId`, `channelId`, `activeCallId` -- shown only when relevant
> to the game). All fields are defined in `draft-atwood-jmap-scene-00`.

### Board Region

Every board game starts with a SceneRegion whose bounds match the
board dimensions. One unit = one tile/square/cell.

```json
{
  "id": "region-game-001",
  "name": "Game Table",
  "bounds": { "min": [0, 0, 0], "max": [8, 0, 8] },
  "viewHint": "2d-topdown",
  "spawnPosition": [4, 0, -1],
  "simulationUri": "wss://sim.example.com/games/chess/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic",
    "gridVisible": true
  }
}
```

> **Note:** Purely turn-based games with no real-time physics MAY use
> `simulationUri: null`. In that case the server handles all game logic
> via JMAP method calls alone (`SceneObject/set` for moves, custom
> methods for turn validation). The `wss://` examples throughout this
> guide show the general case; omitting `simulationUri` is a valid
> choice for simple board games.

### Game Pieces

Pieces are SceneObjects with `interactable: true` and
`physicsMode: "none"`. Piece identity lives in `customProperties`.

```json
{
  "id": "piece-001",
  "regionId": "region-game-001",
  "name": "White Pawn",
  "position": [3, 0, 1],
  "visualRef": "blob-white-pawn",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "pieceType": "pawn",
    "color": "white"
  }
}
```

> **Note on `supportedVisualTypes`:** Deployments serving SVG game
> assets should include `"image/svg+xml"` in the region's
> `supportedVisualTypes` list alongside the spec-required
> `"model/gltf-binary"`. Clients that do not support SVG will fall
> back to the glTF asset when both are listed.

### Game State Object

An invisible, non-interactable SceneObject tracks shared game state:

```json
{
  "id": "state-001",
  "regionId": "region-game-001",
  "name": "Game State",
  "position": [0, 0, 0],
  "visualRef": null,
  "visualType": null,
  "physicsMode": "none",
  "interactable": false,
  "visible": false,
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 1,
    "phase": "main",
    "winner": null
  }
}
```

The simulation layer rejects `SceneObject/set` updates from a player
when it is not their turn. Turn transitions update the game state
object; all clients receive the `StateChange` and the UI updates
accordingly.

In the per-game sections that follow, `### Game State` subsections
show only the `customProperties` portion of the game state
SceneObject. The enclosing object always uses `visible: false`,
`interactable: false`, `visualRef: null`, `visualType: null`, and
`physicsMode: "none"` per the pattern above.

### Move Mechanics

**Click-to-select, click-to-place:** Player clicks a piece (generates
`"click"` interaction), valid destinations highlight, player clicks
a destination. Best for turn-based games with discrete moves.

**Drag-and-drop:** Player grabs a piece (`"grab"` interaction), drags
to target position, releases (`"release"` interaction). The simulation
layer validates the move; on rejection, the client snaps the piece
back to its prior position.

### Hidden Information

For games where players cannot see each other's pieces (Battleship,
Stratego), the server uses visibility filtering. When a piece has
`ownerId` set to one player, the server returns a face-down or
generic visual (`visualRef` pointing to a card-back blob) to the
other player. The true `customProperties` are omitted or redacted in
the response sent to non-owners.

### Chat Integration

When `SceneRegion.chatId` is set, game events are posted to the bound
Chat as system messages: "Alice moved Knight to e4", "Bob captured
Alice's Rook", "Game over -- Alice wins." Players can also chat with
each other in the same channel.

### Spectators

SceneAvatars beyond the active players are spectators. They can see
all public game state but cannot interact with pieces. The simulation
layer ignores interaction events from non-player avatars.

### Real-Time Event Notifications

Clients wanting real-time push notifications for move updates
(SceneObjectEvent for piece position changes, turn transitions, and
game state changes) should see the
[JMAP Scene WebSocket Guide](jmap-scene-wss-guide.md) for
subscription setup.


---

## 1. Tic-Tac-Toe

The simplest possible board game. Good for validating a Scene game
implementation end-to-end.

### Region

3x3 board. Two players.

```json
{
  "id": "region-ttt-001",
  "name": "Tic-Tac-Toe: Alice vs Bob",
  "bounds": { "min": [0, 0, 0], "max": [3, 0, 3] },
  "viewHint": "2d-topdown",
  "spawnPosition": [1.5, 0, -1],
  "simulationUri": "wss://sim.example.com/games/ttt/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic",
    "gridVisible": true
  }
}
```

### Pieces

No initial pieces. Pieces are created on each turn via
`SceneObject/set create`. The board starts empty.

**X mark placed at center:**

```json
{
  "id": "ttt-piece-001",
  "regionId": "region-ttt-001",
  "name": "X",
  "position": [1, 0, 1],
  "visualRef": "blob-mark-x",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "mark": "X",
    "col": 1,
    "row": 1
  }
}
```

Marks are non-interactable once placed -- they never move.

### Interaction Model

Click-to-place. The player clicks an empty cell on the board. The
client sends a `SceneObject/set create` with the position
corresponding to the clicked cell. The simulation layer validates:

1. The cell is empty (no existing piece at that position).
2. It is the player's turn.
3. The game is not over.

On success, the piece is created and the turn advances. On failure,
the server returns a `SetError`.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 5,
    "phase": "playing",
    "winner": null,
    "playerX": "user:alice@example.com",
    "playerO": "user:bob@example.com"
  }
}
```

When three in a row is detected, the simulation layer sets
`phase: "finished"` and `winner` to the winning player's ID.

### Hidden Information

None. All state is public.


---

## 2. Go

An ancient territory-control game on a grid. Notable for its large
board, capture mechanics, and ko rule.

### Region

Standard board sizes are 9x9, 13x13, or 19x19. A 19x19 region can
hold up to 361 stones.

```json
{
  "id": "region-go-001",
  "name": "Go: Alice (Black) vs Bob (White)",
  "bounds": { "min": [0, 0, 0], "max": [19, 0, 19] },
  "viewHint": "2d-topdown",
  "spawnPosition": [9.5, 0, -1],
  "simulationUri": "wss://sim.example.com/games/go/001",
  "accessPolicy": "invite",
  "environment": {
    "boardSize": 19,
    "boardTheme": "bamboo",
    "gridVisible": true,
    "starPoints": true
  }
}
```

### Pieces

Stones are created on placement, destroyed on capture. No pre-existing
pieces.

**Black stone placed at intersection (3, 3):**

```json
{
  "id": "go-stone-042",
  "regionId": "region-go-001",
  "name": "Black 42",
  "position": [3, 0, 3],
  "visualRef": "blob-stone-black",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "color": "black",
    "moveNumber": 42
  }
}
```

### Interaction Model

Click-to-place, same as Tic-Tac-Toe. The simulation layer validates:

1. The intersection is empty.
2. It is the player's turn.
3. The placement does not violate the ko rule (cannot recreate the
   immediately preceding board position).
4. The placement does not result in self-capture (unless it first
   captures opponent stones, giving it liberties).

On capture, the simulation layer issues `SceneObject/set destroy` for
all captured stones and increments the capturing player's prisoner
count in the game state object.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 42,
    "phase": "playing",
    "winner": null,
    "blackPrisoners": 3,
    "whitePrisoners": 5,
    "lastMove": [3, 3],
    "koPoint": null,
    "consecutivePasses": 0,
    "komi": 6.5
  }
}
```

**Passing:** A player may pass instead of placing a stone (sends
an interaction event with `action: "pass"` on the game state object).
Two consecutive passes end the game and trigger scoring.

**Scoring:** Territory counting is complex. The simulation layer
computes territory after both players pass, updates the game state
with final scores (territory + prisoners + komi for white), and sets
`phase: "scoring"`. Players confirm dead stones via `"activate"`
interactions on disputed groups, then the simulation finalizes the
result.

### Hidden Information

None. All state is public (Go is a perfect-information game).


---

## 3. Checkers

An 8x8 board game with diagonal movement and mandatory captures.

### Region

8x8 board using only the dark squares (32 playable positions).

```json
{
  "id": "region-checkers-001",
  "name": "Checkers: Alice (Red) vs Bob (Black)",
  "bounds": { "min": [0, 0, 0], "max": [8, 0, 8] },
  "viewHint": "2d-topdown",
  "spawnPosition": [4, 0, -1],
  "simulationUri": "wss://sim.example.com/games/checkers/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic",
    "gridVisible": true
  }
}
```

### Pieces

12 pieces per player, placed on dark squares of the first three rows.

**Red checker at position (1, 0):**

```json
{
  "id": "checker-r01",
  "regionId": "region-checkers-001",
  "name": "Red 1",
  "position": [1, 0, 0],
  "visualRef": "blob-checker-red",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "color": "red",
    "kinged": false
  }
}
```

### Interaction Model

Drag-and-drop. Player grabs a checker and drops it on a target square.
The simulation layer validates:

1. The move is diagonal (one square forward, or any direction if
   kinged).
2. If a capture is available, the player must capture (mandatory
   capture rule).
3. Multi-jump: if the landing square allows another capture, the
   player must continue jumping. The simulation holds the turn open
   until no more captures are available.

On capture, the jumped piece is destroyed via `SceneObject/set
destroy`.

**Kinging:** When a piece reaches the opposite back row, the
simulation updates `customProperties.kinged: true` and changes
`visualRef` to the kinged visual (e.g., a stacked checker image).

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 18,
    "phase": "playing",
    "winner": null,
    "redPieces": 10,
    "blackPieces": 9,
    "multiJumpInProgress": false
  }
}
```

The game ends when one player has no pieces remaining or no legal
moves.

### Hidden Information

None.


---

## 4. Chess

The canonical board game. 8x8 board with six piece types and complex
movement rules.

### Region

```json
{
  "id": "region-chess-001",
  "name": "Chess: Alice (White) vs Bob (Black)",
  "bounds": { "min": [0, 0, 0], "max": [8, 0, 8] },
  "viewHint": "2d-topdown",
  "spawnPosition": [4, 0, -1],
  "simulationUri": "wss://sim.example.com/games/chess/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic-wood",
    "gridVisible": true,
    "notation": "algebraic"
  }
}
```

### Pieces

16 pieces per player in standard starting positions. Each piece has a
`pieceType` and tracks whether it has moved (for castling and pawn
double-move eligibility).

**White King at e1 (position [4, 0, 0]):**

```json
{
  "id": "chess-wk",
  "regionId": "region-chess-001",
  "name": "White King",
  "position": [4, 0, 0],
  "visualRef": "blob-chess-white-king",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "pieceType": "king",
    "color": "white",
    "hasMoved": false
  }
}
```

**Black Pawn at d7 (position [3, 0, 6]):**

```json
{
  "id": "chess-bp4",
  "regionId": "region-chess-001",
  "name": "Black Pawn",
  "position": [3, 0, 6],
  "visualRef": "blob-chess-black-pawn",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:bob@example.com",
  "customProperties": {
    "pieceType": "pawn",
    "color": "black",
    "hasMoved": false
  }
}
```

### Interaction Model

Drag-and-drop or click-to-select, click-to-place. The simulation
layer validates all standard chess rules:

1. Piece movement constraints (rook: straight lines; bishop:
   diagonals; queen: both; knight: L-shape; king: one square; pawn:
   forward with capture diagonals).
2. Cannot move into check.
3. Cannot move a pinned piece if it exposes the king.
4. Captures destroy the captured piece.

**Special moves handled by the simulation layer:**

- **Castling:** Player moves the king two squares toward a rook. The
  simulation validates neither piece has moved, no squares are under
  attack, and the path is clear. On success, the simulation moves
  both the king and the rook in a single state update.
- **En passant:** When a pawn advances two squares and lands beside an
  opponent's pawn, the opponent may capture it as if it moved one
  square. The simulation tracks the en passant target square in the
  game state.
- **Pawn promotion:** When a pawn reaches the back rank, the
  simulation sets `phase: "promotion"` and waits for the player to
  choose a piece type. The client renders a promotion picker. The
  player sends an interaction event with `action: "promote"` and
  `data: { "pieceType": "queen" }`. The simulation updates the pawn's
  `pieceType`, `visualRef`, and `name`.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 23,
    "phase": "playing",
    "winner": null,
    "check": false,
    "enPassantTarget": null,
    "halfMoveClock": 4,
    "whiteCanCastleKingside": true,
    "whiteCanCastleQueenside": false,
    "blackCanCastleKingside": true,
    "blackCanCastleQueenside": true,
    "moveHistory": ["e4", "e5", "Nf3", "Nc6", "Bb5"]
  }
}
```

**Game end conditions:** Checkmate (`winner` set, `phase: "finished"`),
stalemate (`phase: "draw"`, `drawReason: "stalemate"`), draw by
agreement, threefold repetition, fifty-move rule, or insufficient
material.

### Hidden Information

None.

### Time Controls

Chess clocks are tracked in the game state object:

```json
{
  "customProperties": {
    "timeControl": "5+3",
    "whiteTimeMs": 245000,
    "blackTimeMs": 298000,
    "lastMoveTimestamp": "2026-06-06T14:23:45Z"
  }
}
```

The simulation layer decrements the active player's clock between
moves and enforces time forfeit. The client renders a countdown timer
from these values, interpolating locally between state updates.


---

## 5. Sorry!

A race-to-home board game with cards instead of dice. 2-4 players
move pawns around a shared track with slides and safe zones.

### Region

The board is a square track with per-player start/home areas. The
coordinate system maps the track to a perimeter path.

```json
{
  "id": "region-sorry-001",
  "name": "Sorry!: Alice, Bob, Carol, Dave",
  "bounds": { "min": [0, 0, 0], "max": [16, 0, 16] },
  "viewHint": "2d-topdown",
  "spawnPosition": [8, 0, -1],
  "simulationUri": "wss://sim.example.com/games/sorry/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic",
    "gridVisible": false,
    "trackPositions": 60,
    "slidePositions": [[1, 4], [9, 12], [16, 19], [24, 27]]
  }
}
```

### Pieces

4 pawns per player, starting in each player's Start area.

```json
{
  "id": "sorry-pawn-r1",
  "regionId": "region-sorry-001",
  "name": "Red Pawn 1",
  "position": [1, 0, 0],
  "visualRef": "blob-pawn-red",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "color": "red",
    "trackPosition": -1,
    "inStart": true,
    "inHome": false,
    "inSafeZone": false
  }
}
```

### Interaction Model

Card-based movement. On a player's turn, the simulation draws a card
and presents it as a SceneObject visible to all:

```json
{
  "id": "sorry-card-current",
  "regionId": "region-sorry-001",
  "name": "Drawn Card",
  "position": [8, 0, 8],
  "visualRef": "blob-sorry-card-7",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "cardValue": 7,
    "description": "Move one pawn forward 7 spaces, or split the move between two pawns"
  }
}
```

The player then clicks a pawn to move. For cards allowing split
moves (7, 11), the simulation enters a `"split"` sub-phase. The
**Sorry!** card lets a player swap positions with an opponent's pawn,
sending the opponent back to Start.

**Key card rules the simulation enforces:**
- **1:** Enter from Start, or move forward 1
- **2:** Enter from Start, or move forward 2; draw again
- **3:** Move forward 3
- **4:** Move backward 4
- **5:** Move forward 5
- **7:** Move forward 7, or split between two pawns
- **8:** Move forward 8
- **10:** Move forward 10, or backward 1
- **11:** Move forward 11, or swap with an opponent's pawn
- **12:** Move forward 12
- **Sorry!:** Take a pawn from Start and swap with any opponent pawn

**Slides:** When a pawn lands on the start of an opponent's slide,
it slides to the end, bumping any pawns in the way back to their
Start. The simulation handles this automatically on landing.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 15,
    "phase": "choose-pawn",
    "winner": null,
    "currentCard": 7,
    "drawAgain": false,
    "pawnsHome": {
      "user:alice@example.com": 1,
      "user:bob@example.com": 0,
      "user:carol@example.com": 2,
      "user:dave@example.com": 0
    }
  }
}
```

First player to get all 4 pawns Home wins.

### Hidden Information

The draw pile is hidden. Only the currently drawn card is visible.
Players do not see upcoming cards.


---

## 6. Monopoly

A property-trading board game with dice, money, and negotiation.
2-8 players.

### Region

The board is a 40-space perimeter track. The region uses a larger
coordinate space to accommodate the property cards and community
areas in the center.

```json
{
  "id": "region-monopoly-001",
  "name": "Monopoly: Alice, Bob, Carol",
  "bounds": { "min": [0, 0, 0], "max": [11, 0, 11] },
  "viewHint": "2d-topdown",
  "spawnPosition": [5.5, 0, -1],
  "simulationUri": "wss://sim.example.com/games/monopoly/001",
  "chatId": "chat-monopoly-001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "classic",
    "gridVisible": false,
    "currency": "$"
  }
}
```

### Pieces

**Player token:**

```json
{
  "id": "monopoly-token-alice",
  "regionId": "region-monopoly-001",
  "name": "Top Hat",
  "position": [0, 0, 0],
  "visualRef": "blob-token-tophat",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "tokenType": "tophat",
    "boardPosition": 0,
    "inJail": false,
    "jailTurns": 0
  }
}
```

**Property deed (visible in center area when owned):**

```json
{
  "id": "monopoly-prop-boardwalk",
  "regionId": "region-monopoly-001",
  "name": "Boardwalk",
  "position": [8, 0, 5],
  "visualRef": "blob-deed-boardwalk",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "propertyName": "Boardwalk",
    "colorGroup": "dark-blue",
    "purchasePrice": 400,
    "rent": [50, 200, 600, 1400, 1700, 2000],
    "houseCost": 200,
    "houses": 0,
    "mortgaged": false,
    "ownedBy": "user:alice@example.com"
  }
}
```

**Dice (pair, centered on board):**

```json
{
  "id": "monopoly-dice",
  "regionId": "region-monopoly-001",
  "name": "Dice",
  "position": [5.5, 0, 5.5],
  "visualRef": "blob-dice-3-4",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "die1": 3,
    "die2": 4,
    "doubles": false
  }
}
```

### Interaction Model

**Turn sequence:**

1. Player clicks dice to roll (`"activate"` interaction). Simulation
   generates random values, updates dice visual, and advances the
   player's token along the track.
2. **Landing on unowned property:** Simulation sets
   `phase: "buy-or-auction"`. Player clicks "Buy" or "Auction" button
   (rendered as interactable SceneObjects in the center).
3. **Landing on owned property:** Simulation auto-deducts rent from
   the player's balance and credits the owner.
4. **Community Chest / Chance:** Simulation draws a card, displays it
   as a temporary SceneObject, applies the effect, then destroys the
   card object.
5. **Jail:** Three doubles in a row sends the player to Jail. Player
   can pay bail, use a Get Out of Jail Free card, or try to roll
   doubles on subsequent turns.

**Trading:** Players initiate trades via interaction events on each
other's tokens. The simulation enters a `"trade"` sub-phase with a
trade proposal SceneObject showing the offered and requested items.
Both players must `"activate"` an Accept button to complete the trade.

**Building:** Player `"activate"`s a property deed when they own the
full color group. Simulation validates and updates `houses` count and
`visualRef` to show houses/hotel.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 34,
    "phase": "roll",
    "winner": null,
    "doublesCount": 0,
    "balances": {
      "user:alice@example.com": 1250,
      "user:bob@example.com": 890,
      "user:carol@example.com": 1540
    },
    "bankruptPlayers": [],
    "freeParking": 0,
    "housesAvailable": 28,
    "hotelsAvailable": 12
  }
}
```

### Hidden Information

Minimal. In standard rules, all property ownership and balances are
public. Chance and Community Chest decks are hidden; only the drawn
card is revealed.


---

## 7. Battleship

A two-player hidden-information game. Each player places ships on a
private grid and takes turns guessing coordinates to find and sink
the opponent's ships.

### Region

Two 10x10 grids side by side: the player's own grid (showing their
ships and opponent's guesses) and the opponent's grid (showing the
player's guesses).

```json
{
  "id": "region-battleship-001",
  "name": "Battleship: Alice vs Bob",
  "bounds": { "min": [0, 0, 0], "max": [22, 0, 10] },
  "viewHint": "2d-topdown",
  "spawnPosition": [11, 0, -1],
  "simulationUri": "wss://sim.example.com/games/battleship/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "ocean",
    "gridVisible": true,
    "gridLabels": true
  }
}
```

The left 10x10 area (x: 0-9) is the player's own grid. The right
10x10 area (x: 12-21, with a gap at x: 10-11) is the tracking grid
for guesses against the opponent. Each player sees their own view;
the server returns different SceneObject sets per player using
visibility filtering.

### Pieces

**Ship (visible only to owner):**

```json
{
  "id": "bs-ship-alice-carrier",
  "regionId": "region-battleship-001",
  "name": "Carrier",
  "position": [2, 0, 3],
  "visualRef": "blob-ship-carrier-h",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "shipType": "carrier",
    "length": 5,
    "orientation": "horizontal",
    "hits": [false, false, false, false, false],
    "sunk": false
  }
}
```

This object is only returned by the server to Alice. Bob's
`SceneObject/get` does not include it until the ship is sunk, at
which point the server reveals it on Bob's tracking grid.

**Hit marker (visible to both):**

```json
{
  "id": "bs-hit-014",
  "regionId": "region-battleship-001",
  "name": "Hit",
  "position": [3, 0, 3],
  "visualRef": "blob-marker-hit",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "result": "hit",
    "guessedBy": "user:bob@example.com",
    "col": 3,
    "row": 3
  }
}
```

**Miss marker:**

```json
{
  "id": "bs-miss-015",
  "regionId": "region-battleship-001",
  "name": "Miss",
  "position": [15, 0, 7],
  "visualRef": "blob-marker-miss",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "result": "miss",
    "guessedBy": "user:alice@example.com",
    "col": 3,
    "row": 7
  }
}
```

### Interaction Model

**Setup phase:** Each player drag-and-drops ships onto their grid.
Right-click or `"activate"` rotates between horizontal and vertical.
The simulation validates no overlapping ships. When both players
confirm placement (`"activate"` on a Ready button), the game begins.

**Play phase:** Player clicks a cell on the opponent's tracking grid
(right side). The simulation checks the opponent's ship positions and
creates a hit or miss marker on both grids. On a hit, the
corresponding ship's `hits` array is updated. When all cells of a
ship are hit, the ship is `sunk: true` and revealed on the
opponent's tracking grid.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 22,
    "phase": "playing",
    "winner": null,
    "aliceShipsRemaining": 4,
    "bobShipsRemaining": 3,
    "setupComplete": {
      "user:alice@example.com": true,
      "user:bob@example.com": true
    }
  }
}
```

### Hidden Information

**Critical.** This is Battleship's core mechanic. The server should
not return ship positions to the opponent until the ship is sunk. The
visibility filtering pattern:

- `SceneObject/get` for Alice returns: Alice's ships (all), Alice's
  hit/miss markers received, Alice's guesses on Bob's grid, and any
  of Bob's sunk ships.
- Bob's un-sunk ships are never returned to Alice.
- The server stores the complete state; each player sees only their
  authorized view.


---

## 8. Stratego

A two-player hidden-information strategy game on a 10x10 board.
Players place 40 pieces with hidden ranks. Attacking reveals both
pieces; the higher rank wins.

### Region

```json
{
  "id": "region-stratego-001",
  "name": "Stratego: Alice vs Bob",
  "bounds": { "min": [0, 0, 0], "max": [10, 0, 10] },
  "viewHint": "2d-topdown",
  "spawnPosition": [5, 0, -1],
  "simulationUri": "wss://sim.example.com/games/stratego/001",
  "accessPolicy": "invite",
  "environment": {
    "boardTheme": "military",
    "gridVisible": true,
    "lakes": [[2, 4], [3, 4], [2, 5], [3, 5], [6, 4], [7, 4], [6, 5], [7, 5]]
  }
}
```

Two 2x2 lake areas in the center are impassable.

### Pieces

40 pieces per player. Each player's pieces show their rank to the
owner and a generic back to the opponent.

**Owner's view (Alice sees her own Marshal):**

```json
{
  "id": "stratego-a-marshal",
  "regionId": "region-stratego-001",
  "name": "Marshal",
  "position": [4, 0, 0],
  "visualRef": "blob-stratego-red-marshal",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "rank": 10,
    "rankName": "Marshal",
    "color": "red",
    "movable": true
  }
}
```

**Opponent's view (Bob sees a generic red piece):**

The server returns the same object to Bob but with `visualRef`
replaced by `"blob-stratego-red-back"` and `customProperties`
redacted to `{ "color": "red" }`. The rank is hidden.

### Interaction Model

**Setup phase:** Each player arranges their 40 pieces in their half
of the board (rows 0-3 for one player, rows 6-9 for the other). Drag
and drop to position. Confirm when ready.

**Play phase:** Drag a piece to an adjacent square (one space,
orthogonal, no diagonals). If the destination contains an opponent
piece, combat occurs:

1. Both pieces are briefly revealed (the simulation updates both
   `visualRef` values to show their faces).
2. Higher rank wins. The losing piece is destroyed.
3. Equal rank: both pieces are destroyed.
4. Special cases:
   - **Spy (rank 1) attacks Marshal (rank 10):** Spy wins.
   - **Miner (rank 3) attacks Bomb:** Miner wins (defuses the bomb).
   - **Any other piece attacks Bomb:** Attacker is destroyed.
   - **Flag:** Cannot move. If captured, the capturing player wins.

After combat resolution, both pieces' true identities are known to
both players (the revealed `visualRef` persists in the destroyed
piece's position briefly before removal, or the surviving piece
retains its revealed visual).

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 31,
    "phase": "playing",
    "winner": null,
    "alicePiecesRemaining": 34,
    "bobPiecesRemaining": 36,
    "revealedPieces": ["stratego-a-scout2", "stratego-b-miner1"]
  }
}
```

### Hidden Information

**Critical.** Each player's piece ranks are hidden from the opponent
until revealed through combat. The server uses per-player visibility
filtering identical to the Battleship pattern: the response to
`SceneObject/get` replaces `visualRef` and redacts `customProperties`
for opponent pieces that have not been revealed through combat.


---

## 9. Catan

A 3-6 player resource-management and trading game on a hex grid.
Players build settlements, cities, and roads to earn victory points.

### Region

The Catan board is a hex grid, typically 19 hexes arranged in a
roughly circular pattern. The coordinate system uses axial hex
coordinates mapped onto the X/Z plane.

```json
{
  "id": "region-catan-001",
  "name": "Catan: Alice, Bob, Carol",
  "bounds": { "min": [-3, 0, -3], "max": [3, 0, 3] },
  "viewHint": "2d-topdown",
  "spawnPosition": [0, 0, -4],
  "simulationUri": "wss://sim.example.com/games/catan/001",
  "chatId": "chat-catan-001",
  "accessPolicy": "invite",
  "environment": {
    "boardLayout": "standard",
    "hexCoordinateSystem": "axial",
    "portPositions": [[0, -3], [2, -3], [3, -1], [3, 1], [1, 2], [-1, 3], [-3, 2], [-3, 0], [-2, -2]]
  }
}
```

### Pieces

**Hex tile (terrain + number token):**

```json
{
  "id": "catan-hex-00",
  "regionId": "region-catan-001",
  "name": "Hills (5)",
  "position": [0, 0, 0],
  "visualRef": "blob-hex-hills",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "terrain": "hills",
    "resource": "brick",
    "numberToken": 5,
    "hasRobber": false
  }
}
```

**Settlement (placed at hex vertex):**

```json
{
  "id": "catan-settle-a1",
  "regionId": "region-catan-001",
  "name": "Alice's Settlement",
  "position": [0.5, 0, -0.87],
  "visualRef": "blob-settlement-red",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "buildingType": "settlement",
    "color": "red",
    "vertexId": "v-0-0-NE",
    "victoryPoints": 1
  }
}
```

**Road (placed on hex edge):**

```json
{
  "id": "catan-road-a1",
  "regionId": "region-catan-001",
  "name": "Alice's Road",
  "position": [0.75, 0, -0.43],
  "visualRef": "blob-road-red",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "color": "red",
    "edgeId": "e-0-0-E"
  }
}
```

**City (upgraded settlement):**

Same position as the settlement it replaces. The simulation destroys
the settlement and creates a city with `victoryPoints: 2` and a
different `visualRef`.

**Robber:**

```json
{
  "id": "catan-robber",
  "regionId": "region-catan-001",
  "name": "Robber",
  "position": [0, 0, 0],
  "visualRef": "blob-robber",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "onHex": "catan-hex-00"
  }
}
```

### Interaction Model

**Turn sequence:**

1. **Roll dice:** Player `"activate"`s the dice object. Simulation
   generates a roll (2-12). All hexes with matching number tokens
   produce resources for players with adjacent settlements/cities.
2. **Roll 7 / Robber:** Players with more than 7 resource cards must
   discard half (simulation enters `"discard"` sub-phase). The active
   player then moves the robber to a hex (`"grab"` + `"release"` on
   the robber) and steals one random resource from an adjacent player.
3. **Build:** Player clicks a valid vertex (settlement), edge (road),
   or existing settlement (upgrade to city). The simulation validates
   resources and placement rules (settlements must be 2+ edges apart,
   roads must connect to existing network).
4. **Buy development card:** Player `"activate"`s the dev-card deck.
   A card is added to their hand (hidden from opponents).
5. **Trade:** Player proposes a trade via an interaction event. The
   simulation creates a trade-offer SceneObject visible to all
   players. Other players `"activate"` Accept or Decline buttons.
   Port trades (2:1 or 3:1) are handled automatically when the player
   has a settlement on a port.

**Development cards (hidden hand):**

```json
{
  "id": "catan-devcard-a3",
  "regionId": "region-catan-001",
  "name": "Development Card",
  "position": [0, 0, -3.5],
  "visualRef": "blob-devcard-knight",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "cardType": "knight",
    "playable": true,
    "turnAcquired": 12
  }
}
```

Only visible to Alice. Other players see the count of dev cards held,
not their types.

### Game State

The game state object is returned with per-player redaction. The
example below shows what Alice receives: her own full resource
breakdown and only card totals for opponents.

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "turnNumber": 28,
    "phase": "main",
    "winner": null,
    "lastRoll": [3, 4],
    "resources": {
      "user:alice@example.com": { "brick": 2, "lumber": 3, "wool": 1, "grain": 0, "ore": 2 },
      "user:bob@example.com": { "_total": 8 },
      "user:carol@example.com": { "_total": 6 }
    },
    "victoryPoints": {
      "user:alice@example.com": 5,
      "user:bob@example.com": 4,
      "user:carol@example.com": 3
    },
    "longestRoad": "user:alice@example.com",
    "largestArmy": null,
    "knightsPlayed": {
      "user:alice@example.com": 1,
      "user:bob@example.com": 0,
      "user:carol@example.com": 2
    },
    "devCardsRemaining": 20,
    "setupPhase": false
  }
}
```

The server maintains the full resource breakdown internally. Each
player's response includes their own full breakdown and only a
`_total` count for all other players.

### Hidden Information

- **Resource cards:** Each player's resource hand is private. Other
  players see only the total count (e.g., "Alice has 8 cards").
- **Development cards:** Hidden until played. The server uses
  per-player visibility filtering: Alice's dev cards show their type
  to Alice and are hidden from other players.
- **Victory point dev cards:** Hidden until the player reaches 10+
  points and declares victory.

### Win Condition

First player to reach 10 victory points (from settlements, cities,
longest road, largest army, and victory point development cards) wins.
The simulation detects this after each build/play action and sets
`phase: "finished"` with the winner.


---

## 10. Pool (Billiards)

Eight-ball pool on a standard table. A turn-based competitive game
with real-time physics: each player takes a shot by specifying an
angle and power; the simulation layer runs the physics and reports
where all balls come to rest.

Pool is the canonical example of a **turn-based + physics hybrid**:
the game state is discrete (whose turn, which balls are pocketed),
but each action triggers continuous physics that the simulation
layer must resolve before the next turn can begin.

### Region

A standard pool table is roughly 2.54 m × 1.27 m (9-foot table).
One unit = one meter.

```json
{
  "id": "region-pool-001",
  "name": "Pool Table: Alice vs Bob",
  "bounds": { "min": [0, 0, 0], "max": [2.54, 0, 1.27] },
  "viewHint": "2d-topdown",
  "spawnPosition": [1.27, 0, -0.3],
  "simulationUri": "wss://sim.example.com/games/pool/001",
  "accessPolicy": "invite",
  "environment": {
    "tableClothColor": "green",
    "gridVisible": false,
    "feltFriction": 0.98,
    "cushionRestitution": 0.75
  }
}
```

The `simulationUri` is required. Pool physics (cushion bounce,
ball-ball collisions, pocket detection) is not expressible as JMAP
method calls alone; the simulation layer handles all of it.

### Objects

**Cue ball:**

```json
{
  "id": "pool-ball-cue",
  "regionId": "region-pool-001",
  "name": "Cue Ball",
  "position": [0.635, 0, 0.953],
  "visualRef": "blob-pool-ball-cue",
  "visualType": "image/svg+xml",
  "physicsMode": "dynamic",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "ballNumber": 0,
    "ballType": "cue",
    "pocketed": false
  }
}
```

**Numbered ball (solid or stripe):**

```json
{
  "id": "pool-ball-07",
  "regionId": "region-pool-001",
  "name": "7 Ball",
  "position": [1.524, 0, 0.635],
  "visualRef": "blob-pool-ball-07",
  "visualType": "image/svg+xml",
  "physicsMode": "dynamic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "ballNumber": 7,
    "ballType": "stripe",
    "pocketed": false
  }
}
```

**Eight ball:**

```json
{
  "id": "pool-ball-08",
  "regionId": "region-pool-001",
  "name": "8 Ball",
  "position": [1.651, 0, 0.635],
  "visualRef": "blob-pool-ball-08",
  "visualType": "image/svg+xml",
  "physicsMode": "dynamic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "ballNumber": 8,
    "ballType": "eight",
    "pocketed": false
  }
}
```

**Table (rails, cushions, and bed — static collider):**

```json
{
  "id": "pool-table-001",
  "regionId": "region-pool-001",
  "name": "Pool Table",
  "position": [1.27, 0, 0.635],
  "visualRef": "blob-pool-table",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "pockets": [
      [0.0,  0, 0.0],  [1.27, 0, 0.0],  [2.54, 0, 0.0],
      [0.0,  0, 1.27], [1.27, 0, 1.27], [2.54, 0, 1.27]
    ],
    "pocketRadius": 0.057
  }
}
```

The table object is a single static collider representing rails,
cushions, and the playing surface boundary. The simulation layer
uses the pocket positions and radius to detect when a ball center
crosses a pocket threshold.

All 15 numbered balls plus the cue ball use `physicsMode: "dynamic"`.
The table uses `physicsMode: "static"`. The simulation applies
cushion restitution and felt friction from `environment` when
resolving collisions.

### Interaction Model

Pool uses a custom `"shoot"` action rather than grab/release.
The player cannot drag the cue ball directly; instead, they aim
and specify power via the `"shoot"` interaction on the cue ball.

**Player shoots:**

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-pool-001",
  "objectId": "pool-ball-cue",
  "userId": "user:alice@example.com",
  "action": "shoot",
  "data": {
    "angleRad": 1.047,
    "power": 0.65
  }
}
```

`angleRad` is the shot direction in radians, measured clockwise
from the positive X axis in the 2D top-down plane. `power` is
0.0–1.0 representing normalized cue force; the simulation maps
this to a physical impulse.

The simulation layer on receiving a `"shoot"` event:

1. Validates that it is this player's turn and `gamePhase` is
   `"break"` or `"play"` or `"8-ball"`.
2. Applies the impulse to the cue ball.
3. Runs the physics loop at 60+ Hz until all balls are at rest
   (per-ball velocity below threshold).
4. Evaluates the outcome: balls pocketed, cue ball pocketed (foul),
   failure to contact own group first (foul), no rail contact
   after hit (foul).
5. Updates all ball positions via `SceneObject/set`.
6. Updates the game state object: pocketed sets, fouls, whose turn.

**Server response after a shot:**

```json
{
  "methodResponses": [[
    "SceneObject/set",
    {
      "accountId": "u1",
      "updated": {
        "pool-ball-cue": { "position": [0.82, 0, 0.71] },
        "pool-ball-03":  { "position": [1.27, 0, 0.98] },
        "pool-ball-07":  {
          "position": [0.0, 0, 0.0],
          "visible": false,
          "customProperties/pocketed": true
        }
      }
    },
    "0"
  ]]
}
```

Pocketed balls are moved to a sentinel position and set
`visible: false`. The simulation retains the objects so they can
be restored on a cue-ball-pocketed foul.

### Game State

```json
{
  "customProperties": {
    "gamePhase": "play",
    "currentPlayer": "user:alice@example.com",
    "playerAssignments": {
      "user:alice@example.com": "solids",
      "user:bob@example.com": "stripes"
    },
    "ballsPocketed": {
      "user:alice@example.com": [1, 2, 4],
      "user:bob@example.com": [9, 10]
    },
    "eightBallPocketed": false,
    "fouls": {
      "user:alice@example.com": 0,
      "user:bob@example.com": 1
    },
    "winner": null
  }
}
```

`gamePhase` values:
- `"break"` — opening shot; ball-type assignments not yet made
- `"play"` — normal play; each player pockets their assigned group
- `"8-ball"` — both players have cleared their group; next legal
  8-ball pocket wins
- `"finished"` — game over; `winner` is set

Ball-type assignments are absent from `playerAssignments` until
the first legal ball is pocketed after the break. The simulation
sets `"solids"` or `"stripes"` at that point.

### Tick Rate and Simulation Activity

Pool physics runs at **60+ Hz during shot resolution** but is
**idle between shots**. This is the opposite of a continuous
real-time game: the simulation layer only needs to be active for
the few seconds while balls are rolling after each shot.

A deployment MAY implement this as an on-demand simulation: the
layer wakes when a `"shoot"` interaction arrives, runs until all
balls are at rest, publishes the final positions via
`SceneObject/set`, and then suspends. This avoids holding a running
physics loop open during the indefinite time between turns.

For physics authority models, tick rate selection, and on-demand
vs. continuous simulation patterns, see the [JMAP Scene Simulation
Layer Guide](jmap-scene-simulation-guide.md).

### Hidden Information

Minimal. Pool is a fully observable game: all 16 ball positions are
visible to both players at all times. Neither player has private
information about the table state.

The only "hidden" element is the **planned shot trajectory** — the
aim line a client renders for the shooting player before they
commit. This is local client-side UI; it is never sent to the
server or the opponent. The server receives only the committed
`"shoot"` event with angle and power.


---

---

## Part II: Card Games

Card games use `viewHint: "2d-topdown"` with per-player visibility
filtering for hidden hands. Cards are SceneObjects with `ownerId`
controlling who can see the face. The server returns `visualRef`
pointing to a card-back blob for cards the requesting player is not
authorized to see.


---

## 11. Old Maid

A simple draw-and-discard matching game for 2-5 players. The goal is
to avoid being stuck with the unmatched Queen (the Old Maid).

### Region

Cards are arranged in a fan in front of each player. The center area
shows the discard pile of matched pairs.

```json
{
  "id": "region-oldmaid-001",
  "name": "Old Maid: Alice, Bob, Carol",
  "bounds": { "min": [0, 0, 0], "max": [12, 0, 10] },
  "viewHint": "2d-topdown",
  "spawnPosition": [6, 0, -1],
  "simulationUri": "wss://sim.example.com/games/oldmaid/001",
  "accessPolicy": "invite",
  "environment": {
    "tableTheme": "green-felt"
  }
}
```

### Pieces

One Queen is removed from a standard 52-card deck before dealing,
leaving 51 cards. Players remove matching pairs from their hand
during the deal, then take turns.

**Card in Alice's hand (visible to Alice, back to others):**

```json
{
  "id": "om-card-a07",
  "regionId": "region-oldmaid-001",
  "name": "Card",
  "position": [3.5, 0, 1],
  "visualRef": "blob-card-7h",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "suit": "hearts",
    "rank": "7",
    "inHand": true,
    "handOwner": "user:alice@example.com"
  }
}
```

Bob and Carol receive this object with `visualRef: "blob-card-back"`
and `customProperties` redacted to `{ "inHand": true }`.

**Matched pair in discard pile (visible to all):**

```json
{
  "id": "om-pair-03",
  "regionId": "region-oldmaid-001",
  "name": "Matched Pair: 7s",
  "position": [6, 0, 5],
  "visualRef": "blob-card-pair-7",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "rank": "7",
    "matchedBy": "user:alice@example.com"
  }
}
```

### Interaction Model

On a player's turn, they draw one card from the next player's hand.
The opponent's cards are displayed face-down in a fan; the active
player clicks one to draw it. The simulation:

1. Transfers the card to the active player's hand (changes `ownerId`
   and `handOwner`, updates `position` to the player's hand fan).
2. Checks for a new pair. If the drawn card matches a card in the
   player's hand, both are removed and added to the discard pile.
3. Advances the turn.

Players who empty their hand are safe and out of the game. The last
player holding the unmatched Queen loses.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:bob@example.com",
    "drawFrom": "user:carol@example.com",
    "phase": "playing",
    "loser": null,
    "handSizes": {
      "user:alice@example.com": 0,
      "user:bob@example.com": 5,
      "user:carol@example.com": 4
    },
    "eliminated": ["user:alice@example.com"],
    "pairsDiscarded": 22
  }
}
```

### Hidden Information

Each player's hand is hidden from all other players. Only the count
of cards in each hand is public.


---

## 12. Go Fish

A draw-and-ask matching game for 2-6 players. Players collect sets of
four matching cards by asking opponents for specific ranks.

### Region

```json
{
  "id": "region-gofish-001",
  "name": "Go Fish: Alice, Bob, Carol",
  "bounds": { "min": [0, 0, 0], "max": [12, 0, 10] },
  "viewHint": "2d-topdown",
  "spawnPosition": [6, 0, -1],
  "simulationUri": "wss://sim.example.com/games/gofish/001",
  "chatId": "chat-gofish-001",
  "accessPolicy": "invite",
  "environment": {
    "tableTheme": "green-felt"
  }
}
```

### Pieces

Standard 52-card deck. 7 cards dealt to each player (5 cards in 4+
player games). Remaining cards form the draw pile (fish pond).

**Card in hand (owner-visible):**

```json
{
  "id": "gf-card-a03",
  "regionId": "region-gofish-001",
  "name": "Card",
  "position": [2.5, 0, 1],
  "visualRef": "blob-card-qs",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "suit": "spades",
    "rank": "Q",
    "inHand": true,
    "handOwner": "user:alice@example.com"
  }
}
```

**Draw pile (face-down, centered):**

```json
{
  "id": "gf-drawpile",
  "regionId": "region-gofish-001",
  "name": "Fish Pond",
  "position": [6, 0, 5],
  "visualRef": "blob-card-pile",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "cardsRemaining": 28
  }
}
```

**Completed set (visible to all):**

```json
{
  "id": "gf-set-a1",
  "regionId": "region-gofish-001",
  "name": "Set of Queens",
  "position": [2, 0, 3],
  "visualRef": "blob-card-set-Q",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "rank": "Q",
    "completedBy": "user:alice@example.com"
  }
}
```

### Interaction Model

On a player's turn:

1. **Ask:** Player clicks one of their own cards to select the rank,
   then clicks an opponent's avatar (or a target-selection UI object)
   to ask: "Do you have any Queens?" This is sent as an interaction
   event with `action: "ask"` and `data: { "rank": "Q", "target":
   "user:bob@example.com" }`.

2. **Opponent has the rank:** The simulation transfers all matching
   cards from the opponent's hand to the asking player. The transfer
   is animated (cards slide across the table). A system message is
   posted to the bound Chat: "Alice asked Bob for Queens and got 2."
   The asking player gets another turn.

3. **"Go Fish!":** If the opponent has no matching cards, the
   simulation draws a card from the fish pond and adds it to the
   asking player's hand. If the drawn card matches the asked rank,
   the player reveals it ("I fished my wish!") and gets another
   turn. Otherwise, the turn passes.

4. **Complete set:** When a player collects all four cards of a rank,
   the simulation removes them from the hand and creates a completed
   set SceneObject on the table.

### Game State

```json
{
  "customProperties": {
    "currentTurn": "user:alice@example.com",
    "phase": "ask",
    "winner": null,
    "handSizes": {
      "user:alice@example.com": 6,
      "user:bob@example.com": 4,
      "user:carol@example.com": 7
    },
    "completedSets": {
      "user:alice@example.com": 2,
      "user:bob@example.com": 1,
      "user:carol@example.com": 0
    },
    "drawPileRemaining": 21,
    "lastAsk": {
      "asker": "user:alice@example.com",
      "target": "user:bob@example.com",
      "rank": "Q",
      "result": "got-2"
    }
  }
}
```

Game ends when all 13 sets are completed. Most sets wins.

### Hidden Information

Each player's hand is private. The draw pile is face-down. Completed
sets and hand sizes are public. The Chat integration posts all ask
results publicly, building a social deduction element (remembering
who asked for what).


---

## 13. Solitaire (Klondike)

The classic single-player card game. Seven tableau columns, four
foundation piles, one stock, one waste.

### Region

```json
{
  "id": "region-solitaire-001",
  "name": "Klondike Solitaire",
  "bounds": { "min": [0, 0, 0], "max": [7, 0, 12] },
  "viewHint": "2d-topdown",
  "spawnPosition": [3.5, 0, -1],
  "simulationUri": "wss://sim.example.com/games/solitaire/001",
  "accessPolicy": "invite",
  "environment": {
    "tableTheme": "green-felt",
    "drawMode": "draw-3"
  }
}
```

`accessPolicy: "invite"` with no other users invited creates a
single-player region. Spectators may join by being added to the
invite list.

### Pieces

52 cards. The tableau is dealt face-down with the top card face-up in
each column (1 card in column 1, 2 in column 2, ... 7 in column 7).

**Face-up tableau card:**

```json
{
  "id": "sol-card-22",
  "regionId": "region-solitaire-001",
  "name": "7 of Hearts",
  "position": [3, 0, 5],
  "visualRef": "blob-card-7h",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "suit": "hearts",
    "rank": "7",
    "color": "red",
    "faceUp": true,
    "pile": "tableau",
    "column": 3,
    "stackIndex": 4
  }
}
```

**Face-down tableau card:**

```json
{
  "id": "sol-card-18",
  "regionId": "region-solitaire-001",
  "name": "Hidden Card",
  "position": [3, 0, 4.5],
  "visualRef": "blob-card-back",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "faceUp": false,
    "pile": "tableau",
    "column": 3,
    "stackIndex": 3
  }
}
```

Face-down cards use the card-back visual but are still visible
objects (you see the back). They become interactable when they are
the top face-down card in their column (clicking flips them face-up).

**Stock pile (top-left):**

```json
{
  "id": "sol-stock",
  "regionId": "region-solitaire-001",
  "name": "Stock",
  "position": [0, 0, 0],
  "visualRef": "blob-card-back",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "pile": "stock",
    "cardsRemaining": 24
  }
}
```

**Foundation pile (one per suit, top-right area):**

```json
{
  "id": "sol-foundation-hearts",
  "regionId": "region-solitaire-001",
  "name": "Hearts Foundation",
  "position": [4, 0, 0],
  "visualRef": "blob-foundation-empty-hearts",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "pile": "foundation",
    "suit": "hearts",
    "topRank": null
  }
}
```

### Interaction Model

**Drag-and-drop** for all card movement:

- **Tableau to tableau:** Drag a face-up card (and all cards below
  it) to another column. The simulation validates: target card must
  be opposite color and one rank higher, or the column must be empty
  and the dragged card must be a King.
- **Tableau/waste to foundation:** Drag a card to the appropriate
  suit's foundation. Must be the next sequential rank (Ace, then 2,
  then 3, ..., King).
- **Stock click:** Clicking the stock flips 1 or 3 cards (per
  `drawMode`) from the stock to the waste pile. When the stock is
  empty, clicking recycles the waste pile back to the stock.
- **Flip face-down card:** When a face-down card becomes the top card
  in its column (all face-up cards above it have been moved), clicking
  it flips it face-up.

**Auto-complete:** When all cards are face-up and no more tableau
moves are needed, the simulation can auto-complete by moving all
cards to foundations in sequence.

### Game State

```json
{
  "customProperties": {
    "phase": "playing",
    "winner": null,
    "moveCount": 42,
    "score": 315,
    "elapsedSeconds": 187,
    "stockCycles": 1,
    "foundationCounts": {
      "hearts": 3,
      "diamonds": 5,
      "clubs": 1,
      "spades": 0
    },
    "undoAvailable": true
  }
}
```

Win condition: all 52 cards in the four foundation piles.

### Hidden Information

Face-down cards in the tableau and stock pile have hidden identity.
Since it is single-player, this is not per-player visibility
filtering but per-card state: `faceUp: false` means the client
renders the back. The simulation knows all card positions and
enforces valid moves.


---

## Part III: Arcade and Video Games

These games go beyond turn-based play. They use the Scene simulation
layer (`simulationUri`) at high frequency (10-60 Hz) for real-time
position updates, physics, and collision detection. The JMAP state
layer provides snapshots for persistence and reconnection; the
simulation layer provides the real-time game loop.

**Key difference from board games:** Board games can operate entirely
through JMAP method calls (`SceneObject/set`) for moves, with the
simulation layer enforcing rules on each move. Video games require
the simulation layer to run a continuous game loop -- processing
input, updating physics, detecting collisions, and broadcasting
state -- at frame rate. JMAP snapshots (`SceneAvatar/get`,
`SceneObject/get`) serve as periodic checkpoints. For architecture
patterns, authority models, tick rate selection, state reconciliation,
and transport options for the simulation layer, see the [JMAP Scene
Simulation Layer Guide](jmap-scene-simulation-guide.md).

**Dimensionality:** All games using a first-person perspective use
`viewHint: "3d"`, but they fall into distinct categories:

- **2.5D** (Doom, Duke Nukem 3D): Sector-based geometry, auto
  vertical aim, billboard sprites. Environment sets
  `renderMode: "2.5d"`.
- **True 3D** (Quake, Minecraft): Full mesh geometry, free mouselook,
  polygon models. Environment sets `renderMode: "3d"` or omits it
  (3D is the default).
- **6DOF** (Descent): True 3D plus zero gravity and full roll
  freedom. Environment sets `gravity: 0` and `sixDOF: true`.


---

## 14. Pitfall! (Side-Scroller / Platformer)

A side-scrolling platformer in the style of Atari's Pitfall!. The
player runs, jumps, swings on vines, and avoids hazards. This is the
canonical `viewHint: "2d-side"` game.

### Region

A long horizontal world. The camera follows the player's avatar,
showing a viewport-sized window into the region.

```json
{
  "id": "region-pitfall-001",
  "name": "Jungle Run",
  "bounds": { "min": [0, 0, 0], "max": [2048, 16, 0] },
  "viewHint": "2d-side",
  "spawnPosition": [2, 4, 0],
  "simulationUri": "wss://sim.example.com/games/pitfall/001",
  "accessPolicy": "public",
  "environment": {
    "gravity": 9.81,
    "theme": "jungle",
    "scrollDirection": "horizontal",
    "cameraTracking": "follow-player"
  }
}
```

With `viewHint: "2d-side"`, X is horizontal (left/right), Y is
vertical (up/down, with ground at Y=0), and Z is unused (depth,
always 0). The client renders an orthographic camera looking along
the Z axis from the side.

### Pieces

**Terrain (static, non-interactable):**

```json
{
  "id": "pit-ground-001",
  "regionId": "region-pitfall-001",
  "name": "Ground Segment",
  "position": [0, 0, 0],
  "visualRef": "blob-terrain-jungle-ground",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "scale": [64, 1, 2],
  "customProperties": {
    "terrainType": "solid",
    "collisionBox": [64, 1, 2]
  }
}
```

**Pit (gap in ground):**

```json
{
  "id": "pit-hazard-003",
  "regionId": "region-pitfall-001",
  "name": "Pit",
  "position": [45, 0, 0],
  "visualRef": "blob-terrain-pit",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "scale": [4, 1, 2],
  "customProperties": {
    "terrainType": "hazard",
    "damage": "instant-death"
  }
}
```

**Vine (swingable):**

```json
{
  "id": "pit-vine-002",
  "regionId": "region-pitfall-001",
  "name": "Vine",
  "position": [47, 0, 8],
  "visualRef": "blob-vine",
  "visualType": "image/png",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "objectType": "vine",
    "grabPoint": [47, 0, 7],
    "swingArc": 3.0
  }
}
```

**Collectible (treasure):**

```json
{
  "id": "pit-gold-012",
  "regionId": "region-pitfall-001",
  "name": "Gold Bar",
  "position": [52, 0, 3],
  "visualRef": "blob-gold-bar",
  "visualType": "image/png",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "objectType": "collectible",
    "points": 500,
    "collected": false
  }
}
```

**Hazard (moving enemy):**

```json
{
  "id": "pit-croc-005",
  "regionId": "region-pitfall-001",
  "name": "Crocodile",
  "position": [60, 0, 2],
  "visualRef": "blob-crocodile",
  "visualType": "image/png",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "objectType": "enemy",
    "patrolMin": 58,
    "patrolMax": 64,
    "speed": 2.0,
    "damage": "instant-death"
  }
}
```

### Interaction Model

The simulation layer runs a game loop (see the
[Simulation Layer Guide](jmap-scene-simulation-guide.md) for tick
rate selection and authority models):

1. **Input:** Client sends movement input via the simulation
   connection: left/right arrows for movement, up/space for jump.
   Input is streamed at the simulation layer's tick rate, not as
   JMAP method calls.
2. **Physics:** The simulation applies gravity, handles collision
   with terrain and platforms, detects contact with hazards and
   collectibles.
3. **Avatar position:** The simulation broadcasts the player's
   SceneAvatar position at 30-60 Hz. The client interpolates between
   updates for smooth rendering.
4. **Camera:** The client camera follows the SceneAvatar, scrolling
   the viewport horizontally. The viewport width is typically 16-20
   units.
5. **Vine swinging:** When the player's avatar contacts a vine
   (`"grab"` interaction), the simulation attaches the avatar to a
   pendulum arc. The player releases with jump to launch.
6. **Collectibles:** On avatar-collectible collision, the simulation
   sets `collected: true`, updates the score, and changes the
   visual to invisible (or a "collected" animation).
7. **Death/respawn:** On hazard contact, the simulation respawns the
   avatar at the last checkpoint. Lives are tracked in game state.

### Game State

```json
{
  "customProperties": {
    "phase": "playing",
    "score": 12500,
    "lives": 2,
    "timeRemaining": 1142,
    "checkpointPosition": [40, 0, 4],
    "treasuresCollected": 7,
    "treasuresTotal": 32
  }
}
```

### Multiplayer Variant

Multiple SceneAvatars in the same region. Each player sees all
avatars. The simulation handles independent collision per avatar.
Cooperative or competitive variants are application logic -- the
Scene spec is agnostic.

### Genre Pattern: Side-Scrollers

This pattern generalizes to all side-scrolling games:

| Subgenre | Examples | Key Differences |
|---|---|---|
| Platformer | Pitfall, Mario, Celeste | Jump physics, platforms |
| Run-and-gun | Contra, Metal Slug | Projectile objects, enemies |
| Metroidvania | Castlevania, Hollow Knight | Large interconnected map, abilities unlock areas |
| Endless runner | Temple Run (2D) | Procedural terrain generation, auto-scroll |
| Beat-em-up | Streets of Rage | Melee combat, enemy AI |

All use `viewHint: "2d-side"`, horizontal scrolling, gravity, and
platform collision. The differences are in the simulation layer's
game logic, not in the Scene data model.

### Hidden Information

None -- all SceneObject positions and states are public. In
multiplayer variants all players see the same world state.


---

## 15. Asteroids (Top-Down Arcade)

A top-down 2D arcade game with vector-style rendering. The player
controls a ship that rotates and thrusts, shooting asteroids that
split into smaller pieces.

### Region

A wrap-around arena. Objects exiting one edge reappear on the
opposite edge (toroidal topology).

```json
{
  "id": "region-asteroids-001",
  "name": "Asteroid Field",
  "bounds": { "min": [0, 0, 0], "max": [40, 0, 30] },
  "viewHint": "2d-topdown",
  "spawnPosition": [20, 0, 15],
  "simulationUri": "wss://sim.example.com/games/asteroids/001",
  "accessPolicy": "public",
  "environment": {
    "wrapEdges": true,
    "background": "starfield",
    "friction": 0,
    "renderStyle": "vector"
  }
}
```

The entire arena is visible at once (no scrolling). The camera is
fixed, showing the full bounds.

### Pieces

**Player ship (SceneAvatar, not SceneObject):**

The player's ship is their SceneAvatar. The avatar's `orientation`
quaternion encodes the ship's facing direction. Thrust applies
velocity in the facing direction; the simulation handles inertia
(no friction in space).

```json
{
  "id": "ast-avatar-alice",
  "regionId": "region-asteroids-001",
  "userId": "user:alice@example.com",
  "displayName": "Alice",
  "position": [20, 0, 15],
  "orientation": [0, 0.707, 0, 0.707],
  "visualRef": "blob-ship-vector",
  "visualType": "image/svg+xml",
  "customProperties": {
    "velocity": [2.5, 0, -1.3],
    "shielded": false,
    "weaponCooldown": 0
  }
}
```

**Asteroid (large, medium, small):**

```json
{
  "id": "ast-rock-007",
  "regionId": "region-asteroids-001",
  "name": "Asteroid",
  "position": [12, 0, 22],
  "orientation": [0, 0.3, 0, 0.95],
  "visualRef": "blob-asteroid-large",
  "visualType": "image/svg+xml",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "size": "large",
    "velocity": [-1.2, 0, 0.8],
    "rotationSpeed": 0.5,
    "hp": 1,
    "splitInto": 2,
    "splitSize": "medium",
    "points": 20
  }
}
```

When a large asteroid is destroyed, the simulation creates two medium
asteroids at the same position with randomized velocities. Medium
splits into two small. Small asteroids are destroyed outright.

**Bullet (short-lived projectile):**

```json
{
  "id": "ast-bullet-042",
  "regionId": "region-asteroids-001",
  "name": "Bullet",
  "position": [21, 0, 14],
  "visualRef": "blob-bullet-dot",
  "visualType": "image/svg+xml",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "velocity": [8, 0, -4],
    "firedBy": "user:alice@example.com",
    "ttl": 2.0
  }
}
```

Bullets are created by the simulation when the player fires, travel
in a straight line at high speed, and are destroyed after `ttl`
seconds or on collision with an asteroid. The simulation handles
collision detection.

**UFO (periodic enemy):**

```json
{
  "id": "ast-ufo-001",
  "regionId": "region-asteroids-001",
  "name": "UFO",
  "position": [0, 0, 10],
  "visualRef": "blob-ufo-vector",
  "visualType": "image/svg+xml",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "size": "large",
    "velocity": [3, 0, 0],
    "shootInterval": 1.5,
    "points": 200
  }
}
```

### Interaction Model

The simulation layer runs a continuous game loop at 60 Hz. For tick
rate selection, authority models, and state reconciliation for
top-down arcade games, see the [Simulation Layer Guide](jmap-scene-simulation-guide.md).

1. **Input:** Left/right arrow rotates the ship (updates
   `orientation`). Up arrow applies thrust in the facing direction
   (adds to `velocity`). Space fires a bullet. Input is streamed
   via the simulation connection.
2. **Physics:** All objects move by their velocity each tick. Objects
   wrap at region boundaries (toroidal). The simulation detects
   collisions between bullets and asteroids, asteroids and the ship,
   and UFO bullets and the ship.
3. **Collision:** Bullet hits asteroid: destroy bullet, damage
   asteroid (destroy or split). Asteroid hits ship: destroy ship,
   decrement lives, respawn after delay with brief invulnerability.
4. **Wave progression:** When all asteroids are destroyed, the
   simulation spawns a new wave with more and faster asteroids.
5. **Scoring:** Points for each asteroid and UFO destroyed.

### Game State

```json
{
  "customProperties": {
    "phase": "playing",
    "score": 4280,
    "lives": 3,
    "wave": 3,
    "asteroidsRemaining": 7,
    "highScore": 12450
  }
}
```

### Multiplayer

Multiple SceneAvatars (ships) in the same arena. Cooperative
(all players vs asteroids) or competitive (friendly fire enabled).
The simulation layer decides the mode.

### Hidden Information

None -- all object positions are visible to all players. The
wrap-around arena has no fog-of-war.


---

## 16. Battlezone (2.5D Vector Tank Game)

The classic 1980 Atari coin-op vector tank game. First-person view
from inside a tank, rendered with vector-style wireframe graphics.
Battlezone is 2.5D: the arena is a flat ground plane, the tank cannot
aim vertically, and all enemies are rendered as flat vector shapes.
Like Doom, it uses `viewHint: "3d"` for the first-person perspective
but the gameplay is constrained to 2D movement with height used only
for visual depth.

### Region

A flat arena with distant geometric mountains on the horizon.

```json
{
  "id": "region-battlezone-001",
  "name": "Battlezone Arena",
  "bounds": { "min": [-200, 0, -200], "max": [200, 0, 200] },
  "viewHint": "3d",
  "spawnPosition": [0, 0, 0],
  "simulationUri": "wss://sim.example.com/games/battlezone/001",
  "accessPolicy": "public",
  "environment": {
    "renderStyle": "vector-wireframe",
    "horizonColor": "#00ff00",
    "groundPlane": true,
    "skybox": null,
    "renderMode": "2.5d",
    "verticalAim": "none",
    "entityRendering": "wireframe",
    "radarEnabled": true,
    "radarRadius": 50
  }
}
```

`viewHint: "3d"` because the player sees a first-person perspective
view, even though movement is constrained to a 2D plane.
`renderMode: "2.5d"` signals the 2.5D constraints.

### Pieces

**Player tank (SceneAvatar):**

```json
{
  "id": "bz-avatar-alice",
  "regionId": "region-battlezone-001",
  "userId": "user:alice@example.com",
  "displayName": "Alice",
  "position": [0, 1, 0],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-tank-wireframe",
  "visualType": "model/gltf-binary",
  "customProperties": {
    "velocity": [0, 0, 0],
    "turretAngle": 0,
    "reloadTime": 0,
    "alive": true
  }
}
```

**Enemy tank:**

```json
{
  "id": "bz-enemy-004",
  "regionId": "region-battlezone-001",
  "name": "Enemy Tank",
  "position": [45, 1, -30],
  "orientation": [0, 0.5, 0, 0.866],
  "visualRef": "blob-enemy-tank-wireframe",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "aiState": "hunting",
    "speed": 3.0,
    "fireRate": 2.0,
    "points": 1000
  }
}
```

**Shell (projectile):**

```json
{
  "id": "bz-shell-015",
  "regionId": "region-battlezone-001",
  "name": "Shell",
  "position": [5, 1, -2],
  "visualRef": "blob-shell-vector",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "velocity": [0, 0, -20],
    "firedBy": "user:alice@example.com",
    "ttl": 3.0
  }
}
```

**Obstacle (geometric, indestructible):**

```json
{
  "id": "bz-obstacle-002",
  "regionId": "region-battlezone-001",
  "name": "Block",
  "position": [20, 1, -15],
  "visualRef": "blob-cube-wireframe",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "obstacleType": "cube",
    "provideCover": true
  }
}
```

### Interaction Model

For authority models, conflict resolution, and state reconciliation
for real-time vehicle games, see the [Simulation Layer
Guide](jmap-scene-simulation-guide.md).

1. **Movement:** Forward/back arrows move the tank. Left/right rotate.
   The simulation applies tank treads physics (tank turns in place,
   then moves forward/back in the facing direction).
2. **Firing:** Space bar fires a shell from the turret position. One
   shell at a time; must wait for the previous shell to expire or
   hit before firing again.
3. **Radar:** The client renders a minimap overlay showing positions
   of all objects within radar range. This is a client-side rendering
   of SceneObject/SceneAvatar positions, not a separate data source.
4. **Enemy AI:** The simulation runs enemy tank AI -- patrol, detect
   player, pursue, take cover behind obstacles, fire shells.
5. **Collision:** Shell hits enemy tank: destroy enemy, spawn
   explosion effect (temporary SceneObject), award points. Enemy
   shell hits player: player is destroyed, respawn after delay.

### Game State

```json
{
  "customProperties": {
    "phase": "playing",
    "score": 7000,
    "lives": 3,
    "enemiesDestroyed": 7,
    "wave": 2,
    "enemiesRemaining": 3
  }
}
```

### Multiplayer

Multiple player tanks (SceneAvatars) in the same arena, cooperative
vs AI enemies or PvP deathmatch. Spatial audio via VTC integration
(see [JMAP Scene VTC Integration Guide](jmap-scene-vtc-integration-guide.md))
adds immersion -- you hear enemy engines getting louder as they approach.

### Hidden Information

None -- all object positions are public. Enemies behind obstacles
are still present in the SceneObject set; line-of-sight culling is
a client rendering decision, not a server visibility filter. The
radar displays all objects within range regardless of occlusion.


---

## 17. Doom (2.5D First-Person Shooter)

The canonical 2.5D FPS. Doom presents a first-person perspective but
its world is fundamentally 2D: levels are defined as sectors on a 2D
floor plan with floor and ceiling heights. There is no room-over-room.
Vertical aim is automatic (the engine auto-targets enemies above or
below). Enemies and pickups are billboard sprites, not 3D models.

Despite the 2.5D rendering, the Scene region uses `viewHint: "3d"`
because the player experiences a first-person perspective view. The
2.5D constraints are expressed in the environment configuration and
enforced by the simulation layer, not by the viewHint.

**2.5D constraints vs true 3D:**

| Property | Doom (2.5D) | Quake (true 3D) |
|---|---|---|
| Level geometry | 2D sectors with heights | Full 3D mesh |
| Room-over-room | No | Yes |
| Vertical aim | Auto-aim | Free mouselook |
| Entities | Billboard sprites | 3D models |
| Physics | 2D collision + height check | Full 3D collision |
| Coordinate use | X/Z primary, Y = height | X/Y/Z all active |

### Region

A level defined as 2D sectors with floor/ceiling heights. The
`renderMode: "2.5d"` environment hint tells the client it can use
BSP raycasting or simplified rendering rather than full 3D mesh
collision.

```json
{
  "id": "region-doom-e1m1",
  "name": "Hangar (E1M1)",
  "bounds": { "min": [0, 0, 0], "max": [128, 32, 128] },
  "viewHint": "3d",
  "spawnPosition": [10, 1, 10],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/games/doom/e1m1",
  "chatId": "chat-doom-frag-log",
  "accessPolicy": "public",
  "activeCallId": "call-doom-001",
  "environment": {
    "renderMode": "2.5d",
    "lightingModel": "sector-based",
    "ambientLight": 0.3,
    "gravity": 15.0,
    "damageFlash": true,
    "automap": true,
    "verticalAim": "auto",
    "entityRendering": "billboard"
  }
}
```

The `activeCallId` binds to a VTC call for voice chat. The
`chatId` binds to a Chat for the frag log (kill feed) and text
chat.

### Pieces

**Level geometry (static architecture):**

Level geometry is loaded as a single large SceneObject (or a set of
sector objects) from a glTF model. Walls, floors, ceilings, and doors
are all part of the level mesh.

```json
{
  "id": "doom-level-geometry",
  "regionId": "region-doom-e1m1",
  "name": "E1M1 Level",
  "position": [0, 0, 0],
  "visualRef": "blob-e1m1-level",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "levelFormat": "doom-bsp",
    "sectorCount": 85
  }
}
```

**Door (interactive, animated):**

```json
{
  "id": "doom-door-007",
  "regionId": "region-doom-e1m1",
  "name": "Door",
  "position": [32, 0, 48],
  "visualRef": "blob-door-metal",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "doorState": "closed",
    "requiresKey": null,
    "openHeight": 4.0,
    "autoCloseDelay": 4.0
  }
}
```

**Weapon pickup:**

```json
{
  "id": "doom-pickup-shotgun",
  "regionId": "region-doom-e1m1",
  "name": "Shotgun",
  "position": [25, 0.5, 30],
  "visualRef": "blob-shotgun-pickup",
  "visualType": "model/gltf-binary",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "pickupType": "weapon",
    "weapon": "shotgun",
    "ammoIncluded": 8,
    "respawnDelay": 30
  }
}
```

**Health/armor/ammo pickups:**

```json
{
  "id": "doom-pickup-health-003",
  "regionId": "region-doom-e1m1",
  "name": "Medikit",
  "position": [40, 0.5, 55],
  "visualRef": "blob-medikit",
  "visualType": "model/gltf-binary",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "pickupType": "health",
    "healAmount": 25,
    "respawnDelay": 30
  }
}
```

**Enemy (AI-controlled):**

```json
{
  "id": "doom-imp-004",
  "regionId": "region-doom-e1m1",
  "name": "Imp",
  "position": [50, 0, 60],
  "orientation": [0, 0.707, 0, 0.707],
  "visualRef": "blob-imp-model",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "enemyType": "imp",
    "hp": 60,
    "state": "idle",
    "meleeRange": 2.0,
    "projectileType": "fireball",
    "alertSound": "blob-imp-alert",
    "deathSound": "blob-imp-death",
    "points": 100
  }
}
```

**Projectile (fireball, rocket, etc.):**

```json
{
  "id": "doom-fireball-019",
  "regionId": "region-doom-e1m1",
  "name": "Imp Fireball",
  "position": [48, 1.5, 58],
  "visualRef": "blob-fireball",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "velocity": [-8, 0, -6],
    "damage": 10,
    "firedBy": "doom-imp-004",
    "splashRadius": 0,
    "ttl": 5.0
  }
}
```

### Interaction Model

The simulation layer runs a full FPS game loop at 35+ Hz (classic
Doom's tick rate). See the [Simulation Layer Guide](jmap-scene-simulation-guide.md)
for hitscan authority, state reconciliation, and transport protocol
selection for FPS games.

1. **Movement:** WASD for movement, mouse for look (pitch and yaw
   update the SceneAvatar's `orientation`). The simulation applies
   collision with level geometry -- the player cannot walk through
   walls.
2. **Shooting:** Mouse click fires the current weapon. Vertical aim
   is automatic -- the simulation auto-targets enemies at different
   heights along the horizontal aim line (2.5D constraint). The
   simulation performs a hitscan (instant ray for bullet weapons) or
   creates a projectile SceneObject (for rockets, plasma). Hitscan
   results are resolved server-side: the simulation checks
   line-of-sight, applies damage to the target, and broadcasts the
   hit event.
3. **Weapon switching:** Number keys or scroll wheel. The simulation
   updates the player's `currentWeapon` in their SceneAvatar
   `customProperties`. The client renders the weapon model in first
   person from a local asset (not a SceneObject).
4. **Doors:** Player approaches and presses use key (E / `"activate"`
   interaction). Simulation checks key requirements, opens the door
   (animates position upward), and auto-closes after a delay.
5. **Pickups:** On avatar-pickup collision, simulation adds the item
   to the player's inventory, plays a pickup sound, and hides the
   object (or destroys and recreates it after `respawnDelay` in
   deathmatch mode).
6. **Enemy AI:** The simulation runs per-enemy state machines: idle,
   alert (heard gunfire or saw player), chase, attack (melee or
   ranged), pain (briefly stunned on taking damage), death.
7. **Damage:** Health tracked in SceneAvatar `customProperties`. On
   death, the avatar enters a death state (camera falls to floor),
   then respawns at a spawn point after a delay.
8. **Frag log:** Each kill posts a system message to the bound Chat:
   "Alice fragged Bob with the Rocket Launcher." The Chat serves as
   the kill feed.

### Player Avatar

```json
{
  "id": "doom-avatar-alice",
  "regionId": "region-doom-e1m1",
  "userId": "user:alice@example.com",
  "displayName": "Alice",
  "position": [10, 1, 10],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-doomguy-model",
  "visualType": "model/gltf-binary",
  "customProperties": {
    "hp": 85,
    "armor": 50,
    "currentWeapon": "shotgun",
    "weapons": ["fist", "pistol", "shotgun"],
    "ammo": {
      "bullets": 42,
      "shells": 16,
      "rockets": 0,
      "cells": 0
    },
    "kills": 12,
    "deaths": 3,
    "alive": true
  }
}
```

Other players see Alice's avatar model in third person. Alice sees
a first-person weapon view rendered client-side from the
`currentWeapon` asset.

### Game State

```json
{
  "customProperties": {
    "gameMode": "deathmatch",
    "phase": "playing",
    "fragLimit": 20,
    "timeLimit": 600,
    "elapsedSeconds": 187,
    "scores": {
      "user:alice@example.com": 12,
      "user:bob@example.com": 8,
      "user:carol@example.com": 15
    },
    "spawnPoints": [
      [10, 1, 10], [50, 1, 80], [100, 1, 40], [20, 1, 110]
    ],
    "itemsActive": 14,
    "monstersRemaining": 0
  }
}
```

**Game modes:**
- **Deathmatch:** Free-for-all, first to frag limit wins.
- **Cooperative:** Players vs AI enemies, clear the level together.
- **Team deathmatch:** Two teams, team frag limit.

### Spatial Audio

With `activeCallId` set, VTC audio is spatialized:
- Other players' voice chat is positioned at their SceneAvatar
  location (hear them louder when closer).
- Weapon sounds, door sounds, and enemy sounds are SceneObject audio
  effects positioned in 3D space.
- The Web Audio API PannerNode graph handles attenuation and HRTF.

For detailed spatial audio implementation (Web Audio PannerNode,
distance models, coordinate mapping), see the
[JMAP Scene VTC Integration Guide](jmap-scene-vtc-integration-guide.md).

### Hidden Information

All SceneObject positions are public (no fog-of-war in classic Doom).
Player inventory (`customProperties` on SceneAvatar) is public --
in classic Doom you can see other players' health on the HUD.

For a fog-of-war variant, the server would use `SceneObject/query`
spatial filtering to return only objects within the player's line of
sight, computed by the simulation layer.


---

## 18. Quake (True 3D First-Person Shooter)

The first true-3D FPS. Where Doom's world is a 2D floor plan with
height, Quake's world is full 3D polygon mesh: rooms over rooms,
sloped surfaces, true vertical aim with free mouselook, 3D polygon
models for all entities, and physics that work in all three axes
(rocket jumping is vertical).

### Region

A fully 3D level with true volumetric architecture.

```json
{
  "id": "region-quake-dm4",
  "name": "The Bad Place (DM4)",
  "bounds": { "min": [-64, -32, -64], "max": [64, 48, 64] },
  "viewHint": "3d",
  "spawnPosition": [10, 2, -5],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/games/quake/dm4",
  "chatId": "chat-quake-frag-log",
  "accessPolicy": "public",
  "activeCallId": "call-quake-001",
  "environment": {
    "renderMode": "3d",
    "lightingModel": "lightmapped",
    "ambientLight": 0.15,
    "gravity": 15.0,
    "airControl": 0.3,
    "damageFlash": true,
    "automap": false,
    "verticalAim": "free",
    "entityRendering": "model"
  }
}
```

Key differences from Doom's environment:
- `"renderMode": "3d"` -- full mesh rendering required, no BSP
  raycasting shortcut.
- `"verticalAim": "free"` -- the player aims freely in all axes with
  mouselook. No auto-aim.
- `"entityRendering": "model"` -- enemies and items are 3D models
  (glTF), not billboard sprites.
- `"airControl": 0.3` -- players can change direction mid-air,
  enabling strafejumping and rocket jumping.

### Pieces

**Level geometry:**

Full 3D mesh, including rooms stacked vertically, ramps, platforms
over voids, and underwater sections. Unlike Doom's sector-based
geometry, this is a true polygon soup with arbitrary geometry.

```json
{
  "id": "quake-level-geometry",
  "regionId": "region-quake-dm4",
  "name": "DM4 Level",
  "position": [0, 0, 0],
  "visualRef": "blob-dm4-level",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "levelFormat": "quake-bsp3",
    "brushCount": 2048,
    "hasWater": true,
    "hasLava": true
  }
}
```

**Enemy (3D model, not sprite):**

```json
{
  "id": "quake-ogre-003",
  "regionId": "region-quake-dm4",
  "name": "Ogre",
  "position": [20, 8, -15],
  "orientation": [0, 0.707, 0, 0.707],
  "visualRef": "blob-ogre-model",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "enemyType": "ogre",
    "hp": 200,
    "state": "patrol",
    "meleeWeapon": "chainsaw",
    "rangedWeapon": "grenade",
    "model": "blob-ogre-model",
    "animationState": "walk"
  }
}
```

Unlike Doom's billboard sprites, the Ogre is a 3D model with skeletal
animation. It has the same appearance from all viewing angles.

**Weapon pickup (3D model, bobbing):**

```json
{
  "id": "quake-pickup-rl",
  "regionId": "region-quake-dm4",
  "name": "Rocket Launcher",
  "position": [5, 4.5, 10],
  "visualRef": "blob-rocketlauncher-pickup",
  "visualType": "model/gltf-binary",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "pickupType": "weapon",
    "weapon": "rocket_launcher",
    "ammoIncluded": 5,
    "respawnDelay": 30,
    "bobAnimation": true
  }
}
```

**Rocket (3D projectile with splash damage):**

```json
{
  "id": "quake-rocket-027",
  "regionId": "region-quake-dm4",
  "name": "Rocket",
  "position": [12, 6, -3],
  "orientation": [0, 0.3, 0, 0.95],
  "visualRef": "blob-rocket-model",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "velocity": [0, 2, -20],
    "damage": 100,
    "splashRadius": 5.0,
    "splashDamage": 80,
    "firedBy": "user:alice@example.com",
    "ttl": 5.0
  }
}
```

Note the Y component in velocity -- rockets travel in full 3D,
including upward and downward. Splash damage affects everything in
`splashRadius`, including the shooter (self-damage enables rocket
jumping).

### Interaction Model

The simulation layer runs a game loop at 77 Hz (Quake's server tick
rate, equivalent to `sv_fps 77`). For client-side prediction,
authority models, and state reconciliation at high tick rates, see
the [Simulation Layer Guide](jmap-scene-simulation-guide.md).

1. **Movement:** WASD + mouselook. The critical difference from Doom:
   **free vertical look** (pitch + yaw). The player aims with the
   mouse in all axes. The simulation tracks the full 3D orientation
   quaternion.
2. **Strafejumping:** By combining forward movement with strafe keys
   and smooth mouse turning, the player accelerates beyond normal
   speed. The simulation's air-control physics make this emergent --
   no special code required, just correct physics.
3. **Rocket jumping:** The player aims at the ground, jumps, and
   fires a rocket. The splash damage pushes the player upward (and
   deals self-damage). The simulation applies the impulse from
   `splashRadius` to the shooter's SceneAvatar, launching them to
   higher platforms.
4. **Vertical combat:** Players fight across multiple height levels.
   Aim requires tracking targets above and below, not just on the
   horizon. Projectiles arc in 3D space (grenades are affected by
   gravity; rockets fly straight).
5. **Water/lava:** Entering water sectors changes movement physics
   (reduced gravity, swimming). Lava deals continuous damage. The
   simulation detects avatar-volume intersection with liquid sectors.
6. **Item respawn:** In deathmatch, destroyed items respawn after
   `respawnDelay`. Mega-health and quad damage have longer respawn
   timers, creating map control dynamics.

### Player Avatar

```json
{
  "id": "quake-avatar-alice",
  "regionId": "region-quake-dm4",
  "userId": "user:alice@example.com",
  "displayName": "Alice",
  "position": [10, 2, -5],
  "orientation": [0.1, 0.3, 0, 0.95],
  "visualRef": "blob-quake-player-model",
  "visualType": "model/gltf-binary",
  "customProperties": {
    "hp": 75,
    "armor": 100,
    "armorType": "yellow",
    "currentWeapon": "rocket_launcher",
    "weapons": ["axe", "shotgun", "super_shotgun", "rocket_launcher"],
    "ammo": {
      "shells": 20,
      "nails": 0,
      "rockets": 12,
      "cells": 0
    },
    "powerups": {
      "quad": 0,
      "pent": 0,
      "ring": 0
    },
    "kills": 8,
    "deaths": 5,
    "alive": true,
    "velocity": [3.2, 0, -1.5]
  }
}
```

The `orientation` quaternion encodes both yaw and pitch -- the
player looks up and down freely, unlike Doom's horizontal-only view.

### Game State

```json
{
  "customProperties": {
    "gameMode": "deathmatch",
    "phase": "playing",
    "fragLimit": 25,
    "timeLimit": 600,
    "elapsedSeconds": 312,
    "scores": {
      "user:alice@example.com": 8,
      "user:bob@example.com": 12,
      "user:carol@example.com": 6
    },
    "quadDamageRespawnAt": 1720000000,
    "megaHealthRespawnAt": null
  }
}
```

### What Makes This True 3D (vs Doom's 2.5D)

Everything that Doom could not do, Quake can:

- **Room-over-room:** A bridge over a hallway. Players on the bridge
  and players in the hallway below occupy different Y positions in
  the same X/Z coordinates.
- **True vertical aiming:** Aim up at a player on a ledge above you,
  aim down at a player in a pit below you. No auto-aim.
- **3D projectile physics:** Grenades bounce in 3D space with
  gravity. Rockets travel along the player's 3D aim vector.
- **3D collision:** Walk up ramps, fall off ledges, swim in 3D.
  Collision is against arbitrary mesh, not sector heights.
- **Skeletal animation:** Player and enemy models use skeletal
  animation (run, attack, pain, death), not sprite rotations.
- **Dynamic lighting:** Rocket explosions cast dynamic light on
  surrounding geometry (not possible with Doom's sector-based
  lighting).

### Hidden Information

Same as Doom: all SceneObject positions are public (no fog-of-war
in standard Quake deathmatch). Player inventory and health are
public via SceneAvatar `customProperties`. For competitive modes
that require fog-of-war, the server would use spatial filtering on
`SceneObject/query` to return only objects within the player's
line of sight.


---

## 19. Descent (6DOF True 3D)

Descent is a six-degrees-of-freedom (6DOF) first-person shooter in
zero gravity. The player pilots a ship through mine tunnels and can
move and rotate freely in all three axes -- there is no "up." This
represents the extreme end of `viewHint: "3d"`, using the full
generality of the Scene coordinate system.

### Region

A 3D mine network with tunnels in all orientations, including
vertical shafts and inverted rooms.

```json
{
  "id": "region-descent-mine1",
  "name": "Lunar Outpost Mine 1",
  "bounds": { "min": [-100, -100, -100], "max": [100, 100, 100] },
  "viewHint": "3d",
  "spawnPosition": [0, 0, 0],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/games/descent/mine1",
  "chatId": "chat-descent-001",
  "accessPolicy": "public",
  "environment": {
    "renderMode": "3d",
    "lightingModel": "vertex-lit",
    "gravity": 0,
    "sixDOF": true,
    "tunnelGeometry": true,
    "reactorPresent": true
  }
}
```

`"gravity": 0` and `"sixDOF": true` -- the player can pitch, yaw,
and roll freely. There is no ground plane. The scene coordinate
system's Y-up convention is used for the glTF models but has no
gameplay significance.

### Pieces

**Mine tunnel geometry:**

```json
{
  "id": "descent-level-geometry",
  "regionId": "region-descent-mine1",
  "name": "Mine 1 Tunnels",
  "position": [0, 0, 0],
  "visualRef": "blob-mine1-level",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "levelFormat": "descent-pof",
    "segmentCount": 320
  }
}
```

**Robot enemy (full 3D, moves in all axes):**

```json
{
  "id": "descent-robot-012",
  "regionId": "region-descent-mine1",
  "name": "Medium Hulk",
  "position": [30, -15, 42],
  "orientation": [0.2, 0.5, -0.1, 0.83],
  "visualRef": "blob-medium-hulk",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "robotType": "medium_hulk",
    "hp": 150,
    "state": "patrol",
    "weapon": "vulcan",
    "fireRate": 3.0,
    "speed": 5.0,
    "cloaked": false,
    "dropItem": "energy_pack"
  }
}
```

The orientation quaternion encodes an arbitrary rotation -- this robot
may be "upside down" relative to the spawn orientation, which is
normal in zero-G mine tunnels.

**Player ship (SceneAvatar with full 6DOF state):**

```json
{
  "id": "descent-avatar-alice",
  "regionId": "region-descent-mine1",
  "userId": "user:alice@example.com",
  "displayName": "Alice",
  "position": [0, 0, 0],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-pyro-gx",
  "visualType": "model/gltf-binary",
  "customProperties": {
    "velocity": [2.0, -1.5, 3.0],
    "angularVelocity": [0, 0.1, 0],
    "shields": 100,
    "energy": 100,
    "primaryWeapon": "laser_level3",
    "secondaryWeapon": "concussion_missile",
    "ammo": {
      "concussion": 8,
      "homing": 4,
      "smart": 0,
      "mega": 1
    },
    "keys": {
      "blue": false,
      "gold": true,
      "red": false
    },
    "alive": true
  }
}
```

**Reactor (level objective):**

```json
{
  "id": "descent-reactor",
  "regionId": "region-descent-mine1",
  "name": "Reactor Core",
  "position": [-60, 10, -80],
  "visualRef": "blob-reactor-core",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "hp": 500,
    "state": "active",
    "selfDestructCountdown": null
  }
}
```

### Interaction Model

The simulation layer runs a game loop at 30+ Hz. 6DOF physics
introduces full quaternion angular velocity and orientation-relative
movement; for authority models and conflict resolution in 6DOF
environments, see the [Simulation Layer
Guide](jmap-scene-simulation-guide.md).

1. **6DOF movement:** The player has six independent controls:
   forward/back, strafe left/right, rise/drop (vertical strafe),
   pitch (look up/down), yaw (turn left/right), and roll (tilt
   clockwise/counterclockwise). All movement is relative to the
   ship's current orientation -- there is no absolute "up."
2. **Weapons:** Primary weapons (lasers, vulcan, plasma, fusion)
   fire continuously and consume energy. Secondary weapons (missiles)
   are finite ammo. Homing missiles track targets in 3D space.
3. **Navigation:** The player navigates labyrinthine mine tunnels
   that twist in all three axes. Orientation can become disorienting
   -- the automap (a wireframe 3D map overlay) helps players
   navigate.
4. **Key doors:** Colored key gates block access to deeper mine
   sections. Keys are dropped by specific robots or found as pickups.
5. **Reactor destruction:** The level objective is to destroy the
   reactor core, then escape through the exit before the
   self-destruct countdown expires. On reactor destruction, the
   simulation sets `selfDestructCountdown` and the level begins
   collapsing (dynamic SceneObjects simulate cave-ins).
6. **Hostage rescue:** Hostages (non-interactable SceneObjects) are
   scattered through the mine. Flying near them triggers automatic
   rescue (proximity collision).

### Game State

```json
{
  "customProperties": {
    "gameMode": "single-player",
    "phase": "playing",
    "level": 1,
    "score": 28500,
    "hostagesRescued": 4,
    "hostagesTotal": 7,
    "reactorDestroyed": false,
    "selfDestructCountdown": null,
    "secretsFound": 1,
    "secretsTotal": 3
  }
}
```

**Multiplayer deathmatch:** 6DOF combat is uniquely disorienting --
enemies can attack from literally any direction, including directly
above, below, or behind while inverted. Spatial audio (via VTC
integration) is critical for situational awareness. For detailed
spatial audio implementation (Web Audio PannerNode, distance models,
coordinate mapping), see the
[JMAP Scene VTC Integration Guide](jmap-scene-vtc-integration-guide.md).

### What Makes This Full 6DOF (vs Standard 3D)

| Property | Quake (3D) | Descent (6DOF) |
|---|---|---|
| Gravity | Yes (fall down) | No (zero-G) |
| Ground plane | Yes (walk on floors) | No (no "floor") |
| Roll | No (camera stays level) | Yes (tilt freely) |
| Vertical strafe | No (jump only) | Yes (rise/drop) |
| Orientation | Yaw + pitch | Yaw + pitch + roll |
| Navigation | Walk/jump/swim | Fly freely |

Descent uses the SceneAvatar's full quaternion orientation --
including roll -- which most other `viewHint: "3d"` games constrain
to yaw and pitch only.

### Hidden Information

None in single-player -- all robot and item positions are known to
the server and visible to the player. In multiplayer deathmatch,
all positions are public. Cloaked robots (`cloaked: true`) are
visually hidden by the client but remain in the SceneObject set;
the simulation layer may optionally withhold cloaked enemy positions
from `SceneObject/get` responses for competitive fairness.


---

## 20. Flight Combat (Wing Commander / Star Fox)

A flight combat game representing the third-person variant of 3D
gameplay. The player controls a vehicle (spacecraft, aircraft) from a
behind-the-ship camera, engaging enemies in 3D space.

### Region

Open space or atmospheric environment. Large bounds relative to
entity size.

```json
{
  "id": "region-flight-001",
  "name": "Nav Point Alpha",
  "bounds": { "min": [-2000, -500, -2000], "max": [2000, 500, 2000] },
  "viewHint": "3d",
  "spawnPosition": [0, 0, 0],
  "spawnOrientation": [0, 0, 0, 1],
  "simulationUri": "wss://sim.example.com/games/flight/alpha",
  "chatId": "chat-flight-comms",
  "accessPolicy": "public",
  "activeCallId": "call-flight-001",
  "environment": {
    "renderMode": "3d",
    "lightingModel": "pbr",
    "gravity": 0,
    "skybox": "blob-skybox-starfield",
    "cameraMode": "third-person-chase",
    "cameraDistance": 15,
    "cameraOffset": [0, 3, 0],
    "atmosphericFog": false,
    "radarMode": "sphere"
  }
}
```

`"cameraMode": "third-person-chase"` -- the client positions the
camera behind and above the player's SceneAvatar, looking forward.
This is a client-side rendering decision driven by the environment
hint.

### Pieces

**Player ship (SceneAvatar):**

```json
{
  "id": "flight-avatar-alice",
  "regionId": "region-flight-001",
  "userId": "user:alice@example.com",
  "displayName": "Maverick",
  "position": [0, 0, 0],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-fighter-model",
  "visualType": "model/gltf-binary",
  "customProperties": {
    "velocity": [0, 0, -50],
    "throttle": 0.6,
    "maxSpeed": 300,
    "afterburnerFuel": 100,
    "shields": { "front": 80, "rear": 100 },
    "armor": 50,
    "weapons": {
      "guns": { "type": "laser", "energy": 100, "fireRate": 8 },
      "missiles": { "type": "heatseeking", "count": 6 },
      "torpedoes": { "type": "heavy", "count": 2 }
    },
    "kills": 3,
    "alive": true
  }
}
```

**Enemy fighter (AI-controlled):**

```json
{
  "id": "flight-enemy-dralthi-02",
  "regionId": "region-flight-001",
  "name": "Dralthi Fighter",
  "position": [500, 50, -800],
  "orientation": [0.1, 0.8, 0.1, 0.58],
  "visualRef": "blob-dralthi-model",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "enemyType": "dralthi",
    "velocity": [30, -10, 40],
    "hp": 120,
    "state": "engaging",
    "target": "flight-avatar-alice",
    "weapon": "mass_driver",
    "points": 500
  }
}
```

**Capital ship (large static or slow-moving):**

```json
{
  "id": "flight-capship-001",
  "regionId": "region-flight-001",
  "name": "TCS Tiger's Claw",
  "position": [-500, 0, 200],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-bengal-carrier",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": true,
  "visible": true,
  "scale": [1, 1, 1],
  "customProperties": {
    "shipType": "carrier",
    "hp": 5000,
    "dockable": true,
    "landingBay": [0, -20, 50]
  }
}
```

**Missile (tracking projectile):**

```json
{
  "id": "flight-missile-015",
  "regionId": "region-flight-001",
  "name": "Heat Seeker",
  "position": [5, 0, -10],
  "orientation": [0, 0, 0, 1],
  "visualRef": "blob-missile-model",
  "visualType": "model/gltf-binary",
  "physicsMode": "kinematic",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "velocity": [0, 0, -120],
    "trackingTarget": "flight-enemy-dralthi-02",
    "turnRate": 2.0,
    "damage": 200,
    "firedBy": "user:alice@example.com",
    "ttl": 10.0
  }
}
```

Homing missiles use `trackingTarget` -- the simulation adjusts the
missile's velocity vector each tick to track toward the target's
current position, limited by `turnRate`. The target can evade by
turning sharply or using countermeasures.

### Interaction Model

1. **Flight controls:** Pitch/yaw/roll via mouse or joystick.
   Throttle via keyboard (+/-). Afterburner for burst speed. The
   simulation applies simplified flight physics (no realistic
   aerodynamics -- this is arcade-style space combat).
2. **Guns:** Hold fire button for continuous laser/mass driver fire.
   Energy regenerates over time. Guns are hitscan at close range,
   projectile at long range.
3. **Missiles:** Lock-on targeting (hold reticle over enemy until
   lock tone), then fire. The simulation creates a tracking missile
   SceneObject.
4. **Landing/docking:** Approach the carrier's landing bay slowly,
   align with the bay, and the simulation auto-docks (like auto-land).
5. **Wingman commands:** Send orders to AI wingmates via interaction
   events -- "Attack my target", "Break and attack", "Return to base."
   Orders are posted to the bound Chat channel.

### Game State

```json
{
  "customProperties": {
    "gameMode": "patrol",
    "phase": "combat",
    "objectivesComplete": 1,
    "objectivesTotal": 3,
    "navPoints": [
      { "name": "Alpha", "position": [0, 0, 0], "status": "cleared" },
      { "name": "Beta", "position": [1000, 0, -1500], "status": "active" },
      { "name": "Home", "position": [-500, 0, 200], "status": "pending" }
    ],
    "enemiesRemaining": 4,
    "wingmanAlive": true,
    "ejected": false
  }
}
```

### Hidden Information

None in single-player -- all enemy positions are known. In
multiplayer cooperative missions, all wingman and enemy positions
are shared among players. For PvP modes, radar range limits
could serve as a fog-of-war mechanism: the server withholds
`SceneObject/get` results for enemies beyond radar range, returning
them only when they enter detection distance.


---

## Part IV: Genre Patterns

The preceding games illustrate specific titles. This section
abstracts the patterns for broad game genres, showing which Scene
primitives and `viewHint` each genre uses.


---

## 21. Genre: 2D Top-Down Games

`viewHint: "2d-topdown"`. Orthographic camera looking down Y axis.

### Subgenres

| Subgenre | Examples | Key Mechanics |
|---|---|---|
| Board/card games | Chess, Go, Catan, Poker | Turn-based, game state object, hidden info |
| Top-down shooter | Asteroids, Robotron | Continuous sim, projectiles, wave spawning |
| Twin-stick shooter | Geometry Wars, Smash TV | Dual input (move + aim), bullet-hell density |
| Top-down RPG | Early Zelda, Pokémon | Tile map, NPCs, inventory, dialog |
| Real-time strategy | StarCraft (2D era), C&C | Unit SceneObjects, fog-of-war, resource tracking |
| Tower defense | Bloons, Kingdom Rush | Path-following enemies, placed tower objects |
| Puzzle | Tetris, Bejeweled | Grid state, gravity/match mechanics |

### Common Pattern

```json
{
  "viewHint": "2d-topdown",
  "environment": {
    "tileSize": 1.0,
    "gridVisible": false,
    "cameraMode": "fixed|follow-player|scrolling"
  }
}
```

**Camera modes:**
- **Fixed:** Full board visible (board games, Asteroids).
- **Follow-player:** Camera centers on player avatar (RPG, RTS).
- **Scrolling:** Camera pans across a large map (RTS, tower defense).

**Object density:** Board games have tens of objects. Arcade games
have hundreds (asteroids, bullets, particles). The simulation layer
must manage object creation/destruction efficiently -- short-lived
objects like bullets should use lightweight SceneObject lifecycle
(create, TTL, auto-destroy).


---

## 22. Genre: 2D Side-View Games

`viewHint: "2d-side"`. Orthographic camera looking along Z axis
from the side. X is horizontal, Y is vertical.

### Subgenres

| Subgenre | Examples | Key Mechanics |
|---|---|---|
| Platformer | Mario, Celeste, Pitfall | Jump physics, platforms, gravity |
| Run-and-gun | Contra, Metal Slug | Horizontal projectiles, enemy waves |
| Metroidvania | Castlevania, Hollow Knight | Interconnected map, ability gates |
| Fighting game | Street Fighter, Mortal Kombat | Two avatars, hitboxes, combos, rounds |
| Endless runner | Canabalt, Jetpack Joyride | Auto-scroll, procedural generation |
| Puzzle platformer | Limbo, Braid | Physics puzzles, time mechanics |

### Common Pattern

```json
{
  "viewHint": "2d-side",
  "environment": {
    "gravity": 9.81,
    "scrollDirection": "horizontal",
    "cameraTracking": "follow-player",
    "parallaxLayers": 3
  }
}
```

**Gravity:** All side-view games apply gravity (negative Y
acceleration). The simulation handles collision with ground, platform
edges, and ceilings. Jump mechanics define the subgenre feel -- jump
height, air control, coyote time, wall-jumping.

**Parallax:** The client can render background layers at different
scroll rates for depth. Background SceneObjects at large Y values
scroll slower, creating a parallax effect. This is a client-side
rendering optimization.

**Fighting games** are a special case: two SceneAvatars face each
other in a bounded arena. The simulation layer handles hitbox/hurtbox
collision, combo state machines, and round management. The camera
zooms to keep both fighters in frame.

```json
{
  "id": "region-fighter-001",
  "name": "Arena: Alice vs Bob",
  "bounds": { "min": [0, 0, 0], "max": [16, 10, 0] },
  "viewHint": "2d-side",
  "simulationUri": "wss://sim.example.com/games/fighter/001",
  "environment": {
    "gravity": 20.0,
    "roundsToWin": 2,
    "roundTime": 99,
    "cameraTracking": "frame-both-players"
  }
}
```


---

## 23. Genre: 2.5D First-Person Games

`viewHint: "3d"` with `environment.renderMode: "2.5d"`. These games
present a first-person perspective but their world is built on a 2D
floor plan extruded into heights. The client uses `viewHint: "3d"`
because the player experiences a perspective view, but the simulation
layer enforces 2D constraints on movement and collision.

### What Makes a Game 2.5D

The defining constraints:

1. **Sector-based geometry:** Levels are 2D polygonal sectors with
   floor and ceiling heights, not arbitrary 3D meshes. Walls are
   vertical planes between sectors.
2. **No room-over-room:** Two sectors cannot occupy the same X/Z
   coordinates at different heights. No bridges, no multi-story
   spaces sharing a footprint.
3. **2D movement:** The player moves on a 2D plane. Height changes
   happen automatically (stairs, elevators) rather than through
   player-controlled vertical movement.
4. **Auto-vertical aim:** The player aims horizontally only; the
   simulation auto-targets enemies above or below the aim line.
5. **Billboard entities:** Enemies and pickups are 2D sprites that
   always face the camera, not 3D models.

### Subgenres

| Subgenre | Examples | Key Differences |
|---|---|---|
| Raycaster | Wolfenstein 3D | Uniform ceiling height, grid-aligned walls |
| BSP FPS | Doom, Heretic, Hexen | Variable heights, stairs, lighting |
| BUILD engine | Duke Nukem 3D, Blood, Shadow Warrior | Fake room-over-room, destructible environments, more interactivity |
| RPG dungeon | Ultima Underworld, Elder Scrolls Arena | RPG mechanics in 2.5D dungeon crawl |

### Common Pattern

```json
{
  "viewHint": "3d",
  "environment": {
    "renderMode": "2.5d",
    "verticalAim": "auto",
    "entityRendering": "billboard",
    "lightingModel": "sector-based",
    "gravity": 15.0,
    "automap": true
  }
}
```

**Why `viewHint: "3d"` and not a dedicated `"2.5d"` hint:** The
viewHint describes what the player sees (a perspective first-person
view), not how the world is internally structured. A `"3d"` viewHint
tells the client to use a perspective camera and WASD+mouse controls.
The `renderMode: "2.5d"` environment hint tells the client it may
optimize rendering (sector-based raycasting instead of full mesh
rasterization) and that vertical aim is automatic.

A client that ignores `renderMode` and renders the level as full 3D
mesh (converting sectors to triangles) will produce correct results
-- just less efficiently than a sector-aware renderer.

### BUILD Engine Variant (Duke Nukem 3D)

The BUILD engine extended the 2.5D model with:
- **Fake room-over-room:** Teleporters at sector boundaries create
  the illusion of overlapping spaces.
- **Destructible environment:** Walls and objects can be destroyed
  via `SceneObject/set destroy`, revealing hidden areas.
- **Interactive environment:** Light switches, toilets, pool tables,
  vending machines -- many SceneObjects with `interactable: true` and
  deployment-specific `"activate"` behaviors.
- **Shrink/grow mechanics:** Player size changes affect collision
  bounds and visibility.

```json
{
  "environment": {
    "renderMode": "2.5d",
    "verticalAim": "auto",
    "entityRendering": "billboard",
    "destructibleGeometry": true,
    "interactionDensity": "high"
  }
}
```


---

## 24. Genre: True 3D Games

`viewHint: "3d"` with `environment.renderMode: "3d"` (or omitted,
since full 3D is the default). These games use the complete 3D
capability of the Scene spec: arbitrary mesh geometry, full 3D
physics, free camera orientation, and 3D model entities.

### Subgenres

| Subgenre | Examples | Camera | Key Mechanics |
|---|---|---|---|
| Arena FPS | Quake, Unreal Tournament | First-person | Weapons, movement physics, deathmatch |
| Tactical FPS | Counter-Strike, Valorant | First-person | Round-based, economy, bomb/defuse |
| Battle royale | Fortnite, PUBG | First/third | Shrinking zone, loot, last-standing |
| 6DOF shooter | Descent, Overload | First-person | Zero-G, full roll, mine tunnels |
| Third-person action | Tomb Raider, Dark Souls | Third-person | Camera follows avatar, melee/ranged |
| Flight/space combat | Wing Commander, Star Fox | Third-person chase | Vehicle physics, missiles, dogfighting |
| Racing | Need for Speed, TrackMania | Chase/cockpit | Track geometry, vehicle physics |
| Exploration | Myst, Gone Home | First-person | Puzzle objects, narrative triggers |
| Survival/crafting | Minecraft, Rust | First-person | Block creation/destruction, day/night |
| Social/virtual world | VRChat, Hubs, Second Life | First/third | Avatars, voice chat, user content |
| Horror | Amnesia, Outlast | First-person | Limited visibility, audio cues |

### Common Pattern

```json
{
  "viewHint": "3d",
  "environment": {
    "renderMode": "3d",
    "gravity": 9.81,
    "lightingModel": "pbr",
    "skybox": "blob-skybox-day",
    "fogDistance": 100,
    "collisionModel": "mesh",
    "verticalAim": "free",
    "entityRendering": "model",
    "cameraMode": "first-person|third-person-chase|third-person-orbit"
  }
}
```

### Rendering

The client uses Three.js, Babylon.js, or a native wgpu renderer.
Level geometry is loaded as glTF models. PBR materials provide
realistic lighting. All entities are 3D polygon models with skeletal
animation. The simulation layer handles physics and collision; the
client handles rendering, input, and audio.

### First-Person Weapon Rendering

The player's current weapon is rendered as a client-side overlay (not
a SceneObject). The weapon model is loaded from a blob referenced in
the SceneAvatar's `customProperties.currentWeapon`. This avoids the
latency of round-tripping weapon position through the server.

### Third-Person Camera

For third-person games, the client positions the camera behind the
player's SceneAvatar. Camera collision prevents clipping through
walls (the client casts a ray from the avatar to the desired camera
position and moves the camera forward on collision). The environment
configuration specifies camera distance and offset.

### Network Model

First-person and third-person 3D games are the most latency-sensitive
category. The simulation layer must run at 20+ Hz tick rate.
Client-side prediction (the client simulates movement locally and
reconciles with server corrections) is essential for responsive feel.
The Scene spec's snapshot model (JMAP `SceneAvatar/get` at 1-5 Hz)
is too slow for gameplay; the simulation connection at
`simulationUri` carries the real-time stream. For detailed coverage of
transport options (WebSocket, WebRTC, QUIC, WebTransport), authority
models, interpolation, and state reconciliation, see the [JMAP Scene
Simulation Layer Guide](jmap-scene-simulation-guide.md).

### Spatial Audio

```
Player A SceneAvatar.position --> PannerNode(posA)  \
Player B SceneAvatar.position --> PannerNode(posB)  --> AudioContext --> Speakers
Gunshot SceneObject.position --> PannerNode(posG)   /
```

All audio sources (voice, weapons, ambient) are positioned in 3D
space using the Web Audio API, creating directional sound that matches
the visual scene.

For detailed spatial audio implementation (Web Audio PannerNode,
distance models, coordinate mapping), see the
[JMAP Scene VTC Integration Guide](jmap-scene-vtc-integration-guide.md).

### Tactical FPS Pattern (Counter-Strike)

Round-based games add a match-state layer on top of the standard FPS
pattern:

```json
{
  "customProperties": {
    "gameMode": "bomb-defusal",
    "phase": "buy-phase",
    "round": 5,
    "roundTime": 115,
    "score": { "team-a": 3, "team-b": 1 },
    "economy": {
      "user:alice@example.com": 4750,
      "user:bob@example.com": 3200
    },
    "bombPlanted": false,
    "bombSite": null
  }
}
```

The simulation manages round transitions (freeze time, buy phase,
live round, post-round), economy (money for kills, round wins,
equipment purchases), and win conditions (bomb plant/defuse,
elimination, timeout).

### Survival/Crafting Pattern (Minecraft)

The key difference is that **players create and destroy SceneObjects**
as core gameplay. Block placement is `SceneObject/set create` with a
grid-aligned position. Block destruction is `SceneObject/set destroy`.
The simulation validates placement rules and handles resource
deduction from inventory.

```json
{
  "id": "mc-block-new",
  "regionId": "region-survival-001",
  "name": "Stone Block",
  "position": [42, 5, 18],
  "visualRef": "blob-block-stone",
  "visualType": "model/gltf-binary",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "blockType": "stone",
    "hardness": 1.5,
    "drops": ["cobblestone"],
    "placedBy": "user:alice@example.com"
  }
}
```

This makes `SceneObject/set` the primary game action rather than just
a state-management tool. Object creation rate can be high (several
per second during building), so the simulation layer should batch
state updates efficiently.

### 2.5D vs True 3D Decision Guide

| Question | If Yes → | If No → |
|---|---|---|
| Can rooms stack vertically? | True 3D | 2.5D sufficient |
| Does the player aim vertically? | True 3D | 2.5D auto-aim |
| Are entities 3D models? | True 3D | 2.5D billboards |
| Does physics operate in 3D? | True 3D | 2.5D |
| Is development budget limited? | 2.5D (simpler) | True 3D |

2.5D is a legitimate design choice, not a limitation. Many successful
games choose 2.5D constraints for faster development, retro aesthetic,
or gameplay reasons (Doom's auto-aim makes the game more accessible).
The Scene spec supports both through environment configuration.


---

## Part V: Cooperative and Non-Game Activities

Competitive games with turn alternation and winners are the most
common use case, but Scene's primitives also model cooperative
activities where all participants work toward a shared goal. The
distinguishing features of this category:

- **No opposing teams** -- `accessPolicy: "invite"` with all
  participants sharing a single goal state.
- **Object interaction chains** -- interacting with one object
  changes the state of another (solving a puzzle unlocks a door).
- **State machines on objects** -- each puzzle or door transitions
  through a finite set of named states tracked in `customProperties`.
- **Shared inventory** -- picked-up items belong to the group, not
  an individual, stored on the game state object.
- **No real-time physics simulation** -- `simulationUri: null` for
  turn-based / event-driven cooperative games. State transitions
  happen entirely through JMAP method calls.


---

## 25. Escape Room

A real-world activity recreated digitally. 2-8 players are locked in
a themed virtual room and must find hidden items, solve puzzles, and
unlock doors to escape before a countdown timer expires. All players
cooperate -- there is no opposing team and no individual winner.

What makes it a useful spec exercise:

- **Object interaction chains:** solving puzzle A changes the state
  of door B with no player touching the door directly.
- **State machines on objects:** puzzles transition
  `locked → solved`; doors transition `locked → unlocked → open`.
- **Shared inventory:** items are not owned by an individual player;
  they live in `customProperties.inventory` on the game state object.
- **Countdown timer:** a simple integer in game state, decremented
  server-side, that all clients observe.
- **`simulationUri: null`:** the room runs entirely on JMAP method
  calls. There is no real-time physics engine; the server validates
  interaction events and issues `SceneObject/set` patches directly.

### SceneRegion

A single room (or a short sequence of rooms). The example uses a
single-room escape with a 2D top-down view. A multi-room variant
can use `"3d"` with first-person navigation.

```json
{
  "id": "region-escape-001",
  "name": "Escape Room: The Laboratory",
  "bounds": { "min": [0, 0, 0], "max": [10, 0, 10] },
  "viewHint": "2d-topdown",
  "spawnPosition": [5, 0, 1],
  "simulationUri": null,
  "accessPolicy": "invite",
  "environment": {
    "theme": "laboratory",
    "gridVisible": false,
    "timerSeconds": 3600,
    "maxPlayers": 8
  }
}
```

> **`simulationUri: null`:** Because the room has no physics and all
> state transitions are discrete, a separate simulation service is
> unnecessary. The JMAP server handles all game logic directly through
> `SceneObject/set` and `SceneInteractionEvent` processing.

### Key Objects

The room contains three categories of objects: **puzzles** (interactable,
stateful), **doors** (interactable, state-gated), and **items**
(interactable, grabbable, placed in shared inventory).

**Puzzle: Combination lock (locked state)**

```json
{
  "id": "escape-puzzle-lock",
  "regionId": "region-escape-001",
  "name": "Combination Lock",
  "position": [2, 0, 4],
  "visualRef": "blob-combo-lock-locked",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "puzzleType": "combination-lock",
    "state": "locked",
    "requiredCode": null,
    "hint": "The answer is on the whiteboard.",
    "unlocksObjectId": "escape-door-lab"
  }
}
```

The server holds the solution (`requiredCode`) server-side and never
sends it to clients. The `unlocksObjectId` field tells the server
which object to update when the puzzle transitions to `"solved"`.

**Puzzle: Combination lock (solved state)**

After a correct `"activate"` interaction with the right code in the
`data` payload, the server patches the puzzle:

```json
{
  "id": "escape-puzzle-lock",
  "visualRef": "blob-combo-lock-open",
  "customProperties": {
    "puzzleType": "combination-lock",
    "state": "solved",
    "requiredCode": null,
    "hint": "The answer is on the whiteboard.",
    "unlocksObjectId": "escape-door-lab"
  }
}
```

**Door: Laboratory door (transitions through three states)**

```json
{
  "id": "escape-door-lab",
  "regionId": "region-escape-001",
  "name": "Laboratory Door",
  "position": [5, 0, 9.5],
  "visualRef": "blob-door-locked",
  "visualType": "image/svg+xml",
  "physicsMode": "static",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "state": "locked",
    "requiredPuzzleId": "escape-puzzle-lock",
    "transitions": {
      "locked": "Cannot open -- solve the combination lock first.",
      "unlocked": "Press use to open.",
      "open": "Door is open -- escape!"
    }
  }
}
```

State machine:
- `"locked"` → (puzzle solved by server) → `"unlocked"` (players did
  not touch the door; the server updated it as a side-effect of the
  puzzle being solved)
- `"unlocked"` → (player sends `"activate"` interaction on door) →
  `"open"`

When the server applies the puzzle-solved patch it simultaneously
patches the door's `customProperties.state` to `"unlocked"` and
`visualRef` to `"blob-door-unlocked"` in the same `SceneObject/set`
call. All clients receive the `StateChange` and update both objects.

**Item: Red key (before pickup)**

```json
{
  "id": "escape-item-key-red",
  "regionId": "region-escape-001",
  "name": "Red Key",
  "position": [8, 0, 3],
  "visualRef": "blob-key-red",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "customProperties": {
    "itemType": "key",
    "itemId": "key-red",
    "pickedUp": false
  }
}
```

**Item: Red key (after pickup)**

A player sends a `"grab"` interaction. The server:

1. Sets `pickedUp: true` and `interactable: false` on the item
   SceneObject (it disappears from the floor).
2. Appends `"key-red"` to `customProperties.inventory` on the game
   state object.

```json
{
  "id": "escape-item-key-red",
  "interactable": false,
  "visible": false,
  "customProperties": {
    "itemType": "key",
    "itemId": "key-red",
    "pickedUp": true
  }
}
```

The item is hidden rather than destroyed so the server can restore it
if the session is reset. The canonical record of what the team carries
lives on the game state object, not on the item object.

**Whiteboard (non-interactable clue)**

```json
{
  "id": "escape-clue-whiteboard",
  "regionId": "region-escape-001",
  "name": "Whiteboard",
  "position": [1, 0, 6],
  "visualRef": "blob-whiteboard-code",
  "visualType": "image/svg+xml",
  "physicsMode": "static",
  "interactable": false,
  "visible": true,
  "customProperties": {
    "clueType": "text",
    "text": "Remember: the year the element was discovered."
  }
}
```

Static clue objects use `interactable: false` and `physicsMode:
"static"`. They are purely visual and require no server-side logic.

### Interaction Model

**Puzzle solving:**

1. Player clicks a puzzle object (`"click"` interaction). Client
   displays a close-up UI panel (e.g., a keypad or dial).
2. Player enters the solution and confirms. Client sends an
   `"activate"` interaction with the attempted answer in `data`:

   ```json
   {
     "@type": "SceneInteractionEvent",
     "regionId": "region-escape-001",
     "objectId": "escape-puzzle-lock",
     "userId": "user:alice@example.com",
     "action": "activate",
     "data": { "code": "1898" }
   }
   ```

3. Server checks the answer. On success it patches the puzzle state
   to `"solved"` and, if `unlocksObjectId` is set, patches the linked
   object as a side-effect in the same `SceneObject/set` call. On
   failure the server returns an error action; the client displays
   a shake animation.

**Item pickup:**

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-escape-001",
  "objectId": "escape-item-key-red",
  "userId": "user:bob@example.com",
  "action": "grab",
  "data": null
}
```

Server responds by patching the item to `visible: false` and
appending the item to the game state object's inventory array.

**Door interaction:**

When the door's state is `"unlocked"`, a player can open it:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-escape-001",
  "objectId": "escape-door-lab",
  "userId": "user:alice@example.com",
  "action": "activate",
  "data": null
}
```

If the door is still `"locked"` the server rejects the interaction
with an error and the client displays the transition hint text.

**Using an inventory item on a locked object:**

Some puzzles require a specific item from inventory rather than a
code. The player selects the item from the shared inventory panel and
clicks the target object. The client sends an `"activate"` interaction
with the item identifier in `data`:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-escape-001",
  "objectId": "escape-puzzle-keylock",
  "userId": "user:carol@example.com",
  "action": "activate",
  "data": { "useItem": "key-red" }
}
```

The server checks that `"key-red"` is in `customProperties.inventory`
on the game state object, then transitions the puzzle to `"solved"`.

### Game State

The game state object tracks the timer, the shared inventory, and
the overall escape progress.

```json
{
  "id": "escape-state-001",
  "regionId": "region-escape-001",
  "name": "Game State",
  "position": [0, 0, 0],
  "visualRef": null,
  "visualType": null,
  "physicsMode": "none",
  "interactable": false,
  "visible": false,
  "customProperties": {
    "phase": "playing",
    "timerSecondsRemaining": 2847,
    "timerRunning": true,
    "puzzlesSolved": 1,
    "puzzlesTotal": 4,
    "inventory": ["key-red"],
    "escaped": false,
    "failedAttempts": 2,
    "playerCount": 3,
    "players": [
      "user:alice@example.com",
      "user:bob@example.com",
      "user:carol@example.com"
    ]
  }
}
```

**Timer:** The server decrements `timerSecondsRemaining` at 1 Hz
and patches the game state object. Clients receive `StateChange`
notifications and update a countdown display. When the timer reaches
zero the server sets `phase: "failed"` and `timerRunning: false`.

**Inventory:** `inventory` is a flat array of item identifier strings.
Because the room is cooperative, no `ownerId` is used for inventory
items -- the array belongs to the team. Clients render a shared
inventory panel visible to all players.

**Win condition:** When all puzzles are solved and the final exit door
is opened, the server sets `phase: "escaped"` and `escaped: true`.
All players in the region at that moment are recorded as having
escaped.

### Hidden Information

Escape rooms are mostly shared state -- all players see the same room,
the same puzzle states, and the same inventory. The only server-side
secrets are puzzle solutions.

**What the server withholds:**

- The `requiredCode` field on combination-lock puzzles is always
  `null` in client responses. The server holds the correct value
  in its internal storage and validates `"activate"` interactions
  against it without ever sending it to clients.
- Similarly, any puzzle that accepts an item (`requiredItem`) does not
  expose the expected value. Clients only learn a puzzle's requirement
  through in-room clue objects and narrative hints.

**What is fully shared:**

- All puzzle `state` values (`"locked"`, `"solved"`) are visible to
  all players simultaneously. There is no per-player puzzle state.
- The game state object (timer, inventory, solved count) is returned
  identically to all players. No per-player redaction is needed.
- Door states are visible to all players. When a door unlocks, every
  client observes the same `StateChange`.

### Variants

**Multi-room:** Use multiple SceneRegions linked by `environment.exitTo`
(a custom field pointing to the next region's id). When the team
escapes one room the server updates SceneAvatar positions to move
everyone to the next region.

**3D first-person:** Replace `viewHint: "2d-topdown"` with
`viewHint: "3d"` and `environment.renderMode: "3d"`. Objects get
full 3D positions. Players navigate with avatar movement. All the
interaction, inventory, and state-machine logic is identical.

**Hint system:** Add an `"activate"` interaction on a hint-request
SceneObject. Each use decrements a `hintsRemaining` counter in game
state and posts a hint message to the bound chat channel
(`SceneRegion.chatId`).


---

## 26. Virtual Classroom

A virtual classroom demonstrates Scene as a general-purpose
collaboration layer. There is no simulation, no game physics, and no
win condition. The value is in tri-capability integration: Scene
provides the spatial layout, VTC carries the lecture audio, and Chat
provides text discussion. A teacher controls which slides are visible,
students raise their hand or respond to a poll, and all three layers
stay synchronized through two binding fields on the SceneRegion.

### Region

The classroom is a flat 2D space (`viewHint: "2d-topdown"`) with no
simulation layer. `simulationUri` is `null` -- there is no real-time
physics engine. `chatId` binds the in-room text discussion channel.
`activeCallId` binds the live lecture call. Both fields together
constitute the full tri-capability binding.

```json
{
  "id": "region-classroom-001",
  "name": "CS 101: Introduction to Networking",
  "bounds": { "min": [0, 0, 0], "max": [20, 0, 15] },
  "viewHint": "2d-topdown",
  "spawnPosition": [10, 0, 12],
  "simulationUri": null,
  "accessPolicy": "invite",
  "chatId": "chat-cs101-room-001",
  "activeCallId": "call-lecture-cs101-001",
  "environment": {
    "roomTheme": "classroom",
    "gridVisible": false
  }
}
```

`chatId` links to a Chat channel where students post questions and
the teacher posts slide notes as system messages. `activeCallId`
links to the active VTCCall carrying the teacher's audio and video.
When the lecture ends and the VTCCall transitions to `"ended"`, the
server automatically clears `activeCallId` to `null` and emits a
`StateChange` for the SceneRegion -- no manual cleanup required.

> **Tri-capability integration.** `chatId` and `activeCallId` on the
> SceneRegion are the two binding fields that tie all three layers
> together. Chat delivers text; VTC delivers audio/video; Scene
> delivers spatial layout and interaction events. Each layer operates
> independently -- students can text-chat before the call starts, and
> the slide layout persists after the call ends.

For spatial audio configuration (how VTC participant audio is
positioned relative to SceneAvatars), see the
[JMAP Scene VTC Integration Guide](jmap-scene-vtc-integration-guide.md).

### Objects

The classroom contains three categories of SceneObjects: display
slides, interactive controls, and an invisible state tracker.

**Slide object (display-only):**

```json
{
  "id": "obj-slide-001",
  "regionId": "region-classroom-001",
  "name": "Slide 1: OSI Model",
  "position": [10, 0, 2],
  "visualRef": "blob-slide-osi-model",
  "visualType": "image/png",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:teacher@example.com",
  "customProperties": {
    "objectKind": "slide",
    "slideIndex": 1,
    "title": "OSI Model"
  }
}
```

Slide objects are `interactable: false` -- students cannot move or
manipulate them. The teacher advances slides by patching `visible`
on the current and next slide objects (see Teacher Controls below).

**Exhibit object (reference diagram, display-only):**

```json
{
  "id": "obj-exhibit-001",
  "regionId": "region-classroom-001",
  "name": "TCP/IP Stack Diagram",
  "position": [3, 0, 6],
  "visualRef": "blob-diagram-tcpip",
  "visualType": "image/png",
  "physicsMode": "none",
  "interactable": false,
  "visible": true,
  "ownerId": "user:teacher@example.com",
  "customProperties": {
    "objectKind": "exhibit",
    "caption": "TCP/IP Stack vs OSI Model"
  }
}
```

**Raise-hand button (interactive, per-student):**

Each student has an interactable button object. Clicking it sends a
`SceneInteractionEvent` with `action: "activate"`. The server records
the raise in the classroom state object (see Game State below).

```json
{
  "id": "obj-hand-alice",
  "regionId": "region-classroom-001",
  "name": "Raise Hand",
  "position": [4, 0, 13],
  "visualRef": "blob-icon-hand",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:alice@example.com",
  "customProperties": {
    "objectKind": "hand-raise-button",
    "forStudent": "user:alice@example.com"
  }
}
```

**Poll object (interactive, visibility-filtered responses):**

The poll object is visible to all participants, but each student's
response is stored under their own key in `customProperties`. The
server enforces that a student may only write to their own response
key. Students see the poll question and their own answer; the teacher
sees all responses.

```json
{
  "id": "obj-poll-001",
  "regionId": "region-classroom-001",
  "name": "Quick Poll",
  "position": [16, 0, 6],
  "visualRef": "blob-ui-poll",
  "visualType": "image/svg+xml",
  "physicsMode": "none",
  "interactable": true,
  "visible": true,
  "ownerId": "user:teacher@example.com",
  "customProperties": {
    "objectKind": "poll",
    "question": "Which layer handles routing?",
    "options": ["Layer 2 (Data Link)", "Layer 3 (Network)", "Layer 4 (Transport)"],
    "responses": {
      "user:alice@example.com": "Layer 3 (Network)",
      "user:bob@example.com": "Layer 2 (Data Link)",
      "user:carol@example.com": "Layer 3 (Network)"
    }
  }
}
```

> **Visibility filtering.** The `responses` map contains per-student
> answers. The server returns the full `responses` object only to the
> teacher (owner). For each student it returns only their own key,
> filtering out other students' answers. This uses the same
> server-side `customProperties` filtering pattern as hidden card
> hands in card games.

### Interaction Model

**Student raises hand:**

The student clicks their raise-hand button. The client sends a
`SceneInteractionEvent` over the WebSocket:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-classroom-001",
  "objectId": "obj-hand-alice",
  "userId": "user:alice@example.com",
  "action": "activate",
  "data": null
}
```

The server appends `user:alice@example.com` to `raisedHands` in the
classroom state object and fans the event out to the teacher's
client so the teacher sees a raised-hand indicator in real time.

**Student submits poll answer:**

The student clicks an option on the poll object. The client sends an
`activate` interaction with the selected answer in `data`:

```json
{
  "@type": "SceneInteractionEvent",
  "regionId": "region-classroom-001",
  "objectId": "obj-poll-001",
  "userId": "user:bob@example.com",
  "action": "activate",
  "data": { "response": "Layer 2 (Data Link)" }
}
```

The server writes `responses["user:bob@example.com"]` into the poll
object's `customProperties`. Bob's client receives a confirmation
showing his own answer. The teacher's client receives the full
updated `responses` map.

### Teacher Controls

The teacher advances the lecture with standard JMAP method calls --
no simulation layer is needed.

**Advance to next slide (hide slide 1, show slide 2):**

```json
[["SceneObject/set", {
  "accountId": "teacher-account",
  "update": {
    "obj-slide-001": { "visible": false },
    "obj-slide-002": { "visible": true }
  }
}, "0"]]
```

**Clear raised hands after calling on a student:**

```json
[["SceneObject/set", {
  "accountId": "teacher-account",
  "update": {
    "state-classroom-001": {
      "customProperties/raisedHands": []
    }
  }
}, "0"]]
```

All connected clients receive a `StateChange` and fetch the updated
object state, so the hand-raise indicators disappear from the UI
in real time.

### Game State

An invisible, non-interactable SceneObject tracks shared classroom
state: which slide is active, whether a poll is open, and who has
raised their hand.

```json
{
  "id": "state-classroom-001",
  "regionId": "region-classroom-001",
  "name": "Classroom State",
  "position": [0, 0, 0],
  "visualRef": null,
  "physicsMode": "none",
  "interactable": false,
  "visible": false,
  "ownerId": "user:teacher@example.com",
  "customProperties": {
    "objectKind": "classroom-state",
    "currentSlideIndex": 2,
    "totalSlides": 8,
    "activePollId": "obj-poll-001",
    "raisedHands": [
      "user:alice@example.com",
      "user:carol@example.com"
    ],
    "phase": "lecture"
  }
}
```

`visible: false` means students cannot see this object. Only the
teacher (owner) and server-side logic can read it.

### Role-Based Visibility Summary

| Object kind        | Teacher sees        | Student sees              |
|--------------------|---------------------|---------------------------|
| Slide (active)     | Full object         | Full object               |
| Slide (hidden)     | Full object         | Not returned by server    |
| Exhibit            | Full object         | Full object               |
| Poll               | All responses       | Own response only         |
| Hand-raise button  | All buttons         | Own button only           |
| Classroom state    | Full object         | Not returned by server    |

The server enforces visibility by filtering `SceneObject/get` and
`SceneObject/query` results based on `ownerId` and the `visible`
flag. No client-side trust is required.

### What Makes This Interesting for the Spec

- **Non-game use case.** Scene is a general spatial-state layer, not
  a game engine wrapper. No simulation, no physics, no win condition.

- **Tri-capability integration.** A single SceneRegion binds all
  three capabilities: `chatId` for text discussion, `activeCallId`
  for lecture audio/video, and the Scene spatial layer for slides and
  interaction. These three fields together are the complete
  integration surface.

- **Role-based visibility.** Teacher and students see different
  subsets of the same SceneObject graph. The poll object holds all
  responses but each student sees only their own. The classroom state
  object is invisible to students entirely. This is the same
  mechanism as hidden card hands in card games, applied to a
  non-game context.

- **Interactable objects for audience participation.** Raise-hand
  buttons and poll objects are standard interactable SceneObjects.
  No new primitives are needed -- `SceneInteractionEvent` with
  `action: "activate"` covers both use cases.

- **No simulation layer.** `simulationUri: null`. All state
  transitions (slide advance, poll response, hand raise) are handled
  by JMAP method calls directly. This is the simplest possible
  deployment profile.


---

## Implementation Complexity Summary

### Board, Card, and Cooperative Activities

| Game | Layout | Players | Objects | Hidden Info | Complexity |
|---|---|---|---|---|---|
| Tic-Tac-Toe | 3x3 | 2 | 0-9 | None | Trivial |
| Checkers | 8x8 | 2 | 24 | None | Low |
| Go | 9-19x | 2 | 0-361 | None | Medium (ko, scoring) |
| Chess | 8x8 | 2 | 32 | None | Medium (special moves) |
| Old Maid | Fan layout | 2-5 | 51 cards | Hands | Low |
| Go Fish | Fan layout | 2-6 | 52 cards | Hands | Low |
| Solitaire | Tableau | 1 | 52 cards | Face-down | Low |
| Sorry! | Track | 2-4 | 16 + cards | Draw pile | Medium |
| Battleship | 10x10 x2 | 2 | 10 + markers | Ship positions | Medium (visibility) |
| Stratego | 10x10 | 2 | 80 | Piece ranks | High (visibility) |
| Monopoly | Track | 2-8 | Tokens + props | Chance/CC deck | High (trading) |
| Catan | Hex grid | 3-6 | ~100+ | Resources, cards | High (trading, hex) |
| Pool (Billiards) | 2.54x1.27 m | 2 | 16 + table | None | Medium (physics, on-demand sim) |
| Escape Room | Single room | 2-8 | 10-30 | Puzzle solutions | Medium (state chains) |
| Virtual Classroom | 2D flat | 5-30 | 5-20 | Poll responses, state | Low (no sim, tri-capability) |

### Video Games

| Game | viewHint | Dimensionality | Players | Sim Rate | Objects | Complexity |
|---|---|---|---|---|---|---|
| Pitfall | 2d-side | 2D | 1-4 | 30-60 Hz | ~100 | Medium (physics) |
| Asteroids | 2d-topdown | 2D | 1-4 | 60 Hz | 20-100 | Medium (wrap, split) |
| Battlezone | 3d | 2.5D | 1-8 | 30 Hz | 20-50 | Medium (AI, radar) |
| Doom | 3d | 2.5D | 1-16 | 35+ Hz | 100-500 | High (BSP, AI, weapons) |
| Quake | 3d | True 3D | 1-16 | 77 Hz | 100-500 | High (mesh, physics) |
| Descent | 3d | True 3D (6DOF) | 1-8 | 30+ Hz | 50-200 | High (6DOF, tunnels) |
| Flight Combat | 3d | True 3D | 1-8 | 30+ Hz | 20-100 | High (AI, tracking) |

### Genre Complexity

| Genre | viewHint | Dimensionality | Key Challenge |
|---|---|---|---|
| Board/card | 2d-topdown | 2D | Turn logic, hidden info filtering |
| Top-down arcade | 2d-topdown | 2D | High-frequency sim, collision |
| Platformer | 2d-side | 2D | Jump physics, level design |
| Fighting | 2d-side | 2D | Hitbox precision, combo state |
| 2.5D FPS | 3d | 2.5D | BSP sectors, auto-aim, billboards |
| True 3D FPS | 3d | True 3D | Mesh collision, 3D physics, prediction |
| 6DOF | 3d | True 3D | Full quaternion orientation, zero-G |
| Third-person 3D | 3d | True 3D | Camera collision, animation blending |
| Survival/crafting | 3d | True 3D | Object creation rate, large worlds |

### Dimensionality Spectrum

```
2D top-down    2D side       2.5D           True 3D        6DOF
(Chess,        (Pitfall,     (Doom,         (Quake,        (Descent)
 Asteroids)     Mario)        Duke3D)        Minecraft)
    |              |              |              |              |
 viewHint:      viewHint:     viewHint:      viewHint:      viewHint:
 2d-topdown     2d-side       3d             3d             3d
                              renderMode:    renderMode:    renderMode:
                              2.5d           3d             3d
                              verticalAim:   verticalAim:   gravity: 0
                              auto           free           sixDOF: true
```

**Recommended implementation order for a new Scene game server:**
Tic-Tac-Toe (basic validation), Checkers (captures), Old Maid
(hidden info), Solitaire (single-player, complex layout), Chess
(special moves), Asteroids (real-time simulation), Pitfall (2d-side
physics), Battlezone (2.5D basics), Doom (2.5D FPS), Quake (true 3D
FPS), Descent (6DOF).
