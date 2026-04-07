# E11: Foundry-Native Platform — Design Doc

*Status: Step 1 — Refined, Ready for Execution*
*Created: April 6, 2026*
*Updated: April 7, 2026 — Two review rounds complete, auth punted to E12, stories refined*
*Authors: Dan Hannah & Clay*

---

## Overview

E11 is the epic where Foundry stops being a GitHub viewer and becomes a self-contained documentation platform. Today, Foundry treats GitHub as the primary content store and operates as a read-only renderer. This creates a 7-step edit cycle, webhook-dependent freshness, deploy key complexity, and split state between content (Git) and annotations (SQLite).

After E11, content and annotations live in the same system. GitHub becomes a sync target — a backup, not the source of truth. AI agents can read, search, annotate, AND edit docs through MCP tools without leaving Foundry.

### Why Now

This was observed directly during the E10 development session (April 6, 2026). While the annotation read/reply/resolve cycle took ~30 seconds via MCP tools, updating the actual doc content after resolving comments required cloning a repo, editing, committing, pushing, pulling inside the container, and triggering a reindex — ~10 minutes. The friction gap between "comment on a doc" and "update a doc" is the core problem.

Additionally, the current deploy architecture is fragile:
- Cold starts require SSH clone from GitHub (deploy keys, network dependency)
- Webhook-triggered content refreshes don't work in local dev
- Content staleness is a recurring issue (webhook misses, cache invalidation bugs)
- Anvil's ONNX indexing blocks the Express event loop for ~15 minutes on cold start (fixed with `/healthz` proxy endpoint, but the root cause remains)

### What's NOT in E11

- Auth & multi-user (YAML config, roles, multi-token) → **E12** (needs its own design doc)
- Collaborative editing (real-time, multi-cursor) → future
- Version history UI (browsing past versions) → future
- WYSIWYG editor → future (markdown-in, markdown-out is the right abstraction now)
- Human in-browser editing UI → future (agents edit via MCP, humans review via annotations for now)
- Full user management system (profiles, permissions matrix, admin panel) → future

---

## Sub-Systems

### SS1: Native Content Storage

**Problem:** Content lives in Git, annotations live in SQLite — two sources of truth for the same document. Every edit requires a round-trip through GitHub.

**Decision: Files on disk (hybrid model).**

Foundry stores docs as markdown files on the persistent Fly volume (`/data/docs/`). A SQLite `docs_meta` table tracks metadata per document. This gives us the best of both worlds — filesystem for content (Astro and Anvil already read from disk), SQLite for queryable metadata.

**Architecture:**

```
Current:
  GitHub repo → SSH clone → /app/content-repo/docs/ → Astro SSR renders
  Annotations → SQLite at ./foundry.db (separate DB)

After E11:
  /data/docs/ → Astro SSR renders (persistent Fly volume)
  /data/foundry.db → SQLite (annotations + reviews + docs_meta)
  GitHub → optional push-based sync (backup)
```

#### Data Model: docs_meta

```sql
CREATE TABLE docs_meta (
  path TEXT PRIMARY KEY,            -- canonical path (e.g., 'methodology/process')
  title TEXT NOT NULL,               -- extracted from first H1 or frontmatter
  access TEXT DEFAULT 'public',      -- 'public' or 'private' (derived from .access.json)
  content_hash TEXT NOT NULL,        -- SHA-256 of file content (for change detection)
  modified_at TEXT NOT NULL,         -- ISO 8601
  modified_by TEXT DEFAULT 'system', -- user_id from auth token
  created_at TEXT NOT NULL           -- ISO 8601
);
```

**Column notes:**
- `path` as PK — simpler than UUID, no joins needed. Renames are rare enough that delete+insert is fine.
- `access` is derived from `.access.json` prefix rules during import/create. `.access.json` remains the authority for v1. Migrating to per-doc access metadata is E12+ scope (ties into auth redesign).
- No `frontmatter` JSON column for v1 — title extracted from H1, access from `.access.json`. Custom frontmatter is an additive change later.
- `content_hash` enables future optimistic locking and staleness detection.

