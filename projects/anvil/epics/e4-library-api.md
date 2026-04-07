# E4: Library API + Embedding Providers

*Status: Step 0 — Design Doc (Refinement Complete)*
*Epic: Anvil v0.2*
*Created: March 30, 2026*
*Updated: March 30, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

Refactor Anvil to expose its core capabilities as an importable JavaScript/TypeScript library, in addition to the existing MCP server interface. Add OpenAI as an embedding provider option, fixing the 256-token truncation bug.

### Problem Statement

Anvil v0.1 is MCP-only — every consumer must speak the MCP protocol. When Foundry's API server needs semantic search, it shouldn't have to spawn Anvil as a subprocess and communicate over stdio. It should `import { createAnvil } from '@claymore-dev/anvil'` and call functions directly.

Additionally, the current embedding model (all-MiniLM-L6-v2) has a 256-token context limit that truncates long document chunks, degrading search quality.

### Goals

- Export Anvil core as an importable library API (`createAnvil()` factory)
- MCP tools become thin wrappers around the library (zero logic duplication)
- Configurable embedding provider: `local` (MiniLM, current) or `openai` (text-embedding-3-small)
- OpenAI embeddings: 8,191 token context, 1,536 dimensions — fixes truncation bug
- SQLite-vss dimension handling: detect config change, auto-rebuild vector table
- CLI commands use library API internally

### Non-Goals

- New MCP tools (tool interfaces unchanged)
- npm publish (done manually after human review)
- Additional embedding providers beyond local + OpenAI
- Breaking changes to existing MCP tool schemas
- Format adapters (Word, PDF — still v0.3+)

---

## Context

### Why Now?

Foundry E2 (API Server + Agent MCP Foundation) needs to import Anvil as a library. This is a prerequisite — Foundry can't consume Anvil until the library API exists.

### Dependencies

- None — this is self-contained Anvil work

### Dependents

- **Foundry E2** — imports `@claymore-dev/anvil` as library for search
- **Any future service** that needs Anvil search without MCP overhead

---

## Design

### Two-Interface Architecture

```
                    @claymore-dev/anvil
┌──────────────────────────────────────────────┐
│                                              │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │  Library API     │  │  MCP Server      │  │
│  │  (importable)    │  │  (stdio/SSE)     │  │
│  │                  │  │                  │  │
│  │  createAnvil()   │  │  search_docs     │  │
│  │  anvil.search()  │  │  get_page        │  │
│  │  anvil.getPage() │  │  get_section     │  │
│  │  anvil.listPages │  │  list_pages      │  │
│  │  anvil.getStatus │  │  get_status      │  │
│  │  anvil.index()   │  │                  │  │
│  └────────┬─────────┘  └───────┬──────────┘  │
│           │                    │              │
│           └────────┬───────────┘              │
│                    │                          │
│           ┌────────▼─────────┐                │
│           │   Core Layer     │                │
│           │                  │                │
│           │  query.ts        │                │
│           │  db.ts           │                │
│           │  indexer.ts      │                │
│           │  embedder.ts     │                │
│           │  chunker.ts      │                │
│           └──────────────────┘                │
└──────────────────────────────────────────────┘
```

Both interfaces call the same core layer. Library is direct function calls. MCP tools are thin wrappers that parse MCP input, call the library, and format MCP output.

### Library API Surface

```typescript
// Entry point: @claymore-dev/anvil
export async function createAnvil(config: AnvilConfig): Promise<Anvil>;

export interface AnvilConfig {
  docsPath: string;                    // Path to markdown docs
  dbPath?: string;                     // SQLite location (default: .anvil/)
  embedding?: {
    provider: 'local' | 'openai';     // Default: 'local'
    model?: string;                    // Default per provider
    apiKey?: string;                   // For OpenAI (or OPENAI_API_KEY env)
  };
}

export interface Anvil {
  search(query: string, topK?: number): Promise<SearchResult[]>;
  getPage(filePath: string): Promise<PageResult>;
  getSection(filePath: string, headingPath: string): Promise<SectionResult>;
  listPages(): Promise<PageListResult[]>;
  getStatus(): Promise<StatusResult>;
  index(options?: { force?: boolean }): Promise<IndexResult>;
  close(): Promise<void>;
}
```

