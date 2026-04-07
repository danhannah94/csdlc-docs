# E4: Annotations + Unified Review Thread

*Status: Step 1 Complete — Ready for Step 2 (Agent Prompt Crafting)*
*Epic: Foundry v0.2*
*Created: March 30, 2026*
*Updated: March 31, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

The meatiest Foundry epic yet. Add a full inline annotation system: highlight text, add comments in a unified doc-scoped thread panel, batch-submit reviews to the OpenClaw main session, and receive AI replies threaded back into Foundry. Includes orphan detection when docs change.

### Problem Statement

Currently, doc feedback happens via copy-paste into Telegram — losing context about which section, which text, and which doc. Comments get buried in chat history. There's no way to:

- Annotate a specific section and have that context travel with the comment
- Batch multiple comments into a structured review
- Get AI replies anchored to the original comment
- Track what's been addressed vs. what's still open

Foundry annotations capture precise context, batch it for refinement, and close the feedback loop.

### Goals

- Text highlight → comment in unified thread panel
- Comments stored in SQLite (local) with heading path + content hash anchoring
- Unified doc-scoped thread panel (~35% right, ~65% doc)
- Batch "Submit Review" → structured payload to OpenClaw main session via MCP
- AI replies threaded under parent comments via MCP `create_annotation`
- Orphan detection when doc headings change (collapsed section, nothing silently deleted)
- Comment lifecycle: draft → submitted → replied → resolved → orphaned

### Non-Goals (Deferred to Post-E4)

- Foundry OpenClaw skill (fast-follow after E4 ships)
- Re-anchor UI for orphaned comments
- Orphan visual polish (strikethrough/dim — iterate after seeing real data)
- Real-time reply push via SSE (currently refresh-to-see)
- Multi-user auth / GitHub OAuth (hardcoded `user_id='dan'` for MVP)
- Shared/collaborative annotations (needs Supabase)
- Accept/reject workflow UI

---

## Architecture

### Data Model

#### `annotations` Table

```sql
CREATE TABLE annotations (
  id TEXT PRIMARY KEY,             -- cuid2
  doc_path TEXT NOT NULL,
  heading_path TEXT NOT NULL,      -- e.g., "## Architecture > ### Tech Stack"
  content_hash TEXT NOT NULL,      -- SHA-256 of section text at annotation time
  quoted_text TEXT,                -- selected text (nullable for general section comments)
  content TEXT NOT NULL,           -- comment body
  parent_id TEXT,                  -- FK to annotations.id (threading)
  review_id TEXT,                  -- FK to reviews.id (batch grouping)
  user_id TEXT DEFAULT 'dan',      -- nullable for MVP, required for multi-user
  author_type TEXT NOT NULL DEFAULT 'human',  -- 'human' | 'ai'
  status TEXT NOT NULL DEFAULT 'draft',       -- draft | submitted | replied | resolved | orphaned
  created_at TEXT NOT NULL,        -- ISO 8601
  updated_at TEXT NOT NULL,        -- ISO 8601
  FOREIGN KEY (parent_id) REFERENCES annotations(id),
  FOREIGN KEY (review_id) REFERENCES reviews(id)
);
```

#### `reviews` Table

```sql
CREATE TABLE reviews (
  id TEXT PRIMARY KEY,             -- cuid2
  doc_path TEXT NOT NULL,
  user_id TEXT DEFAULT 'dan',
  status TEXT NOT NULL DEFAULT 'draft',  -- draft | submitted | complete
  submitted_at TEXT,               -- ISO 8601, null until submitted
  completed_at TEXT,               -- ISO 8601, null until all comments resolved
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

### API Endpoints (E4 Additions)

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/annotations?doc_path=X` | List annotations for a doc (with optional section filter) |
| `POST` | `/annotations` | Create annotation (used by both UI submit and AI reply) |
| `PATCH` | `/annotations/:id` | Update status (resolve, orphan, re-anchor) |
| `POST` | `/reviews` | Create review batch, submit to OpenClaw |
| `GET` | `/reviews?doc_path=X` | List review history for a doc |

### MCP Tools (E4 Implementation of E2 Stubs)

| Tool | Params | Description |
|------|--------|-------------|
| `list_annotations` | `doc_path`, `section?`, `status?` | Query annotations with filters |
| `create_annotation` | `doc_path`, `section`, `content`, `parent_id?`, `author_type?` | Create comment or reply |
| `resolve_annotation` | `annotation_id` | Mark as resolved |
| `submit_review` | `doc_path`, `annotation_ids?` | Package and submit review batch |

---

## Stories

### S1: SQLite Schema + Annotation CRUD API

**Batch:** 1 (foundation — all other stories depend on this)

**Scope:**

- Create `annotations` and `reviews` tables with full schema
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

### S2: Unified Thread Panel — Comment Display + Navigation

**Batch:** 2 ⚡ (parallel with S3, depends on S1)

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

### S3: Highlight-to-Comment — Draft Creation UX

