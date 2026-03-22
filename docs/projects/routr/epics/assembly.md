# Assembly & Joints — Epic Design Doc

*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review

| Item | Status | Notes |
|------|--------|-------|
| Assembly feature flag | **No dedicated flag** | Assembly mode is hidden via `AppMode` — workshop mode hides the "Assembly" tab (`setAppMode` in store). No `featureFlags.ts` entry for assembly; the only flag there is `ROUNDOVER_ENABLED`. |
| `AssemblyPanel.tsx` | **Dead component** | Full form-based link management panel in `src/components/panels/AssemblyPanel.tsx`. Superseded by `AssemblyCanvas.tsx` (visual edge-clicking). Still imported in tests (`AssemblyPanel.jointSelector.test.tsx`) but hidden from users. |
| Dovetail joint type | **Type-only stub** | `JointType = 'dovetail'` and `DovetailParams` defined in `types/index.ts`. No geometry generator, no toolpath, no 3D rendering. |
| Rabbet joint type | **Partial** | `JointType = 'rabbet'` defined. Dato/rabbet share a UI selector in `AssemblyPanel`. Constraint solver uses `jointDepth` for dado/rabbet. No dedicated geometry mod or toolpath generator — only edge treatments handle rabbet. |
| Dado joint type | **Partial** | Same as rabbet — `DadoParams` defined, UI exists in AssemblyPanel, solver respects depth. No `SubtractModification` or toolpath gen specific to dado joints. |
| Constraint solver conflicts | **Unused path** | `ConflictInfo[]` is returned by solver but the detection code was replaced by closed-loop validation. `conflicts` is always `[]` in practice. |
| Box joint 3D corner artifacts | Fixed via `_getCornerSlotDepth` in `BoardMeshShared.tsx` (commit `5bb8053`). |

## Overview

### What Is This Epic?

Assembly mode lets users connect multiple boards at specific edges with joints (box, miter, butt, etc.), automatically generating matching male/female joint geometry on both boards, solving 3D positions from link topology, and previewing the assembled furniture in 3D.

### Problem Statement

CNC workshop software typically treats boards in isolation. Users designing furniture need to understand how boards connect — which edges mate, what joint type to use, and whether the assembly is geometrically valid before cutting stock.

### Goals & Non-Goals

**Goals:**
- Visual edge-clicking UX for creating board links
- Automatic male/female joint generation from a single link definition
- Constraint-based 3D positioning (BFS from root board)
- Over-constrained and unreachable board detection
- Box joint and miter joint geometry in 3D preview
- Box joint toolpath generation (G-code with dog-bone relief)

**Non-Goals (current):**
- Dovetail joint geometry/toolpath (type defined, not implemented)
- Dado/rabbet joint-specific geometry mods (only via edge treatments)
- Physics simulation or collision detection
- Assembly instructions / exploded views

## Context

### Affected Systems

| System | How |
|--------|-----|
| `src/types/index.ts` | Joint, JointType, BoardLink, JointParams, Board.joints, Project.links |
| `src/store/useProjectStore.ts` | Link CRUD, solver integration, `autoPositionBoards()`, `appMode` |
| `src/engine/assembly/` | Constraint solver, edge validator, auto-link inference |
| `src/engine/joints/` | Joint geometry modification system (box, miter) |
| `src/engine/toolpath/boxJointCuts.ts` | G-code generation for box joint pockets |
| `src/components/AssemblyCanvas.tsx` | Visual link creation UI |
| `src/components/panels/AssemblyPanel.tsx` | Form-based link management (dead) |
| `src/components/panels/JointPropertiesPanel.tsx` | Per-board joint property editor |
| `src/components/preview3d/BoardMeshShared.tsx` | 3D mesh generation with joint geometry |

### Dependencies

- **Three.js** — constraint solver uses Three.js Matrix4/Vector3/Euler for 3D math
- **@react-three/fiber + drei** — 3D preview rendering
- **Zustand** — state management (useProjectStore)