Types (`SearchResult`, `PageResult`, etc.) already exist in `types.ts` — they just need to be exported.

### Embedding Provider Abstraction

```typescript
// src/embedder.ts — refactored
export interface EmbeddingProvider {
  readonly dimensions: number;
  init(): Promise<void>;
  embed(text: string): Promise<Float32Array>;
  embedBatch?(texts: string[]): Promise<Float32Array[]>;
}

export class LocalEmbedder implements EmbeddingProvider {
  // Existing Xenova/all-MiniLM-L6-v2 logic
  readonly dimensions = 384;
}

export class OpenAIEmbedder implements EmbeddingProvider {
  // New: calls OpenAI API
  readonly dimensions = 1536;
  // Uses text-embedding-3-small
  // 8,191 token context
  // embedBatch() for efficient bulk indexing
}

export function createEmbedder(config: AnvilConfig['embedding']): EmbeddingProvider;
```

### SQLite-vss Dimension Handling

Current code hardcodes `embedding(384)`. This must become dynamic:

1. On `createAnvil()`, store embedding dimensions in `anvil_meta` table:
   ```sql
   INSERT OR REPLACE INTO anvil_meta (key, value) VALUES ('embedding_dimensions', '1536');
   ```
2. On startup, check stored dimension vs current config dimension
3. **If mismatch:**
   - Log warning: "Embedding dimensions changed (384 → 1536). Rebuilding vector index."
   - Drop `chunks_vss` table
   - Recreate with new dimensions: `CREATE VIRTUAL TABLE chunks_vss USING vss0(embedding(1536))`
   - Trigger full re-embed of all existing chunks
4. **If match:** proceed normally
5. `--force` flag always rebuilds

### File Changes

| File | Change |
|------|--------|
| `src/anvil.ts` | **NEW** — factory function, Anvil class wrapping core modules |
| `src/embedder.ts` | Refactor to provider interface + local/OpenAI implementations |
| `src/db.ts` | Dynamic dimensions in vss table creation, dimension mismatch detection |
| `src/tools/index.ts` | Refactor to call library API instead of core modules directly |
| `src/server.ts` | Use `createAnvil()` internally |
| `src/cli.ts` | Use `createAnvil()` for `index` and `serve` commands |
| `src/config.ts` | Add embedding provider config schema |
| `src/types.ts` | Export all result types |
| `package.json` | Add `"exports"` field for library entry point, add `openai` dependency |

---

## Edge Cases & Gotchas

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| OpenAI API key missing when provider = 'openai' | Clear error: "OpenAI API key required. Set embedding.apiKey or OPENAI_API_KEY env var." | Don't silently fall back — explicit failure is better |
| Dimension mismatch on startup | Auto-rebuild vss table + re-embed all chunks | Could be slow on large indexes — log progress |
| OpenAI rate limit during bulk indexing | Respect 429 headers, exponential backoff | `embedBatch()` should handle this gracefully |
| Network failure during OpenAI embed | Retry with backoff, fail after 3 attempts | Single embed failure shouldn't crash indexing — skip chunk, log warning |
| Library consumer doesn't call close() | Log warning on process exit if DB not closed | Prevents WAL checkpoint issues |
| MCP tool behavior changes | Must NOT change — same inputs produce same outputs | Library refactor is internal only |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| SQLite-vss dimension rebuild loses data | Low | High | Only drops vss table, not chunks table. Content preserved, just re-embedded. |
| OpenAI embedding quality different from local | Low | Low | Should be BETTER — larger context, better model. Test with known queries. |
| Package exports break existing MCP consumers | Low | High | MCP tool schemas unchanged. Integration test with mcporter before merge. |
| `openai` npm dependency adds bloat | Low | Low | It's one HTTP call wrapper. Minimal footprint. |

---

## Testing Strategy

### Test Layers

