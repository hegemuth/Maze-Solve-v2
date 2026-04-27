# Game Engine Specification
# Maze Traversal / Solver Puzzle Game
# TypeScript + React + Vite + PixiJS

## 0. Purpose of this document

This document defines the core game engine specification for a browser-based maze traversal puzzle game.

The goal is to build a reusable, data-driven maze engine where every level is loaded from JSON and can define:

- Maze size
- Maze layout
- Start and finish positions
- Active rules
- Special tile mechanics
- Move budget
- Time limit
- Checkpoints
- Visual tile/sprite information

The game should be implemented using:

- TypeScript
- React
- Vite
- Browser only, no backend initially
- PixiJS or Canvas for rendering the maze board
- React for menus, HUD, overlays, buttons, screens, and modals

The most important architectural rule:

The game engine must be independent from React and PixiJS.

React and PixiJS should consume game state, not own the gameplay rules.

The engine should be written as pure TypeScript logic as much as possible so that it can be tested without rendering anything.

---

# 1. High-level concept

The player controls a character on a tile-based maze.

The player starts at a start tile and moves one tile at a time using arrow keys.

The basic objective is to reach the finish tile.

Depending on the level configuration, the player may also need to obey additional rules, such as:

- Cannot step on any previously visited tile
- Must collect all checkpoints before the finish counts as completed
- Must finish within a move budget
- Must finish within a time limit

The level may also include special tiles, such as:

- Mud tiles
- Floating tiles
- Teleport tiles
- Refill tiles

The engine must support levels with:

- No rules and no special tiles
- One active rule
- Multiple active rules
- One special tile type
- Multiple special tile types
- Any combination of supported rules and mechanics

Example level configurations:

- 10x10 maze, no rules, no special tiles
- 10x10 maze, no backtracking + checkpoints
- 15x12 maze, move limit + mud tiles
- 12x12 maze, all rules + many special tiles

---

# 2. Main design principle

The game should be data-driven.

A single reusable GameScreen should be able to load any valid level JSON and run it using the same engine.

The level JSON defines the content.

The engine defines the rules.

The renderer displays the current state.

The UI displays information and feedback.

Recommended data flow:

Level JSON
  -> normalize and validate level
  -> build level lookup maps
  -> create initial GameState
  -> process player input through engine
  -> update GameState
  -> React HUD and PixiJS renderer display GameState

The engine must not depend on:

- React components
- PixiJS objects
- DOM APIs
- CSS
- UI images
- Browser routing

The renderer must not decide:

- Whether a move is legal
- Whether the player won
- Whether a rule was violated
- How much a move costs
- Whether a tile is consumed
- Whether a checkpoint is collected

The renderer only displays the result of the engine state.

---

# 3. Core gameplay contract

The following design decisions are locked for the first implementation:

1. Movement is strict tile-by-tile using arrow keys.
2. Checkpoints can be collected in any order.
3. Refill tiles are one-time use.
4. Mud cost applies when leaving the mud tile.
5. Floating tiles disappear after the player leaves them.
6. No-backtracking means the player cannot step on any previously visited tile.
7. The finish tile can be entered before all checkpoints are collected, but the level does not complete until all required checkpoints are collected.
8. Levels are manually authored JSON files.
9. Maze logic is tile-defined, but displayed using sprites.
10. First implementation priority is the maze/rules engine, not menu polish.

---

# 4. Main engine responsibilities

The game engine is responsible for:

- Loading normalized level data
- Creating initial game state
- Receiving movement commands
- Calculating target position
- Validating movement
- Applying rule checks
- Applying movement cost
- Applying tile effects
- Updating visited tiles
- Updating consumed tiles
- Updating disappeared tiles
- Updating checkpoint state
- Checking win conditions
- Checking loss conditions
- Producing feedback events for UI and rendering

The game engine is not responsible for:

- Drawing sprites
- Playing animations
- Playing sounds directly
- Rendering buttons
- Rendering menus
- Managing route navigation
- Displaying modals
- Loading image assets directly
- Styling UI

---

# 5. Runtime architecture

Recommended modules:

src/game/model/
  Direction.ts
  Coordinate.ts
  Tile.ts
  LevelDefinition.ts
  GameState.ts
  GameEvent.ts
  RuleConfig.ts
  TileEffectConfig.ts

src/game/level/
  loadLevel.ts
  normalizeLevel.ts
  validateLevel.ts
  buildLevelLookups.ts

src/game/engine/
  createInitialGameState.ts
  processMove.ts
  movement/
    getTargetPosition.ts
    validateMove.ts
    calculateMoveCost.ts
  rules/
    noBacktrackingRule.ts
    checkpointRule.ts
    moveLimitRule.ts
    timeLimitRule.ts
  effects/
    mudEffect.ts
    refillEffect.ts
    floatingEffect.ts
    teleportEffect.ts
  winLoss/
    checkWinCondition.ts
    checkLossCondition.ts

src/game/rendering/
  pixi/
    MazePixiView.tsx
    PixiMazeRenderer.ts
    TileLayer.ts
    PlayerLayer.ts
    EffectsLayer.ts
    OverlayLayer.ts

src/screens/GameScreen/
  GameScreen.tsx
  GameHUD.tsx
  GameResultModal.tsx

src/store/
  useGameSessionStore.ts
  useProgressStore.ts
  useSettingsStore.ts

---

# 6. Core data concepts

## 6.1 Coordinate

A coordinate identifies one tile on the board.

Coordinates use zero-based indexing.

x = column
y = row

