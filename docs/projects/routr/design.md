# Routr — Project Design Doc

*A retroactive design doc capturing the architecture, decisions, and lessons learned from building Routr.*
*Last updated: March 2026*

---

## Overview

### What Is This?

Routr is a browser-based CNC workshop planning app. Users design woodworking projects by defining boards, drawing shapes (cuts, pockets, drill holes, paths), applying edge treatments, and exporting G-code for their CNC machine. Think of it as the missing link between "I have a woodworking plan" and "my CNC is cutting it" — no CAD expertise required.

The app models a real woodworking shop: table saw for straight cuts, router for pockets and profiles, drill press for holes, planer for surfacing, band saw for freeform paths. Users think in terms of tools they already know, and Routr translates that into machine-ready G-code.

### Who Is It For?

Hobby and prosumer CNC owners — specifically desktop/benchtop machine users (Shapeoko, X-Carve, Altmill, Onefinity). These users know woodworking but aren't CAM experts. They want to go from a plan to cut parts without learning Fusion 360 or VCarve.

**Pain points:**
- CAM software has a steep learning curve
- Most tools are designed for machinists, not woodworkers
- Going from a PDF plan to G-code is a multi-tool, multi-hour process
- No tool thinks about the project as assembled furniture — they think about individual toolpaths

### Business Model

Freemium on export. The entire app is free to use — design, simulate, preview. No account required. G-code export requires a subscription. Users invest time designing their project, which creates natural conversion pressure at the export moment.

**Pricing tiers:**
- Free: full app access, 3 free exports
- Early Adopter: $12/mo or $99/yr
- Standard: $20/mo or $149/yr
- Lifetime: $299

Product-led growth — the app is the sales pitch. No marketing site needed beyond a landing page with a CTA to the app.

**Patent:** Provisional application #63/998,973 filed March 6, 2026 (Workshop Mode methodology).

---

## Product Vision

### The Two Modes

Routr has two product modes that represent different approaches to CNC woodworking:

**Assembly Mode** was the original vision — a full furniture design-to-manufacture pipeline. Design individual boards, connect them with joints (box, dovetail, dado, rabbet, miter), preview the assembled result in 3D, nest boards onto stock sheets, and simulate the CNC toolpaths. Five steps: Design → Assembly → 3D Preview → Nesting → Simulation.

The problem was combinatorial complexity. Every combination of board shapes × joint types × edge connections created an enormous surface area. After months of development, Assembly Mode could essentially make boxes. Useful, but far from the vision.

**Workshop Mode** was the strategic pivot for launch. Instead of modeling entire assemblies, focus on single boards and think in terms of woodworking tools — table saw, router, drill press, planer, band saw. This dramatically reduced complexity while delivering immediate value. Workshop Mode is the launch product; Assembly Mode is the long-term growth path.

This pivot is one of the most important decisions in the project's history. It traded ambition for shippability — and it worked.

### The Documentation Story

This project was built without design docs for the first several months. Code was the only source of truth. When CSDLC (Claymore Software Development Lifecycle) was formalized as a methodology, the first major application was creating retroactive design docs for Routr — reading the actual source code, documenting what exists, and surfacing inconsistencies that had accumulated without architectural documentation.

The docs you're reading now are the result. They serve as proof that design docs don't have to come first to be valuable — but they're dramatically more valuable when they do.

### Future Pipelines

Beyond the two existing modes, Routr's vision includes additional input pipelines:

- **PDF-to-Gcode** — AI-powered pipeline that reads woodworking plan PDFs, extracts dimensions and cut lists, and generates G-code. The "moonshot" feature.
- **CAD-to-Gcode** — Import CAD models and translate them into CNC operations.

### Patent Strategy

The woodworking CNC software space is remarkably under-patented. Vectric: zero patents. Carbide 3D: zero. Inventables: one (hardware only). This creates a significant first-mover advantage.

