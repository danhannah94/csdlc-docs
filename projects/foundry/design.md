# Foundry — Project Design Doc

*Status: Step 0 — Decisions Locked, E1 Ready for Story Breakdown*
*Created: March 29, 2026*
*Updated: March 30, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This?

Foundry is a documentation platform built for human-AI collaborative workflows. It serves the same markdown docs to two audiences: humans get a rich interactive UI with inline comments, audio playback, and collaboration features. AI agents get the same functionality via MCP tools — the API is the product, the UI is one client.

Think MkDocs Material meets Google Docs meets podcast player — but purpose-built for teams working with AI agents.

### Why This Exists

Current doc platforms (MkDocs, Docusaurus, GitBook) solve one problem well: making docs readable for humans. But in a CSDLC workflow where an AI agent is a first-class collaborator, there are gaps:

- **No inline collaboration** — you can't highlight a paragraph and ask your AI "why did we decide this?"
- **No audio** — docs are text-only. Can't listen while driving, in the shop, or doing CNC work.
- **No agent awareness** — the platform doesn't know an AI exists. No optimized serving, no contextual chat.
- **No decision tracking** — you know WHAT the doc says but not WHY it says it.
- **No MCP interface** — AI agents can't programmatically read, search, or annotate docs through a standard protocol.

Foundry fills these gaps — for us first, maybe others later.

### The Killer Features

| Feature | What It Does | Why It Matters |
|---------|-------------|----------------|
| **Contextual Chat** | Highlight text → chat panel opens with full doc context → ask questions, get answers, suggest edits | Eliminates copy-paste workflows. The AI knows what you're reading. |
| **Audio Playback** | TTS on any section or full doc. Play/pause, speed control. | Review docs while driving, in the shop, on a walk. Accessibility win. |
| **Inline Annotations** | Highlight + comment. Private or shared. Linked to doc sections, not line numbers. | Comments survive doc edits. Conversations attached to context. |
| **Batch Comment Submission** | Add multiple comments across a doc, then submit all at once to OpenClaw main session | Batched review = one structured message instead of 10 interruptions. Preserves conversational context for refinement. |
| **Reply Annotations** | Clay's responses appear as threaded replies under your comments in Foundry | Closes the feedback loop. Accept/reject/reply without leaving the doc. |
| **Suggest Edit** | "Suggest Edit" button in chat → proposes inline doc change via PR | Every conversation can improve the docs. Knowledge compounds. |
| **MCP-First API** | Every feature available as an MCP tool — search, annotate, suggest edits | AI agents are first-class clients, not afterthoughts. The differentiator vs every other doc platform. |

### How It Fits in the Claymore Ecosystem

```
🔨 Anvil    — Semantic search & retrieval engine (the brain)
🏭 Foundry  — Doc platform & collaboration UI (the interface)
🔥 Forge    — MCP monetization layer (future)
```

Foundry consumes Anvil as its search backend (v0.2+). Same index, same embeddings — Foundry just adds the UI and collaboration layer on top.

---

## Architecture

### High-Level Architecture Evolution

```
v0.1 — Static Only
┌─────────────────────────┐
│   GitHub Pages           │
│   Astro static site      │
│   - renders docs         │
│   - nav from nav.yaml    │
│   - client-side search   │
│     (pre-built JSON)     │
└─────────────────────────┘

v0.2 — Static Site + API Server
┌─────────────────────┐   HTTPS   ┌──────────────────────┐
│  GitHub Pages        │ ────────▶ │  API Server          │
│  Astro static site   │ ◀──────── │  - Anvil search      │
│  + annotation UI     │           │  - SQLite annotations│
│  + batch submit      │           │  - batch submit      │
└─────────────────────┘           └──────────────────────┘

v0.3 — MCP-First API (Two Clients)
┌─────────────────────┐   HTTPS   ┌──────────────────────┐
│  GitHub Pages        │ ────────▶ │  API Server (MCP)    │
│  Astro static site   │ ◀──────── │  - Anvil search      │
│  + chat panel        │           │  - Annotations CRUD  │
│  + reply viewer      │           │  - Suggest edit (GH) │
│  + suggest edit UI   │           │  - Chat relay        │
└─────────────────────┘           └──────────────────────┘
                                          ▲
                                          │ MCP protocol
                                   ┌──────┴───────┐
                                   │ OpenClaw /    │
                                   │ AI Agents     │
                                   └──────────────┘
```

