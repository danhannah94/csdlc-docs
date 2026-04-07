# E9: MCP Agent Surface — Design Doc

*Status: Step 1 — Stories Locked*
*Created: April 2, 2026*
*Authors: Clay (draft), Dan (review)*
*Refined via Foundry review loop — 9 threads, 14+ replies*

---

## Overview

E9 establishes MCP as the **single agent interface** to Foundry. Today, agents interact with Foundry through a mix of HTTP curl commands (via the Foundry skill) and MCP tools (via mcporter). This creates confusion about which surface to use, makes the skill brittle (hardcoded URLs, manual auth headers), and means there's no way to programmatically test the review lifecycle without a human in a browser.

After E9, the rule is simple: **agents use MCP tools. Browsers use HTTP. Nothing crosses the line.**

### Why Now

Three drivers:

1. **The skill is wrong.** `skills/foundry/SKILL.md` documents HTTP endpoints with curl patterns. Every time Clay replies to annotations, it's raw HTTP. This works but it's fragile — auth tokens in shell commands, URL encoding, no error recovery. MCP tools handle all of that internally.

2. **We can't test what we build.** The Foundry review loop is our flagship feature, but the only way to verify it end-to-end is Dan on an iPad clicking things. With a complete MCP tool surface, Clay can run the full lifecycle — create review, add comments, submit, reply, resolve, delete — all via `mcporter call`. That's integration testing without a browser.

3. **Missing operations.** There's no way to delete an annotation (the test reply from E4 is still stuck in the DB — noted in tech debt). There's no MCP tool to edit annotation content. No way to list or inspect reviews. These gaps block both testing and normal workflow.

### Scope

- New MCP tools (delete, edit, review operations, single-annotation get, nav listing)
- Corresponding HTTP endpoints where missing (DELETE route, GET by ID)
- Rewritten Foundry skill (mcporter-only, no HTTP)
- No frontend changes (except delete confirmation dialog)
- No schema changes (existing DB supports all operations)

---

## Architecture: How MCP Connects to Deployed Foundry

Understanding the flow is critical — the MCP server does NOT require Foundry running locally:

```
┌─ Your Machine ──────────────────────────────────────┐
│                                                      │
│  Agent (Clay)                                        │
│    │                                                 │
│    ▼                                                 │
│  mcporter ──stdio──▶ Foundry MCP Server (local)     │
│                        │                             │
│                        │ HTTP (http-client.ts)       │
│                        │ FOUNDRY_API_URL env var     │
│                        │ FOUNDRY_WRITE_TOKEN env var │
└────────────────────────┼─────────────────────────────┘
                         │
                         ▼  HTTPS
              ┌─ Fly.io ─────────────┐
              │  Foundry HTTP API     │
              │  (Express + SQLite)   │
              └───────────────────────┘
```

1. mcporter spawns the MCP server as a **local child process** (stdio transport — no network listener, no port)
2. MCP server is a thin HTTP client — reads `FOUNDRY_API_URL` (defaults to `https://foundry-claymore.fly.dev`) from env
3. Agent calls `mcporter call foundry.list_annotations` → MCP server makes HTTPS request to Fly.io → returns result via stdio
4. Auth token is in mcporter config, never in shell commands or process lists

**Security:** stdio transport = local only. No external agent can connect to the MCP server. The Bearer token protects the HTTP API. MCP is actually MORE secure than the current curl-based approach where tokens appear in shell history.

### The API Boundary Principle

```
MCP tools (agent surface) → HTTP API (business logic) → SQLite DB (storage)
                             ↑
Browser/Frontend ────────────┘
```

- **MCP → HTTP → DB.** MCP tools call the HTTP API via `http-client.ts`. Never touch the DB directly.
- HTTP API is the single source of truth for business logic (validation, deduplication, review_id inheritance)
- MCP tools are thin wrappers that translate tool schemas → HTTP calls → structured responses
- Frontend and agents share the same validation/logic path
- Adding a new MCP tool = add function to `http-client.ts` + register tool in the appropriate tools file

### Why Not MCP → DB Direct?