Top-left tile is:

x = 0
y = 0

Example:

{ x: 3, y: 5 }

Means column 3, row 5.

## 6.2 Tile key

Internally, coordinates should be converted to string keys for fast lookup in sets and maps.

Recommended format:

"x,y"

Examples:

"0,0"
"3,5"
"10,2"

Tile keys are useful for:

- visitedTiles
- disappearedTiles
- consumedTiles
- checkpoint lookup
- mud lookup
- refill lookup
- teleport lookup
- floating lookup

## 6.3 Direction

Supported movement directions:

- up
- down
- left
- right

Direction offsets:

up:
  dx = 0
  dy = -1

down:
  dx = 0
  dy = 1

left:
  dx = -1
  dy = 0

right:
  dx = 1
  dy = 0

The engine should only accept these four movement commands for player movement.

Diagonal movement is not supported.

---

# 7. Base tile types

The base grid should define the physical maze.

Supported base tile types:

## wall

A wall tile blocks movement.

The player cannot enter a wall tile.

## floor

A normal walkable tile.

## start

The player's initial position.

There must be exactly one start tile in each level.

The start tile is considered walkable.

The start tile is considered visited at the beginning of the level.

## finish

The exit/goal tile.

The finish tile is walkable.

The player may enter the finish tile before collecting all checkpoints.

If all required objectives are completed when entering the finish tile, the level is won.

If objectives are not completed, the player remains on the finish tile and the level continues.

## checkpoint

A walkable tile that marks progress toward the level objective.

Each checkpoint must have a checkpointId.

Checkpoint IDs should be unique per level.

Checkpoints can be collected in any order.

A checkpoint is considered collected when the player enters its tile.

---

# 8. Special tile mechanics

Special tile mechanics are defined separately from the base grid.

This allows one base tile to also have a special behavior.

Examples:

- floor + mud
- floor + refill
- floor + teleport
- checkpoint + floating
- finish + refill

The base tile decides if the tile is physically walkable.

The special mechanic decides what happens when the player enters or leaves that tile.

Supported special mechanics:

1. Mud tile
2. Floating tile
3. Refill tile
4. Teleport tile

---

# 9. Special tile behavior

## 9.1 Mud tile

Mud affects the cost of leaving the tile.

Mud does not apply when entering the tile.

Example:

Player moves from normal floor to mud tile:
  cost = cost of leaving normal floor

Player moves from mud tile to normal floor:
  cost = mud exit cost

Recommended property name:

exitMoveCost

Example:

mud tile at x=4, y=2 has exitMoveCost = 3

This means any successful move away from tile 4,2 costs 3 moves total.

If no move limit is active, mud still records move cost in move history, but it may not cause a loss.

If a move limit is active and the player does not have enough remaining moves to pay the mud exit cost, the move is rejected.

Mud tile trigger:

onBeforeLeave

## 9.2 Floating tile

Floating tiles disappear after the player leaves them.

Behavior:

- Player may enter a floating tile if it has not disappeared.
- While the player stands on the floating tile, the tile remains active.
- After the player successfully moves away from the floating tile, the tile disappears.
- Once disappeared, the tile becomes non-walkable.
- The player cannot step on a disappeared tile again.

Floating tile trigger:

onAfterLeave

Important:

The tile should disappear only after a successful move away from it.

If the player attempts an invalid move while standing on a floating tile, the tile should not disappear.

## 9.3 Refill tile

Refill tiles add moves to the player's remaining move budget.

Behavior:

- Refill tiles trigger when the player enters the tile.
- Refill tiles are one-time use.
- After being used, they are marked as consumed.
- A consumed refill tile remains walkable, but no longer gives moves.
- Refill amount is defined in the level JSON.

Refill tile trigger:

onAfterEnter

Recommended properties:

amount
oneTime

For the initial version, all refill tiles are one-time use.

Refill may increase remaining moves above the original move limit unless the level later defines a max move cap.

Default MVP behavior:

Refill can overfill beyond the starting move budget.

## 9.4 Teleport tile

Teleport tiles move the player from one endpoint to the paired endpoint.

Behavior:

- Teleport triggers when the player enters a teleport tile.
- Teleporting costs no extra moves.
- The move cost is paid only for the movement into the teleport tile.
- Teleport happens as part of the same move.
- The final player position becomes the teleport destination.
- Teleport pairs are symmetrical.
- Only one teleport activation is allowed per move.
- Teleport chains are not activated in the same move.

Example:

Teleport pair A has endpoints:

Endpoint 1: x=2, y=3
Endpoint 2: x=8, y=3

If the player steps onto 2,3:
  player is moved to 8,3

If the player steps onto 8,3:
  player is moved to 2,3

Teleport tile trigger:

onAfterEnter

Important no-backtracking behavior:

If no-backtracking is active, both the teleport entry tile and the teleport destination tile should be considered.

Recommended rule:

- The player cannot intentionally enter a teleport tile if the teleport destination is already visited.
- The final teleport destination must be valid.
- If the destination is invalid, the entire move is rejected before the state is committed.

---

# 10. Rules

Rules are optional and configured per level.

Supported rules for the first version:

1. No backtracking
2. Checkpoint requirement
3. Move limit
4. Time limit

Rules should be modular.

A level can activate zero, one, multiple, or all rules.

Rules should not be hardcoded based on level category.

---

## 10.1 No-backtracking rule

Meaning:

The player cannot step on any tile that has already been visited.

Visited tiles include:

- The start tile at level start
- Every tile the player has entered
- The final teleport destination if teleporting occurs

