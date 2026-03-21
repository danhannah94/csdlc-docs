# Workflow Template — Project-Specific CSDLC Implementation

*Copy this template into your project repo as `WORKFLOW.md` and fill in the details.*
*Based on [PROCESS.md](process.md).*

---

## Project Info

- **Repo**: `org/repo-name` (public/private)
- **Stack**: (e.g., React + TypeScript + Vite + Zustand)
- **Local path**: `/path/to/project`
- **Deploy**: (e.g., GitHub Pages, Vercel, Cloudflare Pages)
- **Tests**: (e.g., `npm test -- --run`, `pytest`)
- **Build**: (e.g., `npm run build`, `cargo build`)

---

## Component / Module Map

Understanding which module owns what is critical. Sub-agents MUST target the correct files.

```
src/
├── components/         # UI components
│   ├── Feature A/
│   └── Feature B/
├── engine/             # Core logic / business rules
├── store/              # State management
├── types/              # Shared type definitions
└── utils/              # Helpers
```

!!! tip "Keep this map updated"
    Every time the architecture changes, update this section. Stale maps cause agents to target the wrong files.

### Key File Locations

| Component | File | Notes |
|-----------|------|-------|
| Main entry | `src/main.tsx` | |
| App shell | `src/App.tsx` | |
| Global store | `src/store/useStore.ts` | |
| Types | `src/types/index.ts` | |

### ⚠️ Dead / Deprecated Components

List any components that exist in code but are NOT actively rendered. Agents will try to modify these if not warned.

- `src/components/OldFeature.tsx` — replaced by `NewFeature.tsx`. Do NOT modify.

---

## Setup Instructions

```bash
# Clone and install
cd /path/to/project
git checkout main && git pull origin main
npm install  # or yarn, pnpm, pip install, etc.

# Dev server
npm run dev  # or equivalent

# Verify
npm run build
npm test -- --run
```

---

## Git & PR Requirements

### Branch Naming

- `feat/{ticket}-{description}` — new features
- `fix/{ticket}-{description}` — bug fixes
- `cleanup/{description}` — refactoring

### Every Sub-Agent Must

```bash
# 1. Setup
cd /path/to/project
git checkout main && git pull origin main
git checkout -b {type}/{ticket-id}-{description}

# 2. Configure git identity
git config user.name "agent-username"
git config user.email "agent@email.com"

# 3. Make changes, then verify
npm run build        # Must succeed
npm test -- --run    # Must pass all tests

# 4. Commit and push
git add -A
git commit -m "{type}({ticket-id}): Description"
git push -u origin {branch-name}

# 5. Create PR
gh pr create \
  --title "{type}({ticket-id}): Description" \
  --body "## Summary\n..." \
  --base main \
  --head {branch-name}
```

---

## Build / Test / Deploy Commands

| Command | Purpose | When to Run |
|---------|---------|-------------|
| `npm run build` | Production build | Before every PR |
| `npm test -- --run` | Run test suite | Before every PR |
| `npm run dev` | Local dev server | During development |
| `npm run lint` | Lint check | Optional, before PR |
| `npm run deploy` | Deploy to production | After merge to main |

---

## Style Guide

Define your project's visual and code conventions here:

### Visual Theme
- Background colors: `#___`
- Border colors: `#___`
- Text colors: primary `#___`, secondary `#___`, muted `#___`
- Error/Warning/Success states
- Font family and sizes

### Code Conventions
- File naming: (e.g., PascalCase for components, camelCase for utils)
- Import ordering: (e.g., React → third-party → local)
- State management patterns
- Error handling patterns

---

## QA Checklist

After implementation, before human review:

### Automated
- [ ] Build passes (`npm run build`)
- [ ] All tests pass (`npm test -- --run`)
- [ ] No new lint warnings
- [ ] No TypeScript errors

### Manual / Visual QA
- [ ] Feature works as described in acceptance criteria
- [ ] Edge cases tested (empty state, error state, boundary values)
- [ ] No visual regressions in adjacent components
- [ ] Responsive behavior (if applicable)
- [ ] Accessibility basics (keyboard nav, contrast, screen reader)

### QA Agent Template

```
## QA: {TICKET} (PR #{N}) — Visual QA

You are a QA agent. Verify the PR against acceptance criteria.

### Setup
cd /path/to/project
git fetch origin
git checkout {branch} && git pull origin {branch}
npm run build && npm test -- --run
npm run dev &
sleep 3

### Verify each acceptance criterion
For each AC:
- ✅ PASS — criterion met
- ❌ FAIL — criterion not met
- ⚠️ NOTE — works with minor issues

### Cleanup
# Kill dev server when done
```

---

## The Pipeline

```
1. AI Lead writes ticket with ACs
2. Sub-agent implements (1 ticket = 1 PR)
3. AI Lead code reviews
4. QA agent runs visual QA ← MANDATORY
5. Human reviews and merges
```

---

## Project Board

Link to your sprint board (GitHub Projects, Jira, Trello, etc.):

- **Board URL**: (link)
- **Labels**: `Current` (active work), `Future` (parked ideas)
- **Epic labels**: (e.g., `epic:feature-a`, `epic:feature-b`)

---

## Common Gotchas

Document project-specific pitfalls here. Examples:

1. **`localhost` → use `127.0.0.1`** — macOS resolves localhost to IPv6, causing connection issues
2. **Shared working directory** — Sub-agents share the same repo checkout. Use `git checkout -b` immediately to avoid polluting main.
3. **Environment variables** — List any required `.env` setup
4. **External services** — API keys, database connections, third-party dependencies

!!! warning "Keep this section updated"
    Every time an agent hits a surprising issue, add it here. Future agents will thank you.

---

*This workflow implements [PROCESS.md](process.md) for this specific project. Universal methodology stays in PROCESS.md; project specifics live here.*
