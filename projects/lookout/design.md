# Lookout — Visual Regression Testing MCP Server

*Status: Idea — Not Yet Refined*
*Created: April 2, 2026*
*Origin: E9 refinement discussion (Dan's idea)*

---

## The Idea

A generic, project-agnostic MCP server for visual regression testing of web applications. Any AI agent connects via mcporter, screenshots pages, compares against baselines, and flags visual diffs — no browser automation knowledge required.

Third package in the `@claymore-dev` suite alongside Anvil (semantic search) and Foundry (doc review).

### Why This Matters

Right now, visual QA is manual. Dan clicks through pages after every deploy. Agents can write code and run tests, but they're blind to what the UI actually looks like. Lookout gives agents eyes.

### Target Use Cases

- **Foundry:** Screenshot key pages after deploy, diff against baselines, catch CSS regressions (like the sticky panel bug from E7)
- **Routr/CNC App:** Verify simulator renders, canvas layouts, 3D views
- **GMPPU Race Strategy App:** Automated visual QA that doesn't require a CIO to click through dashboards
- **Any web project:** Works on any URL — not tied to any specific app

### Proposed MCP Tools

| Tool | What It Does |
|------|-------------|
| `screenshot_page` | Capture a screenshot of any URL (full page or viewport) |
| `compare_screenshots` | Diff a screenshot against a stored baseline, return similarity score + highlighted diff regions |
| `list_baselines` | List all tracked baseline screenshots with metadata |
| `approve_diff` | Accept a new screenshot as the updated baseline |
| `screenshot_element` | Capture a specific CSS selector or region (for component-level testing) |

### Architecture (Rough)

```
Agent → mcporter → Lookout MCP Server (local, stdio)
                        │
                        ▼
                   Playwright (headless)
                        │
                        ▼
                   Screenshot + pixelmatch diff
                        │
                        ▼
                   Local baseline store (~/.lookout/baselines/)
```

- **Playwright** for headless browser rendering
- **pixelmatch** or similar for image diffing
- **Local storage** for baselines (S3/cloud optional for team use)
- **stdio transport** — same pattern as Anvil and Foundry MCP servers

### What Makes This a Product (Not Just a Script)

1. **MCP-native** — agents use it like any other tool, no browser automation boilerplate
2. **Project-agnostic** — configure baseline sets per project, works on any URL
3. **Diff intelligence** — not just "images are different" but "here's what changed and where"
4. **Baseline management** — approve/reject diffs, track baseline history
5. **CI/CD integration** — run after every deploy, fail the pipeline on unexpected visual changes

### Open Questions (For Future Refinement)

- How to handle dynamic content (timestamps, animations, loading states)?
- Viewport sizes — test multiple? Default set?
- Auth for protected pages — cookie injection? Token in config?
- Threshold for "acceptable" diff (anti-aliasing, subpixel rendering)?
- Image storage format and compression for baselines?
- How does this interact with the existing `playwright-browser-automation` skill and OpenClaw `browser` tool?

---

*This is a stub. Full design doc and epic scoping will happen when we're ready to build. The idea emerged from E9 refinement and has potential as both an internal tool and a `@claymore-dev` open-source package.*
