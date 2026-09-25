# PDF Narrator v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a Chrome MV3 side-panel extension that opens an academic PDF, narrates it with Web Speech, sync-highlights text, skips by section, and auto-pauses on equations with a symbol key.

**Architecture:** Vite + TypeScript extension (`@crxjs/vite-plugin`). Side panel hosts PDF.js viewer + UI. Pure TS modules build a `DocumentModel` (segments/sections), drive a narrator state machine, and speak through a `TtsAdapter` (Web Speech first). Highlights are DOM overlays on the PDF.js viewport.

**Tech Stack:** TypeScript, Vite 6, `@crxjs/vite-plugin`, `pdfjs-dist`, Vitest, Chrome Manifest V3 (side panel). No React in v1 — vanilla TS + CSS.

## Global Constraints

- Chrome/Chromium extension only (Manifest V3); no Firefox/Safari in v1.
- Open PDFs via file picker / drag-drop in the side panel (no broad host permissions).
- TTS v1 = Web Speech API behind `TtsAdapter`; no paid API required.
- Equations are never spoken; pause → key UI → skip on resume.
- No OCR; scanned PDFs show a clear error.
- Types and names must match `docs/superpowers/specs/2026-09-24-pdf-narrator-design.md`.
- Unit tests with Vitest; TDD for pure logic modules; frequent commits.

---

## File structure

```
package.json
tsconfig.json
vite.config.ts
vitest.config.ts
manifest.config.ts          # CRX manifest (MV3 + side_panel)
index.html                  # unused shell if CRX needs it — prefer sidepanel only
src/
  background.ts             # service worker: open side panel on action click
  sidepanel/
    index.html
    main.ts                 # wire UI ↔ pipeline ↔ narrator
    styles.css
    viewer.ts               # PDF.js render + highlight overlay
    transport.ts            # play/pause/speed/section controls + keys
    equationKey.ts          # equation symbol key panel
  model/
    types.ts                # BBox, Segment, Section, DocumentModel
  pdf/
    textItems.ts            # PDF.js text → TextItemWithBox[]
    readingOrder.ts         # lines + sort
    classify.ts             # segment kinds + equation symbols
    sections.ts             # outline + heuristic sections
    buildDocument.ts        # orchestrate → DocumentModel
  narrate/
    tts.ts                  # TtsAdapter + WebSpeechAdapter
    engine.ts               # narrator state machine
  lib/
    id.ts                   # simple id helper
tests/
  readingOrder.test.ts
  classify.test.ts
  sections.test.ts
  engine.test.ts
  tts.webSpeech.test.ts
fixtures/
  textItems/                # JSON fixtures for classifiers
  README.md                 # how to add a real course PDF for manual test
docs/superpowers/specs/2026-09-24-pdf-narrator-design.md
```

---

### Task 1: Scaffold extension + Vitest

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `vitest.config.ts`, `manifest.config.ts`
- Create: `src/background.ts`, `src/sidepanel/index.html`, `src/sidepanel/main.ts`, `src/sidepanel/styles.css`
- Modify: `README.md` (dev/load instructions)
- Modify: `.gitignore` (add `*.local`, coverage if needed)

**Interfaces:**
- Consumes: nothing
- Produces: loadable unpacked extension; `npm test` runs Vitest (empty suite OK until Task 2)

- [ ] **Step 1: Init package and install deps**

```bash
cd "/Users/bryceroberts/Desktop/Fall 2026/Coding Projects/pdf-narrator"
npm init -y
npm install -D vite@6 typescript @types/chrome vitest @crxjs/vite-plugin@2
npm install pdfjs-dist
npx tsc --init --strict --module ESNext --moduleResolution bundler --target ES2022 --lib DOM,ES2022 --types chrome,vite/client
```

- [ ] **Step 2: Add `manifest.config.ts`**

```ts
import { defineManifest } from "@crxjs/vite-plugin";

export default defineManifest({
  manifest_version: 3,
  name: "PDF Narrator",
  version: "0.1.0",
  description: "Narrate academic PDFs with synced highlighting",
  action: { default_title: "PDF Narrator" },
  background: { service_worker: "src/background.ts", type: "module" },
  side_panel: { default_path: "src/sidepanel/index.html" },
  permissions: ["sidePanel"],
});
```