| Patent | Status | Application # | Summary |
|--------|--------|---------------|---------|
| Workshop Mode | **Filed** ✅ | #63/998,973 (March 6, 2026) | Operation-first paradigm — users think in woodworking tools, not CAM operations. 23 claims. Micro entity ($65). |
| PDF-to-Gcode Pipeline | Drafted | — | AI document understanding → cut list extraction → G-code generation. 30 claims drafted. |
| Integrated Browser Pipeline | Identified | — | End-to-end reactive propagation in browser. Rated ⭐⭐⭐⭐⭐ novelty. |
| Assembly Constraint Solver + Joint Geometry | Identified | — | Automated joint generation from edge connections with constraint solving. Rated ⭐⭐⭐⭐ novelty. |

**Not patentable:** Nesting (prior art since 1960s), heightmap simulation (expired 1990s patents).

---

## Tech Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | React 18 + TypeScript | Component model fits the multi-tab UI. TypeScript catches type errors in the complex data model — essential when shapes have type-specific params. |
| Build | Vite | Fast HMR, simple config. No webpack complexity needed for a single-page app. |
| State Management | Zustand | Minimal boilerplate, no provider wrapping, easy to persist. Single store keeps all project state co-located. |
| 3D Preview | Three.js via @react-three/fiber | React bindings for Three.js. Used for the 3D assembly preview and toolpath simulator. |
| Auth | Supabase Auth | Free tier handles 50K MAU. Email/password + Google OAuth (Google not yet configured). |
| Database | Supabase (Postgres) | User profiles, export tracking, promo codes. RLS policies enforce per-user data isolation. |
| Payments | Stripe | Checkout Sessions + Customer Portal. Webhook → Cloudflare Pages Function → Supabase for plan activation. |
| Deployment | Cloudflare Pages | Auto-deploys from main branch. Edge-cached globally. Pages Functions for serverless API (checkout, webhooks, portal). |
| Geometry | Clipper.js (clipper2-js) | Boolean geometry operations for SVG pocket detection and island subtraction. Handles contour offset pocketing. |

### Key Libraries & Dependencies

| Library | Purpose | Notes |
|---------|---------|-------|
| `@react-three/fiber` | React renderer for Three.js | 3D preview + simulator visualization |
| `@react-three/drei` | Three.js helpers (OrbitControls, etc.) | Camera controls, view cube |
| `clipper2-js` | Polygon boolean ops + offset | SVG pocket detection, island subtraction, contour pocketing |
| `zustand` | State management | Single global store (`useProjectStore`) |
| `vitest` | Test framework | 634+ tests, run with `npm test -- --run` |
| `jsdom` | DOM environment for tests | Required for component tests |

---

## System Architecture

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        UI Layer (React)                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │  Board    │ │ Nesting  │ │ Assembly │ │ Simulator / 3D    │  │
│  │  Setup    │ │ Canvas   │ │ Canvas   │ │ Preview           │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────────┬──────────┘  │
│       │             │            │                 │             │
│  ┌────┴─────────────┴────────────┴─────────────────┴──────────┐ │
│  │              Zustand Store (useProjectStore)                │ │
│  │  project: { boards, links, toolSettings, placements, ... } │ │
│  └────────────────────────────┬────────────────────────────────┘ │
└───────────────────────────────┼─────────────────────────────────┘
                                │
┌───────────────────────────────┼─────────────────────────────────┐
│                        Engine Layer                             │
│  ┌──────────┐ ┌──────────┐ ┌─┴────────┐ ┌───────────────────┐  │
│  │ Toolpath │ │ G-code   │ │ Assembly │ │ SVG Parser /      │  │
│  │ Generators│ │ Generator│ │ Solver   │ │ Import            │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │ Nesting  │ │ Tool     │ │ Simulator│ │ Edge Treatments   │  │
│  │ Engine   │ │ Library  │ │ Heightmap│ │                   │  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                │
┌───────────────────────────────┼─────────────────────────────────┐
│                     External Services                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ Supabase │ │ Stripe   │ │Cloudflare│                        │
│  │ Auth + DB│ │ Payments │ │ Pages    │                        │
│  └──────────┘ └──────────┘ └──────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### Layer Descriptions