**Batch:** 2 ⚡ (parallel with S2, depends on S1)

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

### S4: Submit Review + Orphan Detection

**Batch:** 3 (depends on S2 + S3)

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
- No re-anchoring UI for orphaned comments (post-E4 polish).
- Orphan styling is collapsed-only for MVP (strikethrough/dim deferred).

---

### S5: MCP Tool Implementation + OpenClaw Integration

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

---

## Execution Plan

```
Batch 1: S1 (schema + CRUD API) — foundation, must ship first
Batch 2: S2 + S3 (thread panel + highlight UX) — Lightning Strike ⚡, git worktrees
Batch 3: S4 (submit + orphans) — depends on S2 + S3
Batch 4: S5 (MCP implementation) — depends on S1 + S4
```

Batch 2 is parallel via git worktrees. All other batches are sequential (single agent).

---

## Decisions Log

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
| 9 | Orphaned comment styling | Collapsed section for MVP, iterate styling after seeing real data | Mar 30 |
| 10 | Foundry OpenClaw skill | Fast-follow after E4 ships, NOT in E4 scope | Mar 30 |

---

## Refinement Stories (Post-QA)

*Added: April 1, 2026 — from live QA session on deployed Foundry instance*

### R1: Simplify Orphan Detection

**Context:** Current orphan logic is too aggressive — heading path drift from rehype-autolink-headings causes false positives (all previous comments marked orphaned on every page load). Heading text cleanup helps but doesn't address the fundamental design issue.

**New Rule (replaces current logic):**

- **Orphaned** = the quoted text literally cannot be found anywhere in the current document (content was deleted/rewritten). If `quoted_text` is null (general section comment), orphan only if the heading text doesn't exist anywhere in the doc.
- **Resolved** = explicitly marked by a human or AI via the UI or API.
- Everything else stays visible regardless of heading path changes.

**Scope:**

- Remove heading-path-based orphan detection from `AnnotationThread.tsx`
- Replace with quoted-text search: scan `article.content` for the annotation's `quoted_text` string
- For annotations without `quoted_text`, fall back to heading text existence check (fuzzy, last segment only)
- Remove content hash drift detection (was never reliable — hash changes on any edit, even typo fixes)
- Stop auto-PATCHing annotations to `orphaned` status on page load — only mark orphaned when text is truly gone

**Acceptance Criteria:**

- [ ] Comments with `quoted_text` that still exists in the doc are never orphaned
- [ ] Comments with `quoted_text` that's been deleted show as orphaned
- [ ] Comments without `quoted_text` only orphan if heading text is completely gone
- [ ] No false orphans from heading path drift or content hash changes
- [ ] Existing orphaned annotations restored to previous status on deploy

---

### R2: Thread Panel — Inline Reply UI

**Context:** Currently only Clay can reply to comments (via API). Dan can't reply to Clay's comments from the Foundry UI. The thread panel needs a reply affordance.

**Scope:**

- Add "↩ Reply" button on each comment in the thread panel
- Click opens a textarea inline below the comment (similar to CommentDraft editor)
- Reply is submitted directly to API as a new annotation with `parent_id` set
- Reply uses `author_type: 'human'` and inherits the parent's `review_id`
- After submission, thread refreshes to show the new reply threaded under the parent
- No draft stage for replies — submit immediately (replies are quick responses, not batch-reviewed)

**Acceptance Criteria:**

- [ ] Reply button visible on every comment (human and AI)
- [ ] Clicking opens inline textarea below the comment
- [ ] Submit creates annotation with correct `parent_id` and `review_id`
- [ ] Thread re-fetches and shows reply threaded under parent
- [ ] Cancel button closes the textarea without submitting
- [ ] Only visible when authenticated

---

### R3: Thread Panel — Resolve Button

**Context:** No way to mark comment threads as resolved from the UI. Needed to close the feedback loop — human reviews AI reply, marks it done.

**Scope:**

- Add "✅ Resolve" button on top-level comments (not replies)
- Click PATCHes the annotation status to `resolved`
- Resolved comments collapse to one-line summary (already implemented in render logic)
- Add "↩ Reopen" on collapsed resolved comments to set status back to `submitted`
- When all annotations in a review are resolved, PATCH the review status to `complete`

**Acceptance Criteria:**

- [ ] Resolve button on top-level comments (draft, submitted, replied)
- [ ] Click sets status to `resolved` via API
- [ ] Resolved comment collapses to summary line
- [ ] Reopen button on resolved comments restores to `submitted`
- [ ] Review auto-completes when all child annotations are resolved
- [ ] Only visible when authenticated

---

### R4: Refactor MCP Tools to Wrap HTTP API

**Context:** Current MCP annotation tools (from S5) go directly to SQLite, bypassing the REST API. This means MCP only works co-located with the Foundry process. Refactoring to wrap the HTTP API enables remote agent access (e.g., OpenClaw at home hitting Fly.io deployment).