The static site never goes away. It gains a backend friend. The API serves both the web UI and AI agents through the same endpoints.

### Tech Stack (LOCKED)

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Static site framework** | **Astro** + React islands | Content-first architecture. Ships near-zero JS by default, interactive components added as React islands where needed. |
| **Search (MVP)** | **Pre-built JSON index** (client-side) | Works on pure static hosting. No server required. |
| **Search (v0.2+)** | **Anvil MCP server** via API | Full semantic search when API server is available. |
| **Annotations (v0.2)** | **SQLite** (local to API server) | Zero infrastructure, fast iteration on schema. |
| **Annotations (v0.3+)** | **Supabase** migration | When we need sync, multi-user, shared annotations. |
| **TTS** | Edge TTS or ElevenLabs | On-demand generation, aggressive caching. |
| **Chat integration** | OpenClaw main session | Batch comments sent as structured payload. Preserves refinement context. |
| **Hosting (static)** | **GitHub Pages** | Free, native auth via repo visibility, works from any device. |
| **Hosting (API, v0.2+)** | **TBD** (Cloudflare Workers, Fly.io, etc.) | Decided when we need it. Infrastructure is the easiest thing to swap. |
| **Source** | Git-backed markdown, **multi-repo** | `foundry.config.yaml` lists source repos. Build script clones + copies. |

### Project Structure (Monorepo)

```
foundry/
├── packages/
│   ├── site/               # Astro static site (v0.1)
│   │   ├── astro.config.mjs
│   │   ├── src/
│   │   │   ├── layouts/
│   │   │   │   └── DocLayout.astro
│   │   │   ├── components/
│   │   │   │   ├── Sidebar.astro
│   │   │   │   ├── Breadcrumbs.astro
│   │   │   │   ├── ThemeToggle.tsx    # React island
│   │   │   │   └── SearchBar.tsx      # React island
│   │   │   ├── pages/
│   │   │   │   └── [...slug].astro    # dynamic route
│   │   │   └── styles/
│   │   │       └── global.css
│   │   ├── public/
│   │   └── package.json
│   └── api/                # API server (v0.2+, placeholder for now)
│       └── package.json
├── foundry.config.yaml     # source repos, access rules, nav
├── nav.yaml                # navigation tree
├── scripts/
│   └── build.sh            # clones source repos, copies docs, builds site
├── package.json            # monorepo root (npm workspaces)
└── README.md
```

### Key Design Decisions

**Multi-repo content sourcing.** Foundry pulls markdown from one or more git repos at build time. Configured in `foundry.config.yaml`:

```yaml
sources:
  - repo: danhannah94/csdlc-docs
    branch: main
    paths: ["methodology/", "about/"]
    access: public
  - repo: danhannah94/csdlc-private
    branch: main
    paths: ["projects/"]
    access: private
```

Build script clones each repo, copies specified paths, and Astro renders them into one site. For MVP, we deploy two GitHub Pages sites (one public, one with all docs including private — restricted by repo visibility).

**Public/private access via GitHub's native auth.** No Cloudflare Access, no custom auth, no passwords. Public docs in a public repo → anyone can read. Private docs deployed from a private repo → restricted to repo collaborators. GitHub Pro ($4/mo) or Enterprise required for private repo Pages. Dan has Pro personally, enterprise license in progress at work.

**Navigation from `nav.yaml`.** Curated, explicit ordering and titles. Parsed at build time to generate sidebar. Single source of truth for doc hierarchy. Future-proofed for per-doc metadata flags.

**Annotations stored separately from markdown.** SQLite DB (v0.2), linked by doc path + content hash. Source markdown stays clean. Schema includes `parent_id` from day one for threaded replies. Comments survive doc reorganization.

**Foundry API = MCP server (v0.3+).** The API exposes every feature as both REST endpoints (for the web UI) and MCP tools (for AI agents). Same logic, two protocols. This is the key differentiator — AI agents are first-class clients.

| MCP Tool | What It Does | Version |
|----------|-------------|---------|
| `search_docs` | Anvil semantic search | v0.2 |
| `list_annotations` | Get annotations for a doc/section | v0.2 |
| `create_annotation` | Add a comment programmatically | v0.2 |
| `submit_review` | Batch submit annotations to OpenClaw | v0.2 |
| `suggest_edit` | Propose a doc change (creates GitHub PR) | v0.3 |

