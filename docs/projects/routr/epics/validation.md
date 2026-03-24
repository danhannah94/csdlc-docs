# Validation Pipeline — Epic Design Doc

*Status: 🔄 In Refinement (Step 0)*
*Authors: Dan Hannah & Clay*
*Created: March 23, 2026*

---

## Overview

### What Is This Epic?

The Validation Pipeline is Routr's quality assurance backbone — a multi-layered testing system that ensures the app produces dimensionally accurate, safe G-code for real CNC machines. It bridges the gap between "the app works in the browser" and "this G-code is safe to run on wood."

This is Routr's first forward-looking design doc. Every previous doc was retroactive — reading existing code and documenting what was already built. This one designs before building. That's the CSDLC leveling up.

### Problem Statement

Routr has solid unit test coverage across toolpath generators, coordinate transforms, and SVG parsing. But there is **no automated way to verify that the full pipeline — from board configuration to final G-code output — produces correct results.** And beyond digital verification, there's no formal process for validating that G-code performs correctly on a physical CNC machine.

**What's broken today:**

- A change to the toolpath generator could silently alter G-code output for every operation type, and no test would catch it
- Coordinate system bugs (like the edge treatment flip and kerf line flip) made it to real cuts before being discovered
- The design tab, simulator, and G-code can each show *different* things — a chamfer placed on the top edge in the design tab showed up on the bottom in the sim tab but cut correctly in real life. No automated test catches cross-layer visual inconsistencies.
- There's no "known good" G-code baseline to compare against
- Physical validation cuts have been done ad hoc — no formal protocol, no documented results, no traceability from design dimensions to measured dimensions
- There's no way for AI sub-agents to programmatically interact with the app for testing — all interaction requires UI automation through a human-designed interface

**The deeper problem:**

In an AI-driven development workflow, the AI writes the code — but how does the AI *prove to the human* that the code works? Unit tests are invisible to most humans. The validation pipeline is the mechanism by which the AI demonstrates correctness: "Here's the G-code output, here's what the design tab looks like, here's what the sim looks like, here are the dimensional measurements, and here's how all of these match." The human reviews *evidence*, not code. This shifts the human role from *writing* the proof to *reviewing* the proof — the AI-driven equivalent of test-driven development.

**What triggered this work:**

The coordinate flip bug class — the same category of bug appeared in edge treatments and table saw kerf lines. Both were caught during physical cuts, not by tests. The chamfer top/bottom flip between design and sim tabs proved that **G-code correctness alone isn't sufficient** — visual consistency across the app's layers matters too. With launch approaching, the question is: what else might be wrong that we haven't cut yet?

### Goals

- Define the complete validation pipeline from automated tests to physical verification
- Establish integration tests driven by JSON test fixtures that validate G-code output, 2D canvas rendering, AND 3D simulator rendering
- Create a formal physical validation protocol with documented results
- Build an in-app measurement tool for fast dimensional comparison between design and physical cuts — available in both 2D design tab and 3D sim tab for cross-layer verification
- Define AI-accessible hooks that let sub-agents programmatically drive and test the app
- Build an MCP server that wraps the app's capabilities for AI-driven exploratory testing and future product features
- Make "is this operation validated?" answerable for any feature at any time
- Enable fast, confident iteration on the codebase without fear of silent regressions
- Establish a testing strategy pattern that applies to all future epics/features

### Non-Goals

- **Not fixing the kerf line flip bug** — that's a standalone ticket
- **Not adding roundover validation** — roundover is feature-flagged off for launch
- **Not automating physical cuts** — the human runs the CNC machine; this epic defines the protocol

---

## Context

### Current State

**What exists today:**

| Layer | Status | Coverage |
|-------|--------|----------|
| Unit tests | ✅ Solid | Individual toolpath generators, edge mapping, bit offset, arc G-code, coordinate transforms, SVG parsing |
| Functional/Integration tests | ❌ None | No end-to-end pipeline tests, no JSON fixtures, no visual regression |
| E2E tests | ❌ None | No formal physical validation protocol, no dimensional measurement tooling |

**The unit tests are solid but test functions in isolation.** None of them test the full pipeline: "given this board with these shapes and these tool settings, the G-code output should be exactly X and the 2D canvas should look exactly like Y and the sim should look exactly like Z."

### Affected Systems

| System / Layer | How It's Affected |
|---------------|-------------------|
| G-code Pipeline | Primary target — integration tests validate end-to-end G-code output |
| Workshop Mode (2D Canvas) | Visual regression screenshots validate design tab rendering; measurement tool overlay |
| Simulator (3D Canvas) | Visual regression screenshots validate 3D sim rendering; measurement tool overlay |
| Coordinate Systems | Cross-layer visual tests AND cross-layer measurement catch coordinate transform inconsistencies |
| Edge Treatments | Specific validation needed for each treatment type × edge combination |
| SVG Import | Engrave and pocket toolpaths need integration test coverage |
| Zustand Store | AI-accessible hooks need programmatic store access |

### Dependencies

- **Coordinate flip fix (SHIPPED)** — centralized `edgeMapping.ts` must be in place before establishing baselines
- **Kerf line flip bug (OPEN)** — should be fixed before table saw baselines are established
- **Playwright** — already available; needed for screenshot capture and AI-driven app interaction

### Dependents

- **Production launch** — launch requires minimum validation confidence per operation type
- **Future epics/features** — every new feature enters the pipeline through this testing framework
- **PDF-to-G-code pipeline** — AI hooks built for testing become the foundation for AI-driven app features
- **Epic design template** — testing strategy pattern from this epic gets extracted into the template for all future epics

---

## Design

### Three-Layer Testing Architecture