**Key decisions (LOCKED):**
- ✅ **Files on disk + SQLite metadata** — not BLOBs, not files-only
- ✅ **`FOUNDRY_DB_PATH=/data/foundry.db`** — annotations, reviews, AND docs_meta all in one DB on the persistent Fly volume. Current default of `./foundry.db` (app directory) gets wiped on deploy.
- ✅ **Fly volume persistence** — `/data/` persists across deploys and machine restarts. Only risk is volume deletion.
- ✅ **Cold start handling** — S1 creates `/data/docs/` directory structure on first boot if volume is empty.
- ✅ **Initial seeding via CLI + MCP tool** — `foundry import --repo <url>` for humans, `foundry.import_repo(repo, branch, path)` MCP tool for agents. Both share same core logic.
- ✅ **Path as PK** — no UUID, simpler queries, renames handled as delete+insert.
- ✅ **`.access.json` stays as access authority** — docs_meta.access is derived from it.

### SS2: GitHub Sync (Optional)

**Problem:** Losing Git history and backup safety is a valid concern.

**Decision: On-demand sync first.**

Foundry can optionally push content snapshots to a configured GitHub repo. Since files persist on the Fly volume, this is a backup mechanism, not a save mechanism.

**Implementation — no symlinks, copy to temp staging dir:**

1. Create `/tmp/foundry-sync/` (or reuse if exists)
2. `git init` (or reuse existing repo)
3. Copy all files from `/data/docs/` into the staging dir
4. `git add -A && git commit -m "Foundry sync <timestamp>"`
5. `git push --force` to the configured remote

`/data/docs/` stays clean — no `.git` directory, no git metadata. The staging dir is transient. If the push fails, `/data/docs/` is untouched.

**Why not serve from the git directory directly?**
- Anvil indexes recursively — `.git/` folder has hundreds of internal files that would contaminate the search index
- Astro's `[...slug].astro` route maps filesystem paths to URLs — `.git/` files could match slug patterns
- Disk usage — git history grows over time on a limited Fly volume
- Separation of concerns — `/data/docs/` is the content source of truth, git is a backup mechanism

**Auth:** SSH deploy key with write access on the csdlc-docs repo. Same mechanism as current pull setup, just need write permission enabled. Key path and remote URL stored in env vars (YAML config deferred to E12 with auth).

**What syncs:**
- Doc content (markdown files)
- NOT annotations, reviews, or user data (those stay in Foundry)

**Key decisions (LOCKED):**
- ✅ **On-demand first** — manual trigger via MCP tool or API endpoint
- ✅ **No symlinks** — copy to temp staging dir, force push, keep `/data/docs/` clean
- ✅ **SSH deploy key with write access** for push auth
- ✅ **GitHub is the right target** for v1
- ✅ **Foundry always wins** on conflict (force push)
- ✅ **Keep inbound clone for migration** — one-time import, then remove clone dependency from startup

### SS3: MCP Section-Level CRUD

**Problem:** Foundry's MCP tools are read-only for content. The annotation workflow works well, but updating docs after resolving comments requires going outside Foundry.

**Decision:** Four section-level CRUD tools that match Foundry's existing mental model.

**Why section-level, not element-level:**
- A word-processor API (`create_h2`, `add_paragraph`) would require 5-10 tool calls per logical edit
- The AI already knows how to write markdown — it just needs Foundry to handle *where* to put it
- Matches the existing read model: `get_section` reads by heading path, these tools write by heading path

#### API Routes & MCP Tools

| HTTP Route | Method | MCP Tool | Purpose |
|------------|--------|----------|---------|
| `/api/docs` | POST | `create_doc` | Create a new document at a path |
| `/api/docs/:path/sections` | PUT | `update_section` | Replace section content by heading path |
| `/api/docs/:path/sections` | POST | `insert_section` | Insert new section after a heading |
| `/api/docs/:path/sections` | DELETE | `delete_section` | Remove a section by heading path |

**Implementation architecture (MCP → HTTP → disk):**

