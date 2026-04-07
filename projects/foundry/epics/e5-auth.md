# E5: Authentication + Write Protection

*Status: Step 1 Complete — Ready for Step 2 (Agent Prompt Crafting)*
*Epic: Foundry v0.2*
*Created: March 31, 2026*
*Updated: March 31, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

Add bearer token authentication to Foundry's API. This is the deployment gate — without auth, anyone who discovers the Fly.io URL can create annotations, submit reviews, and inject content that reaches Clay via MCP tools (prompt injection vector). Token auth is the permanent machine-auth layer that stays even when human-friendly auth (OAuth, username/password) ships later.

### Problem Statement

Foundry is being deployed to the public internet (Fly.io). Without auth:
- Anyone who discovers the URL can create annotations and submit reviews
- Submitted reviews reach Clay via MCP — a direct prompt injection vector
- Annotation content (review feedback, internal discussion) is readable by anyone
- No identity tracking on annotations (hardcoded `user_id='dan'`)

### Goals

- [ ] Bearer token middleware on all write endpoints
- [ ] Bearer token required for annotation read endpoints (review content is sensitive)
- [ ] Frontend token entry modal (first visit when interacting with annotations)
- [ ] Frontend includes token in all API calls that require it
- [ ] MCP tools pass token in requests
- [ ] Token configurable via environment variable
- [ ] Document the security model in DEPLOY.md

### Non-Goals (Deferred)

- **Public/private doc access control** (E6 — separate content visibility layer)
- **Username/password auth** (someday — token auth covers single-user needs)
- **GitHub OAuth** (someday — when multi-user is needed)
- **Role-based access control** (just authenticated vs. not for now)
- **Rate limiting** (nice-to-have, not MVP)
- **Token rotation with multi-token support** (single token is fine for one user)
- **Static site content protection** (docs are meant to be read; private docs = E6)

---

## Architecture

### Security Model

**One layer, two concerns:**

1. **Write protection** — All POST/PATCH/DELETE endpoints require `Authorization: Bearer <token>`. This prevents unauthorized annotation creation, review submission, and any content that could reach MCP tools.

2. **Annotation read protection** — GET endpoints under `/api/annotations` and `/api/reviews` require auth. Annotations may contain sensitive review feedback, code discussion, or internal notes. Doc content (static site, `/api/docs/*`, `/api/search`) stays public.

### Token Flow

```
Server startup:
  FOUNDRY_WRITE_TOKEN env var → loaded into memory

Browser (human):
  1. Visit site → docs render (no auth needed)
  2. Open thread panel or try to comment → check localStorage for token
  3. No token? → modal: "Enter your access token"
  4. User pastes token → stored in localStorage
  5. All annotation API calls include Authorization header
  6. 401 response → clear localStorage → re-show modal

MCP client (Clay/agents):
  1. Token configured in MCP client config or env
  2. All tool calls include token in request headers
  3. 401 → tool returns error (agent handles gracefully)
```

### What's Protected vs. Public

| Endpoint Pattern | Auth Required | Rationale |
|-----------------|---------------|-----------|
| Static site (HTML/CSS/JS) | ❌ | Docs are meant to be read |
| `GET /api/docs/*` | ❌ | Doc content = same as static site |
| `GET /api/search` | ❌ | Search over public doc content |
| `GET /api/health` | ❌ | Health check for monitoring |
| `GET /api/annotations*` | ✅ | Review content is sensitive |
| `GET /api/reviews*` | ✅ | Review metadata is sensitive |
| `POST /api/annotations*` | ✅ | Write = injection vector |
| `PATCH /api/annotations*` | ✅ | Write = injection vector |
| `DELETE /api/annotations*` | ✅ | Write = injection vector |
| `POST /api/reviews*` | ✅ | Write = injection vector |
| All MCP write tools | ✅ | Same token, same middleware |
| All MCP read tools (docs) | ❌ | Public content |
| MCP annotation tools | ✅ | Annotation content is sensitive |

### Middleware Design

```typescript
// src/middleware/auth.ts
export function requireAuth(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token || token !== process.env.FOUNDRY_WRITE_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

// Applied selectively in route registration:
app.use('/api/annotations', requireAuth);
app.use('/api/reviews', requireAuth);
// MCP tools check token internally
```

