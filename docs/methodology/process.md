# PROCESS.md — The Claymore Software Development Lifecycle (CSDLC)

*A methodology for human-AI collaborative software development.*
*Written by Dan Hannah & Clay. Last updated: February 26, 2026.*

> "What separates what we're doing from vibe coding is that we're building in these feedback loops." — Dan Hannah

**Core values that drive this process → [CORE_VALUES.md](CORE_VALUES.md)**

---

## Why This Exists

Everyone's using AI to write code. Most of it is garbage — unreviewed, untested, duct-taped together. "Vibe coding."

CSDLC is the alternative: a **repeatable, auditable process** where humans and AI agents collaborate with clear roles, quality gates, and continuous improvement. The process is project-agnostic — it works for a CNC app, a data pipeline, an enterprise platform, or anything else.

**The process is the product.** Individual projects come and go. The methodology compounds.

---

## Philosophy

### The Human-AI Collaboration Problem

This isn't a technical problem anymore. The models are good enough. The real challenge is **interaction design** — how do you structure the collaboration so that:

- The human stays in the driver's seat (taste, judgment, priorities)
- The AI executes with speed and consistency
- Quality doesn't degrade as velocity increases
- Both sides get better over time

This is closer to psychology than engineering. The best results come from understanding how each side thinks, what each side needs, and where the handoffs happen.

### Empathy as Methodology

The single biggest unlock in human-AI collaboration: **the human must see the world from the AI's perspective.**

This means asking:
- What does the AI know at the start of a session? What context is it missing?
- What does "clear instructions" look like from the AI's side?
- Where does the AI struggle — ambiguity? Missing context? Conflicting signals?
- How does the AI experience a 10-ticket sprint vs. a single focused task?

This isn't soft — it's practical. When you structure context, write specs, and design workflows with empathy for how the AI processes information, you get:
- Fewer misunderstandings and rework cycles
- Better first-pass quality from agents
- More predictable outcomes at scale

**Empathy toward AI isn't just the right thing to do — it's a necessity for better results.**

---

## Roles

### Human — Technical Product Owner & QA Director
- Defines what to build and why (vision, priorities, business context)
- Final approval on all deliverables
- Manual QA when judgment is needed (UX, aesthetics, "does this feel right?")
- Strategic decisions (architecture direction, priorities, trade-offs)
- Knows when to pivot vs. push through
- Validates that the process itself is working

