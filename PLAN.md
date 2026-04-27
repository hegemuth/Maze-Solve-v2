# PLAN.md
# Maze Traversal / Solver Puzzle Game Implementation Plan

## 0. Purpose

This document defines the implementation plan for the Maze Traversal / Solver Puzzle Game.

The main project specification is stored in:

GAME_ENGINE_SPEC.md

Codex should read GAME_ENGINE_SPEC.md before implementing any milestone.

This plan breaks the project into safe, testable milestones. Do not attempt to implement the entire game in one pass.

The first priority is the pure TypeScript maze/rules engine. React UI, PixiJS rendering, menu screens, animations, PvP, PvC, and polish come later.

---

# 1. Project goals

The final project is a browser-based educational maze puzzle game using:

- TypeScript
- React
- Vite
- Browser only, no backend initially
- PixiJS or Canvas for the maze board later
- React for menus, overlays, HUD, buttons, and screen routing
- JSON-authored maze levels

The game must support:

- Tile-by-tile arrow-key movement
- Manually authored JSON levels
- Variable maze sizes
- Start, finish, wall, floor, and checkpoint tiles
- Optional rules:
  - no backtracking
  - checkpoint requirement
  - move limit
  - time limit
- Optional special mechanics:
  - mud tiles
  - refill tiles
  - floating tiles
  - teleport tiles

---

# 2. Core architecture rule

Preserve this separation at all times:

Pure TypeScript engine:
- Owns game rules
- Owns movement validation
- Owns win/loss logic
- Owns tile effects
- Owns state transitions

React:
- Owns screens
- Owns HUD
- Owns menus
- Owns overlays and modals
- Owns routing

PixiJS:
- Owns board rendering
- Owns sprites
- Owns animations and visual effects

Zustand/store:
- Owns current game session state
- Calls the pure engine
- Connects UI input to engine actions

Level JSON:
- Owns content, board layout, rules, mechanics, and visual references

The engine must not import React, PixiJS, DOM APIs, CSS, or Zustand.

---

# 3. Milestone overview

## Milestone 1: Pure TypeScript engine

Goal:

Create the full deterministic maze/rules engine without UI or rendering.

Includes:

- Core model types
- Level validation
- Level normalization
- Lookup generation
- Initial game state creation
- Movement processing
- No-backtracking rule
- Checkpoint rule
- Move-limit rule
- Mud tiles
- Refill tiles
- Floating tiles
- Teleport tiles
- Engine tests
- Sample JSON levels

Does not include:

- React screens
- PixiJS rendering
- CSS
- Menus
- Animations
- Sound
- PvP
- PvC
- AI solver

This is the most important milestone.

---

## Milestone 2: Minimal playable GameScreen

Goal:

Create a basic React screen that loads one level and lets the player move around.

Includes:

- GameScreen
- Simple React/CSS grid renderer or minimal placeholder renderer
- Keyboard input
- HUD showing:
  - level title
  - remaining moves
  - checkpoints collected
  - feedback message
  - game status
- Restart button
- Win/loss modal or simple status display

Does not include:

- PixiJS final renderer
- Full menu navigation
- Generated background images
- Advanced animations
- Sound

This milestone proves the engine works in a browser.

---

## Milestone 3: Basic level selection and routing

Goal:

Allow the app to select and load multiple levels.

Includes:

- Landing screen placeholder
- Level selection screen placeholder
- Route or state-based navigation
- Level catalog
- Loading levels by ID
- Restarting and exiting a level

Does not include:

- Final background images
- Advanced UI polish
- Full rule/puzzle/difficulty menu hierarchy

---

## Milestone 4: PixiJS maze renderer

Goal:

Replace or supplement the placeholder board renderer with a proper sprite-based PixiJS renderer.

Includes:

- MazePixiView React wrapper
- Pixi renderer setup and cleanup
- Tile layers
- Player layer
- Special tile overlays
- Consumed/disappeared states
- Basic movement animation
- Board scaling for variable maze sizes

Does not include:

- Complex particles
- Sound
- Full visual polish

---

## Milestone 5: Full menu/UI image workflow

Goal:

Integrate generated screen images as backgrounds and place interactive React buttons on top.

Includes:

- Landing page with image background
- Tutorial entry
- Choose mode screen
- Rules screen
- Puzzles screen
- Difficulty ladder screen
- Difficulty screen
- Button overlay positioning system
- Responsive scaling based on fixed design resolution

Does not include:

- PvP/PvC final gameplay
- Online features

---

## Milestone 6: Tutorial system

Goal:

Create guided tutorials that reuse the real engine and GameScreen.

Includes:

- Tutorial metadata in level JSON
- Tutorial prompt overlay
- Event-triggered tutorial messages
- Optional tile highlights
- Tutorial completion tracking

---

## Milestone 7: Progress, scoring, and polish

Goal:

Add game progression and presentation polish.

Includes:

- localStorage progress
- completed levels
- best moves
- best time
- optional stars
- improved win/loss modal
- improved feedback messages
- settings
- accessibility options

---

## Milestone 8: PvC mode

Goal:

Add player versus computer / algorithm mode.

Includes:

- Solver system
- BFS for basic mazes
- Dijkstra for weighted mud levels
- AI path playback
- Race-style UI
- Comparison between player route and algorithm route

This milestone should only start after the core engine is stable.

---

## Milestone 9: PvP mode

Goal:

Add local player-versus-player mode.

Includes:

- Two-player state
- Two sets of controls
- Separate player positions
- Shared or separate objectives
- Collision rules
- Split HUD or combined HUD

This milestone should only start after single-player and PvC are stable.

---

# 4. Milestone 1 detailed plan

Milestone 1 is the immediate implementation target.

## 4.1 Create model types

Create files under:

src/game/model/

Required model concepts:

- Coordinate
- Direction
- Tile
- TileType
- LevelDefinition
- NormalizedLevel
- RuleConfig
- MechanicsConfig
- GameState
- GameStatus
- GameEvent
- MoveHistoryEntry

The model should support the specification from GAME_ENGINE_SPEC.md.

Acceptance criteria:

- Types are clear and reusable.
- No React/Pixi/Zustand imports.
- TypeScript typecheck passes.

---

## 4.2 Implement coordinate utilities

Create utilities for:

- coordinateToKey
- keyToCoordinate
- addDirectionToCoordinate
- isSameCoordinate
- isInsideBoard

Acceptance criteria:

- Utilities are covered by tests.
- Coordinate keys use the format "x,y".
- Coordinates use zero-based indexing.

---

## 4.3 Implement level validation

Create:

src/game/level/validateLevel.ts

Validation should check:

- board width and height are positive
- tile rows match board height
- each row matches board width
- tile types are valid
- exactly one start tile exists
- at least one finish tile exists
- checkpoint tiles have unique checkpointId values
- mechanic positions are inside the board
- mechanics are not placed on walls
- teleport pairs have exactly two endpoints
- refill amount is positive
- mud exitMoveCost is at least 1
- move limit is positive if enabled
- time limit is positive if enabled

Acceptance criteria:

- Valid levels pass.
- Invalid levels throw useful error messages.
- Tests cover common invalid cases.

---

## 4.4 Implement level normalization

Create:

src/game/level/normalizeLevel.ts
src/game/level/buildLevelLookups.ts

Normalization should produce:

- startPosition
- finish tile set
- checkpoint lookup map
- wall tile set
- mud tile lookup
- floating tile set
- refill tile lookup
- teleport destination lookup
- normalized rule config with defaults

Acceptance criteria:

- Engine uses NormalizedLevel, not raw JSON.
- Lookups allow fast tile checks.
- Tests confirm lookups are built correctly.

---

## 4.5 Implement initial game state

Create:

src/game/engine/createInitialGameState.ts

Initial state should include:

- status = playing or not_started, depending on design
- player at start position
- previousPosition = null
- visitedTiles includes start tile
- disappearedTiles empty
- consumedTiles empty
- collectedCheckpointIds empty
- movesUsed = 0
- remainingMoves = maxMoves if move limit is enabled, otherwise null
- timer initialized based on time limit
- feedback initialized
- latestEvents empty

Acceptance criteria:

- Start tile is visited.
- Move budget initializes correctly.
- Checkpoint list initializes correctly.
- Tests cover initial state.

---

## 4.6 Implement basic movement

Create:

src/game/engine/processMove.ts
src/game/engine/movement/getTargetPosition.ts
src/game/engine/movement/validateMove.ts
src/game/engine/movement/calculateMoveCost.ts

Basic movement should support:

- up/down/left/right
- out-of-bounds rejection
- wall rejection
- disappeared tile rejection
- successful move to floor
- successful move to finish
- player position update
- move history update
- events and feedback

Acceptance criteria:

- Invalid moves do not mutate state.
- Invalid moves do not consume moves.
- Successful moves update player position.
- processMove is deterministic.
- Tests cover valid and invalid movement.

---

## 4.7 Implement win condition

Create:

src/game/engine/winLoss/checkWinCondition.ts

Win condition:

- player is on finish tile
- required checkpoints are collected if checkpoint rule is enabled

Acceptance criteria:

- Finish wins when no checkpoints required.
- Finish does not win when checkpoints are missing.
- Finish wins after all checkpoints are collected.
- Win takes priority over loss when both happen on final move.

---

## 4.8 Implement checkpoint rule

Create:

src/game/engine/rules/checkpointRule.ts

Behavior:

- checkpoints are collected when entered
- checkpoints can be collected in any order
- duplicate collection does not duplicate state
- finish is enterable before checkpoint completion, but level does not complete

Acceptance criteria:

- Checkpoint collection events are emitted.
- Collected checkpoint state updates correctly.
- Tests cover any-order collection.
- Tests cover entering finish early.

---

## 4.9 Implement no-backtracking rule

Create:

src/game/engine/rules/noBacktrackingRule.ts

Behavior:

- player cannot enter any tile in visitedTiles
- start tile is visited at level start
- final teleport destination also must respect visitedTiles
- invalid visited move does not mutate state or consume moves

Acceptance criteria:

- Returning to start is blocked.
- Returning to any previous tile is blocked.
- Invalid visited move emits useful event.
- Tests cover no-backtracking behavior.

---

## 4.10 Implement move limit rule

Create:

src/game/engine/rules/moveLimitRule.ts

Behavior:

- normal moves cost 1
- mud exit moves may cost more
- remainingMoves decreases by move cost
- move rejected if not enough remaining moves
- level lost if remainingMoves reaches 0 without winning
- win takes priority over loss

Acceptance criteria:

- Normal movement consumes 1.
- Not enough moves blocks movement.
- Final move can win even if remaining moves becomes 0.
- Tests cover move limit behavior.

---

## 4.11 Implement mud tiles

Create:

src/game/engine/effects/mudEffect.ts

Behavior:

- mud cost applies when leaving the mud tile
- entering mud costs the previous tile's exit cost
- invalid move from mud does not consume mud cost
- mud exitMoveCost must be used by move cost calculation

Acceptance criteria:

- Entering mud costs normal cost.
- Leaving mud costs configured cost.
- Not enough moves prevents leaving mud.
- Tests cover mud behavior.

---

## 4.12 Implement refill tiles

Create:

src/game/engine/effects/refillEffect.ts

Behavior:

- refill triggers when entered
- refill is one-time use
- consumed refill remains walkable
- refill may increase remaining moves above original max for MVP

Acceptance criteria:

- Refill adds configured amount.
- Refill is marked consumed.
- Re-entering consumed refill does not add moves again.
- Tests cover refill behavior.

---

## 4.13 Implement floating tiles

Create:

src/game/engine/effects/floatingEffect.ts

Behavior:

- floating tile disappears after player successfully leaves it
- entering floating tile does not make it disappear
- invalid move from floating tile does not make it disappear
- disappeared tile cannot be entered

Acceptance criteria:

- Floating tile disappears after successful leave.
- Disappeared tile blocks movement.
- Invalid move does not disappear tile.
- Tests cover floating behavior.

---

## 4.14 Implement teleport tiles

Create:

src/game/engine/effects/teleportEffect.ts

Behavior:

- teleport triggers when entered
- teleport moves player to paired endpoint
- teleport costs no extra moves
- only one teleport activation per move
- no teleport chains
- teleport destination must be valid
- no-backtracking applies to entry and destination
- teleport entry and destination are considered visited after successful teleport

Acceptance criteria:

- Teleport moves player correctly.
- Teleport does not chain.
- Teleport destination validation works.
- Teleport respects no-backtracking.
- Tests cover teleport behavior.

---

## 4.15 Implement timer tick

This can be implemented in Milestone 1 or postponed to Milestone 2 if needed.

Create:

src/game/engine/processTick.ts
src/game/engine/rules/timeLimitRule.ts

Behavior:

- elapsed time increases while playing
- remaining time decreases
- time expiration causes loss
- won/lost/paused games do not continue ticking

Acceptance criteria:

- Timer loss works.
- Timer does not override win.
- Tests cover timer behavior.

---

## 4.16 Add sample levels

Create sample levels under:

src/data/levels/

Minimum sample levels:

- basic-finish.json
- checkpoint-basic.json
- no-backtracking-basic.json
- move-limit-basic.json
- mud-refill-basic.json
- floating-basic.json
- teleport-basic.json
- mixed-demo.json

Acceptance criteria:

- All sample levels pass validation.
- Sample levels cover implemented systems.
- Sample levels are small and easy to debug.

---

## 4.17 Add tests

Use the test framework already present in the project.

If none exists, add a reasonable modern test setup for a Vite + TypeScript project.

Recommended:

- Vitest

Test categories:

- coordinate utilities
- level validation
- level normalization
- initial state
- basic movement
- finish/win
- checkpoints
- no-backtracking
- move limit
- mud
- refill
- floating
- teleport
- timer if implemented

Acceptance criteria:

- Tests pass.
- Tests are deterministic.
- Tests do not require browser rendering.

---

# 5. Milestone 1 done criteria

Milestone 1 is complete when:

- The pure TypeScript engine is implemented.
- No engine file imports React, PixiJS, Zustand, DOM APIs, or CSS.
- All sample levels validate successfully.
- Engine tests pass.
- TypeScript typecheck passes.
- Build passes if available.
- processMove is deterministic.
- Invalid moves do not mutate previous state.
- The engine supports:
  - walls
  - floor
  - start
  - finish
  - checkpoints
  - no-backtracking
  - move limit
  - mud
  - refill
  - floating
  - teleport
- Win/loss and feedback events are produced correctly.

---

# 6. Important implementation notes

## Immutability

Avoid mutating previous GameState directly.

Preferred approach:

- clone sets/maps before modifying them
- return a new GameState object
- keep pure functions deterministic

## Event-driven feedback

The engine should emit events for UI/rendering.

Do not rely only on strings.

Use event types such as:

- move_success
- invalid_wall
- invalid_out_of_bounds
- invalid_disappeared
- invalid_visited
- invalid_not_enough_moves
- checkpoint_collected
- refill_collected
- teleport_used
- floating_tile_disappeared
- finish_reached_incomplete
- level_won
- level_lost
- timer_expired

## Win/loss priority

If a move both completes the level and consumes the final move, the result is win.

Win should take priority over move-limit loss.

## Finish before checkpoints

The player can enter finish before collecting all checkpoints.

The level should not complete until all required checkpoints are collected.

If no-backtracking is active, the finish tile becomes visited when entered early.

This is intentional.

---

# 7. What not to do during Milestone 1

Do not implement:

- PixiJS renderer
- React board
- final menu screens
- generated image background workflow
- sound effects
- animations
- localStorage progress
- scoring/stars
- PvP
- PvC
- AI solver
- procedural generation
- level editor
- mobile touch controls

Keep Milestone 1 focused on the engine.

---

# 8. Recommended Codex workflow

For each milestone:

1. Read GAME_ENGINE_SPEC.md.
2. Read PLAN.md.
3. Implement only the current milestone.
4. Run tests/typecheck/build.
5. Fix failures.
6. Summarize:
   - files changed
   - behavior implemented
   - validation commands run
   - known gaps or follow-up tasks

Do not claim success unless validation commands were actually run successfully.