```mermaid
graph TD
    subgraph L1["Layer 1: Unit Tests (Existing)"]
        UT["Individual function tests<br/>Math, transforms, parsing"]
    end

    subgraph L2["Layer 2: Functional / Integration Tests (NEW)"]
        JSON["JSON Test Fixture<br/>(board + shapes + tools)"]
        JSON --> HOOKS["AI-Accessible Hooks<br/>Load fixture → Zustand store"]
        HOOKS --> GCODE["G-code Output<br/>→ assert matches expected .nc"]
        HOOKS --> CANVAS2D["2D Design Tab Screenshot<br/>→ pixel-diff against golden .png"]
        HOOKS --> SIM3D["3D Sim Tab Screenshot<br/>→ pixel-diff against golden .png"]
        HOOKS --> MEASURE_CROSS["Cross-layer measurement<br/>Design tab dims = Sim tab dims?"]
        GCODE --> CROSS["Cross-layer consistency<br/>All outputs agree?"]
        CANVAS2D --> CROSS
        SIM3D --> CROSS
        MEASURE_CROSS --> CROSS
    end

    subgraph L3["Layer 3: E2E Tests (NEW)"]
        FIXTURE["Test fixture with<br/>known dimensions"]
        FIXTURE --> MEASURE_APP["In-app measurement tool<br/>→ expected dimensions"]
        FIXTURE --> CUT["Cut on CNC machine"]
        CUT --> MEASURE_REAL["Measure with calipers<br/>→ actual dimensions"]
        MEASURE_APP --> COMPARE["Compare:<br/>design vs actual"]
        MEASURE_REAL --> COMPARE
        COMPARE --> RECORD["Validation record<br/>±0.5mm tolerance"]
        EYEBALL["Simulator eyeball check<br/>Does the sim look right?"]
        EYEBALL --> CUT
    end

    L1 --> |"Functions correct"| L2
    L2 --> |"Pipeline + visuals correct"| L3
    L3 --> |"Real-world confirmed"| LAUNCH["Launch Confidence ✅"]
```

Each layer catches different categories of bugs:

| Layer | What It Catches | Cost | Frequency |
|-------|----------------|------|-----------|
| **Unit tests** | Math errors, logic bugs, type mismatches | Free (seconds) | Every code change |
| **Functional/Integration** | G-code pipeline regressions, coordinate bugs, visual rendering inconsistencies between 2D design/3D sim/G-code, cross-layer dimensional mismatches | Free (seconds–minutes) | Every code change |
| **E2E** | Real-world factors: tool deflection, material variance, machine quirks, dimensional accuracy, simulator visual trust | High (material + time) | New operation types, coordinate changes, pre-launch |

---

### Quality Gates

Quality gates define WHERE in the CSDLC pipeline each check lives. This maps the testing layers to the development process:

```mermaid
graph LR
    subgraph DEV["Development (Step 3)"]
        UNIT_G["🚦 Gate 1: Unit Tests<br/>Must pass before PR"]
        INT_G["🚦 Gate 2: Integration Tests<br/>G-code assertions must match"]
    end

    subgraph CI["CI / PR (Automated)"]
        CI_G["🚦 Gate 3: GitHub Actions<br/>Unit + G-code integration<br/>BLOCKS merge"]
    end

    subgraph REVIEW["Review (Steps 4-6)"]
        VIS_G["🚦 Gate 4: Visual Regression<br/>AI Lead reviews screenshot diffs"]
        LEAD_G["🚦 Gate 5: AI Lead Review<br/>Output matches ticket scope"]
        HUMAN_G["🚦 Gate 6: Human Review<br/>Intent met, feels right"]
    end

    subgraph LAUNCH["Pre-Launch"]
        E2E_G["🚦 Gate 7: Physical Validation<br/>Dimensional accuracy on CNC"]
    end

    DEV --> CI --> REVIEW --> LAUNCH
```

| Gate | Type | Where in Pipeline | Who | Blocks? |
|------|------|------------------|-----|---------|
| **Gate 1: Unit Tests** | Automated | Step 3 (sub-agent execution) | Sub-agent runs before submitting | Yes — PR not created if tests fail |
| **Gate 2: Integration Tests** | Automated | Step 3 (sub-agent execution) | Sub-agent verifies G-code assertions | Yes — PR not created if assertions fail |
| **Gate 3: CI Check** | Automated | PR creation | GitHub Actions | **Yes — required status check, blocks merge** |
| **Gate 4: Visual Regression** | Semi-automated | Step 4 (AI Lead review) | AI Lead reviews Playwright screenshot diffs | Yes — AI Lead must approve visual changes |
| **Gate 5: AI Lead Review** | Manual (AI) | Step 4 | AI Lead verifies scope, quality, no regressions | Yes — work doesn't reach human until AI Lead approves |
| **Gate 6: Human Review** | Manual (Human) | Step 6 | Dan reviews intent, UX, "feels right" | Yes — final approval before merge/ship |
| **Gate 7: Physical Validation** | Manual (Human) | Pre-launch / new op types | Dan cuts on CNC, measures with calipers | Yes — new operation types must pass before launch |

**The key insight:** Gates 1-3 are cheap and automated — they run on every change. Gates 4-5 are AI-assisted — the AI Lead surfaces evidence for review. Gate 6 is human judgment — the irreplaceable "does this feel right?" check. Gate 7 is physical — rare but critical for real-world trust.

---

### CI/CD Strategy

**Hybrid approach: GitHub Actions CI for cheap, repetitive checks + local execution for AI-driven work.**

| Test Type | Where It Runs | Why |
|-----------|--------------|-----|
| Unit tests | **GitHub Actions CI** | Fast, deterministic, zero token cost. Catches regressions on every PR without spending AI compute. |
| G-code integration tests | **GitHub Actions CI** | Deterministic string comparison — perfect for CI. No rendering needed. |
| Visual regression tests (screenshots) | **Local first, CI later** | Playwright screenshot tests can be flaky across environments. Start local, promote to CI once stable. |
| E2E (physical validation) | **Local only** | Human runs the CNC machine. |

The sub-agent development workflow makes traditional CI/CD less critical for the *development* loop — the AI Lead reviews and tests before PRs are created. But CI adds a safety net *between* sessions: a PR that sneaks in a coordinate bug gets caught on push, even if no one ran the tests manually. CI is the overnight security guard; sub-agents are the daytime crew.

