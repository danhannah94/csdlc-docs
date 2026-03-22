# Sub-system Design Doc Template

*Copy this template to `docs/projects/your-project/sub-systems/subsystem-name.md` and fill it out through collaborative refinement.*
*Based on [PROCESS.md](process.md) — sub-system design docs sit between the project-level doc and epic-level docs.*

---

## Overview

### What Is This Sub-system?

One paragraph. What architectural boundary or product-level capability does this sub-system define? Why does it exist as its own layer?

### The Story

How did this sub-system come to be? What decisions led here? This is where you capture the strategic narrative — pivots, trade-offs, evolution over time.

### Current Status

What's shipped, what's in progress, what's planned?

| Capability | Status |
|-----------|--------|
| | Shipped / In Progress / Planned / Deferred |

---

## Architecture

### System Boundary

What does this sub-system own? What's inside the boundary vs. outside? Where does it interface with other sub-systems?

### Architecture Diagram

```
[Sub-system internal architecture — components, data flow, boundaries with other sub-systems]
```

### Key Interfaces

How does this sub-system expose its capabilities to the rest of the application? What do epics and features plug into?

| Interface | Type | Consumers |
|-----------|------|-----------|
| | Function / Component / Store / Event | Which epics or sub-systems use this |

---

## Related Epics

Epics that are built on or within this sub-system. Each epic has its own design doc with implementation details.

| Epic | Doc | Status | Summary |
|------|-----|--------|---------|
| | [link](../epics/epic-name.md) | Shipped / In Progress / Planned | One-line summary |

---

## Cross-Cutting Concerns

How does this sub-system interact with project-level cross-cutting concerns?

| Concern | How This Sub-system Is Affected |
|---------|-------------------------------|
| | |

---

## Decisions Log

Significant decisions about this sub-system's scope, boundaries, or approach.

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| | | | |

---

## Risks & Constraints

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| | | | |

---

## Known Issues / Tech Debt

| Issue | Severity | Notes |
|-------|----------|-------|
| | | |

---

*Sub-system docs define architectural boundaries and product-level capabilities. They sit between the project design doc (system-wide) and epic docs (feature-level). The test: if you removed this sub-system, would multiple unrelated features break? If yes, it's a sub-system.*
*Update this doc when the sub-system's boundaries or interfaces change.*
