# Workshop Mode — Epic Design Doc
*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review

1. **`circle` shape type generates pocket, not profile** — `ShapeType = 'circle'` with `cutType: CutType` (which can be `'profile' | 'pocket' | 'drill'`) always maps to `OperationType = 'pocket'` in `getOperationType()` (`workshopOperations.ts:119`). There's no way to cut a circle as a profile (outline only). This may be intentional (circles as pockets) but differs from rectangles which also always map to pocket.

2. **`path` with `cutType === 'profile'` maps to `OperationType = 'straight-cut'`** — In `getOperationType()` line 123, a freeform path profile returns `'straight-cut'` rather than a dedicated `'profile-cut'` type. This is semantically misleading — `'straight-cut'` implies table saw, but band saw freeform cuts use the same type.

3. **Slot shape type (`SlotParams`) has no toolpath engine** — `ShapeType = 'slot'` is defined in types but has no case in `generateShapeGcode()`. It would fall through to the stub. The toolbar marks it "coming soon" in assembly mode; in workshop mode, slot functionality is handled via `path` shapes with `closed: false`.

4. **Roundover edge treatment disabled at code level** — `roundoverCuts.ts` is imported but roundover generation has an unreachable `continue` statement before the call (`workshopOperations.ts:~280`). Comment says "disabled for launch (E20-S4)". Feature-flagged off in toolbar too (`ROUNDOVER_ENABLED`).

5. **Dog-bone relief in rectangular pockets uses G2 full circles** — `pocketCuts.ts:~75` generates `G2` arcs with `I={bitRadius} J0` at each corner. This creates a circular relief at each corner but the arc starts and ends at the same point with only I offset — verify this produces correct G-code on all controllers (some interpret same-start-end G2 as a full circle, others as zero movement).

6. **`isNested` flag defined on `Shape` but explicitly NOT used** — Comment in `generateShapeGcode()` says "isNested (geometric detection) is NOT used here — fill-based isIsland is the source of truth." The `isNested` field is dead weight on the type.

7. **Hardcoded `stepOverPercent: 40`** for rectangular pockets in `workshopOperations.ts:~225` and pocket paths. Not configurable per-shape.

8. **Onion skin hardcoded to 0.5mm** for straight cuts and band saw paths. Not user-configurable.

9. **`generateCirclePocketGcode` doesn't exist** — Circle shapes (`type: 'circle'`) reuse the rectangular pocket engine (`generatePocketGcode`) which generates a rectangular clearing pattern. No helical/circular pocket strategy exists for circles. The result is a rectangular zigzag inside what should be a circular boundary.

## Overview

### What Is This Epic?
Workshop Mode is the primary UX paradigm of Routr. Instead of exposing CAM concepts (profile cuts, pockets, drilling cycles), users select familiar woodworking tools — **Table Saw**, **Router**, **Drill Press**, **Planer**, **Band Saw** — and the system maps these to appropriate CNC operations and toolpath strategies automatically.

### Problem Statement
Traditional CAM software requires users to understand G-code operations, tool compensation, and machining strategies. Hobbyist woodworkers think in terms of shop tools. Workshop Mode bridges this gap: pick a tool, draw on the board, get G-code.

### Goals & Non-Goals
**Goals:**
- 1:1 mapping between shop tools and drawing tools
- Each shape on a board becomes one toolpath operation
- Board IS the stock (no separate nesting step in workshop mode)
- Auto-assign CNC tools from library based on operation type

