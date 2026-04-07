# E7: UI/UX Refinement — Design Doc

*Status: Step 0 — Refinement In Progress*
*Created: April 1, 2026*
*Authors: Clay (draft), Dan (review)*

---

## Overview

E7 is a UI polish epic. No new features, no architecture changes — just making what we shipped in E1–E6 look and feel like a modern documentation platform. The goal is to go from "functional prototype" to "something you'd actually show a CIO."

### Why Now

Foundry works. The review loop works. Auth works. But the visual presentation has rough edges: the thread panel feels boxy, margins are off on desktop, the nav sidebar looks like a file tree, and the color palette is... default. Before we demo this at GMPPU or pitch it externally, we need to look the part.

### Scope

All changes are CSS/component-level. No API changes, no schema changes, no new dependencies. Auto-nav generation and cross-repo rebuild triggers are deferred to **E8: Live Content Pipeline** (SSR migration).

---

## Current State Assessment

Based on live inspection of the deployed site (April 1, 2026):

### Thread Panel Issues
1. **Boxy comment cards** — `.thread-comment` has `background`, `border`, and `border-radius: 8px`. Every comment is its own bordered box, creating visual clutter
2. **Right border clipping** — The thread toggle arrow (`→`) gets clipped when the panel is at `--thread-panel-width: 320px`. The `thread-panel--hidden` state positions the header at `left: -40px` but the grid column shrinks to `40px`, causing tight overflow
3. **Review groups double-border** — `.thread-review-group` has its own border, then each comment inside has another border. Border inception
4. **No thread collapse** — All replies are always visible, which gets noisy on docs with many comment threads

### Desktop Layout Issues
5. **Content area not centered** — `.content-area` has `max-width: var(--content-max-width)` (800px) but no `margin: 0 auto`. Content hugs the left side of its grid column
6. **Fixed max-width wastes space** — On wide screens (especially work monitors), content sits narrow on the left with huge dead space to the right. Should fill available space fluidly.

### Navigation Sidebar
7. **Panel chrome feels heavy** — Solid background (`--color-sidebar-bg`), hard border-right, feels like a separate application region rather than part of the doc
8. **Collapsed arrows (`▸`) are CSS `::before` content** — Works but looks dated. Modern doc sites use smooth rotation or chevron SVGs
9. **4-level hardcoded nesting** — Known tech debt. Not fixing the nesting depth in E7, but styling changes should work with any depth

### Color Palette
10. **Current dark theme** — Very dark navy (`#0f172a` bg, `#1e293b` secondary). Functional but feels cold/corporate
11. **Accent color** — Deep orange (`#ea580c`) works fine. Distinctive, not generic blue. Keep it.
12. **Light theme** — Standard white/gray. Clean but bland. Could use warmer tones
13. **No design tokens** — We have CSS custom properties, which IS the token system. Just need better values.

---

## Design Decisions

### D1: Thread Panel — Slack-Style Threaded Timeline

**Decision:** Replace bordered card-per-comment with a Slack-style threaded timeline.

**Current:**
```
┌──────────────────────┐
│ 💬 Dan  · 3h ago     │
│ ┌────────────────────┐│
│ │ "quoted text"      ││
│ └────────────────────┘│
│ Comment content       │
└──────────────────────┘
┌──────────────────────┐
│ 🏗️ Clay · 2h ago     │
│ Reply content         │
└──────────────────────┘
```

**Proposed (Slack-style):**
```
💬 Dan · 3h ago
│ "quoted text"
│ Comment content
│ 💬 2 replies ▸        ← collapsed by default
│
│ [expanded:]
│ ├─ 🏗️ Clay · 2h ago
│ │  Reply content
│ └─ 💬 Dan · 1h ago
│    Follow-up
```

**Key behaviors (refined with Dan):**
- **Threads collapse by default** — parent comment visible, replies hidden behind "N replies" indicator
- Click to expand replies inline (NOT a separate overlay — this isn't Slack's full thread view)
- Click again to collapse
- **Active / Archived split** — thread panel has two sections:
  - **Active** — unresolved threads (collapsed parent + reply count)
  - **Archived** — resolved threads in a collapsible section at the bottom (like GitHub PR "resolved conversations")
- Resolved threads move to Archived, keeping the Active area focused on what needs attention
- Archived section is collapsed by default, expandable to review past discussions