**UI Layer** — React components organized by tab. Each tab has its own canvas and right panel. Components are presentational; business logic lives in the engine layer or store actions.

- **Board Setup** — Primary design surface. Users add boards, draw shapes (rectangles, circles, paths, holes, slots), set cut types, configure tools. This is where most time is spent.
- **Nesting** — Arranges boards onto stock sheets for material efficiency. Auto-nest algorithm + manual placement.
- **Assembly** — Visual edge-clicking interface for connecting boards with joints (box, dovetail, dado, rabbet, miter). Constraint solver detects over-constrained and unreachable boards.
- **Simulator / 3D Preview** — Three.js-powered toolpath visualization with playback controls. Heightmap-based material removal simulation. 3D assembly preview shows how boards fit together.

**Zustand Store** — Single global store (`useProjectStore`) holds all project state. Additional stores for auth (`useAuthStore`), theming (`useThemeStore`), and tool library (`useToolLibraryStore`). The project store is the source of truth — components read from it, engine functions receive data from it.

**Engine Layer** — Pure TypeScript functions with no React dependencies. This is where the real work happens:

- `engine/toolpath/` — Generates toolpath coordinates for each shape/operation type
- `engine/gcode/` — Converts toolpaths to G-code strings
- `engine/assembly/` — Constraint solver, edge validation, auto-link inference
- `engine/svg/` — SVG file parsing and import (path commands, bezier flattening, nesting detection)
- `engine/nesting/` — Auto-nest algorithm, placement validation
- `engine/tools/` — Tool library management, V-bit geometry calculations
- `engine/simulator/` — Heightmap-based material removal for visual simulation

**External Services** — Auth, database, and payments are all external. The app is a static SPA with no backend server — Cloudflare Pages Functions handle the serverless API needs (Stripe checkout sessions, webhooks, customer portal).

### Data Flow: Design to G-code

```
User draws shapes on board
        ↓
Shapes stored in Board.shapes[] (Zustand store)
        ↓
"Generate Toolpaths" triggers workshopOperations.ts
        ↓
Each shape → appropriate toolpath generator (straightCut, pocketCuts, drillCuts, etc.)
        ↓
Toolpath generators produce ToolpathOperation[] with coordinates
        ↓
Operations ordered: profiles → joints → pockets → drills → edge treatments
        ↓
Each operation → G-code generator (generator.ts)
        ↓
G-code blocks concatenated with tool change headers between operations
        ↓
Final G-code exported as .nc file
```

---

## Data Model

### Core Entities

The data model is centered on the `Project`, which contains everything needed to fully describe a woodworking project.

```
Project
├── boards: Board[]              # The physical boards being designed
│   ├── dimensions               # Width, height, thickness (all mm internally)
│   ├── shapes: Shape[]          # Cuts, pockets, holes, paths drawn on this board
│   ├── joints: Joint[]          # Joint definitions on board edges
│   └── edgeTreatments: EdgeTreatment[]  # Chamfers, roundovers, rabbets, dados on edges
├── links: BoardLink[]           # Assembly connections between boards (shared joint params)
├── toolSettings: ToolSettings   # Global default tool settings
├── stockSheets: StockSheet[]    # Raw material sheets for nesting
├── nestingSettings              # Spacing, margins, defaults for stock layout
├── tabSettings: TabSettings     # Holding tab configuration for profile cuts
└── placements: BoardPlacement[] # Where each board sits on which stock sheet
```

### Key Design Decisions in the Data Model

**Shapes are polymorphic via `params`.** All shapes share common fields (`id`, `type`, `cutType`, `position`, `depth`, `rotation`, `scale`) but have type-specific params (`RectParams`, `CircleParams`, `PathParams`, etc.). This means you **must check `shape.type` before accessing type-specific params** — a lesson learned from multiple bugs.

**Joints live on both boards AND links.** A `BoardLink` connects two boards at specific edges and holds shared `jointParams`. When a link is created, matching joints are generated on both boards. Modifying the link's params updates both boards' joints.

