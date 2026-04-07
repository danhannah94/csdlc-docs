# Edge Treatments — Epic Design Doc
*Retroactive design doc — documents the implemented system as of March 2026.*

## Flags for Review

1. **Roundover is disabled** — `ROUNDOVER_ENABLED = false` in `featureFlags.ts`. The roundover code path (`roundoverCuts.ts`) uses its own local `getEdgeLine()` with **screen-space coordinates (Y-down)**, while all other edge treatments use the centralized `edgeMapping.ts` with **CNC coordinates (Y-up)**. This coordinate mismatch was never resolved; the feature was disabled before launch.

2. **Roundover not wired into the unified dispatcher** — `generateEdgeTreatmentGcode()` in `edgeTreatmentCuts.ts` has no `case 'roundover'` — it throws `Unsupported edge treatment type` for roundover. The roundover code is completely standalone.

3. **Miter cuts use a third coordinate convention** — `miterCuts.ts` has its own `calculateEdgeCoordinates()` that treats top as min-Y / bottom as max-Y (screen-space), distinct from both `edgeMapping.ts` (CNC-space) and `roundoverCuts.ts` (also screen-space but different implementation). Three separate edge coordinate implementations exist in the codebase.

4. **The 3-commit Y-flip cycle** — Git history shows `d543099` (add Y-flip), `5a07977` (flip edge treatment coords), `39b2dc9` (remove incorrect Y-flip), `93d7e03` (remove screenEdgeToCncEdge flip). The edge coordinate system was reworked at least 3 times before stabilizing with the current `edgeMapping.ts` centralization (`afb3ed3`).

5. **V-bit angle convention** — `vBitAngle` is the **half-angle** (TA from tool file), e.g., 45° for a 90° V-bit. This is documented in code but easy to misuse — chamfer default is `vBitAngle ?? 45` which silently assumes a 90° V-bit.

---

## Overview

### What Is This Epic?

Edge treatments are decorative or functional modifications applied to individual edges of boards in Routr. The system supports chamfers, roundovers (disabled), rabbets, and dados. Each treatment generates its own G-code toolpath operations for CNC execution.

### Problem Statement

Workshop projects require edge modifications — chamfers for aesthetics, rabbets for panel backs, dados for shelves. These need to be modeled per-edge, parameterized, and translated into accurate G-code with correct CNC coordinates.

### Goals & Non-Goals

**Goals:**
- Per-edge treatment selection and parameterization via UI
- G-code generation for chamfer, rabbet, and dado operations
- Multi-pass depth progression respecting tool stepDown limits
- V-bit band logic for chamfers exceeding angled edge depth
- Unit-aware parameter inputs (metric/imperial)