**Implementation:**
- Remove `background`, `border`, `border-radius` from `.thread-comment`
- Add a left-border timeline indicator (2px `--color-border` or accent-tinted)
- Use spacing (`margin-bottom`, `padding-bottom` + subtle separator) instead of boxes
- Keep hover state but make it a subtle background tint, not a box highlight
- Review groups: remove `.thread-review-group` border, use a section header with separator
- Add collapse/expand state per thread (React state, not persisted)
- Add Active/Archived section split with collapsible Archived header

### D2: Fix Thread Panel Clipping

**Decision:** Increase toggle tab width and adjust grid column for hidden state.

Changes:
- Hidden state grid: `40px` → `44px` for the collapse column
- Toggle tab: `left: -40px; width: 40px` → `left: -44px; width: 44px`
- Add `overflow: visible` on `.thread-panel` to prevent clipping
- Ensure the toggle arrow has adequate padding

### D3: Desktop Content — Fluid Fill Layout

**Decision:** Content area fills available space fluidly instead of fixed max-width.

**Refined with Dan:** The fixed 800px max-width wastes screen real estate, especially on work monitors. Content should expand to fill the space between sidebar and thread panel.

Changes:
- Remove `--content-max-width` cap entirely
- `.content-area`: fluid width with comfortable margins (`2rem` on each side)
- When sidebar hidden: content expands left
- When thread panel hidden: content expands right
- When both hidden: content fills viewport with margins
- This matches MkDocs Material's behavior — content breathes on wide screens without feeling like a wall of text

### D4: Navigation Sidebar Restyle — MkDocs Material Reference

**Decision:** Copy MkDocs Material's sidebar styling. It's the gold standard for doc nav.

