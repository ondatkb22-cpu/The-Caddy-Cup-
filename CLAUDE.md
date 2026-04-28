# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **single-file static web app**. The entire product lives in `index.html` (~2.8K lines). There is no build system, no `package.json`, no dependencies, no tests, and no server. The README is intentionally minimal (`caddy cup / Caddy Champ dfs optimizer`).

To run/preview, open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`). There is nothing to install, lint, build, or test — changes are verified by reloading the page.

## What the app does

A mobile-first PGA DFS (DraftKings) lineup optimizer for a specific contest (Caddy Cup 26 / Cadillac Championship at Trump Doral). It loads a fixed slate of players, lets the user filter/sort/lock/exclude, and computes optimized 6-player lineups for cash and GPP modes via Monte Carlo simulation.

## File layout inside `index.html`

The single file is structured in four contiguous regions — know which region you're editing before making changes:

| Lines | Region | Notes |
|------:|--------|-------|
| 8–1308 | `<style>` | Design tokens (`:root` CSS vars at top), mobile-first layout, dark theme |
| 1313 | `<script type="application/json" id="player-data">` | Slate data — single JSON array of player objects, parsed at startup |
| 1314–1689 | HTML body | Header, tab bar, `<main>` with one `<div id="tab-*">` per tab, floating lineup sheet, FAB |
| 1690–2792 | `<script>` | All app logic (vanilla JS, no framework) |

Tabs (data-tab values): `pool`, `top-plays`, `builds`, `field`, `guide`, plus `builder` which opens the lineup sheet instead of switching tabs.

## App architecture (the JS region)

Single global `state` object (line ~1693) holds: current `tab`, `search`/`tier`/`salary`/`form` filters, `sort` direction, `expanded` row, `lineup[6]` array of player names (nulls for empty slots), `Set`s for `locked` and `excluded`, and UI flags for the builder sheet. There is no framework — every state mutation is followed by manual calls to the relevant `render*` functions.

Key pieces:

- **Players** come from `PLAYERS = JSON.parse(...)` at startup. To update the slate, replace the JSON inside the `<script type="application/json" id="player-data">` tag — that's the single source of truth.
- **Filtering / sorting**: `filterPlayers()` reads `state` and returns the displayed subset.
- **Renderers** (`renderPool`, `renderLineupPanel`, `renderTopPlays`, `renderField`, `renderGuide`, `renderBuilds`, `renderStatusBar`, `renderSignatureSpecialists`) each rebuild their tab's DOM from `PLAYERS` + `state`. After mutating state, call the renderers that depend on it (typically `renderPool()`, `renderLineupPanel()`, `renderStatusBar()`).
- **Optimizer**: `optimizeLineup(mode)` (single best lineup) and `buildTopLineups(mode, n, minUnique)` (top-N diverse) use recursive backtracking with salary-cap pruning over the top ~30 candidates ranked by `scoreFn`, then re-rank finalists by Monte Carlo win probability.
- **Simulation**: `simulateLineup(players, nSims, seed)` uses Box-Muller on `(p.proj, p.sigma)` per player and counts hits above thresholds. **Pass a seed for reproducibility** — the optimizer always does (`12345 + i` for single-best, `7777 + idx` for multi). `mulberry32(seed)` is the deterministic PRNG.
- **Event handling**: a single delegated click handler on `document.body` (line ~2300) dispatches on `data-lock`, `data-exclude`, `data-unlock`, `data-unexclude`, `data-remove`, `data-load-lineup`, `.row-check`, and player rows. Add new interactive controls by adding a `data-*` attribute and a branch in this handler rather than per-element listeners.

## Domain constants and conventions

These numbers are baked into the optimizer and simulator — change them in lockstep if you change them at all:

- **Roster**: 6 players, salary cap **$50,000**, min salary floor **$49,200** (industry standard "no more than $800 left on cap"). Fallback floor of **$47,000** if no candidate hits the strict floor.
- **Field thresholds**: cash median **452** DK pts, top-20% **497** DK pts (`T_DU` / `T_T20` in `simulateLineup`, mirrored in `state.fieldThreshold`).
- **Course-fit bonus**: `cfBonus = max(0, (35 - doral_rank) / 5)` — this Doral-specific weighting (distance corr 0.993 with fit) is duplicated in both `optimizeLineup` and `buildTopLineups` score functions; keep them in sync.
- **Score formulas** (also duplicated across the two optimizers):
  - Cash: `proj + 0.5 * valuePerK + cfBonus`
  - GPP: `0.4 * proj + 0.4 * ceiling + 0.6 * leverage − 0.2 * own_pct + 0.3 * valuePerK + cfBonus`
- **Tiers**: `SHARP` / `CONTRARIAN` / `CHALK` / `FADE` / `NEUTRAL` (via `tout_tier` field). `NEUTRAL` renders as an em dash, not a badge.
- **Lineup array invariant**: always length 6, with `null` for empty slots. Use `padLineup(arr)` to normalize after any mutation that could break this.

`SIG_EVENTS` (line ~1821) is a hardcoded mapping of player → signature-event finishes, with `SIG_YEAR_WEIGHTS` weighting current season at 1.0 and prior at 0.6. This drives the "Signature Event Specialists" card on the Top Plays tab.

## CSS conventions

CSS variables in `:root` (line ~16) define spacing, surfaces, borders, text colors, brand/sentiment colors, and tier colors. Use the tokens (`var(--bg-1)`, `var(--accent)`, `var(--tier-sharp)`, etc.) rather than literal colors. The design intentionally avoids decoration — dense data, breathable spacing, "Stokastic/4for4 aesthetic" per the comment at the top of the stylesheet.

## Working in this repo

- Edit `index.html` directly; there is no other source file. Prefer `Edit` over `Write` — the file is large.
- Reading the whole file in one `Read` call exceeds the token limit. Use `offset`/`limit` to target the region you need (see the table above).
- Avoid `grep`/`awk` over the whole file from Bash — line 1313's embedded JSON is enormous and floods stdout. Search with line-number-anchored Read calls or grep with `head` limits.
- When changing optimizer scoring, course-fit logic, or salary constants, update **both** `optimizeLineup` and `buildTopLineups` — the score functions and pruning bounds are duplicated.
- After any state mutation in event handlers, call the renderers manually (typical trio: `renderPool()`, `renderLineupPanel()`, `renderStatusBar()`).

## Git workflow

Active development branch for this task: `claude/add-claude-documentation-mFsvH`. Commit history is a long series of `Update index.html` commits — single-file iteration is the norm here.