If the player attempts to move onto a visited tile, the move is rejected.

This rule is checked before committing movement.

Important:

This is stricter than only blocking immediate reverse movement.

Example:

Path:

start -> A -> B -> C

With no-backtracking active, the player cannot later move to:

start
A
B
C

unless the current tile is C and C is only occupied by the player. The current tile being occupied is allowed, but trying to enter a visited tile as a movement target is not allowed.

The no-backtracking rule applies to the target tile and, if teleporting, to the final teleport destination.

## 10.2 Checkpoint requirement rule

Meaning:

The player must collect all required checkpoints before the level can be completed.

Checkpoints can be collected in any order.

A checkpoint is collected when the player enters the checkpoint tile.

If the player enters the finish tile before all checkpoints are collected:

- The player is allowed to stand on the finish tile.
- The level does not complete.
- The game shows a feedback message.
- The level continues.

Recommended feedback:

"Exit reached, but not all checkpoints are collected."

Important with no-backtracking:

If the player enters finish too early and no-backtracking is active, the finish tile becomes visited.

This may make the route unsolvable.

This is allowed and is considered part of the puzzle.

Recommended feedback in this case:

"You reached the exit before collecting all checkpoints. Because No Backtracking is active, this route may no longer be solvable."

## 10.3 Move limit rule

Meaning:

The level has a limited move budget.

Movement consumes moves.

Normal tile exit cost:

1

Mud tile exit cost:

configured exitMoveCost

Refill tiles can add moves.

If the player attempts a move and does not have enough remaining moves to pay the movement cost, the move is rejected.

Loss condition:

The player loses when there are no valid moves left because of the move budget, or when the move budget reaches 0 and the player has not completed the level.

Simpler MVP behavior:

If remainingMoves becomes 0 after a move and the level is not won, status becomes lost.

Optional future improvement:

Only lose when remainingMoves is 0 and no zero-cost actions or refills are possible. Since movement always costs at least 1 in MVP, immediate loss at 0 is acceptable.

## 10.4 Time limit rule

Meaning:

The level has a countdown timer.

The engine should track elapsed time or remaining time.

If time expires before the level is completed, the player loses.

Important:

Timer logic should not be deeply tied to rendering.

React/Zustand can own the interval, but the engine should own the rule evaluation.

Recommended model:

- UI/store ticks once per second or using requestAnimationFrame.
- The tick updates elapsedSeconds or remainingSeconds.
- The engine checks if timeRemaining <= 0.
- If time is expired and status is playing, game status becomes lost.

For MVP, time limit can be implemented after move rules.

---

# 11. Game status

Possible game status values:

not_started
playing
paused
won
lost

Recommended usage:

not_started:
  Level is loaded but gameplay has not started yet.

playing:
  Player can move and timer can run.

paused:
  Player cannot move and timer should stop.

won:
  Level completed successfully. Player cannot move.

lost:
  Level failed. Player cannot move.

The processMove function should only accept movement if status is playing.

---

# 12. GameState model

The runtime game state should contain:

GameState:
  levelId
  status

  player:
    position
    previousPosition
    facingDirection

  moves:
    movesUsed
    remainingMoves
    moveHistory

  timer:
    elapsedSeconds
    remainingSeconds

  visitedTiles
  disappearedTiles
  consumedTiles

  checkpoints:
    requiredCheckpointIds
    collectedCheckpointIds

  rules:
    activeRules

  feedback:
    lastMessage
    lastEventType

  events:
    latestEvents

Recommended details:

## player.position

Current player position.

If teleport occurs, this should be the final destination after teleporting.

## player.previousPosition

The position before the latest successful move.

Useful for animation.

## player.facingDirection

The last movement direction.

Useful for sprite direction.

## moves.movesUsed

Total accumulated move cost used.

Normal moves usually add 1.

Leaving mud may add more than 1.

## moves.remainingMoves

Remaining move budget.

If no move limit is active, this may be null or undefined.

## moves.moveHistory

Array of successful movement records.

Each record may contain:

- from position
- requested target position
- final position
- direction
- cost
- tile effects triggered
- checkpoint collected
- timestamp or turn number

## visitedTiles

Set of tile keys the player has entered.

The start tile is added at level start.

The current tile is always considered visited.

## disappearedTiles

Set of tile keys that are no longer walkable because of floating tile behavior.

## consumedTiles

Set of tile keys that have already triggered a one-time effect.

Used for:

- Refill tiles

May later be used for:

- One-time switches
- Keys
- Traps

## checkpoints.requiredCheckpointIds

All checkpoint IDs that must be collected.

## checkpoints.collectedCheckpointIds

Checkpoint IDs already collected.

## feedback.lastMessage

Human-readable message for UI.

Examples:

- "Blocked by wall."
- "You cannot step on a visited tile."
- "Not enough moves."
- "Checkpoint A collected."
- "Exit reached, but not all checkpoints are collected."
- "Level complete."

## feedback.lastEventType

Machine-readable event type for UI/rendering.

Examples:

invalid_wall
invalid_visited
invalid_disappeared
invalid_out_of_bounds
invalid_not_enough_moves
checkpoint_collected
refill_collected
teleport_used
floating_tile_disappeared
finish_reached_incomplete
level_won
level_lost

## events.latestEvents

A list of events created by the latest engine action.

This lets the renderer and UI animate multiple things from a single move.

Example:

Player leaves mud, enters refill, then wins.

Events:

- move_success
- refill_collected
- finish_reached
- level_won

---

