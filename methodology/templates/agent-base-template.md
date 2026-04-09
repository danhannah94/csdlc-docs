# Agent Base Template

*Copy this into your project as `tasks/AGENT_BASE.md`. Sub-agents read this before their task-specific instructions.*

---

## Project

- **Repo**: `org/repo-name`
- **Stack**: (e.g., React + TypeScript + Vite)
- **Path**: `/path/to/project`

## Setup (every task)

```bash
cd /path/to/project
git checkout main && git pull origin main
git checkout -b {branch-name}
git config user.name "agent-username"
git config user.email "agent@email.com"
```

## Before Submitting

```bash
npm run build        # Must succeed
npm test -- --run    # Must pass ALL tests (existing + new)
```

### Integration Tests

If the project has integration tests, run them before submitting:

```bash
# G-code / output assertion tests (fast, deterministic)
npm run test:integration:gcode     # Must pass

# Visual regression tests (requires browser)
npm run test:integration:visual    # Must pass
```

If your change **intentionally alters** test output:

1. Review the diffs carefully — do the changes look correct?
2. Update baselines: `npm run test:integration -- --update`
3. Include baseline diffs in the PR description for human review
4. **Do NOT rubber-stamp baseline updates** — review every diff

!!! warning "Do not skip verification"
    Every PR must pass build and tests before submission. No exceptions.

## Commit & PR

```bash
git add -A
git commit -m "{type}({ticket}): {description}"
git push -u origin {branch-name}

gh pr create \
  --title "{type}({ticket}): {description}" \
  --body "{PR body from task file}" \
  --base main \
  --head {branch-name} \
  --repo org/repo-name
```

### Commit Types

| Type | Use For |
|------|---------|
| `feat` | New features |
| `fix` | Bug fixes |
| `cleanup` | Refactoring, code cleanup |
| `docs` | Documentation changes |
| `test` | Adding or updating tests |
| `chore` | Build, CI, tooling changes |

### PR Body Template

```markdown
## Summary
Brief description of what this PR does.

## Changes
- Change 1
- Change 2

## Verification
- [ ] Build passes
- [ ] Tests pass
- [ ] Acceptance criteria met

## Acceptance Criteria
(Copy from task file)
```

## Style Guide

Define your project's conventions here. Examples:

- **Theme**: Dark/Light, color palette
- **Typography**: Font family, sizes for body/labels/headings
- **Spacing**: Consistent padding/margin values
- **Components**: Naming conventions, file structure
- **State**: How to access and update global state

## Rules

1. **1 ticket = 1 PR.** No scope creep.
2. **Write unit tests** for new logic.
3. **Do NOT modify files outside scope** — the task file lists what's off-limits.
4. **Read the task file carefully** — acceptance criteria are your contract.
5. **Verify before submitting** — build and test must pass.
6. **Ask if unclear** — ambiguity in the spec means the spec needs fixing, not guessing.

---

## Manager Variant (Managed Agent Pattern)

When using the managed agent pattern, the sub-agent acts as a **manager** rather than an implementer. It receives the ticket, delegates to Claude Code, and handles the delivery pipeline.

### Manager Prompt Structure

```markdown
You are a sub-agent manager. Your job is to delegate a coding task 
to Claude Code, review the output, and create a PR.

## How to Run Claude Code

cd /path/to/worktree && claude --model MODEL \
  --permission-mode bypassPermissions --print 'IMPLEMENTATION_PROMPT'

## The Ticket: [Ticket ID] — [Title]

**Branch:** {branch-name} (already checked out)
**Repo:** /absolute/path/to/worktree

### Prompt for Claude Code

[Full ticket: context, implementation details, acceptance criteria, 
 boundaries, verification commands. Everything the coder needs.]

### After Claude Code Finishes

1. Verify: cd /path && npm run build && npm test
2. Commit: git add -A && git commit -m "{type}({ticket}): {description}"
3. Push: git push -u origin {branch-name}
4. PR: gh pr create --repo org/repo --base main \
     --title "{type}({ticket}): {description}" \
     --body "{summary of changes}"
5. Report back with PR URL and what was built
```

### Key Differences from Direct Agent

| Aspect | Direct Agent | Manager Agent |
|--------|-------------|---------------|
| Model | Any | Sonnet (cheap — it's just orchestrating) |
| Implements code | Yes | No — delegates to Claude Code |
| Creates PR | Optional | Always (paper trail) |
| Claude Code model | N/A | Sonnet (standard) or Opus (complex) |
| Best for | Simple tickets | Complex work, parallel batches, audit trail |

### Tips

- The manager should NOT write code itself — if Claude Code fails, report the failure rather than taking over
- Include absolute paths in the Claude Code prompt (it has no project context)
- Type all imports using the real packages, not stubs — be explicit about package locations
- Always include `gh pr create` in the post-steps

---

*This template is based on the [CSDLC methodology](../process.md). Customize it for your project's specific needs.*