**Reference:** [MkDocs Material sidebar](https://squidfunk.github.io/mkdocs-material/)

Changes:
- Remove or significantly lighten `border-right`
- Sidebar background: match main background (`--color-bg`) — no separate panel color
- Replace `▸` text arrows with inline SVG chevrons that rotate smoothly (CSS `transform: rotate(90deg)` with transition)
- Active page indicator: left accent bar (3px `--color-accent`) instead of background highlight
- Section titles: slightly bolder weight, uppercase or small-caps for top-level categories
- Reduce sidebar padding for a tighter, more integrated feel
- Keep our own color tokens — just steal the layout/interaction patterns

### D5: Color Palette Refresh (Iterative)

**Decision:** Evolve the existing palette, don't overhaul. Keep the orange accent. Warm up the dark theme. **Ship first pass, then iterate based on how it looks live.**

Colors are hard to judge from hex values in a table. S5 ships the initial warmer palette, Dan eyeballs it on the live site, and we adjust in follow-up commits.

#### Dark Theme Updates
| Token | Current | Proposed | Rationale |
|---|---|---|---|
| `--color-bg` | `#0f172a` (cold navy) | `#111318` (near-black, neutral) | Less blue, more universal |
| `--color-bg-secondary` | `#1e293b` (blue-gray) | `#1a1d24` (dark gray, warm) | Warmer, less corporate |
| `--color-bg-code` | `#1e293b` | `#1e2028` | Slightly distinct from secondary |
| `--color-border` | `#334155` (blue tint) | `#2a2d35` (neutral gray) | Less blue contamination |
| `--color-sidebar-bg` | `#1e293b` | matches `--color-bg` | Sidebar = part of page |
| `--color-sidebar-active` | `#1e3a5f` | `rgba(249, 115, 22, 0.1)` | Orange tint instead of blue |

#### Light Theme Updates
| Token | Current | Proposed | Rationale |
|---|---|---|---|
| `--color-bg` | `#ffffff` | `#fafafa` | Slightly off-white, easier on eyes |
| `--color-bg-secondary` | `#f8f9fa` | `#f4f4f5` | Subtle distinction |
| `--color-sidebar-bg` | `#f8f9fa` | matches `--color-bg` | Sidebar = part of page |
| `--color-sidebar-active` | `#eff6ff` (blue tint) | `#fff7ed` (orange tint) | Match accent color family |
| `--color-sidebar-active-text` | `#2563eb` (blue) | `#ea580c` (accent) | Active state uses accent |

#### Accent Gradient Bar (Optional Enhancement)
- Thin 2-3px gradient bar across the top of the header
- Orange accent fading to a warm complementary tone
- Low risk, high visual impact — makes the site feel branded
- Can be toggled easily if it doesn't look right

### D6: ENH-1 — Refresh Button on Thread Panel

**Decision:** Add a 🔄 button in the thread header next to the toggle arrow.

Implementation:
- New button in `.thread-header`, between the title and toggle
- On click: calls `fetchAnnotations()` (already exposed as a callback)
- Brief loading spinner or disabled state while fetching
- Subtle, doesn't compete with the title visually

### D7: ENH-2 — Bidirectional Inline Highlighting

**Decision:** Highlight text in the document that has existing comments. Bidirectional navigation between highlighted text and thread comments.

**Interactions (refined with Dan):**
1. **Doc → Thread:** Click highlighted text → thread panel scrolls to the EXACT comment (not just section heading), comment flash-highlights
2. **Thread → Doc:** Click comment's jump button → document scrolls to the EXACT quoted text (not just section heading), text flash-highlights

This tightens the existing "jump to section" feature from E4 to be quote-level precise.

**Implementation:**
- After annotations load, scan document for `quoted_text` matches
- Wrap matched text in `<mark class="annotation-highlight">` with `data-annotation-id`
- Styling: subtle background tint (warm orange at ~10% opacity), slightly more visible on hover
- Click handler on mark: scroll thread panel to corresponding comment, flash-highlight it
- Update existing "📍 jump to section" to scroll to exact quoted text when available (fall back to heading when no quoted_text match)
- Clean up marks on annotation refresh to prevent stale highlights

**Risk:** DOM manipulation on rendered content can be fragile. If quoted text spans HTML elements or was modified since annotation, the match may fail. Fallback: no highlight for that annotation (graceful degradation).

---

## Story Breakdown

### S1: Thread Panel — Slack-Style Timeline + Active/Archived Split
**Batch:** 1 (standalone, no dependencies)

**Scope:**
- Remove bordered card styling from `.thread-comment`
- Implement timeline-style layout (left border, spacing-based separation)
- **Slack-style thread collapse:** parent visible, replies hidden behind "N replies" click-to-expand
- **Active/Archived sections:** unresolved threads in Active, resolved in collapsible Archived section
- Restyle `.thread-review-group` header (no border box, use separator line)
- Adjust `.thread-reply` indentation for flat timeline
- Update hover states to subtle background tint
- Dark theme adjustments

**Acceptance Criteria:**
- [ ] Thread comments display without individual border boxes
- [ ] Timeline left-border connects related comments visually
- [ ] Threads default to collapsed (parent + "N replies" indicator)
- [ ] Click expands thread inline, click again collapses
- [ ] Active section shows unresolved threads
- [ ] Archived section at bottom shows resolved threads (collapsed by default)
- [ ] Resolving a thread moves it from Active to Archived
- [ ] Review groups use header + separator, not bordered container
- [ ] Replies indent correctly with timeline connector
- [ ] Hover state is subtle tint, not box highlight
- [ ] Dark theme looks correct
- [ ] Orphaned section styling still distinct (warning colors)

---

### S2: Thread Panel Toggle Fix + Refresh Button
**Batch:** 1 (parallel with S1, independent CSS + small React change)

**Scope:**
- Fix toggle tab clipping (increase width, adjust grid columns)
- Add 🔄 refresh button to thread header
- Wire button to `fetchAnnotations()` callback
- Loading state while refreshing

**Acceptance Criteria:**
- [ ] Toggle arrow fully visible, no clipping on right edge
- [ ] Refresh button visible in thread header
- [ ] Click triggers annotation re-fetch
- [ ] Brief loading indicator during refresh
- [ ] Button doesn't disrupt header layout

---

### S3: Desktop Layout — Fluid Content Fill
**Batch:** 1 (parallel with S1/S2, pure CSS)

**Scope:**
- Remove fixed `max-width` from content area
- Content fills available space between sidebar and thread panel with `2rem` margins
- Content expands when sidebar or thread panel is hidden
- Content fills viewport (with margins) when both panels hidden
- Test at 1280px, 1440px, 1920px viewport widths

**Acceptance Criteria:**
- [ ] Content fills available space fluidly (no fixed max-width)
- [ ] `2rem` margins on each side of content
- [ ] Hiding sidebar expands content left
- [ ] Hiding thread panel expands content right
- [ ] Hiding both = full-width content with margins
- [ ] No horizontal scrollbar at any viewport width
- [ ] Mobile responsive layout still works
- [ ] Tables and code blocks handle wide content gracefully

---

### S4: Navigation Sidebar Restyle (MkDocs Material Reference)
**Batch:** 2 (depends on S3 for layout, plus color tokens from S5)

**Scope:**
- Remove/soften sidebar border and background (match main bg)
- Replace `▸` text arrows with SVG chevrons
- Smooth rotation animation on expand/collapse
- Active page left-accent indicator (3px orange bar)
- Section title styling (weight/case)
- Sidebar padding adjustments
- Reference: MkDocs Material sidebar patterns

**Acceptance Criteria:**
- [ ] Sidebar feels integrated with page (no hard panel boundary)
- [ ] Background matches main content background
- [ ] Chevrons rotate smoothly on expand/collapse
- [ ] Active page has left accent bar indicator
- [ ] Section titles have clear visual hierarchy
- [ ] All 4 nesting levels render correctly
- [ ] Mobile sidebar overlay still works
- [ ] Dark theme sidebar correct

---

### S5: Color Palette Refresh + Gradient Bar
**Batch:** 2 (parallel with S4, pure CSS variables)

**Scope:**
- Update CSS custom properties per D5 color table
- Add optional accent gradient bar to header top (2-3px)
- Verify all component states render correctly with new colors
- Test light/dark theme toggle
- Verify admonitions, code blocks, tables still look right
- Ensure accent orange is consistent across all interactive elements
- **Iterative:** ship first pass, expect follow-up adjustments

**Acceptance Criteria:**
- [ ] Dark theme uses warmer neutrals (not blue-gray)
- [ ] Light theme uses off-white background
- [ ] Sidebar colors match new palette
- [ ] Active states use accent color family (not blue)
- [ ] Gradient bar renders at top of header (orange accent)
- [ ] Code blocks, tables, admonitions render correctly in both themes
- [ ] No contrast/accessibility regressions (text remains readable)
- [ ] Dan approves the look on live site

---

### S6: Bidirectional Inline Text Highlighting (ENH-2)
**Batch:** 3 (depends on S1 thread restyle for visual consistency)

**Scope:**
- After annotations load, find `quoted_text` matches in document DOM
- Wrap matched text in `<mark>` elements with annotation data attributes
- Subtle highlight styling (warm accent at low opacity)
- **Doc → Thread:** click highlight scrolls to EXACT comment in thread panel, flash-highlights it
- **Thread → Doc:** update existing jump button to scroll to EXACT quoted text (not just heading), flash-highlights it
- Cleanup on annotation refresh
- Handle edge cases (no match, partial match, cross-element spans)

**Acceptance Criteria:**
- [ ] Quoted text from annotations highlighted in document
- [ ] Highlight styling is subtle, not distracting
- [ ] Click on highlight scrolls thread to exact corresponding comment
- [ ] Comment flash-highlights when scrolled to from doc
- [ ] Click jump button in thread scrolls to exact quoted text in doc
- [ ] Quoted text flash-highlights when scrolled to from thread
- [ ] Falls back to heading-level scroll when quoted text can't be matched
- [ ] Highlights cleaned up on annotation refresh
- [ ] No match = graceful degradation (no error, no highlight)
- [ ] Works in both light and dark themes

---

## Execution Plan

| Batch | Stories | Strategy |
|---|---|---|
| 1 | S1 + S2 + S3 | Parallel — all independent CSS/component work. Use git worktrees. |
| 2 | S4 + S5 | Parallel — sidebar restyle and color palette are independent. |
| 3 | S6 | Sequential — needs S1 merged for visual consistency. |

**Estimated effort:** 6 stories, 3 batches. S1 is the heaviest (thread collapse + Active/Archived split). Rest are straightforward CSS + small component work.

---

## Deferred to E8: Live Content Pipeline

The following were originally considered for E7 but belong in a dedicated epic:
- **Auto-nav generation** from directory structure (no manual nav.yaml)
- **Cross-repo rebuild triggers** (webhook or repository_dispatch)
- **Astro SSR migration** (serve pages dynamically, not static build)
- **Runtime content fetch** (git pull on webhook, not Docker build time)
- **Live Anvil reindexing** triggered after content updates
- **Webhook endpoint** in Foundry API (`POST /api/webhooks/content-update`)

E8 is the right place for this — it's a real architecture shift, not a UI polish.

---

## Dependencies

- None. All E1–E6 epics are complete.
- BUG-5 and BUG-6 are independent of E7 (can be fixed before, during, or after).
- Issues #22 and #23 are merged (PRs #30, #31).