- [ ] **Step 3: Add Vite + Vitest configs**

`vite.config.ts`:
```ts
import { defineConfig } from "vite";
import { crx } from "@crxjs/vite-plugin";
import manifest from "./manifest.config";

export default defineConfig({
  plugins: [crx({ manifest })],
  build: { outDir: "dist", sourcemap: true },
});
```

`vitest.config.ts`:
```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: { environment: "node", include: ["tests/**/*.test.ts"] },
});
```

`package.json` scripts:
```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

- [ ] **Step 4: Background opens side panel; side panel stub**

`src/background.ts`:
```ts
chrome.sidePanel.setPanelBehavior({ openPanelOnActionClick: true }).catch(() => {});
```

`src/sidepanel/index.html`:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>PDF Narrator</title>
    <link rel="stylesheet" href="./styles.css" />
  </head>
  <body>
    <header class="toolbar">
      <h1>PDF Narrator</h1>
      <input type="file" id="file" accept="application/pdf" />
    </header>
    <p id="status">Open a PDF to begin.</p>
    <main id="app"></main>
    <script type="module" src="./main.ts"></script>
  </body>
</html>
```

`src/sidepanel/main.ts`:
```ts
const status = document.getElementById("status")!;
const fileInput = document.getElementById("file") as HTMLInputElement;
fileInput.addEventListener("change", () => {
  const f = fileInput.files?.[0];
  status.textContent = f ? `Selected: ${f.name}` : "Open a PDF to begin.";
});
```

`src/sidepanel/styles.css`: minimal readable layout (toolbar + status); dark text on light background is fine for v1.

- [ ] **Step 5: Build and smoke-load in Chrome**

```bash
npm run build
```

Expected: `dist/` with manifest. Chrome → Extensions → Load unpacked → select `dist/`. Click action → side panel opens with file input.

- [ ] **Step 6: Update README with load steps; commit**

```bash
git add package.json package-lock.json tsconfig.json vite.config.ts vitest.config.ts manifest.config.ts src README.md .gitignore
git commit -m "chore: scaffold Chrome MV3 side panel with Vite"
```

---

### Task 2: Core types + reading order

**Files:**
- Create: `src/model/types.ts`, `src/lib/id.ts`, `src/pdf/readingOrder.ts`
- Test: `tests/readingOrder.test.ts`

**Interfaces:**
- Consumes: none
- Produces:
  - Types: `BBox`, `SegmentKind`, `Segment`, `Section`, `DocumentModel` (exact fields from design spec)
  - `export type TextItemWithBox = { str: string; box: BBox; fontHeight: number }`
  - `export function orderTextItems(items: TextItemWithBox[]): TextItemWithBox[]`

- [ ] **Step 1: Write failing reading-order test**

```ts
// tests/readingOrder.test.ts
import { describe, expect, it } from "vitest";
import { orderTextItems } from "../src/pdf/readingOrder";

describe("orderTextItems", () => {
  it("sorts top-to-bottom, then left-to-right within a line", () => {
    const items = [
      { str: "B", box: { x: 100, y: 200, w: 10, h: 12, page: 0 }, fontHeight: 12 },
      { str: "A", box: { x: 50, y: 200, w: 10, h: 12, page: 0 }, fontHeight: 12 },
      { str: "C", box: { x: 50, y: 100, w: 10, h: 12, page: 0 }, fontHeight: 12 },
    ];
    expect(orderTextItems(items).map((i) => i.str).join("")).toBe("CAB");
  });
});
```

- [ ] **Step 2: Run test — expect fail**

```bash
npm test -- tests/readingOrder.test.ts
```

Expected: FAIL (module not found or function missing)

- [ ] **Step 3: Add types + implementation**

`src/model/types.ts` — copy the design-spec types verbatim (`BBox`, `SegmentKind`, `Segment`, `Section`, `DocumentModel`).

