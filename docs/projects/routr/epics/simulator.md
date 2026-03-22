# Simulator — Epic Design Doc
*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review
- **Heightmap computation is synchronous and blocks the UI thread.** `precomputeKeyframes()` runs in a `setTimeout(..., 50)` but is not offloaded to a Web Worker. Large stock sheets with many segments will freeze the UI during computation.
- **Heightmap disabled in Workshop Mode** due to performance cost of surfacing toolpaths (`SimulatorTab.tsx` explicitly skips it). Material removal visualization only works in Assembly Mode.
- **Heightmap resolution is fixed at 1mm/cell** (`CELL_SIZE = 1`). Fine for large profile cuts, but may miss detail on small engravings or V-bit work.
- **G-code parser does not round-trip perfectly.** The parser re-interprets generated G-code for visualization, but only handles G0/G1/G2/G3 motion commands. M-codes (spindle, coolant), G4 dwells, and tool changes are silently ignored — this is correct for visualization but means the parser cannot validate the full G-code program.
- **Keyframe interval is hardcoded to 5.** No user control over simulation fidelity vs. memory trade-off.
- **Arc linearization uses fixed 10° segments** (π/18 radians), producing minimum 8 segments per arc regardless of radius. Small arcs get over-segmented; large arcs may appear slightly faceted.

## Overview

### What Is This Epic?
The Simulator epic provides a 3D toolpath visualization system with animated playback, a heightmap-based material removal simulation, and interactive controls. It lets users watch their G-code execute virtually on a 3D stock sheet before sending it to a CNC machine.

### Problem Statement
Users generating CNC toolpaths need visual confirmation that cuts will be correct before committing material. Without simulation, errors in nesting, tool selection, or operation ordering are only discovered at the machine — wasting stock and time.

### Goals & Non-Goals
**Goals:**
- Parse generated G-code back into 3D-renderable segments
- Animate a cutting head along the toolpath with playback controls (play/pause, speed, scrub, skip operations)
- Show progressive material removal via heightmap displacement
- Display real-time HUD with feed rate, Z-depth, spindle speed, and move type
- Support both flat end mill and V-bit tool geometry visualization
- Heat-map color toolpath lines by feed rate

**Non-Goals:**
- Physics-based simulation (chip load, deflection, vibration)
- Audio simulation
- Collision detection between tool and clamps/fixtures
- Multi-stock-sheet simultaneous simulation

## Context

### Affected Systems
- **3D Scene** (`components/shared/Scene3D`) — shared R3F canvas used by both Preview3D and Simulator
- **Project Store** (`store/useProjectStore`) — holds `SimulationState`, `timeline`, and exposes `simTick`, `simPlay`, `simPause`, `simTogglePlayPause`, `simSetSpeed`, `simScrubTo`, `simReset`
- **Toolpath Engine** (`engine/toolpath/`) — G-code generators produce the G-code that the parser re-interprets
- **Nesting / Workshop tabs** — upstream: boards must be placed or shapes defined before operations can be generated

### Dependencies
- **@react-three/fiber** + **@react-three/drei** — 3D rendering (Canvas, useFrame, Line, OrbitControls)
- **three** — geometry, materials, vector math
- **G-code generators** — profile cuts, miter cuts, box joints, straight cuts, surfacing, engrave, drill, pocket
- **Tool library** — provides bit diameter, tool type, speeds/feeds per operation

### Dependents
- **Export** — uses the same operations and G-code that the simulator visualizes
- **SimulatorPanel** — sidebar UI reads simulation state for progress bars and operation info

## Design

### Approach

```
G-code string (per operation)
  → parseGcode() → ParsedToolpath { segments[], bounds, stats }
  → buildTimeline() flattens all enabled operations into TimelineSegment[]
  → SimulatorTab renders:
      ├─ StockSheetMesh (transparent box at origin)
      ├─ NestedBoardMeshes (Assembly Mode only)
      ├─ ToolpathLines (heat-mapped, progressive or static)
      ├─ HeightmapSurface (displaced plane with vertex colors)
      ├─ SimulationController → CuttingHead (animated tool)
      ├─ PlaybackControls (scrub bar, speed, operation skip)
      └─ SimulatorHUD (live feed rate, Z-depth, RPM, move type)
```