Established in E4, proven through E7. The HTTP layer handles:
- BUG-4 idempotency (30-second dedup window)
- BUG-6 review_id inheritance from parent
- Input validation (required fields, status transitions)
- Auth (Bearer token on writes)

Duplicating this in MCP tools would be a maintenance nightmare. One path, one set of rules.

---

## Current Tool Inventory

### MCP Tools (exist today)

| Tool | What It Does | Gaps |
|------|-------------|------|
| `list_annotations` | List annotations filtered by doc_path, section, status | No pagination, no review_id filter |
| `create_annotation` | Create annotation or reply (with parent_id) | Works well, no changes needed |
| `resolve_annotation` | Set status to "resolved" | No inverse (reopen) via MCP |
| `submit_review` | Create review + batch-update annotations | Complex, works, no changes needed |
| `search_docs` | Semantic search via Anvil | Separate concern, no changes in E9 |

### HTTP Routes (exist today)

| Route | MCP Equivalent | Status |
|-------|---------------|--------|
| `GET /api/annotations` | `list_annotations` | ✅ Covered |
| `POST /api/annotations` | `create_annotation` | ✅ Covered |
| `PATCH /api/annotations/:id` | `resolve_annotation` (partial) | ⚠️ Only resolve exposed, not edit/reopen |
| `DELETE /api/annotations/:id` | — | ❌ Route doesn't exist |
| `GET /api/reviews` | — | ❌ Not exposed via MCP |
| `POST /api/reviews` | `submit_review` (internal) | ✅ Used internally by submit_review |
| `PATCH /api/reviews/:id` | `submit_review` (internal) | ✅ Used internally by submit_review |
| `GET /api/search` | `search_docs` | ✅ Covered |
| `GET /api/access` | — | ❌ Not exposed via MCP (low priority) |

---

## Stories

### FND-E9-S1: `delete_annotation`

**Why:** Can't clean up test data, stale annotations, or mistakes. The test reply from E4 is literally still in prod because there's no delete.

**MCP Schema:**
```json
{
  "name": "delete_annotation",
  "description": "Delete an annotation by ID. If the annotation has child replies, all replies are cascade-deleted.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "annotation_id": { "type": "string", "description": "ID of the annotation to delete" }
    },
    "required": ["annotation_id"]
  }
}
```

**HTTP Route:** `DELETE /api/annotations/:id`
- Returns 204 on success
- Returns 404 if not found
- **Cascade deletes** all child replies (parent_id = this ID)
- Cascading delete of orphaned reviews (if last annotation in a review is deleted, delete the review)
- Frontend shows confirmation dialog: "Delete this thread? X replies will also be deleted"

**http-client.ts:** `deleteAnnotation(annotationId: string) → { status: 'deleted' | 'error', message?: string }`

**Acceptance Criteria:**
- [ ] `DELETE /api/annotations/:id` returns 204 and removes annotation from DB
- [ ] Child replies are cascade-deleted
- [ ] Orphaned reviews are cleaned up
- [ ] MCP tool `delete_annotation` calls HTTP endpoint correctly
- [ ] Returns 404 for non-existent IDs
- [ ] Tests cover: single delete, cascade delete, orphan review cleanup, 404

### FND-E9-S2: `edit_annotation`

**Why:** Typos happen. Both humans and AI should be able to fix annotation content without delete/recreate.

**MCP Schema:**
```json
{
  "name": "edit_annotation",
  "description": "Edit the content of an existing annotation.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "annotation_id": { "type": "string", "description": "ID of the annotation to edit" },
      "content": { "type": "string", "description": "New content for the annotation" }
    },
    "required": ["annotation_id", "content"]
  }
}
```

**HTTP Route:** Already exists — `PATCH /api/annotations/:id` with `{ content: "..." }`. Just needs the MCP wrapper.

**http-client.ts:** `editAnnotation(annotationId: string, content: string) → Annotation`

**Acceptance Criteria:**
- [ ] MCP tool `edit_annotation` updates annotation content via PATCH
- [ ] Returns updated annotation object
- [ ] Returns 404 for non-existent IDs
- [ ] `updated_at` timestamp is refreshed
- [ ] Tests cover: successful edit, 404

