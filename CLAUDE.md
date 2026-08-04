# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single self-contained HTML file, `kids-wordle.html` — a Wordle clone for kids with grade-leveled word lists (K–1, 2–3, 4–5). There is no build system, no package manager, no dependencies, and no test suite. All markup, CSS, and JavaScript live in that one file (`<style>` in `<head>`, an IIFE `<script>` at the end of `<body>`).

## Working with the code

There is nothing to install or build. To check a change:

```bash
# Open directly in a browser (file:// URL, works fully offline)
open kids-wordle.html          # macOS
xdg-open kids-wordle.html      # Linux

# Or serve it locally (only needed for testing across devices on a LAN)
python3 -m http.server 8000    # then visit http://localhost:8000/kids-wordle.html
```

There's no automated test suite. Verification for past changes has been done ad hoc with Playwright (Chromium at `/opt/pw-browsers/chromium` in this environment) — driving the page with `page.keyboard.type()`/`page.tap()` and asserting on tile classes (`tile.correct`/`present`/`absent`), the hint box's content, and `#overlay`'s `show` class. There's no committed test script; write one temporarily in a scratch location if you need to verify gameplay logic or responsive breakpoints, and check DOM state/computed styles rather than relying on screenshots alone for pass/fail.

## Deployment

The `main` branch is served via GitHub Pages (`https://magooru3.github.io/WFK/kids-wordle.html`). Enabling/reconfiguring Pages itself is a one-time repo Settings change with no API exposed here — it's already on. Merging to `main` is what ships a change; there's no CI/CD step in between.

## Architecture (all inside `kids-wordle.html`)

**Word data.** `WORD_LISTS` is keyed by grade band (`k1`, `g23`, `g45`), each an array of `[WORD, definition, category]` triples. All words are exactly 5 uppercase letters. `category` is a deliberately vague, no-letters-given clue; `definition` is a fuller one-sentence explanation (also shown as the answer explainer after a win). Adding words means appending triples to the right grade's array — keep definitions and categories from literally containing the target word (a `BERRY` definition once said "like a strawberry" and had to be rewritten; there's no automated check for this, verify by eye).

**Game state** is a flat set of module-scoped `let`s at the top of the IIFE (`grade`, `targetWord`, `targetHint`, `targetCategory`, `currentGuess`, `currentRow`, `gameOver`, `hintLevel`, `keyStatus`). `resetGame(newGrade)` is the single reset path — called on grade switch, "New Word", and "Play Again" — and re-picks a word, rebuilds the board/keyboard DOM from scratch (`buildBoard()`/`buildKeyboard()`), and clears hint state.

**Progressive hints.** `hintLevel` (0–3) gates `hintText(level)`: level 1 is the category, level 2 the definition, level 3 the definition plus the first letter as a letter skeleton (`M _ _ _ _`) — only that last tier ever reveals a letter, and only the first one. Hints render cumulatively (each click appends a `<div>` to `#hint-box` rather than replacing it). The hint button's label and disabled state are kept in sync via `updateHintButtonLabel()`.

**Input handling is unified**: physical keydown, on-screen keyboard clicks, and touch taps all funnel through `handleKeyInput(key)`. A load-bearing detail — every interactive button calls `.blur()` on click/tap before running its handler. Without it, a focused button (e.g. a grade selector) natively re-activates on a subsequent physical Enter keydown and silently resets the board mid-guess; this was a real regression, not defensive boilerplate. The global `keydown` listener also calls `preventDefault()` for the same reason.

**Guess evaluation** (`evaluateGuess`) is the standard two-pass Wordle algorithm: mark exact-position matches first, then resolve remaining letters against not-yet-consumed target letters (a `used[]` array), so duplicate letters score correctly. `revealRow` staggers the flip animation per tile via `setTimeout` chains (not CSS animation events) — this matters because `prefers-reduced-motion` disables the flip *animation* but the color/class changes still happen on their own timers, so reduced-motion users still see correct results, just without the flip.

**Sound effects** are synthesized at runtime with the Web Audio API (`beep()` builds oscillator+gain nodes) — there are no audio asset files, by design, to keep the single-file/no-dependencies constraint.

**Responsive/device layout** is handled entirely in CSS via `pointer: coarse` + viewport-size media queries rather than user-agent sniffing (more robust, and iPadOS Safari's UA doesn't reliably identify as iPad anyway):
- `min-width: 700px` + `pointer: coarse` — tablet sizing (bigger board/keyboard/type).
- `pointer: coarse` + `max-height: 500px` + `orientation: landscape` — phone landscape reflows into a two-column **CSS Grid** (info/controls left, board+keyboard stacked right) rather than shrinking a vertical stack to illegible sizes. This grid repurposes `body`'s existing child elements via explicit `grid-column`/`grid-row` placement — there's no wrapper `<div>` for each column, so if you add a top-level element to `body`, give it an explicit grid placement in that media query or it will land in an unintended cell.
- iOS-specific meta tags (`apple-mobile-web-app-*`, `viewport-fit=cover`) and `env(safe-area-inset-*)` padding on `body` handle the notch/Dynamic Island/home indicator.
- Width formulas that mix a fixed max with a viewport-relative unit (`min(500px, calc(100vw - 16px))` for `#keyboard`) must account for `body`'s own horizontal padding explicitly — a plain `98vw` once overflowed the viewport by ~2px on a 320px-wide phone because it didn't subtract that padding.
