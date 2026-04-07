# Epic Design Doc Template

*Copy this template to `docs/projects/your-project/epics/epic-name.md` and fill it out through collaborative refinement.*
*Based on [PROCESS.md](../process.md) — epic-level design docs.*

---

## Overview

### What Is This Epic?

One paragraph. What capability does this epic add or define? Why does it exist?

### Problem Statement

What's broken, missing, or painful without this? What triggered this work?

### Goals & Non-Goals

**Goals:**
- What this epic WILL accomplish

**Non-Goals:**
- What this epic explicitly WON'T do (prevents scope creep)

---

## Context

### Current State

How does the system work today (before this epic)? What's the baseline?

### Affected Systems

Which parts of the architecture does this epic touch? Reference the project design doc's architecture diagram.

| System / Layer | How It's Affected |
|---------------|-------------------|
| | |

### Dependencies

What must exist before this epic can be implemented? What other epics or systems does this depend on?

### Dependents

What other epics or features depend on this one? Who's blocked until this ships?

---

## Design

### Approach

How does this work? Describe the technical approach at a level that resolves ambiguity without dictating every implementation detail. Include diagrams where they help.

### Data Model Changes

New entities, modified schemas, or state changes introduced by this epic.

```typescript
// New or modified type definitions
```

### Key Algorithms / Logic

If this epic involves non-trivial logic (calculations, transformations, state machines), describe it here. This is the section that prevents agents from guessing.

### API / Interface Changes

New or modified interfaces — between components, between layers, or external APIs.

---

## Edge Cases & Gotchas

Things that will trip up implementers if not documented. Be specific.

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| | | |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| | | | |

---

## Testing Strategy

*How does this epic enter the project's validation pipeline? Reference the [Validation Pipeline epic](../../projects/routr/epics/validation.md) for the full testing framework.*

### Test Layers

Which testing layers apply to this epic?

| Layer | Applies? | Notes |
|-------|:--------:|-------|
| **Unit tests** | Yes / No | New functions, algorithms, or transforms that need isolated testing |
| **Integration tests (G-code)** | Yes / No | Does this epic affect G-code output? If yes, fixtures are mandatory. |
| **Integration tests (Visual)** | Yes / No | Does this epic change 2D design tab or 3D sim tab rendering? If yes, screenshot baselines are mandatory. |
| **Cross-layer measurement** | Yes / No | Does this epic place or move things spatially? If yes, 2D and 3D measurements must agree. |
| **Physical validation (E2E)** | Yes / No | Does this epic introduce a new operation type or change coordinate logic? If yes, a real CNC cut is required. |

### Required Fixtures

List the test fixtures this epic needs. Each fixture tests one concern.

| Fixture Name | What It Tests | Priority |
|-------------|--------------|----------|
| | | 🔴 High / 🟡 Medium / 🟢 Low |

### Verification Rules

1. **Every new feature must have at least one fixture.** No exceptions.
2. **G-code baselines are mandatory** if the feature affects G-code output.
3. **Visual baselines are mandatory** if the feature changes what the design tab or sim tab renders.
4. **Cross-layer measurement assertions are mandatory** for spatial features.
5. **Physical validation is required only for new operation types** — new cut type, new edge treatment, new toolpath algorithm.
6. **Fixture updates require human review** — diffs must be inspected, not rubber-stamped.

---

## Stories

Features/stories extracted from this epic. Each becomes a ticket for sub-agent execution.

| Story | Summary | Status | PR |
|-------|---------|--------|----|
| S0 | | | |
| S1 | | | |
| S2 | | | |

*Stories are broken down during Step 1 (Story Breakdown) with full acceptance criteria. This table is the index.*

---

## Decisions Log

Decisions made during refinement of this epic.

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| | | | |

---

## Known Issues / Tech Debt

Issues discovered during or after implementation. Feeds back into the project-level tech debt tracker.

| Issue | Severity | Notes |
|-------|----------|-------|
| | | |

---

*This epic doc is refined collaboratively (Step 0) before stories are broken down (Step 1). Once refined, the AI Lead extracts context from this doc to craft sub-agent prompts (Step 2).*
*Update this doc as implementation reveals new information — design docs are living documents.*
