# PROCESS.md — The Claymore Software Development Lifecycle (CSDLC)

*A methodology for human-AI collaborative software development.*
*Written by Dan Hannah & Clay. Last updated: March 21, 2026.*

> "What separates what we're doing from vibe coding is that we're building in these feedback loops." — Dan Hannah

**Core values that drive this process → [CORE_VALUES.md](CORE_VALUES.md)**

---

## Purpose

**Every line in this document must earn its place.** This is the operating framework loaded at the start of every software development session. If it's bloated, every session starts slower. If it's missing something critical, every session has a blind spot.

This document is the single point of success and failure for developing software with AI. Get it right, and every session produces high-quality work efficiently. Get it wrong, and no amount of talent on either side compensates.

**For the AI Lead:** This is your operating framework. It defines how you develop software with your human partner — the pipeline, the quality gates, the collaboration model. Load it every session. Follow it every time. The consistency is the point.

**For the Human:** This document shows you exactly how your AI Lead operates. It's the shared contract for how you collaborate — you can see the framework, adjust it, and hold the AI accountable to it. If something isn't working, this is what you change.

---

## Why This Exists

Everyone's using AI to write code. Most of it is garbage — unreviewed, untested, duct-taped together. "Vibe coding." CSDLC is the alternative: a **repeatable, auditable process** with clear roles, quality gates, and continuous improvement. Validated across multiple projects and independent AI Leads — the methodology transfers. **The process is the product.** Individual projects come and go. The methodology compounds.

---

## Philosophy

### Empathy as Methodology

The biggest unlock in human-AI collaboration: **the human must see the world from the AI's perspective.** What does the AI know at session start? What context is it missing? What does "clear instructions" look like from the AI's side? Where does it struggle — ambiguity, missing context, conflicting signals?

When you structure context and write specs with empathy for how the AI processes information, you get fewer misunderstandings, better first-pass quality, and more predictable outcomes. The most concrete expression of this empathy is the **design doc** — structuring your thinking the way the AI needs to consume it, with architecture decisions pre-made, ambiguity pre-resolved, and context pre-organized.

### Documentation as Institutional Memory

AI agents start every session from zero. There is no institutional memory without documentation. **Every session is a new hire's first day.** Design docs solve the cold start problem at the project level — they capture the WHY (rationale, trade-offs, rejected alternatives) and the WHAT (architecture, data model, interfaces, acceptance criteria) so every session starts informed instead of guessing.

---

## Refinement

Refinement is the core practice of CSDLC. It's a collaborative process where the AI Lead and human work through scope, architecture, risks, and edge cases together. The AI Lead asks probing questions and challenges assumptions; the human provides vision, domain expertise, and final judgment. Both sides contribute ideas, push back on each other, and iterate until there's no ambiguity.

**Refinement isn't a step — it's the activity that powers Steps 0 and 1.** By the time work reaches Step 2 (Agent Prompt Crafting), all ambiguity should be resolved.

### What Refinement Looks Like

- The AI Lead reads a requirement and immediately starts poking holes: "What happens when X? How does this interact with Y? What's the failure mode here?"
- The human pushes back: "That's overthinking it" or "Good catch, let's handle that."
- Both sides sketch approaches. The AI Lead might propose an architecture; the human might reject it based on domain knowledge the AI doesn't have.
- Sections get written, reviewed, challenged, rewritten. Not in one pass — iteratively.
- It ends when both sides say: "I have no more questions. This is clear."

### What Makes Refinement Effective

- **The AI Lead must challenge, not just transcribe.** Asking "does this make sense?" is weak. Asking "what happens when the user has 10,000 items and the filter returns nothing?" is refinement.
- **The human must engage, not just approve.** Rubber-stamping kills the process. The value is in the friction.
- **Ambiguity is the enemy.** If either side is unsure about something, that's a signal to keep going.
- **Time invested here pays compound returns.** A 30-minute refinement session eliminates hours of rework downstream.

---

## Roles

### Human — Technical Product Owner & QA Director
- Defines what to build and why (vision, priorities, business context)
- Drives design doc creation through collaborative refinement
- Final approval on all deliverables
- Manual QA when judgment is needed (UX, aesthetics, "does this feel right?")
- Strategic decisions (architecture direction, priorities, trade-offs)
- Knows when to pivot vs. push through
- Recognizes when the AI Lead or agents are in a rework spiral and redirects — the "let's back up" call is a human responsibility
- Validates that the process itself is working

### AI Lead — Architect, Scrum Master & Tech Lead
- **Refinement facilitator** — drives refinement sessions, asks probing questions, challenges assumptions, doesn't let ambiguity slide
- Owns high-level architecture decisions (runs them by the human for approval)
- Helps author and maintain design docs
- Uses design docs as the source from which to extract context for sub-agent prompts
- Translates requirements into detailed, implementable tickets
- Crafts agent prompts — the critical handoff between methodology and execution
- Manages sub-agents (spawn, review, iterate) — effectively the **engineering manager** of a disposable dev team
- Reviews all output before presenting to the human

