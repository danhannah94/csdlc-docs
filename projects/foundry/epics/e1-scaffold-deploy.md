# E1: Scaffold + Static Site + Deploy

*Status: ✅ Complete — All Stories Shipped (March 30, 2026)*
*Epic: Foundry v0.1*
*Created: March 30, 2026*
*Updated: March 30, 2026*

---

## Overview

### What Is This Epic?

Stand up the Foundry project from scratch: Astro monorepo, markdown rendering pipeline, navigation, dark mode, and deploy to GitHub Pages with public/private repo separation. This is the foundation everything else builds on.

### Problem Statement

Dan can't read CSDLC docs from his iPad at work. MkDocs is local-only. Project docs shouldn't be public. There's no deployed, access-controlled way to read the docs from any device.

### Goals

- Astro monorepo scaffold (`packages/site/` + placeholder `packages/api/`)
- Markdown rendering: tables, code blocks (syntax highlighting), Mermaid diagrams, admonitions, heading anchors
- Navigation sidebar generated from `nav.yaml`
- Breadcrumbs
- Dark mode (system preference + manual toggle, localStorage)
- Multi-repo content sourcing via `foundry.config.yaml` + build script
- GitHub Pages deployment (public repo for public docs, private repo for all docs)

### Non-Goals

- Search (cut from MVP — nav sidebar sufficient for ~30 docs, Anvil semantic search comes in E2)
- Anvil integration (E2 / v0.2 — requires API server)
- TTS playback (E3 / v0.2)
- Annotations or comments (E4 / v0.2)
- API server (v0.2)
- Custom branding or themes beyond clean defaults
- Mobile-specific responsive design (basic responsive is fine, dedicated mobile UX is not)

---

## Context

### Current State

CSDLC docs live in `csdlc-docs/docs/` as markdown files, served locally via MkDocs. No remote access, no access control, no way to read from iPad.

### Affected Systems

| System / Layer | How It's Affected |
|---------------|-------------------|
| `csdlc-docs/` repo | Source content — unchanged, consumed at build time |
| GitHub | New repo(s) for Foundry project + GitHub Pages deployment |

### Dependencies

- GitHub Pro account (Dan has this) for private repo Pages
- `csdlc-docs/` repo with markdown content
- `mkdocs.yml` nav tree (reference for building `nav.yaml`)

### Dependents

- **E2 (Search)** — Anvil integration builds on the site scaffold (v0.2, requires API)
- **E3 (TTS)** — needs rendered pages to add audio controls
- **E4 (Annotations)** — needs the UI to add highlight/comment layer
- **All future epics** depend on the monorepo structure and build pipeline established here

---

## Design

### Approach

**Monorepo structure** with npm workspaces. `packages/site/` contains the Astro static site. `packages/api/` is a placeholder for v0.2. Shared config lives at the root.

**Build pipeline:**
1. `scripts/build.sh` reads `foundry.config.yaml`
2. Clones each source repo (or pulls latest if cached)
3. Copies specified paths into `packages/site/content/`
4. Runs `astro build` to generate static HTML/CSS/JS
5. Outputs to `dist/` for GitHub Pages deployment

**Content rendering:** Astro's built-in markdown support handles most features. Mermaid via `astro-mermaid` integration. Admonitions need a remark plugin — MkDocs uses `!!!` syntax, we'll need either a compatible remark plugin or a lightweight adapter.

**Navigation:** `nav.yaml` at the project root defines the doc tree. Parsed at build time by a utility function. Sidebar component renders the tree with active page highlighting and expand/collapse.

**Dark mode:** System preference detection via `prefers-color-scheme` media query. Manual toggle stored in localStorage. Theme applied via CSS custom properties.

**GitHub Pages deploy:** GitHub Actions workflow triggered on push to main. Runs the build script, deploys `dist/` to GitHub Pages. Two deployment targets:
- Public repo: methodology docs only
- Private repo: all docs (restricted to collaborators)

### Configuration Files

**`foundry.config.yaml`:**
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

**`nav.yaml`** (derived from mkdocs.yml):
```yaml
- title: Home
  path: index.md
- title: Methodology
  children:
    - title: "The CSDLC Thesis"
      path: methodology/thesis.md
    - title: CSDLC Process
      path: methodology/process.md
- title: Projects
  access: private
  children:
    - title: Foundry
      children:
        - title: Design
          path: projects/foundry/design.md
```

### Key Algorithms / Logic

**Build script (`scripts/build.sh`):**
1. Parse `foundry.config.yaml`
2. For each source: `git clone --depth 1 --branch <branch> <repo>` (or `git pull` if cached)
3. Copy specified paths to `packages/site/content/`
4. Generate `.access.json` metadata file
5. Run `cd packages/site && npm run build`

