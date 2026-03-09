# D401 Flong Scoreboard — Claude Code Context

## What this is
A single-file HTML dashboard for tracking a drinking game called **D401 Flong**.

## Tech stack
- Single file: `flong-dashboard.html`
- React 18 UMD + Babel standalone (loaded from cdnjs CDN)
- No `export default` — breaks UMD React
- Tailwind is NOT used; all styles are inline JSX or a small `<style>` block at the top
- localStorage key: `"d401_flong_v6"` (bump version to flush stale data when schema changes)
- Never use `localStorage`/`sessionStorage` inside artifacts — but this is a plain HTML file so it's fine here

## Game rules
- 4 players, one at each corner of a diamond table
- Teammates diagonal, opponents adjacent
- Each player starts with 5 cups (10 per team)
- Throw ball into **partner's** cup; if made, partner slides it to an adjacent opponent
- Opponent must drink and flip to eliminate the cup
- **Simul**: both teams land simultaneously → both must drink/flip before resuming
- **Rerack**: unlimited, any time between shots
- **Naked Lap**: loser player with 0 cups hit → auto-tagged
- **Puskas**: hit 5 in a row to start → manually toggled
- First team to eliminate all 10 opposing cups wins

## Players
```
p1=David, p2=Jack, p3=Jordan, p4=Karthik, p5=Richard, p6=Yishak
```
`PIDS = ["p1","p2","p3","p4","p5","p6"]` — always use these keys, never names directly.

## Data schema
```json
{
  "players": {"p1":"David","p2":"Jack",...},
  "games": [
    {
      "id": 1741000001,
      "teamA": ["p1","p2"],
      "teamB": ["p3","p4"],
      "cups": {"p1":5,"p2":5,"p3":4,"p4":5,"p5":0,"p6":0},
      "puskas": {"p1":false,...},
      "nakedLap": {"p1":false,...},
      "winner": "A"
    }
  ],
  "recaps": {"<gameId>": {"text":"...", "ts":1234567890}}
}
```

## Seed data (5 games baked in)
| G | Team A | Team B | Winner | Cups |
|---|--------|--------|--------|------|
| 1 | David+Jack | Jordan+Karthik | David+Jack | D:5,J:5,Jo:4,K:5 |
| 2 | David+Jack | Richard+Yishak | David+Jack | D:5,J:5,R:3,Y:4 |
| 3 | Richard+Yishak | Jordan+Karthik | Richard+Yishak | R:5,Y:5,Jo:3,K:4 |
| 4 | David+Richard | Jack+Yishak | David+Richard | D:5,R:5,J:5,Y:4 |
| 5 | Richard+Karthik | Jordan+Jack | Richard+Karthik | R:5,K:5,J:2,Jo:4 |

## Tabs
`lb` · `log` · `hist` · `recaps` · `champ` · `duel` · `injury` · `music`

## Key features

### Leaderboard (`lb`)
14-column CSS grid. CSS class `.lbg` defines the grid; `.lbscroll` wraps it for horizontal scroll.
Columns: Rank · Player · Last5 · GP · GB · W/L · Win% · Cups · Cup% · Cup+/− · ⚽Puskas · 🩲NakedLaps · CPR · 🏆%

**CPR (Claude Player Rating)** — composite 0–100:
| Component | Max pts |
|---|---|
| Win Rate | 30 |
| Cup Efficiency (cups / GP*5) | 20 |
| Cup Differential (normalized) | 15 |
| Recent Form (last 3 games) | 12 |
| Clutch Factor (win rate in close games) | 10 |
| Puskas Rate | 5 |
| Opponent Strength | 5 |
| Sample Size bonus | 3 |
| Naked Lap penalty | −4 each, capped −12 |

**🏆% (Championship Probability)** — Monte Carlo (3,000 runs), TARGET = max(current+8, 18), win prob via softmax on CPR sums (temp=30).

**GB (Games Behind)** — standard sports formula, leader shows `—`.

**🔥 On-fire**: ≥2 wins in last 3 AND recent win rate or cup rate beats overall by 15%+.

**👑**: 75%+ win rate.

### Matchup suggester
`suggest(games)` → calls `suggestN(games, 1, [])[0]`
`suggestN(games, n, excludePids=[])` — scores all valid 2v2 combos:
- +30 per repeated teammate pair
- +10 per repeated opponent pair  
- +8 per player who played in last 3 games
- +2 per total games played
- −3 per sitting player per game behind most-played

