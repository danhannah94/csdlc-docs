# Lessons Learned — Historical Archive

*This is the full historical record of lessons learned through CSDLC practice. Key principles are embedded in the main [CSDLC Process](process.md) document. This archive preserves the detail and context behind those principles.*

---

## Origins

- **Lightning Strikes Origin:** March 4, 2026 — shipped the entire Tier 2 Path Builder epic (14 PRs, ~24 min agent time) in one session. Dan said "that's not a sprint, that's a lightning strike." The name stuck.

## Process

- **Acceptance criteria are non-negotiable.** "How to verify" must be on every ticket. Without it, QA becomes opinion instead of evidence.
- **Visual/manual QA catches what automated tests miss.** Something can pass tests and still be wrong. The human eye catches interaction bugs, layout drift, and "this doesn't feel right" problems.
- **Include file/area targets in tickets.** Agents don't have full project context — tell them exactly where to work. Without targets, agents guess, and guesses create wrong-target bugs.
- **Include "Do NOT modify" sections.** Prevents agents from "helpfully" refactoring adjacent code. Explicit boundaries are cheaper than rework.
- **Don't skip QA.** Every time you skip it, something slips through. Every time. This lesson has been learned repeatedly, which is itself the lesson.
- **Adjacent scope bleed is real.** When parallel tickets touch related areas, agents may implement each other's scope. Add explicit boundaries between parallel work.
- **Retros aren't overhead — they're compound interest.** Every lesson documented is a mistake never repeated. The 10 minutes spent writing a retro saves hours downstream.
- **Design docs eliminate most refinement friction at the story level.** When architecture decisions are already captured in a design doc, story breakdown becomes a fast mechanical exercise instead of a drawn-out negotiation. This was the single biggest process improvement discovered.
- **Lightning Strikes are earned through thorough refinement, not speed.** The 30-minute Step 0 investment is the single biggest enabler of fast execution. Rushing refinement doesn't make you faster — it makes you slower with extra rework.

## Architecture

- **Know your system.** Before writing a ticket, verify the current state. Systems evolve fast — yesterday's architecture diagram may be wrong. Agents working from stale context produce stale solutions.
- **Spike before implementing complex features.** Discover fundamental issues early, not mid-implementation. A 20-minute spike is cheaper than a 3-hour rework.
- **One ticket, one deliverable.** Multi-concern work is harder to review, harder to revert, and creates merge conflicts. Keep it atomic.

## Human-AI Dynamics

- **Direct action over explanation.** Show results, not plans. Humans trust what they can see and interact with.
- **Match the human's energy.** Excitement for breakthroughs, pragmatism for boring stuff. Read the room.
- **Full transparency about risks and trade-offs.** Trust comes from honesty, not optimism. Flag concerns early, even if the human doesn't want to hear them.
- **Self-healing loops:** failure → agent retry → AI Lead escalation → human intervention. Don't block on first failure, but don't hide recurring failures either.
- **Context fatigue is real.** Both for humans (too many PRs to review) and AIs (too much startup context). Manage it actively — batch reviews, trim context, take breaks.
- **Standup is calibration, not ceremony.** Quick, honest, useful. If it feels like overhead, it's too heavy.
- **The human's QA instinct catches what agents miss.** Interaction bugs (click-vs-drag, edge behavior) are judgment calls that need human eyes. Never skip QA.
- **The human's "back up" instinct catches rework spirals.** AI agents will happily iterate forever on a bad approach. When the human says "let's back up," that's not hesitation — it's pattern recognition. Trust it.
- **The methodology transfers across AI Leads and projects.** CSDLC has been validated by independent AI Leads on separate projects. This isn't just one team's process — it's a methodology that works regardless of who's running it.
