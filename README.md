# D401 Flong Scoreboard

A single-file HTML dashboard for tracking **D401 Flong**, a beer pong / flip cup hybrid drinking game played by 6 friends at MIT.

No build step, no server — open `flong-dashboard.html` in a browser and go.

---

## For Players (Using the App)

### Getting Started

1. Open `flong-dashboard.html` in any modern browser (Chrome, Safari, Firefox)
2. The dashboard loads with all game history and lands on the **Leaderboard** tab

If you're on GitHub Pages, just visit the hosted URL — game data loads automatically.

### Tabs

| Tab | What it does |
|-----|-------------|
| **Leaderboard** | Full stats table — Win%, Cups, CPR rating, Championship%, and more |
| **Log Game** | Record a new game result |
| **History** | Browse all past games (newest first), delete games if needed |
| **Recaps** | AI-generated ESPN-style 2-sentence game recaps |
| **Championship** | Monte Carlo simulation of each player's odds to finish top 4 |
| **FlongDuel** | Vegas-style sportsbook odds for the next suggested matchup |
| **Injuries** | Mark a player injured; auto-suggests matchups without them |
| **Playlist** | Curated Spotify playlists for game night vibes |

### Logging a Game

1. Go to the **Log Game** tab
2. Click players to assign them to **Team A** (cyan) or **Team B** (pink) — 2 per team
3. Enter each player's remaining cups (0–5)
4. Toggle **Puskas** (hit 5 in a row) or **Naked Lap** (0 cups on losing team) if applicable
5. Hit **+ ADD GAME** — the winner is calculated automatically
6. A JSON backup file auto-downloads to your `~/Downloads/` folder

### Data & Backups

All game data is saved in your **browser's localStorage**. This means:

- Your data persists between sessions in the same browser
- A different browser or device starts fresh
- Clearing browser data erases your games

**To protect your data:**

- **Export**: Click the export button on the Leaderboard tab to download a full JSON backup
- **Import**: Click import to restore from a previously exported JSON file
- **Auto-export**: Every time you log a game, a JSON snapshot auto-downloads

### Keeping Everyone in Sync

The app tries to fetch `data/flong-latest.json` on load. When hosted on GitHub Pages, this means everyone gets the latest data automatically — as long as someone pushes the updated JSON after each game session.

**After a game night:**

1. Find the auto-downloaded JSON in `~/Downloads/` (named `flong-gN-YYYY-MM-DD.json`)
2. Copy it to `data/flong-latest.json` in the repo
3. Push to GitHub

### Key Stats Explained

- **CPR (Claude Player Rating)**: Composite 0–100 score factoring win rate, cup efficiency, recent form, clutch play, and more
- **GB (Games Behind)**: How far behind the leader, using the standard sports formula
- **Cup%**: Total cups kept / maximum possible (GP x 5)
- **Cup +/-**: Your average cups per game minus your opponents' average
- **Championship%**: Monte Carlo simulation (3,000 runs) of your odds to finish top 4

Icons: **fire** = hot streak (2+ wins in last 3, stats trending up 15%+), **crown** = 75%+ win rate

---

## For Developers (Editing the App)

### Project Structure

```
flong_dashboard/
├── flong-dashboard.html    <- entire app (single file)
├── CLAUDE.md               <- detailed dev context doc
├── README.md               <- this file
├── data/
│   └── flong-latest.json   <- canonical latest game data (fetched on load)
└── history/
    ├── html/               <- old dashboard iterations (46 versions)
    └── json/               <- old data snapshots (23 exports)
```

### Tech Stack

- **React 18** (UMD build from CDN) + **Babel standalone** for JSX transpilation
- All in one HTML file — no npm, no bundler, no build step
- Inline styles and a small `<style>` block (no Tailwind)
- Data stored in `localStorage` under key `d401_flong_v6`

### Architecture

The app is a single React component (`App`) using hooks (`useState`, `useMemo`, `useEffect`, `useRef`). All game logic, stats computation, and UI live in one `<script type="text/babel">` block.

**Data flow:**

1. On load: read `localStorage` -> fall back to hardcoded `SEED` constant
2. On mount: async fetch `data/flong-latest.json` -> update state if it has more games
3. On any state change: auto-save to `localStorage`
4. On game logged: auto-download JSON backup

### Key Functions

| Function | Purpose |
|----------|---------|
| `load()` | Synchronous initial load from localStorage or SEED |
| `fetchRemote()` | Async fetch of `data/flong-latest.json` on mount |
| `addGame()` | Validates form, creates game object, triggers auto-export |
| `cpr(pid, games)` | Computes Claude Player Rating (0-100) |
| `suggest(games)` | Suggests next matchup based on fairness + rotation |
| `suggestN(games, n, exclude)` | Suggests N matchups, optionally excluding players |
| `genRecap(game, players)` | Calls Claude API to generate ESPN-style recap |
| `imp(e)` / `exp()` | Import/export JSON data |

### Making Changes

1. Edit `flong-dashboard.html`
2. Run the brace balance check:
   ```bash
   node -e "
   const fs=require('fs');
   const s=fs.readFileSync('flong-dashboard.html','utf8').match(/<script type=\"text\/babel\">([\s\S]*?)<\/script>/)[1];
   let o=0,c=0; for(const ch of s){if(ch==='{')o++;if(ch==='}')c++;}
   console.log(o===c?'✓ BALANCED':'✗ UNBALANCED',o,c);
   "
   ```
3. Open in a browser and test
4. Push to GitHub — GitHub Pages redeploys automatically (~60s)

### Common Pitfalls

- **Never define React components inside IIFE render functions** — define them as plain functions before the return
- **Never reference a variable before it's defined** — `lb` must be computed before anything that reads `st`
- **Always guard optional fields**: `g.nakedLap&&g.nakedLap[pid]` (older seed data may lack these)
- **Bump the localStorage key** (e.g., `d401_flong_v7`) when changing the game object schema, otherwise old cached data will crash the app
- **No `export default`** — breaks UMD React

### Data Schema

```json
{
  "players": { "p1": "David", "p2": "Jack", ... },
  "games": [
    {
      "id": 1741000001,
      "teamA": ["p1", "p2"],
      "teamB": ["p3", "p4"],
      "cups": { "p1": 5, "p2": 5, "p3": 4, "p4": 5, "p5": 0, "p6": 0 },
      "puskas": { "p1": false, ... },
      "nakedLap": { "p1": false, ... },
      "winner": "A"
    }
  ],
  "recaps": { "<gameId>": { "text": "...", "ts": 1234567890 } }
}
```

### Updating Seed Data

When new games are played and you want them baked into the HTML (so fresh browsers get them immediately):

1. Copy the latest JSON into `data/flong-latest.json`
2. Update the `SEED` constant in the HTML with the new games
3. The `fetchRemote()` mechanism handles the GitHub Pages case, but updating SEED covers offline/local file opens too

See `CLAUDE.md` for more detailed dev context including CPR formula weights, odds model math, color constants, and CSS grid specs.