The simulation loop is driven by `useFrame()` in `SimulationController`. Each frame:
1. `simTick(delta)` advances `currentSegmentIndex` and `currentProgress` based on `playbackSpeed`
2. `interpolatePosition()` computes the 3D scene position from the timeline
3. `CuttingHead` renders at that position with the correct tool geometry
4. `HeightmapSurface` looks up the nearest precomputed keyframe and displaces vertices
5. `ToolpathLines` in progressive mode draws segments up to `currentSegmentIndex`
6. `SimulatorHUD` reads current segment data (feed rate, Z, RPM, move type)

### Data Model

**`SimulationState`** (in project store):
```typescript
interface SimulationState {
  isPlaying: boolean;
  playbackSpeed: number;         // 0.5, 1, 2, 5, 10
  currentSegmentIndex: number;   // global index across all ops
  currentProgress: number;       // 0-1 within current segment
  totalSegments: number;
}
```

**`TimelineSegment`** (flattened from all enabled operations):
```typescript
interface TimelineSegment {
  operationId: string;
  operationIndex: number;
  operationName: string;
  segmentIndex: number;          // index within operation's parsed toolpath
  segment: ToolpathSegment;
  globalIndex: number;
  toolType: 'flat end mill' | 'v-bit';
  bitDiameter: number;           // mm
  spindleSpeed: number;          // RPM
}
```

**`ToolpathSegment`**:
```typescript
interface ToolpathSegment {
  type: 'rapid' | 'linear';
  from: { x: number; y: number; z: number };
  to: { x: number; y: number; z: number };
  feedRate: number;              // mm/min, 0 for rapids
}
```

**`ParsedToolpath`**:
```typescript
interface ParsedToolpath {
  id: string;
  name: string;
  segments: ToolpathSegment[];
  bounds: { min: Point3D; max: Point3D };
  stats: {
    totalCutLength: number;
    totalRapidLength: number;
    estimatedTime: number;       // seconds (rapids assume 3000 mm/min)
    maxFeedRate: number;
    minFeedRate: number;
  };
}
```

**`Heightmap`** / **`HeightmapKeyframes`** (engine-level, not in store):
```typescript
interface Heightmap {
  cols: number;                  // ceil(stockWidth / 1mm)
  rows: number;                  // ceil(stockHeight / 1mm)
  data: Float32Array;            // depth values (0 = uncut, negative = cut)
  stockWidth: number;
  stockHeight: number;
  stockThickness: number;
}

interface HeightmapKeyframes {
  frames: Float32Array[];        // snapshots every N segments
  segmentIndices: number[];      // which segment each frame corresponds to
  cols: number; rows: number;
  stockWidth: number; stockHeight: number; stockThickness: number;
}
```

### Key Algorithms / Logic

#### G-code Parser (`gcodeParser.ts`)

Stateful line-by-line parser that maintains position, feed rate, units mode (G20/G21), and absolute/incremental mode (G90/G91).

- **G0/G1**: Creates `ToolpathSegment` with `rapid` or `linear` type. Skips zero-movement commands.
- **G2/G3 (arcs)**: Linearizes circular arcs into line segments. Uses I/J offsets (always relative to current position) to find arc center. Full circles detected when start == end. Arcs subdivided into `max(8, ceil(|sweep| / 10°))` segments. Helical bores supported (Z changes interpolated linearly across arc segments).
- **Unit conversion**: Inch coordinates multiplied by 25.4 at the point-of-use (converted coordinates stored in segments).
- **Comments**: Lines starting with `;` or `(` are stripped. Inline comments after `;` are ignored.
- **Stats**: Computes total cut length, rapid length, estimated time, and feed rate range. Rapids assumed at 3000 mm/min for time estimation.

#### Heightmap Material Removal (`heightmap.ts`)

2D grid at 1mm resolution representing stock surface depth.

