# Copilot instructions — Space Quiz

## Running and checking

- There is no build, package manager, test runner, or linter. The whole app is `index.html` (inline `<style>` and `<script>`), opened directly via `file://`. Do not add a server, bundler, or external dependencies (including CDN fonts/libraries).
- Verify changes by opening `file:///…/space-quiz/index.html` in the integrated browser and reloading. Smoke tests are done by running JavaScript in the page (e.g. click `.option[data-i="N"]`, then `#next`, and assert classes, `#score` text, focus, and `.results` content). To check a single question, reload and act on just the first `.option` set.
- Correct answer indices for the current question set, in order: `[1, 2, 1, 3, 2, 0, 3, 0, 1, 1]` — useful for scripted perfect runs. Update this list if `QUESTIONS` changes.
- A Playwright MCP server is configured in `.mcp.json` (requires Node/`npx`; uses installed Chrome, with `--allow-unrestricted-file-access` so it can load the `file://` URL). Use it for repeatable checks such as real Tab/Enter keyboard flows and light/dark rendering (`browser_navigate`, `browser_snapshot`, `browser_press_key`). It writes snapshots to `.playwright-mcp/`; delete that folder when done.

## Architecture

- State is two module-level variables, `index` and `score`. The single `#card` element is fully re-rendered via `innerHTML` template strings:
  - `start(moveFocus)` → resets state → `renderQuestion(moveFocus)`
  - `choose(i)` mutates the current question's DOM in place (classes, feedback, score, progress) — it does **not** re-render.
  - `next()` → `renderQuestion()` or `renderResults()` after the last question.
- Because the card is re-rendered, event listeners are reattached inside each render function; element IDs (`#score`, `#fill`, `#progress`, `#feedback`, `#next`, `#question`, `#results-title`, `#restart`) are recreated every render.
- Questions live in the `QUESTIONS` array as `{ q, a: [4 options], c: correctIndex }`. Question count is derived from `QUESTIONS.length` everywhere (progress, score readout, results), so don't hardcode `10`.
- Results tiers (emoji/title/message) are chosen by score ratio in `renderResults()`.
- The card entrance animation is restarted on each render with `card.style.animation = "none"; card.offsetHeight; card.style.animation = ""`.

## Conventions

- **Theming:** all colours come from CSS custom properties in `:root`, overridden in `@media (prefers-color-scheme: dark)`. Use these variables (`--accent`, `--accent-2`, `--good`, `--bad`, `--muted`, `--border`, `--card`, etc.) and `color-mix()` for tints instead of literal colours, so both themes stay correct.
- **Typography:** system sans-serif stack, 15px body (keep within 14–16px). Monospace (`--mono`, defined on `.hud`) is reserved for the "mission-control" header HUD.
- **Feedback states:** `.option.correct` (green + `pop` animation), `.option.wrong` (red + `shake` animation), `.option.dim` for unchosen options; `.feedback.good` / `.feedback.bad` for the message. On a wrong answer the correct option is also marked `.correct`.
- **Accessibility (keep these intact when editing):**
  - Answer choices are native `<button>`s and are disabled after choosing.
  - Focus management: no focus on initial load; after answering, focus moves to `#next`; on a new question, to the `#question` heading (`tabindex="-1"`); on results, to `#results-title`; after restart, back to `#question`.
  - `#progress` is a `role="progressbar"` whose `aria-valuenow` is updated in `choose()`, not just the visual width.
  - The score is a `role="status"` live region; `scoreMarkup()` renders a visually hidden "N out of M" for screen readers plus an `aria-hidden` zero-padded display (`01/10`). Update the score through `scoreMarkup()`, not raw text.
  - Decorative emoji/LEDs are `aria-hidden`; `@media (prefers-reduced-motion: reduce)` disables all animations and transitions.
- Layout is a single narrow centered column (`.app { max-width: 440px }`).
