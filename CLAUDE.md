# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A self-contained **Fantasy Premier League Hall of Fame** for the Pundits Group Invitational — a friends' FPL league running since 2015/16. The entire app is one portable HTML file with no build step, no server, no dependencies (fonts degrade gracefully offline).

## No build system

There are no commands to run. Open `index.html` directly in a browser. Validate logic changes by porting critical JS to Python via Bash.

The site is published via GitHub Pages at `https://ninosekeleni.github.io/invitational-fpl/` — that canonical URL is what gets shared. Commit and push to `main` to deploy.

## File structure

| File | Purpose |
|------|---------|
| `index.html` | The entire app (self-contained) |
| `og-maker.html` | Browser-run asset generator — open once to download og-image.png, icon-512.png, icon-192.png |
| `manifest.webmanifest` | PWA manifest referencing icon-192.png and icon-512.png |
| `og-image.png`, `icon-192.png`, `icon-512.png` | Generated from og-maker.html, must sit next to index.html |
| `Club Badges/` | Source PNGs — not used at runtime (logos are base64-embedded in `LOGOS`) |
| `FPL Gameweek History/` | Source screenshots used to transcribe player season data |

## Architecture: single-file 5-tab SPA

All logic, styles, data, and base64 assets live inside `index.html`. Tab routing uses URL hashes and the `TABS` array:

```js
const TABS = ['home','hall','champions','rules','register'];
function showTab(name) { ... history.replaceState(null,'','#'+name); }
```

Tabs: **Dashboard** (`#home`) · **Hall of Fame** (`#hall`) · **Champions** (`#champions`) · **Rules** (`#rules`) · **Register** (`#register`)

## Key data blocks (all near the top of the `<script>`)

### `PLAYERS` — FPL history
Each player object: `{name, full, club, s}` where `s` is `{"YYYY/YY": [points, worldRank]}`.
- `name` is the short key used everywhere internally
- `full` is the display name shown on the page
- `club` must match a key in `LOGOS` and `CLUBCODE`

### `DEBUTS` — Invitational debut season
Separate from FPL data history. Used to scope all Invitational stats — only seasons from `max(A.debut, B.debut)` onwards count as shared Invitational seasons.

```js
const DEBUTS = { "Nino":"2015/16", "Azi":"2022/23", ... };
const FOUNDER = "Nino"; // drives "Founder & Commissioner" trophy in profile modals
```
After setting `DEBUTS`, the code attaches `p.debut` and `p.founder` to each player object.

### `CHAMPIONS` — Classic League Roll of Honour
11 seasons of winners. Array of `["YYYY/YY", "PlayerName"]` pairs. Names must match `p.name`.

### `H2H_CHAMPIONS` — H2H League Roll of Honour
FPL Head-to-Head League winners, run concurrently with the Classic League. Array of `["YYYY/YY", name|null, h2hPoints]`. Use `null` for seasons not contested (e.g. 2025/26). Names must match `p.name`.

### `XII` — Registration config
```js
const XII = {
  season:"2026/27", entryFee:350, targetEntrants:30,
  deadline:"2026-12-01",      // YYYY-MM-DD, drives countdown
  whatsapp:"27817482282",     // digits only, no +
  group:"https://chat.whatsapp.com/...",  // group invite link
  bank:"",                    // paste EFT details; "" shows "coming soon"
  pot:[...]
};
const REGISTERED = [];        // add {name, team, paid:bool} entries here
```

### `LOGOS` — base64 club crests
Six clubs: `Arsenal`, `Chelsea`, `Leeds United`, `Liverpool`, `Man City`, `Man Utd`. Resize source PNGs to 160px via `sips -Z 160`, encode via Python `base64.b64encode`, embed as `data:image/png;base64,...`.

## Leaderboard & filter mode

`statsFor(player, seasons)` computes `{tot, cnt, avg, best, peakP}` for any season subset.

The global `MODE` variable drives both the leaderboard and the Head-to-Head comparator simultaneously. Three filter tabs:
- `'this'` — 25/26 Season only
- `'inv'` — The Invitational (2015/16+, default, debut-aware)
- `'all'` — All-Time (full FPL history from 2011/12)

Per-season picker drops a single season string into `MODE`. `windowSeasons(mode)` converts `MODE` to the relevant season array. `renderAll()` re-renders the leaderboard and H2H comparator together when the filter changes.

**Debut-awareness is load-bearing.** The Invitational leaderboard, dashboard teaser, biggest movers, and H2H comparator all gate on `p.debut` — a late joiner cannot accumulate pre-league seasons toward their Invitational total. The All-Time view uses the full FPL dataset by design.

## Head-to-Head comparator

`renderH2H()` builds the comparator in the Hall of Fame tab. It follows the active `MODE` via `h2hSeasons(A, B)`:
- `'inv'` → `sharedSeasons(A, B)` (debut-aware overlap)
- `'all'` / single-season → `windowSeasons(MODE).filter(s=>A.s[s]&&B.s[s])`

`sharedSeasons(A, B)` starts from `max(A.debut, B.debut)` and only includes seasons both players have scores for — this is the canonical function for any Invitational-scoped comparison.

**Single-season mode** returns early with a focused output (margin, both scores/ranks, verdict) — streak/momentum/superiority cards are deliberately suppressed.

**Superiority Rating** (multi-season only) is zero-sum around 5.5:
```
5.5 + (winRate−0.5)×6 + (pointsShare−0.5)×8 + rankEdge(±0.4)   clamped 1–10
```
The two managers always contrast (e.g. 7.9 vs 3.1).

Dropdowns are alphabetical by `p.full`. `fillH2HSelects()` populates them.

## Champions tab

Two stacked sections rendered by separate functions:
- `renderChampions()` → Classic League roll of honour (`#hofstats`, `#rollwrap`)
- `renderH2HChampions()` → H2H League roll of honour (`#h2hhofstats`, `#h2hrollwrap`)

Click handlers must be scoped to their own container (`#rollwrap .rollrow[data-nm]` vs `#h2hrollwrap .rollrow[data-nm]`) to avoid cross-table conflicts.

## Trophy cabinet

`playerTrophies(p)` auto-awards badges shown in player profile modals: Founder & Commissioner, Founding Member (debut==="2015/16" and not founder), Classic League Champion, H2H League Champion, all-time superlatives, FPL rank milestones. The Trophy Room section on the Hall of Fame tab (`renderTrophyRoom`) is a separate grouped display — not everything in `playerTrophies` appears there.

## OG / social meta

Lines 15–24 of `<head>` contain `og:url`, `og:image`, and `twitter:image` pointing to `https://ninosekeleni.github.io/invitational-fpl/`. Keep these in sync if the Pages URL ever changes.

## Adding a new player

1. Add to `PLAYERS` with their FPL season data (transcribed from `FPL Gameweek History/` screenshots)
2. Add to `DEBUTS` with their first Invitational season
3. Add their club crest to `LOGOS` if the club isn't already present
4. Commit and push to deploy

## Keeping data current

Bump the `UPDATED` constant (near `const CURRENT`) when you refresh the season data — it drives the "Last updated" line in the footer.

## Validating logic without a server

The sandbox blocks `python3 -m http.server`. Validate by:
- Extracting JS logic and porting to Python via Bash (regex-parse `PLAYERS`/`DEBUTS`/`ALLSEASONS` from the HTML, recompute key outputs)
- Bracket-balance check: strip string literals/comments, count `{([` vs `})]` — catches gross syntax errors