The MCP server delegates ALL tool calls via HTTP to Express API routes (existing pattern in `server.ts` / `http-client.ts`). The CRUD tools follow this same pattern:

1. **Express API routes** live in `packages/api/src/routes/docs.ts` — all disk I/O, docs_meta updates, cache invalidation, and Anvil reindex logic lives here.
2. **MCP tool definitions** in `server.ts` call these routes via `http-client.ts`.
3. **After each write**, the Express route must:
   - Write to disk (`/data/docs/`)
   - Update `docs_meta` (content_hash, modified_at, modified_by)
   - Trigger Anvil delta reindex for the affected file
   - Invalidate Astro page cache (POST to `/api/invalidate-cache.json` or call `invalidateAll()` directly)

Cache invalidation is critical — without it, Astro serves stale content from its in-memory page cache (`page-cache.ts`) until restart.

#### Section Parsing Logic

`update_section` replaces **body content only** (not the heading). The heading is the address — you update what's under it, not the address itself. To rename a heading, use `delete_section` + `insert_section`.

Parsing steps:
1. Parse the markdown file into an AST (remark)
2. Find the heading matching the heading path
3. Identify section boundaries (from this heading to the next heading of equal or higher level)
4. Replace everything between those boundaries with the new content
5. Serialize back to markdown

#### Duplicate Heading Detection

On parse, build a `Map<string, number>` of heading paths → count. If the requested path matches a heading that appears more than once, return HTTP 400:

```
Error: Ambiguous heading path — found 2 matches for
'## Foo > ### Examples'. Rename one heading to disambiguate.
```

This is the caller's signal to either use a more specific path or ask the user to rename a heading. Full heading paths (`## Parent > ### Child`) should be unambiguous in well-structured docs.

#### Template Enforcement on create_doc

`create_doc` requires a `template` parameter:

```
create_doc(path, template, title?)
```

Known templates: `epic`, `subsystem`, `project`, `workflow`, `blank`

Templates are sourced from the existing CSDLC templates in the content directory. Using `blank` is an intentional opt-out — the default enforces the methodology. This prevents agents from creating unstructured docs by accident.

**Key decisions (LOCKED):**
- ✅ **Heading path format matches Anvil exactly** — e.g., `## Architecture > ### Tech Stack`
- ✅ **Heading path mismatch = HTTP 400 error** (not fuzzy match) — explicit is better
- ✅ **Duplicate heading = HTTP 400 error** with descriptive message
- ✅ **Body-only replacement** on `update_section` — heading is the address, not the content
- ✅ **Last-write-wins for concurrency** — we're the only two users. Optimistic locking with content_hash is the right future answer but out of scope. TODO comment in code.
- ✅ **Template required on create_doc** — `blank` is the intentional opt-out
- ✅ **Express routes first, MCP wiring second** — S5a establishes routes, S5b wires MCP tools

### SS4: Doc Path Normalization (Refactor)

**Problem:** Frontend stores URL-style paths (`/docs/Race%20Strategy/sub-systems/...`), MCP stores filesystem paths (`Race Strategy/sub-systems/...`). PR #71 added a bandaid (generate 4 variants on every query). This is brittle.

**Decision:** Normalize on write, not on read. Atomic migration.

**Canonical format (LOCKED):**
```
methodology/process
```
- No leading slash
- No trailing slash
- No `.md` extension
- No URL encoding (decoded)
- No `/docs/` prefix
- No `/foundry/` prefix
- Raw path relative to content root

**Implementation:**
1. `normalizeDocPath()` runs on every annotation/review CREATE
2. **Atomic migration** — normalize all existing `doc_path` values in annotations and reviews tables in a single transaction. Tested against a dump of prod data first.
3. All queries use single canonical comparison (no variants needed)
4. Frontend sends whatever it wants — the API normalizes before storage
5. Remove `docPathVariants()` bandaid after migration

**Migration safety:** During S0 codebase audit, dump all existing `doc_path` values, catalog formats, and test `normalizeDocPath()` against real data. This confirms atomic migration is safe before S4 executes.

