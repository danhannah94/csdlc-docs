# E10: UI/UX Polish Part 2 — Design Doc

*Status: Step 0 Complete — Ready for Execution*
*Created: April 6, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

E10 is the second UI/UX polish epic. Where E7 focused on foundational layout, thread panel, sidebar, and color — E10 adds the features that make Foundry feel like a finished product. Search, settings, table of contents, sticky headings, code copy buttons, branding, loading states. The things where absence is noticeable.

### Why Now

E8 (live content pipeline) and E9 (MCP agent surface) are shipped. The platform is functionally complete for v0.2. But when you show it to someone — a CIO, a potential user, Jack — the first thing they notice isn't the MCP tools or the SSR pipeline. They notice whether it *feels* polished. E10 closes that gap.

### Scope

Mostly frontend — React islands, CSS, and Astro layout changes. One backend touchpoint (search endpoint already exists via Anvil, just needs a UI). No schema changes, no new API endpoints, no infrastructure work.

### What's NOT in E10

- Authentication / multi-user → E11
- Collaborative features → E12
- Real-time push / WebSocket → future
- Auto-nav generation improvements → already shipped in E8

---

## Feature Inventory

### From Dan's List
1. **Settings menu** — consolidate auth indicator (lock) + theme toggle into a single gear dropdown
2. **Search bar** — Anvil-backed search in the header with `Cmd+K` shortcut
3. **Sticky section headings** — headings stick to top while scrolling through their content
4. **Table of contents** — right-side TOC with scroll spy when thread panel is hidden
5. **Foundry SVG logo** — real brand mark replacing the 🏭 emoji

### From Clay's Additions
6. **Code block copy button** — one-click copy on fenced code blocks
7. **Breadcrumb enhancement** — verify breadcrumbs are prominent and functional (component exists, may need visibility work)
8. **Back-to-top button** — floating button after scrolling down
9. **Favicon + meta tags** — proper `og:title`, `og:description`, `og:image` from the new logo
10. **Mobile polish pass** — touch targets, hamburger behavior, swipe gestures, thread panel on mobile
11. **Loading skeletons** — shimmer placeholders while SSR renders and annotations load
12. **Annotation count badges** — small badges on section headings showing comment count
13. **Nav tree ordering fix** — numeric-aware sorting so E10 appears after E9, not after E1
14. **TTS enable/disable toggle** — in settings modal, persisted to localStorage
15. **Comment modal fixed sizing** — consistent width, not dynamic based on selection length
16. **Draft edit/delete in thread panel** — edit/delete buttons directly on draft cards (not bottom-of-doc draft list)

### Deferred (Not E10)
- Keyboard navigation (`j`/`k` for sections, `?` for shortcuts overlay) — cool but low ROI right now
- Print / export styles (`@media print`) — nice-to-have, not demo-critical

### Nav Ordering Decision

**Problem:** Auto-nav sorts alphabetically, so `E10` appears after `E1` instead of after `E9`. This is lexicographic string sorting — `"e10" < "e2"` because character `'1' < '2'`.

**Decision:** Implement natural (numeric-aware) sorting in the nav generator. The `sortEntries()` function in `nav.ts` gets a comparator that treats embedded numbers as numeric values. This is a small change to `packages/site/src/utils/nav.ts` — no frontmatter `order` fields needed for docs that follow a numbered naming convention.

```typescript
// Natural sort: "E1" < "E2" < "E10" (not "E1" < "E10" < "E2")
function naturalCompare(a: string, b: string): number {
  return a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' });
}
```

`String.localeCompare` with `{ numeric: true }` handles this natively — no regex or custom parsing needed. Added to S8 scope.

---

## Design Decisions

### D1: Settings Menu — Gear Dropdown

**Current state:** Header right section has `AuthIndicator` (lock icon) and `ThemeToggle` (sun/moon) as separate buttons. As we add more settings, this becomes a row of mystery icons.

**Decision:** Replace both with a single gear icon (SVG) that opens a centered settings modal.

**Modal sections (v1):**
- **Appearance** — Light / Dark / System (radio group, replaces ThemeToggle)
- **Audio** — TTS enabled/disabled (toggle switch, default ON). When disabled, hides all play buttons and TTS controls bar via `body.tts-disabled` CSS class.
- **Authentication** — Shows current auth state (locked/unlocked icon). Click to open TokenModal.
- *Future slots:* Font size, thread panel default state