**All internal dimensions are millimeters.** Display conversion happens at the UI layer based on the `units` state (`metric` | `imperial`). Engine functions always receive and return mm.

**SVG imports use `svgGroupId`.** Multi-path SVG files import as multiple shapes that share a `svgGroupId`. They move together as a unit. The `isNested` and `isIsland` flags handle SVG pocket detection (filled shapes = pockets, white fills = islands to leave uncut).

### State Management

**Zustand with no middleware.** The store is a single flat object with action functions. No Redux-style reducers, no middleware, no devtools. This was a deliberate choice for simplicity — the data model is complex enough without adding state management complexity on top.

**No persistence layer.** Projects exist only in browser memory. No save/load, no cloud sync (yet). Export is the persistence mechanism — users export G-code and can re-create projects. This is a known limitation and future epic.

**Derived state is computed on the fly.** Toolpath operations, nesting validation, assembly solver results — all computed when needed rather than cached in the store. This avoids stale-state bugs at the cost of recomputation.

---

## Deployment & Infrastructure

### Environments

| Environment | URL | Purpose | Deploy Trigger |
|-------------|-----|---------|---------------|
| Local | `127.0.0.1:5173` | Development | `npm run dev` |
| Staging | `staging.routr.shop` | Pre-production, gated by Cloudflare Access | Auto-deploy on push to `main` |
| Production | `app.routr.shop` | Live (not yet launched) | Manual promotion from staging |

### CI/CD Pipeline

Cloudflare Pages auto-builds and deploys on every push to `main`. No separate CI — build failures block deployment. Tests run locally before push (enforced by process, not automation).

### Infrastructure Dependencies

| Service | Purpose | What Breaks If Down |
|---------|---------|-------------------|
| Cloudflare Pages | Hosting + CDN + serverless functions | Entire app unavailable |
| Supabase | Auth + user profiles + export tracking | Can't sign in, can't track exports, free tier stops working |
| Stripe | Payment processing | Can't upgrade, can't manage subscriptions |

### Serverless Functions (Cloudflare Pages Functions)

| Function | Purpose |
|----------|---------|
| `create-checkout-session` | Creates Stripe Checkout session for plan upgrade |
| `webhook` | Receives Stripe events, updates Supabase user plan |
| `portal` | Creates Stripe Customer Portal session for subscription management |

---

## Security Model

### Authentication & Authorization

- **Auth provider:** Supabase Auth (email/password, Google OAuth planned)
- **Auth is optional for the app.** Users can design freely without an account. Auth is required only for G-code export.
- **Export gating:** First 3 exports free (tracked in Supabase `user_profiles`), then paywall triggers.
- **Staging access:** Cloudflare Access restricts `staging.routr.shop` to an email allowlist.

### Data Protection

- **No sensitive user data stored in-browser.** Projects are ephemeral (browser memory only).
- **Supabase RLS policies** enforce per-user data isolation on `user_profiles`.
- **Stripe handles all payment data.** No card numbers touch our infrastructure.
- **API keys:** Stripe secret key and Supabase service key are Cloudflare environment variables, never exposed to the client.

### Known Attack Surfaces

- **Feedback endpoint:** Rate-limited to 3 bug reports + 3 feature requests per month per user (auth-gated).
- **Export counter:** Server-side enforcement via Supabase. Client-side checks are cosmetic only.
- **No server-side input validation on G-code:** The app generates G-code client-side. A malicious user could craft dangerous G-code by modifying client state — mitigated by the export disclaimer (user assumes responsibility for verifying G-code before running on their machine).

---

## Cross-Cutting Concerns

| Concern | Summary | Affected Epics | Dedicated Doc? |
|---------|---------|---------------|----------------|
| Coordinate Systems | Screen Y-down vs CNC Y-up vs Three.js scene coordinates. The `flipY()` transform in G-code export. Edge treatment top/bottom swap. | G-code pipeline, Simulator, Edge treatments, SVG import | Yes — [Coordinate Systems](epics/coordinate-systems.md) |
| Internal Units (mm) | All dimensions stored in mm. Display conversion at UI layer. `MM_PER_INCH = 25.4`. | All epics | N/A — simple convention |
| Tool Settings Resolution | Operations can use a tool from the library (`toolId`) or manual settings. Resolved at generation time. V-bit geometry has its own math (`vBitGeometry.ts`). | Workshop tools, Edge treatments, SVG engrave | N/A |
| Feature Flags | `ASSEMBLY_MODE_ENABLED`, roundover disabled via feature flag. Controls what users see in production. | Assembly, Edge treatments | N/A |