**Non-Goals:**
- Roundover (deferred — disabled via feature flag)
- Compound treatments (multiple treatments on same edge — data model supports it but UI doesn't encourage it)
- Automatic treatment propagation across linked boards

---

## Context

### Affected Systems

| System | Role |
|--------|------|
| `src/types/index.ts` | Data model — `EdgeTreatment`, params types |
| `src/engine/edgeTreatments.ts` | Query helpers, default params |
| `src/engine/toolpath/edgeTreatmentCuts.ts` | G-code generation (chamfer, rabbet, dado) |
| `src/engine/toolpath/edgeMapping.ts` | Centralized coordinate mapping (Screen↔CNC↔Three.js) |
| `src/engine/toolpath/roundoverCuts.ts` | Roundover G-code (disabled, standalone) |
| `src/engine/toolpath/miterCuts.ts` | Miter cuts (related — shares V-bit geometry concepts) |
| `src/engine/tools/vBitGeometry.ts` | V-bit angled edge depth calculation |
| `src/components/panels/EdgeTreatmentPanel.tsx` | UI panel for treatment editing |
| `src/lib/featureFlags.ts` | `ROUNDOVER_ENABLED` flag |

### Dependencies

- **Coordinate systems** — Three distinct systems: Screen (Y-down), CNC (Y-up), Three.js Scene. `edgeMapping.ts` is the single source of truth for conversions.
- **V-bit geometry** — Chamfer depth/width calculations depend on `getAngledEdgeDepth()` from `vBitGeometry.ts`, which uses the tool's half-angle (TA field) and diameter.
- **Nesting/projection** — `getEffectiveDimensions()` provides rotation-aware board dimensions.
- **GcodeBuilder** — Shared G-code output abstraction with post-processor support (grbl, linuxcnc).
- **Zustand store** — `useProjectStore` provides `updateEdgeTreatment`, `removeEdgeTreatment`.

### Dependents

- G-code export pipeline consumes edge treatment output
- 3D preview uses `cncToScene()` for visualization
- Tool change detection considers edge treatments for tool selection

---

## Design

### Approach

End-to-end flow: **Edge selection → Treatment type + params → Toolpath generation → G-code**

1. **Selection**: User clicks a board edge in select mode → `EdgeTreatmentPanel` auto-expands. Treatment is created with an `id`, `edge`, `type`, and default `params`.

2. **Parameterization**: `EdgeTreatmentPanel.tsx` renders type-specific inputs via `TreatmentParamInputs`. All dimensional values use `ClampedNumberInput` with unit conversion (`toDisplay`/`toInternal`). Params are clamped to board thickness on blur.

3. **Coordinate mapping**: Edge names from the UI are in screen-space. `edgeMapping.ts` provides `screenEdgeToCncEdge()` to flip top↔bottom for CNC. `getCncEdgeLine()` and `getCncDadoLine()` return start/end points in stock-sheet CNC space.

4. **Toolpath generation**: `generateEdgeTreatmentGcode()` dispatches by type to `generateChamferGcode`, `generateRabbetGcode`, or `generateDadoGcode`. Each generates multi-pass G-code with safe-height retracts.

5. **G-code output**: Uses `GcodeBuilder` with post-processor support for comment style and line endings.

### Data Model

```typescript
type EdgeTreatmentType = 'chamfer' | 'roundover' | 'rabbet' | 'dado';

interface EdgeTreatment {
  id: string;
  type: EdgeTreatmentType;
  edge: Edge;              // 'top' | 'bottom' | 'left' | 'right'
  params: EdgeTreatmentParams;
}

// On Board:
interface Board {
  edgeTreatments?: EdgeTreatment[];
}
```

**Params per type:**

| Type | Params | Defaults |
|------|--------|----------|
| Chamfer | `width` (mm from edge), `angle` (degrees) | width = boardThickness, angle = 45° |
| Roundover | `radius` (mm) | radius = boardThickness / 4 (panel) or 3mm (engine) |
| Rabbet | `width` (mm into board), `depth` (mm down) | width = thickness/2, depth = thickness/2 |
| Dado | `width` (groove width), `depth`, `position` (mm from edge start) | width = thickness/2, depth = thickness/3, position = 10mm |

> **Note:** Default params differ slightly between `edgeTreatments.ts` (engine) and `EdgeTreatmentPanel.tsx` (UI). Engine roundover default radius is 3mm; panel uses `thickness/4`. Engine dado position default is 0; panel uses 10.

### Key Algorithms / Logic

#### Chamfer Depth Calculation with V-bits

```
totalDepth = chamferWidth × tan(chamferAngle)
passOffset = (totalDepth - passDepth) / tan(vBitHalfAngle)
```

Passes progress from **inside to outside**: first pass is deepest inward (largest offset from edge), last pass is at the edge (offset = 0). This is the correct direction for V-bit chamfering — the final pass defines the clean edge line.

#### Chamfer Band Logic

When `totalDepth > angledEdgeDepth`, the cut is split into **bands** to keep the V-bit within its angled cutting edge (avoiding the flat vertical portion above):

```
maxBandDepth = angledEdgeDepth
numBands = ceil(totalDepth / maxBandDepth)
```

Bands progress from innermost (shallowest) to outermost (deepest, at edge). Each band has its own set of stepDown passes.

#### V-bit Angled Edge Depth (`vBitGeometry.ts`)

```
angledEdgeDepth = diameter / (2 × tan(halfAngle))
```

Clamped to `fluteLength` if available. Returns `undefined` for non-V-bit tools. This determines the maximum single-band depth for chamfers — plunging beyond this engages the vertical portion of the bit and creates a flat shelf.

#### Rabbet Pass Strategy

Double loop: outer loop is depth passes (`ceil(depth / stepDown)`), inner loop is step-over passes (`ceil(width / bitDiameter)`). Each pass is a single linear cut along the edge at the step-over offset.

#### Dado Positioning

Dados run **parallel** to the specified edge, inset by `dadoPosition`. Step-over passes widen the groove perpendicular to the edge. Uses `getCncDadoLine()` which is functionally identical to `getCncEdgeLine()` — both offset from the edge in the same way.

#### Edge Coordinate Mapping (edgeMapping.ts)

Three coordinate systems are managed:

1. **Screen/2D (Y-down)**: top = min-Y, bottom = max-Y. This is what the user sees.
2. **CNC/G-code (Y-up)**: top = max-Y (back of machine), bottom = min-Y (front).
3. **Three.js Scene**: CNC X→scene X, CNC Y→scene Z, CNC Z→scene Y (all centered on stock).

`screenEdgeToCncEdge()` swaps top↔bottom, preserves left/right. This is its own inverse (round-trip safe).

`getCncEdgeLine(edge, bx, by, width, height, inset)` returns a `{start, end}` line in CNC stock-sheet space:
- **top**: horizontal at `by + height - inset` (max-Y side)
- **bottom**: horizontal at `by + inset` (min-Y side)
- **left**: vertical at `bx + inset`
- **right**: vertical at `bx + width - inset`

### API / Interface Changes

Edge treatments are stored on the `Board` model as `edgeTreatments?: EdgeTreatment[]`. The Zustand store exposes `updateEdgeTreatment(boardId, treatmentId, partial)` and `removeEdgeTreatment(boardId, treatmentId)`.

The UI filters available treatment types at runtime: `['chamfer', ...(ROUNDOVER_ENABLED ? ['roundover'] : []), 'rabbet', 'dado']`.

---

## Edge Cases & Gotchas

### The Top/Bottom Edge Swap

Screen "top" (min screen-Y) maps to CNC "bottom" (min CNC-Y, front of machine). This was the source of a multi-commit bug cycle. The current solution centralizes the mapping in `edgeMapping.ts` — all toolpath generators receive edges already in CNC space and use `getCncEdgeLine()` directly. The `screenEdgeToCncEdge()` function exists but is **not called** by the edge treatment G-code generators — they receive edges as-is and work in CNC coordinates.

### The 3-Layer Patch Cycle

Git history reveals the Y-flip was added, removed, and re-added across commits `d543099` → `5a07977` → `39b2dc9` → `93d7e03`. The root cause was confusion about which layer should do the flip. Resolution: the flip happens **once** at the UI→engine boundary, not inside G-code generators.

### V-bit Angle Math

`vBitAngle` is the **half-angle** (not the included angle). A 90° V-bit has `vBitAngle = 45`. The chamfer code uses this directly as `halfAngleRad = vBitAngle * PI / 180`. Getting this wrong doubles or halves the offset.

### Roundover Coordinate Mismatch

`roundoverCuts.ts` has its own `getEdgeLine()` function that treats top as min-Y (screen-space), while `edgeMapping.ts`'s `getCncEdgeLine()` treats top as max-Y (CNC-space). These are incompatible. The roundover code was never migrated to use the centralized mapper.

### Default Param Divergence

The engine (`edgeTreatments.ts`) and the UI panel (`EdgeTreatmentPanel.tsx`) have separate `getDefaultParams` functions with slightly different defaults.

---

## Risks

- **Roundover re-enablement** will require rewriting `roundoverCuts.ts` to use `edgeMapping.ts` and wiring it into `generateEdgeTreatmentGcode()`.
- **Miter cuts** use yet another coordinate convention — if edge treatments and miter cuts ever need to interact (e.g., chamfer meeting a miter), the coordinate systems will conflict.
- **No hardware validation** — edge treatment G-code has not been validated on physical CNC hardware (noted in the roundover flag comment, but applies broadly).

---

## Stories (retroactive)

| ID | Title | Status |
|----|-------|--------|
| E20-S1 | Input blur-clamp audit — consistent onBlur clamping | ✅ Done (`#377`) |
| E20-S2 | Edge coordinate mapping — centralized 2D↔CNC↔3D mapper | ✅ Done (`#378`) |
| E20-S3 | Chamfer V-bit angled edge depth calculation (band logic) | ✅ Done (`#379`) |
| E20-S4 | Disable roundover edge treatment for launch | ✅ Done (`#376`) |
| E20-S5 | QA cleanup — roundover toolbar, edge mapping fix, dado canvas | ✅ Done (`#380`) |
| #322 | Edge selection in select mode + auto-expand panel | ✅ Done (`#336`) |
| #330 | Chamfer pass progression inside to outward | ✅ Done (`#333`) |
| #338 | Roundover toolpath fixes (multiple attempts) | ⚠️ Fixed then disabled (`#341`, `#342`, `#343`) |
| #339 | Dado toolpath edge rotation | ✅ Done (`#340`) |

---

## Decisions Log

| Decision | Rationale |
|----------|-----------|
| Centralize edge mapping in `edgeMapping.ts` | Eliminate per-file coordinate bugs; single source of truth for Screen↔CNC↔Scene |
| Disable roundover via feature flag, not code removal | Preserve the implementation for future re-enablement; code is complete but coordinate-incompatible |
| V-bit `vBitAngle` = half-angle (TA from tool file) | Matches industry-standard tool library format; avoids division in hot paths |
| Band logic for deep chamfers | Prevents V-bit flat-shelf artifacts when chamfer depth exceeds the angled cutting edge |
| Chamfer passes progress inside→outside | Final pass defines the clean visible edge line |
| `getCncDadoLine` separate from `getCncEdgeLine` | Semantic clarity, even though implementation is identical — dado "offset" means distance from edge, not inset from cut line |

---

## Known Issues / Tech Debt

1. **Roundover disabled** (`ROUNDOVER_ENABLED = false`) — Uses standalone coordinate system incompatible with `edgeMapping.ts`. Never integrated into the unified `generateEdgeTreatmentGcode()` dispatcher (throws on `'roundover'` type). Needs full rewrite to centralized coordinates before re-enablement.

2. **Three coordinate implementations** — `edgeMapping.ts` (CNC, used by chamfer/rabbet/dado), `roundoverCuts.ts::getEdgeLine()` (screen-space), and `miterCuts.ts::calculateEdgeCoordinates()` (screen-space with corner clearance logic). Should be consolidated.

3. **Default param divergence** — `edgeTreatments.ts` and `EdgeTreatmentPanel.tsx` both define `getDefaultParams` with different values. Should be single source.

4. **No hardware validation** — All G-code output is tested via unit tests against expected coordinates, but has not been run on physical CNC machines.

5. **`screenEdgeToCncEdge()` exists but may be unused** — The edge treatment G-code path appears to receive edges already in CNC convention. The function's actual call sites should be audited to confirm it's used at the correct boundary.

6. **Miter cuts not in edge treatment system** — Miter joints generate their own G-code via `miterCuts.ts` but are not modeled as `EdgeTreatment` objects. They use the joints system instead. If a board has both a miter joint and an edge treatment on the same edge, behavior is undefined.