GitHub Actions CI will be configured as a **required status check** — PRs cannot merge until unit tests and G-code integration tests pass. This is a quality gate, same as any other in the pipeline.

---

### Cross-Cutting Concern: AI-Accessible Interface

Since this app is designed and written by AI, it makes sense to build first-class interfaces for AI to drive and test it. This isn't just about testing — it's about making the app AI-native. The hooks built here for validation become the foundation for future AI features like PDF-to-G-code.

**This may be a core architectural pattern for AI-designed apps.** Every modern app has a UI layer (for humans), an API layer (for services), and a data layer (for persistence). AI-designed apps need an **AI layer** — a structured interface purpose-built for LLMs to understand, drive, and extend the app. Not a REST API (too low-level, no semantic context). Not the UI (too fragile, designed for humans). An MCP-style interface that gives AI agents the same fluency with the app that a senior developer has. This pattern should be considered for every CSDLC project going forward.

**Dev-mode API (`window.__routr`):**

Exposed in development/test builds, and on staging behind a feature flag for AI feature experimentation. NOT in production until a product feature requires it.

```typescript
// Proposed dev-mode API surface
window.__routr = {
  // Fixture loading
  loadFixture(fixture: TestFixture): void,      // Load JSON fixture into Zustand store
  getState(): ProjectState,                       // Read current store state

  // G-code
  generateGcode(): string,                        // Trigger pipeline, return G-code string
  generateToolpaths(): ToolpathOperation[],        // Get intermediate toolpath data

  // Measurement (same engine used by in-app measurement tool)
  measureDistance(pointA: Point, pointB: Point): number,  // Distance between two points (mm)
  measureFromEdge(shapeId: string, edge: 'top' | 'bottom' | 'left' | 'right'): number,

  // Canvas capture
  captureDesignCanvas(): Promise<ImageData>,       // 2D canvas screenshot
  captureSimCanvas(): Promise<ImageData>,          // 3D sim screenshot

  // Store actions
  addBoard(config: BoardConfig): string,           // Returns board ID
  addShape(boardId: string, shape: ShapeConfig): string,
  setToolSettings(settings: ToolSettings): void,
  addEdgeTreatment(boardId: string, treatment: EdgeTreatmentConfig): void,
};
```

**Why this matters:**

- **Testing**: Sub-agents call `loadFixture()` + `generateGcode()` instead of clicking through UI
- **Reliability**: No flaky CSS selectors, no waiting for animations, no "button didn't render yet"
- **Speed**: Direct function calls are orders of magnitude faster than Playwright UI automation
- **Future features**: PDF-to-G-code AI reads a plan, calls `addBoard()` + `addShape()` to build the design programmatically
- **Validation**: `measureDistance()` and `measureFromEdge()` make dimensional verification instant — in both 2D and 3D contexts

#### MCP Server: AI-Native Tool Interface (Separate Package)

The `window.__routr` API gives programmatic access from within the browser. But to let AI agents **reason about and drive the app from outside** — connecting from Claude, OpenClaw sub-agents, or any MCP-compatible client — we wrap the same capabilities in an [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) server.

The MCP server lives as a **separate package** in the repo (`packages/routr-mcp` or similar) with its own versioning. This keeps concerns clean — the app doesn't depend on MCP, and MCP can evolve independently. It also positions the MCP server for distribution beyond just testing (future product features, marketplace integrations).

**What MCP adds over a raw API:**

MCP doesn't just expose endpoints — it gives each tool rich descriptions, parameter schemas, and contextual guidance so the LLM knows *when, why, and how* to use each tool. The AI doesn't just know "there's a `loadFixture` function." It knows "use `loadFixture` when you want to set up a test scenario — it takes a fixture JSON describing a board with shapes and tool settings, and after loading you can generate G-code or capture screenshots to verify the result."

**Proposed MCP tool surface:**

```
MCP Server: routr-tools
├── create_board          — Create a board with dimensions and material
├── add_shape             — Add a cut, pocket, hole, path, or slot to a board
├── add_edge_treatment    — Apply chamfer, dado, or rabbet to a board edge
├── set_tool_settings     — Configure bit, feeds, speeds
├── import_svg            — Import an SVG file for engrave/pocket
├── generate_toolpaths    — Generate toolpath operations from current state
├── generate_gcode        — Generate G-code and return as string
├── capture_design_view   — Screenshot the 2D design canvas
├── capture_sim_view      — Screenshot the 3D simulator at a standard angle
├── measure_distance      — Measure between two points or shape-to-edge
├── load_fixture          — Load a test fixture JSON into the app
├── get_project_state     — Read the current Zustand store state
└── validate_gcode        — Compare generated G-code against an expected file
```

#### AI-Driven Exploratory Testing & the Protagonist/Antagonist Pattern

Scripted integration tests (fixtures + assertions) catch regressions. But an AI agent connected via MCP can do something far more powerful — **creative, exploratory testing:**

- *"Design a board with a 2-inch pocket centered on a 6×12 board, generate the G-code, and verify the pocket coordinates are correct"*
- *"Try every edge treatment on every edge and check for visual inconsistencies between the design and sim tabs"*
- *"Create a stress test — maximum shapes, weird dimensions, overlapping cuts — and report what breaks"*
- *"Verify that changing the bit diameter correctly adjusts all toolpath offsets"*

The AI understands woodworking concepts, understands the app's capabilities, and can creatively find edge cases that no human would think to write a fixture for. This is a fundamentally different testing paradigm — not "did it change?" (regression) but "is it correct?" (verification).

**Protagonist/Antagonist Agent Pattern:**

This maps naturally to the CSDLC pipeline:

```mermaid
graph LR
    PROTAGONIST["🟢 Protagonist Agent<br/>(Step 3: Implementation)<br/>Writes the code,<br/>creates the PR"] --> ANTAGONIST["🔴 Antagonist Agent<br/>(Step 4-5: Review + QA)<br/>Connects via MCP,<br/>tries to break it"]
    ANTAGONIST --> |"Finds issues"| PROTAGONIST
    ANTAGONIST --> |"No issues found"| HUMAN["👤 Human Review<br/>(Step 6)"]
```

