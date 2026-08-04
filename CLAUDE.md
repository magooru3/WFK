# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single self-contained HTML file, `kids-wordle.html` — "Wordle with Phoebe", a Wordle clone for kids with grade-leveled word lists (K–1, 2–3, 4–5). There is no build system, no package manager, no dependencies, and no test suite. All markup, CSS, and JavaScript live in that one file (`<style>` in `<head>`, an IIFE `<script>` at the end of `<body>`).

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

**Theming.** Every colour is a CSS custom property on `:root`, redefined under `@media (prefers-color-scheme: dark)` using NYT Wordle's own dark values. Components must read tokens (`var(--surface)`, `var(--text)`) rather than literal hex, or they break in one theme — a hardcoded `color: #fff` on the active grade pill once put white text on a light-grey pill in dark mode.

**Word data comes in two lists with different jobs.**

`ANSWER_WORDS` (~370 words) is the only source the game picks answers from. It is keyed by grade band (`k1`, `g23`, `g45`), each an array of `[WORD, definition, category]` triples, all exactly 5 uppercase letters. `category` is a deliberately vague, no-letters-given clue; `definition` is a fuller one-sentence explanation (also shown as the answer explainer after a win). It is hand-curated — common kid vocabulary (`PIZZA`, `PUPPY`, `SMILE`), no plurals, nothing a child would need a dictionary for. Adding words means appending triples to the right grade's array, and every added word must also exist in `VALID_GUESSES` or the game will reject its own answer. Keep definitions and categories from literally containing the target word or a stem of it (a `BERRY` definition once said "like a strawberry" and had to be rewritten; `IDEAL` once had the category "an idea"); there's no automated check in the repo, so verify by eye or with a throwaway script.

`VALID_GUESSES` (~8,500 words) is used *only* to decide whether a typed guess is a real word. It is built from ENABLE (the public-domain Enhanced North American Benchmark Lexicon), filtered to exactly 5 letters — ENABLE carries no proper nouns or abbreviations to begin with — minus an explicit list of profanity, slurs and sexual terms, plus every `ANSWER_WORDS` entry. Plurals are deliberately *kept* here: `CARTS` is a real word a kid may type and rejecting it would just look broken; they are simply never answers, since the answer list is hand-written. Words like `CHESS` and `DRESS` are ordinary entries — nothing tries to strip trailing S. The list is stored lower-case as one whitespace-separated template-literal blob and upper-cased into a `Set` at load; as an array of quoted string literals the same data costs roughly twice the bytes, and it is already the majority of the file.

**Game state** is a flat set of module-scoped `let`s at the top of the IIFE (`grade`, `targetWord`, `targetHint`, `targetCategory`, `currentGuess`, `currentRow`, `gameOver`, `hintLevel`, `keyStatus`). `resetGame(newGrade)` is the single reset path — called on grade switch, "New Word", and "Play Again" — and re-picks a word, rebuilds the board/keyboard DOM from scratch (`buildBoard()`/`buildKeyboard()`), and clears hint state.

**Progressive hints.** `hintLevel` (0–3) gates `hintText(level)`: level 1 is the category, level 2 the definition, level 3 the definition plus the first letter as a letter skeleton (`M _ _ _ _`) — only that last tier ever reveals a letter, and only the first one. Hints render cumulatively (each click appends a `<div>` to `#hint-box` rather than replacing it).

Hints are rationed against *guesses*, not clicks: `hintAvailable()` requires `HINTS_AFTER` (3) completed guesses before the first one, and then one further guess after each hint (tracked by `lastHintRow`), so the three tiers land at rows 3/4/5 at the earliest and all three are still reachable within six turns. A rejected guess doesn't advance `currentRow`, so it earns no hint progress. `updateHintButtonLabel()` is the single source of truth for both the label (`🔒 Hint after N tries` vs `💡 Hint (N left)`) and `disabled`, so the button can't look clickable while the handler refuses; it's called from `resetGame()` and from the reveal `setTimeout` — not right after `currentRow++`, which happens synchronously while the row is still flipping.

**Input handling is unified**: physical keydown, on-screen keyboard clicks, and touch taps all funnel through `handleKeyInput(key)`. Two load-bearing details there:
- Every interactive button calls `.blur()` on click/tap before running its handler. Without it, a focused button (e.g. a grade selector) natively re-activates on a subsequent physical Enter keydown and silently resets the board mid-guess; this was a real regression, not defensive boilerplate. The global `keydown` listener also calls `preventDefault()` for the same reason.
- `isRevealing` gates all input for the length of the flip (`REVEAL_MS`). `gameOver` is only set when that timer fires, so without the gate a player can keep typing during the ~1s reveal and land extra guesses *after* already winning.

**Audio must be unlocked in a gesture.** Every sound fires from a `setTimeout` during the reveal, which is not a user gesture — so iOS Safari would create the `AudioContext` suspended and the game would be silent on iPhone/iPad. `unlockAudio()` opens the context on the first `pointerdown`/`keydown` and `getCtx()` resumes it if suspended. Don't move context creation back into the sound path.

**Guess validation** happens in `submitGuess` before anything is scored: too-short guesses and words missing from `VALID_GUESSES` both shake the row, show a message, and return early — `currentRow` is not advanced, so a rejected guess costs no turn and the letters stay on the board for the player to edit.

**Guess evaluation** (`evaluateGuess`) is the standard two-pass Wordle algorithm: mark exact-position matches first, then resolve remaining letters against not-yet-consumed target letters (a `used[]` array), so duplicate letters score correctly. `revealRow` staggers the flip animation per tile via `setTimeout` chains (not CSS animation events) — this matters because `prefers-reduced-motion` disables the flip *animation* but the color/class changes still happen on their own timers, so reduced-motion users still see correct results, just without the flip.

**Sound effects** are synthesized at runtime with the Web Audio API (`beep()` builds oscillator+gain nodes) — there are no audio asset files, by design, to keep the single-file/no-dependencies constraint.

**Responsive/device layout** is handled entirely in CSS via `pointer: coarse` + viewport-size media queries rather than user-agent sniffing (more robust, and iPadOS Safari's UA doesn't reliably identify as iPad anyway):
- `min-width: 700px` + `pointer: coarse` — tablet sizing (bigger board/keyboard/type).
- `pointer: coarse` + `max-height: 500px` + `orientation: landscape` — phone landscape reflows into two flex columns (controls left, board+keyboard right) rather than shrinking a vertical stack to illegible sizes. The `.panel` and `.stage` wrappers are `display: contents` by default, so they are invisible to the portrait layout and only become real columns here. An earlier CSS-Grid version of this shared row tracks between the columns, which dragged the sidebar's buttons down to align with the tall board — hence the wrappers.
- The board is width-driven (square tiles), so its width is capped by viewport height as well — `min(340px, 92vw, 38vh)` — otherwise the keyboard is pushed off the bottom on a phone and the kid has to scroll between reading the board and typing. Opening all three hints does still push past the fold on small phones; that is the accepted trade for opt-in help.
- iOS-specific meta tags (`apple-mobile-web-app-*`, `viewport-fit=cover`) and `env(safe-area-inset-*)` padding on `body` handle the notch/Dynamic Island/home indicator.
- Width formulas that mix a fixed max with a viewport-relative unit (`min(500px, calc(100vw - 16px))` for `#keyboard`) must account for `body`'s own horizontal padding explicitly — a plain `98vw` once overflowed the viewport by ~2px on a 320px-wide phone because it didn't subtract that padding.