**Why modal over dropdown:** A dropdown gets cramped as settings grow. A centered modal gives room for grouped sections and scales to future preferences without layout gymnastics.

**Implementation:**
- New React island: `SettingsModal.tsx`
- Click gear icon → modal appears centered with backdrop overlay
- Click outside, press `Escape`, or click X to close
- Theme selection updates `data-theme` and `localStorage` (same logic as current ThemeToggle)
- TTS toggle writes to `localStorage('foundry-tts')`, body gets class `tts-disabled` when off
- Auth row shows status and opens existing `TokenModal` on click
- Remove standalone `ThemeToggle` and `AuthIndicator` from header
- Modal styled to match existing modal patterns (same as comment editor modal)

**Header layout after E10:**
```
[☰ sidebar toggle] [Foundry SVG logo] ————————— [🔍 search SVG] [⚙ gear SVG]
```

**All header icons are inline SVGs** (no emojis). Uses `currentColor` for automatic theme adaptation. The sidebar toggle already uses an SVG hamburger — search (magnifying glass) and settings (gear) follow the same pattern.

### D2: Search Bar — Cmd+K Modal with Anvil Backend

**Current state:** Anvil semantic search works via MCP tools and API (`GET /api/search?q=...`). No UI search exists in Foundry.

**Decision:** Add a search bar in the header + `Cmd+K` / `Ctrl+K` keyboard shortcut to open a search modal.

**Search UX:**
- **Header search input:** Compact text field in the header between the logo and settings. Clicking it (or pressing `Cmd+K`) opens the full search modal.
- **Search modal:** Centered overlay (like Algolia DocSearch / MkDocs Material search). Auto-focused text input at top, results stream below as you type.
- **Results:** Each result shows page title, section heading, snippet with highlighted match text. Click navigates to that page + scrolls to section.
- **Keyboard navigation:** Arrow keys to move through results, `Enter` to navigate, `Escape` to close.
- **Debounce:** 300ms debounce on typing before firing API request.
- **Empty state:** "Search across all docs..." placeholder. No results = "No results found for 'query'"

**API:** `GET /api/search?q=<query>` already exists and returns Anvil results with `doc_path`, `section`, `content`, and relevance score.

**Implementation:**
- New React island: `SearchModal.tsx`
- Header gets a search trigger (compact input or search icon that opens the modal)
- Global `keydown` listener for `Cmd+K` / `Ctrl+K`
- Modal overlay with backdrop blur
- `fetch` to existing search endpoint with debounce
- Results rendered as clickable cards
- Navigate on click: `window.location.href = /docs/${doc_path}#${section_slug}`

### D3: Sticky Section Headings

**Current state:** Headings scroll off screen. On long docs, you lose context about which section you're reading.

**Decision:** `h2` headings stick to the top of the viewport when their section is in view.

**Known gotcha (from E7 tech debt):** `overflow: hidden` on `.content-area` breaks `position: sticky`. The current CSS has `overflow-x: hidden` on both `.content-area` and `.content`. This MUST be resolved for sticky to work.