`src/lib/id.ts`:
```ts
let n = 0;
export function nextId(prefix: string): string {
  n += 1;
  return `${prefix}-${n}`;
}
export function resetIdsForTests(): void {
  n = 0;
}
```

`src/pdf/readingOrder.ts`:
```ts
import type { BBox } from "../model/types";

export type TextItemWithBox = {
  str: string;
  box: BBox;
  fontHeight: number;
};

/** Cluster by y (line), sort lines by y ascending (PDF y often bottom-up — normalize: treat smaller y as higher on page if using viewport coords where y grows downward). */
export function orderTextItems(items: TextItemWithBox[]): TextItemWithBox[] {
  if (items.length === 0) return [];
  const sorted = [...items].sort((a, b) => a.box.y - b.box.y || a.box.x - b.box.x);
  const lines: TextItemWithBox[][] = [];
  const threshold = (h: number) => Math.max(4, h * 0.5);
  for (const item of sorted) {
    const last = lines[lines.length - 1];
    if (
      last &&
      Math.abs(item.box.y - last[0].box.y) <= threshold(item.fontHeight)
    ) {
      last.push(item);
    } else {
      lines.push([item]);
    }
  }
  for (const line of lines) line.sort((a, b) => a.box.x - b.box.x);
  return lines.flat();
}
```

Note: PDF.js viewport coordinates typically have **y increasing downward**. Keep that convention in `BBox` everywhere (document in a one-line comment on `BBox`).

- [ ] **Step 4: Run test — expect pass**

```bash
npm test -- tests/readingOrder.test.ts
```

Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/model/types.ts src/lib/id.ts src/pdf/readingOrder.ts tests/readingOrder.test.ts
git commit -m "feat: add document types and reading-order sort"
```

---

### Task 3: Classify segments + equation symbols

**Files:**
- Create: `src/pdf/classify.ts`
- Create: `fixtures/textItems/paragraph.json`, `fixtures/textItems/equation.json`
- Test: `tests/classify.test.ts`

**Interfaces:**
- Consumes: `TextItemWithBox`, `Segment` fields
- Produces:
  - `export function classifyRun(items: TextItemWithBox[]): { kind: SegmentKind; text: string; symbols?: { glyph: string; gloss: string }[] }`
  - `export function groupIntoRuns(ordered: TextItemWithBox[]): TextItemWithBox[][]` (split on large y-gaps / blank lines)
  - Built-in gloss map for common glyphs (`Σ`→`sum`, `∫`→`integral`, `α`→`alpha`, …); unknown glyphs get gloss `"symbol"`

- [ ] **Step 1: Write failing classifier tests**

```ts
import { describe, expect, it } from "vitest";
import { classifyRun } from "../src/pdf/classify";

