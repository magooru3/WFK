# Game Overview: Wordle with Phoebe

## 1. What the game does

**Genre:** Word-guessing puzzle game, a Wordle clone aimed at kids.

**Core gameplay loop:**
- Player picks a grade band (K–1, 2–3, or 4–5), which determines the pool of possible answer words.
- Player has 6 attempts to guess a hidden 5-letter word, entering guesses via a keyboard (physical or on-screen).
- Each guess is scored letter-by-letter: correct position (green), present but wrong position (yellow), absent (grey) — standard Wordle rules, including correct handling of duplicate letters.
- A rejected guess (too short, or not a real word) doesn't cost a turn.
- After enough guesses without a win, the player can request progressive hints (category → definition → definition + first letter).
- On win or loss, a modal shows the result, a score (tries + hints used), an explanation of the answer, and — on a win — a "Share with Grandma" button that builds an emoji-grid recap and hands it to the OS share sheet (or falls back to clipboard/SMS link).

**Key mechanics and features:**
- Grade-leveled word lists (`k1`, `g23`, `g45`) so difficulty/vocabulary matches the player's age.
- Progressive, guess-gated hint system (hints unlock only after a set number of guesses, not on demand).
- Dictionary validation against a ~8,500-word list, separate from the curated ~370-word answer list.
- Synthesized sound effects via the Web Audio API (no audio asset files).
- Light/dark theming driven entirely by CSS custom properties and `prefers-color-scheme`.
- Responsive layout for phone portrait, phone landscape, and tablet, using CSS media queries (`pointer: coarse`, viewport size) rather than user-agent sniffing.
- iOS-specific handling: audio unlock on first user gesture, safe-area insets for notch/home indicator, `navigator.share` integration timed to stay inside the click gesture.
- Reduced-motion support: flip animations are skipped but scoring/color changes still happen on their own timers.

**Tech stack:**
- Plain HTML, CSS, and vanilla JavaScript (an IIFE) — no framework, no build tooling, no package manager, no external JS libraries.
- Word list data (`VALID_GUESSES`) is derived from ENABLE (the public-domain Enhanced North American Benchmark Lexicon), embedded directly in the file.

**Scope:**
- Single-player only, single device — no multiplayer, no accounts, no networking beyond loading the static page itself.
- Targets browsers generally, with explicit extra attention to iOS Safari (audio unlock, share sheet, safe-area insets) and tablet/phone responsive layouts.

## 2. Architecture

**High-level structure:** The entire game is one self-contained file, `kids-wordle.html`, at the repo root. There are no folders, modules, or separate source files.

- `<head>`: meta tags (viewport, iOS web-app tags, theme-color) and a single `<style>` block containing all CSS (tokens, layout, responsive breakpoints).
- `<body>`: the game markup (board, keyboard, controls, hint box, modal) plus one `<script>` block at the end containing all game logic, wrapped in an IIFE.

**Entry point:** `kids-wordle.html` itself — there is no separate JS/CSS entry file to build or bundle.

Inside the script, notable internal structure (not separate files, just organization within the IIFE):
- Module-scoped `let` state: `grade`, `targetWord`, `targetHint`, `targetCategory`, `currentGuess`, `currentRow`, `gameOver`, `hintLevel`, `keyStatus`.
- `resetGame(newGrade)` — single reset path for grade switch, "New Word", and "Play Again".
- `buildBoard()` / `buildKeyboard()` — DOM construction.
- `evaluateGuess()` / `revealRow()` — guess scoring and animated reveal.
- `handleKeyInput()` — unified entry point for physical keydown, on-screen keyboard clicks, and touch taps.
- `hintText()` / `hintAvailable()` / `updateHintButtonLabel()` — hint gating and rendering.
- `showEndModal()` — win/loss modal, scoring, and sharing.
- `beep()` / `getCtx()` / `unlockAudio()` — Web Audio sound synthesis.

**External services:** None. The game makes no network calls, has no API, no database, and no authentication. All data (word lists) is embedded in the file at load time.

## 3. Hosting requirements

- **Server type/runtime:** None required beyond a static file host. The file is plain HTML/CSS/JS served as-is — no server-side runtime, no interpreter version, no process to run.
- **Ports:** N/A — nothing listens on a port; it's a static asset.
- **Environment variables / config:** None. There is no config file of any kind in the repo (no `.env`, no config JSON, no framework config).
- **Database or storage:** None. No database, no server-side storage. (The game does not appear to persist any state across sessions — no `localStorage` usage is described in the architecture notes or implied by the code structure covered.)
- **Build steps / build output:** None. There is no package manager manifest (`package.json`), no bundler config, no build script, and no CI/CD config in the repo. The file is deployed as-is.
- **Third-party services or API keys:** None.
- **Resource needs:** Effectively zero. Fully static, stateless, single file (~120 KB). No websockets, no persistent storage, no server compute — it can be served by any static file host or CDN.

**Actual deployment (per `CLAUDE.md`):** The `main` branch is served via GitHub Pages at `https://magooru3.github.io/WFK/kids-wordle.html`. Enabling Pages was a one-time repo Settings change (already done, not represented in repo config). Merging to `main` is the only "deploy" step — there is no CI/CD pipeline in between.

## 4. Local dev setup

No install or build step is needed — there are no dependencies to install.

**Prerequisites:** A web browser. That's it.

**To run:**
```bash
# Open directly as a file:// URL — works fully offline
open kids-wordle.html          # macOS
xdg-open kids-wordle.html      # Linux
```

Or serve it locally (only needed to test across multiple devices on a LAN):
```bash
python3 -m http.server 8000
# then visit http://localhost:8000/kids-wordle.html
```

**Testing:** There is no committed automated test suite. Per `CLAUDE.md`, past verification has been done ad hoc with Playwright (driving the page with `page.keyboard.type()` / `page.tap()` and asserting on tile classes, hint box content, and the modal's `show` class). Any test script should be written temporarily in a scratch location rather than committed.