**Design Principle:** Agents should interact with Foundry through MCP tools, not raw HTTP. MCP is the single interface surface for AI agents — typed schemas, tool discovery, consistent auth. The REST API serves the browser UI and acts as the backend for MCP.

**Scope:**

- Refactor all 4 MCP annotation tools to call HTTP API instead of direct SQLite:
  - `list_annotations` → `GET /api/annotations`
  - `create_annotation` → `POST /api/annotations`
  - `resolve_annotation` → `PATCH /api/annotations/:id`
  - `submit_review` → `POST /api/reviews` + `POST /api/annotations` + status PATCHes
- Add `FOUNDRY_API_URL` config (defaults to `http://localhost:3001` for co-located, configurable for remote)
- MCP server passes `FOUNDRY_WRITE_TOKEN` as Bearer auth on all API calls
- Update Foundry OpenClaw skill to reference MCP tools instead of curl commands
- Update mcporter config to spawn Foundry MCP server

**Acceptance Criteria:**

- [ ] All 4 MCP tools work against remote Foundry instance (Fly.io)
- [ ] MCP tools produce identical results to direct API calls
- [ ] `FOUNDRY_API_URL` configurable via env var
- [ ] Auth token passed through correctly
- [ ] Existing MCP tests updated and passing
- [ ] Foundry skill updated to use MCP tools
- [ ] mcporter config wired up

---

## Execution Plan (Refinements)

```
Batch R1: R1 (orphan simplification) — standalone, no dependencies
Batch R2: R2 + R3 (reply + resolve UI) — Lightning Strike ⚡, can parallel
Batch R3: R4 (MCP refactor) — standalone, can run anytime
```

R1 is the most urgent (current behavior actively breaks things). R2+R3 complete the review loop. R4 is architectural cleanup.

---

## Bugs Found During Live QA (April 1, 2026)

### BUG-1: Aggressive Orphan Detection Breaks Review Workflow (CRITICAL — blocked by R1)
Current orphan logic re-orphans annotations on every page load when heading path doesn't exactly match DOM `textContent` (anchor text drift from rehype-autolink-headings). This means:
- Human submits review comments → page reload → all comments marked orphaned
- AI replies via API → replies have `parent_id` to orphaned annotations → hidden
- Review loop is broken until R1 ships

**Workaround:** None. AI replies are invisible until orphan logic is simplified.

### BUG-2: Can Comment on Draft List UI Elements
The CommentDraft component renders a draft list at the bottom of `article.content`. The text selection listener doesn't exclude this area, so users can highlight draft text and create a comment *on their own drafts*. The heading path for these "sections" is meaningless, and submitting causes a 400 error.

**Fix:** Add `data-no-comment` attribute to the draft list container and check for it in the `handleSelection` listener. Exclude from `article.content` selection detection.

### BUG-3: Submitted Drafts Still Visible in Draft List
After submitting a review, the draft list at the bottom of the page still shows the submitted drafts. The CommentDraft component reads from localStorage, and either `clearDrafts()` isn't firing, or the component isn't re-rendering after the `foundry-draft-updated` event.

**Fix:** Ensure CommentDraft listens for `foundry-draft-updated` events (same pattern as SubmitReview). May also need to listen for `foundry-review-submitted`.

### BUG-4: Duplicate Annotations on Submit
Two identical annotations created from one submit action (`gdzjz996` and `mm155zbq` — same content, same quoted text, same review). Likely a double-submit from rapid button clicks or missing debounce.

**Fix:** Add `disabled` state during submission + server-side idempotency check (reject duplicate `content` + `quoted_text` + `review_id` within a short window).

### BUG-5: Heading Path Includes Anchor Numbering
Despite the `getCleanHeadingText()` fix deployed in this session, heading paths are still being stored with anchor artifacts (e.g., `## Fast-Follow (Post-E4)3`). Either the fix isn't working on the deployed build, or the heading structure in the E4 doc produces different anchor text than expected.

**Needs investigation:** Check if the deployed build has the `getCleanHeadingText()` changes, and verify the anchor structure on E4 page specifically.

---

## UI/UX Enhancements (from QA)

### ENH-1: Refresh Button on Thread Panel
Add a 🔄 refresh button in the thread panel header (next to the collapse toggle). Currently the only way to see new replies is a full page refresh. This is a quick win — just call `fetchAnnotations()` on click.

### ENH-2: Inline Text Highlighting for Commented Text
Highlight text in the document that has comments on it (Dan's suggestion from R2 review). Currently only section margin badges indicate comments exist. Inline highlighting would be more intuitive — see exactly which text is annotated without opening the thread panel.

**Scope:** Fast-follow after R2, not in scope for current refinements.

---

## Fast-Follow (Post-E4)

- ~~Foundry OpenClaw skill (teaches Clay how to process review batches)~~ ✅ Done (April 1)
- Re-anchor UI for orphaned comments
- Orphan visual polish (strikethrough/dim after seeing real data)
- Real-time reply push via SSE (currently refresh-to-see)
- Inline text highlighting for commented passages (ENH-2)
