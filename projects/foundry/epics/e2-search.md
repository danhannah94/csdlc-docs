# E2: API Server + Agent MCP Foundation

*Status: Step 0 — Design Doc (Refinement Complete)*
*Epic: Foundry v0.2*
*Created: March 30, 2026*
*Updated: March 30, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

Stand up Foundry's API server and expose agent-facing MCP tools for semantic search and annotation operations. This is the enablement layer that E3 (TTS) and E4 (Annotations) build on.

### Problem Statement

E1 shipped a static docs site on GitHub Pages — great for humans reading docs, but AI agents have no structured way to search, comment on, or interact with Foundry content. The features we want downstream (annotations, batch comments, suggest edits, TTS) all need a backend. E2 builds that backend and connects the first consumer: Anvil-powered semantic search.

### What Changed From the Original E2 Stub

The original E2 was scoped as "search bar for humans." During refinement, we reframed it:

| Original Scope | Revised Scope |
|----------------|--------------|
| Human-facing search bar UI | API server + agent MCP interface |
| Needed external hosting | Runs locally (agents are local) |
| Standalone search feature | Foundation for E3/E4 |
| Anvil via MCP sidecar | Anvil as imported library |

**Key insight:** The most valuable thing isn't a search bar — it's creating the interface for AI agents to interact with Foundry content. The human search bar drops in naturally once the API exists (E3+).

### Goals

- Express API server in `packages/api/`
- Anvil integrated as a library (not MCP sidecar)
- MCP server exposing agent tools (search, annotation schemas)
- Doc registry endpoints (list docs, get structure)
- Search endpoint proxying Anvil semantic search
- Local development workflow (`npm run dev`)

### Non-Goals

- Human-facing search bar UI (E3+)
- Annotation CRUD implementation (E4)
- TTS generation or audio serving (E3)
- External hosting / deployment (local only for v0.2)
- Auth beyond CORS
- MCP write tools for doc editing (v0.3+ — useful for nav.yaml atomic updates but overkill now)
- Mobile or responsive API concerns

---

## Context

### Dependencies

- **E1** — site scaffold, monorepo structure, `packages/api/` placeholder
- **[Anvil E4: Library API + Embedding Providers](../../anvil/epics/e4-library-api.md)** — Anvil must export core functions as importable library + OpenAI embedding swap
- **Re-index** — CSDLC docs re-indexed with new embedding model (Anvil E4 P2 equivalent)

### Dependents

- **E3 (TTS Playback)** — needs API server for audio generation endpoints
- **E4 (Annotations + Batch Submit)** — needs API server + annotation MCP tools implemented
- **v0.3 MCP-First** — builds on the MCP server scaffold established here
- **Human search bar** — trivial to add once search API endpoint exists

---

## Prerequisites

### P1: Anvil Library API Refactor

Anvil today is MCP-only — all access goes through MCP protocol. Foundry needs to import Anvil as a library, not spawn it as a sidecar process.

**Current architecture:**
```
Consumer → MCP Client → stdio → Anvil MCP Server → core logic
```

**Target architecture:**
```
Foundry API → import @claymore-dev/anvil → core logic (direct)
AI Agent → MCP Client → Anvil MCP tools → same core logic
```

**What this requires in Anvil:**

1. **Export core functions as library API:**
   - `createAnvil(config)` → returns an Anvil instance
   - `anvil.search(query, topK?)` → `SearchResult[]`
   - `anvil.getPage(filePath)` → `PageResult`
   - `anvil.getSection(filePath, headingPath)` → `SectionResult`
   - `anvil.listPages()` → `PageListResult[]`
   - `anvil.getStatus()` → `StatusResult`
   - `anvil.index(options?)` → re-index on demand

2. **MCP tools become thin wrappers:**
   - `tools/index.ts` calls the same exported functions
   - Zero logic duplication

3. **Embedder abstraction:**
   - Current: hardcoded `Xenova/all-MiniLM-L6-v2` via `@huggingface/transformers`
   - Target: config-driven provider (`local` or `openai`)
   - OpenAI adapter: `text-embedding-3-small` (8,191 token context, 1,536 dimensions)
   - Fixes the 256-token truncation bug 🐛

4. **npm package update:**
   - Export library API from package entry point
   - `@claymore-dev/anvil` becomes both a CLI/MCP server AND an importable library