---

## Risks & Constraints

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Coordinate bugs in new features | High | High — produces wrong G-code, damages material or machine | Dedicated coordinate systems doc, integration tests for all edges |
| No project persistence | Certain | Medium — users lose work on browser close | Planned epic for save/load |
| Client-side G-code generation | Certain | Low — users could craft bad G-code | Export disclaimer, terms of use, user assumes responsibility |
| Supabase free tier limits | Low | Medium — 50K MAU, 500MB DB | Unlikely to hit soon; upgrade path available |

### Known Limitations

- **No save/load.** Projects exist only in browser memory.
- **No undo/redo** (keyboard shortcuts exist for some actions but no full undo stack).
- **Origin point is fixed.** Users cannot choose which corner of the board is the CNC origin — currently cuts assume a specific corner, which may be 180° opposite of the user's setup. This is a high-priority post-launch fix.
- **Roundover edge treatment disabled.** Feature-flagged off due to depth calculation complexity. Chamfer, rabbet, and dado work.
- **Assembly mode is feature-flagged off.** Works but not polished enough for launch.
- **No curved pocket toolpaths.** Router pocket for freeform (path) shapes doesn't follow curves correctly.

### Tech Debt

- **Dead components:** `AssemblyPanel.tsx` (old dropdown UI, replaced by `AssemblyCanvas.tsx`) still exists in code.
- **Operation reorder doesn't persist.** Dragging operations in the simulator panel resets on "Generate Toolpaths."
- **Edge treatment depth calculations** were patched multiple times. The current implementation works but the code path is convoluted (3-layer patch cycle during March 5-6 sprint).
- **No CI/CD tests.** Tests run locally only. Build failures are the only automated gate.

---

## Sub-system Index

Sub-systems are architectural boundaries or product-level capabilities. If you removed one, multiple unrelated features would break.

| Sub-system | Doc | Status | Summary |
|------------|-----|--------|---------|
| Workshop Mode | [workshop-mode.md](sub-systems/workshop-mode.md) | Shipped (launch product) | Operation-first paradigm — users think in woodworking tools |
| Assembly Mode | [assembly-mode.md](sub-systems/assembly-mode.md) | Shipped (feature-flagged off) | Multi-board furniture pipeline: Design → Assembly → 3D → Nesting → Sim |
| G-code Pipeline | [gcode-pipeline.md](sub-systems/gcode-pipeline.md) | Shipped | Shape → toolpath → operation ordering → G-code export |
| Coordinate Systems | [coordinate-systems.md](sub-systems/coordinate-systems.md) | Documented | Screen (Y-down), CNC (Y-up), Three.js — transforms and mapping |
| Simulator | [simulator.md](sub-systems/simulator.md) | Shipped | Toolpath visualization, heightmap material removal, playback |

## Epic Index

Epics are deliverable feature work built on top of sub-systems. Remove one and a specific capability disappears.

