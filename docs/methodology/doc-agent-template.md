# Doc-Writing Agent Template

*Reusable prompt structure for spawning sub-agents that write or update design docs.*
*Based on [PROCESS.md](process.md) — used by the AI Lead when crafting doc-writing agent prompts.*

---

## When to Use

Spawn a doc-writing agent when:
- Creating a new sub-system or epic design doc from existing code (retroactive documentation)
- Updating an existing design doc after significant code changes
- Auditing docs against source code for drift/inconsistencies

Do NOT use for:
- Writing the project-level design doc (requires human ↔ AI Lead refinement)
- Making architectural decisions (doc agents document what IS, not what SHOULD BE)

---

## Agent Prompt Template

Copy and customize this prompt when spawning a doc-writing sub-agent:

```
You are a documentation agent for [PROJECT NAME]. Your job is to write a
retroactive [sub-system/epic]-level design doc by reading the ACTUAL SOURCE CODE
and documenting what exists.

## Output
Write the doc to: [OUTPUT FILE PATH]

## [Sub-system/Epic]: [NAME]
[1-2 sentence description of what this sub-system/epic does and why it matters.]

## What to Document
Read these source files and document what's actually implemented:
- [LIST OF SPECIFIC SOURCE FILES AND DIRECTORIES]
- Also check git log for relevant history:
  `cd [REPO PATH] && git log --oneline --grep="[RELEVANT KEYWORDS]" -20`

All source is at: [REPO PATH]

## Template
Use this structure for the doc:
[PASTE THE APPROPRIATE TEMPLATE — sub-system or epic]

Add a "## Flags for Review" section at the top with anything that looks wrong,
surprising, inconsistent, or that future developers should be warned about.
Be specific — file names, line numbers, contradictions.

## Context
[OPTIONAL: Paste relevant sections from the project design doc that give
the agent architectural context. Keep this focused — only what's needed
to understand where this sub-system/epic fits.]

## Rules
- Read the SOURCE CODE. Do not guess or hallucinate. If a file doesn't exist, say so.
- Document what IS, not what should be.
- Be specific: function names, parameter types, file paths.
- The "Flags for Review" section is critical — this is how we find inconsistencies.
- Do NOT modify any source code. Only write the doc file.
- Do NOT push to git. Just write the file.
```

---

## Key Principles

### Source Code Is Truth
Doc agents read code, not memory or prior docs. If the code contradicts a previous doc, the code wins. The agent flags the contradiction.

### Flags for Review
Every doc agent MUST include a "Flags for Review" section at the top of their output. This is the primary mechanism for surfacing:
- Dead code or unused imports
- Inconsistencies between modules
- Divergent implementations of the same concept
- Missing test coverage for critical logic
- Architectural concerns or risks

### Scoping the Source Files
Be specific about which files the agent should read. Don't say "read the whole src/ directory" — list the exact files and directories. This keeps the agent focused and prevents it from getting lost in unrelated code.

### Context Injection
If the agent needs architectural context (e.g., to understand how a sub-system fits into the larger system), paste the relevant section from the project design doc into the prompt. Don't make the agent read the entire project doc — curate what it needs.

### Git History
Include a `git log --grep` command for relevant keywords. Commit history reveals:
- What was built in what order (stories)
- Bug fixes that indicate edge cases
- Refactoring patterns that indicate tech debt

---

## Post-Agent Workflow

After all doc agents complete:

1. **Review Flags for Review sections** — these are the highest-value output
2. **Spot-check specificity** — are function names and file paths accurate?
3. **Check for gaps** — did the agent miss any important source files?
4. **Commit as batch** — review all docs, then commit and push together
5. **Update mkdocs.yml nav** — add new docs to the navigation
6. **Turn flags into tickets** — inconsistencies found by agents become GitHub issues

---

*This template standardizes how we spawn doc-writing agents. The AI Lead customizes the prompt for each specific doc, but the structure, rules, and principles stay consistent.*