**`.access.json` interaction:** Currently uses path prefixes like `"projects/"`. The Astro `[...slug].astro` route checks access via `checkAccess(slug)`. Since `.access.json` already uses the canonical format (no `/docs/` prefix, no encoding), normalization should not affect access control. Verified during S0.

---

## Story Breakdown

### Batch 1: Foundation (serial)

**S0: End-to-end plumbing verification + codebase audit**

Not a spike — a verification that our existing architecture supports `/data/docs/`. Astro already runs in SSR mode (`output: 'server'`) with a dynamic route at `[...slug].astro` that reads markdown from disk via `fs.readFileSync()` and respects the `CONTENT_DIR` env var.

*Acceptance criteria:*
- Set `CONTENT_DIR=/data/docs/`, copy test docs there, Astro serves them at `/docs/...`
- Modify a file on disk, hit `/api/invalidate-cache.json`, Astro serves the new version
- Anvil indexes from `/data/docs/` and search returns results
- No build step required after content changes

*Codebase audit (done simultaneously):*
- Document where `CONTENT_DIR` / `CONTENT_CLONE_DIR` / `CONTENT_REPO` are referenced
- Trace startup sequence in `index.ts`
- Map all ContentFetcher call sites
- Identify any drift between architecture docs and actual implementation
- Dump all existing `doc_path` values from annotations/reviews for SS4 migration prep
- Verify nav tree auto-generation logic (does adding a file auto-update nav?)
- Produce findings doc that interns reference for S1-S3

**S1: Native content storage**

`/data/docs/` filesystem + `docs_meta` SQLite table.

*Acceptance criteria:*
- `docs_meta` table created with schema from SS1 Data Model section
- `FOUNDRY_DB_PATH=/data/foundry.db` set in fly.toml
- `/data/docs/` directory created on cold start if missing
- Existing annotations/reviews DB migrated to `/data/foundry.db` if not already there

*Files to touch:*
- `packages/api/src/db.ts` — add `docs_meta` table creation
- `fly.toml` — update `FOUNDRY_DB_PATH`

**S2: Startup refactor — Remove ContentFetcher**

*Acceptance criteria:*
- Foundry boots without network access to GitHub
- No SSH keys required in production
- Cold start time drops significantly (no clone step)
- Anvil indexes from `/data/docs/` on startup
- Existing webhook endpoint returns 410 Gone (not 500)

*Files to modify/remove:*
- `packages/api/src/content-fetcher.ts` — remove or gut
- `packages/api/src/index.ts` — remove `contentFetcher.cloneOrPull()` on boot
- `packages/api/src/routes/` — remove webhook handler route
- `fly.toml` / `Dockerfile` — remove `DEPLOY_KEY_PATH`, `CONTENT_REPO`, `CONTENT_BRANCH` env vars

*What to keep:*
- `CONTENT_DIR` env var (now points to `/data/docs/`)
- `/api/invalidate-cache.json` endpoint (needed for CRUD cache busting)
- `/healthz` proxy endpoint (still needed)

**S3: Import CLI + MCP tool**

`foundry import --repo <url>` for humans, `foundry.import_repo()` for agents.

*Decisions (LOCKED):*
1. **Preserve directory structure** — strip configurable source prefix (default: `docs/`). `docs/methodology/process.md` → `/data/docs/methodology/process.md`.
2. **Populate docs_meta** — scan each file, extract title from first H1 (fall back to filename), compute SHA-256 content_hash, derive access from `.access.json` prefix rules, INSERT OR REPLACE.
3. **Full Anvil reindex** after all files copied. One reindex, not per-file.
4. **`.access.json` copied** to `/data/.access.json` — config file, not a doc.
5. **Idempotent** — safe to run multiple times. Overwrites files, upserts docs_meta.
6. **CLI entry point** — `packages/api/src/cli.ts`, simple script importing shared modules from API package.