describe("classifyRun", () => {
  it("labels plain prose as paragraph", () => {
    const items = [
      { str: "This is a normal sentence about graphs.", box: { x: 0, y: 0, w: 200, h: 12, page: 0 }, fontHeight: 12 },
    ];
    expect(classifyRun(items).kind).toBe("paragraph");
  });

  it("labels symbol-heavy runs as equation and extracts symbols", () => {
    const items = [
      { str: "Σ", box: { x: 0, y: 0, w: 10, h: 14, page: 0 }, fontHeight: 14 },
      { str: "ᵢ", box: { x: 12, y: 0, w: 8, h: 14, page: 0 }, fontHeight: 14 },
      { str: " xᵢ", box: { x: 20, y: 0, w: 30, h: 14, page: 0 }, fontHeight: 14 },
    ];
    const r = classifyRun(items);
    expect(r.kind).toBe("equation");
    expect(r.symbols?.some((s) => s.glyph === "Σ" && s.gloss === "sum")).toBe(true);
  });

  it("labels large-font short lines as heading", () => {
    const items = [
      { str: "Chapter 1", box: { x: 0, y: 0, w: 80, h: 24, page: 0 }, fontHeight: 24 },
    ];
    expect(classifyRun(items).kind).toBe("heading");
  });
});
```

- [ ] **Step 2: Run — expect fail**

```bash
npm test -- tests/classify.test.ts
```

- [ ] **Step 3: Implement `classify.ts`**

Heuristics (keep tunable constants at top of file):
- `equation` if math-unicode ratio ≥ 0.3 **or** (letters < 40% of non-space chars and length ≤ 80)
- `heading` if `fontHeight >= 1.4 * medianBodyHeight` (pass median as arg or compute from run) and text length < 80
- else `paragraph`
- `extractSymbols(text)`: unique chars matching `/[\u2200-\u22FF\u0370-\u03FF∑∫∏√∞≈≠≤≥∂∇]/` plus map lookup

```ts
const GLOSS: Record<string, string> = {
  "Σ": "sum", "∑": "sum", "∫": "integral", "α": "alpha", "β": "beta",
  "θ": "theta", "π": "pi", "∞": "infinity", "≤": "less or equal", "≥": "greater or equal",
};
```

- [ ] **Step 4: Run — expect pass; commit**

```bash
npm test -- tests/classify.test.ts
git add src/pdf/classify.ts tests/classify.test.ts fixtures/textItems
git commit -m "feat: classify text runs including equations"
```

---

### Task 4: Section builder

**Files:**
- Create: `src/pdf/sections.ts`
- Test: `tests/sections.test.ts`

**Interfaces:**
- Consumes: `Segment[]` with kinds; optional outline entries
- Produces:
  - `export type OutlineEntry = { title: string; segmentIndex: number; level: number }`
  - `export function buildSections(segments: Omit<Segment, "sectionId">[], outline?: OutlineEntry[]): { sections: Section[]; segments: Segment[] }`
  - If `outline` present and non-empty: one section per outline entry; assign `sectionId` by segment index ranges
  - Else: start a section at each `heading` segment; if none, single section titled `"Document"`

- [ ] **Step 1: Failing tests**

```ts
import { describe, expect, it } from "vitest";
import { buildSections } from "../src/pdf/sections";
import { resetIdsForTests } from "../src/lib/id";

describe("buildSections", () => {
  it("uses outline when provided", () => {
    resetIdsForTests();
    const segs = [
      { id: "s0", kind: "heading" as const, text: "Ch1", boxes: [] },
      { id: "s1", kind: "paragraph" as const, text: "a", boxes: [] },
      { id: "s2", kind: "heading" as const, text: "Ch2", boxes: [] },
      { id: "s3", kind: "paragraph" as const, text: "b", boxes: [] },
    ];
    const { sections, segments } = buildSections(segs, [
      { title: "Chapter 1", segmentIndex: 0, level: 1 },
      { title: "Chapter 2", segmentIndex: 2, level: 1 },
    ]);
    expect(sections).toHaveLength(2);
    expect(sections[0].title).toBe("Chapter 1");
    expect(segments[1].sectionId).toBe(sections[0].id);
    expect(segments[3].sectionId).toBe(sections[1].id);
  });

  it("falls back to headings", () => {
    resetIdsForTests();
    const segs = [
      { id: "s0", kind: "heading" as const, text: "Intro", boxes: [] },
      { id: "s1", kind: "paragraph" as const, text: "hi", boxes: [] },
    ];
    const { sections } = buildSections(segs);
    expect(sections[0].title).toBe("Intro");
  });

  it("uses Document when no headings", () => {
    resetIdsForTests();
    const segs = [{ id: "s0", kind: "paragraph" as const, text: "only", boxes: [] }];
    const { sections } = buildSections(segs);
    expect(sections).toHaveLength(1);
    expect(sections[0].title).toBe("Document");
  });
});
```

- [ ] **Step 2–4: Fail → implement → pass → commit**

```bash
npm test -- tests/sections.test.ts
git add src/pdf/sections.ts tests/sections.test.ts
git commit -m "feat: build sections from outline or headings"
```

---

### Task 5: PDF pipeline → `DocumentModel` + viewer open

**Files:**
- Create: `src/pdf/textItems.ts`, `src/pdf/buildDocument.ts`, `src/sidepanel/viewer.ts`
- Modify: `src/sidepanel/main.ts`, `src/sidepanel/index.html`, `src/sidepanel/styles.css`
- Test: `tests/buildDocument.test.ts` (uses synthetic items, not a real PDF)

**Interfaces:**
- Consumes: `orderTextItems`, `classifyRun` / `groupIntoRuns`, `buildSections`
- Produces:
  - `export async function extractTextItems(pdf: PDFDocumentProxy): Promise<TextItemWithBox[]>`
  - `export function buildDocumentFromItems(name: string, items: TextItemWithBox[], outline?: OutlineEntry[]): DocumentModel`
  - `export async function buildDocumentFromPdf(file: File): Promise<DocumentModel>` — loads pdf.js, extracts, maps outline destinations to segment indices best-effort (if mapping fails, call `buildSections` without outline)
  - `export class PdfViewer` with `load(file: File)`, `renderPage(pageIndex: number)`, `setHighlights(boxes: BBox[], className: string)`, `clearHighlights()`, `scrollToBoxes(boxes: BBox[])`
  - Empty text → throw `Error` with message exactly: `This PDF has no text layer; OCR not in v1`

- [ ] **Step 1: Test `buildDocumentFromItems`**

```ts
import { describe, expect, it } from "vitest";
import { buildDocumentFromItems } from "../src/pdf/buildDocument";
import { resetIdsForTests } from "../src/lib/id";