# 13. LevelDefinition model

The level JSON should contain:

LevelDefinition:
  id
  title
  description
  category
  difficulty

  board:
    width
    height
    tiles

  rules:
    noBacktracking
    checkpointsRequired
    moveLimit
    timeLimit

  mechanics:
    mudTiles
    floatingTiles
    refillTiles
    teleportPairs

  visuals:
    tileset
    background
    playerSprite
    theme

  tutorial:
    enabled
    prompts

---

# 14. Board grid model

The board should define width, height, and tiles.

width = number of columns
height = number of rows

tiles should be a 2D array with height rows and width columns.

Each tile should be an object.

Example:

{
  "id": "basic-level-01",
  "title": "First Maze",
  "description": "Reach the exit.",
  "category": "puzzles",
  "difficulty": "easy",
  "board": {
    "width": 5,
    "height": 5,
    "tiles": [
      [
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "start" },
        { "type": "floor" },
        { "type": "finish" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "floor" },
        { "type": "wall" },
        { "type": "floor" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "floor" },
        { "type": "floor" },
        { "type": "floor" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" }
      ]
    ]
  }
}

---

# 15. Rule config model

Recommended rule config:

rules:
  noBacktracking:
    enabled: true

  checkpointsRequired:
    enabled: true

  moveLimit:
    enabled: true
    maxMoves: 30

  timeLimit:
    enabled: false
    seconds: 120

Alternative simpler MVP structure:

rules:
  noBacktracking: true
  checkpointsRequired: true
  moveLimit: 30
  timeLimitSeconds: null

The object-based version is recommended because it is easier to extend later.

---

# 16. Mechanics config model

Recommended mechanics config:

mechanics:
  mudTiles:
    - position:
        x: 3
        y: 2
      exitMoveCost: 3

  floatingTiles:
    - position:
        x: 4
        y: 4

  refillTiles:
    - position:
        x: 5
        y: 6
      amount: 5
      oneTime: true

  teleportPairs:
    - pairId: "A"
      endpoints:
        - x: 2
          y: 3
        - x: 8
          y: 3

Rules:

- mechanics may be empty
- each mechanic array may be empty
- special mechanic positions must be inside the board
- special mechanic positions must not be walls
- teleport pairs must have exactly two endpoints
- refill amount must be positive
- mud exitMoveCost must be greater than or equal to 1
- floating tiles should not be placed on walls

---

# 17. Level normalization

Before a level is played, the raw JSON should be normalized.

Normalization should:

1. Validate board dimensions.
2. Find the start tile.
3. Find the finish tile.
4. Find all checkpoints.
5. Build coordinate lookup maps.
6. Build mechanic lookup maps.
7. Build rule config with defaults.
8. Convert arrays to maps/sets where useful.
9. Produce a NormalizedLevel object.

The game engine should use NormalizedLevel, not raw JSON.

---

# 18. NormalizedLevel model

NormalizedLevel should contain:

NormalizedLevel:
  id
  title
  description
  category
  difficulty

  width
  height
  tiles

  startPosition
  finishPositions
  checkpointMap

  rules

  lookups:
    wallTiles
    checkpointTiles
    mudTiles
    floatingTiles
    refillTiles
    teleportTiles

  visuals
  tutorial

Recommended lookup examples:

wallTiles:
  Set of tile keys

checkpointTiles:
  Map tileKey -> checkpointId

mudTiles:
  Map tileKey -> mudConfig

floatingTiles:
  Set of tile keys

refillTiles:
  Map tileKey -> refillConfig

teleportTiles:
  Map tileKey -> teleportDestination

---

# 19. Level validation rules

A level is valid if:

1. board.width is greater than 0.
2. board.height is greater than 0.
3. board.tiles has exactly board.height rows.
4. every row has exactly board.width tiles.
5. every tile has a valid type.
6. exactly one start tile exists.
7. at least one finish tile exists.
8. checkpoint tiles have checkpointId.
9. checkpoint IDs are unique.
10. all mechanic positions are inside the board.
11. no mechanic is placed on a wall unless explicitly allowed in the future.
12. teleport pairs have exactly two endpoints.
13. teleport endpoint positions are unique inside the same pair.
14. refill amount is positive.
15. mud exitMoveCost is at least 1.
16. moveLimit.maxMoves is positive if enabled.
17. timeLimit.seconds is positive if enabled.

Recommended validation behavior:

- During development, invalid levels should throw clear errors.
- In production, invalid levels should fail gracefully and show an error screen.

Example validation errors:

- "Level basic-01 has no start tile."
- "Level mud-03 has a mud tile outside the board at x=10, y=5."
- "Level teleport-02 has teleport pair A with only one endpoint."
- "Level checkpoint-04 has duplicate checkpointId A."

---

# 20. Movement processing pipeline

This is the most important part of the engine.

Every move should follow the same pipeline.

Recommended exact order:

1. Receive movement direction.
2. Check that game status is playing.
3. Calculate target coordinate from current position and direction.
4. Check target is inside the board.
5. Check target is not a wall.
6. Check target has not disappeared.
7. Check no-backtracking rule against target tile.
8. Check teleport destination validity if target tile is a teleport.
9. Calculate move cost based on the tile the player is leaving.
10. Check move budget if move limit is active.
11. Commit movement to target tile.
12. Apply target tile effects:
    - checkpoint collection
    - refill collection
    - teleport