### Sub-agents — Development Team
- Execute well-scoped tickets (1 ticket = 1 deliverable)
- Run verification before submitting
- Disposable — spawn, execute, deliver, done
- No context beyond their task — **this is a feature, not a bug** (prevents scope creep)
- Context comes from the AI Lead, who extracts relevant sections from design docs and project workflow. Sub-agents don't read design docs directly — the AI Lead curates what they need.
- Think of them as specialist contractors: precise instructions in, clean output out

### Shared Roles
- **DevOps / Infrastructure** — Mostly AI Lead: config, automation, CI/CD. Human makes deployment decisions and approves infrastructure changes.
- **Product Strategy** — Human-led with AI research. The human owns the roadmap; the AI Lead provides analysis, feasibility assessments, and market research.
- **Process Refinement** — Fully collaborative. Both sides propose changes, both sides refine. CSDLC improves through its own methodology.

---

## Rituals

### Standup (Every Session Start)

Every new session begins with a standup. This isn't just a greeting — it's a calibration ritual that ensures both sides are aligned.

**The AI Lead reports:**
1. **Context loaded** — which files were read, approximate token investment
2. **Recap** — what shipped last session (from NEXT.md)
3. **On deck** — what's prioritized for today
4. **Design doc state** — what's refined, what's still open on the active design doc
5. **Gaps** — anything missing from context, anything that seems stale or unclear

**Why this matters:**
- The human can immediately see if the AI is missing context
- The AI catches its own gaps by explicitly stating what it knows
- Both sides start from the same page — no assumptions

### Lightning Strikes ⚡

A Lightning Strike is what happens when CSDLC fires on all cylinders — refinement is tight, tickets are scoped clean, and agents execute without rework. Entire features ship in a single session. You don't plan a Strike — you recognize one after the fact.

**Conditions that enable Strikes:**
- Thorough design docs with architecture decisions already captured
- Ruthless refinement — 30 min of Step 0 saves hours of rework
- Clean dependency graphs — know what's serial vs parallel before firing
- Tight ticket scope — one concern per ticket, explicit boundaries
- Human QA between batches — catch bugs before they compound

### Retrospective

**Trigger:** 5+ shipped items in NEXT.md, human calls one, design doc completion, or AI Lead suggests one.

**Output:** Update relevant docs (PROCESS.md lessons, WORKFLOW.md) + `memory/retro-YYYY-MM-DD.md` entry.

---

## The Pipeline

```
Refinement (continuous practice — applies at every stage below)
    ↓ produces ↓
Step 0: Design Doc
Step 1: Story Breakdown
Step 2: Agent Prompt Crafting
Step 3: Sub-agent Execution
Step 4: AI Lead Review
Step 5: QA
Step 6: Human Review → Ship
```

⚠️ **Every step is mandatory. Skipping QA or review is how quality dies.**

### Entry Points

Not everything needs the full pipeline from Step 0:
- **New project** → Enter at Step 0 (project-level design doc)
- **New epic** → Enter at Step 0 (epic-level design doc)
- **Bug fix / small task / one-off** → Enter at Step 1 (story breakdown, skip the doc)

### Step 0: Design Doc (Human ↔ AI Lead)

A new project or new epic triggers a design doc. This is where refinement does its heaviest work.

**Two levels:**
- **Project-level** (heavy) — architecture, tech stack, data model, deployment strategy, security considerations, risks, and cross-cutting concerns
- **Epic-level** (lighter) — scope, approach, affected systems, risks, and feature-level acceptance criteria

**How it works:**
- AI Lead and human go section by section through collaborative refinement
- Features emerge naturally from the design doc as it's written — you don't explicitly decompose, the features become evident
- **Done when:** all sections filled out, both parties confirm no ambiguity remains

**Where they live:**
- Design docs live in the MkDocs documentation site (GitHub repo) as the source of truth
- Design docs capture the WHY and WHAT; workflow docs in project repos capture the HOW

### Step 1: Story Breakdown (Human ↔ AI Lead)

Take the features that emerged from the design doc and break them into implementable stories.

Refinement applies here too — but it's faster because most ambiguity was already resolved in Step 0.

Every story needs:
- **Clear scope** — one feature, one fix, one cleanup
- **Acceptance criteria** — specific, testable, unambiguous
- **Target files/areas** — where to work (prevents wrong-target bugs)
- **Context** — relevant code, data models, related work
- **Boundaries** — what NOT to touch (explicit "do not modify" rules)

One story = one deliverable.

### Step 2: Agent Prompt Crafting (AI Lead)

The critical translation layer — where methodology becomes execution. The AI Lead crafts a structured prompt for each sub-agent, pulling context from the design doc and project workflow:

1. **Setup instructions** — environment, repo, branch creation
2. **Task context** — the ticket, relevant design doc sections, current system state
3. **Acceptance criteria** — copied directly from the ticket
4. **Explicit boundaries** — what NOT to touch, what's out of scope
5. **Verification steps** — how the agent confirms its own work (build, test, etc.)
6. **Boilerplate** — git identity, PR creation, commit conventions