| Layer | Applies? | Notes |
|-------|:--------:|-------|
| **Unit tests** | Yes | Library API, embedder abstraction, dimension handling |
| **Integration tests** | Yes | `createAnvil()` → search → results. MCP tools still work. |
| **Regression tests** | Yes | All 149 existing tests must pass |
| **Manual verification** | Yes | Search quality comparison: MiniLM vs OpenAI on same docs |

### Verification Rules

1. All 149 existing tests pass
2. `createAnvil()` returns working instance
3. `anvil.search()` returns results identical to MCP `search_docs`
4. `anvil.getPage()` returns results identical to MCP `get_page`
5. OpenAI embedder produces valid 1536-dim vectors
6. Dimension mismatch triggers auto-rebuild with warning
7. `--force` reindex works with both providers
8. MCP tools work identically to before (regression)
9. CLI commands work identically to before (regression)
10. Library can be imported from another package: `import { createAnvil } from '@claymore-dev/anvil'`

---

## Stories

| Story | Summary | Batch | Dependencies | Status | PR |
|-------|---------|-------|-------------|--------|----| 
| S1 | Embedding provider abstraction (interface + local + OpenAI) | 1 | None | | |
| S2 | Dynamic SQLite-vss dimensions + mismatch detection | 1 | None | | |
| S3 | Library API (`createAnvil()` + Anvil class) | 2 | S1, S2 | | |
| S4 | Refactor MCP tools + CLI to use library API | 3 | S3 | | |

**Execution plan:** [S1, S2 parallel] → S3 → S4

### S1: Embedding Provider Abstraction

**Summary:** Refactor `embedder.ts` to a provider interface pattern. Extract current local model as `LocalEmbedder`, add `OpenAIEmbedder`.

**Acceptance Criteria:**
1. `EmbeddingProvider` interface defined with `dimensions`, `init()`, `embed()`, `embedBatch?()`
2. `LocalEmbedder` class — existing Xenova/MiniLM logic, `dimensions = 384`
3. `OpenAIEmbedder` class:
   - Uses `text-embedding-3-small` model
   - `dimensions = 1536`
   - `embed(text)` calls OpenAI API, returns Float32Array
   - `embedBatch(texts)` sends batch request (up to 2048 inputs per API call)
   - API key from constructor config or `OPENAI_API_KEY` env
   - Retry with exponential backoff on 429/5xx (3 attempts)
   - Clear error message if API key missing
4. `createEmbedder(config)` factory function selects provider
5. Config type: `{ provider: 'local' | 'openai', model?: string, apiKey?: string }`
6. Tests: local embedder works as before, OpenAI embedder mocked for unit tests
7. Existing tests that use the embedder still pass

**Boundaries:**
- Do NOT change the database layer (S2)
- Do NOT create the library API (S3)
- Do NOT add providers beyond local + OpenAI

### S2: Dynamic SQLite-vss Dimensions

**Summary:** Make the vector table dimension configurable and add mismatch detection/auto-rebuild.

**Acceptance Criteria:**
1. `db.ts` constructor accepts `dimensions: number` parameter
2. `CREATE VIRTUAL TABLE` uses dynamic dimension: `embedding(${dimensions})`
3. On startup, store current dimension in `anvil_meta`: key `embedding_dimensions`
4. On startup, if stored dimension ≠ config dimension:
   - Log warning with old and new dimensions
   - Drop `chunks_vss` table
   - Recreate with new dimensions
   - Set `anvil_meta` key `needs_reembed` = `true`
5. `needsReembed()` method returns true when dimension change detected
6. Existing chunks preserved (only vss table is dropped/rebuilt)
7. All existing DB tests pass with `dimensions = 384` (backward compatible)
8. New tests: dimension mismatch detection, vss table rebuild

**Boundaries:**
- Do NOT trigger re-embedding (that's the indexer's job, orchestrated by S3/S4)
- Do NOT change the embedder (S1)

### S3: Library API (createAnvil + Anvil Class)

**Summary:** Create the main library entry point that wires together all core modules into a clean, importable API.