13. If teleport occurred, update final player position.
14. Apply final-position checkpoint collection if teleport lands on checkpoint.
15. Mark previous tile as visited if needed.
16. If previous tile was floating, mark it as disappeared.
17. Mark final player position as visited.
18. Update move history.
19. Check finish condition.
20. Check loss conditions.
21. Return updated GameState and events.

Important:

Steps 3 to 10 are validation and preparation.

Steps 11 onward commit the new state.

If validation fails, the engine should return the old GameState plus an invalid-move event.

Invalid moves should not:

- consume moves
- change player position
- collect checkpoints
- consume refill tiles
- disappear floating tiles
- update move history

---

# 21. Move cost calculation

Move cost is based on the tile the player is leaving.

Default exit cost:

1

If current tile is a mud tile:

exit cost = mudTile.exitMoveCost

If multiple cost modifiers are added in the future:

- define a clear priority system
- for MVP, only mud affects exit cost

Examples:

Normal floor -> floor:
  cost = 1

Normal floor -> mud:
  cost = 1

Mud with exitMoveCost 3 -> floor:
  cost = 3

Mud with exitMoveCost 3 -> teleport:
  cost = 3

Teleport destination does not add extra movement cost.

---

# 22. Win condition

The player wins when:

1. Player is on a finish tile.
2. All required checkpoints are collected, if checkpoint rule is enabled.
3. Game status is playing.

If checkpointsRequired is disabled:

- entering the finish tile wins immediately.

If checkpointsRequired is enabled:

- entering the finish tile wins only if all checkpoint IDs are collected.

If player enters finish early:

- do not win
- keep playing
- show feedback

---

# 23. Loss conditions

Possible loss conditions:

## Move limit loss

If move limit is active and remainingMoves reaches 0 before winning:

status = lost

Recommended MVP behavior:

- If the latest move does not win and remainingMoves <= 0, player loses.

## Time limit loss

If time limit is active and remainingSeconds <= 0 before winning:

status = lost

## Future possible loss conditions

Not required for MVP:

- trap tile
- enemy collision
- impossible route detection
- softlock detection
- puzzle validation failure

---

# 24. Finish tile behavior before checkpoints

The finish tile is walkable even if not all checkpoints are collected.

If player enters finish without all checkpoints:

- status remains playing
- player remains on finish
- finish tile is marked as visited
- feedback event is produced

If no-backtracking is active, this may prevent the player from completing the level later.

This is acceptable and should be treated as part of the puzzle.

Recommended event:

finish_reached_incomplete

Recommended feedback:

"Exit reached, but not all checkpoints are collected."

If no-backtracking is active:

"You reached the exit before collecting all checkpoints. Because No Backtracking is active, this route may no longer be solvable."

---

# 25. Teleport validation details

Teleport should be validated before committing movement.

If the target tile is a teleport tile, the engine should determine the destination.

Before allowing the move, validate:

1. Teleport entry tile is not a wall.
2. Teleport entry tile has not disappeared.
3. Teleport entry tile is not visited if no-backtracking is active.
4. Teleport destination is inside the board.
5. Teleport destination is not a wall.
6. Teleport destination has not disappeared.
7. Teleport destination is not visited if no-backtracking is active.

If any validation fails:

- reject the entire move
- do not move player to the teleport entry
- do not consume moves
- do not trigger effects

Teleport chains:

If the teleport destination is also a teleport tile, do not activate the second teleport in the same move.

Only one teleport activation per move.

Teleport and checkpoints:

If the teleport destination is a checkpoint, collect that checkpoint.

Teleport and finish:

If the teleport destination is a finish tile, check win condition after teleporting.

Teleport and floating tile:

If the teleport entry tile or destination tile is floating, floating behavior still follows the normal rule:

- the previous tile disappears after the player leaves it
- the teleport destination does not disappear until the player later leaves it

---

# 26. Refill behavior details

Refill tiles trigger when entered.

If the refill tile is not consumed:

1. Add configured amount to remainingMoves.
2. Mark tile as consumed.
3. Emit refill_collected event.

If already consumed:

1. Do not add moves.
2. Emit no refill event, or optionally emit refill_already_consumed for debugging.

Refill tile remains walkable after consumption.

If no move limit is active:

- Refill may do nothing.
- It may still emit an event for visual feedback if desired.
- Recommended MVP behavior: only apply refill if move limit is active.

---

# 27. Floating behavior details

Floating tiles disappear after the player successfully leaves them.

If the player is currently on a floating tile and successfully moves away:

1. Add previous tile key to disappearedTiles.
2. Emit floating_tile_disappeared event.
3. Tile becomes non-walkable for future moves.

If player attempts invalid movement from a floating tile:

- floating tile does not disappear.

If player teleports away from a floating tile:

- the tile still disappears because the player successfully left it.

If player enters a floating tile:

- it does not disappear immediately.

---

# 28. No-backtracking details

No-backtracking should use visitedTiles.

At level start:

visitedTiles contains start tile.

After each successful move:

visitedTiles contains final player position.

If teleporting:

- the teleport destination is the final player position and is added to visitedTiles.
- recommended: the teleport entry tile should also be considered visited because the player entered it during the move.

Recommended strict behavior:

If the player steps onto a teleport tile, both the teleport entry tile and destination tile become visited.

Reason:

The player physically entered the teleport tile before being moved.

This matters if the player later returns to the teleport entry tile.

Invalid movement:

Invalid movement should not add anything to visitedTiles.

Finish tile:

Finish tile is added to visitedTiles when entered, even if checkpoints are incomplete.

---

# 29. Checkpoint behavior details

Checkpoints are collected when the player enters a checkpoint tile.

