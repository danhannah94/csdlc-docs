# Applying CSDLC to New Projects

*A practical guide for bootstrapping the methodology on a new codebase.*

---

## Getting Started

CSDLC isn't something you install — it's something you practice. But there's a concrete setup that makes the practice work. Here's how to get a new project running.

### 1. Write a Design Doc

Before writing code, write a project-level design doc through collaborative refinement. This is the single most important first step.

- Create a project folder in the MkDocs repo: `docs/projects/your-project/`
- Write `index.md` as the project design doc — architecture, tech stack, data model, deployment, security, risks
- Go section by section with the AI Lead. Challenge everything. Resolve all ambiguity.
- Features will emerge naturally from the doc. Don't force decomposition — let it happen.

### 2. Create a Project Folder in the MkDocs Repo

```
csdlc-docs/docs/projects/your-project/
├── index.md          # Project-level design doc
└── epics/            # Epic-level design docs (as needed)
```

Update `mkdocs.yml` nav to include the new project.

### 3. Create a WORKFLOW.md in the Code Repo

The code repo gets its own `WORKFLOW.md` that implements PROCESS.md for that specific context:

- Tech stack and environment setup
- Repo conventions (branching, commit messages, PR templates)
- Component/module map
- QA procedures and templates
- Project-specific gotchas
- Agent prompt templates (`tasks/` directory)

The design doc captures WHY and WHAT. The workflow captures HOW.

### 4. Set Up Sprint Tracking

Even a simple kanban board is better than nothing. Columns should mirror the pipeline:

```
Backlog → Refined → In Progress → In Review → QA → Done
```

The specific tool (GitHub Projects, Linear, Trello, etc.) is your call — document it in WORKFLOW.md.

### 5. Start with Refinement

Your first few sessions should be heavy on refinement:
- Calibrate the AI Lead's understanding of your domain
- Build up project-specific patterns and vocabulary
- Discover what context the AI needs vs. what's noise

### 6. Retrospect Early

Don't wait for a milestone. After your first sprint or first few tickets:
- What worked? What was friction?
- Update both PROCESS.md (universal lessons) and WORKFLOW.md (project-specific lessons)
- The first retro always reveals gaps you couldn't predict

---

## Deploying to a New OpenClaw Instance

To bootstrap CSDLC on a fresh setup:

1. **Copy core files** into the new workspace:
   - `PROCESS.md` — the methodology
   - `CORE_VALUES.md` — the principles

2. **Create a starter `AGENTS.md`** that references them in the startup flow

3. **Customize identity files:**
   - `SOUL.md` — personality and communication style for the new context
   - `USER.md` — human context, preferences, working style

4. **Clone the MkDocs repo** for access to design docs and methodology documentation

5. **Create project-specific `WORKFLOW.md` files** as needed in each code repo

*Future: Package CSDLC as an installable skill for one-command setup.*
