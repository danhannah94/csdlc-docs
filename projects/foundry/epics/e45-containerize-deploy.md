# E4.5: Containerize + Deploy

*Status: Step 1 Complete — Ready for Step 2 (Agent Prompt Crafting)*
*Epic: Foundry v0.2.1*
*Created: March 31, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

Package Foundry as a single Docker container that serves both the static site and API from one process, one port. Deploy to Fly.io for persistent remote access. Eliminates the split deployment problem (static site on GitHub Pages, API needs a separate server).

### Problem Statement

E4 shipped annotations, but the API server must run locally for the site to function. Dan works from his iPad at remote locations while the Mac runs at home. GitHub Pages serves static HTML but can't talk to a local API. We need:

1. A single container that IS the product (site + API + SQLite in one)
2. A deployed instance accessible from anywhere (iPad, work, home)
3. Zero-config portability — same image runs locally, at work (Docker), or in the cloud

### Goals

- Single Docker image: Express serves static Astro build + API + MCP on one port
- SQLite data persists via volume mount
- Environment-variable-driven configuration (PORT, DB path, CORS origins)
- Dockerfile + docker-compose.yml for local dev
- Deploy to Fly.io free tier
- GitHub Actions CD: push to main → build → deploy

### Non-Goals

- AWS / Terraform / heavy cloud infra (earned when needed)
- Supabase migration (separate epic when multi-user is needed)
- Custom domain (can add later via Fly.io certs, 2-minute task)
- Auth / login (still using GitHub repo visibility for now)

---

## Architecture

### Before (E4)

```
GitHub Pages (static)  ←→  localhost:3001 (API)
      ↑                         ↑
   iPad/Work               Mac at home only
   (works)                 (doesn't work remote)
```

### After (E4.5)

```
┌───────────────────────────────┐
│  Foundry Container (one port) │
│                               │
│  Express server (:3001)       │
│    ├── /*       → static site │
│    ├── /api/*   → REST routes │
│    ├── /mcp/*   → SSE + MCP   │
│    └── /health  → healthcheck │
│                               │
│  SQLite → /data/foundry.db    │
│  Config → env vars            │
└───────────────────────────────┘
        ↑
  Fly.io / Docker / anywhere
```

### Key Design Decisions

1. **Express serves everything** — `express.static()` serves the Astro build output. API routes mounted under `/api/` prefix to avoid path collisions. MCP stays at `/mcp/*`.

2. **Route prefix migration** — Current API routes are at root (`/annotations`, `/reviews`, etc.). Move to `/api/annotations`, `/api/reviews`. Update the frontend fetch calls to use relative URLs (`/api/annotations`) instead of absolute (`http://localhost:3001/annotations`). This makes the site work regardless of where it's hosted.

3. **SQLite volume** — `FOUNDRY_DB_PATH` env var, defaults to `/data/foundry.db` in container. Volume mount: `-v foundry-data:/data`.

4. **Multi-stage Docker build** — Stage 1: build Astro site. Stage 2: build API. Stage 3: minimal runtime image with both outputs.

5. **CORS simplified** — When serving from the same origin, CORS is no longer needed for the UI. Keep CORS middleware for external MCP clients.

6. **GitHub Pages stays as-is** — Don't remove the GitHub Pages deploy. It's the public/open docs version. The container is the full-featured version with annotations.

---

## Stories

### S1: API Route Prefixing + Relative URLs

**Batch:** 1 (foundation — S2 depends on this)

**Scope:**

- Move all API routes under `/api/` prefix in Express:
  - `/annotations` → `/api/annotations`
  - `/reviews` → `/api/reviews`
  - `/health` → `/api/health`
  - `/mcp/*` stays at `/mcp/*` (MCP transport has its own convention)
- Update all frontend fetch calls in React components to use relative URLs:
  - `http://localhost:3001/annotations` → `/api/annotations`
  - Remove `apiBaseUrl` prop from components (no longer needed)
- Update DocLayout.astro to remove apiBaseUrl passing
- Update existing tests to use new `/api/` prefixed routes

**Acceptance Criteria:**

- [ ] All REST endpoints respond at `/api/*` paths
- [ ] Old root paths return 404
- [ ] Frontend components use relative `/api/*` URLs
- [ ] `apiBaseUrl` prop removed from AnnotationThread and SubmitReview
- [ ] MCP endpoints still at `/mcp/*`
- [ ] Existing tests updated and passing

**Boundaries:**