### FND-E9-S3: `reopen_annotation`

**Why:** The browser UI already supports reopening from the archive. This exposes the same capability to agents via MCP.

**MCP Schema:**
```json
{
  "name": "reopen_annotation",
  "description": "Reopen a previously resolved annotation.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "annotation_id": { "type": "string", "description": "ID of the annotation to reopen" }
    },
    "required": ["annotation_id"]
  }
}
```

**HTTP Route:** Already exists — `PATCH /api/annotations/:id` with `{ status: "submitted" }`. MCP wrapper only.

**http-client.ts:** `reopenAnnotation(annotationId: string) → Annotation`

**Acceptance Criteria:**
- [ ] MCP tool `reopen_annotation` sets status to "submitted" via PATCH
- [ ] Returns updated annotation
- [ ] Returns 404 for non-existent IDs
- [ ] Tests cover: reopen resolved annotation, 404

### FND-E9-S4: `get_annotation`

**Why:** Enables lookup by ID with reply thread. Useful for verifying created annotations, drilling down from review listings, and future notification workflows.

**MCP Schema:**
```json
{
  "name": "get_annotation",
  "description": "Get a single annotation by ID, including its reply thread.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "annotation_id": { "type": "string", "description": "ID of the annotation" }
    },
    "required": ["annotation_id"]
  }
}
```

**HTTP Route:** `GET /api/annotations/:id` (new — currently only list endpoint exists)
- Returns the annotation plus an array of child replies (parent_id = this ID)
- Returns 404 if not found

**http-client.ts:** `getAnnotation(annotationId: string) → { annotation: Annotation, replies: Annotation[] }`

**Acceptance Criteria:**
- [ ] `GET /api/annotations/:id` returns annotation + replies array
- [ ] Replies are sorted by created_at ASC (chronological thread order)
- [ ] Returns 404 for non-existent IDs
- [ ] MCP tool calls HTTP endpoint correctly
- [ ] Tests cover: get with replies, get without replies, 404

### FND-E9-S5: `list_reviews` / `get_review`

**Why:** Reviews exist in the DB but agents can only interact with them through `submit_review`. Can't list past reviews, check review status, or inspect what's in a review.

**MCP Schemas:**

**list_reviews:**
```json
{
  "name": "list_reviews",
  "description": "List reviews for a document, optionally filtered by status.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "doc_path": { "type": "string", "description": "Path to the document" },
      "status": { "type": "string", "description": "Optional status filter (draft, submitted, completed)" }
    },
    "required": ["doc_path"]
  }
}
```

**get_review:**
```json
{
  "name": "get_review",
  "description": "Get a review by ID with all its annotations.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "review_id": { "type": "string", "description": "ID of the review" }
    },
    "required": ["review_id"]
  }
}
```

**HTTP Routes:**
- `GET /api/reviews` already exists (list)
- `GET /api/reviews/:id` — new, returns review + associated annotations

**http-client.ts:** `listReviews(docPath, status?) → Review[]` and `getReview(reviewId) → { review: Review, annotations: Annotation[] }`

**Acceptance Criteria:**
- [ ] `list_reviews` returns reviews filtered by doc_path and optional status
- [ ] `get_review` returns review object + all associated annotations
- [ ] `GET /api/reviews/:id` returns 404 for non-existent IDs
- [ ] MCP tools call HTTP endpoints correctly
- [ ] Tests cover: list with filters, get with annotations, 404

### FND-E9-S6: Skill Rewrite (mcporter-only)

**Why:** The current `skills/foundry/SKILL.md` documents HTTP endpoints with curl commands. After E9, it should exclusively use `mcporter call foundry.*` commands. Agents should never know about the HTTP API.

**Changes:**
- Replace all curl examples with `mcporter call foundry.<tool>` equivalents
- Document all tools (existing + new from E9)
- Add workflow recipes: "How to reply to comments", "How to run a full review cycle", "How to clean up test data"
- Remove BASE_URL/TOKEN setup (mcporter handles auth via config)
- Document cross-tool pattern: "For doc content, use `mcporter call anvil.get_page`. For annotations, use `mcporter call foundry.*`"

