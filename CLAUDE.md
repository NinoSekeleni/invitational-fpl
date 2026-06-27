# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A self-contained **Fantasy Premier League Hall of Fame** for the Pundits Group Invitational — a friends' FPL league running since 2015/16. The entire app is one portable HTML file with no build step, no server, no dependencies (fonts degrade gracefully offline).

## No build system

There are no commands to run. Open `index.html` directly in a browser. Validate logic changes by porting critical JS to Python via Bash.

The site is published via GitHub Pages at `https://ninosekeleni.github.io/invitational-fpl/` — that canonical URL is what gets shared. Commit and push to `main` to deploy. (There used to be a duplicate `Invitational Hall of Fame.html` kept in sync by hand; it was removed in favour of the single canonical Pages URL.)

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
Separate from FPL data history. Used to compute rivalry windows — only seasons from `max(A.debut, B.debut)` onwards count as shared Invitational seasons.

```js
const DEBUTS = { "Nino":"2015/16", "Azi":"2022/23", ... };
const FOUNDER = "Nino"; // drives "Founder & Commissioner" trophy
```
After setting `DEBUTS`, the code attaches `p.debut` and `p.founder` to each player object.

### `CHAMPIONS` — Roll of Honour
11 seasons of winners. Array of `["YYYY/YY", "PlayerName"]` pairs. Names must match `p.name`.

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

## Rivalry system

`sharedSeasons(A, B)` is the critical function — it determines valid rivalry overlap. It starts from `max(A.debut, B.debut)` and only includes seasons where both players have a recorded score. Any change to rivalry logic must go through this function, not raw FPL data overlap.

Rivalry scoring formula (in `detectRivalries`):
```
score = closeness×175 + leadChanges×15 + balance×16 + sharedSeasons×0.9 + (familyBonus?24:0)
```
In `playerTopRival`, the family bonus is reduced to 6 so closeness dominates for per-player "fiercest rival" display.

**7 distinct rivalry copy templates** (`RIV_TEMPLATES`) + **2 family templates** (`FAM_TEMPLATES`) — no two rivalries should share the same opener, body, or closing line. The `rivBits(r)` helper extracts all needed variables.

## Leaderboard

`statsFor(player, seasons)` computes `{tot, cnt, avg, best, peakP}` for any season subset. Leaderboard filters: This Season · Past 5 · Past Decade · All-time · Invitational era (2015/16, default) · per-season picker. Sort toggle: Total Points vs Avg/Season.

## Trophy cabinet

`playerTrophies(p)` auto-awards badges: Founder, Founding Member (debut==="2015/16" only), Champion, all-time superlatives, FPL rank milestones. Displayed in player profile modals.

## OG / social meta

Lines 15–24 of `<head>` contain `og:url`, `og:image`, and `twitter:image` pointing to `https://ninosekeleni.github.io/invitational-fpl/`. Keep these in sync if the Pages URL ever changes.

## Adding a new player

1. Add to `PLAYERS` with their FPL season data (transcribed from `FPL Gameweek History/` screenshots)
2. Add to `DEBUTS` with their first Invitational season
3. Add their club crest to `LOGOS` if the club isn't already present
4. Commit and push to deploy

## Keeping data current

Bump the `UPDATED` constant (near `const CURRENT`) when you refresh the season data — it drives the "Last updated" line in the footer.
