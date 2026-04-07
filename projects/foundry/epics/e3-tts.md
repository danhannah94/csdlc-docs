# E3: TTS Playback (Native Browser Speech)

*Status: Step 0 — Design Doc (Refinement Complete)*
*Epic: Foundry v0.1 (no API dependency)*
*Created: March 30, 2026*
*Updated: March 30, 2026*
*Authors: Dan Hannah & Clay*

---

## Overview

### What Is This Epic?

Add text-to-speech audio playback to Foundry using the browser's native Web Speech API. Play buttons appear next to each section heading — click to hear that section read aloud. A "Play All" button at the doc title reads the entire document. No server, no API, no cost.

### Problem Statement

Dan reviews docs while driving, in the shop, or doing CNC work. Text-only docs require a screen. TTS makes docs accessible in hands-free contexts. This should work from any device (iPad, phone, laptop) without requiring the API server to be running.

### What Changed From the Original Stub

| Original Scope | Revised Scope |
|----------------|--------------|
| Server-side TTS (Edge TTS / ElevenLabs) | Native browser Web Speech API |
| API endpoint for audio generation | Pure frontend — no server needed |
| Audio file caching by content hash | No files — browser generates speech on the fly |
| Required E2 API server | Independent — can ship before or after E2 |
| Controlled voice selection | Device-native voices (iOS Siri voices, macOS, Chrome) |

**Key insight:** The Web Speech API (`speechSynthesis`) is built into every modern browser. iOS voices are near-human quality. This eliminates the entire server-side audio pipeline — no generation, no storage, no caching, no cost. E3 becomes a pure static site feature.

### Goals

- Play button (▶️) next to each section heading — reads that section
- "Play All" button at doc title — reads the full document section by section
- Play/pause/speed controls (0.75x / 1x / 1.25x / 1.5x / 2x)
- Visual indicator showing which section is currently being read
- Auto-scroll to follow the currently-reading section (toggleable)
- Works offline, works on static site, works everywhere

### Non-Goals