**Nav parser:**
- Read `nav.yaml` → build tree structure
- At build time, inject into Astro as a data file
- Sidebar component renders recursively with active state

---

## Edge Cases & Gotchas

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| Source repo clone fails (network, auth) | Build fails with clear error message | CI needs GitHub token for private repos |
| Markdown with MkDocs-specific syntax | Render gracefully or show raw | Admonition `!!!` syntax isn't standard |
| Nav references a file that doesn't exist | Build warning, skip the entry | Mismatched nav.yaml and actual files |
| Mermaid diagram in dark mode | Should be readable in both themes | `astro-mermaid` claims theme switching support |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| MkDocs admonition syntax incompatible | Medium | Medium | Write a small remark plugin or adapt docs |
| GitHub Pages private repo requires Pro | Low (Dan has Pro) | High (blocker) | Confirmed Dan has Pro |
| Astro learning curve for sub-agents | Low | Medium | Well-documented, provide Astro docs in agent prompt |
| Build script complexity (multi-repo clone) | Low | Low | Simple bash, well-tested |

---

## Testing Strategy

### Test Layers

| Layer | Applies? | Notes |
|-------|:--------:|-------|
| **Unit tests** | Yes | Nav parser, config parser |
| **Integration tests** | Yes | Build script produces valid output, all markdown renders |
| **Visual tests** | No | Manual QA by Dan (does it look right on iPad?) |
| **E2E** | No | Static site — no runtime behavior to test |

### Verification Rules

1. `npm run build` succeeds with no errors
2. All markdown files in nav.yaml render to HTML pages
3. Dark mode toggle works
4. Mermaid diagrams render
5. GitHub Pages deploy succeeds and site is accessible

---

## Stories

| Story | Summary | Batch | Dependencies | Status | PR |
|-------|---------|-------|-------------|--------|----| 
| S0 | Monorepo scaffold + Astro project setup | 1 | None | ✅ Done | Clay direct |
| S1 | Markdown rendering pipeline (tables, code, mermaid, admonitions) | 2 | S0 | ✅ Done | Sub-agent |
| S3 | Dark mode (system pref + toggle + localStorage) | 2 | S0 | ✅ Done | Sub-agent |
| S4 | Multi-repo build script + `foundry.config.yaml` | 2 | S0 | ✅ Done | Sub-agent |
| S2 | Nav system (`nav.yaml` parser + sidebar + breadcrumbs) | 3 | S1, S3, S4 | ✅ Done | Sub-agent |
| S6 | GitHub Pages deploy (Actions workflow + public/private repos) | 4 | All | ✅ Done | Sub-agent |

**Execution plan:** 4 batches. Batch 2 is fully parallel (3 agents). Batch 3 needs all of Batch 2 complete. S6 is the capstone.

**Cut:** S5 (pre-built client-side search) — removed from MVP scope. Nav sidebar is sufficient for discovery on a ~30 doc site. Semantic search via Anvil comes in E2 when the API server is ready. If basic search is missed, Pagefind is a drop-in static solution.

### S0: Monorepo Scaffold + Astro Project Setup

**Summary:** Initialize the Foundry monorepo with npm workspaces, create the Astro project in `packages/site/`, create placeholder `packages/api/`, and set up root config files.

**Acceptance Criteria:**
1. Root `package.json` with npm workspaces pointing to `packages/*`
2. `packages/site/` — fresh Astro project (`npm create astro@latest`)
   - Astro config with React integration (`@astrojs/react`)
   - TypeScript enabled
   - `src/layouts/DocLayout.astro` — base layout with header, content area, sidebar placeholder
   - `src/pages/index.astro` — landing page (can be placeholder)
   - `src/styles/global.css` — CSS custom properties for theming (colors, spacing, fonts)
3. `packages/api/` — minimal `package.json` with `"name": "@foundry/api"`, no code yet
4. Root `foundry.config.yaml` — example config with the csdlc-docs source definition
5. Root `nav.yaml` — translated from current mkdocs.yml nav tree
6. `README.md` with project overview and dev setup instructions
7. `npm install` from root succeeds
8. `npm run dev` from `packages/site/` starts Astro dev server
9. `npm run build` from `packages/site/` produces `dist/` with valid HTML

**Boundaries:**
- Do NOT implement markdown rendering beyond Astro defaults
- Do NOT implement nav sidebar (just the layout slot for it)
- Do NOT set up GitHub Actions or deployment
- Do NOT add search functionality

### S1: Markdown Rendering Pipeline