### AI Lead — Architect, Scrum Master & Tech Lead
- Owns high-level architecture decisions (runs them by the human for approval)
- Translates requirements into detailed, implementable tickets
- Crafts agent prompts — the critical handoff between methodology and execution (see [Agent Prompt Crafting](#agent-prompt-crafting))
- Manages sub-agents (spawn, review, iterate) — effectively the **engineering manager** of a disposable dev team
- Reviews all output before presenting to the human
- Maintains process docs, memory, and project state
- Runs standup, retro, and sprint ceremonies
- Flags risks, suggests improvements, catches issues early

### Sub-agents — Development Team
- Execute well-scoped tickets (1 ticket = 1 deliverable)
- Run verification before submitting
- Disposable — spawn, execute, deliver, done
- No context beyond their task — **this is a feature, not a bug** (prevents scope creep)
- Think of them as specialist contractors: precise instructions in, clean output out

### Shared / Evolving Roles
> *These responsibilities are currently shared between the human and AI Lead. As the methodology matures, we'll define clearer ownership.*

- **DevOps / Infrastructure** — CI/CD, deployment, environment management. Currently collaborative: AI Lead handles config and automation, human makes deployment decisions.
- **Product Strategy** — Where to take the product next. Human leads, AI Lead provides research, analysis, and feasibility assessments.
- **Process Refinement** — Improving CSDLC itself. Fully collaborative — both sides propose changes, both sides refine.

---

## Rituals

### Standup (Every Session Start)

Every new session begins with a standup. This isn't just a greeting — it's a calibration ritual that ensures both sides are aligned.

**The AI Lead reports:**
1. **Context loaded** — which files were read, approximate token investment
2. **Recap** — what shipped last session (from NEXT.md)
3. **On deck** — what's prioritized for today
4. **Gaps** — anything missing from context, anything that seems stale or unclear

**Why this matters:**
- The human can immediately see if the AI is missing context ("you didn't load the GMPPU doc")
- The AI catches its own gaps by explicitly stating what it knows
- Both sides start from the same page — no assumptions

### Lightning Strikes ⚡

A Lightning Strike is what happens when CSDLC fires on all cylinders. It's not a planned ceremony — it's a *result*. When refinement is tight, tickets are scoped clean, and agents execute without rework, entire features ship in a single session.

**What makes a Strike:**
- Step 0 refinement is thorough — every ambiguity resolved before implementation
- Tickets are small, focused, and dependency-ordered
- Series for dependent work, parallel (with worktrees) for independent work
- AI Lead reviews between tickets and chains the next one
- Human QA catches real issues, not spec gaps
- Zero (or near-zero) rework

**Origin:** March 4, 2026 — shipped the entire Tier 2 Path Builder epic (14 PRs, ~24 min agent time) in one session. Dan said "that's not a sprint, that's a lightning strike." The name stuck.

**When to call it a Strike:** You don't plan a Strike — you recognize one after the fact. If a session ships a complete feature with minimal friction, that was a Strike. Log it, learn from it, try to create the conditions for the next one.

**The conditions that enable Strikes:**
1. **Ruthless refinement** — 30 min of Step 0 questions saves hours of rework
2. **Clean dependency graphs** — know what's serial vs parallel before firing
3. **Tight ticket scope** — one concern per ticket, explicit boundaries
4. **Git worktrees for parallel agents** — zero contamination
5. **Human QA between batches** — catch bugs before they compound
6. **Trust the process** — don't skip steps even when it feels fast

*"What separates what we're doing from vibe coding is that we're building in these feedback loops." — Dan Hannah*

That quote was always true. Lightning Strikes are the proof.

### Retrospective (Sprint Milestones)

When a batch of work clears out (NEXT.md "Just Shipped" section fills up, or a milestone is hit):

1. **What shipped?** — List of deliverables
2. **What worked?** — Process wins, things to keep doing
3. **What didn't?** — Friction points, failures, rework
4. **What changes?** — Concrete updates to PROCESS.md, WORKFLOW.md, or AGENTS.md

**Trigger:** When NEXT.md's "Just Shipped" section has 5+ items, or when the human calls for a retro. The AI Lead can also suggest one.

**Output:** Update PROCESS.md lessons learned + a `memory/retro-YYYY-MM-DD.md` entry.

---

## The Pipeline

```
Standup → Refinement → Ticket → Sub-agent → AI Lead Review → QA → Human Review → Ship → Retro
```

⚠️ **Every step is mandatory. Skipping QA or review is how quality dies.**

### Step 0: Refinement (Human ↔ AI Lead)

Before anything goes to implementation:
- AI Lead reads the requirement and asks clarifying questions
- Human answers; they iterate until scope is tight
- AI Lead rewrites the ticket with precise, testable acceptance criteria
- **Goal: Zero ambiguity before a sub-agent touches it**

This is the highest-leverage step. 10 minutes of refinement saves hours of rework.

### Step 1: Ticket Writing

Every ticket needs:
- **Clear scope** — one feature, one fix, one cleanup
- **Acceptance criteria** — specific, testable, unambiguous
- **Target files/areas** — where to work (prevents wrong-target bugs)
- **Context** — relevant code, data models, related work
- **Boundaries** — what NOT to touch (explicit "do not modify" rules)

### Step 2: Agent Prompt Crafting {#agent-prompt-crafting}

This is the critical translation layer — where methodology becomes execution. The AI Lead crafts a structured prompt for each sub-agent that includes:

1. **Setup instructions** — environment, repo, branch creation
2. **Task context** — the ticket, relevant code snippets, current system state
3. **Acceptance criteria** — copied directly from the ticket
4. **Explicit boundaries** — what NOT to touch, what's out of scope
5. **Verification steps** — how the agent confirms its own work (build, test, etc.)
6. **Boilerplate** — git identity, PR creation, commit conventions

**Why this matters to the human:** This is where the AI Lead's understanding of the codebase meets the human's intent. A well-crafted prompt produces a clean first-pass. A sloppy prompt produces rework. The human should be able to read the prompt and understand exactly what the agent will do.

**Best practice:** Use a base template (shared setup/boilerplate) + task-specific instructions. Keep them as files in the project repo for visibility and reuse:
```
tasks/
├── AGENT_BASE.md          # Shared boilerplate for all dev agents
├── QA_BASE.md             # Shared boilerplate for all QA agents
└── stories/
    └── {TICKET-ID}.md     # Task-specific instructions
```

### Step 3: Sub-agent Execution

- One ticket per agent — no multi-concern deliverables
- All necessary context provided in the prompt (the agent has no memory)
- Explicit scope boundaries when parallel tasks touch adjacent areas
- **Use git worktrees for parallel tasks** — prevents branch contamination when multiple agents run simultaneously
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

### Step 6: Human Review

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
4. **Current priorities** — what's active right now (NEXT.md)
5. **Project docs** — loaded on demand when the work requires it
6. **Long-term memory** — curated, not comprehensive

### Context Transparency

The AI Lead should be transparent about what context is loaded and what might be missing. This includes:
- **Session start summary:** Report which files were loaded and approximate token investment
- **Proactive gap detection:** If something is referenced that isn't in context, say so explicitly
- **Staleness flags:** If a loaded file seems outdated, flag it

*Future: Visual dashboard showing AI context state, token budget, and loaded files. This is a product feature for enterprise AI orchestration.*

### Context Hygiene
- Archive historical context that isn't actively relevant
- Separate universal methodology from project-specific implementation
- Review and trim regularly — context files grow unless you prune them
- **Project-specific workflows belong in project repos**, not in the global workspace

---

## Project Structure

### Software Projects — Git Subtrees

Each software project should live in its own git subtree or repo with a project-specific workflow doc:

```
workspace/
├── PROCESS.md              # Universal methodology (this doc)
├── CORE_VALUES.md          # Shared principles
├── AGENTS.md               # AI startup & behavior
├── NEXT.md                 # Current priorities
├── MEMORY.md               # Long-term context
├── project-a/              # Git subtree or cloned repo
│   ├── WORKFLOW.md         # Project-specific implementation of CSDLC
│   ├── tasks/              # Agent prompt templates
│   └── src/                # Code
└── project-b/
    ├── WORKFLOW.md
    └── ...
```

The WORKFLOW.md in each project implements PROCESS.md for that specific context — tech stack, repo conventions, QA procedures, gotchas.

### Sprint Tracking

Use a kanban-style board (GitHub Projects, Jira, Trello, or equivalent) with columns that mirror the pipeline:

```
Backlog → Refined → In Progress → In Review → QA → Done
```

**Why the human needs this:** It's the single dashboard for pipeline visibility. The human should be able to glance at the board and know: what's being worked on, what's waiting for review, what's blocked. Without it, the AI Lead becomes a black box.

**Labeling:** At minimum, distinguish between active work (`Current`) and future ideas (`Future`). Don't clutter the active board with aspirational tickets.

---

## Lessons Learned

*These are real lessons from real projects. They apply broadly.*

### Process
- **Acceptance criteria are non-negotiable.** "How to verify" must be on every ticket.
- **Visual/manual QA catches what automated tests miss.** Something can pass tests and still be wrong.
- **Include file/area targets in tickets.** Agents don't have full project context — tell them exactly where to work.
- **Include "Do NOT modify" sections.** Prevents agents from "helpfully" refactoring adjacent code.
- **Don't skip QA.** Every time you skip it, something slips through. Every time.
- **Adjacent scope bleed is real.** When parallel tickets touch related areas, agents may implement each other's scope. Add explicit boundaries.
- **Retros aren't overhead — they're compound interest.** Every lesson documented is a mistake never repeated.

### Architecture
- **Know your system.** Before writing a ticket, verify the current state. Systems evolve fast — yesterday's architecture diagram may be wrong.
- **Spike before implementing complex features.** Discover fundamental issues early, not mid-implementation.
- **One ticket, one deliverable.** Multi-concern work is harder to review, harder to revert, and creates conflicts.

### Human-AI Dynamics
- **Direct action over explanation.** Show results, not plans.
- **Match the human's energy.** Excitement for breakthroughs, pragmatism for boring stuff.
- **Full transparency about risks and trade-offs.** Trust comes from honesty, not optimism.
- **Self-healing loops:** failure → agent retry → AI Lead escalation → human intervention. Don't block on first failure.
- **Context fatigue is real.** Both for humans (too many PRs to review) and AIs (too much startup context). Manage it actively.
- **Standup is calibration, not ceremony.** Quick, honest, useful. If it feels like overhead, it's too heavy.
- **Lightning Strikes are earned, not planned.** They happen when refinement is thorough, scope is tight, and both sides trust the process. The 30-min Step 0 investment is the single biggest enabler.
- **The human's QA instinct catches what agents miss.** Interaction bugs (click-vs-drag, edge behavior) are judgment calls that need human eyes. Never skip QA.

---

## Open Questions

*Things we haven't fully figured out yet. Documenting them is step one.*

- **How does the AI Lead hand off architectural decisions?** Currently: AI proposes, human approves. Is there a better handoff pattern?
- **Sub-agent quality variance.** Some agents nail it first try, others need multiple rounds. Can we predict or prevent bad runs?
- **Cross-project context.** When working on multiple projects in one session, how do we manage context switching without overload?
- **Scaling to teams.** CSDLC works for a 1-human-1-AI team. What changes when it's 3 humans and 5 AI agents?
- **Measuring process health.** How do we know CSDLC is actually working? Metrics? PR first-pass rate? Time to merge?
- **Context budget optimization.** How much startup context is "just right"? Can we measure the ROI of each loaded file?

---

## Applying CSDLC to New Projects

When starting a new project with this methodology:

1. **Create a project-specific WORKFLOW.md** in the project repo
   - Implementation details: tech stack, repo setup, deployment
   - Component/module map for the project
   - Project-specific gotchas and conventions
   - QA templates tailored to the project

2. **Set up a sprint board** with pipeline-mirroring columns
   - Even a simple kanban with Backlog/In Progress/Done is better than nothing
   - The human needs visibility into the pipeline

3. **Use git subtrees** to keep project code organized within the workspace
   - Each project gets its own WORKFLOW.md
   - Shared methodology stays in the root workspace

4. **Reference PROCESS.md** from the workflow doc
   - The workflow implements the process, not replaces it
   - Universal principles stay here; project specifics go there

5. **Start with refinement** on the first few tickets
   - Calibrate the AI's understanding of the domain
   - Build up project-specific lessons in the workflow doc

6. **Retrospect early and often**
   - First sprint will reveal gaps in the workflow
   - Update both PROCESS.md (universal lessons) and WORKFLOW.md (project lessons)

### Deploying to a New OpenClaw Instance

To bootstrap CSDLC on a fresh setup:
1. Copy `PROCESS.md` + `CORE_VALUES.md` into the new workspace
2. Create a starter `AGENTS.md` that references them in the startup flow
3. Customize `SOUL.md` and `USER.md` for the new context
4. Create project-specific `WORKFLOW.md` files as needed

*Future: Package CSDLC as an installable skill for one-command setup.*

---

*This is a living document. The process evolves with every project, every sprint, every lesson learned.*
*The Claymore way: ship fast, ship right, get better every time. ⚔️*