**Estimated effort:** Half day. Anvil internals (`query.ts`, `db.ts`, `embedder.ts`, `indexer.ts`) are already well-separated. This is mostly wiring exports and adding the OpenAI embedder.

### P2: Re-Index CSDLC Docs

After embedding model swap, re-index all docs:
```bash
node anvil/dist/cli.js index --force --docs ./csdlc-docs/docs
```

Verify search quality with test queries against known content.

---

## Design

### Two-Interface Architecture

Foundry's API serves two distinct clients with different needs:

```
┌──────────────────────────────────────────────┐
│              Foundry API Server               │
│              (packages/api/)                  │
│                                               │
│  ┌─────────────────┐  ┌───────────────────┐  │
│  │   REST Layer     │  │   MCP Layer       │  │
│  │   (Express)      │  │   (MCP Server)    │  │
│  │                  │  │                   │  │
│  │  GET /health     │  │  search_docs      │  │
│  │  GET /docs       │  │  list_annotations │  │
│  │  GET /docs/:path │  │  create_annotation│  │
│  │  POST /search    │  │  resolve_annotati…│  │
│  │  POST /tts (E3)  │  │  submit_review    │  │
│  │  * /annotations  │  │                   │  │
│  │    (E4)          │  │                   │  │
│  └────────┬─────────┘  └────────┬──────────┘  │
│           │                     │              │
│           └──────────┬──────────┘              │
│                      │                         │
│           ┌──────────▼──────────┐              │
│           │   Core Logic        │              │
│           │                     │              │
│           │  import anvil →     │              │
│           │    search, getPage  │              │
│           │                     │              │
│           │  Annotation service │              │
│           │    (E4)             │              │
│           │                     │              │
│           │  TTS service (E3)   │              │
│           └─────────────────────┘              │
└──────────────────────────────────────────────┘
```

**REST Layer (HTTP)** — for the web UI (and any HTTP client):
- Search, doc browsing, TTS, annotation display
- Standard REST endpoints, JSON responses
- CORS configured for GitHub Pages origin

**MCP Layer (MCP protocol)** — for AI agents:
- Semantic search, annotation CRUD, batch review submission
- Agents connect via MCP client (mcporter or direct)
- Exposes only what agents need — no TTS, no audio serving

**Core Logic** — shared:
- Both layers call the same service functions
- Anvil imported as library for search
- Annotation service (E4), TTS service (E3) added later
- Zero logic duplication between REST and MCP

### API vs MCP — What Goes Where

| Capability | REST (Web UI) | MCP (Agent) | Epic |
|-----------|:---:|:---:|------|
| Health check | ✅ | — | E2 |
| List docs | ✅ | ✅ | E2 |
| Get doc structure | ✅ | ✅ | E2 |
| Semantic search | ✅ | ✅ `search_docs` | E2 |
| TTS generation | ✅ | — | E3 |
| Audio serving | ✅ | — | E3 |
| List annotations | ✅ | ✅ `list_annotations` | E4 |
| Create annotation | ✅ | ✅ `create_annotation` | E4 |
| Resolve annotation | ✅ | ✅ `resolve_annotation` | E4 |
| Batch submit review | ✅ | ✅ `submit_review` | E4 |
| Suggest edit (PR) | ✅ | ✅ `suggest_edit` | v0.3 |
| Doc write / nav update | — | ✅ (future) | v0.3+ |

**Design principle:** Agents work with content. Humans work with presentation. TTS is presentation — agents already have the text.

### REST Endpoints (E2 Scope)

```
GET  /health              → { status: "ok", version, anvil: { indexed, lastIndexed } }
GET  /docs                → [{ path, title, lastModified, chunkCount }]
GET  /docs/:path          → { path, title, sections: [{ heading, level, charCount }] }
POST /search              → { results: [{ path, heading, snippet, score }] }
     body: { query: string, topK?: number }
```

### MCP Tools (E2 Scope)

```typescript
// Implemented in E2:
search_docs: {
  query: string;      // Natural language search query
  top_k?: number;     // Max results (default 10)
} → SearchResult[]

// Schema defined in E2, implemented in E4:
list_annotations: {
  doc_path: string;
  section?: string;   // Filter by heading path
} → Annotation[]

create_annotation: {
  doc_path: string;
  section: string;    // Heading path anchor
  content: string;    // Comment text
  parent_id?: string; // For threading
} → Annotation

resolve_annotation: {
  annotation_id: string;
} → Annotation

submit_review: {
  doc_path: string;
  annotation_ids?: string[]; // Specific annotations, or all unsubmitted
} → { submitted: number, session_message: string }
```