**Single container deployment (v0.2.1+).** Foundry ships as a single Docker container: Express serves both the static Astro build and the API/MCP server on one port. SQLite persists via volume mount. Same image runs locally (`docker compose up`), at work (existing Docker setup), or in the cloud (Fly.io free tier). GitHub Pages remains as the public static-only version; the container is the full-featured version with annotations.

**Infrastructure is earned complexity.** v0.1 shipped on GitHub Pages with zero infrastructure. v0.2.1 earned a Docker container because annotations require a server. The container IS the product — `docker run -p 3001:3001 foundry` gives you everything. Cloud hosting (Fly.io) was chosen for free tier + zero config. AWS/Terraform are reserved for when enterprise demand justifies them.

---

## Scope & Epics

### v0.1 — "Read, Search, Access" (MVP)

| Epic | Scope |
|------|-------|
| **E1: Scaffold + Static Site + Deploy** | Astro monorepo, markdown rendering (tables, code, mermaid, admonitions), nav from `nav.yaml`, dark mode, pre-built search index, GitHub Pages deploy, multi-repo content sourcing, public/private via repo visibility |
| **E2: API Server + Agent MCP Foundation** | Express API server, Anvil as imported library (not MCP sidecar), MCP tools for agents (search + annotation schemas), doc registry endpoints. Human search bar deferred to E3+. |

**MVP test:** Can Dan read CSDLC docs from his iPad at work with public/private access working?

**Note:** E2 was reframed during refinement from "human search bar" to "API + agent interface." The agent MCP foundation enables E3 (TTS) and E4 (Annotations). Anvil's library refactor (export core functions as importable API) and OpenAI embedding upgrade are prerequisites.

### v0.2 — "Listen and Comment"

| Epic | Scope |
|------|-------|
| **E3: TTS Playback** | Native browser Web Speech API — play buttons per section, Play All, speed controls. Pure frontend, no API dependency. |
| **E4: Annotations + Unified Review Thread** | Highlight → comment → unified doc thread → batch submit to OpenClaw → AI replies threaded back. Full annotation lifecycle with orphan handling. |
| **API Server** | Established in E2, extended in E4 for annotation CRUD + review batches |

### v0.2.1 — "Ship It"

| Epic | Scope |
|------|-------|
| **E4.5: Containerize + Deploy** | Single Docker container (site + API + MCP on one port), multi-stage Dockerfile, docker-compose for local dev, Fly.io deployment with GitHub Actions CD. |
| **E5: Authentication + Write Protection** | Bearer token middleware on write endpoints, frontend token entry, MCP token config. Gate before production deployment. |

### v0.3+ — "Close the Loop" (MCP-First)

| Feature | Description |
|---------|-------------|
| **Suggest Edit** | Clay proposes inline doc changes via GitHub PR |
| **Human doc editing** | Edit docs from the Foundry UI — push changes to source repo |
| **Decision log indexing** | Annotations tagged as "decision" get indexed by Anvil |
| **Supabase migration** | Sync annotations across devices, multi-user sync |
| **GitHub OAuth** | Multi-user identity, `user_id` = GitHub username, replaces manual token for browser users |
| **CSDLC templates** | New epic, new ticket, new design doc scaffolds |
| **Doc diff / change playback** | "What changed since last Tuesday?" |
| **Foundry OpenClaw Skill** | Skill teaching Clay how to process review batches and reply via MCP |

---

## E4: Annotations + Unified Review Thread — Design Doc

*Status: Step 0 Complete, Step 1 Complete — Ready for Step 2 (Agent Prompt Crafting)*
*Refined: March 30, 2026*

### Vision

E4 turns Foundry into a doc review platform. The core workflow: read a doc, highlight text, add comments, submit them all as a batch to Clay's main session, get threaded replies back. Think MS Word track changes meets MkDocs meets AI agent — the review conversation is doc-scoped but connected to the main chat.

This is the feature that accelerates CSDLC refinement. Currently, Dan reads docs → switches to Telegram → types feedback referencing specific sections → context gets lost in chat history. E4 eliminates that friction: comments are anchored to the doc, batched for focused review, and replies thread back to where they belong.

