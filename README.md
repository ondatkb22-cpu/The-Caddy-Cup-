# Caddy Cup 26 — DFS Lineup Optimizer

A single-page DraftKings lineup optimizer for the **Cadillac Championship at Trump Doral** (Caddy Cup 26 / signature event, no cut, $20M purse). Mobile-first, dark-themed, no backend.

## Run it

This is a static HTML file. Two ways to open it:

- **Direct**: double-click `index.html` and your browser opens it.
- **Local server** (recommended on iOS/Safari to avoid `file://` quirks):
  ```bash
  python3 -m http.server 8000
  ```
  …then visit `http://localhost:8000`.

There is no build step, no package manager, no install. Edit `index.html` and reload to see changes.

## What's inside

Six tabs at the top:

| Tab | What it does |
|---|---|
| **Pool** | Sortable / filterable list of all priced players. Lock, fade, or add to lineup with a tap. Tap a row to expand SG breakdown, course-fit decomposition, recent form, and intel. |
| **Top Plays** | Top 6 by Doral course fit (filtering out poor-form players), plus a "Signature Event Specialists" panel that highlights players with elite finishes at no-cut signature events (2026 Pebble + RBC, weighted 1.0; 2025 RBC + Truist, weighted 0.6). |
| **Builds** | Three optimal cash lineups + three GPP lineups, generated automatically. Each has a "Load to Builder" button to drop it into My Lineup. |
| **Field** | Field intel: missing top-15 OWGR, past Doral winners in field, injury watch, course profile, late updates (WDs, weather, sharp consensus). |
| **Guide** | Methodology — how scores are computed, how cash vs GPP differ, column legend. |
| **My Lineup** | The lineup you're building. Salary used / left, simulated Cash %, Top 20 %, mean, ceiling. Optimize buttons for cash and GPP. |

## How the optimizer works

- **Roster**: 6 players, **$50,000 salary cap**, **$49,200 minimum** (industry standard — leaves no more than $800 on the cap).
- **Two modes**:
  - **Cash**: maximize `cash_proj + 0.5 × valuePerK + cfBonus`. Optimized for hitting the field median (~452 DK pts).
  - **GPP**: maximize `0.4 × proj + 0.4 × ceiling + 0.6 × leverage − 0.2 × own_pct + 0.3 × valuePerK + cfBonus`. Optimized for top-20% (~497 DK pts) with low-ownership leverage.
- **Course-fit bonus** (`cfBonus`): `max(0, (35 − doral_rank) / 5)`. Distance correlates 0.993 with course fit at Doral, so high doral_rank players get up to a +7 score boost.
- **Search**: recursive backtracking over the top ~30 candidates by composite score, then re-rank finalists by Monte Carlo win probability (Box-Muller on `(proj, sigma)`, 4,000 sims per candidate, deterministic seed for reproducible results).

## Updating the slate

All player data lives in one place — the JSON array inside this `<script>` tag in `index.html`:

```html
<script type="application/json" id="player-data">[ … ]</script>
```

To update for a new event or fresh DK pricing, replace that JSON array. Each player record has the shape:

```json
{
  "name": "Player Name",
  "salary": 9800,
  "cash_proj": 88.0,
  "ceiling": 105.0,
  "sigma": 16.7,
  "doral_rank": 31,
  "dist_rank": 43,
  "leverage": 39.8,
  "own_pct": 20.1,
  "tout_tier": "SHARP" | "CONTRARIAN" | "CHALK" | "FADE" | "NEUTRAL",
  "form_grade": "good" | "ok" | "poor" | "mixed" | "unknown",
  "equity_tier": 1-5,
  "news_note": "Any news / WD / injury text",
  ...
}
```

Set `salary: null` to mark a player as withdrawn — the filter and optimizer will skip them automatically.

## Editing the code

Everything lives in `index.html` (~3K lines):

| Lines | Region |
|---:|---|
| 8–1486 | `<style>` — design tokens, layout, theming |
| 1491 | embedded JSON player data |
| 1492–1893 | HTML body — header, tabs, tab content, lineup sheet, FAB |
| 1894+ | `<script>` — vanilla JS, no framework |

See `CLAUDE.md` for architectural details, conventions, and gotchas before making non-trivial changes.

## Stack

Vanilla HTML, CSS, and JavaScript. No build tools, no dependencies, no tests, no server. The whole product is one file.