- No Docker work (that's S2)
- No static file serving (that's S2)
- No deployment (that's S3)

---

### S2: Dockerfile + Static Serving + docker-compose

**Batch:** 2 (depends on S1)

**Scope:**

- Add `express.static()` middleware to serve Astro build output:
  ```typescript
  app.use(express.static(path.join(__dirname, '../../site/dist')));
  // Fallback for SPA-style routing
  app.get('*', (req, res) => {
    res.sendFile(path.join(__dirname, '../../site/dist/index.html'));
  });
  ```
  IMPORTANT: Static file middleware goes AFTER API routes, BEFORE the catch-all fallback.

- Create multi-stage `Dockerfile` at repo root:
  ```dockerfile
  # Stage 1: Build static site
  FROM node:22-alpine AS site-builder
  WORKDIR /app
  COPY packages/site/ packages/site/
  COPY package*.json foundry.config.yaml nav.yaml scripts/ ./
  RUN npm ci --prefix packages/site && npm run build --prefix packages/site

  # Stage 2: Build API
  FROM node:22-alpine AS api-builder
  WORKDIR /app
  COPY packages/api/ packages/api/
  RUN npm ci --prefix packages/api && npm run build --prefix packages/api

  # Stage 3: Production runtime
  FROM node:22-alpine
  WORKDIR /app
  COPY --from=site-builder /app/packages/site/dist packages/site/dist
  COPY --from=api-builder /app/packages/api/dist packages/api/dist
  COPY --from=api-builder /app/packages/api/node_modules packages/api/node_modules
  COPY --from=api-builder /app/packages/api/package.json packages/api/
  COPY foundry.config.yaml nav.yaml ./
  
  ENV PORT=3001
  ENV FOUNDRY_DB_PATH=/data/foundry.db
  EXPOSE 3001
  
  RUN mkdir -p /data
  VOLUME ["/data"]
  
  CMD ["node", "packages/api/dist/index.js"]
  ```

- Create `docker-compose.yml` at repo root:
  ```yaml
  services:
    foundry:
      build: .
      ports:
        - "3001:3001"
      volumes:
        - foundry-data:/data
      environment:
        - PORT=3001
        - FOUNDRY_DB_PATH=/data/foundry.db
  volumes:
    foundry-data:
  ```

- Add `.dockerignore` (node_modules, .git, dist, *.db)

- Update Express `index.ts`:
  - Add static file serving after all API routes
  - Make the static file path configurable via env var `FOUNDRY_STATIC_PATH` (default: `../../site/dist` relative to api dist)
  - Add catch-all route for client-side routing

- Verify: `docker build -t foundry . && docker run -p 3001:3001 foundry` serves both site and API

**Acceptance Criteria:**

- [ ] `docker build` succeeds
- [ ] `docker run -p 3001:3001 foundry` serves static site at `/`
- [ ] API endpoints respond at `/api/*`
- [ ] MCP endpoints respond at `/mcp/*`
- [ ] SQLite persists across container restarts (volume mount)
- [ ] `docker-compose up` works
- [ ] Health endpoint returns valid status

**Boundaries:**

- No CI/CD pipeline (that's S3)
- No Fly.io config (that's S3)
- No Anvil/search changes — if Anvil fails to init, log warning and continue (search disabled, annotations work)

---

### S3: Fly.io Deploy + GitHub Actions CD

**Batch:** 3 (depends on S2)

**Scope:**

- Create `fly.toml` at repo root:
  ```toml
  app = "foundry-claymore"
  primary_region = "iad"  # US East (closest to Charlotte)

  [build]

  [http_service]
    internal_port = 3001
    force_https = true

  [mounts]
    source = "foundry_data"
    destination = "/data"

  [[vm]]
    size = "shared-cpu-1x"
    memory = "256mb"
  ```

- Create `.github/workflows/deploy-fly.yml`:
  ```yaml
  name: Deploy to Fly.io
  on:
    push:
      branches: [main]
    workflow_dispatch:
  jobs:
    deploy:
      runs-on: ubuntu-latest
      steps:
        - uses: actions/checkout@v4
        - uses: superfly/flyctl-actions/setup-flyctl@master
        - run: flyctl deploy --remote-only
          env:
            FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
  ```

- NOTE: Dan will need to:
  1. `brew install flyctl` (or `curl -L https://fly.io/install.sh | sh`)
  2. `fly auth signup` (free account)
  3. `fly launch` (one-time setup from repo root)
  4. `fly volumes create foundry_data --region iad --size 1` (1GB free)
  5. Add `FLY_API_TOKEN` to GitHub repo secrets

- Document these steps in a `DEPLOY.md` at repo root

**Acceptance Criteria:**

- [ ] `fly.toml` configures single-VM deployment with persistent volume
- [ ] GitHub Actions workflow triggers on push to main
- [ ] `DEPLOY.md` has clear setup instructions
- [ ] Health endpoint accessible at `https://foundry-claymore.fly.dev/api/health`
- [ ] Annotations persist across deploys (volume mount)

**Boundaries:**

- No custom domain setup (2-minute follow-up task, not worth a story)
- No SSL config (Fly handles this automatically)
- No monitoring/alerting (earned complexity)

---

## Execution Plan

```
Batch 1: S1 (route prefixing + relative URLs) — foundation
Batch 2: S2 (Dockerfile + static serving) — the container
Batch 3: S3 (Fly.io + CD) — deployment
```

All sequential — each builds on the previous. Estimated: ~30 min total agent time.

---

## Decisions Log

| # | Question | Decision | Date |
|---|----------|----------|------|
| 1 | Hosting platform | Fly.io free tier (3 shared VMs, 1GB volume) | Mar 31 |
| 2 | Container architecture | Single Express process serves site + API + MCP | Mar 31 |
| 3 | Route strategy | API under `/api/*`, MCP under `/mcp/*`, static at `/*` | Mar 31 |
| 4 | Database | SQLite in volume mount, Supabase deferred | Mar 31 |
| 5 | GitHub Pages | Keep as public docs version, container is full-featured | Mar 31 |
| 6 | CD pipeline | GitHub Actions → Fly.io on push to main | Mar 31 |
