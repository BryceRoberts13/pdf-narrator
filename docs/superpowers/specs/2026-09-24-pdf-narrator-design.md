# PDF Narrator — Design Spec

**Date:** 2026-09-24  
**Status:** Approved for implementation planning  
**Repo:** `pdf-narrator` (Chrome extension)

## Goal

A Chrome/Chromium extension that reads academic PDFs aloud with synced highlighting, chapter-level skip, and special handling for equations. HTML page narration is a later phase using the same core engine.

## Success criteria (v1)

- Open a text-based academic PDF in the extension side panel and hear it narrated in reading order.
- Current phrase/word is highlighted on the page in sync with speech.
- Play, pause, speed up/down (`[` / `]`), skip to next/previous section.
- On encountering an equation: auto-pause, show a distinct equation highlight + symbol key; on unpause, skip the equation and continue with following text.
- Works offline for basic narration (browser Web Speech API).

## Non-goals (v1)

- OCR for scanned/image-only PDFs
- Reading equations aloud as mathematics
- Voice cloning or third-party voice imitation
- Firefox / Safari
- Cloud accounts, sync, or paid API requirement
- Perfect extraction on every publisher layout

## Product decisions

| Decision | Choice |
|----------|--------|
| Surface | Chrome extension only |
| Primary content | Academic PDFs (chapters, math) |
| Viewer | Extension-owned PDF.js (not Chrome’s built-in PDF viewer) |
| TTS v1 | Web Speech API behind a pluggable adapter |
| TTS later | Local Piper/Kokoro; optional cloud; own-voice clone research only |
| Equations | Pause → key UI → skip on resume |
| Sections | PDF outline first; heading heuristics fallback |
| HTML | Phase 2 — same segment model + narrator |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Side panel UI                                           │
│  PDF.js canvas + highlight layer | transport | eq key   │
└───────────────┬─────────────────────────┬───────────────┘
                │                         │
                ▼                         ▼
┌───────────────────────────┐   ┌─────────────────────────┐
│ PDF pipeline              │   │ Narrator engine         │
│ load → items → order →    │──▶│ queue speakable units   │
│ classify → sections       │   │ sync highlight callbacks│
└───────────────────────────┘   └───────────┬─────────────┘
                                            │
                                            ▼
                                ┌─────────────────────────┐
                                │ TTS adapter             │
                                │ speak / pause / cancel  │
                                │ onBoundary / onEnd      │
                                └─────────────────────────┘
```

**Why side panel + PDF.js:** Chrome’s built-in PDF viewer does not expose reliable text geometry for sync highlighting. Owning the viewer gives bounding boxes and a stable DOM/canvas overlay.

**Phase 2 HTML:** a content script produces the same `DocumentModel` (segments + sections); the side panel or an in-page toolbar drives the same narrator engine.

---

## Core data model

```ts
type BBox = { x: number; y: number; w: number; h: number; page: number };

type SegmentKind = "heading" | "paragraph" | "equation" | "caption" | "other";

type Segment = {
  id: string;
  kind: SegmentKind;
  text: string;          // empty or placeholder for equations
  boxes: BBox[];         // for highlight
  sectionId: string;
  // equations only:
  symbols?: { glyph: string; gloss: string }[];
};

type Section = {
  id: string;
  title: string;
  level: number;         // 1 = chapter-like
  segmentIds: string[];
};