If already collected:

- no duplicate collection occurs.

Checkpoint collection should emit:

checkpoint_collected

Event payload:

checkpointId
position

If teleport destination is a checkpoint:

- checkpoint is collected after teleport completes.

If checkpoint is also a refill tile:

Recommended effect order:

1. collect checkpoint
2. apply refill

Both events may be emitted.

If checkpoint is also finish:

Avoid this for MVP unless specifically needed.

---

# 30. Event system

The engine should produce events after each action.

Events allow UI and renderer to react without embedding logic.

Example event types:

move_success
move_invalid
invalid_out_of_bounds
invalid_wall
invalid_disappeared
invalid_visited
invalid_not_enough_moves
checkpoint_collected
refill_collected
teleport_used
floating_tile_disappeared
finish_reached
finish_reached_incomplete
level_won
level_lost
timer_expired

Each event should have:

type
message
payload

Example event:

type: "checkpoint_collected"
message: "Checkpoint A collected."
payload:
  checkpointId: "A"
  position:
    x: 4
    y: 2

The UI may show message.

The renderer may animate based on type.

The sound system may play audio based on type.

---

# 31. Input handling

Input should be separate from engine logic.

Keyboard controls:

ArrowUp -> direction up
ArrowDown -> direction down
ArrowLeft -> direction left
ArrowRight -> direction right

Optional alternative keys:

W -> up
A -> left
S -> down
D -> right

Input handler responsibilities:

- Listen for key presses
- Ignore input if game is paused/won/lost
- Convert key to direction
- Call processMove(direction)
- Prevent repeated movement if animation lock is active, if desired

The engine should not listen to keyboard events directly.

---

# 32. Animation and movement locking

The engine updates state immediately.

The renderer may animate movement.

Recommended approach:

- Engine processes move instantly.
- Store receives new state.
- Renderer animates player from previousPosition to position.
- During animation, input can be temporarily locked.

Input lock should be UI/store-level, not engine-level.

This prevents the player from submitting many moves before animations finish.

For debugging, instant movement should be supported.

---

# 33. Rendering requirements

The maze renderer should display:

- wall tiles
- floor tiles
- start tile
- finish tile
- checkpoint tiles
- mud tiles
- floating tiles
- refill tiles
- teleport tiles
- consumed refill state
- disappeared floating state
- player sprite
- optional visited tile overlay
- optional valid/invalid move feedback

The renderer should receive:

- NormalizedLevel
- GameState
- visual config
- latest events

The renderer should not:

- validate movement
- calculate move cost
- check rules
- modify game state directly

---

# 34. Sprite rendering model

The board is tile-defined but rendered using sprites.

Each base tile type should map to a sprite.

Example:

wall -> wall sprite
floor -> floor sprite
start -> start/floor sprite
finish -> finish sprite
checkpoint -> checkpoint sprite

Special tiles may add overlays:

mud -> mud overlay
floating -> floating platform sprite/overlay
refill -> refill item sprite
teleport -> teleport portal sprite

Recommended layer order:

1. Base floor layer
2. Wall layer
3. Special tile layer
4. Checkpoint/finish layer
5. Consumed/disappeared overlay layer
6. Player layer
7. Effects/particles layer
8. Debug overlay layer

Disappeared floating tiles:

- Should be rendered as gap/hole/disabled tile.
- Must not be walkable.

Consumed refill tiles:

- Should appear faded, empty, or inactive.

Visited tiles:

- Optional overlay, especially useful for no-backtracking levels.

---

# 35. Board scaling

The renderer must support different board sizes.

Examples:

- 10x10
- 15x12
- 12x12
- larger future mazes

The renderer should calculate tile size dynamically.

Given:

availableBoardWidth
availableBoardHeight
columns = board.width
rows = board.height

tileSize = min(
  availableBoardWidth / columns,
  availableBoardHeight / rows
)

boardPixelWidth = columns * tileSize
boardPixelHeight = rows * tileSize

offsetX = center board horizontally
offsetY = center board vertically

This allows any valid board size to fit into the maze area.

---

# 36. React HUD requirements

The HUD should display:

- Level title
- Active rules
- Remaining moves, if move limit is active
- Timer, if time limit is active
- Checkpoints collected / total
- Last feedback message
- Pause button
- Restart button
- Exit/back button

Example active rules display:

Rules:
- No Backtracking
- Collect all checkpoints
- Move limit: 30

Example checkpoint display:

Checkpoints: 2 / 4

Example move display:

Moves left: 12

Example timer display:

Time left: 01:15

---

# 37. GameScreen responsibilities

GameScreen should:

1. Load selected level by ID.
2. Normalize and validate level.
3. Create initial GameState.
4. Initialize input handling.
5. Render MazePixiView.
6. Render HUD.
7. Render pause/win/loss modals.
8. Connect UI actions to store:
   - restart
   - pause
   - resume
   - exit
9. Pass GameState and NormalizedLevel to renderer.

GameScreen should not contain low-level rule logic.

---

# 38. Store responsibilities

Recommended Zustand store:

useGameSessionStore

State:

- level
- gameState
- inputLocked
- latestEvents

Actions:

- loadLevel(levelId)
- startLevel()
- processMove(direction)
- tickTimer(deltaSeconds)
- pause()
- resume()
- restart()
- exitLevel()

The store may call pure engine functions.

Example flow:

User presses ArrowRight
  -> input handler calls store.processMove("right")
  -> store calls engine.processMove(level, gameState, "right")
  -> engine returns updated state and events
  -> store saves updated state
  -> UI and renderer update

