# SVG Import & Engrave — Epic Design Doc
*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review
- **Text elements unsupported** — parser warns but cannot convert `<text>` to paths; user must flatten in their SVG editor first.
- **`<use>`/`<defs>`/`<symbol>` unsupported** — referenced elements are silently skipped with a warning. Complex SVGs from design tools often rely on these.
- **Island detection is color-only** — only exact white values (`white`, `#fff`, `#ffffff`, `rgb(255,255,255)`) are recognized as islands. Near-white or semi-transparent fills are not detected.
- **No CSS style parsing** — fill/stroke are read from element attributes only, not from `<style>` blocks or inline `style=""` attributes.
- **Clipper.js integer precision** — scale factor of 1000 means sub-micron precision (~0.001mm), but very large SVGs could hit integer overflow.
- **`pocketCuts.ts` is the legacy rectangular pocket** — still uses zigzag clearing. Freeform SVG pockets use the newer `pocketPathCuts.ts` with Clipper.js contour offset.

## Overview

### What Is This Epic?
A complete SVG import pipeline for the Routr CNC workshop planning app. Users import `.svg` files containing vector artwork, which are automatically parsed, classified (pocket / engrave / island), and converted into CNC toolpaths using contour-offset pocketing with island subtraction.

### Problem Statement
CNC users want to engrave logos, text (as outlines), and decorative patterns onto wood. This requires importing standard SVG vector files and generating appropriate toolpaths — pockets for filled areas, profile traces for open/stroke-only paths, and island preservation for white-filled regions.

### Goals & Non-Goals
**Goals:**
- Parse standard SVG path commands (M, L, H, V, C, S, Q, T, A, Z) and shape elements (rect, circle, ellipse, polygon, polyline, line)
- Flatten bezier curves and arcs to polylines for CNC compatibility
- Auto-detect pocket vs engrave vs island based on SVG fill/stroke attributes
- Generate contour-offset pocket toolpaths with Clipper.js boolean operations for island subtraction
- Support V-bit depth-dependent effective diameter
- Group multi-path SVGs for unified drag/scale on canvas

**Non-Goals:**
- Text rendering (planned, currently "Coming Soon" in UI)
- QR code generation (planned, currently "Coming Soon" in UI)
- CSS style block parsing
- SVG filter, mask, or clipPath support
- Raster image embedding

## Context

### Affected Systems
- **SVG Engine** (`src/engine/svg/`) — parser + import logic
- **Toolpath Engine** (`src/engine/toolpath/pocketPathCuts.ts`) — contour offset pocketing
- **Engrave UI** (`src/components/engrave/EngraveModal.tsx`) — file picker modal
- **Type System** (`src/types/index.ts`) — Shape flags for SVG-specific behavior
- **Canvas** — SVG shapes rendered with special handling (grouped drag, hatched fill for pockets)

### Dependencies
- **js-clipper** (ClipperLib) — boolean polygon operations and polygon offsetting
- **DOMParser** (browser built-in) — SVG XML parsing

### Dependents
- **Workshop Operations** (`workshopOperations.ts`) — reads `engrave` flag to select operation type `'engrave'`
- **Canvas rendering** — reads `svgImport`, `svgGroupId` for grouped selection/movement
- **G-code export** — pocket and profile toolpaths feed into final G-code generation

## Design

### Approach

Full pipeline: **SVG file → parser → parsed paths → import/classification → shapes → toolpath generation → G-code**

1. **File Selection** — `EngraveModal.tsx` presents a file picker (`.svg` only). FileReader reads the file as text.
2. **Parsing** (`svgParser.ts`) — `parseSVG()` uses DOMParser to parse XML, walks the DOM tree recursively, handles transforms (matrix, translate, scale, rotate, skew), converts each element to `Point2D[]` polylines. ViewBox and unit conversion (mm, cm, in, pt, px) normalize coordinates to mm.
3. **Nesting Detection** (`svgParser.ts`) — `detectNesting()` checks every path pair using bounding-box containment + point-in-polygon ray casting. Nested paths get `isNested: true`.
4. **Import & Classification** (`svgImport.ts`) — `importSVGToShapes()` normalizes each path's segments relative to its centroid, calculates board position preserving relative layout, scales to fit board, and classifies each path:
   - **Closed path, dark/default fill** → `cutType: 'pocket'`
   - **Closed path, white fill** → `isIsland: true` (no toolpath)
   - **Open path or `fill="none"`** → `cutType: 'profile'`, `engrave: true`
5. **Shape Addition** — EngraveModal adds shapes to the board. Pocket shapes default to half board thickness depth. Engrave shapes default to 1mm depth.
6. **Toolpath Generation** (`pocketPathCuts.ts`) — `generatePocketPathGcode()` creates concentric inward offset paths using Clipper.js `ClipperOffset`. Island holes are subtracted via `subtractHoles()` (Clipper boolean difference) before offsetting. Cutting order is innermost-first, outermost-last (finish pass).

### Data Model