it("builds paragraphs and preserves page boxes", () => {
  resetIdsForTests();
  const model = buildDocumentFromItems("demo.pdf", [
    { str: "Hello ", box: { x: 0, y: 10, w: 40, h: 12, page: 0 }, fontHeight: 12 },
    { str: "world", box: { x: 40, y: 10, w: 40, h: 12, page: 0 }, fontHeight: 12 },
  ]);
  expect(model.source).toEqual({ type: "pdf", name: "demo.pdf" });
  expect(model.segments.some((s) => s.text.includes("Hello"))).toBe(true);
  expect(model.sections.length).toBeGreaterThan(0);
});
```

- [ ] **Step 2: Implement pipeline**

`textItems.ts`: for each page, `getViewport({ scale: 1 })`, `getTextContent()`, convert each item’s transform to `{x,y,w,h,page}` in **viewport** coordinates (y downward). Skip empty `str`.

`buildDocument.ts`: order → group runs → classify each run into partial segments → `buildSections` → `DocumentModel`.

Wire `pdfjs-dist` worker in viewer/main:
```ts
import * as pdfjs from "pdfjs-dist";
pdfjs.GlobalWorkerOptions.workerSrc = new URL(
  "pdfjs-dist/build/pdf.worker.min.mjs",
  import.meta.url,
).toString();
```

- [ ] **Step 3: PdfViewer UI**

HTML: `#viewer` scroll container with `#pages` and overlay `#highlights`. On file select: `buildDocumentFromPdf` + `viewer.load`; show section count in `#status`. On no-text error, set status to that error message.

- [ ] **Step 4: Manual check** — open a course PDF; pages render; status shows segments/sections.

- [ ] **Step 5: Commit**

```bash
git add src/pdf src/sidepanel tests/buildDocument.test.ts
git commit -m "feat: extract DocumentModel and render PDF in side panel"
```

---

### Task 6: TTS adapter + narrator engine

**Files:**
- Create: `src/narrate/tts.ts`, `src/narrate/engine.ts`
- Test: `tests/engine.test.ts`, `tests/tts.webSpeech.test.ts`

**Interfaces:**
- Consumes: `DocumentModel`
- Produces:

```ts
export type NarratorState = "idle" | "playing" | "paused-equation" | "paused-user";

export interface TtsAdapter {
  speak(text: string, opts: { rate: number }): Promise<void>;
  pause(): void;
  resume(): void;
  cancel(): void;
  onBoundary(cb: ((charIndex: number) => void) | null): void;
}

export class WebSpeechAdapter implements TtsAdapter { /* ... */ }

export type NarratorHandlers = {
  onState(state: NarratorState): void;
  onHighlight(boxes: BBox[], kind: "speech" | "equation"): void;
  onEquation(segment: Segment): void;
  onClearEquation(): void;
  onSection(sectionId: string): void;
};

export class NarratorEngine {
  constructor(tts: TtsAdapter, handlers: NarratorHandlers);
  load(model: DocumentModel): void;
  play(): void;
  pause(): void;           // user pause → paused-user
  resume(): void;          // from paused-user OR skip equation if paused-equation
  setRate(rate: number): void;
  skipSection(delta: 1 | -1): void;
  seekToSegment(segmentId: string): void;
  getRate(): number;
  getState(): NarratorState;
}
```

Engine rules:
- Speak only non-equation segments with non-empty `text`.
- Before speaking, `onHighlight(seg.boxes, "speech")`.
- When next segment is `equation`: `cancel` speech, `onHighlight(eq.boxes, "equation")`, `onEquation(eq)`, state `paused-equation`.
- `resume()` from `paused-equation`: `onClearEquation()`, advance index past equation, continue playing.
- `pause()` while playing → `tts.pause()`, `paused-user`.
- `skipSection`: cancel, move to first speakable segment of next/prev section, play if was playing.
- Default rate `1`; clamp rate to `[0.5, 3]`.

- [ ] **Step 1: Fake TTS + engine tests**

```ts
class FakeTts implements TtsAdapter {
  spoken: string[] = [];
  speak(text: string) { this.spoken.push(text); return Promise.resolve(); }
  pause() {}
  resume() {}
  cancel() {}
  onBoundary() {}
}

it("pauses on equation and skips on resume", async () => {
  const model: DocumentModel = {
    source: { type: "pdf", name: "t.pdf" },
    sections: [{ id: "sec-1", title: "S", level: 1, segmentIds: ["a", "e", "b"] }],
    segments: [
      { id: "a", kind: "paragraph", text: "Before", boxes: [], sectionId: "sec-1" },
      { id: "e", kind: "equation", text: "", boxes: [{ x: 0, y: 0, w: 1, h: 1, page: 0 }], sectionId: "sec-1", symbols: [{ glyph: "Σ", gloss: "sum" }] },
      { id: "b", kind: "paragraph", text: "After", boxes: [], sectionId: "sec-1" },
    ],
  };
  const states: NarratorState[] = [];
  const tts = new FakeTts();
  const eng = new NarratorEngine(tts, {
    onState: (s) => states.push(s),
    onHighlight: () => {},
    onEquation: () => {},
    onClearEquation: () => {},
    onSection: () => {},
  });
  eng.load(model);
  eng.play();
  await Promise.resolve();
  expect(tts.spoken).toContain("Before");
  expect(eng.getState()).toBe("paused-equation");
  eng.resume();
  await Promise.resolve();
  expect(tts.spoken).toContain("After");
});
```

Also test: `skipSection(1)` jumps; user `pause` → `paused-user`.

- [ ] **Step 2: Implement `WebSpeechAdapter`**

Use `speechSynthesis` + `SpeechSynthesisUtterance`. `speak` returns Promise that resolves on `onend` / rejects on `onerror`. `onBoundary` maps `event.charIndex` when `event.name === "word"`. If `speechSynthesis` missing, methods throw with message: `Web Speech unavailable — check OS text-to-speech settings`.

- [ ] **Step 3: Vitest for WebSpeech — mock `globalThis.speechSynthesis`** with a tiny fake that fires `onend`; assert `speak` resolves.

- [ ] **Step 4: Pass tests; commit**

```bash
npm test -- tests/engine.test.ts tests/tts.webSpeech.test.ts
git add src/narrate tests/engine.test.ts tests/tts.webSpeech.test.ts
git commit -m "feat: narrator engine and Web Speech TTS adapter"
```

---

### Task 7: Transport UI, highlights, equations, sections