type DocumentModel = {
  source: { type: "pdf"; name: string };
  sections: Section[];
  segments: Segment[];
};
```

**Speakable units:** contiguous non-equation segments (or sub-chunks of a paragraph) that the TTS adapter can play. Equation segments are never enqueued as speech in v1; they trigger pause + UI instead.

---

## Components

### 1. Extension shell

- Manifest V3 service worker (message routing, optional offscreen doc later for audio).
- Side panel as primary UI.
- Action icon opens the side panel.
- Permissions: minimal — `sidePanel`, host access only as needed for fetching `file:` / blob PDFs the user opens; prefer user file picker / drag-drop into the panel to avoid broad host permissions in v1.

### 2. PDF pipeline

1. Load PDF with PDF.js.
2. Per page: `getTextContent` + transform matrices → text items with boxes (PDF user space → viewport).
3. **Reading order:** cluster into lines (y-proximity), sort lines top-to-bottom, items left-to-right; handle simple multi-column only if clearly separable (v1: single-column bias for course notes).
4. **Classify segments:**
   - Equation: math Unicode blocks, high symbol/letter ratio, sparse layout, known math fonts when detectable; use structural tags/MathML when the PDF provides them.
   - Heading: larger font size / outline destinations.
   - Caption: short line under figure-like gaps (best-effort).
   - Else paragraph.
5. **Sections:** build tree from `pdf.getOutline()` when present; map outline destinations to segment ranges. Fallback: consecutive heading segments as section breaks (`Chapter N`, numbered titles).
6. Emit `DocumentModel`.

### 3. Equation UX

- When the playhead reaches an equation segment: pause narration, scroll into view, apply **equation highlight style** (distinct from speech highlight).
- Side panel **key**: list detected symbols with short glosses when inferable (e.g. `Σ` → “sum”); otherwise a single line like “Equation (skipped on resume)”.
- User hits play/unpause → advance playhead past the equation to the next speakable segment.
- No TTS of raw TeX or symbol soup in v1.

### 4. Narrator engine

- Input: `DocumentModel` + start index (segment or section).
- State: `idle | playing | paused-equation | paused-user`.
- Advances through speakable units; invokes TTS adapter; on word/char boundary (when available) maps to highlight boxes; on unit end advances.
- Controls: play, pause, setRate, skipSection±, seekToSegment.
- Keyboard: `[` slower, `]` faster (mpv-style), Space play/pause when panel focused.

### 5. TTS adapter interface

```ts
interface TtsAdapter {
  speak(text: string, opts: { rate: number }): Promise<void>;
  pause(): void;
  resume(): void;
  cancel(): void;
  // Optional; Web Speech may provide word boundaries inconsistently
  onBoundary?: (cb: (charIndex: number) => void) => void;
}
```

- **v1:** `WebSpeechAdapter` (browser voices; quality varies by OS).
- **v1.5:** `PiperAdapter` or `KokoroAdapter` (local) behind the same interface.
- Highlight fallback when boundaries are missing: estimate timing from unit duration / character count, or highlight whole phrase until `onEnd`.

### 6. Highlight + transport UI

- Overlay divs (or canvas) aligned to PDF.js viewport; recompute on zoom/scroll/page change.
- Speech highlight vs equation highlight (two styles).
- Section list with current section marked; click to jump.
- Speed readout; equation key panel (collapsible).

---

## Data flow (play)

1. User opens PDF → pipeline builds `DocumentModel` → UI renders page 1 + section list.
2. User presses Play → engine starts at first speakable segment (or current selection).
3. For each speakable unit: highlight boxes → `tts.speak` → on end → next unit.
4. If next segment is equation: enter `paused-equation`, show key, wait.
5. On resume: skip equation, continue.
6. Skip section: move playhead to first speakable segment of next/prev section; cancel current utterance.

---

## Error handling

| Case | Behavior |
|------|----------|
| Encrypted / unreadable PDF | Clear error in panel; no crash |
| No extractable text (scanned) | Message: “This PDF has no text layer; OCR not in v1” |
| Empty outline | Heuristic sections or single “Document” section |
| Web Speech unavailable / no voices | Error with hint to check OS TTS; block Play |
| TTS interrupted (tab sleep) | Pause state; user can resume |
| Highlight misalignment after zoom | Rebind boxes on viewport change; if mismatch, degrade to line-level highlight |

---

## Testing strategy

- **Unit:** reading-order sort; equation classifier fixtures (sample text items); outline → section mapping; narrator state machine (play → equation pause → skip → next).
- **Fixture PDFs:** one clean textbook-like PDF with outline + equations; one outline-less notes PDF; one textless scanned page (expect graceful failure).
- **Manual:** Chrome side panel load, speed keys, section skip, equation pause/resume on a real course PDF.
- No requirement for automated e2e in v1; add Playwright later if valuable.

---

## Implementation phases

1. **Scaffold** — MV3 extension, side panel, PDF.js viewer, open local PDF.
2. **Extract** — text + boxes + reading order + basic sections.
3. **Narrate** — Web Speech + phrase highlight + transport + speed keys.
4. **Equations** — detect, pause, key UI, skip on resume.
5. **Polish** — section list UX, error states, sample fixtures.
6. **Later** — HTML path; Piper/Kokoro adapter; voice-clone spike (own voice only).

---

## Voice exploration (parallel notes)

- Open-source candidates: Piper, Kokoro, Coqui XTTS (clone-capable).
- Cloning **your** voice with consent is fine for a later spike; cloning someone else without consent is out of scope and disallowed.
- Keep exploration out of the v1 critical path; document findings in `docs/` when spiked.

---

## Open risks (accepted)

- Web Speech quality and boundary events vary by OS/browser.
- Equation detection will be imperfect; false positives pause too often — tune with fixtures.
- Complex multi-column journal layouts may read out of order until heuristics improve.
```
