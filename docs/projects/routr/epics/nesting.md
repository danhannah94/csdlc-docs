# Nesting — Epic Design Doc
*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review
- **Tab support incomplete on polygon paths**: `tracePolygonPass()` has tab distribution comments but tabs are NOT actually implemented for polygon (box-joint) profile cuts — only rectangle cuts get tabs. The function traces the polygon at full depth without any Z-raise for tabs.
- **Cleanup heuristic may be aggressive**: `GuillotinePacker.cleanupFreeRects()` removes any free rect that overlaps a used rect, but after the guillotine split this could incorrectly discard valid free space in edge cases where the split rects partially overlap the padded used rect.
- **No multi-sheet auto-creation limit**: `autoNestBoards` will keep creating new default-sized sheets indefinitely until all boards are placed. No cap on sheet count.
- **Spring pass always runs**: The nesting G-code always adds a spring pass (duplicate of final depth) regardless of whether it's useful for the material/tool combo.

## Overview

### What Is This Epic?
Nesting is the system that arranges designed boards onto stock sheets of raw material to minimize waste before CNC cutting. It includes an auto-nesting algorithm (guillotine bin packing), manual drag-and-drop placement, multi-sheet support, placement validation, and nesting-specific G-code generation with layer-by-layer cutting strategy and tab support.

### Problem Statement
Users design multiple boards in Routr and need to efficiently lay them out on physical stock material before generating toolpaths. Without nesting, each board would need to be cut individually from its own piece of stock, wasting material. The system needs to handle varying board sizes, stock sheet dimensions, tool clearances, and edge margins.

### Goals & Non-Goals
**Goals:**
- Automatic bin-packing of boards onto stock sheets (minimize waste)
- Manual placement with drag-and-drop for user control
- Real-time validation (overlap, spacing, boundary, edge margin)
- Multi-sheet support with auto-creation of additional sheets
- Layer-by-layer G-code generation across all boards per sheet
- Tab support to hold boards in place during cutting
- Box joint polygon profile cuts with dog-bone relief

**Non-Goals (not implemented):**
- Arbitrary rotation angles (only 0°/90° for auto-nest; 0/90/180/270 for manual)
- Irregular/non-rectangular board nesting
- Material grain direction optimization
- Cost optimization across stock sizes

## Context

### Affected Systems
- **Board designer** — boards flow into nesting as read-only inputs
- **G-code generator** — nesting has its own G-code pipeline (`nestingGcode.ts`) separate from per-board generation
- **Project state** — `stockSheets`, `nestingSettings`, `tabSettings`, and `placements` stored in project

### Dependencies
- `src/types/index.ts` — `Board`, `StockSheet`, `NestingSettings`, `BoardPlacement`, `TabSettings`, `ToolSettings`
- `src/engine/gcode/generator.ts` — `GcodeBuilder`, `POST_PROCESSORS`
- `src/engine/toolpath/boardOutline.ts` — `generateBoardOutline`, `computeDogBoneReliefs` (for box-joint polygon paths)
- `src/engine/nesting/projection.ts` — 2D projection and dimension helpers
- `uuid` — placement and sheet ID generation

### Dependents
- UI components: `NestingPanel`, `NestingCanvas`, `BoardTray`, `PlacedBoard` (~2600 LOC total)

## Design

### Approach

The nesting workflow is:

1. **Stock sheet setup** — User creates stock sheets with width/height/thickness. Defaults: 610×1219×19mm (24"×48"×¾").
2. **Board placement** — Boards start "unplaced" (`isPlaced: false`) in a tray. They can be manually dragged onto sheets or auto-nested.
3. **Auto-nest** — Guillotine bin packing places all boards onto existing sheets, creating new default-sized sheets as needed. Boards sorted largest-first (decreasing area).
4. **Validation** — Continuous validation checks overlaps, spacing (vs bit diameter and board spacing), boundary violations, and edge margin violations. Stale flags track when upstream boards/sheets change.
5. **G-code generation** — Layer-by-layer strategy: all boards on a sheet are cut at depth N before any board goes to depth N+1, plus a final spring pass. Tabs hold boards in place on the last pass. Box-joint boards get polygon profile cuts with dog-bone relief instead of simple rectangles.

### Data Model

```typescript
interface StockSheet {
  id: string;
  name: string;
  width: number;       // mm
  height: number;      // mm
  thickness: number;   // mm
  material?: MaterialType;
}

interface NestingSettings {
  defaultStockWidth: number;    // 610mm (24")
  defaultStockHeight: number;   // 1219mm (48")
  defaultStockThickness: number;// 19mm (¾")
  boardSpacing: number;         // mm, default 2× bit diameter
  edgeMargin: number;           // mm, default 25.4 (1")
  edgeMarginEnabled: boolean;   // false for vacuum tables
}

interface TabSettings {
  enabled: boolean;
  width: number;    // mm, default 6
  height: number;   // mm, default 2
  spacing: number;  // mm, default 150
  minTabs: number;  // default 4
}

interface BoardPlacement {
  id: string;
  boardId: string;
  stockSheetId: string;
  x: number;          // mm, top-left of bounding rect
  y: number;          // mm
  rotation: number;   // 0, 90, 180, 270
  isPlaced: boolean;  // false = in tray
}
```

### Key Algorithms / Logic

**Guillotine Bin Packing (`autoNest.ts`)**
- `GuillotinePacker` class manages free rectangles within a single sheet
- Usable area = sheet dimensions minus 2× edge margin (when enabled)
- Board spacing added to each board's required footprint
- Best Short Side Fit heuristic: tries both 0° and 90° orientations, picks the free rect that leaves the smallest short-side remainder
- After placement, the free rect is split via guillotine cuts: vertical split (full height) on the right, horizontal split (used width only) below
- Boards sorted by area descending before packing
- If a board doesn't fit on any existing sheet, a new default-sized sheet is auto-created
- Returns detailed stats: per-sheet utilization, board counts, used/usable areas

**Validation (`validation.ts`, `nestingValidator.ts`)**
- Four validation checks per placement:
  1. **Boundary** (error): board extends beyond stock sheet edges
  2. **Edge margin** (warning): board in margin zone but within sheet
  3. **Overlap** (error): pairwise AABB overlap between placed boards
  4. **Spacing** (error if < bit diameter, warning if < board spacing): pairwise distance check
- `nestingValidator.ts` adds change detection:
  - Board resize → flag affected placements as stale
  - Board delete → remove orphaned placements
  - Stock resize → flag affected placements as stale
  - Stock delete → move placements back to unplaced
- `validateAllPlacements()` aggregates per-sheet results into a summary with overall state (valid/warning/error)

**Projection (`projection.ts`)**
- `projectBoardTo2D()`: extracts width×height from board dimensions (ignores joints/thickness for MVP)
- `getEffectiveDimensions()`: swaps width/height for 90°/270° rotations
- `calculateBoundingBox()`: computes min/max extents across all placements
- `doesBoardFitInStock()`: simple bounds check

**Nesting G-code (`nestingGcode.ts`) — differs from per-board G-code:**
- **Layer-by-layer across all boards**: Each depth pass cuts ALL boards on the sheet before going deeper. Per-board G-code would cut one board fully before starting the next.
- **Nearest-neighbor cut ordering**: Boards are ordered to minimize rapid travel distance between cuts.
- **Spring pass**: After all depth passes, a final pass at full depth for clean finish.
- **Tab support on rectangle cuts**: On the last pass, Z raises to `-(stockThickness - tabHeight)` at evenly-distributed tab locations along the perimeter. Tabs offset by ⅛ perimeter to avoid corners. `minTabs` enforced.
- **Polygon profile for box joints**: Boards with box joints use `buildPolygonToolPath()` which generates an offset polygon (miter offset at tool radius) with dog-bone relief detours at inside corners. Coordinate transform handles placement rotation (0/90/180/270).
- **Tool radius offset**: Rectangle cuts offset the outline by tool radius outward (cutting outside the board). Polygon paths use miter offset clamped to 2× tool radius.

### API / Interface Changes
The nesting system is purely client-side. No backend API — all state lives in the project model. The UI components (`NestingPanel.tsx` at 1425 LOC, `NestingCanvas.tsx` at 735 LOC) manage the canvas rendering, drag-and-drop, and settings panels.

## Edge Cases & Gotchas
- **Floating point tab matching**: Tab locations are stored as perimeter distances and matched via `Math.round(distance * 1000)` key lookup. The edge tracing also does a brute-force 0.1mm step scan as fallback.
- **Miter offset clamping**: At acute angles, miter offset is clamped to 2× tool radius to avoid extreme spikes in the polygon path.
- **Edge margin disabled**: When `edgeMarginEnabled` is false, the full sheet area is usable (for vacuum tables or screwed-down stock).
- **Degenerate boards**: If a board is too large for the default stock sheet with current margin/spacing settings, auto-nest returns an error with the specific dimensions.
- **Rotation semantics differ**: Auto-nest only uses 0° and 90°. Manual placement supports 0/90/180/270. The projection and G-code systems handle all four.

## Risks
- The guillotine packing is a greedy heuristic — not optimal. Complex layouts with many boards of varied sizes may have poor utilization compared to more advanced algorithms.
- Tab support is missing from polygon (box-joint) profile cuts, meaning box-joint boards have no holding tabs during the final pass.

## Stories (retroactive)
Based on git history:
- **E9-S2b**: Nesting tab UX fixes — units, drag, viewport
- **E9-S2c**: Nesting input precision and unplace board button
- **E9-S3**: Auto-nesting with guillotine bin packing
- **E9-S4**: Nesting validation and sync
- **E15-S1**: Nesting-aware profile cut generation with tabs

## Decisions Log
- **Guillotine over MaxRects**: Guillotine bin packing chosen (simpler, good enough for typical workshop stock sheets with <20 boards).
- **Layer-by-layer cutting**: All boards cut at each depth before going deeper — reduces thermal stress and ensures even material removal across the sheet.
- **Best Short Side Fit**: Chosen over Best Area Fit or Best Long Side Fit as the rectangle selection heuristic.
- **Board spacing = 2× bit diameter default**: Ensures enough clearance for the tool to pass between boards without collision.
- **Tab offset by ⅛ perimeter**: Avoids placing tabs at corners where they'd be structurally weak.

## Known Issues / Tech Debt
- **No tabs on polygon paths**: `tracePolygonPass()` does not implement tab logic — box-joint boards cut through completely on every pass.
- **No undo for auto-nest**: Auto-nest replaces all placements; no way to revert to previous manual layout.
- **Projection ignores joints**: `projectBoardTo2D()` uses raw board width×height, not accounting for joint fingers that extend beyond nominal dimensions.
- **No sheet material tracking in G-code**: Stock sheet `material` field exists in the type but isn't used for feeds/speeds in nesting G-code.
- **Validation is O(n²)**: Pairwise overlap/spacing checks. Fine for typical counts (<50 boards) but would need spatial indexing for large layouts.
