# E11 Ship Log

*April 7, 2026 — Foundry E11 deployed to production.*

This document was created live on production via MCP tools to verify the full content lifecycle:
**GitHub → import → create/edit via MCP → sync back to GitHub**

## What E11 Changed

Foundry is now a native content platform. Documents live on disk (Fly persistent volume),
not cloned from GitHub on every startup. The full CRUD cycle works through MCP tools,
and content syncs back to GitHub on demand.

## Stats

- 10 stories, 3 PRs, 5 smoke test fixes
- 49 pages imported, 1081 Anvil chunks indexed
- Net: +1,432 / -1,415 lines (same codebase size, completely different architecture)