- The **protagonist** is the sub-agent executing the ticket (Step 3). It writes code, runs scripted tests, creates the PR.
- The **antagonist** is a separate AI agent that reviews the protagonist's work (Steps 4-5). It connects via MCP, loads the affected fixtures, runs exploratory tests, tries creative edge cases, and reports findings.
- The **human** reviews the evidence from both agents (Step 6) and makes the final call.

This adversarial pattern drives quality because the antagonist has different incentives than the protagonist — its job is to find problems, not ship features. It's the AI equivalent of a dedicated QA engineer who didn't write the code.

**The product vision connection:**

The MCP server built for testing IS the integration layer for Routr's AI product features:

| Use Case | How MCP Enables It |
|----------|-------------------|
| **AI-driven testing** | Sub-agents connect via MCP, design test scenarios, execute, report findings |
| **Protagonist/Antagonist QA** | Antagonist agent connects via MCP to challenge protagonist's work |
| **PDF-to-G-code** | AI reads a woodworking plan PDF, connects to Routr via MCP, calls `create_board` + `add_shape` to build the design, exports G-code |
| **AI Design Assistant** | "I want a cutting board with a juice groove" → AI agent uses MCP tools to design it in Routr |
| **Plan Marketplace ingestion** | Automated pipeline that takes uploaded plans and converts them to Routr projects |

**Architecture:**

```mermaid
graph TD
    subgraph CLIENTS["MCP Clients"]
        PROTAGONIST["🟢 Protagonist Agent"]
        ANTAGONIST["🔴 Antagonist Agent"]
        CLAUDE["Claude / AI Agents"]
        OPENCLAW["OpenClaw Sub-agents"]
    end

    subgraph MCP["MCP Server: routr-tools<br/>(separate package)"]
        TOOLS["Tool definitions<br/>+ descriptions + schemas"]
    end

    subgraph APP["Routr App (Browser)"]
        API["window.__routr API"]
        STORE["Zustand Store"]
        ENGINE["Engine Pipeline"]
        CANVAS["Canvas Renderers"]
        MEASURE["Measurement Engine"]
    end

    CLIENTS --> |"MCP protocol"| MCP
    MCP --> |"Playwright bridge<br/>or direct API"| APP
    API --> STORE
    API --> ENGINE
    API --> CANVAS
    API --> MEASURE
```

The MCP server connects to the running Routr app (via Playwright for browser-based interaction, or directly to the engine for headless G-code generation). This means we can run both:

- **Headed mode**: MCP drives the actual app UI (for visual testing)
- **Headless mode**: MCP calls engine functions directly (for fast G-code testing)

---

### Layer 2: Functional / Integration Tests (Detail)

This is the core of the epic. Integration tests tie together multiple outputs from a single input.

#### Integration Test Architecture

```mermaid
graph TD
    subgraph INPUT["Test Input"]
        FIXTURE["JSON Test Fixture<br/>📄 fixture.json"]
    end

    subgraph LOAD["Loading"]
        HOOKS["window.__routr.loadFixture()<br/>or MCP load_fixture"]
    end

    subgraph OUTPUTS["Test Outputs (all from same fixture)"]
        GCODE["G-code Output<br/>📄 expected.nc"]
        DESIGN["2D Design Tab<br/>📸 design.png"]
        SIMTAB["3D Sim Tab<br/>📸 sim.png"]
        MEASURE_2D["2D Measurements<br/>shape-to-edge distances"]
        MEASURE_3D["3D Measurements<br/>toolpath-to-edge distances"]
    end

    subgraph ASSERTIONS["Assertions"]
        A1["G-code matches expected file"]
        A2["2D screenshot matches golden image"]
        A3["3D screenshot matches golden image"]
        A4["2D measurements = 3D measurements<br/>(cross-layer consistency)"]
    end

    FIXTURE --> HOOKS
    HOOKS --> GCODE --> A1
    HOOKS --> DESIGN --> A2
    HOOKS --> SIMTAB --> A3
    HOOKS --> MEASURE_2D --> A4
    HOOKS --> MEASURE_3D --> A4
```

**How the test runner works:**

1. **Load** — Read fixture JSON, call `loadFixture()` to hydrate Zustand store
2. **Generate** — Call `generateGcode()`, capture the output string
3. **Capture** — Trigger Playwright screenshots of design tab and sim tab at locked viewport/camera/theme
4. **Measure** — Call `measureFromEdge()` for key shapes in both 2D design context and 3D sim context
5. **Assert** — Compare all outputs against stored baselines:
    - G-code: character-by-character diff against `.expected.nc`
    - Screenshots: pixel-diff with tolerance threshold against golden `.png` files
    - Measurements: 2D design dimensions must equal 3D sim dimensions (cross-layer consistency)
6. **Report** — On failure, output the diffs for human review

**Baseline management:**

- First run with a new fixture: `npm run test:integration -- --update` saves current outputs as baselines
- Subsequent runs compare against baselines
- When a baseline changes intentionally: review the diff, run `--update` to accept
- **Fixture updates require human review** — the diff must be inspected, not rubber-stamped

**Test commands:**

```bash
# Run G-code integration tests only (fast, CI-friendly)
npm run test:integration:gcode

# Run visual regression tests (requires browser, slower)
npm run test:integration:visual

# Run all integration tests
npm run test:integration

# Update baselines after intentional changes
npm run test:integration -- --update
```

#### The JSON Test Fixture

A fixture file fully describes a project state — everything needed to reproduce a specific board configuration deterministically:

```json
{
  "name": "straight-cut-basic",
  "description": "Single vertical table saw cut through a board",
  "board": {
    "width": 200,
    "height": 100,
    "thickness": 19
  },
  "shapes": [
    {
      "type": "rectangle",
      "cutType": "table-saw",
      "position": { "x": 100, "y": 0 },
      "params": { "width": 0, "height": 100 },
      "depth": 19
    }
  ],
  "toolSettings": {
    "bitDiameter": 6.35,
    "feedRate": 1000,
    "plungeRate": 500,
    "stepDown": 3,
    "safeHeight": 5,
    "spindleSpeed": 18000
  },
  "edgeTreatments": [],
  "units": "metric"
}
```