**Files:**
- Create: `src/sidepanel/transport.ts`, `src/sidepanel/equationKey.ts`
- Modify: `src/sidepanel/main.ts`, `src/sidepanel/index.html`, `src/sidepanel/styles.css`, `src/sidepanel/viewer.ts`
- Test: manual checklist (below); keep unit coverage from Tasks 2–6 green

**Interfaces:**
- Consumes: `NarratorEngine`, `PdfViewer`, `DocumentModel`
- Produces: wired side panel — full v1 loop

- [ ] **Step 1: Extend HTML**

Add:
- `#play`, `#pause`, `#rate` readout
- `#prevSection`, `#nextSection`
- `#sections` list
- `#equationKey` (hidden by default)
- Viewer already present

- [ ] **Step 2: `transport.ts`**

Bind buttons to engine. Keyboard on `window` when panel focused:
- `Space` → play/pause toggle (prevent scroll)
- `[` → `setRate(rate - 0.1)`
- `]` → `setRate(rate + 0.1)`
Update `#rate` text as `1.0x`.

- [ ] **Step 3: Wire highlights + equation key**

In `main.ts` handlers:
- `onHighlight` → `viewer.setHighlights(boxes, kind)` and scroll into view
- `onEquation` → show `#equationKey` via `renderEquationKey(segment)` (list `glyph — gloss`, or single line `Equation (skipped on resume)` if no symbols)
- `onClearEquation` → hide key, clear equation highlights
- `onSection` → mark active item in `#sections`
- Section click → `seekToSegment` first segment of that section + `play`

Speech highlight CSS: semi-transparent blue. Equation: semi-transparent amber/orange. Never use purple glow.

- [ ] **Step 4: Error paths in UI**

- No voices / Web Speech throw → status message, disable Play
- Encrypted PDF / pdf.js failure → `Could not open PDF` + error detail
- Empty text → exact OCR message from Task 5

- [ ] **Step 5: Manual test checklist**

1. Load unpacked `dist/` after `npm run build`
2. Open a text PDF → pages + section list
3. Play → speech + blue highlight advances
4. Hit an equation → pauses, amber highlight, key visible; Resume → skips to following text
5. `[` / `]` change speed; Space pauses
6. Next section skips forward
7. Image-only PDF → OCR error message

- [ ] **Step 6: Commit**

```bash
git add src/sidepanel
git commit -m "feat: transport UI, highlights, and equation key"
```

---

### Task 8: Fixtures note + README polish

**Files:**
- Create: `fixtures/README.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: working extension from Task 7
- Produces: documented manual test path; all `npm test` green

- [ ] **Step 1: Write `fixtures/README.md`**

Explain: unit tests use JSON/synthetic items; for manual QA drop a course PDF locally (do not commit copyrighted PDFs). Optional: list recommended properties (has outline, has equations).

- [ ] **Step 2: README — scripts, load unpacked, keyboard map, v1 limits (no OCR, no HTML yet)**

- [ ] **Step 3: Full test + build**

```bash
npm test
npm run build
```

Expected: all tests PASS; build succeeds.

- [ ] **Step 4: Commit**

```bash
git add fixtures/README.md README.md
git commit -m "docs: usage, fixtures, and v1 limits"
```

---

## Out of scope (do not implement in this plan)

- HTML page narration
- Piper/Kokoro / voice clone
- OCR
- Playwright e2e

---

## Self-review (plan vs spec)

| Spec requirement | Task |
|------------------|------|
| MV3 side panel + PDF.js viewer | 1, 5 |
| File picker open (minimal permissions) | 1, 5 |
| Reading order + DocumentModel | 2, 5 |
| Sections from outline / headings | 4, 5 |
| Web Speech + TtsAdapter | 6 |
| Narrator states + skip section | 6, 7 |
| Sync highlight | 5, 7 |
| Equation pause, key, skip on resume | 3, 6, 7 |
| Speed `[` / `]` | 7 |
| Errors: no text, unreadable, no TTS | 5, 7 |
| Unit tests for order/classify/sections/engine | 2–6 |
| HTML / Piper / clone | Explicitly out of scope |

No TBD placeholders. Types aligned with design spec. Coordinate system: viewport y-down documented on `BBox`.
