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

No build tools, no dependencies (except SheetJS and Supabase CDNs). Just a single HTML file.

**Page routing**: Use hash URLs to link directly to any page — e.g. `#qr1`, `#dashboard`.

---

## Architecture

### Single-file structure (`tournament-v2.html` / `index.html` — ~4700 lines)

Sections appear in this order (search for the `═══` banner comments to jump between them):

| Section | Description |
|---------|-------------|
| **Pre-paint theme script** | Applies saved `bt_theme` before first render |
| **CSS (style 1)** | Light/dark tokens, type scale, component styles, desktop layout |
| **CSS (style 2)** | Mobile breakpoints (768px, 480px, 375px) — separate `<style>` block to avoid parse issues |
| **HTML** | Static skeleton: auth gate, header (theme toggle, More menu), tabs, main panels, modal, toast container, bracket overlay |
| **JS: Auth** | PIN gate (super admin / admin / guest), session persistence |
| **JS: Theme** | `applyTheme()` / segmented toggle wiring |
| **JS: State** | `SCHEMA_VERSION`, `migrateState()`, localStorage persistence, export, import roster (xlsx) |
| **JS: Utils** | `uid()`, `getPlayerName()` / `getPlayerNameSafe()`, `escapeHtml()`, `toast()`, `openModal()` / `confirmModal()`, player/group CRUD |
| **JS: Core Logic** | `validateScore()`, `generateDoublesSchedule()`, `computeStandings()` |
| **JS: Renderers** | `renderSetup()`, `renderQR1()`, `renderQR2()`, `renderSemis()`, `renderFinals()`, `renderDashboard()`, `renderBracket()` |
| **JS: Supabase Sync** | Real-time sync, conflict resolution, offline fallback |
| **JS: Navigation** | Hash-based page routing (ARIA-compliant keyboard nav, active tab scroll-into-view on mobile) |
| **JS: Init** | Default player seeding, `renderAll()`, `applyRoute()` |

### Why single-file?
- Zero-deploy friction (GitHub Pages serves `index.html`)
- Shareable as a single file via email/Slack
- No build step = anyone can edit with a text editor
- All state lives in `localStorage` — no server needed

---

## Tournament Flow

```
Setup (#setup) → Qualifier R1 (#qr1) → Qualifier R2 (#qr2) → Semi-Finals (#semis) → Finals (#finals) → Dashboard (#dashboard)
```

Each page is a hash route — shareable, bookmarkable, with browser back/forward navigation.

**Per-pool advancement**: from QR2 onward Pool A and Pool B progress independently. Each pool has its own "Advance" button (QR2 → Semis, Semis → Final) that enables once that pool's matches are done, so one pool can start its final while the other is still in semis. QR1 → QR2 stays global because both pools are seeded from every group's results.

### 1. Setup
- 30 players, 5 groups of 6
- Each group has a designated leader
- Config: matches/player, points-to-win, deuce cap, win weight, advance-top-N

### 2. Qualifier R1 (Group Stage)
- Each group plays round-robin doubles (randomized pairings)
- `matchesPerPlayer` controls how many matches each player plays (default: 4)
- Ranking: most wins → most total points won → fewest total points lost
- Top N per group → Pool A, rest → Pool B

### 3. Qualifier R2 (Pool Stage)
- Pool A (15 players) and Pool B (15 players) play separately
- Same scoring formula, fresh standings
- Top `semiFinalSlots` (default: 8) advance to semis

### 4. Semi-Finals
- 8 players per pool, seeded 1v8, 2v7, 3v6, 4v5
- Per-pool format toggle: **Single Game** or **Best of 3** (admin chooses at runtime, not setup)
- Switching format mid-tournament preserves existing scores
- Winners advance to finals

### 5. Finals
- 4 remaining players: 2 matches (Pool A Final + Pool B Final)
- Per-pool format configured in Setup: **Single Game** or **Best of 3** (`championPoolFormat`, `consolationPoolFormat`)

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
- Per player: `wins`, `losses`, `totalPointsScored` (PF), `totalPointsConceded` (PA)
- `sortStandings()` ranks by **wins → PF (desc) → PA (asc)**, identically in QR1 and QR2
- Players still tied on all three get a **Tie** badge in the standings so the officiating table resolves the order deliberately; the app does not pick arbitrarily
- `config.winWeight` is retained in saved state for backward compatibility but no longer used

### Score Validation (`validateScore`)
- Normal win: one team reaches `pointsToWin` (21), other has less
- Deuce: if both reach `pointsToWin - 1` (20-20), winner must lead by 2
- Deuce cap: max score is `deuceCap` (25), wins at 25-24 allowed

---

## Auth System

| Role | PIN | Access |
|------|-----|--------|
| **Super Admin** | `2809` | Full CRUD + **Reset tournament** (only role that can reset) |
| **Admin** | `2025` | Full CRUD: score entry, config, export/import — **no reset** |
| **Guest** | *(none)* | Read-only: view standings, matches, dashboard |