### Architecture

```
┌─────────────────────────────────────────────────┐
│  Foundry Static Site (GitHub Pages)             │
│                                                 │
│  ┌──────────────┐  ┌────────────────────────┐   │
│  │  Doc Content  │  │  Unified Thread Panel  │   │
│  │  (~65% width) │  │  (~35% width)          │   │
│  │               │  │                        │   │
│  │  [highlighted │──│→ 💬 Draft comment      │   │
│  │   text]       │  │  💬 Draft comment      │   │
│  │               │  │  ── Submitted ──       │   │
│  │  [section     │──│→ 💬 Dan's comment      │   │
│  │   indicator]  │  │    ↳ 🤖 Clay's reply   │   │
│  │               │  │    ✅ Resolved          │   │
│  │               │  │                        │   │
│  │               │  │  [Submit Review]       │   │
│  └──────────────┘  └──────────┬─────────────┘   │
└──────────────────────────────┬───────────────────┘
                               │
                    localStorage (drafts)
                               │
                    ┌──────────▼─────────────┐
                    │  Foundry API Server     │
                    │  POST /reviews          │
                    │  POST /annotations      │
                    │  GET  /annotations      │
                    │  PATCH /annotations/:id │
                    │                         │
                    │  MCP Tools:             │
                    │  - list_annotations     │
                    │  - create_annotation    │
                    │  - resolve_annotation   │
                    │  - submit_review        │
                    └──────────┬─────────────┘
                               │
                    ┌──────────▼─────────────┐
                    │  SQLite (annotations +  │
                    │  reviews tables)        │
                    └──────────┬─────────────┘
                               │
              ┌────────────────▼───────────────┐
              │  OpenClaw Main Session          │
              │  (via MCP submit_review)        │
              │                                 │
              │  Clay processes batch →          │
              │  replies via create_annotation  │
              │  with parent_id                 │
              └─────────────────────────────────┘
```

### Portability Layers (Not OpenClaw-Locked)

The architecture separates into three layers for future portability:

- **Layer 1 (standalone):** Foundry with annotations, drafts, unified thread. Works for anyone as a doc review platform. No AI required.
- **Layer 2 (any AI):** Batch submit produces structured JSON. The destination is configurable — webhook, API call, email. Not OpenClaw-specific.
- **Layer 3 (OpenClaw-specific):** MCP reply path, session awareness, Telegram notifications. Our secret sauce, but the product doesn't depend on it.

Design principle: **Foundry's core never imports or depends on OpenClaw.** The integration is a plugin/adapter pattern.

### Data Model

#### Updated `annotations` Table

```sql
CREATE TABLE annotations (
  id TEXT PRIMARY KEY,           -- cuid2
  doc_path TEXT NOT NULL,        -- e.g., "methodology/process"
  section TEXT NOT NULL,          -- heading path: "## Architecture > ### Tech Stack"
  content TEXT NOT NULL,          -- comment text (markdown)
  content_hash TEXT,             -- hash of section content at annotation time (drift detection)
  parent_id TEXT,                -- foreign key to annotations.id (threading)
  review_id TEXT,                -- foreign key to reviews.id (batch grouping)
  user_id TEXT DEFAULT 'dan',    -- nullable for MVP, required for multi-user
  author_type TEXT NOT NULL DEFAULT 'human',  -- 'human' | 'ai'
  status TEXT NOT NULL DEFAULT 'draft',       -- draft | submitted | replied | resolved | orphaned
  created_at TEXT NOT NULL,      -- ISO 8601
  updated_at TEXT NOT NULL,      -- ISO 8601
  FOREIGN KEY (parent_id) REFERENCES annotations(id),
  FOREIGN KEY (review_id) REFERENCES reviews(id)
);
```

#### New `reviews` Table