**Depends on:** S1-S5, S7 (needs all tools to document)

**Acceptance Criteria:**
- [ ] No HTTP URLs, curl commands, or token references in SKILL.md
- [ ] All MCP tools documented with examples
- [ ] Workflow recipes for common patterns (reply, review, cleanup)
- [ ] Cross-tool pattern with Anvil documented

### FND-E9-S7: `list_annotations` Enhancement — Review ID Filter

**Why:** When inspecting a specific review's annotations, you currently have to list all annotations and filter client-side. Adding `review_id` as a filter parameter is trivial and useful.

**Changes:**
- Add optional `review_id` parameter to `list_annotations` MCP tool schema
- Add `review_id` query param to `GET /api/annotations` HTTP route
- Update http-client.ts `listAnnotations` signature

**Acceptance Criteria:**
- [ ] `list_annotations` accepts optional `review_id` parameter
- [ ] `GET /api/annotations?review_id=xxx` filters correctly
- [ ] Can combine with existing filters (doc_path + review_id + status)
- [ ] Tests cover: filter by review_id alone, combined filters

### FND-E9-S8: `list_pages` (Nav Tree)

**Why:** Agents need to know what docs exist without guessing paths. Enables navigation of the doc tree programmatically.

**MCP Schema:**
```json
{
  "name": "list_pages",
  "description": "List all pages in the Foundry nav tree with their paths and access levels.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "include_private": { "type": "boolean", "description": "Include private pages (requires auth). Defaults to false." }
    }
  }
}
```

**Implementation:** Read `nav.yaml` at startup, merge with access config. Return flat list of `{ title, path, access }` entries. HTTP route: `GET /api/pages`.

**Note:** Anvil has `list_pages` for indexed content. This Foundry tool includes access levels and matches the nav tree (which may include pages not yet indexed by Anvil).

**http-client.ts:** `listPages(includePrivate?: boolean) → Page[]`

**Acceptance Criteria:**
- [ ] `list_pages` returns all pages from nav tree
- [ ] Each entry includes title, path, and access level (public/private)
- [ ] `include_private=false` (default) excludes private pages
- [ ] `include_private=true` returns all pages (requires auth)
- [ ] Tests cover: public only, all pages, empty nav

---

## Code Organization

During implementation, split the current monolithic `tools/annotations.ts` into:
- `tools/annotation-tools.ts` — CRUD on annotations (list, get, create, edit, delete, reopen)
- `tools/review-tools.ts` — Review lifecycle (list, get, submit)
- `tools/nav-tools.ts` — Navigation (list pages)
- `tools/search.ts` — Already separate, stays as-is

Each file exports a `register*Tools(server)` function. `server.ts` calls all of them. This is internal refactoring, not a separate story.

---

## Agent Interaction Patterns

### Pattern 1: Reply to Human Comments (Current Workflow, Improved)

```
mcporter call foundry.list_annotations doc_path="/foundry/docs/..." status="submitted"
  → Find annotations with author_type="human" and no AI reply
mcporter call foundry.create_annotation doc_path="..." section="..." content="..." parent_id="<human-annotation-id>"
```

### Pattern 2: Full Review Lifecycle (Integration Testing)

```
# Create a review with comments
mcporter call foundry.submit_review doc_path="/foundry/docs/..."

# Verify it was created
mcporter call foundry.list_reviews doc_path="/foundry/docs/..."

# Check specific review
mcporter call foundry.get_review review_id="<id>"

# Reply to a comment
mcporter call foundry.create_annotation ... parent_id="<id>"

# Resolve a thread
mcporter call foundry.resolve_annotation annotation_id="<id>"

# Reopen if needed
mcporter call foundry.reopen_annotation annotation_id="<id>"

# Clean up test data
mcporter call foundry.delete_annotation annotation_id="<id>"
```

### Pattern 3: Annotation Cleanup

```
# Find stale annotations
mcporter call foundry.list_annotations doc_path="..." status="draft"

# Delete them
mcporter call foundry.delete_annotation annotation_id="<id>"
```

