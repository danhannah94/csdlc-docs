# E8: Live Content Pipeline — Design Doc

*Status: Deployed — F4-S4 (Deferred Anvil Init) in refinement, all others shipped ✅*
*Created: April 1, 2026*
*Authors: Clay (stub), Dan (refinement)*
*Last updated: April 3, 2026*

---

## Overview

E8 transforms Foundry from a static-build platform into a live content platform. Currently, adding or editing docs requires: push to source repo → manually trigger Foundry rebuild → wait for Docker build + deploy. E8 makes it: push to source repo → Foundry detects the change → pulls fresh content → rebuilds nav + reindexes Anvil → serves updated pages. No manual intervention.

### Why This Matters

1. **GMPPU deployment** — they can't maintain a nav.yaml or trigger manual rebuilds. Content push must Just Work™.
2. **Our own workflow** — every time we add a doc, we have to push content, update nav.yaml, push foundry, wait for deploy. Three steps that should be one — push is the trigger, everything else is automatic.
3. **Foundry as a product** — if we ever offer this to others, "edit your nav.yaml and rebuild Docker" is a non-starter.

### Key Architecture Decision

**Astro SSR migration.** Currently Astro generates static HTML at Docker build time. E8 moves to server-side rendering so pages are generated from markdown at request time. This means:
- No rebuild needed when content changes
- Nav generates from directory structure automatically
- Anvil reindexes on content update
- Single webhook endpoint handles everything

---

## Scope (High Level — Needs Refinement)

### Content Lifecycle
- `POST /api/webhooks/content-update` — receives GitHub webhook on source repo push
- Webhook must verify GitHub signature (`X-Hub-Signature-256`) — see Security section
- Foundry pulls fresh content via git (runtime, not build time)
- Auto-generates nav tree from directory structure + frontmatter metadata
- Triggers Anvil reindex of changed/new content
- Serves updated pages immediately (SSR, no rebuild)

### Astro SSR Migration
- Switch from static generation to on-demand rendering
- Markdown parsed and rendered at request time
- Content cached aggressively (invalidated by webhook)
- Static assets (CSS, JS, images) still served statically

### Auto-Nav Generation

**Decision (from review):** Auto-nav by default. No manual/auto toggle per source in v1.

- Directory structure → nav tree (folder names or frontmatter `title` override)
- Sort: alphabetical default, `order` frontmatter field for custom ordering
- Optional `nav.yaml` override for when someone needs fully custom ordering (escape hatch, not primary mode)
- No hybrid mode in v1 — keep it simple

### Anvil Live Reindexing
- On content update: use GitHub webhook payload (commit diff) to identify changed files, reindex only those
- Anvil owns content intelligence — diffing what changed between versions is a natural extension of its chunking/indexing role
- Foundry owns orchestration — webhook receiver, git pull, triggering Anvil reindex, cache invalidation
- Full reindex available via API/CLI for recovery
- Index serves search results immediately after reindex completes

### Error Handling & Rollback
- **Last-known-good content** — if a pull breaks rendering, serve the previous version while flagging the error
- **Error surface** — status banner in Foundry UI: "Content update failed at [timestamp] — check source repo"
- **Status endpoint** — `GET /api/content/status` for programmatic health checks
- **Logged details** — full error details for debugging, not just "something broke"
- Users should never see a broken page

---

## Security & Configuration

### Webhook Security
- Verify GitHub webhook signatures (`X-Hub-Signature-256`) to prevent spoofing
- Reject requests with invalid or missing signatures

### Git Credentials in Container
- Use **deploy keys** (read-only SSH keys) scoped to the specific source repo
- Do NOT use PATs with broad repository access
- Deploy key configured at Docker level, not stored in application code

### Content Rendering Safety
- Astro sanitizes HTML by default in markdown rendering — verify this covers all XSS vectors
- Consider explicit allowlist for HTML elements in markdown if needed
- Audit: embedded `<script>` tags, event handlers, `javascript:` URLs

### Configuration Layers

**Docker / Infrastructure** (set at deploy time):
- Deploy key path
- Webhook secret
- Source repo URL
- Branch to track

**Foundry UI Settings** (runtime, no redeploy needed):
- *Future:* Which branches to track, nav mode preferences, reindex triggers
- Enables non-technical users (e.g., GMPPU) to configure content sources without touching Docker

---

## Design Decisions (Resolved)