### Project Structure

```
foundry/
├── packages/
│   ├── site/               # Astro static site (E1, unchanged)
│   └── api/
│       ├── src/
│       │   ├── index.ts          # Express app + MCP server entry
│       │   ├── config.ts         # Read foundry.config.yaml
│       │   ├── routes/
│       │   │   ├── health.ts     # GET /health
│       │   │   ├── docs.ts       # GET /docs, GET /docs/:path
│       │   │   └── search.ts     # POST /search
│       │   ├── mcp/
│       │   │   ├── server.ts     # MCP server setup
│       │   │   └── tools/
│       │   │       ├── search.ts     # search_docs tool (E2)
│       │   │       └── annotations.ts # annotation tool schemas (E2), impl (E4)
│       │   └── services/
│       │       ├── anvil.ts      # Anvil library wrapper
│       │       └── annotations.ts # Placeholder for E4
│       ├── package.json
│       └── tsconfig.json
├── foundry.config.yaml
├── nav.yaml
└── ...
```

### Key Design Decisions

**Anvil as library, not MCP sidecar.** Foundry imports `@claymore-dev/anvil` directly and calls `search()`, `getPage()`, etc. as function calls. No MCP protocol overhead, no extra process, no stdio pipes. The MCP layer in Anvil still exists for direct agent use — Foundry just doesn't need it.

**Single process, two protocols.** The Express server and MCP server run in the same Node process. REST endpoints handle HTTP from the web UI. MCP server handles agent connections over stdio (or SSE in the future). Both call the same service layer.

**MCP tool schemas defined early, implemented incrementally.** E2 defines the annotation tool schemas so agents can discover what's available, but returns "not yet implemented" until E4 ships the logic. This lets us validate the interface contract before building the implementation.

**Local only for v0.2.** No external hosting, no deployment pipeline for the API. `npm run dev` starts the server locally. Agents connect locally. The static site on GitHub Pages doesn't need the API to function — it gains search when the API is running, gracefully degrades when it's not.

**CORS for GitHub Pages.** The static site at `danhannah94.github.io` makes `fetch()` calls to `localhost:3001`. CORS headers on the API allow this. When we eventually host the API externally, we just update the allowed origin.

---

## Edge Cases & Gotchas

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| API not running when static site loads | Site works fine, search/annotations disabled | Progressive enhancement — feature detection, not hard dependency |
| Anvil index empty or stale | `/health` reports index status, search returns empty | Need clear UX/agent feedback that index needs building |
| Concurrent MCP + REST requests | Both work independently | Same Anvil instance, SQLite handles concurrent reads fine |
| Large search result payloads | Configurable `topK` with sensible default (10) | Don't return 100 chunks when 10 suffice |
| Doc path mismatch (API vs static site) | Paths must match — both derived from `foundry.config.yaml` | Source of truth is the config, not filesystem paths |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Anvil library refactor takes longer than expected | Low | Medium | Internals already well-separated; mostly wiring exports |
| OpenAI embedding dimension change breaks SQLite-vss | Low | High | Test with fresh index; vss extension handles dimension config |
| MCP server in Express conflicts with HTTP | Low | Medium | Well-documented pattern; MCP uses stdio, Express uses HTTP — no port conflict |
| CORS issues with GitHub Pages → localhost | Medium | Low | Standard CORS config; test early in S1 |

---

## Testing Strategy

### Test Layers

| Layer | Applies? | Notes |
|-------|:--------:|-------|
| **Unit tests** | Yes | Service layer functions, config parsing, route handlers |
| **Integration tests** | Yes | REST endpoints return correct data from Anvil index |
| **MCP tool tests** | Yes | Tool handlers return expected results for search queries |
| **E2E** | No | No deployed environment to test against yet |
| **Manual QA** | Yes | Dan verifies: start server, search works, agent connects |

### Verification Rules

1. `npm run dev` in `packages/api/` starts server on `localhost:3001`
2. `GET /health` returns status with Anvil index info
3. `GET /docs` returns list of all indexed documents
4. `GET /docs/:path` returns document structure with sections
5. `POST /search` returns ranked results from Anvil
6. MCP `search_docs` tool returns same results as REST endpoint
7. MCP annotation tool schemas are discoverable (return "not implemented" gracefully)
8. Server handles missing/empty Anvil index gracefully