*Acceptance criteria:*
- `foundry import --repo https://github.com/danhannah94/csdlc-docs --prefix docs/` populates `/data/docs/` and `docs_meta`
- MCP tool `import_repo(repo, branch, path)` does the same
- Running import twice produces identical results (idempotent)
- All imported docs appear in search after Anvil reindex

### Batch 2: CRUD + Normalization (serial then parallel)

**S4: Doc path normalization**

Canonical format, atomic migration, remove `docPathVariants()`.

*Acceptance criteria:*
- `normalizeDocPath()` function exists and is called on all annotation/review creates
- Migration script normalizes all existing rows in one transaction
- All queries use single canonical comparison
- `docPathVariants()` removed from codebase
- `.access.json` prefix matching still works (verified)

**S5a: Express doc CRUD routes**

Establish `packages/api/src/routes/docs.ts` with all 4 endpoints.

*Acceptance criteria:*
- `POST /api/docs` — creates doc with template, populates docs_meta, triggers Anvil reindex, invalidates Astro cache
- `PUT /api/docs/:path/sections` — updates section body by heading path
- `POST /api/docs/:path/sections` — inserts new section after heading
- `DELETE /api/docs/:path/sections` — removes section by heading path
- All endpoints protected by existing `requireAuth()` middleware
- All endpoints return proper errors (400 for ambiguous headings, 404 for missing docs/sections)
- Testable with curl/httpie before MCP wiring

**S5b: MCP tool wiring**

Register `create_doc`, `update_section`, `insert_section`, `delete_section` in `server.ts`. Add http-client methods.

*Acceptance criteria:*
- All 4 tools callable via MCP stdio transport
- Template parameter required on `create_doc` (with `blank` as opt-out)
- Tools return structured responses matching existing MCP tool patterns

**S6 + S7: Section manipulation tools (parallel after S5)**

S6: `update_section` + S7: `insert_section` / `delete_section` — these build on the S5a scaffold. Can run in parallel since they're adding to existing routes.

*Note: S5 establishes the route file and pattern. S6+S7 add tools to it.*

### Batch 3: Sync + Polish

**S8: GitHub sync**

On-demand push via MCP tool + API endpoint.

*Acceptance criteria:*
- `sync_to_github` MCP tool pushes content to configured repo
- `POST /api/sync` endpoint does the same
- Uses temp staging dir (`/tmp/foundry-sync/`), not `/data/docs/`
- SSH deploy key with write access
- Force push (Foundry always wins)
- Descriptive commit message with timestamp

**S9: Skill, config, and deploy cleanup**

*Scope:*
- **Foundry SKILL.md rewrite** — update from "GitHub-backed read-only" to "native content platform with CRUD." This is what future Clay sessions use to know Foundry's capabilities.
- **CC native MCP config** — configure Foundry MCP server as Claude Code stdio transport in settings. Remove mcporter config dependency. Document: `node packages/api/dist/mcp/stdio.js`.
- **Deploy workflow consolidation** — merge `deploy.yml` and `deploy-fly.yml` into one workflow. Fixes duplicate runs + Fly lease conflicts.
- **Remove `tools/annotations.ts`** dead code — `server.ts` is the real MCP registry. Prevents intern confusion.
- **Add `js-yaml` as explicit dependency** (currently implicit).

**S10: Integration testing**

End-to-end: import docs → create doc (with template) → edit section → annotate → resolve → sync to GitHub → verify search → verify nav.

---

## Dependencies