- Custom voice selection UI (use device default — it's good enough)
- Server-side TTS generation (entire point is no server)
- Audio file download/export
- Background audio playback (browser limitation)
- Remembering playback position across page loads (v0.3+ if wanted)
- Reading non-text content (Mermaid diagrams, code blocks — skip or read code optionally)

---

## Context

### Dependencies

- **E1** — site scaffold, doc rendering, heading structure
- **No E2 dependency** — this is pure frontend

### Dependents

- None directly. TTS is a standalone feature that enhances the reading experience.
- Future: could inform E4 UX (e.g., "read this annotation aloud")

---

## Design

### Approach

A React island component (`TtsPlayer.tsx`) that uses the Web Speech API to read document sections aloud. The component receives section content as props (extracted from the rendered page) and manages playback state.

### Architecture

```
┌─────────────────────────────────────────────┐
│  DocLayout.astro                            │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │  TtsControls.tsx (React island)      │   │
│  │  - Play All / Pause / Speed          │   │
│  │  - Current section indicator         │   │
│  │  - Auto-scroll toggle                │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ## Section Heading        [▶️ play button]  │
│  Section content here...                    │
│                                             │
│  ## Another Section        [▶️ play button]  │
│  More content here...                       │
│                                             │
│          ↕ Web Speech API                   │
│  ┌──────────────────────────────────────┐   │
│  │  Browser speechSynthesis engine      │   │
│  │  (native, zero cost, offline-capable)│   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Web Speech API — What We're Working With

```typescript
// Core API
const utterance = new SpeechSynthesisUtterance(text);
utterance.rate = 1.25;  // 0.1 to 10
utterance.voice = speechSynthesis.getVoices()[0]; // device default
speechSynthesis.speak(utterance);
speechSynthesis.pause();
speechSynthesis.resume();
speechSynthesis.cancel();

// Events
utterance.onstart = () => { /* highlight section */ };
utterance.onend = () => { /* advance to next section */ };
utterance.onboundary = (e) => { /* word-level tracking */ };
```

**Browser support:**
- Safari (iOS/macOS): ✅ Excellent voices (Siri neural), reliable API
- Chrome: ✅ Good voices, sometimes cuts out on very long utterances
- Firefox: ✅ Basic support, fewer voice options
- Edge: ✅ Good voices (same as Edge TTS)

### Section Text Extraction

Sections are defined by heading boundaries. The component needs to:
1. Find all headings (h2, h3, h4) in the rendered doc
2. Extract text content between each heading and the next heading of same or higher level
3. Strip HTML tags, keep readable text
4. Skip or optionally include code blocks (configurable)

This extraction happens client-side from the rendered DOM — no build-time processing needed.

### Playback Modes

**Section Play (▶️ next to heading):**
1. User clicks play button next to "## Refinement"
2. Extract text content for that section
3. Create `SpeechSynthesisUtterance` with section text
4. Highlight the section being read (subtle background color)
5. Button becomes pause (⏸️) during playback
6. On end: button returns to play, highlight removed

**Play All (▶️ at doc title):**
1. User clicks "Play All" at the top
2. Build ordered list of all sections
3. Play section 1 → on end → play section 2 → ... → done
4. Current section highlighted, auto-scroll follows (if enabled)
5. Global pause/resume affects current section
6. Skip forward/back between sections

**Speed Control:**
- Dropdown or segmented control: 0.75x / 1x / 1.25x / 1.5x / 2x
- Stored in localStorage (persists across sessions)
- Applied to current and future utterances

### Chrome Long-Text Workaround

Chrome has a known bug where `speechSynthesis` stops after ~15 seconds of continuous speech. The standard workaround:

```typescript
// Split long text into sentences/chunks (~200 chars each)
// Speak sequentially with onend chaining
// This is invisible to the user — just an implementation detail
```

This is well-documented and every TTS library handles it. Our implementation will chunk text at sentence boundaries.

### UI Components

**TtsControls (floating or sticky):**
- Appears when any playback is active
- Play/Pause button
- Speed selector
- Current section name
- Skip prev / skip next (in Play All mode)
- Auto-scroll toggle
- Close/stop button

**Section Play Button:**
- Small ▶️ icon to the right of each heading
- Appears on hover (desktop) or always visible (mobile/tablet)
- Becomes ⏸️ during playback of that section
- Subtle, doesn't clutter the reading experience

### Key Design Decisions

**Pure client-side, no server.** The entire TTS pipeline runs in the browser. No API calls, no audio files, no caching needed. The browser IS the TTS engine.

**Device-native voices.** We don't control the voice — it's whatever the user's OS provides. This is a feature, not a limitation: iOS Siri voices are excellent, and the user already chose their preferred system voice.

**No E2 dependency.** E3 can ship as a pure static site update. This means it can ship in parallel with or even before E2, unblocking the reading experience immediately.

**Section-level granularity.** Play buttons at heading level, not paragraph level. Keeps the UI clean and maps to how people think about doc structure. "Read me the Refinement section" is natural; "read me paragraph 3" is not.

**Auto-scroll is opt-in.** Default off — some users will read ahead while listening. Toggle saved to localStorage.

---

## Edge Cases & Gotchas

| Scenario | Expected Behavior | Why It's Tricky |
|----------|-------------------|-----------------|
| Browser doesn't support Web Speech API | Play buttons hidden, graceful degradation | Feature detection: `'speechSynthesis' in window` |
| iOS Safari requires user gesture to start speech | First play must be from a click/tap event | Can't auto-play on page load — standard iOS restriction |
| Chrome 15-second timeout on long utterances | Chunk text at sentence boundaries, chain utterances | Well-known Chrome bug, standard workaround |
| Code blocks in sections | Skip by default, or read with "code block" prefix | Reading `const x = 42` aloud is awkward |
| Mermaid diagrams | Skip entirely | Can't meaningfully read a diagram |
| Tables | Read cell-by-cell with row/column context, or skip | Reading tables aloud is tricky — may need "table with N rows" summary |
| User navigates away mid-playback | Cancel speech on page navigation | `speechSynthesis.cancel()` in cleanup |
| Multiple play buttons clicked | Cancel current, start new | Only one utterance at a time |
| Very short sections (one sentence) | Still plays fine | No special handling needed |
| Dark mode | Play buttons and controls styled for both themes | Use CSS custom properties |

---

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Voice quality varies by device | Medium | Low | iOS/macOS are excellent. Chrome/Firefox are decent. Users can change their system voice. |
| Chrome long-utterance bug | High | Low | Well-known workaround (chunking). Every TTS library handles this. |
| Web Speech API deprecated | Very Low | High | W3C standard, shipped in all browsers. No deprecation signals. |
| User confusion about "no voice" | Low | Low | Show "TTS not supported" message if API missing |

---

## Testing Strategy

### Test Layers

| Layer | Applies? | Notes |
|-------|:--------:|-------|
| **Unit tests** | Yes | Text extraction, chunking logic, section parsing |
| **Component tests** | Yes | TtsControls renders correctly, button states toggle |
| **Integration tests** | Partial | Web Speech API can be mocked for state management tests |
| **Manual QA** | Yes | Dan tests on iPad (iOS Safari), laptop (Chrome/Safari) |

### Verification Rules

1. Play button appears next to every h2/h3/h4 heading
2. Clicking play reads the correct section text
3. Play All reads all sections in order
4. Pause/resume works mid-section
5. Speed control changes reading speed immediately
6. Current section is visually highlighted during playback
7. Auto-scroll follows the current section (when enabled)
8. Play buttons hidden if Web Speech API not available
9. Works on iPad Safari (primary use case)
10. Works on Chrome desktop
11. Code blocks are skipped (or handled gracefully)
12. Page navigation cancels active speech

---

## Stories

| Story | Summary | Batch | Dependencies | Status | PR |
|-------|---------|-------|-------------|--------|----| 
| S1 | Section text extraction + Web Speech API wrapper | 1 | E1 | | |
| S2 | Section play buttons (per-heading ▶️/⏸️) | 2 | S1 | | |
| S3 | Play All + sequential section playback | 2 | S1 | | |
| S4 | TTS controls bar (speed, skip, auto-scroll) | 3 | S2, S3 | | |

**Execution plan:** S1 → [S2, S3 parallel] → S4

### S1: Section Text Extraction + Web Speech API Wrapper

**Summary:** Build the utility layer that extracts readable text from doc sections and wraps the Web Speech API with chunking, event handling, and error management.

**Acceptance Criteria:**
1. `src/utils/tts-extractor.ts`:
   - Takes a DOM container element, returns ordered array of sections:
     ```typescript
     interface TtsSection {
       id: string;           // heading element id (for scrolling)
       title: string;        // heading text
       level: number;        // 2, 3, or 4
       text: string;         // readable content (HTML stripped)
       element: HTMLElement;  // reference to heading element
     }
     ```
   - Extracts text between headings (heading N content = everything until next heading of ≤ N level)
   - Strips HTML tags, keeps readable text
   - Skips Mermaid diagram containers (`.mermaid` class)
   - Skips code blocks by default (configurable: `{ includeCode?: boolean }`)
   - Tables: extracts cell text in reading order with row separators
2. `src/utils/tts-engine.ts`:
   - Wraps `speechSynthesis` API
   - `speak(text, options?)` → starts speech
   - `pause()` / `resume()` / `cancel()`
   - `setRate(rate)` — 0.75 / 1 / 1.25 / 1.5 / 2
   - Chunks long text at sentence boundaries (~200 chars) for Chrome workaround
   - Chains chunks via `onend` events (seamless to caller)
   - Events: `onStart`, `onEnd`, `onPause`, `onResume`, `onBoundary`
   - Feature detection: `isSupported()` returns boolean
   - Cleanup: `cancel()` on unmount
3. Unit tests for text extraction (mock DOM)
4. Unit tests for chunking logic (sentence splitting)
5. Feature detection works (returns false when API missing)

**Boundaries:**
- Do NOT build UI components (S2, S3, S4)
- Do NOT add to the doc layout yet
- This is the utility layer only

### S2: Section Play Buttons

**Summary:** Add a play button next to each section heading that reads that section aloud using the TTS engine.

**Acceptance Criteria:**
1. `src/components/SectionPlayButton.tsx` (React island):
   - Small ▶️ button rendered to the right of each h2/h3/h4 heading
   - Desktop: visible on hover over the heading area
   - Mobile/tablet: always visible
   - Click toggles between play (▶️) and pause (⏸️)
   - Clicking play on Section A while Section B is playing: cancel B, start A
2. Integration into `DocLayout.astro`:
   - Script that finds all headings and mounts `SectionPlayButton` next to each
   - Or: Astro component wraps headings during markdown rendering
3. Visual feedback:
   - Active section gets subtle highlight (background color via CSS custom property)
   - Highlight removed when section finishes or is cancelled
4. Styled for both light and dark themes
5. Buttons hidden if `tts-engine.isSupported()` returns false
6. `speechSynthesis.cancel()` called on page navigation (cleanup)
7. Manual QA: works on iPad Safari and Chrome desktop

**Boundaries:**
- Do NOT implement Play All or sequential playback (S3)
- Do NOT add speed controls or TTS control bar (S4)
- Single section at a time only

**Dependencies:** S1 (extraction + engine)

### S3: Play All + Sequential Section Playback

**Summary:** Add a "Play All" button at the doc title that reads the entire document section by section, with skip forward/back capability.

**Acceptance Criteria:**
1. "Play All" button rendered next to the doc title (h1)
   - Click starts sequential playback from first section
   - If a section is already playing, "Play All" starts from that section forward
2. Sequential playback logic:
   - On section end → automatically start next section
   - Highlight advances to current section
   - Skip forward: cancel current, start next section
   - Skip back: cancel current, restart current section (double-tap skip back = previous section)
3. Playback state management:
   - `isPlaying`, `isPaused`, `currentSectionIndex`, `totalSections`
   - Shared state between Play All and individual section buttons
   - If user clicks a section's ▶️ during Play All: jump to that section, continue sequential from there
4. Stop button: cancels all playback, resets state
5. Auto-scroll (foundation):
   - When playing, current section heading scrolls into view
   - Smooth scroll behavior
   - Can be toggled (state managed, toggle UI in S4)
   - Default: off (stored in localStorage key `foundry-tts-autoscroll`)
6. Works on iPad Safari and Chrome desktop

**Boundaries:**
- Do NOT add speed controls (S4)
- Do NOT add floating control bar (S4)
- Skip/stop can use basic buttons near the title for now (refined in S4)

**Dependencies:** S1 (extraction + engine)

### S4: TTS Controls Bar

**Summary:** Add a persistent controls bar for managing playback — speed control, skip, auto-scroll toggle. Appears when any playback is active.

**Acceptance Criteria:**
1. `src/components/TtsControls.tsx` (React island):
   - Sticky bar at bottom of viewport (or below header)
   - Appears with animation when playback starts
   - Disappears when playback stops/cancelled
2. Controls:
   - Play/Pause toggle button
   - Speed selector: 0.75x / 1x / 1.25x / 1.5x / 2x (segmented buttons or dropdown)
   - Skip previous / Skip next buttons (visible in Play All mode)
   - Current section title display
   - Progress: "Section 3 of 12" (in Play All mode)
   - Auto-scroll toggle (🔒 icon or similar)
   - Close/stop button (✕)
3. Speed preference persisted to localStorage key `foundry-tts-speed`
4. Speed change applies immediately to current playback
5. Styled for both light and dark themes
6. Responsive: works on mobile (simplified layout if needed)
7. Doesn't overlap with content (padding/margin adjustment when bar is visible)
8. Keyboard shortcuts (optional but nice):
   - Space: play/pause
   - Left/Right arrow: skip prev/next (when TTS bar is focused)
9. Manual QA: Dan approves the look and feel on iPad + desktop

**Boundaries:**
- Do NOT add voice selection (out of scope entirely)
- Do NOT add playback position memory across page loads
- Keep it simple — this is v1 controls

**Dependencies:** S2 (section buttons), S3 (Play All + state management)

---

## Decisions Log

| Date | Decision | Rationale | Alternatives Considered |
|------|----------|-----------|------------------------|
| Mar 30 | Web Speech API (native browser TTS) | Zero cost, works offline, no server needed, iOS voices are excellent | Edge TTS (server-side, needs API), ElevenLabs (paid, high quality but overkill) |
| Mar 30 | No E2 dependency | Pure frontend feature, can ship on static site | Wait for API server (unnecessary delay) |
| Mar 30 | Section-level play buttons at headings | Maps to how people think about docs. Clean UI. | Paragraph-level (too granular), page-level only (too coarse) |
| Mar 30 | Skip code blocks by default | Reading `const x = 42` aloud is awkward | Always include (bad UX), never include (might miss important pseudocode) |
| Mar 30 | Auto-scroll opt-in (default off) | Some users read ahead while listening | Default on (annoying if you're reading a different section), no auto-scroll (miss the context) |
| Mar 30 | Device-native voices (no selection UI) | iOS/macOS voices are great, adds zero complexity | Voice picker (scope creep, low value for our use case) |
| Mar 30 | Chrome chunking workaround | Well-documented bug, standard solution | Ignore (broken on Chrome), use different API (none available) |

---

## Known Issues / Tech Debt

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| Background tab may pause speech | Low | Accept | Browser behavior — speech pauses when tab isn't focused. No workaround. iOS handles this better than desktop. |
| Table reading is awkward | Low | Accept | Best-effort text extraction from tables. Won't sound natural but conveys info. |
| No playback position memory | Low | Deferred | User has to re-start from beginning if they leave and come back. Could save `currentSectionIndex` to localStorage in v0.3+. |