### Dependents

- **Nesting pipeline** — boards with joints have modified outlines for stock layout
- **Toolpath generation** — box joint cuts run before profile cuts
- **3D Preview** — reads `board.joints` and `project.links` for geometry and positioning

## Design

### Approach

The assembly pipeline flows:

1. **Edge-clicking UX** (`AssemblyCanvas.tsx`) → User clicks edge A on board 1, then edge B on board 2
2. **Validation** (`edgeValidator.ts`) → Checks: no self-links, no edge reuse, edge length mismatch warnings, thickness mismatch warnings
3. **BoardLink creation** → Store adds link with `jointType: 'butt'` default, user upgrades via Configure Joint panel
4. **Joint sync** → Store's `addLink`/`updateLink` calls `_syncJointsFromLinks()` which creates matching `Joint` objects on both boards (board A = male, board B = female)
5. **Constraint solving** (`constraintSolver.ts`) → BFS from root board places all linked boards in 3D space
6. **3D Preview** (`BoardMeshShared.tsx`) → Reads `board.joints` and applies `GeometryModification` (vertex mods for miters, subtract mods for box joints)
7. **Toolpath generation** (`boxJointCuts.ts`) → Generates G-code for box joint pockets with dog-bone corner relief

### Data Model

**`JointType`** (union): `'butt' | 'box' | 'dovetail' | 'dado' | 'rabbet' | 'miter'`

**`JointParams`** (union):
- `BoxJointParams` — `fingerWidth`, `fingerCount`, `depth` (all mm)
- `DovetailParams` — `pinWidth`, `tailWidth`, `angle`, `count` (stub, unused)
- `DadoParams` — `width`, `depth`, `offsetFromEdge`
- `MiterParams` — `angle` (degrees, default 45)
- `ButtParams` — empty `{}`

**`Joint`** (per-board): `{ id, type, edge, params }` — lives on `Board.joints[]`. Created/synced automatically from links.

**`BoardLink`** (project-level): `{ id, boardAId, boardAEdge, boardBId, boardBEdge, jointType, jointParams, offset }` — the source of truth. Board A = male, Board B = female. `offset` is mm slide along the mating edge.

**`GeometryModification`** (interface):
- `VertexModification` — mutates the 8-corner vertex array (used by miter bevels)
- `SubtractModification` — rectangular regions to remove from edge (used by box joints)

### Key Algorithms / Logic

#### Constraint Solver (`constraintSolver.ts`)

**Algorithm:** BFS graph traversal from a root board.