**Summary:** Configure Astro to render CSDLC markdown docs with full feature support: tables, syntax-highlighted code blocks, Mermaid diagrams, MkDocs-style admonitions, and heading anchors.

**Acceptance Criteria:**
1. Astro content collection configured to read markdown from `content/` directory
2. Dynamic route `src/pages/[...slug].astro` renders any markdown file from content collection
3. Tables render with proper styling (borders, alternating rows, responsive overflow)
4. Code blocks render with syntax highlighting (Astro's built-in Shiki)
5. Mermaid diagrams render via `astro-mermaid` integration
6. MkDocs admonition syntax (`!!! note`, `!!! warning`, etc.) renders as styled callout boxes
   - If no existing remark plugin supports `!!!` syntax, write a minimal remark plugin
   - At minimum support: `note`, `warning`, `tip`, `danger`, `info`
7. Headings get anchor IDs and a permalink icon on hover
8. A test markdown file exists in `content/` with examples of all supported features
9. `npm run build` succeeds and all features render correctly in output HTML

**Boundaries:**
- Do NOT add nav or sidebar (S2)
- Do NOT add dark mode styling (S3)
- Do NOT set up build script for cloning repos (S4) — use manually copied test doc

**Context:** Our docs use MkDocs `pymdownx` extensions — Mermaid uses ` ```mermaid ` fences (standard), admonitions use `!!! type` blocks.

### S3: Dark Mode

**Summary:** Implement dark/light theme switching with system preference detection and manual toggle, persisted to localStorage.

**Acceptance Criteria:**
1. CSS custom properties defined for both light and dark themes (background, text, borders, code blocks, sidebar)
2. On first visit: detect system preference via `prefers-color-scheme` media query
3. Manual toggle component (`ThemeToggle.tsx` React island) — sun/moon icon button in header
4. Toggle preference saved to `localStorage` key `foundry-theme`
5. On subsequent visits: localStorage value overrides system preference
6. No flash of wrong theme on page load (inline script in `<head>` reads localStorage before render)
7. Mermaid diagrams respect the active theme
8. Code block syntax highlighting works in both themes

**Boundaries:**
- Do NOT build the full header/nav — just add toggle to whatever header exists from S0
- Styling should use CSS custom properties so all components inherit the theme

### S4: Multi-Repo Build Script + Config

**Summary:** Create the build script that reads `foundry.config.yaml`, clones source repos, and copies markdown files into the Astro content directory.

**Acceptance Criteria:**
1. `scripts/build.sh` (bash):
   - Reads `foundry.config.yaml` (use `yq` or a simple Node script for YAML parsing)
   - For each source entry: `git clone --depth 1 --branch <branch> <repo>` into `.cache/repos/`
   - If repo already exists in cache: `git pull` instead of clone
   - Copies specified `paths` into `packages/site/content/`
   - Preserves directory structure
   - Clears `content/` before copying (fresh build every time)
2. `foundry.config.yaml` schema validated (error if missing required fields: `repo`, `branch`, `paths`)
3. Access field (`public`/`private`) stored as metadata file (`content/.access.json`) for future use
4. Script is idempotent — running twice produces same result
5. Script handles auth for private repos (uses `gh auth` token or `GITHUB_TOKEN` env var)
6. Running `scripts/build.sh && cd packages/site && npm run build` produces a working site with cloned content
7. `.cache/` is in `.gitignore`

**Boundaries:**
- Do NOT modify Astro config — S1 already set up content collection from `content/`
- Do NOT implement deploy — just the content fetching pipeline

### S2: Navigation System

**Summary:** Parse `nav.yaml` at build time and render a sidebar with collapsible sections, active page highlighting, and breadcrumbs.

**Acceptance Criteria:**
1. `nav.yaml` parser utility (`src/utils/nav.ts`):
   - Reads `nav.yaml` at build time
   - Returns typed tree structure: `NavItem { title, path?, children?, access? }`
   - Handles nested sections (3+ levels deep)
2. `Sidebar.astro` component:
   - Renders nav tree recursively
   - Collapsible sections (click to expand/collapse)
   - Active page highlighted based on current URL
   - Parent sections auto-expanded when a child is active
   - Styled for both light and dark themes (uses CSS custom properties from S3)
3. `Breadcrumbs.astro` component:
   - Shows path from root to current page based on nav tree
   - Clickable links for each level
   - Rendered above doc content
4. Sidebar is responsive:
   - Desktop: always visible, fixed left
   - Mobile/tablet: hamburger menu toggle (collapsible overlay)
5. Layout updated: `DocLayout.astro` integrates sidebar + breadcrumbs + content area

**Boundaries:**
- Do NOT add search to the sidebar
- Do NOT implement access filtering

**Dependencies:** S1 (rendered pages), S3 (theme variables), S4 (content populated)

### S6: GitHub Pages Deploy

**Summary:** Set up GitHub Actions workflow for automated build and deploy to GitHub Pages.

**Acceptance Criteria:**
1. `.github/workflows/deploy.yml`:
   - Triggers on push to `main` branch
   - Runs build script (`scripts/build.sh`)
   - Runs `astro build`
   - Deploys `dist/` to GitHub Pages
   - Uses `GITHUB_TOKEN` for private repo cloning in build script
2. GitHub Pages enabled on the Foundry repo
3. Public deployment: methodology docs accessible without auth at Pages URL
4. Private deployment: all docs accessible from private repo Pages URL (restricted to collaborators)
5. Build completes in under 2 minutes
6. Site is accessible from iPad
7. Custom domain setup documented but optional

**Boundaries:**
- Do NOT set up preview deployments for PRs
- Do NOT set up monitoring

**Dependencies:** All other stories — this is the capstone

**QA:** Dan verifies from iPad that docs are readable, nav works, dark mode works, private docs are restricted.

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Mar 30 | Astro + React islands | Content-first, ships less JS, islands for interactive bits | Next.js (too heavy for content site) |
| Mar 30 | GitHub Pages (not Cloudflare) | Free, native auth via repo visibility, works from any device | Cloudflare Pages + Access (extra service, not needed) |
| Mar 30 | Directory-based public/private | Matches dir structure, config in YAML, enforced by repo visibility | Frontmatter flags (more flexible, more work) |
| Mar 30 | Deploy in E1 | Dan needs remote access from iPad to give any feedback | Separate deploy epic (would block all QA) |
| Mar 30 | Multi-repo from day one | Dan needs this for work docs too, config-driven is clean | Single repo (limits future use) |
| Mar 30 | Nav from nav.yaml | Curated titles + ordering, reuse mkdocs.yml content | Auto-generate from dirs (ugly titles, wrong order) |
| Mar 30 | Cut S5 (search) from MVP | Nav sufficient for ~30 docs. Anvil in E2. Pagefind as fallback. | Pre-built JSON index (throwaway work) |
| Mar 30 | Monorepo with packages/ | Clean separation for future API server, no rearchitect needed | Single package (would need restructure in v0.2) |
| Mar 30 | Static MVP, no server | Ship fast, infrastructure is earned complexity | AWS Amplify from day one (over-engineering) |

---

## Known Issues / Tech Debt

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| Wide table/code blowout | Medium | Open | Pages with long table rows (Assembly & Joints, Validation Pipeline) cause layout shrinkage. `overflow-x: auto` on `.content table` partially fixes but doesn't fully resolve. Needs deeper investigation — likely inline code or unbroken strings inside table cells resisting the constraint. |
| Sidebar component is hardcoded 4-level deep | Low | Open | `Sidebar.astro` manually nests 4 levels of `<details>` instead of using recursion. Works but brittle if nav depth increases. Should refactor to a recursive Astro component or use a shared render function. |
| `js-yaml` implicit dependency | Low | Mitigated | `scripts/parse-config.mjs` imports `js-yaml` — added to root `package.json` but was originally only a transitive dep via Astro. Could break if Astro drops it. |
| `about/` path in `foundry.config.yaml` | Low | Open | Config references `docs/about/` which doesn't exist in `csdlc-docs`. Build script warns and skips. Remove from config or create the directory. |
| `nav.yaml` dual maintenance | Low | Accepted | `nav.yaml` duplicates nav structure from `mkdocs.yml`. Acceptable trade-off for explicit control over titles and ordering. Could auto-generate from `mkdocs.yml` later if maintenance becomes painful. |
| MkDocs admonition syntax (`!!!`) | N/A | Resolved | Custom remark plugin (`remark-admonitions.ts`) handles this. Supports note, warning, tip, danger, info. |
| S5 (search) cut from MVP | N/A | By design | Nav sidebar sufficient for ~30 docs. Anvil semantic search comes in E2. |
| Public/private repo split | Low | Deferred | Not tested. Mechanism is GitHub repo visibility (private repo Pages = restricted to collaborators). Test when private content separation is actually needed. |
| Source repo changes don't trigger Foundry rebuild | Medium | Open | Foundry only rebuilds on pushes to the `foundry` repo. Changes to `csdlc-docs` (or any source repo) require manual `workflow_dispatch`. Fix: add `repository_dispatch` webhook — csdlc-docs push triggers Foundry rebuild automatically. |
