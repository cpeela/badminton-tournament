# Badminton Tournament Manager

A single-file web app for managing doubles badminton tournaments with 30 players across 5 groups, progressing through qualifiers, semi-finals, and finals. Built for the **SoCal Badminton Club**.

**Live**: [cpeela.github.io/badminton-tournament](https://cpeela.github.io/badminton-tournament/)

---

## Quick Start

```bash
# Local dev — serve the directory on any port
npx serve -l 8080
# Open http://localhost:8080/tournament-v2.html
```

No build tools, no dependencies (except SheetJS CDN for Excel import). Just a single HTML file.

---

## Architecture

### Single-file structure (`tournament-v2.html` / `index.html` — 2916 lines)

| Section | Lines | Description |
|---------|-------|-------------|
| **CSS (style 1)** | 10–1010 | Design tokens, component styles, 768px responsive breakpoint |
| **CSS (style 2)** | 1011–1133 | Mobile breakpoints (480px, 375px) — separate `<style>` block to avoid parse issues |
| **HTML** | 1134–1200 | Static skeleton: auth gate, header, tabs, main panels, modal, toast container |
| **JS: Auth** | 1204–1295 | PIN gate, guest/admin mode, session persistence |
| **JS: State** | 1297–1380 | State schema, localStorage persistence, export, import roster (xlsx) |
| **JS: Utils** | 1382–1515 | `uid()`, `getPlayerName()`, `toast()`, player/group CRUD, group picker modal |
| **JS: Core Logic** | 1515–1670 | `validateScore()`, `generateDoublesSchedule()`, `computeStandings()` |
| **JS: Renderers** | 1674–2800 | `renderSetup()`, `renderQR1()`, `renderQR2()`, `renderSemis()`, `renderFinals()`, `renderDashboard()` |
| **JS: Navigation** | 2800–2860 | Tab switching (ARIA-compliant keyboard nav) |
| **JS: Init** | 2860–2916 | Default player seeding, `renderAll()` |

### Why single-file?
- Zero-deploy friction (GitHub Pages serves `index.html`)
- Shareable as a single file via email/Slack
- No build step = anyone can edit with a text editor
- All state lives in `localStorage` — no server needed

---

## Tournament Flow

```
Setup → Qualifier R1 → Qualifier R2 → Semi-Finals → Finals → Dashboard
```

### 1. Setup
- 30 players, 5 groups of 6
- Each group has a designated leader
- Config: matches/player, points-to-win, deuce cap, win weight, advance-top-N

### 2. Qualifier R1 (Group Stage)
- Each group plays round-robin doubles (randomized pairings)
- `matchesPerPlayer` controls how many matches each player plays (default: 4)
- Scoring: `WP = (Wins x winWeight) + Sum(WinDiff) - Sum(LossDiff)`
- Top N per group → Champion Pool, rest → Consolation Pool

### 3. Qualifier R2 (Pool Stage)
- Champion Pool (15 players) and Consolation Pool (15 players) play separately
- Same scoring formula, fresh standings
- Top `semiFinalSlots` (default: 8) advance to semis

### 4. Semi-Finals
- 8 players, seeded 1v8, 2v7, 3v6, 4v5
- Single elimination doubles matches
- Winners advance to finals

### 5. Finals
- 4 remaining players: 2 matches (Champion Final + Consolation Final)
- Best-of-3 format with per-game score entry

### 6. Dashboard
- Summary stats, top performers, tournament progress

---

## Key Algorithms

### Doubles Schedule Generation (`generateDoublesSchedule`)
- Takes N player IDs and desired matches per player
- Generates all possible 2v2 combinations (no player on both sides)
- Shuffles and selects matches ensuring each player plays approximately the target count
- Falls back to relaxed constraints if perfect balance isn't achievable

### Standings Computation (`computeStandings`)
- **WP** (Weighted Points) = `(wins * winWeight) + totalWinPointDiff - totalLossPointDiff`
- `winPointDiff` = sum of `(myScore - opponentScore)` for wins only
- `lossPointDiff` = sum of `(opponentScore - myScore)` for losses only
- Tiebreaker: WP → wins → winPointDiff (descending)

### Score Validation (`validateScore`)
- Normal win: one team reaches `pointsToWin` (21), other has less
- Deuce: if both reach `pointsToWin - 1` (20-20), winner must lead by 2
- Deuce cap: max score is `deuceCap` (25), wins at 25-24 allowed

---

## Auth System

| Role | Access |
|------|--------|
| **Admin** (PIN: `2025`) | Full CRUD: score entry, config, reset, export/import |
| **Guest** (no PIN) | Read-only: view standings, matches, dashboard |

- PIN is stored as `const ADMIN_PIN = '2025'` at line ~1207 — change it there
- Role persists in `sessionStorage` (survives refresh, clears on tab close)
- Guest mode applies CSS class `body.guest-mode` which hides `.btn-primary`, `.btn-success`, `.btn-danger`, `.score-entry`, `.header-actions` and disables all inputs

---

## Data Model

```javascript
state = {
  config: {
    tournamentName, date, venue,
    numGroups: 5, playersPerGroup: 6, matchesPerPlayer: 4,
    pointsToWin: 21, deuceEnabled: true, deuceCap: 25,
    winWeight: 20, advanceTop: 3, semiFinalSlots: 8,
    finalsFormat: 'bestOf3'
  },
  players: [{ id, name }],
  groups: [{ id, name, playerIds[], leaderId, court }],
  rounds: {
    qr1:   { status, matches[], standings[] },
    qr2:   { status, pools: { champion[], consolation[] }, matches[], standings[] },
    semis:  { status, matches[] },
    finals: { status, matches[] }
  }
}
```

### Match object
```javascript
{
  id, groupId, matchNum, court,
  team1: { playerIds: [id, id], score: null },
  team2: { playerIds: [id, id], score: null },
  status: 'scheduled' | 'completed',
  winner: null | 'team1' | 'team2',
  birdies: null  // shuttlecock count (optional tracking)
}
```

### Storage
- **localStorage key**: `badminton_tournament_v2`
- **Export**: downloads full state as JSON
- **Import Roster**: accepts `.xlsx/.xls/.csv` with columns `First Name`, `Last Name`, `Group`

---

## Responsive Breakpoints

| Breakpoint | Target | Key changes |
|------------|--------|-------------|
| **> 768px** | Desktop/laptop | Full horizontal layout, all table columns visible |
| **768px** | Tablet | Single-column grids, match cards wrap, reduced padding |
| **480px** | Phone portrait | Header wraps (title truncated), vertical match cards, PF/PA columns hidden, 44px touch targets, compact auth gate |
| **375px** | iPhone SE | Single-column forms, Win/Loss Diff columns also hidden, smallest spacing |

Mobile CSS lives in a **separate `<style>` block** (lines 1011–1133) because the first style block's length causes intermittent CSS parser issues in some browsers when media queries are appended at the end.

---

## Files

| File | Purpose |
|------|---------|
| `tournament-v2.html` | Source file (edit this) |
| `index.html` | Deploy copy (copy from tournament-v2.html before pushing) |
| `tournament.html` | Original v1 (archived, not deployed) |
| `sample-roster.xlsx` | Sample Excel import file (30 players, 5 groups) |
| `sample-import.json` | Sample full-state JSON (for reference, not used by import) |

### Deploy workflow
```bash
cp tournament-v2.html index.html
git add index.html
git commit -m "Update deploy"
git push  # GitHub Pages auto-deploys from main branch
```

---

## External Dependencies

| Library | Version | CDN | Purpose |
|---------|---------|-----|---------|
| SheetJS (xlsx) | 0.20.3 | `cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js` | Parse .xlsx/.xls/.csv roster imports |
| Google Fonts | — | Outfit, DM Sans, DM Mono | Typography |

No npm packages. No build tools.

---

## Design System

### Fonts
- **Display**: Outfit (headings, tabs, labels)
- **Body**: DM Sans (text, inputs, buttons)
- **Mono**: DM Mono (scores, numbers, code)

### Color Palette
- **Primary**: `#d4a853` (warm gold) — actions, active tabs, focus rings
- **Accent**: `#2dd4a8` (teal) — progress bars
- **Success**: `#34d399` — completed states, admin badge
- **Warning**: `#fbbf24` — alerts, champion pool
- **Danger**: `#f87171` — errors, reset button
- **Champion pool**: `#fbbf24` gold rows
- **Consolation pool**: `#a78bfa` purple rows
- **Surfaces**: `#0c1018` → `#111827` → `#1a2233` → `#243044` (dark navy gradient)

### Component Classes
- `.card` — content container with border and hover
- `.match-card` — flex row for match display (stacks vertically on mobile)
- `.badge` / `.badge-{variant}` — status pills
- `.btn` / `.btn-{primary|success|danger|ghost}` / `.btn-{sm|lg}` — buttons
- `.stat-card` — dashboard metric display
- `.player-chip` — roster display with leader star
- `.group-chip` — clickable group reassignment
- `.toast` — bottom-right notifications (success/error/info)

---

## Known Limitations

1. **No real-time sync** — single-device state in localStorage. Multiple admins on different devices will overwrite each other. Workaround: one admin device, guests view on their phones.
2. **PIN is client-side** — not secure, just a convenience gate. Anyone can view source to find the PIN. Adequate for a casual club tournament.
3. **No undo** — score submissions and round advancements are permanent (can only Reset entire tournament).
4. **Fixed 2v2 format** — the schedule generator assumes doubles (2 players per team). Singles or mixed formats would require refactoring `generateDoublesSchedule`.
5. **Mobile CSS in separate style block** — due to a browser CSS parsing quirk, the mobile media queries must live in a second `<style>` tag. Keep them there when editing.

---

## Future Improvements (Not Implemented)

- Real-time sync via Firebase/Supabase for multi-device admin
- QR code display for easy guest access link sharing
- Print-friendly bracket view
- Player stats history across tournaments
- Configurable team sizes (singles, mixed doubles)
- Server-side PIN validation