- **`stampFlatEndMill()`**: Circular footprint — iterates bounding box of bit radius, tests `dx² + dy² ≤ r²`, writes depth (only deeper, never raises).
- **`stampVBit()`**: V-shaped groove — depth at each cell varies with distance from center: `cellDepth = tipDepth + dist / tan(halfAngle)`. Only carves where `cellDepth < 0`.
- **`applySegment()`**: Walks along a toolpath segment at 0.5mm steps (`STAMP_STEP`), stamping the bit footprint at each position. Skips segments entirely above surface (both Z ≥ 0).
- **`precomputeKeyframes()`**: Applies all timeline segments sequentially, saving a `Float32Array` snapshot every `keyframeInterval` segments (default 5) plus the final state. Reports progress via callback.
- **`getFrameAtSegment()`**: Binary search for nearest precomputed frame at or before the requested segment index.

#### Timeline Building (`timeline.ts`)

- **`buildTimeline()`**: Filters to enabled + visible operations, sorts by `order`, flattens all segments into a single array with global indices and per-segment tool metadata (type, diameter, spindle speed).
- **`interpolatePosition()`**: Lerps between segment `from` and `to` using `progress` (0-1), then converts from CNC coordinates to scene coordinates: `x*S - halfWidth*S`, `z*S` (Z becomes Y/up), `halfHeight*S - y*S`.
- **`getCurrentOperationInfo()` / `getOperationProgress()`**: Utility functions for HUD and progress bars.

#### Coordinate Conversion

CNC space → Scene space mapping (used consistently across all simulator components):
- CNC X → Scene X (centered: `x * S - (stockWidth/2) * S`)
- CNC Z (depth) → Scene Y (up axis: `z * S`)
- CNC Y → Scene Z (flipped: `(stockHeight/2) * S - y * S`)

Scale factor `S = 0.01` (1mm = 0.01 scene units).

#### HeightmapSurface Rendering (`HeightmapSurface.tsx`)

- Creates a `PlaneGeometry` with `(cols-1) × (rows-1)` subdivisions, rotated to XZ plane.
- On each frame change: updates vertex Y positions from heightmap depth values, and vertex colors using a depth-based gradient (tan wood → lighter exposed wood → darker shadow at depth, normalized to ~20mm max).
- Triggers `computeVertexNormals()` after displacement for correct lighting.

#### Cutting Head Visualization (`CuttingHead.tsx`)

Two tool models:
- **Flat end mill**: Cylinder (bit) + cone collet + transparent spindle housing
- **V-bit**: Cone tip (inverted, 45° half-angle for 90° V-bit) + wider cutting body cylinder + narrow shank + collet + spindle

Red sphere at tip marks the exact cutting point. Tool geometry scales with `bitDiameter`.

#### Feed Rate Heat Map (`ToolpathLines.tsx`)

Three-stop color gradient based on normalized feed rate:
- Blue (`#0066ff`) → slow/cutting
- Yellow (`#ffff00`) → medium  
- Red (`#ff3333`) → fast/rapid

Rapids assumed at 3000 mm/min. Uses `@react-three/drei` `<Line>` with `vertexColors`.

### API / Interface Changes

The simulator adds these actions to the project store:
- `simPlay()`, `simPause()`, `simTogglePlayPause()`
- `simSetSpeed(speed: number)`
- `simScrubTo(segmentIndex: number)`
- `simReset()`
- `simTick(delta: number)` — called by `useFrame` each render frame

Computed/derived:
- `timeline: TimelineSegment[]` — built from operations + visibility state
- `simulation: SimulationState` — playback state

## Edge Cases & Gotchas

1. **Zero-movement segments skipped**: Parser filters `from === to` segments, preventing division-by-zero in interpolation.
2. **Gimbal lock on ViewCube Top/Bottom**: Tiny Z offset (`0.001`) added to camera position to avoid `lookAt` singularity.
3. **Full-circle arcs**: When G2/G3 endpoint equals start, sweep is forced to ±2π instead of 0.
4. **Workshop Mode heightmap skip**: Explicitly disabled because surfacing operations produce too many segments for synchronous precomputation.
5. **Progressive vs static toolpath rendering**: Two completely separate rendering paths — progressive mode builds a single combined line; static mode renders per-operation lines. Switching between them (entering/exiting simulation) causes a full re-render.
6. **Heightmap only carves deeper**: `stampFlatEndMill` and `stampVBit` use `if (depth < data[idx])` — multiple passes at same depth are idempotent; shallower passes don't undo deeper cuts.