- E10 shipped ✅ (UI polish, search, settings)
- E10.5 shipped ✅ (7 PRs #84-#90 merged, all items landed)
- PR #71 merged ✅ (doc_path bandaid — replaced by SS4)
- PR #80 merged ✅ (user_id fix)
- PR #82 merged ✅ (quoted_text on MCP create_annotation)
- Anvil v0.1.11 ✅ (reindexFiles for delta reindexing — used by SS3 CRUD)
- `/healthz` proxy fix ✅ (prevents health check flapping during Anvil init)
- **`js-yaml` package** — needed for future config parsing, add as explicit dependency in S9
- **`tools/annotations.ts` cleanup** — dead code that could confuse interns. Remove in S9 before Batch 2 CRUD work.

---

## Risks

| Risk | Likelihood | Mitigation |
|------|-----------|-----------|
| Astro can't serve runtime content from `/data/docs/` | **Low** (codebase already supports SSR from disk) | **S0** verifies this first |
| Data loss during migration from GitHub → native | Medium | Keep GitHub clone capability as fallback; test migration thoroughly; GitHub sync as backup |
| Fly.io volume persistence | Low | Fly volumes persist across deploys; GitHub sync is the backup safety net |
| Breaking existing annotation data during path normalization | Medium | Atomic migration in single transaction; test against prod data dump in S0 |
| Scope creep into full CMS | Low | Strict scope: markdown CRUD only, no WYSIWYG, no real-time collab, no admin panel, auth punted to E12 |
| Anvil event loop blocking during re-index after edits | Low | Delta reindex (single file) should be fast; only full reindex is slow |
| **Nav tree out of sync with content** | Medium | Verify auto-generation during S0; add nav check to S5 acceptance criteria |
| **Anvil index drift** (write succeeds but reindex fails) | Medium | `reindex` MCP tool for recovery; add content_hash comparison to `get_status` for staleness detection |

---

## Decisions Log

| # | Question | Decision | Decided |
|---|----------|----------|---------|
| 1 | Files on disk vs SQLite BLOBs? | **Files on disk + SQLite docs_meta** | April 6 |
| 2 | GitHub sync mode? | **On-demand first** (MCP tool / API) | April 6 |
| 3 | Heading path format for CRUD? | **Match Anvil exactly** | April 6 |
| 4 | First-deploy seeding? | **CLI + MCP tool** (import from GitHub) | April 6 |
| 5 | Concurrency handling? | **Last-write-wins** (optimistic locking future) | April 6 |
| 6 | Doc path canonical format? | **No prefix, no suffix, no encoding** | April 6 |
| 7 | SS4 batch placement? | **Moved to Batch 2** (not Batch 1) | April 6 |
| 8 | Human editing UI? | **Out of scope** (agents edit, humans review) | April 6 |
| 9 | Auth & multi-user? | **Punted to E12** — needs its own design doc | April 7 |
| 10 | docs_meta PK? | **Path as PK** (not UUID) | April 7 |
| 11 | Access authority? | **`.access.json` stays** for v1, per-doc access in E12 | April 7 |
| 12 | DB path? | **`/data/foundry.db`** on Fly volume | April 7 |
| 13 | GitHub sync mechanism? | **Copy to temp staging dir, force push** (no symlinks) | April 7 |
| 14 | GitHub push auth? | **SSH deploy key with write access** | April 7 |
| 15 | update_section semantics? | **Body-only replacement** (heading is the address) | April 7 |
| 16 | Duplicate headings? | **HTTP 400 error** with descriptive message | April 7 |
| 17 | create_doc templates? | **Required param** (`blank` as intentional opt-out) | April 7 |
| 18 | Express routes vs MCP? | **Separate stories** (S5a routes, S5b MCP wiring) | April 7 |
| 19 | MCP transport? | **CC native stdio** (not mcporter) | April 7 |
| 20 | S0 framing? | **Plumbing verification + audit** (not generic spike) | April 7 |
| 21 | Import behavior? | **Preserve dirs, strip prefix, populate docs_meta, idempotent** | April 7 |

---

## Related

- [Issue #78: Foundry-native content storage](https://github.com/danhannah94/foundry/issues/78)
- [Issue #79: MCP section-level CRUD tools](https://github.com/danhannah94/foundry/issues/79)
- [Issue #81: MCP quoted_text param](https://github.com/danhannah94/foundry/issues/81) ✅ (PR #82, merged)
- [E10: UI/UX Polish Part 2](e10-ui-ux-part2.md)
- [E10.5: UI Bugfixes & Polish](e10.5-ui-bugfixes.md)
- [E8: Live Content Pipeline](e8-live-content-pipeline.md)
- **E12: Auth & Multi-User** — design doc TBD (punted from E11)
