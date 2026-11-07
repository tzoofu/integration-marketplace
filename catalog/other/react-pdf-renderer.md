# @react-pdf/renderer (Hebrew RTL server-side PDF)

- **category**: other
- **provider**: @react-pdf/renderer (open-source, npm)
- **reusable**: yes — the bidi-run-splitting + font-registration + crash-fallback pattern is a generalizable recipe for any repo that needs to render Hebrew/RTL-mixed-with-LTR text server-side into a PDF with this library.
- **docs**: https://react-pdf.org/

## Overview
Server-side PDF generation (invoices/quotes) using `@react-pdf/renderer`'s React-component-to-PDF renderer (`renderToBuffer`), with a from-scratch reordering layer to work around a real upstream bug in the library's bidi (bidirectional text) handling: it crashes or visually corrupts certain combinations of Hebrew text mixed with Latin/digit runs or ASCII punctuation. The fix is done at three levels — a Unicode-aware bidi run splitter, a Hebrew-punctuation normalizer, and a render-time retry/fallback ladder that progressively strips free-text content if rendering still throws.

## Playbook

### Prerequisites
- `@react-pdf/renderer` and a font capable of rendering Hebrew glyphs (e.g. `@fontsource/noto-sans-hebrew`) installed as dependencies.
- `bidi-js` for standalone Unicode bidi-level computation (used to pre-split text into runs, independent of react-pdf's own internal reordering).
- Willingness to render text yourself run-by-run rather than trusting the library's automatic RTL layout for any paragraph mixing Hebrew with Latin/digits/certain punctuation.

### Setup steps
1. Register a Hebrew-capable font once at module load via `Font.register({ family, fonts: [...woff files by weight] })`, and disable automatic hyphenation (`Font.registerHyphenationCallback((word) => [word])`) since default hyphenation breaks non-Latin scripts.
2. Build a bidi-run splitter using `bidi-js`'s `getEmbeddingLevels()` (not react-pdf's own reordering) to classify each character as LTR/RTL, then chunk the text into homogeneous-direction runs — this becomes the actual unit you hand to `<Text>` components, instead of one `<Text>` per full paragraph.
3. Within RTL runs, further isolate certain ASCII punctuation (comma, period, colon, parens, dashes) into their own runs — these specific characters, when glyph-adjacent to Hebrew letters, are what triggers the library's crash/corruption.
4. Normalize ASCII quote/apostrophe characters to their correct Hebrew typographic equivalents (gershayim `״` / geresh `׳`) before layout — this sidesteps another concrete crash trigger (the common Hebrew abbreviation pattern using a straight double-quote) while also being the typographically correct choice.
5. For clauses that should stay atomic despite containing neutral characters sandwiched between digit runs (e.g. a parenthetical time range, or a `date: number` pair), wrap them in Unicode directional isolate marks (LRI `⁦` / PDI `⁩`) before bidi analysis, then strip the marks after — this stops the bidi algorithm from fragmenting them.
6. Do your own word-wrapping at a conservative character budget before render, treating any of the atomic clauses above as a single unbreakable token — react-pdf's own line-wrapping combined with bidi reordering is what crashes on long mixed-script lines.
7. Wrap every render call in a try/catch that recognizes the specific crash signature (error message/stack mentioning the library's internal reordering functions) and retries with parts of the content progressively stripped (e.g. free-text line descriptions first, then per-entry notes) — re-throw immediately for any other kind of error, since those won't be fixed by stripping content and would just fail identically on retry.

### Core pattern
```ts
import bidiFactory from "bidi-js";
const bidi = bidiFactory();

// Punctuation that corrupts/crashes rendering when glyph-adjacent to Hebrew.
const ISOLATE_CHARS = /([,.:;!?()\-–—])/;

export function normalizeHebrewPunctuation(text: string): string {
  return text.replace(/"/g, "״").replace(/'/g, "׳"); // gershayim / geresh
}

// Split text into homogeneous-direction runs ourselves, instead of trusting
// the library's own internal reordering, which breaks on Hebrew mixed with
// Latin/digits or certain ASCII punctuation.
export function splitBidiRuns(text: string): string[] {
  if (!text) return [text];
  const { levels } = bidi.getEmbeddingLevels(text, "rtl");
  const runs: string[] = [];
  const runIsRtl: boolean[] = [];
  let start = 0;
  for (let i = 1; i <= text.length; i++) {
    if (i === text.length || levels[i] % 2 !== levels[start] % 2) {
      runs.push(text.slice(start, i));
      runIsRtl.push(levels[start] % 2 === 1);
      start = i;
    }
  }
  // Only isolate punctuation within RTL runs.
  return runs
    .flatMap((run, i) => (runIsRtl[i] ? run.split(ISOLATE_CHARS).filter(Boolean) : [run]))
    .reverse(); // visual order for an RTL paragraph
}

// Recognize the library's specific internal crash signature so only that
// class of error triggers a fallback retry; anything else should propagate.
function isBidiTextkitCrash(error: unknown): boolean {
  const text = error instanceof Error ? `${error.message}\n${error.stack ?? ""}` : String(error);
  return /reorderLine|getItemAtIndex|textkit/i.test(text);
}

async function renderWithFallback<T>(
  render: (input: T) => Promise<Buffer>,
  input: T,
  strip: (input: T) => T
): Promise<Buffer> {
  try {
    return await render(input);
  } catch (error) {
    if (!isBidiTextkitCrash(error)) throw error;
    return render(strip(input)); // retry once with risky content stripped
  }
}
```

```tsx
// A <Text>-per-run component that renders each bidi run separately.
function BidiText({ text, style }: { text: string; style?: object }) {
  const runs = splitBidiRuns(normalizeHebrewPunctuation(text));
  if (runs.length <= 1) return <Text style={style}>{text || " "}</Text>;
  return (
    <View style={{ flexDirection: "row", justifyContent: "flex-end" }}>
      {runs.map((run, i) => (
        <Text key={i} style={style}>{run.trim()}</Text>
      ))}
    </View>
  );
}
```

### Env vars
None — font files ship as an npm dependency (`@fontsource/*`) and are loaded from `node_modules` at render time; no external API keys involved.

### Gotchas
- `@react-pdf/renderer`'s internal bidi reordering (in its `textkit` dependency) has a real, reproducible upstream bug: it can throw outright or silently render garbled text on Hebrew mixed with Latin/digit runs, or on certain ASCII punctuation (comma, period, dash, and by extension the rest of that punctuation class) glyph-adjacent to Hebrew. This isn't a one-off edge case — expect to hit it on realistic invoice/quote content (business names, dates, phone numbers, prices).
- A straight ASCII double-quote/apostrophe next to Hebrew letters (e.g. the common `בע"מ`/Ltd.-equivalent abbreviation) reliably triggers it — normalize to the correct Hebrew gershayim/geresh marks, which is both a workaround and the typographically correct rendering.
- Do your own text splitting/wrapping and hand the library only single-direction, pre-wrapped chunks — don't rely on its automatic paragraph-level RTL layout or its own line-wrapper for any text that might mix scripts.
- Only retry-with-content-stripped for the specific recognized crash signature (matching on the error's message/stack mentioning the library's internal reordering function names) — any other error is a different bug and would fail identically on retry, so let it propagate immediately instead of masking it behind a fallback.
- Disable automatic hyphenation (`Font.registerHyphenationCallback`) — the default hyphenation logic assumes Latin scripts and mishandles Hebrew.
- This is a genuine upstream library limitation, not a one-off implementation mistake — expect to need an equivalent workaround again if you upgrade the library or hit a new punctuation/script combination it doesn't yet handle.

### Playbook confidence: medium

## Adoption
Used in **1** repo(s) in this marketplace, for server-side generation of tax invoices and quotes
as Hebrew-RTL PDF documents, with a from-scratch bidi-run-splitting layer and a progressive
content-stripping fallback ladder to work around the upstream rendering crash.