Shape flags on `Shape` type (`src/types/index.ts`):

| Flag | Type | Description |
|------|------|-------------|
| `svgImport` | `boolean` | `true` for all SVG-imported shapes. Enables grouped drag, disables waypoint handles. |
| `svgGroupId` | `string` | Shared ID (`svg-{timestamp}-{random}`) linking all shapes from one SVG import. Shapes with same ID move together. |
| `isNested` | `boolean` | `true` if path is geometrically contained within another closed path (bounding box + point-in-polygon test). |
| `isIsland` | `boolean` | `true` if path has white fill — represents material to leave uncut. No toolpath generated. |
| `engrave` | `boolean` | `true` for engrave trace paths (open paths or `fill="none"` closed paths). Uses profile cut at shallow depth with V-bit. |

Operation type `'engrave'` added to the `OperationType` union.

### Key Algorithms / Logic

#### SVG Path Command Parsing
Full SVG path `d` attribute support via `parseDAttribute()`:

| Command | Params | Description |
|---------|--------|-------------|
| M/m | x,y | Move to (starts new subpath; subsequent coords become implicit L) |
| L/l | x,y | Line to |
| H/h | x | Horizontal line |
| V/v | y | Vertical line |
| C/c | x1,y1,x2,y2,x,y | Cubic bezier |
| S/s | x2,y2,x,y | Smooth cubic (reflects last control point) |
| Q/q | x1,y1,x,y | Quadratic bezier |
| T/t | x,y | Smooth quadratic (reflects last control point) |
| A/a | rx,ry,rot,large,sweep,x,y | Elliptical arc |
| Z/z | — | Close path |

All commands support both absolute (uppercase) and relative (lowercase) coordinates. The tokenizer handles implicit command repetition (e.g., multiple coordinate pairs after a single `L`).

#### Shape Elements
Direct conversion for: `<rect>` (including rounded corners via arc construction), `<circle>`, `<ellipse>`, `<polygon>`, `<polyline>`, `<line>`. Circles and ellipses are approximated with 48 segments.

#### Bezier Curve Flattening
**De Casteljau subdivision** with adaptive flatness tolerance of **0.1mm**:

- **Cubic beziers** (`flattenCubicBezier`) — checks flatness by computing max squared distance of control points from the chord line. If `maxSq <= 16 * tolerance²`, outputs the endpoint. Otherwise subdivides at t=0.5 and recurses on both halves.
- **Quadratic beziers** (`flattenQuadraticBezier`) — promoted to cubic (standard 2/3 control point formula) then uses cubic flattener.

#### Arc Flattening
Full SVG arc endpoint parameterization → center parameterization conversion per the SVG spec. Radii are auto-corrected when too small (lambda scaling). Point count is adaptive: `steps = max(4, ceil(|dθ| / (2 * acos(1 - tolerance/maxRadius))))`.

#### Transform Stack
Recursive transform accumulation through `<g>` nesting. Supports: `matrix`, `translate`, `scale`, `rotate` (with optional center point), `skewX`, `skewY`. Transforms are composed via 2D affine matrix multiplication.

#### Unit Conversion
ViewBox → mm coordinate mapping. Supports: mm, cm, in, pt, px (at 96 DPI = 3.7795 px/mm).

#### Fill-Based Cut Classification
The core classification logic in `importSVGToShapes()`:

```
if fill is white → island (leave uncut)
else if path is open OR fill="none" → engrave profile trace  
else → pocket (material removal)
```

SVG default fill (when attribute is absent/undefined) is treated as black per SVG spec → pocket.

#### Clipper.js Boolean Difference for Islands
`subtractHoles()` in `pocketPathCuts.ts`:
1. Scale polygon vertices to integer space (×1000)
2. Ensure main polygon is CCW orientation (Clipper convention for subjects)
3. Add main polygon as subject, island polygons as clips
4. Execute `ctDifference` — returns polygon(s) with holes removed

#### Contour Offset Pocketing
`generateOffsetPaths()` in `pocketPathCuts.ts`:
1. Subtract islands from main polygon first
2. Start with inward offset = half bit diameter
3. Each subsequent pass offsets by `stepOver = diameter × stepOverPercent/100`
4. Uses `ClipperOffset` with `jtRound` join type and `etClosedPolygon` end type
5. Continues until offset produces empty result (polygon fully consumed)
6. Safety limit: 200 passes maximum
7. **Cutting order**: innermost first → outermost last (finish pass is the boundary)

#### V-Bit Effective Diameter
`effectiveDiameter()` calculates cutting width at a given depth:
- Flat end mill: returns `bitDiameter`
- V-bit: `min(2 × depth × tan(halfAngle), bitDiameter)`, capped by flute length if specified

### API / Interface Changes