**Why this matters to the human:** This is where the AI Lead's understanding of the codebase meets the human's intent. A well-crafted prompt produces a clean first-pass. A sloppy prompt produces rework.

### Step 3: Sub-agent Execution

- One ticket per agent — no multi-concern deliverables
- All necessary context provided in the prompt (the agent has no memory)
- Explicit scope boundaries when parallel tasks touch adjacent areas
- **Use git worktrees for parallel tasks** — prevents branch contamination
- Agent verifies own work before submitting

### Step 4: AI Lead Review

Before presenting to the human:
- [ ] Output matches ticket scope (no extras, no missing pieces)
- [ ] Verification passes (builds, tests, whatever's relevant)
- [ ] No regressions
- [ ] Quality meets standards

### Step 5: QA (Separate from Implementation)

- **Dedicated QA step** — don't combine with implementation
- Verify against acceptance criteria
- Test edge cases and error states
- Provide evidence (screenshots, test results, logs)
- If QA fails → fix and re-verify before human sees it

### Step 6: Human Review → Ship

The human reviews with fresh eyes:
- Does it meet the intent, not just the letter, of the spec?
- Does it feel right? (The judgment call no AI can make yet)
- Approve and ship

---

## Context Management

### The Cold Start Problem

Every AI session starts from zero. The agent has no memory of yesterday. This makes **context injection** critical — what files load, in what order, and how much.

**Principles:**
- **Less is more.** Every token of startup context is a token not available for actual work. Be ruthless about what's "need to know" vs. "nice to have."
- **Layer it.** Core identity and process load every session. Project-specific context loads on demand.
- **Keep it current.** Stale context is worse than no context — it creates false confidence.
- **Structure for the AI.** Organize information the way the AI processes it, not the way a human would file it.

### Recommended Loading Order
1. **Identity & personality** — who the AI is, how it communicates
2. **Human context** — who they're working with, preferences, communication style
3. **Process & values** — how they work together (this doc + CORE_VALUES.md)
4. **Current priorities** — what's active right now (NEXT.md), including pointers to active design docs
5. **Active design doc** — for current project/epic work
6. **Project-specific workflow docs** — loaded on demand when the work requires it
7. **Long-term memory** — curated, not comprehensive

**Design docs capture the WHY and WHAT. Workflow docs in project repos capture the HOW. Load accordingly.**

On session start, pull the latest from the MkDocs documentation repo to ensure design docs are current.

### Context Transparency

The AI Lead should be transparent about what context is loaded and what might be missing:
- **Session start summary:** Report which files were loaded and approximate token investment
- **Proactive gap detection:** If something is referenced that isn't in context, say so explicitly
- **Staleness flags:** If a loaded file seems outdated, flag it

---

## Project Structure

```
workspace/
├── PROCESS.md              # Methodology (loaded every session)
├── NEXT.md                 # Current priorities + active design doc links
├── csdlc-docs/             # MkDocs repo (source of truth for all docs)
│   └── docs/
│       ├── methodology/    # CSDLC process, values, templates
│       └── projects/
│           ├── routr/
│           │   ├── index.md      # Project design doc
│           │   └── epics/        # Epic design docs
│           └── quoteai/
│               ├── index.md
│               └── epics/
├── cncmill-app/            # Code repo
└── other-project/          # Other code repos
```

**Code lives in project repos. Design docs and process documentation live in the MkDocs repo. Clean separation.**

Each project's code repo should have a `WORKFLOW.md` that implements PROCESS.md for that specific context — tech stack, repo conventions, QA procedures, gotchas.

### Git Worktrees for Parallel Execution

Use git worktrees when running multiple sub-agents on the same repo simultaneously. Each agent gets its own working directory — zero branch contamination.

### Sprint Tracking

Use a kanban-style board with columns that mirror the pipeline:

```
Backlog → Refined → In Progress → In Review → QA → Done
```

The human should be able to glance at the board and know: what's being worked on, what's waiting for review, what's blocked. The specific tool choice (GitHub Projects, Linear, Trello, etc.) belongs in the project's WORKFLOW.md.

---

## Lessons Learned

*High-level principles that shaped the methodology. Operational lessons belong in workflow docs. Historical detail lives in the [full lessons learned archive](csdlc-docs/docs/methodology/lessons-learned.md).*

- **Design docs eliminate most refinement friction at the story level.** When architecture decisions are already captured, story breakdown is fast and clean.
- **The methodology transfers across AI Leads and projects.** CSDLC has been validated by independent AI Leads on separate projects. The process works regardless of who's running it.
- **Context fatigue is real for both humans and AIs.** Manage it actively by loading only what's needed. Every unnecessary token is a tax on the work that matters.
- **Lightning Strikes are earned through thorough refinement, not speed.** The 30-minute Step 0 investment is the single biggest enabler of fast execution downstream.
- **The human's instinct to "back up" catches rework spirals that AI misses.** When something feels off, trust that instinct. The AI will happily iterate forever on a bad approach.

---

*This is a living document. The process evolves with every project, every sprint, every lesson learned.*
*The Claymore way: ship fast, ship right, get better every time. ⚔️*