---

# 39. Timer handling

Timer should be implemented carefully.

Recommended MVP approach:

- Store starts timer when level status becomes playing.
- Timer pauses when game status is paused/won/lost.
- Timer calls engine tick function.
- Engine checks time limit rule.

The engine can have a function:

processTick(gameState, deltaSeconds)

This function:

1. Adds elapsed time.
2. Updates remaining time.
3. Checks time limit.
4. Returns updated state and events.

Do not put timer countdown logic directly inside React components except for starting/stopping the interval.

---

# 40. Restart behavior

When the player restarts:

- GameState resets to initial state.
- consumedTiles resets.
- disappearedTiles resets.
- visitedTiles resets to start tile only.
- checkpoints reset.
- moves reset.
- timer resets.
- status becomes playing.

Level definition remains the same.

---

# 41. Pause behavior

When paused:

- Player input is ignored.
- Timer stops.
- Animations may pause if desired.
- Game state is preserved.

When resumed:

- Status returns to playing.
- Timer continues.
- Input is accepted again.

---

# 42. Progress tracking

For MVP, progress tracking can be simple.

Store in localStorage:

- completed level IDs
- best move count
- best time
- stars, if added later

Progress is not part of the core engine.

Progress belongs to app-level state.

The engine only reports level_won.

The app decides how to store progress.

---

# 43. Scoring and stars

Not required for first MVP.

Possible future scoring:

- 3 stars: complete under par moves and par time
- 2 stars: complete level
- 1 star: complete with hints/restarts
- bonus: no invalid moves

Level JSON may later include:

scoring:
  parMoves: 20
  parTimeSeconds: 60

Keep scoring outside the movement engine initially.

---

# 44. Tutorial support

Tutorial should reuse the real GameScreen and engine.

Tutorial levels should be normal levels with additional tutorial metadata.

Tutorial metadata may include:

tutorial:
  enabled: true
  steps:
    - trigger: "level_start"
      message: "Use arrow keys to move."
      highlight:
        x: 1
        y: 1

    - trigger: "checkpoint_collected"
      checkpointId: "A"
      message: "Great. Checkpoints can be collected in any order."

    - trigger: "finish_reached_incomplete"
      message: "The exit is not enough. Collect all checkpoints first."

The tutorial system should listen to engine events.

The tutorial system should not replace the engine.

---

# 45. Debug tools

During development, create debug display options:

- show coordinates on tiles
- show visited tiles
- show disappeared tiles
- show tile keys
- show move cost of mud tiles
- show teleport pair IDs
- show checkpoint IDs
- show current GameState as JSON
- allow reset level
- allow load test level

This will make level authoring much easier.

---

# 46. Manual level authoring requirements

Since levels are manually authored, the project should include helper tools or validation.

Minimum required:

- clear JSON schema
- validation errors
- human-readable maze preview
- coordinate debug overlay in-game

Recommended later:

- simple level editor
- script that validates all JSON levels
- script that checks if level is solvable
- script that exports human-readable maze previews

---

# 47. Solvability validation

Not required in engine MVP, but useful later.

Because rules can make mazes unsolvable, a future validation tool should check if a level can be completed.

For basic levels:

- BFS is enough.

For move costs:

- Dijkstra is better.

For no-backtracking:

- search becomes more complex because visited state matters.

For checkpoints:

- solver state must track collected checkpoints.

For teleport/floating/refill:

- solver state must include consumed/disappeared tiles and move budget.

This can become complex, so do not block MVP on automatic solvability.

For MVP, manually test levels.

---

# 48. Example complete level JSON

Example level using several systems:

{
  "id": "mixed-demo-01",
  "title": "First Mixed Maze",
  "description": "Collect both checkpoints and reach the exit without revisiting tiles.",
  "category": "puzzles",
  "difficulty": "medium",

  "board": {
    "width": 7,
    "height": 5,
    "tiles": [
      [
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "start" },
        { "type": "floor" },
        { "type": "checkpoint", "checkpointId": "A" },
        { "type": "floor" },
        { "type": "finish" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "floor" },
        { "type": "wall" },
        { "type": "floor" },
        { "type": "wall" },
        { "type": "floor" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "floor" },
        { "type": "floor" },
        { "type": "checkpoint", "checkpointId": "B" },
        { "type": "floor" },
        { "type": "floor" },
        { "type": "wall" }
      ],
      [
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" },
        { "type": "wall" }
      ]
    ]
  },

  "rules": {
    "noBacktracking": {
      "enabled": true
    },
    "checkpointsRequired": {
      "enabled": true
    },
    "moveLimit": {
      "enabled": true,
      "maxMoves": 20
    },
    "timeLimit": {
      "enabled": false,
      "seconds": null
    }
  },

  "mechanics": {
    "mudTiles": [
      {
        "position": { "x": 2, "y": 3 },
        "exitMoveCost": 3
      }
    ],
    "floatingTiles": [
      {
        "position": { "x": 4, "y": 1 }
      }
    ],
    "refillTiles": [
      {
        "position": { "x": 1, "y": 3 },
        "amount": 5,
        "oneTime": true
      }
    ],
    "teleportPairs": [
      {
        "pairId": "A",
        "endpoints": [
          { "x": 3, "y": 2 },
          { "x": 5, "y": 3 }
        ]
      }
    ]
  },

  "visuals": {
    "tileset": "default",
    "theme": "warm-puzzle-board",
    "background": "maze-screen-default",
    "playerSprite": "default-player"
  },

  "tutorial": {
    "enabled": false,
    "steps": []
  }
}

