# Epic Design Doc Template

*Copy this template to `docs/projects/your-project/epics/epic-name.md` and fill it out through collaborative refinement.*
*Based on [PROCESS.md](process.md) — epic-level design docs.*

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

## Retrospective Notes

What went well, what didn't, what we'd do differently. Filled in after the epic ships.

---

*This epic doc is refined collaboratively (Step 0) before stories are broken down (Step 1). Once refined, the AI Lead extracts context from this doc to craft sub-agent prompts (Step 2).*
*Update this doc as implementation reveals new information — design docs are living documents.*
