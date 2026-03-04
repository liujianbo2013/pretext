## Text Metrics

DOM-free text measurement using canvas `measureText()` + `Intl.Segmenter`. Two-phase: `prepare()` once, `layout()` is pure arithmetic on resize. ~0.1ms for 500 comments. Full i18n.

### Commands

- `bun start` — serve pages at http://localhost:3000
- `bun run check` — typecheck + lint
- `bun test` — headless tests (HarfBuzz, 100% accuracy)

### Files

- `src/layout.ts` — the library
- `src/measure-harfbuzz.ts` — HarfBuzz backend for headless tests
- `src/test-data.ts` — shared test texts/params
- `src/layout.test.ts` — bun tests: consistency + word-sum vs full-line accuracy
- `pages/accuracy.html + .ts` — sweep across fonts, sizes, widths, i18n texts (working)
- `pages/emoji-test.html` — canvas vs DOM emoji width comparison (working)
- `pages/demo.html + .ts` — visual side-by-side comparison (TODO)
- `pages/benchmark.html + .ts` — performance comparison (TODO)
- `pages/interleaving.html + .ts` — realistic DOM interleaving demo (TODO)

### Key decisions

- Canvas over DOM: no read/write interleaving. Zero DOM reads in prepare() or layout().
- Intl.Segmenter over split(' '): handles CJK (per-character breaks), Thai, all scripts.
- Punctuation merged into preceding word-like segments: "better." measured as one unit. Reduces accumulation error (up to 2.6px at 28px without merging). Only merges into word-like preceding segments (not spaces — that would hide content from line-breaking).
- Trailing whitespace hangs past line edge (CSS behavior): spaces that overflow don't trigger breaks.
- Emoji correction: auto-detected per font size. Canvas inflates emoji widths on Chrome/Firefox at <24px; correction is constant per emoji grapheme, font-independent. Safari is unaffected (correction=0).
- Non-word, non-space segments (emoji, parens) are break points: CSS breaks at the preceding space, so these overflow like words, not like trailing whitespace.
- Kinsoku shori (禁則処理): CJK punctuation (，。「」etc.) merged with adjacent graphemes during CJK splitting so they can't be separated across line breaks.
- HarfBuzz with explicit LTR for headless tests: guessSegmentProperties assigns wrong direction to isolated Arabic words.

### Accuracy

- Chrome: 99.9% (3837/3840). Remaining: 2 Georgia measurement rounding edge cases, 1 bidi boundary break.
- Safari: 98.8% (3792/3840). Remaining: CSS line-breaking rule differences — emoji break opportunities, CJK kinsoku, bidi boundary breaks. NOT measurement errors.
- Firefox: similar emoji issue to Chrome but auto-corrected (+5px at 15px, converges at 28px).
- Headless (HarfBuzz): 100% (1472/1472). Algorithm is exact.

### Known limitations

- system-ui font: canvas and DOM resolve to different optical variants on macOS. Use named fonts.
- Safari CSS rules: emoji-as-break-point, bidi boundary breaks differ from our algorithm. Kinsoku is now implemented.
- Server-side: needs canvas or @napi-rs/canvas with registered fonts. HarfBuzz works for testing.

### Related

- `../text-layout/` — Sebastian Markbage's original prototype + our experimental variants (a-e) and the five-way benchmark comparing Sebastian's, ours, DOM batch, DOM interleaved, and the precise approach.

See [RESEARCH.md](RESEARCH.md) for full exploration log with measurements.

Based on Sebastian Markbage's [text-layout](https://github.com/reactjs/text-layout).