```sql
CREATE TABLE reviews (
  id TEXT PRIMARY KEY,            -- cuid2
  doc_path TEXT NOT NULL,
  user_id TEXT DEFAULT 'dan',
  status TEXT NOT NULL DEFAULT 'draft',  -- draft | submitted | complete
  submitted_at TEXT,              -- ISO 8601, null until submitted
  completed_at TEXT,              -- ISO 8601, null until all comments resolved
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

#### Schema Design Notes

- **cuid2 IDs** from day one — maps 1:1 to Supabase (Postgres) with zero migration friction
- **Standard column types** (TEXT, not SQLite-specific) for clean Supabase migration
- **`user_id` nullable** for MVP (hardcoded 'dan'), required column for future multi-user
- **`author_type`** distinguishes human comments from AI replies in the thread UI
- **Migration path:** One script dumps SQLite → inserts into Supabase. Schema is identical.

### Comment Lifecycle

```
draft → submitted → replied → resolved
                         ↘ orphaned (if section removed/renamed)
```

- **Draft:** Written in Foundry, stored in localStorage. Not yet in SQLite.
- **Submitted:** Part of a review batch. Persisted to SQLite. Sent to OpenClaw main session via MCP `submit_review`.
- **Replied:** Clay has responded (reply annotation with `parent_id` exists).
- **Resolved:** Human has accepted/closed the comment thread.
- **Orphaned:** Section heading path no longer exists after doc rebuild. Annotation preserved with original context, shown in collapsed "Orphaned Comments" section. User can re-anchor or dismiss.

### Annotation Anchoring

**Primary anchor:** Heading path (e.g., `## Architecture > ### Tech Stack`). Human-readable, survives content edits within the same section.

**Drift detector:** Content hash (SHA-256 of section text at annotation time). On doc rebuild:
- Heading path matches + hash matches → ✅ anchored, no drift
- Heading path matches + hash changed → ⚠️ anchored, content drifted (subtle indicator in thread)
- Heading path gone → ❌ orphaned (collapsed section, original context preserved)

Nothing is silently deleted. Orphaned comments show the original heading path and quoted text so the user can re-anchor or dismiss.

### UX: Unified Thread Panel (Option C)

**Layout:** Doc takes ~65% width (left), unified thread panel takes ~35% (right).

**Thread panel contents (chronological):**
- Draft comments (dimmed, "💬" icon, anchored text quoted)
- Submitted comments (solid, grouped by review batch)
- AI replies (threaded under parent comment, different background/avatar)
- Resolved comments (collapsed with ✅, expandable)
- Orphaned comments (collapsed section at bottom)

**Highlight-to-comment flow:**
1. User selects text in a doc section
2. Small floating "💬 Comment" button appears near selection
3. Click → thread panel scrolls to new draft, selected text quoted
4. User types comment, hits Enter
5. Comment appears as draft in thread panel
6. Repeat across the doc
7. "Submit Review" button at bottom collects all drafts → sends batch

**Cross-references:**
- Each comment in the thread has a "📍 jump to section" link → scrolls doc + highlights section
- Section margin indicators in the doc → click to scroll thread to that comment
- Bidirectional linking between doc and thread

**Submit Review flow:**
1. Drafts promoted to `submitted` status
2. Review record created in `reviews` table
3. All annotations persisted to SQLite with `review_id`
4. MCP `submit_review` tool called → structured payload arrives in Clay's main session
5. Clay processes and replies to individual comments via `create_annotation` (with `parent_id`)
6. Replies appear in thread on next page load/refresh

### API Endpoints (E4 additions)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/annotations?doc_path=X` | List annotations for a doc (with optional section filter) |
| `POST` | `/annotations` | Create annotation (used by both UI submit and AI reply) |
| `PATCH` | `/annotations/:id` | Update status (resolve, orphan, re-anchor) |
| `POST` | `/reviews` | Create review batch, submit to OpenClaw |
| `GET` | `/reviews?doc_path=X` | List review history for a doc |

### MCP Tools (E4 implementation of E2 stubs)

| Tool | Params | Description |
|------|--------|-------------|
| `list_annotations` | `doc_path`, `section?`, `status?` | Query annotations with filters |
| `create_annotation` | `doc_path`, `section`, `content`, `parent_id?`, `author_type?` | Create comment or reply |
| `resolve_annotation` | `annotation_id` | Mark as resolved |
| `submit_review` | `doc_path`, `annotation_ids?` | Package and submit review batch |

### Decisions Log