| Epic | Doc | Status | Parent Sub-system | Summary |
|------|-----|--------|-------------------|---------|
| Edge Treatments | [edge-treatments.md](epics/edge-treatments.md) | Shipped (roundover disabled) | G-code Pipeline, Coordinate Systems | Chamfer, roundover, rabbet, dado on board edges |
| SVG Import & Engrave | [svg-import.md](epics/svg-import.md) | Shipped | G-code Pipeline | SVG parsing, pocket detection, island subtraction, engrave |
| Nesting | [nesting.md](epics/nesting.md) | Shipped | Assembly Mode | Multi-sheet stock layout, auto-nest algorithm |
| Assembly & Joints | [assembly-joints.md](epics/assembly-joints.md) | Shipped (flagged off) | Assembly Mode | Board linking, constraint solver, joint geometry |
| Auth & Payments | — | Shipped | — | Supabase auth, Stripe checkout, export gating |
| Theming | — | Shipped | — | Dark/light theme system |

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Early 2026 | Zustand over Redux | Minimal boilerplate, single flat store fits the data model. No middleware needed. | Redux (too much ceremony), Jotai (atomic model doesn't fit project-as-single-entity), Context API (performance concerns with frequent updates) |
| Early 2026 | Client-side G-code generation | No server needed, works offline, simpler architecture. Users own their G-code. | Server-side generation (adds latency, requires backend, no offline) |
| Early 2026 | Freemium on export | Users invest time designing → high conversion at paywall. Full app access free builds trust and word-of-mouth. | Free trial (time pressure kills exploration), pay upfront (kills adoption), ad-supported (wrong audience) |
| Feb 2026 | Cloudflare Pages over Vercel/Netlify | Edge caching, Pages Functions for serverless, DNS already on Cloudflare, generous free tier. | Vercel (good but vendor lock-in concerns), Netlify (fewer edge features), GitHub Pages (no serverless functions) |
| Feb 2026 | Supabase over Firebase | Postgres + RLS is more flexible than Firestore. Free tier is generous. SQL is a known quantity. | Firebase (NoSQL doesn't fit relational data model), Auth0 (overkill for auth-only needs), custom backend (unnecessary complexity) |
| Mar 2026 | No project persistence for launch | Ship faster. Users are making one-off projects initially. Save/load adds significant complexity (versioning, migration, storage). | LocalStorage (fragile, size limits), Supabase storage (adds backend complexity before validating product-market fit) |
| Mar 2026 | All internal units in mm | Single unit system eliminates conversion bugs in the engine. Display conversion is a UI concern only. | Store in user's preferred unit (conversion bugs everywhere), dual storage (complexity) |
| Mar 2026 | Feature-flag assembly mode | Works but UX isn't polished. Don't ship half-baked. | Ship as-is (bad first impression), cut entirely (waste of work) |

---

## Glossary

| Term | Definition |
|------|-----------|
| **Workshop Mode** | The primary design paradigm — users think in terms of woodworking tools (table saw, router, drill press) rather than CAM operations (profile, pocket, drill). |
| **Board** | A physical piece of wood being designed. Has dimensions, shapes, joints, and edge treatments. |
| **Shape** | A cut, pocket, hole, path, or surfacing operation drawn on a board. Polymorphic via `type` + `params`. |
| **Edge Treatment** | A decorative or functional modification to a board edge (chamfer, roundover, rabbet, dado). |
| **Stock Sheet** | Raw material that boards are nested onto for cutting. |
| **Nesting** | Arranging boards on stock sheets to minimize material waste. |
| **Placement** | A board's position and rotation on a specific stock sheet. |
| **Operation** | A toolpath operation in the simulator (profile-cut, pocket, drill, etc.). Each has its own G-code block. |
| **flipY** | The coordinate transform applied during G-code export to convert screen coordinates (Y-down) to CNC coordinates (Y-up). |
| **Tab** | A small bridge of material left uncut during profile operations to prevent the part from shifting. Defined by width, height, and spacing. |
| **BitOffset** | Whether the tool path runs to the left, center, or right of the drawn line (`'left' \| 'center' \| 'right'`). |
| **V-bit** | A conical cutting tool used for chamfers and engraving. Requires special depth/width geometry calculations (`vBitGeometry.ts`). |
| **Island** | In SVG import, a white-filled closed path inside a pocket region. Left uncut (material remains). |
| **Lightning Strike** | A CSDLC session where everything fires on all cylinders — tight refinement leads to rapid, clean execution. Multiple features ship in one session. |

---

*This design doc is the source of truth for Routr's project-level architecture. Epic-level details live in `epics/`. Implementation specifics live in the project repo's `WORKFLOW.md`.*
*Update this doc when architecture changes — stale docs are worse than no docs.*