**Non-Goals:**
- Multi-board nesting (that's Assembly Mode, currently feature-flagged off)
- Manual G-code editing
- Custom toolpath strategies per shape

## Context

### Affected Systems
- `src/engine/toolpath/` — all toolpath generators
- `src/components/toolbar/Toolbar.tsx` — tool selection UI
- `src/components/canvas/BoardCanvas.tsx` — shape drawing interaction
- `src/store/useProjectStore.ts` — project state management
- `src/types/index.ts` — core type definitions

### Dependencies
- **js-clipper** (Clipper.js) — boolean polygon operations for freeform pocket clearing (`pocketPathCuts.ts`)
- **uuid** — operation ID generation
- **Tool Library** (`src/store/useToolLibraryStore.ts`, `src/engine/tools/toolLibrary.ts`) — auto-assignment of CNC tools to operations
- **G-code Parser** (`src/engine/toolpath/gcodeParser.ts`) — parses generated G-code back into `ParsedToolpath` for 3D visualization
- **Arc math** (`src/engine/arc.ts`) — arc center/direction calculations for G2/G3 commands

### Dependents
- **3D Toolpath Visualizer / Simulator** — consumes `ParsedToolpath` and `TimelineSegment` types
- **G-code Export** (`operations.ts: exportGcode()`) — stitches all enabled operations with tool changes
- **Edge Treatment system** (E20) — extends workshop operations with chamfer, rabbet, dado
- **SVG Import** (E16) — imports SVG paths as `Shape` objects that flow through the same pipeline

## Design

### Approach

The core pipeline is: **Tool → Shape → Operation → G-code**

```
User selects tool (Toolbar.tsx)
  → User draws on canvas (BoardCanvas.tsx)
    → Shape added to Board.shapes[]
      → generateWorkshopOperations() iterates all shapes
        → Each shape maps to one ToolpathOperation
          → G-code generated by shape-specific engine
            → Parsed back to ParsedToolpath for visualization
```

#### Tool → Shape Mapping (Toolbar)

| Workshop Tool | `activeTool` ID | Creates `ShapeType` | Default `CutType` |
|---|---|---|---|
| Table Saw | `line-cut` | `line-cut` | `profile` |
| Drill Press | `hole` | `hole` | `drill` |
| Planer | `surfacing` | `surfacing` | — |
| Band Saw | `band-saw` | `path` | `profile` |
| Router → Rectangle | `rectangle` | `rectangle` | `pocket` |
| Router → Slot | `router-slot` | `path` (open) | `pocket` |
| Router → Pocket | `router-pocket` | `path` (closed) | `pocket` |
| Router → Edge Treatments | `edge-*` | `EdgeTreatment` on board | — |

#### Shape → Operation Mapping (`workshopOperations.ts`)

| `ShapeType` | `CutType` | `OperationType` | Generator Function | File |
|---|---|---|---|---|
| `line-cut` | — | `straight-cut` | `generateStraightCutGcode` | `straightCut.ts` |
| `hole` | — | `drill` | `generateDrillGcode` | `drillCuts.ts` |
| `surfacing` | — | `surfacing` | `generateSurfacingGcode` | `surfacingCuts.ts` |
| `rectangle` | — | `pocket` | `generatePocketGcode` | `pocketCuts.ts` |
| `circle` | — | `pocket` | `generatePocketGcode` | `pocketCuts.ts` |
| `path` | `profile` | `straight-cut` | `generateBandSawPathGcode` | `pathCuts.ts` |
| `path` (closed) | `pocket` | `pocket` | `generatePocketPathGcode` | `pocketPathCuts.ts` |
| `path` (open) | `pocket` | `pocket` | `generateRouterSlotGcode` | `pathCuts.ts` |
| any | `engrave: true` | `engrave` | `generateBandSawPathGcode` (shallow) | `pathCuts.ts` |

### Data Model

**Core types** (`src/types/index.ts`):

```typescript
type AppMode = 'workshop' | 'assembly';
type ShapeType = 'rectangle' | 'circle' | 'hole' | 'slot' | 'line-cut' | 'surfacing' | 'path';
type CutType = 'profile' | 'pocket' | 'drill';
type OperationType = 'profile-cut' | 'miter-cut' | 'box-joint-cut' | 'straight-cut' | 'drill' | 'pocket' | 'surfacing' | 'engrave';
type BitOffset = 'left' | 'center' | 'right';
```

**Shape** is the central type. Key fields:
- `type: ShapeType` — what geometry
- `cutType: CutType` — how to cut it
- `position: Point2D` — center position on board (screen coordinates, Y-down)
- `depth?: number` — cut depth in mm; `undefined` = through-cut (full board thickness)
- `rotation?: number` — degrees, free rotation
- `scale?: number` — uniform scale multiplier
- `bitOffset?: BitOffset` — tool compensation side
- `engrave?: boolean` — shallow V-bit trace
- `svgImport?: boolean` — imported from SVG
- `svgGroupId?: string` — multi-path SVG group linkage
- `isIsland?: boolean` — SVG island (white fill), skip toolpath
- `params` — shape-specific parameters (union type)

**ToolpathOperation** wraps generated G-code:
- `toolId: string | null` — references tool library entry (null = manual settings)
- `toolSettings: ToolSettings` — resolved speeds/feeds
- `toolType: 'flat end mill' | 'v-bit'`
- `gcode: string | null` — generated G-code
- `parsedToolpath: ParsedToolpath | null` — parsed for 3D viz
- `enabled: boolean` — user can toggle
- `order: number` — execution sequence

**Coordinate system**: Screen space is Y-down, origin top-left. CNC space is Y-up, origin bottom-left. The `flipY()` function in `workshopOperations.ts` handles the transform: `gcodeY = boardHeight - screenY`.

### Key Algorithms / Logic

#### Table Saw — `straightCut.ts`
- `generateStraightCutGcode(options: StraightCutOptions): string`
- Takes two user control points, extends the line to board boundary intersections via `lineBoardIntersections()` (ray-edge intersection against 4 board edges)
- Multi-pass depth stepping with configurable `stepDown`
- Leaves 0.5mm onion skin (hardcoded)
- Supports `BitOffset` (left/right/center) via perpendicular offset calculation (`calculatePerpendicularOffset`)
- Each pass: rapid to safe height → rapid to start → plunge → linear cut to end → retract

#### Drill Press — `drillCuts.ts`
- `generateDrillGcode(options: DrillOptions): string`
- **Two strategies** selected automatically:
  - `holeDiameter <= bitDiameter` → **Peck drilling** (`generatePeckDrillGcode`): straight plunge with peck retract cycles. Uses 1mm clearance plane between pecks.
  - `holeDiameter > bitDiameter` → **Helical bore** (`generateHelicalBoreGcode`): plunge at center, move to perimeter radius `(holeDiameter - bitDiameter) / 2`, full G2 circle, return to center, repeat per depth pass.
- Through-holes add 0.5mm extra depth.

#### Router Rectangular Pocket — `pocketCuts.ts`
- `generatePocketGcode(options: PocketOptions): string`
- Zigzag clearing pattern: alternating left-right passes with `stepOver` Y increment
- Finish pass: traces the pocket boundary rectangle
- Dog-bone corners: full G2 circle (radius = `bitRadius`) at each corner for square internal corners
- Bit radius compensation on all boundaries (inset by `bitRadius`)

#### Router Freeform Pocket — `pocketPathCuts.ts`
- `generatePocketPathGcode(config: PocketPathConfig): string`
- Uses **Clipper.js contour offset** strategy (replaced earlier zigzag approach per commit `e6ff058`)
- `generateOffsetPaths()`: generates concentric inward offset polygons from the boundary using `ClipperOffset` with round joins
- Cuts innermost path first, finishes with outermost (finish pass)
- **Island subtraction**: uses `subtractHoles()` with Clipper boolean difference before offsetting — islands (SVG white fills) are excluded from pocket area
- **V-bit support**: `effectiveDiameter()` calculates cutting width at depth based on V-angle: `2 * depth * tan(halfAngle)`, capped by flute length
- `pointInPolygon()` (ray casting) used to associate islands with their parent pockets by centroid containment

#### Planer — `surfacingCuts.ts`
- `generateSurfacingGcode(options: SurfacingOptions): string`
- Full-board coverage with configurable direction (`'x'` or `'y'`)
- Extends passes beyond board boundary by `bitDiameter/2` margin for clean edges
- Zigzag pattern (alternating direction per row/column)
- Multi-level Z passes based on `passDepth`

#### Band Saw — `pathCuts.ts`
- `generateBandSawPathGcode(config: BandSawPathConfig): string`
- Multi-pass depth cutting along arbitrary waypoint path
- Supports **arc segments** via `arcControlPoint` on `PathSegment` — generates G2/G3 commands using arc center calculation from `src/engine/arc.ts`
- Supports `BitOffset` via `offsetPath()` — perpendicular offset with miter-join corners (line-line intersection)
- 0.5mm onion skin default
- Also used for engrave operations (with `depth` = shape depth, `onionSkin` = 0)

#### Router Slot — `pathCuts.ts`
- `generateRouterSlotGcode(config: RouterSlotConfig): string`
- Same path-following logic as band saw but cuts to specified depth (not through-cut)
- No onion skin, no bit offset

#### Shape Transforms — `workshopOperations.ts: applyShapeTransforms()`
- Handles rotation and scale for all shape types before G-code generation
- SVG imports have segments stored relative to centroid — translates by position before transform
- Uses `transformShapePoints()` for point-based transforms (line-cut, path)
- Scales `width`/`height` for rectangles, `diameter` for circles

#### Tool Auto-Assignment
- `autoAssignTool()` from `src/engine/tools/toolLibrary.ts`
- Maps operation types to tool categories: `'table-saw'`, `'drill'`, `'pocket'`, `'surfacing'`, `'band-saw'`, `'slot'`, `'engrave'`, `'v-carve'`
- Falls back to manual `toolSettings` if no matching tool in library

#### G-code Export — `operations.ts: exportGcode()`
- Stitches all enabled operations sorted by `order`
- Inserts `M6 T{n}` tool change commands when tool changes between operations (compares `toolId`, `toolType`, and `bitDiameter`)
- Strips `M2`/`M5`/`M30` from individual operation G-code
- Adds file header (project name, date, operation list) and footer (`M5`, retract, `G0 X0 Y0`, `M30`)

### API / Interface Changes
Workshop Mode is selected via `AppMode = 'workshop'` (vs `'assembly'`). Assembly Mode is currently feature-flagged off (commit `9c2f28d`). The toolbar renders different tool sets based on `appMode`.

## Edge Cases & Gotchas

1. **Y-axis flip** — Screen coordinates (Y-down) must be flipped for CNC (Y-up). This was a significant bug source (commit `d543099`, multiple edge treatment fixes `39b2dc9`, `5a07977`, `93d7e03`).

2. **SVG imports are special** — SVG paths store segments relative to centroid (0,0). `applyShapeTransforms()` must translate by `position` before rotation/scale. SVG groups (`svgGroupId`) move together. Islands (`isIsland`) are detected by fill color, not geometry.

3. **Circle pockets use rectangular clearing** — No dedicated circular pocket strategy. The zigzag pattern wastes time cutting air outside the circle boundary.

4. **Board = Stock in Workshop Mode** — No separate stock sheet / nesting. The board dimensions ARE the material dimensions. Edge treatments create synthetic `BoardPlacement` and `StockSheet` objects to reuse assembly-mode infrastructure.

5. **`getOperationType()` returns `'straight-cut'` for band saw paths** — Semantically wrong but functionally fine since the type is used for display/grouping, not dispatch.

## Risks

- **No toolpath simulation validation** — G-code is generated and parsed for visualization but not validated against machine limits or collision detection.
- **Clipper.js integer scaling** — `pocketPathCuts.ts` uses `scaleFactor = 1000` for Clipper's integer math. Very small features (< 0.001mm) or very large boards could hit precision issues.
- **Single-tool-per-operation assumption** — Each shape = one operation = one tool. No support for roughing + finishing passes with different tools.

## Stories (retroactive — what was built)

| Commit | Description |
|---|---|
| `9c2f28d` | Rebrand to Routr + feature-flag Assembly Mode off (Workshop Mode becomes default) |
| `d543099` | Y-axis flip for CNC coordinate system |
| `19f4642` | E16-S0: Shape rotation and uniform scale support |
| `f1d8c77` | E16-S1: SVG parser |
| `a20c45f` | E16-S2: Engrave tool modal + SVG import UI |
| `338433e` | E16-S3: Engrave CutType — profile trace with V-bit |
| `ac5c9ca` | E16-S4a: SVG pocket auto-detection + canvas hatched fill |
| `e6ff058` | E16-S4b: Replace zigzag pocketing with Clipper.js contour offset |
| `1fe1bc8` | E16-S4c: Island subtraction in pocket toolpaths |
| `7f859e4` | E20-S1: Input blur-clamp audit |
| `afb3ed3` | E20-S2: Edge coordinate mapping — centralized mapper |
| `1f109ee` | E20-S3: Chamfer V-bit angled edge depth |
| `f8d1bbd` | E20-S4: Disable roundover for launch |
| `46e3586` | E20-S5: QA cleanup — roundover toolbar, edge mapping, dado canvas |

## Decisions Log

1. **Board IS the stock** — In Workshop Mode, there's no nesting step. Board dimensions equal stock dimensions. This simplifies the mental model dramatically.
2. **One shape = one operation** — No merging of shapes into combined toolpaths. Simple, predictable, but potentially inefficient for many small operations.
3. **Clipper.js over custom zigzag** — Freeform pocket clearing switched from naive zigzag (E16-S4a) to Clipper.js contour offset (E16-S4b) for better surface finish and correct handling of complex shapes.
4. **Islands by fill color, not geometry** — `isIsland` (SVG white fill) is the source of truth for subtraction, not `isNested` (geometric containment test). This is explicitly documented in code comments.
5. **Onion skin = 0.5mm** — Through-cuts leave a thin skin to prevent parts from shifting. Not yet user-configurable.
6. **Assembly Mode feature-flagged off** — Workshop Mode is the shipping product. Assembly Mode (multi-board nesting, joints, profile cuts from stock) exists but is hidden.

## Known Issues / Tech Debt

1. **Circle shapes use rectangular pocket engine** — No dedicated circular pocket strategy (helical, spiral, or circular zigzag). Wastes machining time.
2. **`isNested` field is dead code** — Defined on `Shape` type but explicitly unused. Should be removed.
3. **Hardcoded parameters** — `stepOverPercent: 40`, `onionSkin: 0.5mm`, `clearancePlane: 1.0mm` in drill — should be user-configurable or at least constants.
4. **`SlotParams` type has no engine** — The `slot` ShapeType and `SlotParams` interface exist but have no drawing tool or toolpath generator in Workshop Mode. Slot functionality is handled by `path` shapes instead.
5. **Roundover edge treatment blocked** — Code exists but is disabled with unreachable `continue` statement. Needs 3D ball-end-mill toolpath strategy.
6. **No arc support in rectangular pocket** — Only freeform paths support arc segments. Rectangular pockets with `cornerRadius` don't generate arcs.
7. **`OperationType` naming inconsistency** — Band saw freeform paths return `'straight-cut'` type. Should have a dedicated type or be renamed.
8. **Edge treatment operations use `'pocket'` as OperationType** — All edge treatments (chamfer, rabbet, dado) are typed as `'pocket'` regardless of actual cutting strategy.