---

## Stories

| Story | Summary | Batch | Dependencies | Status | PR |
|-------|---------|-------|-------------|--------|----| 
| S1 | API server scaffold (Express + config + health) | 1 | Anvil E4 | | |
| S2 | Doc registry endpoints (list + structure) | 2 | S1 | | |
| S3 | Search endpoint + Anvil integration | 2 | S1 | | |
| S4 | MCP server + search_docs tool | 3 | S3 | | |
| S5 | MCP annotation tool schemas (interface only) | 3 | S4 | | |

**Execution plan:** [Anvil E4 ships first] → S1 → [S2, S3 parallel] → [S4, S5 parallel]

### P1: Anvil Library API Refactor + OpenAI Embeddings

**Summary:** Refactor `@claymore-dev/anvil` to export core functions as an importable library API. Add OpenAI embedding provider as a config option. MCP tools become thin wrappers around the library.

**Acceptance Criteria:**
1. New file `src/anvil.ts` exports a factory function:
   ```typescript
   export async function createAnvil(config: AnvilConfig): Promise<Anvil>
   ```
2. `Anvil` instance exposes:
   - `search(query: string, topK?: number): Promise<SearchResult[]>`
   - `getPage(filePath: string): Promise<PageResult>`
   - `getSection(filePath: string, headingPath: string): Promise<SectionResult>`
   - `listPages(): Promise<PageListResult[]>`
   - `getStatus(): Promise<StatusResult>`
   - `index(options?: { force?: boolean }): Promise<void>`
3. `AnvilConfig` includes:
   - `docsPath: string` — path to markdown docs
   - `dbPath?: string` — SQLite database location (default: `.anvil/`)
   - `embedding?: { provider: 'local' | 'openai', model?: string, apiKey?: string }`
4. OpenAI embedding adapter:
   - Uses `text-embedding-3-small` (1,536 dimensions)
   - 8,191 token context — eliminates the 256-token truncation bug
   - Falls back to local model if OpenAI unavailable and local configured
   - API key from config or `OPENAI_API_KEY` env var
5. SQLite-vss vector table handles new dimensions:
   - Store embedding dimension in `anvil_meta` table
   - On startup, detect config dimension ≠ stored dimension → drop and rebuild `chunks_vss`
   - Current hardcoded `embedding(384)` in `db.ts` must become `embedding(${dimensions})`
   - `--force` reindex also rebuilds vss table
6. MCP tools (`tools/index.ts`) refactored to call library functions — zero logic duplication
7. CLI commands (`serve`, `init`, `index`) use library API internally
8. Package entry point (`package.json` `"exports"`) includes library API
9. All existing tests pass
10. New tests for library API: `createAnvil()`, `search()`, `getPage()`, `listPages()`

**Boundaries:**
- Do NOT change MCP tool interfaces (input/output schemas stay the same)
- Do NOT add new MCP tools
- Do NOT publish to npm yet (local development only)
- Keep local embedding model as a working option (config choice)

### P2: Re-Index CSDLC Docs

**Summary:** Re-index all CSDLC docs using the new OpenAI embedding model and verify search quality.

**Acceptance Criteria:**
1. Configure Anvil with `embedding.provider: 'openai'`
2. Run `anvil index --force --docs ./csdlc-docs/docs`
3. All 27+ pages indexed successfully
4. Test queries return relevant results:
   - "error handling" → finds Edge Cases & Gotchas sections
   - "how to write a design doc" → finds Step 0 documentation
   - "sub-agent pipeline" → finds execution model docs
5. Search quality noticeably improved over MiniLM (longer chunks not truncated)

**Boundaries:**
- This is a manual verification step, not automated
- Clay executes directly (no sub-agent needed)

### S1: API Server Scaffold

**Summary:** Create the Express server in `packages/api/` with config parsing, health endpoint, and CORS setup.

**Acceptance Criteria:**
1. `packages/api/package.json` with dependencies: `express`, `cors`, `@claymore-dev/anvil` (local path)
2. `packages/api/src/index.ts`:
   - Express server on configurable port (default `3001`)
   - CORS configured for `danhannah94.github.io` origin + `localhost` (dev)
   - Reads `foundry.config.yaml` from project root
   - Initializes Anvil instance via `createAnvil()`
