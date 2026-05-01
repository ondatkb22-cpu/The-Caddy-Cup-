# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Includes behavioral guidelines to reduce common LLM coding mistakes. Merge with project-specific instructions as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
## Repository shape

This is a **single-file static web app**. The entire product lives in `index.html` (~3.4K lines). There is no build system, no `package.json`, no dependencies, no tests, and no server. A more user-facing `README.md` exists alongside it (run instructions, tab overview, slate-update format) — keep them consistent when you change behavior.

To run/preview, open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`). There is nothing to install, lint, build, or test — changes are verified by reloading the page.

## What the app does

A mobile-first PGA DFS (DraftKings) lineup optimizer for a specific contest (Caddy Cup 26 / Cadillac Championship at Trump Doral). It loads a fixed slate of players, lets the user filter/sort/lock/exclude, and computes optimized 6-player lineups for cash and GPP modes via Monte Carlo simulation.

## File layout inside `index.html`

The single file is structured in four contiguous regions — know which region you're editing before making changes:

| Lines | Region | Notes |
|------:|--------|-------|
| 8–1606 | `<style>` | Design tokens (`:root` CSS vars at top), mobile-first layout, dark theme. Four media queries: `min-width: 768px` (desktop scale-up, line ~1312), `min-width: 1200px` (wide, ~1384), `max-width: 767px` (mobile — card layout for the player pool, sticky-stack reduction, enlarged touch targets, scroll-fade on tabs/filters, ~1553), `max-width: 360px` (very narrow phones, ~1602) |
| 1611 | `<script type="application/json" id="player-data">` | Slate data — single JSON array of player objects, parsed at startup. The line itself is enormous; do not `cat`/`grep` it without `head` limits |
| 1612–1992 | HTML body | Header, tab bar, `<main>` with one `<div id="tab-*">` per tab, the `.table-wrap` + sibling `.player-cards` container for the pool, floating lineup sheet, cap-breach banner, FAB, toast |
| 1994–3381 | `<script>` | All app logic (vanilla JS, no framework) |

Tabs (data-tab values, in render order): `guide` (default), `pool`, `top-plays`, `field`, `builds`, plus `builder` which opens the lineup sheet instead of switching tabs (it has a count badge `#builder-badge`).

## App architecture (the JS region)

Single global `state` object (line ~1996) holds: current `tab`, `search`/`tier`/`salary`/`form` filters, `sort` direction, `expanded` row, `lineup[6]` array of player names (nulls for empty slots), `Set`s for `locked` and `excluded`, UI flags for the builder sheet (`builderOpen`, `builderCollapsed`), and `fieldThreshold`. There is no framework — every state mutation is followed by manual calls to the relevant `render*` functions.

Key pieces:

- **Players** come from `PLAYERS = JSON.parse(...)` at startup (line ~1994). To update the slate, replace the JSON inside the `<script type="application/json" id="player-data">` tag — that's the single source of truth.
- **Persistence**: `persistState()` / `restoreState()` (line ~2014) round-trip a subset of `state` (`lineup`, `locked`, `excluded`, `sort`, `tier`, `salary`, `form`, `search`) through `localStorage` under key `caddy-cup-26-state-v1`. `tab` and `expanded` are intentionally **not** persisted. Bump the storage-key version when you change the persisted shape.
- **Filtering / sorting**: `filterPlayers()` (line ~2139) reads `state` and returns the displayed subset.
- **Renderers** (`renderPool`, `renderLineupPanel`, `renderTopPlays`, `renderField`, `renderGuide`, `renderBuilds`, `renderStatusBar`, `renderSignatureSpecialists`) each rebuild their tab's DOM from `PLAYERS` + `state`. After mutating state, call the renderers that depend on it (typical trio: `renderPool()`, `renderLineupPanel()`, `renderStatusBar()`), then `persistState()` if the mutation touched a persisted field.
- **Mobile rendering**: `renderPool()` (line ~2388) writes the same filtered list into both the desktop `<table>` (via `renderDetailRow`/inline `<tr>` markup) AND a sibling `<div id="player-cards">` (via `renderPlayerCard`). A media query at `max-width: 767px` toggles which one is `display:none` — no JS branching on viewport, no re-render on resize. The two views share the per-row detail content via `renderCardDetail(p)`, which both `renderDetailRow` and `renderPlayerCard` call. **Cards reuse the same `data-player`, `data-lock`, `data-exclude`, and `class="row-check"` attributes as the table rows** so the body click delegation works for both without per-view branching.
- **Optimizer**: `optimizeLineup(mode)` (single best, line ~2986) and `buildTopLineups(mode, n, minUnique, seedOffset)` (top-N diverse, line ~3118) use recursive backtracking with salary-cap pruning over the top ~30 candidates ranked by `scoreFn`, then re-rank finalists by Monte Carlo win probability. The Builds tab Rebuild button advances `seedOffset` to cycle through alternate variants.
- **Simulation**: `simulateLineup(players, nSims, seed)` (line ~2614) uses Box-Muller on `(p.proj, p.sigma)` per player and counts hits above thresholds. **Pass a seed for reproducibility** — the optimizer always does (`12345 + i` for single-best, `7777 + idx` for multi). `mulberry32(seed)` is the deterministic PRNG (line ~2603).
- **Event handling**: a single delegated click handler on `document.body` (line ~2736) dispatches on `data-lock`, `data-exclude`, `data-unlock`, `data-unexclude`, `data-remove`, `data-load-lineup`, `.row-check`, and player rows. Add new interactive controls by adding a `data-*` attribute and a branch in this handler rather than per-element listeners.
- **UX helpers**: `showToast(msg)` (line ~2045) flashes a 2.2s confirmation in the `#toast` element. `withButtonLoading(btnId, loadingText, work)` (line ~2056) wraps a heavy synchronous action so the button shows a disabled/loading state during optimize runs (uses `setTimeout(0)` so the paint lands before the work starts). `renderLineupPanel()` toggles the `#cap-breach` banner when `salary > 50000`.

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

`SIG_EVENTS` (line ~2199) is a hardcoded mapping of player → signature-event finishes, with `SIG_YEAR_WEIGHTS` (line ~2237) weighting current-season events at 1.0 and prior at 0.6. This drives the "Signature Event Specialists" card on the Top Plays tab via `renderSignatureSpecialists()` (line ~2262).

## CSS conventions

CSS variables in `:root` (line ~16) define spacing, surfaces, borders, text colors, brand/sentiment colors, and tier colors. Use the tokens (`var(--bg-1)`, `var(--accent)`, `var(--tier-sharp)`, etc.) rather than literal colors. The design intentionally avoids decoration — dense data, breathable spacing, "Stokastic/4for4 aesthetic" per the comment at the top of the stylesheet.

## Working in this repo

- Edit `index.html` directly; there is no other source file. Prefer `Edit` over `Write` — the file is large.
- Reading the whole file in one `Read` call exceeds the token limit. Use `offset`/`limit` to target the region you need (see the table above).
- Avoid `grep`/`awk` over the whole file from Bash — line 1611's embedded JSON is enormous and floods stdout. Search with line-number-anchored Read calls or grep with `head` limits.
- When changing optimizer scoring, course-fit logic, or salary constants, update **both** `optimizeLineup` and `buildTopLineups` — the score functions and pruning bounds are duplicated.
- After any state mutation in event handlers, call the renderers manually (typical trio: `renderPool()`, `renderLineupPanel()`, `renderStatusBar()`), and `persistState()` if the mutation touched a persisted field. Add a `showToast(...)` for confirmable user actions to match the rest of the UI.
- If you change the shape of persisted state, bump the `STORAGE_KEY` version (`caddy-cup-26-state-v1` → `-v2`) so older clients don't restore an incompatible payload.

## Git workflow

Active development branch for this task: `claude/add-claude-documentation-46po0`. Commit history alternates between terse `Update index.html` commits (in-app iteration) and short imperative messages describing the user-visible change (e.g. `Rebuild button now cycles through lineup variants`, `CSV export and overlap highlighting on Builds tab`, `Drag-to-dismiss on lineup sheet + tab content fade-in`). Match the imperative style for substantive changes; `Update index.html` is fine for tiny tweaks.