| Decision | Resolution | Rationale |
|---|---|---|
| Steps to update content | One (push) | User pushes to source repo. Everything else is automatic. |
| Multi-source repos | Single source for E8 | Eliminates concurrency complexity. GMPPU = one repo, we = one repo. Multi-source is future. |
| Nav generation | Auto by default | Directory structure drives nav. Optional nav.yaml override as escape hatch. No hybrid mode in v1. |
| Change detection | Anvil owns diffing, Foundry owns orchestration | Clean package boundary. Anvil becomes more valuable standalone — any MCP consumer can use its diff capability. |
| Error recovery | Last-known-good + error banner | Never show broken pages. Surface errors in UI and logs. |
| Git credentials | Deploy keys (read-only) | Scoped to specific repo. No broad PATs in containers. |
| Settings management | Docker for infra, UI for runtime (future) | Infra concerns at deploy time, user preferences at runtime. |

---

## Open Questions

1. **SSR performance** — is on-demand rendering fast enough, or do we need incremental static regeneration (ISR)?
   - **Approach:** Build with aggressive in-memory caching, benchmark. Astro SSR with cached markdown parsing should handle our scale (< 100 pages). Only pivot to ISR if we see > 500ms TTFB.
2. **Caching strategy** — GitHub webhook payload tells us which files changed, so we can invalidate specific routes. In-memory cache with route-level invalidation? CDN-level cache-busting? Anvil as a change detection layer for smarter invalidation is a v2 idea.
   - **Approach:** In-memory `Map<route, rendered>` with per-route invalidation from webhook payload. CDN-level is premature for our traffic. Anvil smart invalidation deferred to v2.
3. ~~**Auth mechanism for private content gating (F6-S1)**~~ **RESOLVED:** Shared secret token (`X-Foundry-Token` header, checked against env var). Simple binary gate: have the token = see private content, don't = public only. No per-user identity in v1. Upgrade to session-based auth in v2 when multi-user features (user-scoped annotations, edit history) require it.

---

## Story Breakdown

### Feature 1: Astro SSR Migration (F1)

The foundation — switches Foundry from static build to server-rendered pages. Everything else builds on this.

#### F1-S1: SSR Adapter + Dynamic Content Loader

**What:** Install `@astrojs/node` adapter, switch output mode from `static` to `server`, replace Astro content collection with a runtime markdown loader that reads from disk.

**Why:** Currently `[...slug].astro` uses `getCollection('docs')` + `getStaticPaths()` which only works at build time. SSR needs a dynamic page that reads markdown from a content directory at request time.

**Technical details:**
- Add `@astrojs/node` to `packages/site`
- Change `astro.config.mjs`: `output: 'server'`, add node adapter
- Replace `content.config.ts` collection-based loading with a filesystem-based loader
- `[...slug].astro` becomes a dynamic route: read markdown file from `content/` dir, parse frontmatter, render with remark/rehype pipeline
- Keep existing remark plugins (admonitions, mermaid) and rehype plugins (slug, autolink)
- Astro's `<Content />` component won't work for runtime markdown — need to use `unified` pipeline directly (remark → rehype → HTML string)
- Static assets (CSS, JS, component islands) still pre-built

**Acceptance criteria:**
- `npm run dev` serves pages via SSR (no `getStaticPaths`)
- Adding a new `.md` file to `content/` makes it immediately accessible (no rebuild)
- Existing pages render identically (remark/rehype plugins, code highlighting, mermaid)
- React islands (annotation panel, search) still hydrate correctly

**Risk:** Remark/rehype plugin compatibility with runtime rendering. Shiki (syntax highlighting) is async and may need special handling outside Astro's built-in pipeline. Test with a complex page (code blocks + mermaid + admonitions).

#### F1-S2: In-Memory Page Cache

**What:** Cache rendered HTML in memory, keyed by route path. Serve from cache on subsequent requests. Cache entries invalidated by content update pipeline (F3).

**Technical details:**
- `Map<string, { html: string; frontmatter: object; mtime: number }>` in API or Astro middleware
- On request: check cache → hit = serve immediately, miss = read file + render + cache
- Cache warm-up on startup: pre-render all pages (optional, can defer to lazy)
- Memory budget: ~100 pages × ~50KB rendered = ~5MB — trivial
- Expose `invalidateRoute(path)` and `invalidateAll()` for F3 webhook integration

**Acceptance criteria:**
- Second request to same page is < 10ms (cache hit)
- Cache miss renders page and stores result
- `invalidateRoute('/docs/foo')` forces next request to re-render from disk
- `GET /api/content/status` includes cache stats (size, hit rate)

---

### Feature 2: Auto-Nav Generation (F2)

Eliminates `nav.yaml` as a manual step. Directory structure becomes the nav tree.