### FlongDuel (`duel`)
Sportsbook-style odds tab. Book margin: **15% juice** — implied probs always sum to 1.15.

**Odds model**: Binomial(5, p̂ᵢ) per player.
- p̂ᵢ = shrunk per-cup hit probability: `(empiricalP * GP + leagueP * 3) / (GP + 3)`
- empiricalP = cupAvg / 5
- All outcomes (moneyline, spread, O/U, props) computed **analytically** by enumerating 6⁴=1296 joint outcomes — no Monte Carlo
- `toAm(p)`: converts vigged prob to American odds string
- `vig2(trueP)`: returns `[vigP, vigQ, oddsP, oddsQ]` where `vigP = trueP * 1.15`

**Markets**:
- **Moneyline**: derived from same Binomial model (P(sumA > sumB))
- **Cup Spread**: line in [-4.5..+4.5] where P(sumA−sumB > L) closest to 50%
- **Player O/U**: floating line [0.5..4.5] per player, closest to 50% over prob from Binomial CDF
- **Close Game**: P(loser team hits ≥8 cups total), from joint team distribution
- **Naked Lap**: P(at least one loser hits 0) = 1 − ∏(1−(1−pᵢ)⁵) over losing team, weighted by win prob
- **Any Puskas**: 1 − ∏(1−pᵢ_puskas), empirical with shrinkage

**Custom matchup picker**: `duelCustom` state — click 4 players in order (first 2 = Team A, next 2 = Team B). Full odds board rendered for both suggested and custom matchup.

### Injuries (`injury`)
`injuredPid` state — marks one player injured, shows next 3 suggested matchups excluding them via `suggestN(games, 3, [injuredPid])`.

### Auto-export
Every time a game is logged via `addGame()`, a JSON file is automatically downloaded named `flong-gN-YYYY-MM-DD.json`.

## Critical implementation notes

### Things that will crash the app
1. **Never define React components inside IIFE render functions** — define them as plain functions returning JSX before the return statement
2. **Never reference a variable before it's defined** — `lb` must be fully computed before anything that uses `st` (which is built from `lb`)
3. **Always guard `nakedLap` and `puskas` accesses**: `g.nakedLap&&g.nakedLap[pid]` pattern everywhere (older seed data may not have these fields)
4. **Bump localStorage key** (e.g. `d401_flong_v7`) whenever the game object schema changes, otherwise old cached data crashes the app

### Brace balance check
After any edit, verify with:
```bash
node -e "
const fs=require('fs');
const s=fs.readFileSync('flong-dashboard.html','utf8').match(/<script type=\"text\/babel\">([\s\S]*?)<\/script>/)[1];
let o=0,c=0; for(const ch of s){if(ch==='{')o++;if(ch==='}')c++;}
console.log(o===c?'✓ BALANCED':'✗ UNBALANCED',o,c);
"
```

### CSS grid for leaderboard
```css
grid-template-columns: 34px 160px 90px 48px 52px 64px 64px 56px 64px 64px 64px 68px 68px 72px;
min-width: 1060px;
```
Wrapped in `.lbscroll` div with `overflow-x: auto`.

### Colors
```js
const CA="#00f5ff"  // Team A cyan
const CB="#ff2d78"  // Team B pink
const GOLD="#ffd700"
const NAK="#ff6b35"
const FD_GREEN="#1aff8c"  // FlongDuel green (defined inside duel tab IIFE)
const OVER_COL="#1aff8c"  // Over = green
const UNDER_COL="#f0a500"  // Under = amber
```

## Repo structure
```
flong_app/
├── flong-dashboard.html   ← entire app
├── CLAUDE.md              ← this file
├── README.md              ← player & developer docs
├── data/
│   └── flong-latest.json  ← canonical latest game data (fetched on load)
└── history/
    ├── html/              ← old dashboard iterations
    └── json/              ← old data snapshots & backups
```

## GitHub Pages
Served at: `https://jackl385.github.io/flong_app/flong-dashboard.html`
Settings → Pages → Deploy from branch → main / root.

## Workflow for future changes
- Edit `flong-dashboard.html`
- Run brace balance check
- Open in browser and test
- `git add flong-dashboard.html && git commit -m "..." && git push`
- Page redeploys automatically (~60s)
