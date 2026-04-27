# IMPLEMENT.md
# Codex Implementation Instructions

## 0. Role

You are acting as a senior TypeScript/React game engineer.

You are implementing a browser-based maze traversal puzzle game.

Before making changes, read:

1. GAME_ENGINE_SPEC.md
2. PLAN.md
3. AGENTS.md

The current implementation priority is the pure TypeScript game engine.

Do not jump ahead to UI, PixiJS, menu screens, animations, PvP, PvC, or visual polish unless explicitly instructed.

---

# 1. Current task

Implement Milestone 1 from PLAN.md:

Pure TypeScript maze/rules engine.

The engine must support:

- JSON-authored levels
- level validation
- level normalization
- initial game state creation
- tile-by-tile movement
- out-of-bounds validation
- wall validation
- disappeared tile validation
- finish detection
- checkpoint collection
- no-backtracking rule
- move-limit rule
- mud tiles
- refill tiles
- floating tiles
- teleport tiles
- feedback events
- tests

---

# 2. Hard constraints

The engine must be pure TypeScript.

Files inside `src/game/engine`, `src/game/model`, and `src/game/level` must not import:

- React
- PixiJS
- Zustand
- DOM APIs
- CSS files
- browser routing
- UI components

The engine must not directly:

- render sprites
- create DOM elements
- play sounds
- read keyboard input
- use React hooks
- navigate routes
- access localStorage

The engine only receives data and returns new state.

---

# 3. Expected folder structure

Create or adapt the following structure:

src/game/
  model/
    Coordinate.ts
    Direction.ts
    Tile.ts
    LevelDefinition.ts
    NormalizedLevel.ts
    RuleConfig.ts
    TileEffectConfig.ts
    GameState.ts
    GameEvent.ts

  level/
    validateLevel.ts
    normalizeLevel.ts
    buildLevelLookups.ts

  engine/
    createInitialGameState.ts
    processMove.ts
    processTick.ts
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

  utils/
    coordinates.ts

src/data/
  levels/
    basic-finish.json
    checkpoint-basic.json
    no-backtracking-basic.json
    move-limit-basic.json
    mud-refill-basic.json
    floating-basic.json
    teleport-basic.json
    mixed-demo.json

Tests can be placed in the existing test structure. If no structure exists, create one that fits the project.

---

# 4. Engine design requirements

## 4.1 Deterministic processMove

The main function should conceptually work like this:

processMove(normalizedLevel, gameState, direction) -> updated GameState

or:

processMove(normalizedLevel, gameState, direction) -> { state, events }

Either return shape is acceptable if it is consistent and well typed.

The function must be deterministic.

Same level + same state + same direction must produce the same result.

---

## 4.2 Do not mutate previous state

Invalid moves must not mutate state.

Successful moves should return a new state object.

Be especially careful with:

- Set
- Map
- arrays
- nested objects

Clone before modification.

---

## 4.3 Coordinate system

Use zero-based coordinates.

x = column
y = row

Top-left tile is:

{ x: 0, y: 0 }

Use tile keys internally:

"x,y"

Examples:

"0,0"
"3,5"

Coordinate utility functions should exist for:

- coordinateToKey
- keyToCoordinate
- isSameCoordinate
- isInsideBoard
- getTargetPosition

---

# 5. Movement pipeline

Implement the move pipeline according to this order:

1. Receive movement direction.
2. Check that game status is playing.
3. Calculate target coordinate from current position and direction.
4. Check target is inside the board.
5. Check target is not a wall.
6. Check target has not disappeared.
7. Check no-backtracking rule against target tile.
8. If target is teleport, check teleport destination validity.
9. Calculate move cost based on the tile the player is leaving.
10. Check move budget if move limit is active.
11. Commit movement to target tile.
12. Apply target tile effects:
    - checkpoint collection
    - refill collection
    - teleport
13. If teleport occurred, update final player position.
14. Apply final-position checkpoint collection if teleport lands on checkpoint.
15. If previous tile was floating, mark it as disappeared.
16. Mark teleport entry tile as visited if teleport occurred.
17. Mark final player position as visited.
18. Update move history.
19. Check finish condition.
20. Check loss conditions.
21. Return updated state and events.

If validation fails at any point before commit:

- return old state
- add invalid move event
- do not consume moves
- do not move player
- do not collect checkpoints
- do not consume refill tiles
- do not disappear floating tiles
- do not update move history

---

# 6. Rule behavior

## 6.1 No backtracking

If enabled:

- player cannot enter any previously visited tile
- start tile is visited at level start
- final teleport destination must also not be visited
- teleport entry tile and destination tile become visited after successful teleport

Invalid no-backtracking move event:

invalid_visited

Message example:

"You cannot step on a visited tile."

---

## 6.2 Checkpoints

If checkpoint requirement is enabled:

- all checkpoints must be collected before finish completes the level
- checkpoints may be collected in any order
- checkpoint collected when player enters checkpoint tile
- duplicate checkpoint collection should not duplicate state

If player reaches finish before collecting all checkpoints:

- player remains on finish
- status remains playing
- emit finish_reached_incomplete
- do not win

Message example:

"Exit reached, but not all checkpoints are collected."

---

## 6.3 Move limit

If enabled:

- remainingMoves starts at maxMoves
- each move consumes exit cost
- normal exit cost is 1
- mud exit cost may be higher
- movement is rejected if cost > remainingMoves
- if remainingMoves reaches 0 and player has not won, status becomes lost

Important:

If a move reaches finish and uses the last remaining move, status should be won, not lost.

Win takes priority over loss.

---

## 6.4 Time limit

If implemented in Milestone 1:

- processTick should update elapsed/remaining time
- timer only runs while status is playing
- time expiration causes loss
- time expiration must not override an already won level

If time limit is too much for the first pass, implement model support and leave a clear TODO, but do not fake completed behavior.

---

# 7. Special tile behavior

## 7.1 Mud

Mud cost applies when leaving a mud tile.

Example:

normal floor -> mud:
cost = 1

mud with exitMoveCost 3 -> floor:
cost = 3

Mud should be handled in move cost calculation.

---

## 7.2 Refill

Refill triggers when entered.

Behavior:

- add configured amount to remainingMoves
- mark tile as consumed
- consumed refill does not trigger again
- consumed refill remains walkable
- refill can overfill beyond original maxMoves in MVP

Event:

refill_collected

---

## 7.3 Floating

Floating tile disappears after player successfully leaves it.

Behavior:

- entering floating tile does not disappear it
- invalid move from floating tile does not disappear it
- after successful move away, previous floating tile is added to disappearedTiles
- disappeared tile cannot be entered again

Event:

floating_tile_disappeared

---

## 7.4 Teleport

Teleport triggers when entered.

Behavior:

- teleport pair has exactly two endpoints
- entering one endpoint moves player to the other endpoint
- teleport costs no extra moves
- only one teleport activation per move
- no teleport chains
- destination must be inside board
- destination must not be wall
- destination must not be disappeared
- destination must not be visited if no-backtracking is active
- teleport entry and destination become visited after successful teleport

Event:

teleport_used

Payload should include:

- pairId
- from
- to

---

# 8. Level JSON requirements

Use the object-based tile style.

Example tile rows:

[
  [
    { "type": "wall" },
    { "type": "wall" },
    { "type": "wall" }
  ],
  [
    { "type": "wall" },
    { "type": "start" },
    { "type": "finish" }
  ]
]

Supported base tile types:

- wall
- floor
- start
- finish
- checkpoint

Checkpoint tile example:

{ "type": "checkpoint", "checkpointId": "A" }

Recommended rules shape:

"rules": {
  "noBacktracking": {
    "enabled": true
  },
  "checkpointsRequired": {
    "enabled": true
  },
  "moveLimit": {
    "enabled": true,
    "maxMoves": 30
  },
  "timeLimit": {
    "enabled": false,
    "seconds": null
  }
}

Recommended mechanics shape:

"mechanics": {
  "mudTiles": [
    {
      "position": { "x": 3, "y": 2 },
      "exitMoveCost": 3
    }
  ],
  "floatingTiles": [
    {
      "position": { "x": 4, "y": 4 }
    }
  ],
  "refillTiles": [
    {
      "position": { "x": 5, "y": 6 },
      "amount": 5,
      "oneTime": true
    }
  ],
  "teleportPairs": [
    {
      "pairId": "A",
      "endpoints": [
        { "x": 2, "y": 3 },
        { "x": 8, "y": 3 }
      ]
    }
  ]
}

---

# 9. Event requirements

The engine should return machine-readable events.

Use event types such as:

- move_success
- move_invalid
- invalid_out_of_bounds
- invalid_wall
- invalid_disappeared
- invalid_visited
- invalid_not_enough_moves
- checkpoint_collected
- refill_collected
- teleport_used
- floating_tile_disappeared
- finish_reached
- finish_reached_incomplete
- level_won
- level_lost
- timer_expired

Each event should contain:

- type
- message
- optional payload

Events are used later by:

- React HUD
- PixiJS renderer
- sound system
- tutorial system

Do not make the renderer infer game logic from visual state.

---

# 10. Sample levels to create

Create small, easy-to-debug levels.

Required sample levels:

1. basic-finish.json
   - start
   - floor
   - finish
   - no optional rules