### Frontend Token Modal

- Simple modal overlay: text input + "Unlock" button
- Triggered when user tries to interact with annotations (not on page load)
- Token stored in `localStorage` under key `foundry_token`
- All fetch calls to protected endpoints include `Authorization: Bearer ${token}`
- On 401: clear `foundry_token`, re-show modal with "Token expired or invalid" message
- No token = read-only mode (can browse docs, can't see or create annotations)

### MCP Integration

MCP tools that touch annotations check the token the same way:
- Token passed via MCP tool config or extracted from request context
- Tools return structured error on auth failure (not silent fail)
- `submit_review`, `create_annotation`, `list_annotations`, `resolve_annotation` all require auth

---

## Stories

### S1: Auth Middleware + Protected Routes

**Batch:** 1 (foundation, must ship first)

**Scope:**

- Create `src/middleware/auth.ts` with `requireAuth` middleware
- Read `FOUNDRY_WRITE_TOKEN` from environment (error on startup if missing in production)
- Apply middleware to all `/api/annotations*` and `/api/reviews*` routes (GET included)
- Leave `/api/docs/*`, `/api/search`, `/api/health` unprotected
- Return 401 with `{ error: 'Unauthorized' }` for missing/invalid tokens
- Accept token via `Authorization: Bearer <token>` header
- Update existing tests to include auth headers where needed
- Add auth-specific tests (missing token, wrong token, valid token, public endpoints still work)
- Update `.env.example` / environment docs with `FOUNDRY_WRITE_TOKEN`

**Acceptance Criteria:**

- [ ] Protected endpoints return 401 without valid Bearer token
- [ ] Protected endpoints return 200 with valid Bearer token
- [ ] Public endpoints (`/api/docs/*`, `/api/search`, `/api/health`) work without any token
- [ ] `FOUNDRY_WRITE_TOKEN` read from env var
- [ ] Server logs warning if `FOUNDRY_WRITE_TOKEN` is unset (but doesn't crash in dev)
- [ ] Existing annotation/review tests updated to pass auth headers
- [ ] New tests: 401 on missing token, 401 on wrong token, 200 on valid token
- [ ] New tests: public endpoints remain accessible without auth

**Boundaries:**

- No frontend changes (that's S2)
- No MCP changes (that's S3)
- No Docker/deploy changes (that's S4)

---

### S2: Frontend Token Modal + Auth Headers

**Batch:** 2 (depends on S1)

**Scope:**

- Create `TokenModal.tsx` React island — overlay modal with token input + "Unlock" button
- Modal appears when user first interacts with annotation features (opens thread panel, clicks comment button)
- On submit: store token in `localStorage` key `foundry_token`
- Create `src/lib/api.ts` (or update existing) — wrapper around `fetch` that auto-includes `Authorization: Bearer ${token}` header on protected endpoints
- All existing annotation/review fetch calls use the auth-aware wrapper
- On 401 response: clear `foundry_token` from localStorage, show modal with error message "Token expired or invalid — please re-enter"
- No-token state = read-only: thread panel shows "Enter token to view annotations" prompt instead of loading annotations
- Subtle lock/unlock icon in header or thread panel indicating auth state

**Acceptance Criteria:**

- [ ] Token modal appears on first annotation interaction (not on page load)
- [ ] Token stored in localStorage after entry
- [ ] All annotation API calls include Authorization header
- [ ] 401 response triggers token clear + modal re-display with error message
- [ ] Without token: docs browsable, thread panel shows "enter token" prompt
- [ ] With valid token: full annotation functionality works
- [ ] Lock/unlock visual indicator of auth state
- [ ] Token persists across page refreshes (localStorage)
- [ ] Modal is styled consistently with existing Foundry dark/light theme

**Boundaries:**

- No new API endpoints (S1 handles that)
- No MCP changes (that's S3)
- Token validation is server-side only — frontend just stores and sends it

---

### S3: MCP Tool Auth Integration

**Batch:** 2 ⚡ (parallel with S2, depends on S1)

**Scope:**

- Update all MCP annotation tools to include auth token in API requests
- Token source: environment variable `FOUNDRY_WRITE_TOKEN` (same as server, since MCP server runs in-process)
- For SSE/remote MCP clients: token passed via tool configuration
- MCP tools that touch annotations: `list_annotations`, `create_annotation`, `resolve_annotation`, `submit_review`
- On auth failure: return structured MCP error (not silent fail, not crash)
- Update MCP tool tests to verify auth headers sent
- Document MCP client token configuration in DEPLOY.md section

**Acceptance Criteria:**

- [ ] All 4 annotation MCP tools include auth token in requests
- [ ] Auth failure returns structured error message via MCP response
- [ ] In-process MCP (same container) uses `FOUNDRY_WRITE_TOKEN` env var directly
- [ ] Tests verify auth headers on MCP tool requests
- [ ] MCP doc search tools (`search_docs`, `get_page`, etc.) still work without auth

**Boundaries:**

- No frontend changes (that's S2)
- No new MCP tools — just auth on existing ones
- Remote MCP client config is documented, not implemented (we run in-process)

---

### S4: Docker + Deploy Config Update

**Batch:** 3 (depends on S1, S2, S3)

**Scope:**

- Update `docker-compose.yml` to include `FOUNDRY_WRITE_TOKEN` env var (from `.env` file)
- Update `fly.toml` — set token via `fly secrets set FOUNDRY_WRITE_TOKEN=<value>`
- Create `.env.example` with all required env vars documented
- Update `DEPLOY.md` with:
  - Token generation instructions (`openssl rand -hex 32`)
  - Local dev setup (`.env` file)
  - Fly.io secret setup (`fly secrets set`)
  - MCP client configuration
  - Security model explanation (what's protected, what's public)
- Verify Docker build still passes with new code
- Test full flow: build container → run with token → verify auth works

**Acceptance Criteria:**

- [ ] `docker-compose.yml` references `FOUNDRY_WRITE_TOKEN` from `.env`
- [ ] `.env.example` documents all required environment variables
- [ ] `DEPLOY.md` has complete auth setup instructions
- [ ] Docker build passes
- [ ] Container runs with token enforcement working
- [ ] `fly.toml` documented for secret setup (not committed — fly secrets are runtime)

**Boundaries:**

- No Fly.io deployment (Dan does `fly launch` manually after E5 ships)
- No CI/CD changes to GitHub Actions
- Documentation-focused story — minimal code changes

---

## Execution Plan

```
Batch 1: S1 (middleware + protected routes) — foundation, must ship first
Batch 2: S2 + S3 (frontend modal + MCP auth) — Lightning Strike ⚡, git worktrees
Batch 3: S4 (Docker + deploy config) — depends on all above
```

3 batches, 4 stories. Lean epic — this is a security gate, not a feature. Ship fast, deploy after.

---

## Decisions Log

| # | Question | Decision | Date |
|---|----------|----------|------|
| 1 | Annotation read auth | YES — all annotation/review endpoints require auth (content is sensitive) | Mar 31 |
| 2 | Token strategy | Single bearer token via env var. Permanent machine-auth layer. | Mar 31 |
| 3 | Static site protection | NO — docs are public. Private docs = E6. | Mar 31 |
| 4 | Frontend auth flow | Token modal on first annotation interaction → localStorage → auto-header | Mar 31 |
| 5 | CORS | Non-issue — same-origin serving from single container | Mar 31 |
| 6 | Token rotation | Not needed for MVP — single user, change env var + redeploy | Mar 31 |
| 7 | Username/password | Deferred indefinitely — token auth covers single-user needs, OpenClaw pattern | Mar 31 |
| 8 | Public/private docs | E6 epic — separate content visibility layer | Mar 31 |
| 9 | MCP tool auth | Same token, in-process access via env var | Mar 31 |

---

## Roadmap Context

```
E5 (this): Bearer token auth → deployment gate
E6 (next): Public/private doc access control → content visibility
Someday:   Multi-user auth (OAuth/username-password) → when there are multiple users
```

E5's token middleware is the permanent foundation. Everything built on top adds human-friendly auth layers — nothing gets ripped out.