**Exported functions:**
- `parseSVG(svgString: string): SVGParseResult` — main entry point for SVG parsing
- `importSVGToShapes(parseResult, options): ImportedShape[]` — converts parsed SVG to board shapes
- `isIslandFill(fill: string | undefined): boolean` — white fill detection utility
- `generatePocketPathGcode(config: PocketPathConfig): string` — freeform polygon pocket G-code
- `generateOffsetPaths(points, halfBit, stepOver, holes?): Point2D[][]` — Clipper contour offset
- `subtractHoles(points, holes, scaleFactor?): ClipperPath[]` — Clipper boolean difference
- `effectiveDiameter(depth, bitDiameter, vAngle?, fluteLength?): number` — V-bit width calc
- `pointInPolygon(px, py, polygon): boolean` — ray casting PIP test

## Edge Cases & Gotchas

1. **Degenerate arcs** (rx=0 or ry=0) are treated as straight lines to the endpoint.
2. **Implicit L after M** — per SVG spec, coordinates after the initial M/m pair are treated as implicit line-to commands. Implemented correctly.
3. **Smooth curve reflection** (S/T commands) — only reflects the control point if the previous command was the matching cubic/quadratic type. Otherwise uses current point (no reflection).
4. **Rounded rects** — converted to path strings with arc commands and re-parsed through `parseDAttribute`, avoiding duplicate arc flattening code.
5. **Single-point paths** (< 2 segments) are silently dropped.
6. **Same-file re-import** — file input is reset after each import to allow re-importing the same SVG file.
7. **ViewBox-less SVGs** — default to px→mm conversion at 96 DPI when no viewBox is present.
8. **Clipper orientation** — main polygon is forced to CCW before boolean operations; incorrect winding would produce empty results.

## Risks

- **Complex SVGs from design tools** (Figma, Illustrator) often use `<use>`/`<defs>` heavily — these import as empty with only a warning.
- **Style-based fills** (CSS `<style>` blocks, `style=""` attributes) are not parsed — fill detection falls through to SVG default (black → pocket), which may be incorrect.
- **Very large SVGs** with thousands of paths could produce slow Clipper operations or hit the 200-pass safety limit.
- **Text not converted to paths** is the most common user error — warning is shown but may be missed.

## Stories (retroactive)

| Story | PR | Description |
|-------|----|-------------|
| E16-S2 | #368 | Engrave tool modal + SVG import UI |
| E16-S2 fix | #369 | SVG import — unified drag, correct multi-path positioning, simplified panel |
| E16-S2 fix | #370 | SVG group drag, scale input blur-clamp, toolpath position offset |
| E16-S4a | #381 | SVG pocket auto-detection + canvas hatched fill |
| E16-S4b | #382 | Replace zigzag pocketing with Clipper.js contour offset |
| E16-S4-fix | #387 | SVG pocket cleanup |
| E16-S4c | #388 | Island subtraction in pocket toolpaths |

## Decisions Log

1. **Fill-based classification over explicit UI toggle** — SVG fill attribute is the source of truth for pocket vs engrave vs island. No user override UI (yet). Rationale: most SVG artwork already encodes intent in fill/stroke; manual classification per-path doesn't scale.
2. **Contour offset over zigzag pocketing** — PR #382 replaced the original zigzag clearing with Clipper.js concentric offset. Produces cleaner finishes, respects arbitrary polygon shapes, and naturally supports V-bit depth-varying diameter.
3. **Innermost-first cutting order** — clears center material first, finish pass traces the boundary last. Better surface finish on the visible edge.
4. **De Casteljau over parametric sampling** — adaptive subdivision produces fewer points on straight-ish curves and more points on tight curves, vs uniform `t` stepping which over/under-samples.
5. **0.1mm flatness tolerance** — balances G-code size vs visual smoothness for CNC (router bit diameter is typically 1.5-6mm, so sub-0.1mm deviations are invisible).
6. **DOMParser for SVG parsing** — leverages browser's built-in XML parser rather than a custom tokenizer. Robust, handles malformed XML with error detection.
7. **js-clipper over custom boolean ops** — proven library for polygon clipping; handles edge cases (self-intersections, degeneracies) that a hand-rolled implementation would miss.

## Known Issues / Tech Debt

1. **No CSS style parsing** — fill/stroke from `<style>` blocks or `style=""` attributes are ignored. Would require a CSS parser or regex extraction.
2. **`pocketCuts.ts` (rectangular) still uses zigzag** — only freeform paths use contour offset. Could unify under `pocketPathCuts.ts`.
3. **No user override for cut classification** — if the fill-based heuristic is wrong, user has no way to manually switch a shape between pocket/engrave/island.
4. **Text and QR code** — UI placeholders exist ("Coming Soon") but are not implemented.
5. **No multi-color island detection** — only exact white is treated as island. A luminance threshold or user-configurable color mapping would be more robust.
6. **`idCounter` is module-level mutable state** in `svgParser.ts` — not reset between imports, IDs grow monotonically across the session. Harmless but slightly untidy.
7. **No progress indicator** — large SVG files are parsed synchronously on the main thread. Could block UI for complex files.
8. **Warnings shown via `alert()`** — should be replaced with an in-app toast/notification system.