---

# 49. Engine function contracts

The main engine functions should conceptually be:

loadLevel(levelId)
  Loads raw JSON level data.

validateLevel(rawLevel)
  Checks level structure and throws useful errors.

normalizeLevel(rawLevel)
  Converts level JSON into NormalizedLevel.

createInitialGameState(normalizedLevel)
  Creates the initial runtime state.

processMove(normalizedLevel, gameState, direction)
  Processes one movement input and returns a new GameState.

processTick(normalizedLevel, gameState, deltaSeconds)
  Updates timer and checks time-based loss.

checkWinCondition(normalizedLevel, gameState)
  Returns whether level is completed.

checkLossCondition(normalizedLevel, gameState)
  Returns whether level is failed.

The processMove function is the core function.

It should be deterministic.

Same input level + same input state + same direction should always produce the same result.

---

# 50. Testing requirements

The engine should be tested independently from React and PixiJS.

Important tests:

## Basic movement

- player can move onto floor
- player cannot move into wall
- player cannot move out of bounds
- invalid move does not change state
- invalid move does not consume moves

## Finish

- entering finish wins if no checkpoints required
- entering finish does not win if checkpoints missing
- entering finish wins after checkpoints collected

## Checkpoints

- checkpoint is collected when entered
- checkpoints can be collected in any order
- duplicate checkpoint entry does not duplicate state

## No backtracking

- start tile is visited initially
- player cannot step on previously visited tile
- invalid visited move does not consume moves
- finish tile is visited even if reached early

## Move limit

- normal move consumes 1
- mud exit consumes configured cost
- move is rejected if cost exceeds remaining moves
- player loses when moves reach 0 without winning
- player wins if final move reaches finish and moves reach 0 at same time

Important rule:

Win should take priority over loss if the same move both reaches the finish and consumes the final move.

## Mud

- entering mud costs normal current tile cost
- leaving mud costs mud exitMoveCost
- failed move from mud does not consume mud cost

## Floating

- floating tile does not disappear when entered
- floating tile disappears after leaving
- disappeared tile cannot be entered
- invalid move from floating tile does not make it disappear

## Refill

- refill adds moves when entered
- refill is consumed after use
- consumed refill does not add moves again
- refill tile remains walkable

## Teleport

- entering teleport moves player to paired endpoint
- teleport costs no extra moves beyond entering tile
- teleport destination is final position
- teleport chain does not trigger
- teleport destination cannot be wall
- teleport destination cannot be disappeared
- teleport destination cannot be visited if no-backtracking is active

## Timer

- timer decreases while playing
- timer does not decrease while paused
- timer expiration causes loss
- timer expiration after win does not override win

---

# 51. Edge-case priority rules

When multiple outcomes happen during one move, use this priority:

1. Invalid move checks happen before state changes.
2. If invalid, no state changes except feedback/events.
3. Movement cost is paid before tile effects are applied.
4. Refill applies after entering the tile.
5. Teleport applies after entering the tile.
6. Checkpoints are collected after the final position is resolved.
7. Floating tile disappearance applies to the tile the player left.
8. Win condition is checked before loss condition.
9. If a move reaches the finish and uses the last move, win takes priority.
10. If time expires after the level is already won, win remains.

---

# 52. MVP implementation order

Recommended implementation order:

## MVP 1: Pure engine

1. Define core types:
   - Coordinate
   - Direction
   - Tile
   - LevelDefinition
   - NormalizedLevel
   - GameState
   - GameEvent

2. Implement level validation.

3. Implement level normalization.

4. Implement createInitialGameState.

5. Implement processMove with:
   - direction handling
   - wall collision
   - out of bounds
   - finish detection

6. Add move limit rule.

7. Add checkpoint rule.

8. Add no-backtracking rule.

9. Add engine tests.

## MVP 2: Basic visual board

1. Build a simple GameScreen.
2. Load one test level.
3. Render board with simple sprites or colored tiles.
4. Render player.
5. Add keyboard movement.
6. Add HUD.
7. Add win/loss modal.

## MVP 3: Special mechanics

Add mechanics in this order:

1. Mud
2. Refill
3. Floating
4. Teleport

## MVP 4: Content

Create manually authored levels:

- 5 basic maze levels
- 5 no-backtracking levels
- 5 checkpoint levels
- 5 move-limit levels
- 5 mixed mechanic levels

## MVP 5: UI integration

Add:

- landing screen
- tutorial screen
- choose mode screen
- rules screen
- puzzles screen
- difficulty ladder screen
- difficulty screen

All these screens should eventually route into the same reusable GameScreen.

---

# 53. Non-goals for first engine version

Do not implement these in the first engine version:

- PvP
- PvC
- AI solver
- procedural maze generation
- online leaderboards
- backend
- account system
- complex scoring
- level editor
- advanced animations
- sound system
- mobile touch controls
- multiplayer networking

These can be added later after the core engine is stable.

---

# 54. Final architecture rule

The project should always preserve this separation:

Pure TypeScript Engine:
  Owns rules, mechanics, validation, win/loss, state transitions.

React:
  Owns screens, HUD, menus, overlays, modals, routing.

PixiJS:
  Owns visual board rendering, sprites, animations, visual effects.

Zustand/store:
  Owns current session state and connects React input to the engine.

Level JSON:
  Owns content, layout, rules, mechanics, and visual references.

This separation is the key to keeping the project maintainable as the number of levels, rules, screens, and mechanics grows.