#### F2-S1: Nav Tree Generator

**What:** Build nav tree from directory structure + frontmatter metadata. Replace `nav.yaml` parsing with filesystem scan.

**Technical details:**
- Scan `content/` directory recursively
- For each `.md` file: read frontmatter for `title`, `order`, `nav_title` (optional short title)
- Directory names become section labels (title-cased, hyphen → space) unless an `index.md` in the dir provides a `title` in frontmatter
- Sort: `order` frontmatter field first (ascending), then alphabetical by title
- Output: same data structure the sidebar component currently expects (nested `{ label, path, children }`)
- Cache the nav tree in memory — invalidate on content update
- **Escape hatch:** if `nav.yaml` exists in content root, use it instead (full override, not merge)
- Handle edge cases: empty directories (skip), files without frontmatter title (use filename), deeply nested (recursive, not limited to 4 levels)

**Acceptance criteria:**
- Nav tree matches directory structure without any `nav.yaml`
- `order` frontmatter field controls sort order within a section
- Adding/removing a file regenerates nav (after cache invalidation)
- Existing nav items render correctly in sidebar (no visual regression)
- `nav.yaml` override still works if present

#### F2-S2: Sidebar Component Refactor

**What:** Make sidebar component recursive (currently hardcoded to 4 levels — tech debt item). Consume nav tree from F2-S1.

**Technical details:**
- Current sidebar renders 4 nested levels explicitly — refactor to recursive `NavItem` component
- Accept nav tree data as prop (from API endpoint or SSR page data)
- Expand/collapse state per section (persist in localStorage)
- Active page highlighting based on current route
- This also resolves the tech debt: "Sidebar component is 4-level hardcoded — should be recursive"

**Acceptance criteria:**
- Sidebar renders arbitrary depth nav trees (tested with 5+ levels)
- Visual appearance unchanged for existing 4-level nav
- Expand/collapse persists across page navigation
- Active page highlighted correctly

---

### Feature 3: Content Pipeline + Webhook (F3)

The runtime content update mechanism — receives webhook, pulls content, orchestrates cache + nav + search invalidation.

#### F3-S1: Runtime Content Fetcher

**What:** Pull content from source repo at container startup and on-demand via internal trigger. Replaces the build-time `scripts/build.sh` for runtime use.

**Technical details:**
- On startup: clone/pull source repo to `content/` directory using deploy key
- `ContentFetcher` class with methods: `pull()`, `getChangedFiles(oldRef, newRef)`, `getCurrentRef()`
- Uses `child_process.execFile('git', ...)` — not a git library (keep deps small)
- Reads source repo URL + branch from env vars (`CONTENT_REPO`, `CONTENT_BRANCH`)
- Deploy key path from env var (`DEPLOY_KEY_PATH`)
- `pull()` returns `{ ref: string, changedFiles: string[] }` — the ref is the new HEAD SHA, changed files come from `git diff --name-only oldRef newRef`
- Mutex/queue: if a pull is already in progress, skip (drop duplicate webhook triggers)

**Acceptance criteria:**
- Container starts → clones source repo → content directory populated
- `pull()` returns list of changed files
- Concurrent `pull()` calls don't race (second call skipped)
- Works with deploy key SSH auth
- Falls back gracefully if repo is unreachable (logs error, serves last-known content)

#### F3-S2: Webhook Endpoint

**What:** `POST /api/webhooks/content-update` — receives GitHub push events, verifies signature, triggers content pull + downstream updates.

**Technical details:**
- Verify `X-Hub-Signature-256` header using `WEBHOOK_SECRET` env var
- Parse GitHub push event payload: extract `commits[].added`, `commits[].modified`, `commits[].removed`
- On valid webhook: trigger `ContentFetcher.pull()` → get changed files → invalidate page cache (F1-S2) → regenerate nav tree (F2-S1) → trigger Anvil reindex (F4)
- Return `200 OK` immediately, run pipeline async (webhook timeout is 10s)
- Reject: missing signature (401), invalid signature (403), non-push events (200 + ignored)
- Log: timestamp, commit SHA, files changed, pipeline duration

**Acceptance criteria:**
- Valid GitHub webhook triggers content pull + cache invalidation
- Invalid/missing signature rejected with appropriate status code
- Pipeline runs async (webhook responds within 1s)
- Changed pages serve updated content within 5s of webhook
- `GET /api/content/status` shows last webhook timestamp + result

#### F3-S3: Error Handling + Status

**What:** Last-known-good fallback, error banner in UI, and status endpoint for monitoring.