**Acceptance Criteria:**
1. New `src/anvil.ts`:
   - `createAnvil(config: AnvilConfig): Promise<Anvil>` factory function
   - Initializes embedder (via S1's `createEmbedder`)
   - Initializes database (via S2's dynamic dimensions)
   - If `db.needsReembed()`: auto-triggers re-embed of all chunks
   - Returns `Anvil` instance
2. `Anvil` class methods:
   - `search(query, topK?)` — wraps `query.ts` search logic
   - `getPage(filePath)` — wraps `query.ts` getPage logic
   - `getSection(filePath, headingPath)` — wraps `query.ts` getSection logic
   - `listPages()` — wraps `query.ts` listPages logic
   - `getStatus()` — wraps status logic (chunk count, last indexed, index health)
   - `index(options?)` — wraps indexer with optional `force` flag
   - `close()` — closes DB, cleans up resources
3. Package exports updated:
   - `package.json` `"exports"` field includes library entry point
   - `import { createAnvil } from '@claymore-dev/anvil'` works
   - All result types exported (`SearchResult`, `PageResult`, etc.)
4. `AnvilConfig` validated on creation (clear errors for missing required fields)
5. Integration tests: create instance → index docs → search → verify results
6. Works as both ESM import and in the existing CLI/MCP server context

**Boundaries:**
- Do NOT refactor MCP tools or CLI yet (S4)
- Do NOT add new functionality beyond wrapping existing core logic

**Dependencies:** S1 (embedder), S2 (dynamic dimensions)

### S4: Refactor MCP Tools + CLI to Use Library API

**Summary:** Refactor existing MCP tool handlers and CLI commands to use the `Anvil` library API instead of calling core modules directly. Validates zero behavior change.

**Acceptance Criteria:**
1. `src/server.ts` (MCP server entry):
   - Creates `Anvil` instance via `createAnvil()`
   - Passes instance to tool handlers
2. `src/tools/index.ts`:
   - Each MCP tool calls `anvil.search()`, `anvil.getPage()`, etc.
   - No direct imports from `query.ts`, `db.ts`, `indexer.ts`
   - Tool input/output schemas UNCHANGED
3. `src/cli.ts`:
   - `serve` command creates `Anvil` instance, passes to MCP server
   - `index` command creates `Anvil` instance, calls `anvil.index()`
   - `init` command unchanged (no Anvil instance needed)
4. All 149 existing tests pass (regression)
5. mcporter integration test: connect to Anvil MCP, call `search_docs`, verify results
6. CLI test: `anvil index --force --docs ./test-docs` works
7. No direct imports of core modules (`query.ts`, `db.ts`, `indexer.ts`) outside of `anvil.ts`

**Boundaries:**
- Do NOT add new MCP tools
- Do NOT change tool schemas
- This is a pure refactor — behavior must be identical

**Dependencies:** S3 (library API exists)

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Mar 30 | Export library API alongside MCP | Foundry needs direct import; MCP overhead unnecessary for same-process consumers | MCP sidecar (extra process, stdio overhead), HTTP wrapper (extra server) |
| Mar 30 | OpenAI `text-embedding-3-small` | 8K token context fixes truncation bug, $0.02/M tokens, quality upgrade | Larger local model (still limited context), OpenAI `text-embedding-3-large` (overkill, more expensive) |
| Mar 30 | Keep local embedder as option | No hard dependency on OpenAI. Config choice. | OpenAI only (fragile), remove local (loses offline capability) |
| Mar 30 | Auto-rebuild vss on dimension change | Prevents silent failures. Chunks preserved, only vectors rebuilt. | Manual rebuild only (user forgets), error and refuse to start (too strict) |
| Mar 30 | Provider interface pattern | Clean abstraction. Adding future providers (Cohere, local alternatives) is one class. | If/else in embedder (messy), config-only switch (no type safety) |

---

## Known Issues / Tech Debt

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| Re-embedding on dimension change could be slow | Low | Accept | Only happens once per config change. Log progress. For 574 chunks it's <30 seconds even with OpenAI API calls. |
| `openai` npm dependency | Low | Accept | Minimal footprint. Could use raw `fetch()` instead but the SDK handles retries/types. |
