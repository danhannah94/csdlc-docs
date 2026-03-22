# Assembly Mode — Sub-system Design Doc

*Retroactive design doc — documents the original product mode and its evolution.*
*Last updated: March 2026*

---

## Overview

### What Is This Sub-system?

Assembly Mode is the original product vision for Routr — a full furniture design-to-manufacture pipeline. Users design individual boards, connect them into assembled furniture using joints, preview the result in 3D, nest boards onto stock sheets for efficient material use, and simulate the CNC toolpaths before exporting G-code.

It represents the multi-board, multi-step paradigm: **Design → Assembly → 3D Preview → Nesting → Simulation.**

### The Story

Assembly Mode was the first thing built. The idea was ambitious: model an entire piece of furniture, define how boards connect with joints, let the app figure out the 3D positioning, then lay everything out on stock sheets and generate G-code for the whole project.

The problem was combinatorial complexity. Supporting every combination of board shapes × joint types × edge connections × assembly configurations was an enormous surface area. After months of development, Assembly Mode could essentially make boxes — which is useful, but far from the vision.

The strategic pivot came when **Workshop Mode** was conceived: instead of modeling entire assemblies, focus on single boards and think in terms of woodworking tools (table saw, router, drill press). This dramatically reduced complexity while delivering immediate value to CNC owners. Workshop Mode became the launch product, and Assembly Mode was feature-flagged off — still functional, still valuable, but not ready for prime time.

Assembly Mode remains the long-term vision. Workshop Mode gets users in the door; Assembly Mode is where the product grows.

### Current Status

| Capability | Status |
|-----------|--------|
| Board design (shapes, cuts) | Shipped (shared with Workshop Mode) |
| Board linking (edge-click UI) | Shipped, feature-flagged off |
| Joint system (box joints) | Shipped — full geometry + toolpaths |
| Joint system (miter) | Partial — 3D geometry, no toolpath |
| Joint system (dovetail) | Stub — type defined, no implementation |
| Joint system (dado/rabbet) | Partial — params + UI + solver, no geometry |
| Constraint solver | Shipped |
| 3D assembly preview | Shipped, feature-flagged off |
| Multi-sheet nesting | Shipped |
| Auto-nest algorithm | Shipped |
| Simulator | Shipped |

---

## Architecture

### System Boundary

Assembly Mode owns the multi-board workflow: board linking, joint generation, constraint solving, 3D positioning, and the assembly-specific UI (AssemblyCanvas, 3D Preview tab). It shares the board design surface (Board Setup tab) with Workshop Mode, and delegates to the G-code Pipeline and Simulator sub-systems for output.

### Architecture Diagram

```
┌─ Assembly Mode ──────────────────────────────────────────────┐
│                                                              │
│  Board Setup ──→ Assembly Canvas ──→ 3D Preview              │
│  (shared w/      (edge-click         (Three.js               │
│   Workshop)       linking UI)         assembly view)          │
│       │                │                                     │
│       │                ▼                                     │
│       │         Constraint Solver                            │
│       │         (BFS, conflict                               │
│       │          detection)                                  │
│       │                │                                     │
│       ▼                ▼                                     │
│  ┌─────────────────────────────┐                             │
│  │  Joint System               │                             │
│  │  (geometry mods, toolpaths) │                             │
│  └─────────────────────────────┘                             │
│       │                                                      │
│       ▼                                                      │
│  Nesting (stock sheet layout)                                │
│       │                                                      │
└───────┼──────────────────────────────────────────────────────┘
        ▼
  G-code Pipeline (sub-system)  →  Simulator (sub-system)
```

### Key Interfaces

| Interface | Type | Consumers |
|-----------|------|-----------|
| `BoardLink` creation/modification | Store actions | AssemblyCanvas, RightPanel |
| `constraintSolver()` | Engine function | AssemblyCanvas (conflict display), RightPanel (unreachable warnings) |
| `Joint` geometry modifications | Engine functions | 3D Preview (mesh generation), G-code Pipeline (toolpath generation) |
| `BoardPlacement` on `StockSheet` | Store state | Nesting Canvas, G-code export |

---

## Related Epics

| Epic | Doc | Status | Summary |
|------|-----|--------|---------|
| Assembly & Joints | [assembly-joints.md](../epics/assembly-joints.md) | Shipped (flagged off) | Board linking, constraint solver, joint geometry |
| Nesting | [nesting.md](../epics/nesting.md) | Shipped | Multi-sheet stock layout, auto-nest algorithm |

---

## Cross-Cutting Concerns

| Concern | How This Sub-system Is Affected |
|---------|-------------------------------|
| Coordinate Systems | 3D Preview uses Three.js coordinates (Y-up, S=0.01 scale). Joint geometry must produce correct meshes in this space. |
| G-code Pipeline | Joint toolpaths (box joint cuts, miter cuts) feed into the same pipeline as workshop operations. |
| Feature Flags | Assembly Mode hidden via `AppMode` toggle. Workshop Mode hides Assembly and 3D Preview tabs. |

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Early 2026 | Build Assembly Mode first | Full furniture pipeline was the original vision — most ambitious, most differentiated. | Start with simple single-board tools (ended up pivoting to this anyway) |
| Feb 2026 | Pivot to Workshop Mode for launch | Assembly Mode's combinatorial complexity meant it could only make boxes. Workshop Mode delivers immediate value with dramatically less complexity. | Keep pushing on Assembly Mode (too slow to launch), abandon Assembly Mode (waste of work) |
| Feb 2026 | Feature-flag Assembly Mode, don't delete | Working code, still valuable long-term. Hiding it preserves the investment while shipping a polished Workshop Mode. | Delete Assembly Mode (too aggressive), ship both (Assembly not polished enough) |
| Mar 2026 | Hide via AppMode, not dedicated feature flag | Simpler than a separate flag. Workshop Mode just doesn't render the Assembly/3D Preview tabs. | Add ASSEMBLY_MODE_ENABLED flag (mentioned in code but not the actual gating mechanism) |

---

## Risks & Constraints

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Assembly Mode stays permanently flagged off | Medium | Medium — investment wasted | Revisit after Workshop Mode proves product-market fit |
| Joint type expansion is expensive | High | Medium — each new joint type needs geometry + toolpaths + 3D + tests | Prioritize by user demand, not completeness |
| Constraint solver doesn't scale beyond boxes | Medium | High — limits the furniture complexity users can model | Acceptable for now; redesign solver if Assembly Mode becomes priority |

---

## Known Issues / Tech Debt

| Issue | Severity | Notes |
|-------|----------|-------|
| Dead `AssemblyPanel.tsx` (~400 lines) | Low | Old dropdown UI, replaced by AssemblyCanvas. Still has test coverage. |
| Dovetail joint is type-only stub | Medium | Defined in types, no geometry/toolpath/3D implementation |
| Dado/rabbet joints partial | Medium | Params + UI + solver depth, but no geometry modification |
| Miter joint missing toolpaths | Medium | 3D geometry works, but can't generate G-code for miter cuts |
| Untyped `_role`/`_linkId` on Joint objects | Low | Private properties stamped at runtime, not in TypeScript types |
| Box joints wrong edge in 3D (#175) | Low | Known visual bug in 3D preview |

---

*Assembly Mode is the long-term product vision. Workshop Mode is the launch vehicle. This sub-system doc captures both the architecture and the strategic narrative of how they relate.*