| # | Question | Decision | Date |
|---|----------|----------|------|
| 1 | Batch submit integration pattern | MCP `submit_review` tool. Configurable endpoint — not OpenClaw-locked. | Mar 30 |
| 2 | Annotation anchoring | Heading path (primary) + content hash (drift detect) + orphan handling (collapsed section) | Mar 30 |
| 3 | SQLite → Supabase migration | cuid2 UUIDs, standard types, nullable `user_id`, one-script migration | Mar 30 |
| 4 | Highlight-to-comment UX | Option C: unified thread panel (~35% right), doc-scoped (NOT main chat) | Mar 30 |
| 5 | Reply path (Clay → Foundry) | MCP `create_annotation` with `parent_id`. Natural Telegram notification. | Mar 30 |
| 6 | Auth/identity | Hardcoded `user_id='dan'` MVP. `author_type: human\|ai`. GitHub OAuth v0.3+. | Mar 30 |
| 7 | Draft storage | localStorage (browser) until Submit Review | Mar 30 |
| 8 | Review history | `reviews` table with batch grouping, collapsible past reviews in thread | Mar 30 |
| 9 | Orphaned comment styling | Collapsed section for MVP, iterate styling (strikethrough/dim) after seeing real data | Mar 30 |
| 10 | Foundry OpenClaw skill | Fast-follow after E4 ships, NOT in E4 scope | Mar 30 |

### E4 Story Breakdown

#### S1: SQLite Schema + Annotation CRUD API
**Batch:** 1 (foundation — all other stories depend on this)

**Scope:**
- Create `annotations` and `reviews` tables with full schema (see Data Model above)
- Implement REST endpoints: `GET /annotations`, `POST /annotations`, `PATCH /annotations/:id`
- Implement `GET /reviews`, `POST /reviews`
- Wire cuid2 for ID generation
- Content hash generation utility (SHA-256 of section text)
- Basic test suite for all endpoints

**Acceptance Criteria:**
- [ ] Both tables created with all columns from schema above
- [ ] POST creates annotation with correct defaults (`status: 'draft'`, `author_type: 'human'`)
- [ ] GET filters by `doc_path` and optionally `section`, `status`
- [ ] PATCH updates `status` and `updated_at`
- [ ] POST /reviews creates review record
- [ ] cuid2 IDs on all records
- [ ] Tests for CRUD operations + edge cases (missing fields, invalid IDs)