2. checkpoint-basic.json
   - at least two checkpoints
   - checkpoints required
   - finish enterable early

3. no-backtracking-basic.json
   - noBacktracking enabled
   - route where returning to start would be invalid

4. move-limit-basic.json
   - moveLimit enabled
   - small maxMoves

5. mud-refill-basic.json
   - moveLimit enabled
   - mud tile with exitMoveCost > 1
   - refill tile

6. floating-basic.json
   - floating tile that disappears after leaving

7. teleport-basic.json
   - teleport pair
   - finish reachable via teleport

8. mixed-demo.json
   - combination of:
     - checkpoints
     - move limit
     - no-backtracking
     - mud
     - refill
     - floating
     - teleport

All sample levels must validate successfully.

---

# 11. Test requirements

Add tests for the engine.

Use the existing test framework if one exists.

If the project has no test framework, add Vitest.

Required test categories:

## Coordinate utilities

- coordinateToKey
- keyToCoordinate
- isInsideBoard
- direction target calculation

## Level validation

- valid level passes
- missing start fails
- duplicate start fails
- missing finish fails
- invalid dimensions fail
- invalid checkpoint fails
- invalid mechanic position fails
- invalid teleport pair fails

## Level normalization

- start position found
- finish position found
- checkpoint lookup built
- wall lookup built
- mechanic lookups built
- teleport destination lookup built

## Initial state

- player starts at start tile
- start tile is visited
- checkpoints empty
- consumed/disappeared sets empty
- remainingMoves initialized correctly

## Basic movement

- move to floor succeeds
- wall blocks movement
- out of bounds blocks movement
- invalid move does not consume moves
- invalid move does not mutate previous state

## Finish

- finish wins when no checkpoints required
- finish does not win when checkpoints missing
- finish wins after all checkpoints collected
- win priority over move-limit loss

## Checkpoints

- checkpoint collected when entered
- checkpoints can be collected in any order
- duplicate collection does not duplicate

## No backtracking

- start tile visited at start
- cannot step on visited tile
- invalid visited move does not consume moves

## Move limit

- normal move costs 1
- not enough moves blocks movement
- reaching 0 without winning loses
- reaching finish with final move wins

## Mud

- entering mud costs previous tile cost
- leaving mud costs exitMoveCost
- failed move from mud does not consume cost

## Refill

- refill adds moves
- refill consumed after use
- consumed refill does not add moves again
- refill tile remains walkable

## Floating

- floating does not disappear on enter
- floating disappears after leaving
- disappeared tile blocks movement
- invalid move from floating does not disappear it

## Teleport

- entering teleport moves to paired endpoint
- teleport costs no extra moves
- teleport does not chain
- teleport destination validation works
- teleport respects no-backtracking

## Timer

If implemented:

- timer decreases while playing
- timer does not decrease while paused
- timer expiration causes loss
- timer does not override win

---

# 12. Validation commands

Before finishing, run the available validation commands.

Try these, depending on what exists in package.json:

npm run typecheck
npm test
npm run test
npm run build
npm run lint

If a script does not exist, say so clearly.

Do not claim a command passed unless it actually passed.

If tests fail, fix them before finishing.

If build fails because of unrelated existing issues, report them clearly and separate them from your changes.

---

# 13. Completion response format

When done, respond with:

1. Summary of implemented behavior
2. Files created/changed
3. Tests added
4. Validation commands run and results
5. Known limitations or TODOs
6. Recommended next milestone

Example:

Implemented:
- Pure TypeScript engine model types
- Level validation and normalization
- processMove with wall/out-of-bounds/finish/checkpoint logic
- no-backtracking and move-limit rules
- mud/refill/floating/teleport mechanics

Validation:
- npm run typecheck: passed
- npm test: passed
- npm run build: passed

Known TODO:
- timeLimit processTick not implemented yet

Next:
- Milestone 2: minimal playable GameScreen

---

# 14. Do not do these things

Do not:

- implement the entire game at once
- add PixiJS during Milestone 1
- add final menu screens during Milestone 1
- hardcode level-specific behavior
- put game rules inside React components
- put game rules inside Pixi renderer
- mutate previous GameState directly
- silently ignore invalid level JSON
- fake successful validation
- skip tests for core rules
- claim commands passed without running them

---

# 15. Main success definition

Milestone 1 is successful when the project has a reliable pure TypeScript engine that can:

1. Load and validate a level JSON.
2. Normalize the level into lookup maps.
3. Create an initial game state.
4. Process arrow-key movement commands.
5. Enforce walls, no-backtracking, move limit, checkpoints, and special tile mechanics.
6. Produce deterministic state updates and events.
7. Pass automated tests without requiring React or PixiJS.