**Technical details:**
- **Last-known-good:** Before pulling new content, snapshot current `content/` directory ref. If new content breaks rendering (any page throws), revert to snapshot ref and flag error.
- **Status endpoint:** `GET /api/content/status` returns `{ lastUpdate: timestamp, status: 'ok' | 'error', lastError?: string, contentRef: string, cacheSize: number, cacheHitRate: number }`
- **Error banner:** React component in `DocLayout` — fetches `/api/content/status` on mount, shows dismissible warning banner if `status === 'error'`
- **Error detection:** Try rendering each changed page after pull. If any throws, that's a broken page. Revert the whole pull (simpler than partial rollback in v1).

**Acceptance criteria:**
- Broken content in source repo does NOT break live site
- Error banner appears when content update fails
- Status endpoint reflects current pipeline state
- Error details logged server-side for debugging

---

### Feature 4: Anvil Enhancements (F4)

Upgrades Anvil for live content workflows. Bundles the reindex API, new MCP tools, and existing tech debt.

#### Current State — Anvil MCP Tools

Foundry currently exposes 2 Anvil-backed MCP tools (via Foundry's MCP server):
- `search_docs` — semantic search across indexed content
- `list_pages` — list all pages with paths and access levels

The standalone Anvil MCP server (via mcporter) exposes 5 tools:
- `search_docs`, `get_page`, `get_section`, `list_pages`, `get_status`

**Gap:** Foundry's MCP layer only proxies `search_docs` and `list_pages`. It doesn't expose `get_page`, `get_section`, or `get_status` — agents using Foundry's MCP server can search but can't read the actual doc content.

#### F4-S1: Delta Reindex API

**What:** Add `reindexFiles(files: string[])` method to Anvil that reindexes only the specified files. Complements the existing `index({ force })` full-reindex method.

**Technical details:**
- New method on the `Anvil` interface: `reindexFiles(files: string[]): Promise<IndexResult>`
- For each file in the list:
  - If file exists on disk: read content, re-chunk, re-embed, upsert (same as `indexFile` in current `Indexer`)
  - If file was deleted: remove all chunks for that file path
- Returns same `IndexResult` shape as `index()` (files_processed, chunks_added/updated/deleted, duration_ms)
- Foundry calls this from the webhook pipeline (F3-S2) with the changed file list
- Existing `index()` method unchanged (full reindex for recovery/CLI use)
- **Tech debt bundled:** Fix the pre-existing search test failures (5 tests in `search.test.ts` — Anvil mock issue from E2). These tests mock Anvil incorrectly; fix the mocks while we're touching the Anvil integration.

**Acceptance criteria:**
- `reindexFiles(['path/to/changed.md'])` reindexes only that file
- `reindexFiles(['path/to/deleted.md'])` removes chunks for deleted file
- Full `index()` still works for recovery
- Search returns updated content immediately after reindex
- Pre-existing search test failures fixed

#### F4-S2: Foundry MCP Tool Expansion + Content Enrichment ✅ SHIPPED

**What:** Add missing Anvil-backed tools to Foundry's MCP server (`get_page`, `get_section`, `get_status`, `reindex`) and enrich the `get_page` API to return actual markdown content — not just section metadata.

**Why:** Agents using Foundry's MCP server could search but couldn't read doc content. The `GET /api/docs/:path` route only returned heading/level/charCount per section. With this change, `get_page` returns the full readable markdown for each section, and all 4 new MCP tools are wired end-to-end.

**What shipped:**
- MCP tools (`get_page`, `get_section`, `get_status`, `reindex`) were already wired in E9 (server.ts + http-client.ts + API routes)
- `GET /api/docs/:path` enriched: sections now include a `content` field with concatenated markdown from all chunks sharing the same heading, sorted by ordinal
- `GET /api/docs/:path/sections/:heading` already returned content via Anvil passthrough — verified
- 6 docs tests passing, including content aggregation test

**Acceptance criteria:** ✅ All met
- `get_page` via MCP returns page content readable by agents
- `get_section` returns specific section with content
- `get_status` returns index health info
- `reindex` triggers full reindex (auth required)
- All 4 tools work via both MCP (mcporter) and HTTP API
- Existing `search_docs` and `list_pages` tools unchanged

---

#### F4-S3: File Watcher for Local Dev Auto-Reindex ✅ SHIPPED

**What:** Add a file watcher that auto-triggers Anvil reindex when `.md` files change in the content directory during local development. Eliminates the need to manually `POST /api/reindex` after editing docs.

**Why:** In production, webhooks handle content updates. But in dev, you're editing files directly. Without this, developers had to manually trigger reindex after every content change to see updated search results.

**What shipped:**
- New `packages/api/src/file-watcher.ts`: watches `CONTENT_DIR` using `fs.watch({ recursive: true })`
- 500ms debounce — rapid edits batch into a single reindex call
- Prefers `reindexFiles()` (delta) when available, falls back to full `index()`
- **Only runs when `NODE_ENV !== 'production'`** — production uses webhooks
- Wired into `packages/api/src/index.ts` after `app.listen()` (alongside background Anvil index)
- 8 tests covering: reindex on change, non-.md filtering, debounce batching, dedup, delta vs full fallback, error handling

**Acceptance criteria:** ✅ All met
- File watcher auto-triggers Anvil reindex on `.md` file changes in dev mode
- 500ms debounce prevents rapid-fire reindexing
- Uses delta reindex (`reindexFiles`) when available, falls back to full `index()`
- Does NOT run in production (`NODE_ENV=production`)
- All tests pass

---

#### F4-S4: Deferred Anvil Initialization (Lazy Loading)

**What:** Start the Express API server immediately without waiting for Anvil to initialize. Anvil loads in the background and becomes available when ready. Search/doc endpoints return "indexing in progress" until initialization completes.

**Why:** Even with the OpenAI embedding fix (F4-S2 enhancement — dropped init from 49s to ~1s), the architecture is fragile. The entire server startup is blocked on `createAnvil()` completing — every route mount, middleware setup, and health check waits. If the embedding provider has a network hiccup, or we switch back to local embeddings, or the index grows large enough that re-embedding takes time, we're back to white-page cold starts. The server should never be blocked on Anvil.

**Current flow (blocking):**
```
clone content → createAnvil() [BLOCKS] → mount routes → app.listen() → ready
```

**Target flow (non-blocking):**
```
clone content → mount routes (anvil=null) → app.listen() → ready
                                                    ↓ (background)
                                              createAnvil() → swap in → index()
```

**Technical details:**

1. **Anvil holder pattern** — Replace the direct `anvil` variable with a lazy container:
   ```typescript
   // packages/api/src/anvil-holder.ts
   class AnvilHolder {
     private anvil: Anvil | null = null;
     private initializing = false;
     private ready = false;

     get(): Anvil | null { return this.anvil; }
     isReady(): boolean { return this.ready; }
     isInitializing(): boolean { return this.initializing; }

     async init(docsPath: string): Promise<void> {
       this.initializing = true;
       try {
         this.anvil = await loadAnvil(docsPath);
         this.ready = true;
       } finally {
         this.initializing = false;
       }
     }
   }
   ```

2. **Route changes** — Routes already have Anvil-optional fallbacks (`if (anvil) { ... } else { 503 }`). Refactor to pull Anvil from the holder instead of closure variable. When `holder.isInitializing()`, return a specific response:
   ```json
   { "status": "initializing", "message": "Search index is loading, please retry in a few seconds" }
   ```
   Use HTTP 503 with `Retry-After: 5` header so clients/agents know to retry.

3. **Health check** — `/api/health` should return `200 OK` even when Anvil is initializing (the server is healthy, search just isn't ready yet). Add an `anvil.status` field: `"ready"`, `"initializing"`, or `"unavailable"`.

4. **Background init in `index.ts`** — After `app.listen()`, kick off `anvilHolder.init(docsPath)` as a fire-and-forget promise. When it resolves, log success and trigger the background index. Same pattern as the current background `anvil.index()` call.

5. **Webhook handler** — Already checks `if (anvil)` before reindexing. Just needs to pull from holder instead of closure. If Anvil isn't ready when a webhook fires, queue the reindex (or just skip — the background init will do a full index anyway).

**What NOT to change:**
- MCP server initialization (doesn't depend on Anvil — uses HTTP API)
- Static file serving, proxy setup, access map generation
- The embedding provider config (F4-S2 enhancement) — orthogonal

**Acceptance criteria:**
- Server responds to health checks within 3s of container start (no Anvil dependency)
- Pages load and render immediately (SSR doesn't need Anvil)
- Search/doc API returns 503 + `Retry-After` during Anvil init, then works normally
- Health endpoint returns 200 with `anvil.status: "initializing"` during load, `"ready"` after
- No behavior change once Anvil is fully initialized
- Webhook reindex works correctly if Anvil finishes init after a webhook fires
- Tests cover: holder state transitions, route behavior during init, route behavior when ready

**Estimated lift:** Medium — 1-2 hours agent time. Mostly refactoring `index.ts` route mounting to use the holder pattern instead of closure capture. The routes themselves already handle `anvil = null`.

**Risk:** Low. The Anvil-optional paths already exist and are tested. Main risk is ensuring the holder swap is thread-safe (Node is single-threaded so this is basically free, but the async init promise needs clean error handling).

---

### Feature 5: Deployment + Infrastructure (F5)

Updates the Docker build and Fly.io deployment for the SSR + runtime content model.

#### F5-S1: Dockerfile + Docker Compose Updates

**What:** Rebuild Dockerfile for SSR mode. Remove build-time content fetching, add runtime dependencies (git, SSH), configure deploy key volume mount.

**Technical details:**
- **Remove** Stage 1 (content-fetcher) from Dockerfile — content is pulled at runtime now
- **Add** to runtime stage: `git`, `openssh-client` (for deploy key auth)
- Astro builds in SSR mode (`output: 'server'`) — produces a Node.js server instead of static files
- **Entry point:** Start both Astro SSR server + Express API server (or unify into single process — TBD, may be a design decision to add)
- Deploy key mounted as volume: `-v ./deploy-key:/app/.ssh/deploy_key:ro`
- **Env vars:** `CONTENT_REPO`, `CONTENT_BRANCH`, `WEBHOOK_SECRET`, `DEPLOY_KEY_PATH`
- Content directory: `/app/content/` (persistent volume on Fly.io, ephemeral elsewhere)
- **Cold start:** Container starts → pulls content → builds cache + nav → ready to serve

**Acceptance criteria:**
- `docker build` succeeds without any source repo access
- Container starts and pulls content on first boot
- Content survives container restart (Fly.io volume)
- Deploy key auth works for private repos
- Container size remains reasonable (< 500MB)

#### F5-S2: Fly.io Config + GitHub Webhook Setup

**What:** Update `fly.toml` for SSR, create Fly.io volume for content persistence, configure GitHub webhook on source repo.

**Technical details:**
- Update `fly.toml`: internal port matches Astro SSR server port, health check → `/api/content/status`
- Create Fly.io volume: `fly volumes create foundry_content --size 1 --region iad`
- Mount volume at `/app/content/` in `fly.toml`
- Set secrets: `fly secrets set WEBHOOK_SECRET=... DEPLOY_KEY_PATH=/app/.ssh/deploy_key`
- Configure GitHub webhook on `csdlc-docs` repo: push events → `https://foundry.fly.dev/api/webhooks/content-update`
- Generate + configure deploy key: `ssh-keygen` → add public key to repo deploy keys → mount private key in container
- Update GitHub Actions workflow for SSR build (no `FOUNDRY_CONFIG` secret needed — config is runtime)

**Acceptance criteria:**
- Push to `csdlc-docs` triggers webhook → content updates on live site within 30s
- Content persists across deploys (volume mount)
- Health check endpoint works for Fly.io monitoring
- Deploy key has read-only access to source repo only

---

## Dependency Graph

```
F1-S1 (SSR Adapter)
  ↓
F1-S2 (Page Cache) ← F3-S2 invalidates
  ↓
F2-S1 (Nav Generator) ← F3-S2 regenerates
  ↓
F2-S2 (Sidebar Refactor)

F4-S1 (Delta Reindex) — ✅ SHIPPED
  ↓
F4-S2 (MCP Tool Expansion + Content) — ✅ SHIPPED
  ↓
F4-S3 (File Watcher, dev only) — ✅ SHIPPED

F4-S4 (Deferred Anvil Init) — READY

F3-S1 (Content Fetcher) — depends on F1-S1 (needs content dir structure)
  ↓
F3-S2 (Webhook) — depends on F3-S1, F1-S2, F2-S1, F4-S1 (orchestrates all)
  ↓
F3-S3 (Error Handling) — depends on F3-S2

F5-S1 (Dockerfile) — depends on F1-S1 (SSR build output)
  ↓
F5-S2 (Fly.io Config) — depends on F5-S1, F3-S2
```

**Parallelism opportunities:**
- F1 (SSR) and F4 (Anvil) can run in parallel — different packages, no code overlap
- F2-S1 (Nav Generator) can start once F1-S1 ships (needs SSR to serve generated nav)
- F3 and F5 come last — they wire everything together

**Scope:** 16 stories across 6 features. 15 shipped, 1 in refinement (F4-S4).

---

## Dependencies

- E7 (UI/UX Refinement) should ship first — E8 changes serving infrastructure, E7 changes presentation ✅
- E9 (MCP Agent Surface) should ship first — E8 extends the MCP server E9 built ✅
- Anvil library API needs `reindexFiles()` method (F4-S1 — built as part of this epic)

---

## Not In Scope

- Multi-source repos (future epic)
- Multi-user editing (Supabase migration, v0.3+)
- Real-time collaboration (WebSocket-based, future)
- Git-based editing from Foundry UI (Suggest Edit feature, v0.3+)
- Anvil changelog feature ("what changed since your last visit") — cool v2 idea, not E8
- Anvil as CDN-level cache invalidation layer — v2 optimization
- Unifying Astro SSR + Express API into single process — evaluate during F5, decide then

---

## Feature 6: Deployment Cleanup + Access Control (F6)

Post-deployment hardening. Addresses access control gap (all docs currently served as public) and replaces deployment workarounds with proper infrastructure patterns.

#### F6-S1: Runtime Access Map Generation

**What:** Generate `.access.json` from `foundry.config.yaml` after content clone, and enforce access checks in SSR page routes — not just API routes.

**Why:** Both the API layer (`packages/api/src/access.ts`) and the SSR nav generator (`packages/site/src/utils/nav.ts`) already read `.access.json` and respect access levels. But nothing generates that file at runtime. The content fetcher clones the source repo and stops — no `.access.json` gets written. Additionally, `[...slug].astro` serves any file on disk with zero access checks. The nav correctly hides private items from anonymous users, but direct URL access bypasses that entirely.

**Technical details:**
- **Generator function:** After `ContentFetcher.pull()` completes, parse `foundry.config.yaml` sources and produce `.access.json` in the content directory. The format is a prefix→access map:
  ```json
  {
    "methodology/": "public",
    "about/": "public",
    "projects/": "private"
  }
  ```
- **Trigger:** Runs on container startup (after initial clone) AND after every webhook-triggered pull (config could change)
- **SSR access check:** In `[...slug].astro`, after resolving the file path, call the same prefix-matching logic that `nav.ts` uses. If the page is `private` and the request isn't authenticated, return 403 or redirect to a "not authorized" page.
- **API already handles this:** `packages/api/src/routes/search.ts` and `pages.ts` already filter by access level — just need to make sure `loadAccessMap()` is called on startup after `.access.json` is written.
- **Auth model for v1:** Simple — if a request has a valid session/token, they see private content. No per-user granularity. "Public" = everyone, "Private" = authenticated users. Auth mechanism TBD (could be a shared secret header for GMPPU, or a proper login for v2).

**Acceptance criteria:**
- `.access.json` generated in content directory after clone/pull
- SSR pages under `projects/` return 403 (or redirect) for unauthenticated requests
- SSR pages under `methodology/` and `about/` serve normally without auth
- Nav sidebar hides private sections for unauthenticated users (already works if `.access.json` exists)
- API search/pages endpoints respect access levels (already works if `.access.json` exists)
- `.access.json` regenerated on webhook content updates

**Risk:** Auth mechanism is the open question. For GMPPU, a shared secret header or basic auth is fine for v1. Full user auth is a v2 concern. Story should implement the access map generation + SSR gating, with a pluggable auth check that can start as a simple shared secret.

---

#### F6-S2: Replace Hardcoded Sleep with Health Check Polling

**What:** Replace `sleep 10` in `start.sh` with a retry loop that polls `localhost:3001/api/content/status` until the API is actually ready.

**Why:** The `sleep 10` is calibrated to Anvil's current model load time (~8s). If Anvil gets faster, we waste startup time. If it gets slower (bigger model, more content to index), the proxy starts before the API is ready and requests fail. Either direction breaks.

**Technical details:**
- Replace the `sleep 10` in `scripts/start.sh` with a poll loop:
  ```sh
  echo "Waiting for API server..."
  for i in $(seq 1 30); do
    if curl -sf http://localhost:3001/api/content/status > /dev/null 2>&1; then
      echo "API server ready after ${i}s"
      break
    fi
    if [ "$i" -eq 30 ]; then
      echo "ERROR: API server failed to start within 30s"
      exit 1
    fi
    sleep 1
  done
  ```
- Add `curl` to Dockerfile runtime `apt-get install` line
- Fail hard if API never comes up (exit 1) — Fly.io restarts the machine
- Remove the `sleep 10` entirely

**Acceptance criteria:**
- Proxy starts within 2s of API becoming healthy (no wasted time)
- If API takes 20s to load (future heavier model), proxy still waits correctly
- If API never starts, container exits with error within 30s (Fly.io restarts)
- `curl` added to Dockerfile runtime deps
- No hardcoded sleep values remain in start.sh

**Risk:** Minimal. `curl` adds ~2MB to image. Poll pattern is standard container orchestration.

---

#### F6-S3: Canonical Content Directory (Remove Symlink)

**What:** Establish a single canonical content path (`/app/content`) via `CONTENT_DIR` env var. Both the content fetcher and Astro SSR read from this path directly. Remove the symlink from the Dockerfile.

**Why:** Currently: fetcher clones to `packages/site/content/` → Dockerfile creates `ln -s /app/packages/site/content/docs /app/content` → Astro reads from `/app/content`. Three places that need to agree on where content lives. If any one changes, it silently breaks. A single env var eliminates the coordination problem.

**Technical details:**
- `CONTENT_DIR` env var (default: `/app/content`) — the ONE path everything uses
- **Content fetcher** (`ContentFetcher` class): clone/pull directly into `$CONTENT_DIR` instead of `packages/site/content/`
- **Astro SSR content loader** (`[...slug].astro`): read markdown from `$CONTENT_DIR` instead of resolving relative to `process.cwd() + 'content'`
- **Nav generator** (`nav.ts`): scan `$CONTENT_DIR` for directory tree
- **Dockerfile changes:**
  - Remove the `RUN ln -s ...` symlink line
  - `RUN mkdir -p /app/content` stays
  - Add `ENV CONTENT_DIR=/app/content`
- **Local dev:** `CONTENT_DIR=./packages/site/content/docs` (or similar) in `.env` so dev workflow unchanged
- Audit all code referencing content paths — grep for `packages/site/content` and `content/docs`

**Acceptance criteria:**
- No symlinks in Dockerfile
- Content fetcher clones directly to `$CONTENT_DIR`
- Astro SSR resolves pages from `$CONTENT_DIR`
- Nav generator scans `$CONTENT_DIR`
- Local dev still works with `CONTENT_DIR` pointed at local content
- `grep -r "packages/site/content" --include="*.ts" --include="*.mjs" --include="*.sh"` returns zero hits in runtime code (build-time refs in site-builder stage are fine)
- Deployed site renders all pages identically (no visual regression)

**Risk:** Medium — touches multiple components (fetcher, SSR loader, nav generator, Dockerfile). Needs careful grep to find all path references. Worth testing locally with Docker build before deploying.

---

#### F6-S4: Trigger Anvil Index on Startup + Cache Warm-Up

**What:** After content clone completes on startup, trigger a full Anvil index and optionally pre-render all pages into the SSR cache.

**Why:** Two issues discovered in production:
1. Anvil initializes (loads embedding model) but never indexes content on startup — `createAnvil()` doesn't auto-index, and nothing calls `index()` after the initial clone. Result: search returns nothing until a webhook triggers reindex.
2. SSR page cache is cold after every machine restart (Fly.io auto-stop). Every page is a first-hit (~200-500ms render) until someone visits it. Pre-rendering on startup eliminates this.

**Technical details:**
- In `packages/api/src/index.ts`, after the content clone block and Anvil initialization:
  ```typescript
  if (anvil) {
    console.log('📇 Running initial Anvil index...');
    await anvil.index();
    console.log('✅ Anvil index complete');
  }
  ```
- **Cache warm-up (optional, same story):** After Anvil indexes, scan `CONTENT_DIR` for all `.md` files and hit each page route internally to populate the SSR cache. This way the first real user request is always a cache hit.
- The webhook `onContentUpdated` callback already handles reindex for subsequent updates — this story only covers the initial startup gap.
- Note: `reindexFiles()` doesn't exist yet (F4-S1). The webhook currently calls it and gets `TypeError: anvil.reindexFiles is not a function`. Until F4-S1 ships, the webhook should fall back to full `index()`.

**Acceptance criteria:**
- Anvil has >0 indexed pages immediately after startup (verify via `/api/health`)
- Search returns results on first request after machine wake (no prior webhook needed)
- Startup time increase is acceptable (<5s for ~30 pages)
- Webhook reindex doesn't crash (graceful fallback if `reindexFiles` unavailable)

**Risk:** Minimal. `index()` on 30 pages takes ~2-3s. Adds to cold start time but eliminates broken search state.

---

### F6 Dependency Graph

```
F6-S2 (Health Poll) — ✅ SHIPPED
F6-S3 (Content Dir) — ✅ SHIPPED
F6-S1 (Access Map) — in progress
F6-S4 (Anvil Startup Index) — independent, quick win
```