- PINs are stored as `const ADMIN_PIN = '2025'` and `const SUPER_ADMIN_PIN = '2809'` — change them there
- Role persists in `sessionStorage` (survives refresh, clears on tab close)
- **Logout** button appears in header for all roles — returns to auth gate
- Super Admin mode applies CSS class `body.super-admin-mode` which shows `.super-admin-only` elements (the Reset button)
- Guest mode applies CSS class `body.guest-mode` which:
  - Hides `.btn-primary`, `.btn-success`, `.btn-danger`, `.score-entry`, `.header-actions`, `.admin-only`
  - Disables all inputs and selects via `pointer-events: none`
  - Blocks leader toggle and group reassignment clicks
  - Hides edit score pencil icons, finals edit buttons, bulk add/auto-assign, remove player buttons
  - Hides the "Add Player" form entirely
  - JS guards (`if (!isAdmin()) return`) on: `toggleLeader`, `editScore`, `showGroupPicker`, `renamePlayer`, `showBulkAdd`, `autoAssignGroups`, `editFinalGame`

---

## Data Model

```javascript
state = {
  config: {
    tournamentName, date, venue,
    numGroups: 5, playersPerGroup: 6, matchesPerPlayer: 4,
    pointsToWin: 21, deuceEnabled: true, deuceCap: 25,
    winWeight: 20, advanceTop: 3, semiFinalSlots: 8,
    finalsFormat: 'bestOf3',
    championPoolFormat: 'bestOf3',   // Finals format per pool
    consolationPoolFormat: 'single'
  },
  players: [{ id, name }],
  groups: [{ id, name, playerIds[], leaderId, court }],
  rounds: {
    qr1:   { status, matches[], standings[] },
    qr2:   { status, poolStatus: { champion, consolation }, pools: { champion[], consolation[] }, matches[], standings[] },
    semis:  { status, poolStatus: { champion, consolation }, matches[], poolFormats: { champion, consolation } },
    finals: { status, poolStatus: { champion, consolation }, matches[] }
  }
}
```

`poolStatus` values are `'not_started' | 'in_progress' | 'completed'` per pool; the round-level `status` is derived from them by `recomputeRoundStatus()` (`completed` only when both pools are). `schemaVersion` is currently 3 — `migrateState()` backfills older saved/remote states (v2 states get `poolStatus` copied from the round status).

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
| Supabase JS | 2.x | `cdn.jsdelivr.net/npm/@supabase/supabase-js@2` | Real-time sync, remote state storage |
| Google Fonts | — | Outfit, DM Sans, DM Mono | Typography |

No npm packages. No build tools.

---

## Design System

### Theming
- Light and dark themes via CSS custom properties on `html[data-theme="light"|"dark"]`; with no attribute set, the app follows `prefers-color-scheme`.
- Header segmented control: **Light / Dark / System** (a compact cycle button on phones). Choice persists in `localStorage` key `bt_theme`.
- A pre-paint inline `<script>` in `<head>` applies the saved theme before first render to avoid a flash.
- All colors are tokens (`--bg`, `--surface`, `--surface-2`, `--border`, `--text`, `--text-2`, `--text-3`, `--primary`, `--accent`, `--success`, `--warning`, `--danger`, `--champion`, `--consolation`, shadows). Do not hardcode hex values in components.
- Destructive / round-advancing actions (start, advance, reset, roster import) use the in-app `confirmModal()` instead of native `confirm()`.

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
- **Pool A** (internal key `champion`): gold rows
- **Pool B** (internal key `consolation`): purple rows
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

1. **PINs are client-side** — not secure, just a convenience gate. Anyone can view source to find the PINs. Reset is both UI-gated (`.super-admin-only` CSS) and code-guarded (`isSuperAdmin()` check in `resetTournament()`). Adequate for a casual club tournament.
2. **No undo** — score submissions and round advancements are permanent (can only Reset entire tournament).
3. **Fixed 2v2 format** — the schedule generator assumes doubles (2 players per team). Singles or mixed formats would require refactoring `generateDoublesSchedule`.
4. **Mobile CSS in separate style block** — due to a browser CSS parsing quirk, the mobile media queries must live in a second `<style>` tag. Keep them there when editing.
5. **Per-match merge on sync conflicts** — each match carries an `updatedAt` stamp set on score entry/edit. When two admins write simultaneously, the loser of the version race pulls the remote state, keeps any of its own matches that are newer, and pushes again (`mergeMatchEdits`). Non-match changes (config, round advancement) are still last-write-wins, so have one person drive advancement. Incoming updates also preserve whatever a user is mid-typing (`withPreservedInputs`).

---

## Future Improvements (Not Implemented)

- QR code display for easy guest access link sharing
- Print-friendly bracket view
- Player stats history across tournaments
- Configurable team sizes (singles, mixed doubles)
- Server-side PIN validation