**Boundaries:**
- No UI work. API only.
- No MCP tool implementation (that's S5).
- No orphan detection logic (that's S4).

---

#### S2: Unified Thread Panel — Comment Display + Navigation
**Batch:** 2 (parallel with S3, depends on S1)

**Scope:**
- React island: `AnnotationThread.tsx` — right sidebar panel (~35% width)
- Fetches annotations from API for current doc
- Displays comments grouped by review batch, threaded (parent/child)
- Status styling: draft (dimmed), submitted (solid), replied (threaded with AI avatar), resolved (collapsed ✅)
- "📍 Jump to section" links scroll doc to anchored heading
- Section margin indicators in doc that scroll thread to relevant comment
- Collapsible past reviews ("Review #2 — 4 comments, all resolved")
- Responsive: panel collapses to toggle button on mobile widths

**Acceptance Criteria:**
- [ ] Thread panel renders on doc pages with correct layout (~65/35 split)
- [ ] Comments display with correct status styling
- [ ] Threaded replies render under parent comments
- [ ] Jump-to-section scrolls doc and highlights target heading
- [ ] Margin indicators in doc link to thread comments
- [ ] Past reviews collapsible
- [ ] Panel has toggle button to show/hide
- [ ] Mobile-responsive collapse

**Boundaries:**
- No comment creation UI (that's S3).
- No Submit Review button (that's S4).
- Read-only display of existing annotations.

---

#### S3: Highlight-to-Comment — Draft Creation UX
**Batch:** 2 (parallel with S2, depends on S1)

**Scope:**
- Text selection listener on doc content area
- Floating "💬 Comment" button appears near selection
- Click → new draft in thread panel with selected text quoted
- Comment input (textarea + submit)
- Drafts stored in localStorage keyed by doc path
- Draft comments appear in thread panel as dimmed entries
- Can delete/edit drafts before submission
- Section detection: determine which heading path the selection falls under

**Acceptance Criteria:**
- [ ] Text selection in doc shows floating comment button
- [ ] Clicking button creates draft with quoted text and correct section path
- [ ] Draft persists in localStorage (survives page refresh)
- [ ] Draft appears in thread panel with dimmed styling
- [ ] Can edit draft content before submission
- [ ] Can delete drafts
- [ ] Section heading path correctly detected from selection position
- [ ] Multiple drafts across different sections supported

**Boundaries:**
- No submission to API (that's S4).
- No interaction with SQLite — drafts are localStorage only.
- No AI reply display (that's S2).

---

#### S4: Submit Review + Orphan Detection
**Batch:** 3 (depends on S1 + S2 + S3)

**Scope:**
- "Submit Review" button in thread panel
- On submit: creates review record via `POST /reviews`, persists all drafts as annotations via `POST /annotations`, clears localStorage drafts
- Orphan detection on page load: compare annotation heading paths against current doc headings
- Orphaned annotations flagged with `status: 'orphaned'`, shown in collapsed section
- Content hash drift detection: heading matches but content changed → subtle ⚠️ indicator
- Review batch status tracking: draft → submitted → complete (when all comments resolved)

**Acceptance Criteria:**
- [ ] Submit Review button visible when drafts exist
- [ ] Click creates review + persists all draft annotations to SQLite
- [ ] localStorage drafts cleared after successful submission
- [ ] Orphan detection runs on page load
- [ ] Orphaned annotations display in collapsed "Orphaned Comments" section
- [ ] Content drift shows ⚠️ indicator on affected comments
- [ ] Review status updates to 'complete' when all child annotations resolved
- [ ] Error handling: API failure shows user-friendly message, drafts preserved

**Boundaries:**
- No MCP submission to OpenClaw (that's S5).
- No re-anchoring UI for orphaned comments (v0.3+ polish).
- Orphan styling is collapsed-only for MVP (strikethrough/dim deferred).

---

#### S5: MCP Tool Implementation + OpenClaw Integration
**Batch:** 4 (depends on S1 + S4)

**Scope:**
- Implement all 4 MCP annotation tools (replace E2 stubs):
  - `list_annotations` — query with filters
  - `create_annotation` — create comment or threaded reply (AI sets `author_type: 'ai'`)
  - `resolve_annotation` — mark as resolved
  - `submit_review` — trigger batch submission, format structured payload for OpenClaw
- `submit_review` MCP tool sends structured JSON to configurable endpoint (OpenClaw main session for us)
- Payload format: doc path, review ID, all annotations with section paths, quoted text, content
- Test suite for all MCP tool handlers

**Acceptance Criteria:**
- [ ] All 4 MCP tools respond with real data (not "not implemented")
- [ ] `create_annotation` with `parent_id` creates threaded reply
- [ ] `create_annotation` with `author_type: 'ai'` correctly stores AI replies
- [ ] `submit_review` produces well-structured JSON payload
- [ ] `submit_review` calls configurable endpoint (env var or config)
- [ ] `list_annotations` supports `doc_path`, `section`, `status` filters
- [ ] `resolve_annotation` updates status + `updated_at`
- [ ] Tests for all tool handlers
- [ ] E2 stub code removed

**Boundaries:**
- No UI changes. MCP/API only.
- No Foundry OpenClaw skill (fast-follow).
- Endpoint configuration is simple (env var or config file, not a plugin system).

### E4 Execution Plan

```
Batch 1: S1 (schema + CRUD API) — foundation, must ship first
Batch 2: S2 + S3 (thread panel + highlight UX) — parallel, both depend on S1
Batch 3: S4 (submit + orphans) — depends on S2 + S3
Batch 4: S5 (MCP implementation) — depends on S1 + S4
```

**Estimated execution:** 5 stories across 4 batches. Batches 1, 3, 4 are sequential (single agent). Batch 2 is a Lightning Strike (2 parallel agents, same repo — git worktrees).

**Fast-follow (not in E4):**
- Foundry OpenClaw skill (teaches Clay how to process review batches)
- Re-anchor UI for orphaned comments
- Orphan visual polish (strikethrough/dim after seeing real data)
- Real-time reply push via SSE (currently refresh-to-see)

---

## MCP Architecture

### Layer 1: Development MCP (How We Build Foundry)

Sub-agents use standard CSDLC workflow. Anvil available for searching CSDLC docs when implementing features. This isn't Foundry-specific — it's the normal development process.

### Layer 2: Runtime MCP (What Foundry Exposes)

Starting in v0.2, Foundry's API server exposes features as MCP tools. The web UI and AI agents consume the same API. This makes Foundry **fully drivable by agentic AI** — the differentiator vs MkDocs, Docusaurus, and every other doc platform.

---

## Future Directions

### Non-Developer Users

Dan's mom (published author) expressed interest — her books are all in Word documents. This validates the core concept beyond developer docs. The pipeline:

1. **Ingest**: Word doc → markdown (mammoth.js or similar)
2. **Index**: Anvil chunks and embeds
3. **Serve**: Foundry renders like any other doc

Requires format adapters (v0.3+) and simplified onboarding (drag-and-drop upload vs git push).

### npm Package Extraction

If Foundry proves useful beyond our workflow, extract to `@claymore-dev/foundry`. The monorepo structure and config-driven architecture make this extraction clean. Same pattern as Anvil — build for us, package for others.

### Productization Path

The path from "works for us" to "works for others" is packaging, not rewriting:
1. GitHub Pages works ✅
2. API server added when v0.2 needs it ✅
3. Validate demand (someone actually wants this)
4. Dockerize (`Dockerfile` + `docker-compose.yml`) — one afternoon
5. Terraform if repeatable deploys needed — one session
6. Managed hosting (Amplify, Cloud Run) — swap deploy target

Infrastructure decisions are earned through product validation.

---

## Open Questions (Resolved)

| Question | Resolution | Date |
|----------|-----------|------|
| Astro vs Next.js? | **Astro** — content-first, islands architecture, ships less JS | Mar 30 |
| Annotation storage? | **SQLite MVP**, Supabase post-MVP | Mar 30 |
| Deployment? | **GitHub Pages** — free, native auth via repo visibility | Mar 30 |
| Auth mechanism? | **GitHub repo visibility** — public repo = public, private repo = restricted | Mar 30 |
| npm package? | **Standalone app first**, design for extraction | Mar 30 |
| Public/private mechanism? | **Directory-based** config in `foundry.config.yaml`, enforced by repo visibility | Mar 30 |
| Source repos? | **Multi-repo from day one**, configured in `foundry.config.yaml` | Mar 30 |
| Chat integration? | **Batch comments → main session**, reply annotations via MCP in E4 | Mar 30 |
| Server from day one? | **No — static MVP, API added in v0.2.** Infrastructure is earned complexity. Code doesn't change, only the wrapper. | Mar 30 |
| Navigation? | **`nav.yaml`** — curated titles, explicit ordering, parsed at build time | Mar 30 |
| How does API connect to static site? | **Separate deployments, communicate over HTTPS.** Static site makes `fetch()` calls to API. | Mar 30 |
| MCP in architecture? | **Layer 1 (dev):** normal CSDLC. **Layer 2 (runtime):** Foundry API = MCP server in v0.3+ | Mar 30 |
| E4: Batch submit pattern? | **MCP `submit_review`**, configurable endpoint, not OpenClaw-locked | Mar 30 |
| E4: Annotation anchoring? | **Heading path** (primary) + **content hash** (drift detect) + orphan handling | Mar 30 |
| E4: SQLite → Supabase? | **cuid2 UUIDs**, standard types, nullable `user_id`, one-script migration | Mar 30 |
| E4: Comment UX? | **Option C: unified thread panel** (~35% right), doc-scoped, not main chat | Mar 30 |
| E4: Reply path? | **MCP `create_annotation`** with `parent_id`, natural Telegram notification | Mar 30 |
| E4: Auth/identity? | **Hardcoded `user_id='dan'`** MVP, `author_type: human\|ai`, GitHub OAuth v0.3+ | Mar 30 |

---

## Non-Goals (For Now)

- Multi-user auth / teams / permissions (just us for MVP, GitHub handles access)
- Real-time collaborative editing (not Google Docs)
- Non-markdown format support (Word/PDF adapters are post-MVP)
- Mobile app
- AWS/cloud infrastructure (earned when demand justifies it)
- Terraform / IaC (packaging decision, not architecture decision)

---

## Related

- [Anvil Design Doc](../anvil/design.md) — search & retrieval layer
- [CSDLC Process](../../methodology/process.md) — the workflow this optimizes
- [Forge Design Doc](../forge/design.md) — MCP monetization (future)