**Implementation approach:**
- Audit and fix the overflow chain: use `overflow-x: clip` (modern CSS, doesn't create a new scroll container like `overflow-x: hidden`) instead of `overflow-x: hidden` on `.content-area` and `.content`. This contains horizontal overflow (wide tables/code blocks) without breaking sticky positioning.
- `h2` elements get `position: sticky; top: var(--header-height); z-index: 5;`
- Background color matches page bg (so content doesn't show through behind the sticky heading)
- Subtle bottom shadow when stuck to indicate stacking
- Bottom border on `h2` already exists — combine with sticky for clean visual
- Only `h2` sticky (not h3/h4 — too many levels stacking gets messy)

**Fallback:** If `overflow-x: clip` has browser compat issues (Safari < 16), fall back to JavaScript-based sticky detection with `IntersectionObserver` and manual `position: fixed` toggling.

### D4: Table of Contents — Right Sidebar with Scroll Spy

**Current state:** No table of contents. Thread panel occupies the right column. When thread panel is hidden, that space is empty (content expands to fill it, per E7-S3).

**Decision:** Show a TOC in the right column when the thread panel is hidden. TOC disappears when thread panel opens.

**TOC behavior:**
- Extracts all `h2` and `h3` headings from the current page
- Renders as a slim, indented list in the right gutter
- **Scroll spy:** Currently-visible section heading is highlighted (accent color left border, like sidebar active state)
- Click a TOC entry → smooth scroll to that heading
- TOC is sticky (sticks below header, scrolls internally if longer than viewport)
- Hides when thread panel is shown (they share the same visual space)
- Hides on mobile (not enough room)
- Active section determined by `IntersectionObserver` on headings

**Implementation:**
- New React island: `TableOfContents.tsx`
- Receives page headings as a prop (extracted from rendered HTML or passed from `[...slug].astro`)
- Positioned in a new grid column or absolutely positioned in the right margin
- When thread panel is visible: `display: none` on TOC
- When thread panel is hidden: TOC appears
- CSS: `position: sticky; top: calc(var(--header-height) + 1rem); max-height: calc(100vh - var(--header-height) - 2rem); overflow-y: auto;`

**Grid layout change:**
- Currently: `grid-template-columns: var(--sidebar-width) 1fr var(--thread-panel-width)`
- With TOC: same grid. TOC renders inside the content-area as a floated/positioned element in the right margin, OR we add a 4th column. Simpler approach: TOC is absolutely positioned relative to content-area, pinned to the right edge with a fixed width (~200px). This avoids grid changes.

### D5: Foundry SVG Logo

**Decision:** Create a real SVG logo for Foundry. The 🏭 emoji is a placeholder.

**Design direction:** Modern, clean, recognizable at small sizes (16px favicon). Should work in both light and dark themes. Tie into the "foundry / metalwork / forge" theme but keep it abstract enough to feel like a tech product, not a factory illustration.

**Concepts to explore:**
- Stylized anvil shape (ties to our Anvil product)
- Flame / spark motif (foundry = heat + creation)
- Geometric lettermark ("F" with forge/flame element)
- Abstract molten metal pour shape

**Deliverables:**
- `logo.svg` — full logo for header (replace `🏭 Foundry` text)
- `favicon.svg` — simplified mark for favicon (current `favicon.svg` is likely a placeholder or missing)
- Both work on light and dark backgrounds (CSS `fill: currentColor` or theme-aware)
- `og-image.png` — 1200×630 for social/link previews

### D6: Code Block Copy Button

**Current state:** Code blocks render with syntax highlighting (Shiki) but no copy mechanism. Users have to manually select text.

**Decision:** Add a copy button to every fenced code block.

**Implementation:**
- Use a remark/rehype plugin or post-render DOM manipulation
- Each `<pre><code>` block gets a copy button in the top-right corner
- Button: clipboard icon (📋), shows "Copied ✓" for 2 seconds after click
- Positioned `absolute` within the `pre` (which gets `position: relative`)
- Semi-transparent on idle, fully visible on hover over the code block
- Uses `navigator.clipboard.writeText()` — falls back to `execCommand('copy')` for older browsers
- Works in both light and dark themes

**Implementation choice:** Post-render DOM script (like mermaid loading) rather than a remark plugin. Simpler, works with SSR-rendered HTML without touching the markdown pipeline.

### D7: Back-to-Top Button

**Decision:** Floating button in bottom-right corner that appears after scrolling past a threshold.

**Implementation:**
- Shows after scrolling down > 500px
- Smooth scrolls to top on click
- Positioned bottom-right, above TTS controls bar (when visible)
- Subtle: small circle with ↑ arrow, semi-transparent, fully visible on hover
- Animates in/out with CSS transitions
- `position: fixed; bottom: 2rem; right: 2rem;` (adjusted when TTS bar or thread panel is visible)
- Script-only, no React island needed (simple DOM manipulation)

### D8: Favicon + Meta Tags

**Decision:** Proper meta tags and favicon from the new logo.

**Implementation:**
- `favicon.svg` in `public/` (already linked in DocLayout.astro `<head>`)
- Add `<meta property="og:title">`, `<meta property="og:description">`, `<meta property="og:image">` to DocLayout
- `og:title` = page title (already in `<title>`)
- `og:description` = page description or first paragraph
- `og:image` = `/og-image.png` (static asset, generated from logo)
- Twitter card meta tags (`twitter:card`, `twitter:title`, etc.)
- Apple touch icon variant

### D9: Mobile Polish Pass

**Current state:** Basic responsive layout exists — sidebar becomes overlay at 1024px, thread panel becomes slide-in. But hasn't been tested rigorously on actual phones/tablets.

**Decision:** Dedicated pass to fix mobile UX issues.

**Known items to check:**
- Touch targets: all buttons ≥ 44px × 44px (Apple HIG minimum)
- Sidebar overlay: does it have a backdrop? Can you tap outside to close?
- Thread panel: width on small screens, close gesture
- Header: does it feel right on iPhone? Is the search bar usable?
- Comment float button: positioned correctly on mobile? Not clipped by keyboard?
- Comment modal: fixed consistent size (not dynamic based on selection length) — ~480px desktop, 95vw mobile
- TTS controls bar: full-width, nothing cut off?
- Search modal: full-screen on mobile (not the desktop centered overlay)
- Content: text size, line height, spacing feel right?
- Breadcrumbs: wrap correctly on narrow screens?

**Implementation:** Audit on iPhone/iPad, fix CSS issues found. May need media query adjustments and touch-specific event handlers.

### D10: Loading Skeletons

**Current state:** Thread panel shows "Loading annotations..." text. Page content has no loading state (SSR renders server-side, so the HTML arrives complete, but there can be a brief delay on first load or after cache invalidation).

**Decision:** Add shimmer skeleton placeholders for:
1. **Thread panel** — skeleton comment shapes while annotations load
2. **Page content** — skeleton heading + paragraph blocks during SSR render delay (only visible if content hasn't arrived yet — rare but possible on cold starts)
3. **Search results** — skeleton result cards while search query is in flight

**Implementation:**
- CSS shimmer animation (reusable class: `.skeleton`)
- Skeleton shapes match the real content layout (rectangles with rounded corners at the right dimensions)
- Gray placeholder color that fits both light and dark themes
- Thread panel: 3-4 skeleton comment blocks (varying widths to look organic)
- Search: 3 skeleton result cards
- Page content: heading block + 3 paragraph lines (only shown if content div is empty)

```css
.skeleton {
  background: linear-gradient(90deg,
    var(--color-bg-secondary) 25%,
    var(--color-bg-code) 50%,
    var(--color-bg-secondary) 75%
  );
  background-size: 200% 100%;
  animation: skeleton-shimmer 1.5s ease-in-out infinite;
  border-radius: 4px;
}

@keyframes skeleton-shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

### D11: Annotation Count Badges

**Decision:** Show small badges on section headings indicating how many comments exist in that section.

**Implementation:**
- After annotations load, count comments per section heading path
- Inject badge next to the heading: `<span class="annotation-badge">3</span>`
- Subtle styling: small pill, accent color background, white text, positioned after heading text
- Click badge → opens thread panel and scrolls to that section's comments
- Badges update when annotations are refreshed
- Only show for sections with > 0 comments
- Styling: similar to notification badges (rounded pill, small font, ~18px height)

---

## E7 Leftovers (Folded into E10)

These items from the E7 backlog are being addressed in E10:

| Item | What | E10 Story |
|------|------|-----------|
| Remove draft list from bottom of doc | The `CommentDraft` component renders a draft list below the article. Drafts should only appear in the thread panel. | S8 |
| Comment modal width | Modal is `min-width: 400px` which can be tight. Should be responsive. | S9 (mobile pass) |
| Draft highlighting | Highlight quoted text in the doc for draft comments (not just submitted annotations) | S8 |
| Reply button at bottom of thread | Quick-reply button at the bottom of the thread panel for fast follow-ups | S8 |

---

## Story Breakdown

### S1: Settings Modal + TTS Toggle
**Batch:** 1

**Scope:**
- New React island: `SettingsModal.tsx`
- SVG gear icon in header-right, opens centered modal on click
- **Appearance section:** Theme selection (Light / Dark / System) — replaces `ThemeToggle`
- **Audio section:** TTS enabled/disabled toggle (default ON). Writes to `localStorage('foundry-tts')`. Adds `tts-disabled` class to body. CSS hides all `.tts-play-btn`, `.tts-play-all`, `.tts-controls-bar` when disabled.
- **Authentication section:** Shows auth state, click opens `TokenModal`
- Backdrop overlay, click-outside and Escape to close
- Remove standalone `ThemeToggle` and `AuthIndicator` from DocLayout header
- All header icons are inline SVGs (no emojis)

**Acceptance Criteria:**
- [ ] SVG gear icon visible in header
- [ ] Modal opens centered with backdrop overlay
- [ ] Theme switching works (Light / Dark / System) with localStorage persistence
- [ ] TTS toggle enables/disables all TTS UI elements
- [ ] TTS preference persists across page loads (localStorage)
- [ ] Auth row shows current state and opens TokenModal on click
- [ ] Modal closes on click-outside, Escape, and X button
- [ ] `ThemeToggle` and `AuthIndicator` removed from header
- [ ] Keyboard accessible (Tab, Enter, Escape)
- [ ] Dark theme modal styled correctly

---

### S2: Search Bar + Cmd+K Modal
**Batch:** 1 (parallel with S1)

**Scope:**
- New React island: `SearchModal.tsx`
- Compact search input in header (between logo and settings)
- `Cmd+K` / `Ctrl+K` global keyboard shortcut opens modal
- Modal: auto-focused input, debounced API calls (300ms), streaming results
- Results show page title, section heading, snippet with highlighted match text
- Arrow key navigation, Enter to select, Escape to close
- Navigate to `doc_path#section` on result click
- Empty and no-results states
- Backend: existing `GET /api/search?q=...` endpoint (no backend work)

**Acceptance Criteria:**
- [ ] Search input visible in header
- [ ] `Cmd+K` / `Ctrl+K` opens search modal
- [ ] Typing triggers debounced search against Anvil API
- [ ] Results display with page title, section, highlighted snippet
- [ ] Arrow keys navigate results, Enter selects
- [ ] Click on result navigates to correct page + section
- [ ] Escape closes modal
- [ ] Backdrop click closes modal
- [ ] Empty state and no-results state render correctly
- [ ] Works in both light and dark themes
- [ ] Mobile: modal goes full-screen width

---

### S3: Sticky Section Headings
**Batch:** 2 (depends on overflow fix)

**Scope:**
- Fix `overflow-x: hidden` → `overflow-x: clip` on `.content-area` and `.content` (the prerequisite for sticky to work)
- `h2` elements: `position: sticky; top: var(--header-height)`
- Background color on sticky h2 matches page bg (prevent content showing through)
- Subtle box-shadow when heading is stuck (visual stacking indicator)
- `z-index` layering that doesn't conflict with header, thread panel, or sidebar
- Test with long docs that have many h2 sections

**Acceptance Criteria:**
- [ ] `h2` headings stick below header when scrolling through their section
- [ ] Background color prevents content bleed-through
- [ ] Shadow appears when heading is in "stuck" state
- [ ] Wide tables and code blocks still contained (overflow-x: clip works)
- [ ] No z-index conflicts with header, sidebar, or thread panel
- [ ] Mobile: sticky still works or is disabled (test both)
- [ ] Content that has no h2s (rare) is unaffected

**Risk:** `overflow-x: clip` browser support. Chrome 90+, Firefox 81+, Safari 16+. If Safari < 16 is a concern, need JS fallback. For our use case (Dan on iPad — iPadOS 16+ is mainstream), this should be fine.

---

### S4: Table of Contents with Scroll Spy
**Batch:** 2 (parallel with S3)

**Scope:**
- New React island: `TableOfContents.tsx`
- Extracts h2 and h3 headings from page
- Renders slim indented list in right margin
- Scroll spy: highlights current section heading (IntersectionObserver)
- Click → smooth scroll to heading
- Sticky positioning (below header, scrolls internally if long)
- Hidden when thread panel is visible
- Hidden on mobile (< 1024px)
- Positioned as a fixed-width element in the right margin of `.content-area`, not a grid column change

**Acceptance Criteria:**
- [ ] TOC renders when thread panel is hidden
- [ ] TOC disappears when thread panel is shown
- [ ] Active section highlighted with accent color
- [ ] Click scrolls to correct heading smoothly
- [ ] TOC is sticky and scrolls internally when it overflows viewport
- [ ] Handles pages with many headings (20+)
- [ ] Handles pages with few/no headings gracefully (hides TOC if < 2 headings)
- [ ] Hidden on mobile
- [ ] Works in both light and dark themes

---

### S5: Foundry Logo + Branding
**Batch:** 1 (parallel, no code dependencies)

**Scope:**
- Design SVG logo for Foundry
- Create `logo.svg` (full header logo), `favicon.svg` (simplified mark)
- Generate `og-image.png` (1200×630 for social previews)
- Replace `🏭 Foundry` text in header with SVG logo + wordmark
- Update `favicon.svg` in `public/`
- Add Open Graph meta tags to DocLayout: `og:title`, `og:description`, `og:image`
- Add Twitter card meta tags
- Add `apple-touch-icon`

**Acceptance Criteria:**
- [ ] SVG logo renders in header (light + dark themes)
- [ ] Favicon shows in browser tab
- [ ] Sharing a Foundry link on Slack/Discord/Telegram shows og:image preview
- [ ] Logo recognizable at 16px, 32px, and header sizes
- [ ] Dan approves the design

---

### S6: Code Block Copy Button
**Batch:** 2 (parallel with S3/S4)

**Scope:**
- Post-render script that adds copy buttons to all `<pre><code>` blocks
- Button: clipboard icon in top-right corner of code block
- Click → copies code text to clipboard, shows "Copied ✓" for 2 seconds
- `pre` gets `position: relative` for absolute positioning of button
- Semi-transparent idle state, full opacity on hover
- Works with Shiki-highlighted code blocks
- Dark/light theme compatible

**Acceptance Criteria:**
- [ ] Copy button visible on every fenced code block
- [ ] Hover shows button more prominently
- [ ] Click copies code content to clipboard
- [ ] "Copied ✓" feedback shown for 2 seconds
- [ ] Works on code blocks with syntax highlighting (Shiki)
- [ ] Works in both themes
- [ ] Button doesn't interfere with code scrolling
- [ ] Mobile: button always visible (no hover state)

---

### S7: Back-to-Top Button
**Batch:** 2 (parallel, pure script/CSS)

**Scope:**
- Floating button: bottom-right corner, appears after 500px scroll
- Click → smooth scroll to top
- Positioned above TTS controls bar when visible
- Subtle styling: small circle, ↑ arrow, semi-transparent
- CSS transitions for show/hide
- No React island needed — inline script in DocLayout

**Acceptance Criteria:**
- [ ] Button appears after scrolling 500px
- [ ] Button disappears when near top
- [ ] Click smoothly scrolls to top
- [ ] Positioned correctly above TTS bar (when visible)
- [ ] Doesn't overlap with thread panel toggle or other fixed elements
- [ ] Works in both themes
- [ ] Accessible: `aria-label="Scroll to top"`

---

### S8: Thread Panel Draft Cleanup + Annotation Badges + Nav Ordering
**Batch:** 3 (depends on S1-S4 for layout stability)

**Scope:**
- **Remove draft list from bottom of doc** — the `CommentDraft` component currently renders drafts below the article content. Remove this, drafts live only in thread panel.
- **Edit/delete buttons on draft cards** — each draft in the thread panel gets edit (pencil) and delete (trash) icon buttons. Edit opens the draft for modification, delete removes it from localStorage.
- **Comment modal fixed sizing** — set consistent width (~480px desktop, 95vw mobile) with scrollable quoted text region instead of dynamic sizing based on selection length.
- **Draft highlighting** — extend existing `mark.annotation-highlight` pattern to also highlight quoted text for draft comments (not just submitted annotations). Use a distinct style (dashed underline or dimmed highlight to differentiate from submitted).
- **Reply button at thread bottom** — quick "Add a comment" input at the bottom of the thread panel for fast replies without highlighting text first.
- **Annotation count badges** — after annotations load, count per-section and inject badge pills next to h2/h3 headings. Click opens thread panel to that section.
- **Natural nav ordering** — update `sortEntries()` in `nav.ts` to use `localeCompare` with `{ numeric: true }` so E10 sorts after E9.

**Acceptance Criteria:**
- [ ] No draft list rendered below article content
- [ ] Draft cards in thread panel have edit and delete buttons
- [ ] Edit button opens draft for content modification
- [ ] Delete button removes draft from localStorage and thread panel
- [ ] Comment modal has consistent fixed width (not dynamic)
- [ ] Draft comments show highlighted quoted text in doc (distinct from submitted annotation highlights)
- [ ] Quick reply input at bottom of thread panel
- [ ] Reply creates annotation linked to current doc (section = general or last viewed section)
- [ ] Annotation badges show count next to headings with comments
- [ ] Badge click opens thread panel and scrolls to section's comments
- [ ] Badges update on annotation refresh
- [ ] Nav sidebar sorts E10 after E9 (numeric-aware sorting)
- [ ] All features work in light and dark themes

---

### S9: Mobile Polish + Loading Skeletons
**Batch:** 3 (final polish pass)

**Scope:**
- **Mobile audit and fixes:**
  - Touch targets ≥ 44px on all interactive elements
  - Sidebar overlay: backdrop for tap-to-close
  - Thread panel on mobile: reasonable width, close button visible
  - Comment modal: responsive width (`max-width: 95vw`, not `min-width: 400px` on mobile)
  - Search modal: full-screen on mobile
  - Header: all elements accessible on small screens
  - Test on iPhone-sized viewport (375px) and iPad (768px-1024px)

- **Loading skeletons:**
  - Reusable `.skeleton` CSS class with shimmer animation
  - Thread panel: skeleton comment blocks while annotations load
  - Search modal: skeleton result cards while query is in flight
  - Content area: skeleton heading + paragraph lines (only visible on slow SSR, rare)
  - Skeleton shapes match real content dimensions

**Acceptance Criteria:**
- [ ] All interactive elements ≥ 44px touch target on mobile
- [ ] Sidebar overlay has backdrop, tap-outside closes
- [ ] Thread panel usable on mobile (proper width, close button)
- [ ] Comment modal responsive on mobile
- [ ] Search modal full-screen on mobile
- [ ] Skeleton shimmer animation on thread panel loading
- [ ] Skeleton animation on search results loading
- [ ] Skeletons match real content layout shapes
- [ ] No horizontal scrollbar on any mobile viewport
- [ ] Tested at 375px and 768px viewport widths

---

## Execution Plan

| Batch | Stories | Strategy | Dependencies |
|-------|---------|----------|--------------|
| 1 | S1 (settings) + S2 (search) + S5 (logo) | Parallel — independent components. Logo is design + static assets, no code overlap with S1/S2. | None |
| 2 | S3 (sticky headings) + S4 (TOC) + S6 (code copy) + S7 (back-to-top) | Parallel — S3/S4 share the content area but touch different concerns. S6/S7 are independent scripts. | S1/S2 merged (header layout stable) |
| 3 | S8 (drafts + badges) + S9 (mobile + skeletons) | Parallel — S8 is thread panel + annotation changes, S9 is responsive CSS + loading states. | S1-S7 merged (all layout changes finalized) |

**Estimated effort:** 9 stories, 3 batches. Batch 1 is the heaviest (search modal is the most complex single component). Batch 2 is medium. Batch 3 is polish and cleanup.

---

## Dependencies

- E7 shipped ✅ (thread panel, layout, color palette)
- E8 shipped ✅ (SSR, Anvil search API exists)
- E9 shipped ✅ (MCP tools, skill rewrite)
- No blocking dependencies. E10 is purely additive.

---

## Open Questions

| # | Question | Status |
|---|----------|--------|
| 1 | Logo direction — abstract/geometric vs literal foundry/forge imagery? | **Open** — need Dan's input on vibe before generating |
| 2 | `overflow-x: clip` Safari support — test on Dan's iPad? | **Open** — will verify during S3 implementation |
| 3 | TOC positioning — absolute within content area vs 4th grid column? | **Decided** — absolute positioning, simpler, no grid changes |
| 4 | Search results ranking — Anvil semantic search vs simple text match? | **Decided** — use existing Anvil API (semantic search already works) |
| 5 | Loading skeletons scope — worth it for page content or just thread/search? | **Decided** — thread + search primarily, page content skeleton as nice-to-have |
| 6 | Settings: dropdown vs modal? | **Decided** — centered modal. Scales better for grouped settings sections. |
| 7 | Header icons: emojis vs SVGs? | **Decided** — all inline SVGs with `currentColor`. No emojis in UI chrome. |
| 8 | Nav ordering: E10 after E1? | **Decided** — natural sort via `localeCompare({ numeric: true })`. Added to S8. |
| 9 | TTS toggle in E10? | **Decided** — yes, enable/disable in settings modal. Tiny lift, added to S1. |

---

## Related

- [E7: UI/UX Refinement (Part 1)](e7-ui-refinement.md) — foundation for E10
- [E8: Live Content Pipeline](e8-live-content-pipeline.md) — SSR + search backend
- [E9: MCP Agent Surface](e9-mcp-agent-surface.md) — agent-side polish
- [Foundry Design Doc](../design.md) — overall architecture