## Risks

- **Memory**: Each heightmap keyframe is a `Float32Array(cols * rows)`. A 600×400mm stock = 240,000 cells × 4 bytes = ~1MB per frame. At keyframeInterval=5 with 10,000 segments, that's ~2,000 frames = ~2GB. Large projects could exhaust browser memory.
- **Main thread blocking**: Heightmap precomputation is synchronous. A Web Worker migration is the obvious fix but hasn't been done.
- **Parser limitations**: Only G0/G1/G2/G3 are parsed as motion. If future generators emit G28 (home), G53 (machine coords), or canned cycles (G81-G89), the simulator will silently skip them.

## Stories (retroactive)

| Commit | Story |
|--------|-------|
| `3c6e535` | E15-S3a: Simulation playback with animated cutting head |
| `b7c5980` | E15-S3b: Material removal heightmap visualization |
| `4ba0513` | Next/Previous operation buttons in playback controls |
| `b4ba0f9` | E15-S1: Speeds and feeds from bit spec and material |
| `d95c428` | Box joint toolpath generation and simulation |
| `173e685` | Drag-to-reorder operations in Simulator |
| `f0ad44a` | Skip heightmap precomputation in Workshop Mode |
| `28590a2` | Workshop Mode simulator scaffold with stub engines |
| `1d14353` | Add spindle speed (RPM) to simulator HUD |
| `26c51ac` | Add spindleSpeed to simulator timeline builder |

## Decisions Log

1. **Heightmap over CSG**: Chose a 2D heightmap grid over constructive solid geometry for material removal. Heightmaps are O(n) per segment stamp vs. O(n³) for CSG boolean operations. Trade-off: can't represent undercuts or side-wall detail — only top-down depth.
2. **G-code round-trip**: Rather than storing intermediate toolpath data, each operation generates G-code and then re-parses it for visualization. This ensures the simulator shows exactly what the machine will execute, at the cost of information loss (operation metadata, tool type must be carried separately in the timeline).
3. **Precomputed keyframes**: Rather than computing heightmap per-frame during playback, all keyframes are precomputed upfront. Trades memory for smooth playback. The keyframeInterval of 5 is a balance — every segment would be too much memory, every 50 would look jumpy.
4. **Fixed 1mm cell size**: Simple and fast. Matches the precision of most hobby CNC machines. Finer resolution (0.5mm or 0.25mm) would 4-16× the memory and computation cost.
5. **Workshop Mode heightmap skip**: Surfacing operations generate thousands of closely-spaced parallel passes. Precomputing heightmap keyframes for these was taking multiple seconds and freezing the UI. Rather than implement a Web Worker, heightmap was simply disabled for Workshop Mode.

## Known Issues / Tech Debt

1. **No Web Worker for heightmap** — synchronous computation blocks UI. Should be migrated to a worker with `postMessage`-based progress reporting.
2. **Keyframe memory unbounded** — no cap on total memory used by precomputed frames. Large jobs could crash the tab.
3. **Scrub bar resolution** — scrub input is integer segment indices. For operations with few large segments, scrubbing is coarse. Sub-segment scrubbing (using `currentProgress`) is not exposed in the scrub bar.
4. **No arc rendering in toolpath lines** — arcs are linearized in the parser, so toolpath lines show faceted approximations. The segments are fine enough (10° max) that this is barely visible.
5. **HeightmapSurface vertex update is eager** — every segment index change triggers a full vertex buffer rewrite even if the nearest keyframe hasn't changed (the `useMemo` on `frameData` handles this correctly, but the downstream `useEffect` runs on every `frameData` reference change).
6. **Feed rate heat map doesn't account for actual rapid speed** — rapids are hardcoded to 3000 mm/min for color mapping, regardless of machine capabilities.
7. **`SimulatorPanel.tsx` is 400+ lines** — mixes tool library UI, operation list, export controls, simulation progress, and heat map legend. Could benefit from decomposition.
8. **No test coverage for UI components** — `timeline.ts` and `timeline.test.ts` exist, but no tests for `SimulatorTab`, `PlaybackControls`, `HeightmapSurface`, etc.
