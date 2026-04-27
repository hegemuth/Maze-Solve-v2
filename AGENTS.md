# AGENTS.md
# Repository Instructions for AI Coding Agents

## 0. Project summary

This project is a TypeScript + React + Vite browser game.

The game is a tile-based maze traversal / solver puzzle game.

Players move through manually authored JSON mazes using strict tile-by-tile movement.

The game supports configurable rules and mechanics per level:

Rules:
- no backtracking
- checkpoint requirement
- move limit
- time limit

Special mechanics:
- mud tiles
- refill tiles
- floating tiles
- teleport tiles

The project should be implemented in phases.

The first priority is the pure TypeScript game engine.

---

# 1. Read these files first

Before implementing changes, read:

1. GAME_ENGINE_SPEC.md
2. PLAN.md
3. IMPLEMENT.md
4. AGENTS.md

If the task conflicts with the specification, ask for clarification or follow the most specific current user instruction.

---

# 2. Architecture rules

Maintain strict separation between engine, UI, rendering, and state.

## Pure TypeScript engine

Engine files live under:

src/game/engine/
src/game/model/
src/game/level/

The engine owns:

- movement rules
- validation
- tile effects
- move cost calculation
- checkpoint logic
- no-backtracking logic
- move-limit logic
- time-limit logic
- win/loss checks
- event generation
- deterministic state transitions

The engine must not import:

- React
- PixiJS
- Zustand
- DOM APIs
- CSS files
- UI components
- route/navigation code

## React

React owns:

- screens
- menus
- HUD
- overlays
- modals
- buttons
- routing
- user-facing layout

React may consume engine state.

React must not own core gameplay rules.

## PixiJS

PixiJS owns:

- visual board rendering
- sprites
- animation
- visual effects
- board layers

PixiJS must not decide if a move is legal.

PixiJS must not decide if the player won or lost.

## Store

The store connects UI/input to the engine.

Recommended store:

- Zustand

The store may call engine functions.

The store should not duplicate engine rule logic.

## Level JSON

Level JSON owns:

- board layout
- rules enabled for the level
- mechanic placement
- visual references
- tutorial metadata later

Do not hardcode level-specific behavior in the engine.

---

# 3. Coding style

Use TypeScript clearly and strictly.

Prefer:

- small pure functions
- explicit types
- readable names
- deterministic behavior
- testable modules
- simple data structures
- clear validation errors

Avoid:

- large god functions
- hidden mutation
- vague `any` types
- UI logic inside engine files
- rule logic inside React components
- rule logic inside rendering code
- hardcoded assumptions about one specific level

Use `unknown` instead of `any` when handling untrusted JSON, then validate/narrow.

---

# 4. State mutation rules

Be careful with GameState.

Invalid moves must not mutate existing state.

Successful moves should return a new state object.

Clone before modifying:

- Set
- Map
- arrays
- nested objects

Do not directly mutate previous GameState unless the function is explicitly documented as internal and safe.

Preferred pattern:

- receive previous state
- compute next state
- return next state and events

---

# 5. Engine behavior rules

Movement is strict tile-by-tile.

Supported directions:

- up
- down
- left
- right

No diagonal movement.

Coordinate system:

- x = column
- y = row
- top-left is x=0, y=0

Tile key format:

"x,y"

Example:

"3,5"

---

# 6. Locked gameplay decisions

Use these rules unless a later user instruction changes them:

1. Movement is strict tile-by-tile with arrow keys.
2. Checkpoints can be collected in any order.
3. Refill tiles are one-time use.
4. Mud cost applies when leaving the mud tile.
5. Floating tiles disappear after the player leaves them.
6. No-backtracking means the player cannot step on any previously visited tile.
7. The finish tile can be entered before all checkpoints are collected, but the level does not complete until checkpoints are collected.
8. Levels are manually authored JSON files.
9. Maze logic is tile-defined but displayed using sprites.
10. Maze/rules engine is the first priority.

---

# 7. Move pipeline

When implementing movement, follow this order:

1. Receive movement direction.
2. Check game status is playing.
3. Calculate target coordinate.
4. Check target is inside board.
5. Check target is not wall.
6. Check target has not disappeared.
7. Check no-backtracking against target.
8. If target is teleport, validate teleport destination.
9. Calculate move cost based on tile being left.
10. Check move budget if move limit is active.
11. Commit movement to target.
12. Apply target tile effects.
13. If teleport occurred, update final position.
14. Apply checkpoint collection after final position is resolved.
15. If previous tile was floating, mark it disappeared.
16. Mark teleport entry tile as visited if teleport occurred.
17. Mark final player position as visited.
18. Update move history.
19. Check win condition.
20. Check loss condition.
21. Return updated state and events.

If a move is invalid before commit:

- do not move player
- do not consume moves
- do not collect checkpoints
- do not consume refill
- do not disappear floating tile
- do not update move history
- return old state plus invalid move event

---

# 8. Win/loss priority

Win takes priority over loss.

If the player reaches the finish and uses the final available move, the result is won.

Do not mark the game as lost after it has already been won.

Timer expiration must not override a won state.

---

# 9. Events

Engine actions should produce events.

Use machine-readable event types.

Examples:

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

Events should include:

- type
- message
- optional payload

React, PixiJS, sounds, and tutorial systems should react to events later.

---

# 10. Level validation

Invalid level JSON should fail loudly during development.

Validation errors should be useful.

Good:

"Level mixed-demo has no start tile."

Bad:

"Invalid level."

Validate:

- board dimensions
- row lengths
- tile types
- start count
- finish count
- checkpoint IDs
- mechanic positions
- teleport pair shape
- refill amount
- mud exitMoveCost
- move/time limits

---

# 11. Tests

Core engine behavior must be tested.

If the repository already has a test framework, use it.

If not, prefer Vitest for Vite + TypeScript.

Tests should not require:

- React rendering
- PixiJS rendering
- browser DOM
- real keyboard input

Important test groups:

- coordinate utilities
- validation
- normalization
- initial state
- movement
- finish/win
- checkpoints
- no-backtracking
- move limit
- mud
- refill
- floating
- teleport
- timer if implemented

---

# 12. Validation commands

Before finishing work, run the available commands.

Check package.json and run what exists.

Common commands:

npm run typecheck
npm test
npm run test
npm run build
npm run lint

If a command does not exist, say so.

Do not claim a command passed unless it actually passed.

If validation fails, fix the issue before final response unless it is clearly unrelated and pre-existing.

---

# 13. Current implementation priority

Current priority:

Milestone 1: Pure TypeScript game engine.

Do not implement these unless explicitly asked:

- PixiJS renderer
- final UI screens
- menu image background workflow
- PvP
- PvC
- AI solver
- procedural maze generation
- level editor
- sound
- animations
- mobile controls
- backend
- online leaderboard

---

# 14. File organization

Prefer this structure:

src/game/model/
  Types and interfaces

src/game/level/
  Level validation, normalization, lookup building

src/game/engine/
  State creation, movement processing, rule evaluation, effects, win/loss

src/game/utils/
  Shared game utilities like coordinates

src/data/levels/
  JSON level files

src/screens/
  React screen components

src/components/
  Reusable React UI components

src/store/
  Zustand or app state

src/game/rendering/
  PixiJS or Canvas rendering code

---

# 15. Documentation

When adding important behavior, keep code readable and add short comments only where they clarify non-obvious rules.

Good places for comments:

- move pipeline
- teleport edge cases
- no-backtracking interaction with teleport
- win/loss priority
- mud exit cost
- floating tile disappearance timing

Avoid noisy comments that repeat obvious TypeScript.

---

# 16. Final response requirements

When completing a coding task, summarize:

1. What was implemented
2. Files changed
3. Tests added or updated
4. Validation commands run
5. Results of validation
6. Known limitations
7. Suggested next step

Do not include huge code dumps in the final response unless asked.

---

# 17. Honesty requirement

Do not invent successful results.

If you did not run a command, say you did not run it.

If a command failed, report the failure.

If a requested task could not be completed, explain what remains.

If you made an assumption, state it clearly.

---

# 18. Main success principle

The project should remain easy to expand.

A new rule or tile mechanic should be addable without rewriting the entire engine.

A new level should usually require only a new JSON file.

The reusable GameScreen should eventually be able to load any valid level configuration.