1. Build adjacency graph from `BoardLink[]`
2. Place root board at origin (flat on XZ plane, Y = thickness/2)
3. BFS: for each link, compute neighbor's world transform via `computeLinkedTransform()`
4. **Closed-loop detection:** If a board is reached via a second path, validate by measuring edge proximity (gap tolerance: 40mm). Reported as `ClosedLoopInfo` with `edgeGap` and `valid` flag.
5. Report `unreachable` boards (not connected to root's graph component)

**`computeLinkedTransform()`** — The core placement function:
- Gets edge geometry (center, normal, axis, length) for both boards
- Computes contact point at board A's edge surface, offset by `jointDepth` (box joints: material thickness, miters: thickness, butt: 0)
- Builds rotation from basis vectors: board B's edge normal → toward A, board B's up → perpendicular to fold, board B's axis → aligned with hinge
- Applies alignment offset along hinge axis

**Joint depth by type:**
- `box`: `params.depth` (typically material thickness)
- `miter`: material thickness
- `dado`/`rabbet`: `params.depth`
- `butt`: 0 (sits on surface)

#### Edge Validator (`edgeValidator.ts`)

Returns `ValidationMessage[]` (error/warning):
- **Error:** Self-link (board to itself)
- **Error:** Edge already used by another link (one edge, one link rule)
- **Warning:** Edge length mismatch >20%
- **Warning:** Thickness mismatch between boards

Supports edit mode via `currentLinkId` parameter (excludes self from duplicate checks).

#### Auto-Link Inference (`autoLinkInference.ts`)

Finds unlinked edges on positioned boards that are within `tolerance` (default 0.5mm) of each other in world space. Returns `AutoLinkCandidate[]` with deduplication keys. Validates candidates against edge validator before suggesting.

#### Box Joint Generation (`boxJoint.ts`)

- Calculates finger count: `round(edgeLength / fingerWidth)`, minimum 3, forced odd for symmetry
- Male board (A) gets even-indexed cuts, female board (B) gets odd-indexed cuts
- Returns `FingerCut[]` with position (x,y), width, height in board-local coords
- `boxJointGeometryMod()` wraps into `SubtractModification` for 3D rendering
- `recommendedFingerWidth()`: returns `thickness * 0.75` for thick stock, `thickness` for thin

#### Miter Joint Generation (`miterJoint.ts`)

- `miterGeometryMod()` returns a `VertexModification` that moves top-face vertices inward by `thickness * S`
- Creates a triangular bevel along the full edge length
- Both boards in a miter get identical bevels (symmetric joint)

#### Box Joint Toolpath (`boxJointCuts.ts`)

- Generates G-code for rectangular pockets at each finger/slot position
- Transforms board-local cuts to stock sheet coordinates (handles 0°/90°/180°/270° rotation)
- Spiral-outward pocket clearing strategy
- Dog-bone corner relief at all 4 corners of each pocket (plunge cuts)
- Nearest-neighbor optimization for cut ordering
- Runs **before** profile cuts (board still held by stock)

#### 3D Rendering (`BoardMeshShared.tsx`)

- Reads `board.joints[]` and builds `GeometryModification[]` per board
- Box joints: `SubtractModification` → builds edge side faces split into finger/slot strips with interior faces (floor + walls)
- Miters: `VertexModification` → moves vertices to create bevel
- Handles corner slot depth for proper corner geometry (`_getCornerSlotDepth`)
- Male/female role determined by `(joint as any)._role` (set during link sync)

### API / Interface Changes

Store actions added for assembly:
- `addLink(link)` / `removeLink(id)` / `updateLink(id, updates)` / `updateLinkParams(id, params)` / `clearAllLinks()`
- `autoPositionBoards()` — runs constraint solver, updates board `position3D`/`rotation3D`
- `setAssemblyRootBoard(boardId)` — sets BFS root
- `setAppMode('assembly' | 'workshop')` — workshop mode hides Assembly/3D Preview/Nesting tabs

## Edge Cases & Gotchas

1. **Dead `AssemblyPanel.tsx`**: Full-featured form-based panel with link creation, joint type selector, parameter editing, solver status banners. Superseded by `AssemblyCanvas.tsx` for link creation, but its joint configuration UI is still the only place to change joint types on existing links. Has dedicated test file (`AssemblyPanel.jointSelector.test.tsx`).

2. **Over-constrained detection**: The solver's `ConflictInfo` path is defined but never triggered. Closed-loop links (both boards already placed) are validated by edge proximity instead. The 40mm tolerance is generous to handle thickness offsets in box assemblies.

3. **Unreachable boards**: Reported but only as data — the UI shows a yellow banner in `AssemblyPanel` but that panel is dead. The canvas doesn't visualize unreachable boards differently.

4. **One edge, one link rule**: Enforced at the store level — `addLink()` calls `validateLink()` and rejects if errors exist. Tested in `linkValidation.test.ts`.

5. **Joint `_role` and `_linkId`**: Private properties stamped on `Joint` objects during link sync. Used by 3D renderer and toolpath generator to determine male vs female. Not part of the type definition — accessed via `(joint as any)._role`.

6. **Workshop mode hides assembly**: `setAppMode('workshop')` hides tabs `['Assembly', '3D Preview', 'Nesting']`. This is the de facto feature flag (commit `602bc0d`).

## Risks

- **Dovetail/dado/rabbet stubs** could confuse users if joint type dropdown exposes them (dado shows in dropdown via AssemblyPanel)
- **Closed-loop 40mm tolerance** is empirically tuned — may produce false positives on large assemblies
- **No automated test for `_syncJointsFromLinks()`** — the store sync logic is only tested indirectly through integration tests

## Stories (retroactive)

| Story | Description | Status |
|-------|-------------|--------|
| E12-S1 | BoardLink data model + store CRUD | ✅ Shipped |
| E12-S2 | Edge validator (self-link, edge reuse, length/thickness warnings) | ✅ Shipped |
| E12-S3 | Visual edge-clicking canvas (AssemblyCanvas) | ✅ Shipped |
| E12-S4 | Constraint-based 3D positioning engine | ✅ Shipped |
| E12-S5 | Box joint geometry + toolpath generation | ✅ Shipped |
| E12-S6 | Miter joint geometry (3D bevel) | ✅ Shipped |
| E12-S7 | Auto-link inference (proximity-based suggestions) | ✅ Shipped |
| E12-S8 | Cube assembly integration test (5-board box) | ✅ Shipped |
| E12-S9 | Closed-loop validation (edge proximity) | ✅ Shipped |
| E12-Sx | Dovetail joint geometry | ❌ Type-only stub |
| E12-Sx | Dado/rabbet joint geometry | ❌ Params + UI only |

## Decisions Log

| Decision | Rationale |
|----------|-----------|
| BFS (not full constraint solver) | Simple, deterministic, handles tree + single-loop topologies. Full solver (e.g., iterative relaxation) not needed for furniture-scale assemblies. |
| Male/female via link order | Board A = male (protruding fingers), Board B = female (slots). Consistent, no ambiguity. |
| Odd finger count forced | Ensures symmetric box joints — both ends look the same. Implemented in `generateBoxJointCuts()`. |
| Closed-loop validation replaces conflicts | Original design detected "conflicts" when a board was reached via 2 paths. Replaced with edge proximity check — if edges are close enough, the loop is valid. More useful and less noisy. |
| AssemblyCanvas over AssemblyPanel | Visual edge-clicking is more intuitive than dropdown-based form. Panel kept for joint configuration but link creation moved to canvas. |
| Joint sync from links | `Board.joints[]` is derived from `Project.links[]`. Single source of truth is the link; joints are synced automatically. Prevents divergence. |

## Known Issues / Tech Debt

1. **Feature-flagged off via AppMode**: Assembly is hidden in workshop mode (the default/launch mode). No `featureFlags.ts` entry — controlled by `appMode` in store + tab visibility logic.

2. **Dead AssemblyPanel.tsx**: ~400 lines of form-based UI. Has test coverage. Should either be deleted or repurposed as a secondary interface. Currently creates links with `jointType: 'butt'` default — same as canvas.

3. **`_role` / `_linkId` private properties**: Stamped on joints as untyped properties. Should be added to the `Joint` type definition or handled via a lookup map.

4. **Dovetail type exposed in types but unimplemented**: `DovetailParams` has fields (`pinWidth`, `tailWidth`, `angle`, `count`) but no generator. Would need geometry mod + toolpath + 3D rendering.

5. **Dado/rabbet partially wired**: UI in AssemblyPanel allows selecting dado/rabbet, solver respects depth, but no `SubtractModification` generator exists. Edge treatments handle rabbet independently.

6. **No miter toolpath generation**: Miter bevels render in 3D but there's no G-code generator for miter cuts (would need angled bit or tilted table).

7. **Auto-link inference uses `board.position3D`/`rotation3D` directly**: These are solver outputs. If the solver hasn't run, auto-link returns nothing. Could be confusing for new assemblies.

8. **`handleResetLayout` reloads the page**: `AssemblyCanvas.tsx` line — `window.location.reload()` with a TODO comment.