### Pattern 4: Cross-Tool Content + Review

```
# Get doc content (Anvil)
mcporter call anvil.get_page path="projects/foundry/design"

# Review annotations on that doc (Foundry)
mcporter call foundry.list_annotations doc_path="/foundry/docs/projects/foundry/design/"
```

---

## Story Summary

| Story | Title | Scope | Deps |
|-------|-------|-------|------|
| FND-E9-S1 | Delete annotation | HTTP route + MCP tool + http-client + cascade | None |
| FND-E9-S2 | Edit annotation | MCP tool + http-client (HTTP route exists) | None |
| FND-E9-S3 | Reopen annotation | MCP tool + http-client (HTTP route exists) | None |
| FND-E9-S4 | Get single annotation | HTTP route + MCP tool + http-client | None |
| FND-E9-S5 | List/get reviews | HTTP route (get by ID) + 2 MCP tools + http-client | None |
| FND-E9-S6 | Skill rewrite | Replace SKILL.md with mcporter-only | S1-S5, S7-S8 |
| FND-E9-S7 | list_annotations review_id filter | HTTP param + MCP param + http-client | None |
| FND-E9-S8 | List pages (nav tree) | HTTP route + MCP tool + http-client | None |

**Execution plan:** S1-S5 + S7-S8 are all independent — parallel Lightning Strike. S6 is the capstone after everything else is merged and tested.

---

## Design Decisions

| # | Decision | Rationale | Status |
|---|----------|-----------|--------|
| D1 | MCP = agent surface, HTTP = browser surface | Clean separation, single source of business logic in HTTP layer | ✅ Decided |
| D2 | MCP → HTTP → DB (no direct DB from MCP) | Reuse validation, dedup, auth logic. Established in E4, proven. | ✅ Decided |
| D3 | Cascade delete with frontend confirmation | Simple, no force flags. Frontend shows "Delete thread? X replies will be deleted." API just deletes. | ✅ Decided |
| D4 | Split tool registration into separate files | annotation-tools.ts, review-tools.ts, nav-tools.ts, search.ts. Internal refactor during implementation. | ✅ Decided |
| D5 | review_id filter on list_annotations | Trivial to add, useful for review inspection | ✅ Decided |
| D6 | Doc content via Anvil, not Foundry | Anvil already has `get_page` and `get_section`. Foundry owns review layer, Anvil owns content. | ✅ Decided |
| D7 | MCP stdio = local only, no external access | Auth token in mcporter config env, never in shell commands. More secure than curl approach. | ✅ Decided |
| D8 | Visual regression testing = separate project | `@claymore-dev` package, not E9 scope. Playwright-based, works on any URL. Future project design doc. | ✅ Decided (parked) |
| D9 | list_pages includes access levels | Distinct from Anvil's list_pages — includes nav tree structure + public/private access info | ✅ Decided |

---

## Resolved from Refinement

These questions were raised during the Foundry review loop and resolved:

1. **Cascade delete behavior** → Cascade always, frontend confirms. (Dan: "just do a cascade delete with an alert")
2. **get_doc_content tool** → Use Anvil's `get_page`/`get_section` instead. Foundry = review layer, Anvil = content layer.
3. **Tool file split** → Yes, split during implementation. No impact on agents — purely code organization.
4. **Reopen is "new"** → The browser UI already has it. S3 just exposes it to agents via MCP. Smaller story than originally scoped.
5. **Reply notifications** → Future feature, not E9. But `get_annotation` (S4) is still useful for verification and drill-down workflows.
6. **MCP security** → stdio transport = local process only. No external agent can connect. Token in config, not in commands.
7. **Nav tool** → Added as S8. Distinct from Anvil's `list_pages` because it includes access levels.
8. **Reply button at bottom of thread** → E7 UI polish item, not E9 scope. Added to NEXT.md.
9. **Visual testing MCP package** → Big idea with legs (`@claymore-dev/lookout` or similar). Separate project, not E9.

---

*Refined through the Foundry review loop. Ready for execution.*
