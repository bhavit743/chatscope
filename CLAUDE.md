# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

No build step. Open `index.html` directly in a browser:

```bash
# Quick local server (any of these work)
python3 -m http.server 8080
npx serve .
```

There are no tests, no linter, and no package.json. All dependencies (Chart.js 4.4.1, html2canvas 1.4.1, jsPDF 2.5.1) are loaded from CDN.

## Architecture

The entire application is a single file: `index.html`. It is structured in three blocks:

1. **CSS** (lines 12–475) — CSS custom properties at `:root`, then component styles (upload zone, stat grid, person cards, chart cards, chemistry cards, word cloud, share modal).

2. **HTML** (lines 477–624) — Static layout: header, hero, upload section (dropzone + textarea), and a hidden `#results` div that gets populated by JS after analysis.

3. **JavaScript** (lines 626–1999) — Self-contained, no modules. Key engines:

   - **Foul word detection** (`FOUL_ROOTS`, `normalise()`, `isFoulWord()`, `countFoulInMsg()`) — canonical root set with a fuzzy normaliser that maps censored forms and phonetic variants (e.g. `bnchd`, `f**k`) to canonical roots before lookup. Also does bigram matching for two-word phrases.

   - **Filler detection** (`FILLER_SET`, `isFiller()`) — classifies short messages (≤5 tokens) as low-content filler replies.

   - **Parser** (`parseChat()`) — handles two WhatsApp export formats: iOS bracket format `[DD/MM/YY, H:MM:SS AM] Name: text` and Android dash format `DD/MM/YYYY, HH:MM am - Name: text`. Joins continuation lines, strips system messages and U+200E marks.

   - **Analysis entry point** (`analyse()`) — reads the textarea, calls `parseChat()`, builds a per-sender `stats` object (messages, words, emojis, foul, media, filler, chars), then delegates to `renderResults()`.

   - **Word frequency engine** (`STOP_WORDS`, `tokenize()`, `topWords()`, `computeWordFreq()`, `renderWordCloud()`) — strips URLs/mentions, tokenises, removes stop words (English + Hinglish) and dynamically-built sender name tokens, ranks by frequency.

   - **Chemistry engine** (`computeChemistry()`, `renderChemistry()`) — scores every sender pair by reply volume, average reply gap, quick replies (<2 min), and emoji warmth. Renders podium cards and an N×N reply-affinity heatmap.

   - **Share card** (`buildShareCanvas()`, `generateShareCard()`, `downloadCard()`) — draws a 600px×variable Canvas at 2× DPR, exported as PNG.

   - **PDF export** (`generatePDF()`) — multi-page jsPDF report covering overview, per-person table, chemistry pairs, reply matrix, and word clouds. Uses `arraybuffer` + manual Blob URL to avoid CSP issues with `doc.save()`.

## Key Conventions

- All global state is in module-level `let` vars: `currentStats`, `currentTotal`, `currentMsgs`, `currentWordData`, `charts[]`, `shareCanvas`.
- `destroyCharts()` must be called before re-rendering to prevent Chart.js canvas reuse errors.
- The `COLORS` / `COLORS_W` arrays are aligned by index to participants sorted by message count.
- `shortName(n)` strips ` IITB` / ` IIT` suffixes and takes the first word — used in all display contexts.
- `parseTime()` inside `computeChemistry` handles both 12h AM/PM and 24h formats; returns `0` on failure and is guarded by `gap > 0`.