3. `GET /health` endpoint:
   - Returns `{ status: "ok", version: "0.2.0", anvil: { indexed: number, lastIndexed: string } }`
   - Calls `anvil.getStatus()` for index info
4. Error handling middleware (500s return JSON, not stack traces)
5. `npm run dev` script starts server with hot reload (`tsx watch` or similar)
6. `npm run build` compiles TypeScript
7. `npm run start` runs compiled server
8. TypeScript strict mode, ESM modules

**Boundaries:**
- Do NOT implement doc or search endpoints (S2, S3)
- Do NOT set up MCP server (S4)
- Do NOT add authentication
- CORS: allow `localhost:*` in dev, specific origins in config for production

### S2: Doc Registry Endpoints

**Summary:** Add REST endpoints for listing documents and retrieving document structure.

**Acceptance Criteria:**
1. `GET /docs` — returns array of all indexed documents:
   ```json
   [{ "path": "methodology/process.md", "title": "CSDLC Process", "lastModified": "...", "chunkCount": 42 }]
   ```
   - Sources data from `anvil.listPages()`
2. `GET /docs/:path` — returns single document with section structure:
   ```json
   {
     "path": "methodology/process.md",
     "title": "CSDLC Process",
     "lastModified": "...",
     "sections": [
       { "heading": "Purpose", "level": 2, "charCount": 850 },
       { "heading": "Roles", "level": 2, "charCount": 1200 },
       { "heading": "Roles > AI Lead", "level": 3, "charCount": 600 }
     ]
   }
   ```
   - Sources from `anvil.getPage()`
   - `:path` is URL-encoded file path (e.g., `methodology%2Fprocess.md`)
3. 404 response for non-existent doc paths
4. Unit tests for route handlers
5. Integration test: start server, GET /docs returns indexed docs

**Boundaries:**
- Do NOT add search functionality (S3)
- Do NOT add filtering or pagination (not needed for ~30 docs)

**Dependencies:** S1 (server running with Anvil instance)

### S3: Search Endpoint + Anvil Integration

**Summary:** Add the search REST endpoint that proxies queries to Anvil's semantic search.

**Acceptance Criteria:**
1. `POST /search` endpoint:
   - Request body: `{ "query": "string", "topK": number? }`
   - Response: `{ "results": [{ "path": "...", "heading": "...", "snippet": "...", "score": 0.85 }] }`
   - Calls `anvil.search(query, topK)`
   - Default `topK`: 10
2. Returns empty results array (not error) when no matches found
3. Returns empty results with warning when Anvil index is empty
4. Validates request body (400 if `query` is missing or empty)
5. Search returns results within 100ms for typical queries (Anvil + OpenAI embeddings)
6. Unit tests for search route
7. Integration test: POST /search with known query returns relevant results

