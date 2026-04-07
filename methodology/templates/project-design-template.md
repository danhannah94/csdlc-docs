# Project Design Doc Template

*Copy this template to `docs/projects/your-project/design.md` and fill it out through collaborative refinement.*
*Based on [PROCESS.md](../process.md) — project-level design docs.*

---

## Overview

### What Is This?

One paragraph. What does this project do, in plain language? If someone reads nothing else, this paragraph should give them the full picture.

### Who Is It For?

Target users, their pain points, and how this project solves them.

### Business Model

How does this make money (or justify its existence)? Pricing strategy, monetization approach, growth model.

---

## Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | | Why this framework? |
| State Management | | Why this approach? |
| Backend / API | | Why this service? |
| Database | | Why this DB? |
| Auth | | Why this provider? |
| Deployment | | Why this platform? |
| Payments | | Why this processor? |

### Key Libraries & Dependencies

List non-obvious dependencies that agents need to know about. Don't list React or TypeScript — list the things that would surprise someone.

| Library | Purpose | Notes |
|---------|---------|-------|
| | | |

---

## System Architecture

### Architecture Diagram

```
[High-level system diagram — layers, boundaries, data flow]
```

### Layer Descriptions

Describe each architectural layer, its responsibility, and its boundaries. What owns what? Where does logic live vs. UI vs. state?

### Data Flow

How does data move through the system? From user input to persistence (or output). Identify the critical paths.

---

## Data Model

### Core Entities

Describe the primary data structures — what they represent, their relationships, and any non-obvious constraints.

```typescript
// Key type definitions or schema
```

### Entity Relationships

How do entities reference each other? What's the ownership model? What happens when an entity is deleted?

### State Management

How is application state structured? What's ephemeral vs. persisted? What are the key state transitions?

---

## Deployment & Infrastructure

### Environments

| Environment | URL | Purpose | Deploy Trigger |
|-------------|-----|---------|---------------|
| Local | | Development | Manual |
| Staging | | Pre-production testing | |
| Production | | Live | |

### CI/CD Pipeline

How does code get from a merged PR to production? What gates exist?

### Infrastructure Dependencies

External services, API keys, environment variables. What breaks if a third-party goes down?

---

## Security Model

### Authentication & Authorization

How do users authenticate? What are the permission levels? How is auth enforced?

### Data Protection

What's sensitive? How is it stored/transmitted? What are the trust boundaries?

### Known Attack Surfaces

Be honest about where the system is vulnerable. What's mitigated, what's accepted risk?

---

## AI Interface Architecture *(Optional)*

*Consider this section for any project where AI agents will interact with the application — for testing, automation, or as a product feature. See [Routr's Validation Pipeline](../../projects/routr/epics/validation.md) for a concrete example.*

### Dev-Mode API

Does this project expose a programmatic interface for AI agents to drive the application?

```typescript
// Example: window.__yourApp = { ... }
```

| Capability | Method | Purpose |
|-----------|--------|---------|
| | | |

### Exposure Strategy

| Environment | Available? | How? |
|-------------|:----------:|------|
| Development | Yes / No | Always-on / feature flag |
| Staging | Yes / No | Feature flag / restricted |
| Production | Yes / No | Only when a product feature requires it |

### MCP Server *(If Applicable)*

If AI agents need to drive the app from outside the browser (testing, product features, integrations):

- **Package strategy**: Separate package or embedded in app?
- **Tool surface**: What capabilities does the MCP server expose?
- **Connection model**: Playwright bridge to browser, direct engine calls, or both?

### Why This Matters

The AI interface isn't just developer tooling — it's an architectural layer. Applications designed for AI interaction get:

- **Reliable testing**: Direct function calls instead of fragile UI automation
- **AI-driven features**: The same hooks that power testing become the foundation for AI product features
- **Speed**: Programmatic access is orders of magnitude faster than Playwright UI clicking

---

## Cross-Cutting Concerns

Architectural concerns that span multiple epics. These aren't features — they're constraints, conventions, or systems that every epic needs to be aware of.

| Concern | Summary | Affected Epics | Dedicated Doc? |
|---------|---------|---------------|----------------|
| | | | [link](epics/concern-name.md) or N/A |

*If a concern is deep enough to warrant its own document (e.g., a coordinate system that's caused repeated bugs), give it a dedicated doc in `epics/`. If it's simple (e.g., "all internal units are mm"), a row in this table is sufficient.*

---

## Risks & Constraints

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| | | | |

### Known Limitations

What does this system explicitly NOT do? What are the scaling boundaries? What assumptions might break?

### Tech Debt

Known shortcuts, deferred decisions, things that work but shouldn't stay this way.

---

## Sub-system Index

Sub-systems are architectural boundaries or product-level capabilities. If you removed one, multiple unrelated features would break.

| Sub-system | Doc | Status | Summary |
|------------|-----|--------|---------|
| | [link](sub-systems/subsystem-name.md) | Draft / Refined / Complete | One-line summary |

## Epic Index

Epics are deliverable feature work built on top of sub-systems. Remove one and a specific capability disappears.

| Epic | Doc | Status | Parent Sub-system | Summary |
|------|-----|--------|-------------------|---------|
| | [link](epics/epic-name.md) | Draft / Refined / Complete | | One-line summary |

---

## Decisions Log

Significant architectural decisions, with rationale and alternatives considered. Keeps the "why" alive for future sessions.

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| | | | |

---

## Glossary

Project-specific terminology that agents need to understand. Don't assume shared vocabulary.

| Term | Definition |
|------|-----------|
| | |

---

*This design doc is the source of truth for project-level architecture. Epic-level details live in `epics/`. Implementation specifics live in the project repo's `WORKFLOW.md`.*
*Update this doc when architecture changes — stale docs are worse than no docs.*