**Fixture design principles:**

- Each fixture tests **one concern** — a straight cut fixture, a pocket fixture, etc.
- Use **known, simple dimensions** — easy to mentally verify
- Specify **ALL settings explicitly** — never rely on defaults (defaults change, fixtures shouldn't)
- A **comprehensive fixture** combines multiple operations to test ordering and tool changes

#### Three Assertion Types + Cross-Layer Measurement

**1. G-code assertion** — Load fixture via `loadFixture()`, call `generateGcode()`, compare against stored `.expected.nc` file. Character-by-character match. If it differs → developer reviews diff → update expected file (intentional) or fix regression.

**2. 2D canvas screenshot** — Load fixture, capture design tab canvas via Playwright at locked viewport + theme. Compare against golden `.design.png` using pixel-diff with tolerance threshold (absorbs anti-aliasing). Catches: shapes in wrong positions, cuts on wrong side, visual artifacts.

**3. 3D sim screenshot** — Load fixture, generate toolpaths, capture sim tab at a **fixed camera angle**. Compare against golden `.sim.png`. Catches the chamfer-on-wrong-edge class of bugs — where design tab shows one thing and sim shows another.

**4. Cross-layer measurement** — Use the measurement API to check the same dimensions in both the 2D design tab and 3D sim tab. "This pocket is 47mm from the left edge" should be true in BOTH contexts. If the numbers disagree, we've found a coordinate bug. This is the most precise way to catch the kerf line flip class of bugs — not just "does it look right?" but "do the numbers match?"

#### Standard Camera Angles for 3D Sim Screenshots

Consistent camera angles ensure deterministic screenshots:

| Angle | Name | Use Case |
|-------|------|----------|
| **Default perspective** | `perspective-45` | 45° elevation, front-right corner. Standard angle for most fixtures. |
| **Top-down orthographic** | `top-down` | Bird's eye view. Useful for verifying XY positions of cuts, pockets, drill holes. |
| **Front orthographic** | `front` | Straight-on front view. Critical for edge treatment fixtures — shows which edge the treatment is on. |
| **Right orthographic** | `right-side` | Side view. Useful for depth verification. |

Most fixtures use `perspective-45` only. Edge treatment fixtures should also capture `front` to explicitly verify top vs. bottom edge rendering.

#### File Structure

```
cncmill-app/src/__fixtures__/
├── straight-cut-basic/
│   ├── fixture.json          # Input: board + shapes + tools
│   ├── expected.nc           # Expected G-code output
│   ├── design.png            # Golden 2D canvas screenshot
│   └── sim.png               # Golden 3D sim screenshot (perspective-45)
├── pocket-rectangle/
│   └── ...
├── drill-basic/
│   └── ...
├── edge-chamfer-top/
│   ├── fixture.json
│   ├── expected.nc
│   ├── design.png
│   ├── sim.png               # perspective-45
│   └── sim-front.png         # front view (verify correct edge)
├── svg-engrave/
│   └── ...
└── multi-operation/          # Comprehensive: multiple ops on one board
    └── ...
```

#### Fixture Inventory

| Fixture | What It Tests | Priority |
|---------|--------------|----------|
| `straight-cut-basic` | Table saw vertical cut | 🔴 High (blocked on kerf flip fix) |
| `pocket-rectangle` | Router rectangular pocket | 🔴 High |
| `pocket-circle` | Router circular pocket | 🟡 Medium |
| `drill-basic` | Single drill hole | 🔴 High |
| `drill-grid` | Multiple drill holes | 🟡 Medium |
| `profile-cut` | Board outline profile with tabs | 🔴 High |
| `edge-chamfer-top` | Chamfer on top edge | 🔴 High |
| `edge-chamfer-bottom` | Chamfer on bottom edge | 🔴 High |
| `edge-chamfer-left` | Chamfer on left edge | 🟡 Medium |
| `edge-chamfer-right` | Chamfer on right edge | 🟡 Medium |
| `edge-dado` | Dado on edge | 🟡 Medium |
| `edge-rabbet` | Rabbet on edge | 🟡 Medium |
| `svg-engrave` | SVG import with engrave toolpath | 🟡 Medium |
| `svg-pocket` | SVG with pocket detection | 🟡 Medium |
| `surfacing` | Planer/surfacing operation | 🟢 Low |
| `multi-operation` | Comprehensive: straight cut + pocket + drill + edge treatment | 🔴 High |

---

### Layer 3: E2E Tests (Detail)

E2E testing is the full loop: design in app → verify in simulator → cut on CNC → measure physical piece → compare against design dimensions.

#### Simulator Eyeball Check

Before cutting anything, the human reviews the simulator render: "Does this look like what I'd expect this cut to produce?" This is NOT automated — it's a human judgment call. The simulator already exists and works; this just formalizes reviewing sim output before committing material.

**When to do it:**

- After establishing new fixture baselines
- After any coordinate system changes
- Before physical validation cuts (preview what you're about to cut)

#### Physical Validation Protocol

**For each validation cut:**

1. **Design** — Load test fixture in Routr (or create the design matching the fixture)
2. **In-app measurement** — Use the measurement tool in the design tab to record expected dimensions (e.g., "hole center is 47.5mm from left edge, 32mm from top edge"). Verify the same measurements in the sim tab.
3. **Eyeball sim** — Check sim tab — does it look right?
4. **Export** — Generate G-code
5. **Setup** — Load G-code on CNC, set origin, verify material dimensions with calipers
6. **Cut** — Run the program
7. **Measure** — Use calipers to measure the same dimensions recorded in step 2
8. **Record** — Document: expected → actual → delta → pass/fail with comments

**Tolerance target: ±0.5mm (±0.020")**

Starting point. Tighter than most hobby CNC work but achievable with proper setup. Will adjust based on real measurement data.

#### Validation Matrix

| Operation Type | Previously Cut? | Needs Re-cut Pre-Launch? | Status |
|---------------|:-:|:-:|--------|
| Table saw (straight cut) | ✅ | 🔴 Yes (kerf line flip) | Blocked on bug fix |
| Router pocket (rectangle) | ✅ | 🟡 Re-verify dimensions | — |
| Router pocket (circle) | ✅ | 🟡 Re-verify dimensions | — |
| Router pocket (freeform/path) | ✅ | 🟡 Re-verify dimensions | — |
| Drill holes | ✅ | 🟡 Re-verify dimensions | — |
| Profile cut (board outline) | ✅ | 🟡 Re-verify dimensions | — |
| Edge treatment (chamfer) | ✅ | 🟡 Re-verify post-coordinate fix | — |
| Edge treatment (dado) | ❌ | 🔴 Yes (never cut) | Simple cut, low risk |
| Edge treatment (rabbet) | ❌ | 🔴 Yes (never cut) | Simple cut, low risk |
| SVG engrave | ✅ | 🟡 Re-verify | — |
| SVG pocket | ✅ | 🟡 Re-verify | — |
| Surfacing (planer) | ✅ | 🟢 Low priority | Simple operation |

#### Validation Record Template

Physical validation is pass/fail with comments. Photos are optional — useful for documentation but not required.

```markdown
## [Fixture Name] — Physical Validation Record

**Date:** YYYY-MM-DD
**Fixture:** `__fixtures__/[name]/fixture.json`
**Material:** [species, dimensions]
**Machine:** [CNC model]
**Bit:** [type, diameter]

### Measurements

| Dimension | Expected (app) | Actual (calipers) | Delta | Pass/Fail |
|-----------|:-:|:-:|:-:|:-:|
| Cut position from left edge | 100.0mm | 99.8mm | -0.2mm | ✅ |
| Cut depth | 19.0mm | 18.7mm | -0.3mm | ✅ |

### Checks
- [ ] Sim visual matches design tab
- [ ] Cut positions look correct
- [ ] Edge treatments on correct edges
- [ ] Cross-layer measurements match (design tab = sim tab)

### Result: ✅ PASS / ❌ FAIL
**Comments:** [observations, anomalies, lessons, anything worth noting]
```

---

### In-App Measurement Tool

**Purpose:** Surface the dimensional data the app already knows internally in a way that's useful for both validation and everyday use. Available in **both the 2D design tab and the 3D sim tab** for cross-layer verification.

**How it works:**

The app already stores exact positions, dimensions, and relationships between shapes and toolpaths. The measurement tool exposes this data:

- **Point-to-point distance** — Click two points on the canvas, see the distance in current units
- **Shape-to-edge distance** — Select a shape, see its distance from each board edge
- **Shape dimensions** — Select a shape, see its width, height, depth, position
- **Toolpath-to-edge distance** (sim tab) — Same measurements but against generated toolpaths

**Cross-layer validation use case:**

The measurement tool in the design tab shows "this pocket is 47mm from the left edge." The measurement tool in the sim tab should show the toolpath for that pocket starts 47mm from the left edge. If they disagree, we've found a coordinate bug. This is how we'd catch the kerf line flip — the design tab says the kerf line is at X=100, the sim tab says the toolpath is at X=100... but wait, the G-code says X=0 (board width minus 100 after flipY). The measurement tool makes the discrepancy visible without needing to read G-code.

**UX approach:**

Top toolbar "Measure" mode — similar to Fusion 360's approach. A tools dropdown in the top bar that activates measurement mode. When active:

- Clicking on the canvas shows dimension overlays
- Works in both 2D design tab (measures against shapes/board) and 3D sim tab (measures against toolpaths/board)
- Same visual language in both contexts for consistency
- Deactivate by clicking the mode again or pressing Escape

**Implementation approach:**

The measurement engine is shared between the UI tool and the `window.__routr` API. `measureDistance()` and `measureFromEdge()` serve both the in-app overlay and automated testing. This means integration tests can assert measurements programmatically — the same data the human sees in the overlay.

---

## Testing Strategy for New Features

When a new epic or feature is built, it enters the validation pipeline through a standard process:

```mermaid
graph TD
    NEW["New Feature / Epic"] --> FIXTURE["Create test fixture(s)<br/>covering the feature"]
    FIXTURE --> UNIT["Write unit tests<br/>for new engine functions"]
    FIXTURE --> INTEGRATION["Establish integration baselines<br/>G-code + 2D + 3D screenshots<br/>+ cross-layer measurements"]
    UNIT --> REVIEW["AI Lead reviews<br/>all baselines"]
    INTEGRATION --> REVIEW
    REVIEW --> ANTAGONIST{"Run antagonist<br/>agent? (optional)"}
    ANTAGONIST --> |Yes| EXPLORE["🔴 Antagonist explores<br/>via MCP, tries to break it"]
    ANTAGONIST --> |No| PHYSICAL
    EXPLORE --> PHYSICAL{"New operation type?"}
    PHYSICAL --> |Yes| CUT["Physical validation cut<br/>following E2E protocol"]
    PHYSICAL --> |No| DONE["✅ Feature validated"]
    CUT --> DONE
```

**Rules:**

1. **Every new feature must have at least one fixture.** No exceptions. The fixture is created during story breakdown (Step 1).
2. **G-code baselines are mandatory.** If the feature affects G-code output, integration test baselines must be established before the PR is merged.
3. **Visual baselines are mandatory for UI-affecting changes.** If it changes what the design tab or sim tab renders, screenshot baselines must be established.
4. **Cross-layer measurement assertions are mandatory for spatial features.** If the feature places or moves things on the canvas, measurement assertions must verify consistency between design and sim tabs.
5. **Physical validation is required only for new operation types.** New cut type, new edge treatment, new toolpath algorithm → needs a real cut. Bug fixes and UI changes do not.
6. **Fixture updates require human review.** When a fixture baseline changes, the diff must be reviewed by the human before accepting. No rubber-stamping.
7. **Antagonist testing is encouraged for complex features.** Connect an antagonist agent via MCP to exploratory-test the new feature. Not mandatory for every PR, but strongly recommended for new operation types and coordinate-affecting changes.

> 📝 **PROCESS NOTE:** This testing strategy should be extracted into the epic design doc template after this epic is finalized. Every future epic should include a "Testing Strategy" section that references these rules and specifies which fixtures will be created.

---

### Integration Test Strategy

How integration tests are structured and maintained as the codebase grows:

**Test organization:**

```
cncmill-app/
├── src/
│   ├── __fixtures__/              # JSON fixtures + golden files
│   └── __tests__/
│       ├── integration/
│       │   ├── gcode.test.ts      # G-code assertion tests (all fixtures)
│       │   ├── visual.test.ts     # Screenshot comparison tests (all fixtures)
│       │   └── measurement.test.ts # Cross-layer measurement tests
│       └── ...                    # Existing unit tests
├── playwright.config.ts           # Visual test configuration
└── package.json                   # Test scripts
```

**Writing a new integration test:**

1. Create a fixture directory under `__fixtures__/` with `fixture.json`
2. Run `npm run test:integration -- --update` to generate initial baselines
3. **Review the baselines manually** — do the G-code, screenshots, and measurements look correct?
4. Commit the fixture + baselines together
5. From this point forward, any change that affects this fixture's output will cause a test failure

**When tests fail:**

```mermaid
graph TD
    FAIL["❌ Integration test fails"] --> REVIEW["Review the diff"]
    REVIEW --> INTENTIONAL{"Intentional change?"}
    INTENTIONAL --> |Yes| INSPECT["Inspect: does new output<br/>look correct?"]
    INSPECT --> |Yes| UPDATE["npm run test:integration -- --update<br/>Commit updated baselines"]
    INSPECT --> |No| BUG["Bug in the change — fix it"]
    INTENTIONAL --> |No| REGRESSION["Regression — investigate<br/>and fix before merging"]
```

**Avoiding flaky visual tests:**

- Lock viewport size: `1280×720` for all Playwright screenshots
- Lock theme: light mode
- Lock camera angles: standard angles defined in fixture or test config
- Pixel-diff tolerance: allow ~1-2% pixel difference for anti-aliasing
- Run on consistent environment (same browser version)
- If a test is persistently flaky, increase tolerance or switch to structural assertion

---

## Edge Cases & Gotchas

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| Screenshot test fails due to rendering engine update | Pixel diff flags change | Need tolerance threshold; anti-aliasing differences are noise, not signal |
| G-code assertion fails after intentional change | Developer updates `.expected.nc` after reviewing diff | Easy to rubber-stamp — need discipline |
| Imperial vs metric fixtures | All fixtures use mm internally | Display unit shouldn't affect G-code, but need to verify |
| Tool settings defaults change | Assertions break if defaults change | Fixtures specify ALL settings explicitly, never rely on defaults |
| Multiple operations change ordering | G-code assertion catches it | Operation ordering logic is complex — assertion is the safety net |
| 3D camera angle changes | Sim screenshots fail | Camera locked to standard angles in test harness |
| Canvas/viewport resize | Design screenshots fail | Viewport size locked in test harness |
| Coordinate system changes | ALL assertions break (expected) | This is the point — forced review of all outputs after coordinate changes |
| Theme changes (dark/light) | Screenshots differ | Lock theme in test harness (light mode for consistency) |
| `window.__routr` available in prod | Security/bundle size concern | Dev/test default; staging with feature flag; prod only when product feature requires it |
| Measurement disagrees between 2D and 3D | Coordinate bug | This is exactly what cross-layer measurement tests are designed to catch |
| Antagonist agent finds a real bug | Protagonist must fix before merge | Great outcome — the pattern is working |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Screenshot tests are flaky across environments | Medium | High | Fixed viewport, fixed theme, tolerance threshold, consistent environment |
| Integration tests create false confidence | Medium | High | Tests catch *regressions*. Initial baselines must be validated by physical cuts + human review. |
| Fixture maintenance burden as features grow | Medium | Medium | One concern per fixture. Comprehensive fixture catches interaction bugs. |
| Physical validation bottleneck (Dan = only CNC) | High | High | Minimize physical cuts. Integration tests handle regressions. Physical only for new ops. |
| Playwright screenshot tests are slow | Medium | Low | Run visual tests separately from unit/G-code tests. Split test commands. |
| AI-accessible interface scope creep | Medium | Medium | Start with minimum viable API surface. Expand as needs emerge. |
| MCP server maintenance overhead | Medium | Low | Separate package keeps it isolated. Thin wrapper over `window.__routr`. |
| Kerf line flip contaminates table saw baselines | High | Medium | Fix bug BEFORE establishing table saw fixture baseline. |
| Antagonist agent is too aggressive (false positives) | Medium | Low | Human reviews antagonist findings before acting. Findings are suggestions, not blockers. |

---

## Features

Features extracted from this epic. Each becomes a set of implementable stories during Step 1.

| Feature | Summary | Dependencies | Status |
|---------|---------|-------------|--------|
| F1 | **AI-Accessible Interface** — `window.__routr` dev-mode API: fixture loading, G-code generation, canvas capture, measurement | None | |
| F2 | **Test Fixture Library** — JSON fixture format, TypeScript types, initial set of fixtures | F1 | |
| F3 | **G-code Integration Tests** — Load fixture → generate G-code → assert matches `.expected.nc` + CI pipeline | F1, F2 | |
| F4 | **Visual Regression Tests** — Playwright harness for 2D + 3D screenshot capture, pixel-diff against golden images, fixed camera angles | F1, F2 | |
| F5 | **In-App Measurement Tool** — Point-to-point and shape-to-edge measurement in both 2D design tab and 3D sim tab, top toolbar "Measure" mode | F1 | |
| F6 | **Cross-Layer Measurement Tests** — Automated assertions that 2D design dimensions match 3D sim dimensions | F1, F2, F5 | |
| F7 | **Physical Validation Protocol** — Measurement recording template, validation matrix tracker | F2, F5 | |
| F8 | **Comprehensive Fixture** — Multi-operation board exercising full pipeline | F2, F3, F4, F6 | |
| F9 | **MCP Server (routr-tools)** — Separate package, Model Context Protocol wrapper for AI-driven testing and future product features | F1 | |

```mermaid
graph TD
    F1["F1: AI-Accessible Interface<br/>(window.__routr)"] --> F2["F2: Test Fixture Library"]
    F1 --> F5["F5: In-App Measurement Tool<br/>(2D + 3D)"]
    F1 --> F9["F9: MCP Server<br/>(routr-tools, separate pkg)"]
    F2 --> F3["F3: G-code Integration Tests<br/>+ CI Pipeline"]
    F2 --> F4["F4: Visual Regression Tests"]
    F2 --> F6["F6: Cross-Layer<br/>Measurement Tests"]
    F5 --> F6
    F2 --> F7["F7: Physical Validation Protocol"]
    F5 --> F7
    F2 --> F8["F8: Comprehensive Fixture"]
    F3 --> F8
    F4 --> F8
    F6 --> F8
    F9 --> |"Enables protagonist/<br/>antagonist pattern"| F8
```

*Features are broken down into implementable stories during Step 1 (Story Breakdown). This table is the feature index.*

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| 2026-03-23 | Kerf line flip bug is standalone, not in this epic | Bug fix, not validation concern. Blocks table saw baseline. | Include in epic (rejected — different concern) |
| 2026-03-23 | Full pipeline vision in one doc; layers become features | Complete picture in one doc; incremental execution | Only G-code testing (rejected — misses visual regression where real bugs were) |
| 2026-03-23 | First forward-looking design doc | Designing before building prevents the bug class that triggered this epic | Code first, doc later (rejected — caused the coordinate bugs) |
| 2026-03-23 | JSON test fixtures over hardcoded test data | Readable, shareable, used by both G-code and visual tests | Hardcoded TS (rejected — can't share across test types) |
| 2026-03-23 | Visual regression via Playwright screenshots | Catches cross-layer inconsistencies (design vs sim) that G-code tests miss | No visual testing (rejected — chamfer flip proved this is necessary) |
| 2026-03-23 | Simulator validation = human eyeball | Automated photo comparison too complex; eyeball is faster and sufficient | Pixel comparison with physical photos (rejected — brittle) |
| 2026-03-23 | ±0.5mm starting tolerance | Conservative; will adjust after first measurements | — |
| 2026-03-23 | Include measurement tool in this epic | Dual purpose (validation + user feature); enables E2E protocol and cross-layer verification | Separate epic (rejected — too tightly coupled to validation) |
| 2026-03-23 | Three-layer testing model (unit → integration → E2E) | Cleaner than four layers; sim eyeball is part of E2E, not its own layer | Four layers (rejected — over-granular) |
| 2026-03-23 | Build AI-accessible interface (window.__routr) | App is built by AI — should be testable by AI. Hooks enable testing AND future AI features (PDF-to-G-code). Core architectural pattern for AI-designed apps. | Playwright-only UI automation (rejected — fragile, slow, fights against the grain) |
| 2026-03-23 | Hybrid CI/CD: GitHub Actions for deterministic tests, local for visual/AI-driven | CI catches regressions between sessions without token cost; sub-agents handle dev-loop testing | All-CI or all-local (both rejected — see CI/CD section) |
| 2026-03-23 | CI blocks PR merges (required status check) | Quality gate — same principle as all other gates in the pipeline | Advisory only (rejected — too easy to ignore) |
| 2026-03-23 | MCP server as separate package | Clean separation of concerns; independent versioning; positions for distribution beyond testing | Part of app (rejected — couples testing infra to app) |
| 2026-03-23 | MCP server wrapping `window.__routr` for AI-driven testing | MCP gives LLMs structured tool access with context — enables exploratory testing AND future product features. Testing infrastructure IS the product integration layer. | Raw API only (rejected — misses LLM context layer) |
| 2026-03-23 | Protagonist/antagonist agent pattern for QA | Adversarial testing drives quality — antagonist has different incentives than protagonist | Single agent does both (rejected — conflicting goals) |
| 2026-03-24 | Measurement tool in both 2D design tab and 3D sim tab | Cross-layer measurement catches coordinate bugs (like kerf line flip) with numbers, not just visuals | Design tab only (rejected — misses the cross-layer verification value) |
| 2026-03-24 | Measurement tool UX: top toolbar "Measure" mode (Fusion 360 style) | Consistent across 2D/3D contexts; established UX pattern from industry-standard CAD tools | Panel-based (rejected — panels are tab-specific; toolbar is global) |
| 2026-03-24 | `window.__routr` on staging behind feature flag, not in production yet | Staging enables AI feature experimentation without production risk | Dev only (rejected — limits experimentation), Prod (rejected — premature) |
| 2026-03-24 | E2E validation records are pass/fail with comments; photos optional | Photos are documentation, not test mechanisms. Consistent physical photos are impractical. | Mandatory photos (rejected — too rigid, inconsistent results) |
| 2026-03-24 | AI interface as core architectural pattern for AI-designed apps | If AI designs, writes, tests, and will power features of the app — a first-class AI interaction layer is as fundamental as choosing state management. Should be considered for all CSDLC projects. | Treat as optional tooling (rejected — misses the architectural significance) |

---

## Known Issues / Tech Debt

| Issue | Severity | Notes |
|-------|----------|-------|
| Kerf line flip (table saw) | High | Blocks table saw fixture baseline. Standalone ticket. |
| No CI/CD test automation | Medium | Tests run locally only. G-code integration tests are ideal first CI candidate. |
| Roundover disabled | Low | No validation needed until feature is re-enabled. |

---

*This epic doc is refined collaboratively (Step 0) before stories are broken down (Step 1). Once refined, the AI Lead extracts context from this doc to craft sub-agent prompts (Step 2).*
*Update this doc as implementation reveals new information — design docs are living documents.*