**Boundaries:**
- Do NOT build search UI (that's a future E3+ story)
- Do NOT add search analytics, query logging, or caching
- Do NOT implement faceted search or filters

**Dependencies:** S1 (server with Anvil instance)

### S4: MCP Server + search_docs Tool

**Summary:** Add an MCP server to the Foundry API that exposes `search_docs` as a tool for AI agents.

**Acceptance Criteria:**
1. MCP server initialized alongside Express server in `src/index.ts`
2. Uses `@modelcontextprotocol/sdk` for MCP server implementation
3. MCP transport: SSE over HTTP (agent connects to already-running Express server)
   - No separate process needed — MCP server piggybacks on the Express port
   - SSE endpoint at `/mcp` or similar
   - Agents connect via mcporter SSE config or direct HTTP
4. `search_docs` MCP tool:
   - Input: `{ query: string, top_k?: number }`
   - Output: same `SearchResult[]` as REST endpoint
   - Calls same service function as `POST /search`
5. MCP server discoverable via standard MCP handshake
6. mcporter config example in README for connecting agents
7. Test: mcporter can call `search_docs` and get results
8. Server can run both Express (HTTP) and MCP (stdio) simultaneously

**Boundaries:**
- Do NOT implement annotation tools (S5 defines schemas, E4 implements)
- Express and MCP share the same server process and port

**Dependencies:** S3 (search service layer exists)

### S5: MCP Annotation Tool Schemas

**Summary:** Define MCP tool schemas for annotation operations. Tools are discoverable but return "not yet implemented" until E4.

**Acceptance Criteria:**
1. Four MCP tools defined with full input/output schemas:
   - `list_annotations` — `{ doc_path, section? }` → `Annotation[]`
   - `create_annotation` — `{ doc_path, section, content, parent_id? }` → `Annotation`
   - `resolve_annotation` — `{ annotation_id }` → `Annotation`
   - `submit_review` — `{ doc_path, annotation_ids? }` → `{ submitted, session_message }`
2. Each tool returns a structured "not yet implemented — available in E4" message when called
3. Tools appear in MCP tool listing (`tools/list`)
4. `Annotation` type defined:
   ```typescript
   interface Annotation {
     id: string;
     doc_path: string;
     section: string;        // heading path anchor
     content: string;
     parent_id?: string;     // threading
     status: 'open' | 'resolved';
     created_at: string;
     updated_at: string;
   }
   ```
5. Types exported for use in E4 implementation

**Boundaries:**
- Do NOT implement annotation storage or logic (E4)
- Do NOT create database tables (E4)
- This is interface contract only — validates the API design before building

**Dependencies:** S4 (MCP server running)

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Mar 30 | Reframe E2 from "human search bar" to "API + agent foundation" | Agent interface is the real value; search bar drops in later | Keep original scope (search bar first) |
| Mar 30 | API runs locally only (no external hosting) | Agents are local, infrastructure is earned complexity | Cloudflare Workers (premature), Fly.io (premature) |
| Mar 30 | Anvil as imported library, not MCP sidecar | Direct function calls = no protocol overhead, single process | MCP sidecar via mcporter (extra process, stdio overhead) |
| Mar 30 | Swap to OpenAI `text-embedding-3-small` | 8K token context (vs 256), $0.02/M tokens, fixes chunking bug | Larger local model (still limited context), keep MiniLM (bug remains) |
| Mar 30 | REST for web UI, MCP for agents | Different clients, different needs. Agents don't need TTS. Humans don't need MCP. | Single protocol for both (forces compromise) |
| Mar 30 | Define annotation schemas in E2, implement in E4 | Validate interface contract early, don't build what we don't need yet | Implement annotations in E2 (scope creep), defer schemas to E4 (no early validation) |
| Mar 30 | Keep local embedding model as fallback | No hard dependency on OpenAI. Config choice, not architectural change. | OpenAI only (fragile), local only (bug remains) |
| Mar 30 | MCP write tools (doc edit + nav update) deferred to v0.3+ | Useful for external users, overkill for us now | Build in E2 (over-engineering for our use case) |
| Mar 30 | Progressive enhancement for static site | Site works without API. Search/annotations are additive. | Hard API dependency (fragile, breaks offline reading) |

---

## Known Issues / Tech Debt

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| SQLite-vss dimension change | Medium | Prereq | Switching from 384-dim (MiniLM) to 1,536-dim (OpenAI). P1 stores dimension in `anvil_meta`, detects mismatch on startup, auto-rebuilds vss table. Hardcoded `embedding(384)` in `db.ts` becomes configurable. |
| MCP SSE + Express HTTP share one port | Low | Monitor | Both are HTTP — SSE is just a long-lived connection. Standard pattern, but monitor for connection limits. |
| Anvil library API is new surface | Low | Accept | First consumer (Foundry) will shake out any rough edges in the library API. |

---

## Future MCP Tools (Not In Scope)

Documented here for design continuity — these are v0.3+ considerations:

| Tool | What It Does | Why It's Useful |
|------|-------------|-----------------|
| `edit_doc` | Write to a doc file + update nav.yaml atomically | Prevents nav drift when agents edit docs |
| `create_doc` | Create new doc + add to nav.yaml | Agents can scaffold docs without filesystem access |
| `suggest_edit` | Create a GitHub PR with proposed doc changes | Closes the feedback loop — conversation → doc improvement |

These become valuable when external users (without repo access) need to interact with docs. For us, agents write to the repo directly via git.

---

## Related

- [Foundry Design Doc](../design.md) — overall project architecture
- [E1: Scaffold + Deploy](e1-scaffold-deploy.md) — the foundation this builds on
- [Anvil Design Doc](../../anvil/design.md) — search engine consumed by Foundry
- [CSDLC Process](../../../methodology/process.md) — the workflow this enables
