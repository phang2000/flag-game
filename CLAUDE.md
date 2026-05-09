# Flag Rush — Claude Code Handoff Document

## Vision

Flag Rush is a single-page flag identification game with ambitions to become the **global benchmark for flag knowledge and speed**. The core design philosophy taps into speedrunning culture — the score ceiling is mathematically unreachable, so players always have something to chase. The goal is high UGC output on social media driven by the addictive grind for a better score.

**The three pillars of the grind:**
1. **Spelling quality** — perfect answers vs close answers vs revealed
2. **Streak consistency** — consecutive correct answers compound your multiplier
3. **Speed** — end-of-run bonus rewards faster completions

---

## Current State

- **Source file:** `flagrush_v3.html` — single self-contained HTML/CSS/JS file, ~2,020 lines
- **193 flags** covering all sovereign nations, grouped by continent
- **Two play modes:** All flags (193) or by continent (Africa, Americas, Asia, Europe, Oceania)
- **Persistence:** localStorage for leaderboard, recent runs, region bests, missed flags, username
- **Theming:** Light/dark toggle, CSS custom properties throughout

---

## Scoring System (Locked — Do Not Change Without Discussion)

### Per-flag base points
| Result | Points |
|--------|--------|
| Perfect spelling, first attempt | 100 pts |
| Close spelling (within threshold) | 60 pts |
| Used reveal, then typed correctly | 20 pts |
| Skipped | 0 pts |

### Streak multiplier
Applied to base points. Streak increments on **perfect or close** answers. Resets to 0 on **skip or reveal**. **Wrong answers also reset the streak.**

| Streak | Multiplier |
|--------|------------|
| < 3 | 1.0× |
| 3–4 | 1.2× |
| 5–9 | 1.5× |
| 10+ | 2.0× |

Formula: `flag score = base pts × streakMultiplier(currentStreak)`

### Speed bonus (end of run)
Asymptotic cubic curve — approaches but **mathematically never reaches** the maximum. This is intentional and core to the speedrun hook.

```js
function calcSpeedBonus(totalMs, flagCount) {
  const MAX_BONUS = 5000;
  const ASYMPTOTE = 0.92;         // never reaches 100% of max
  const FLOOR_S = 60;             // minimum meaningful time
  const cutoffS = Math.max(60, flagCount * 9.33); // ~30min for all 193
  const t = totalMs / 1000;
  if (t >= cutoffS) return 0;
  const tClamped = Math.max(FLOOR_S, t);
  const r = 1 - tClamped / cutoffS;
  const trueMax = Math.pow(1 - FLOOR_S / cutoffS, 3);
  const raw = r * r * r;
  const scaled = trueMax > 0 ? (raw / trueMax) * ASYMPTOTE : 0;
  return Math.round(scaled * MAX_BONUS);
}
```

- Scales proportionally for continent modes (`flagCount × 9.33` seconds cutoff)
- Shown as a separate line item on the results screen so players understand the contribution
- **Theoretical absolute max: ~42,500 pts — permanently unreachable**

### Final score
`finalScore = sum(flagScores) + speedBonus`

### Star rating (results screen)
| Stars | Score threshold |
|-------|----------------|
| ⭐ | < 8,000 |
| ⭐⭐ | 8,000 – 17,999 |
| ⭐⭐⭐ | 18,000 – 29,999 |
| ⭐⭐⭐⭐ | 30,000 – 38,999 |
| ⭐⭐⭐⭐⭐ | ≥ 39,000 |

5 stars requires near-perfect spelling + long streaks + fast time simultaneously. Peter's personal best (14 min, expert knowledge) sits around 34k — solid 4 stars. Only elite runs break 5 stars.

---

## Close Answer Matching

The fuzzy match uses Levenshtein distance, **scaled by the target word's length** to prevent false positives on short country names:

| Target length | Threshold |
|---------------|-----------|
| ≤ 4 chars (Chad, Cuba, Laos, Oman) | 0 — exact match only |
| 5–8 chars (Egypt, Ghana, Qatar) | 1 edit |
| 9+ chars (Argentina, Bangladesh) | 2 edits |

```js
const threshold = t.length <= 4 ? 0 : t.length <= 8 ? 1 : 2;
```

---

## Game State Variables

```js
// Core
let shuffled = [];          // randomised flag array for current run
let currentIndex = 0;       // current position in shuffled
let currentContinent = null; // null = all 193 flags

// Timing
let timerInterval = null;   // always clearInterval this at top of startGame()
let startTime = 0;
let elapsedMs = 0;
let flagStartTime = 0;

// Counts
let correctCount = 0;
let skippedCount = 0;
let closeCount = 0;
let revealedCount = 0;
let perfectCount = 0;

// Scoring
let currentScore = 0;
let currentStreak = 0;
let bestStreak = 0;

// Data
let splitTimes = [];        // per-flag timing and result type
let missedFlags = [];       // flags skipped or revealed — for Practice mode (Phase 3)

// Meta
let currentUser = '';
let feedbackTimeout = null;
let inputLocked = false;
```

---

## localStorage Keys

| Key | Contents |
|-----|----------|
| `flagrush_leaderboard` | Array of `{name, timeMs, score}` — global all-flags leaderboard |
| `flagrush_runs` | Array of recent runs (max 20) with score, stars, continent, date |
| `flagrush_region_bests` | Object keyed by continent with best time per region |
| `flagrush_missed` | Array of `{code, name, reason}` — accumulated missed flags for Practice mode |
| `flagrush_username` | Last used display name |
| `flagrush_theme` | `'light'` or `'dark'` |

---

## UI Architecture

### Screens (shown/hidden via `showScreen(id)`)
- `#home` — landing, username input, leaderboard
- `#mode-select` — continent picker overlay
- `#game` — active gameplay
- `#results` — end-of-run results with score, stars, breakdown, leaderboard

### Key game screen elements
- `#live-score` — big central score display, bumps with animation on every answer
- `#streak-hud` — below topbar, shows streak progress toward next multiplier tier
- `#timer-display` — right side of topbar, mm:ss elapsed
- `#progress-text` — flag count progress (e.g. "47 / 193")
- `#progress-fill` — progress bar fill width
- `#flag-img-wrap` — flag image container, score pop animations append here
- `#answer-input` — text input, state classes: `state-correct`, `state-wrong`, `state-close`
- `#feedback-label` — "✓ Correct!", "Almost… check your spelling", "Not quite — keep trying"

### Score pop animation
Appended to `.flag-img-wrap`, rises from centre of flag, gold for perfect, purple for close. Font scales with multiplier tier (to be implemented).

---

## Bugs Fixed (Phase 1 — Complete)

| Bug | Fix applied |
|-----|-------------|
| Stale timer interval on back-navigation | `clearInterval(timerInterval)` as first line of `startGame()` |
| Close-match threshold on short names | Threshold now scales by target word length, not input length |
| Hardcoded hex in theme toggle | Replaced with `var(--text)` / `var(--bg)` CSS variables |
| Skip data not persisted | Dedicated `missedFlags[]` array, saved to localStorage on run end |
| Wrong answers not resetting streak | `triggerWrong()` now resets `currentStreak = 0` first |
| Duplicate `revealedCount` declaration | Stray `let revealedCount = 0` removed |
| Score not visible during gameplay | Redesigned topbar — score is central hero at 1.9rem/800wt |
| Score pop appearing top-right of card | Pop now appends to `flag-img-wrap`, rises from centre of flag |

---

## Known Issues / Next Tasks

### Immediate bugs
- [ ] **Close answer auto-advance** — close answers fire a 900ms timeout then call `currentIndex++` and `showFlag()` without the player re-submitting. If the player is still correcting their spelling, the flag yanks away. Should wait for player input, not auto-advance.

### Phase 2 — UX polish (not yet built)
- [ ] Replace native `confirm()` quit dialog with custom in-game overlay card
- [ ] First-time onboarding hint (one example flag + explanation of correct/close/wrong)
- [ ] Continent mode entry point more prominent on home screen

### Phase 3 — Engagement (not yet built)
- [ ] **Practice mode** — use `flagrush_missed` localStorage data to drill the flags the player struggled with. Data is already being collected.
- [ ] **Daily Challenge** — same seed for all players, one attempt per day. Highest-leverage retention feature.
- [ ] **Results screen celebration** — animation on star reveal, personal best highlight

### Phase 4 — Growth (requires backend)
- [ ] **Difficulty tiers** — Easy (50 most common flags), Normal (193), Hard (obscure/similar)
- [ ] **Shared leaderboard** — move beyond localStorage so scores are visible globally. Currently leaderboard is per-device only.
- [ ] **Social share card** — one-tap copy of score, stars, best streak, time, formatted for social. This is the UGC trigger. Example format: `🚩 Flag Rush — 34,203 pts ★★★★☆ | 14:22 | 12× best streak`

### Design / balance
- [ ] Score pop font size should scale with multiplier tier — at 2× it should be noticeably larger
- [ ] Leaderboard should rank by score, not time
- [ ] Leaderboard currently local-only — no cross-device or cross-player visibility

---

## Territories Mode — Planned Feature

### Three game modes
| Mode | Pool | Notes |
|------|------|-------|
| **Countries** | 193 sovereign nations | Current default |
| **Territories** | Dependencies, overseas territories, autonomous regions | New |
| **Everything** | Countries + Territories combined | New |

Continent filter remains available within each mode as a secondary filter.

### Design decisions (locked)
- **Palestine** → Countries pool. Treated as sovereign nation, ISO code `ps`, flag at `flagcdn.com/w320/ps.png`
- **Territories scope** — inclusive approach. Well-known territories with distinct flags: Puerto Rico, Greenland, Hong Kong, Macau, Gibraltar, Faroe Islands, Bermuda, Guam, New Caledonia, French Polynesia, Aruba, Curaçao, Cayman Islands, Cook Islands, and more. Exact list TBD — aim for completeness, not curation.
- **Separate leaderboard per mode** — Countries, Territories, and Everything do not share leaderboards. Continent sub-modes also get their own.
- **Scoring auto-scales with flag count** — speed bonus cutoff already uses `flagCount × 9.33s`. Star rating thresholds must also scale:

### Scaled star rating formula
```js
function getStarRating(score, flagCount) {
  const scale = flagCount / 193;
  if (score >= Math.round(39000 * scale)) return 5;
  if (score >= Math.round(30000 * scale)) return 4;
  if (score >= Math.round(18000 * scale)) return 3;
  if (score >= Math.round( 8000 * scale)) return 2;
  return 1;
}
```

### Flag data format for territories
Add `type` field to distinguish from countries:
```js
{ code: "pr", name: "Puerto Rico", alt: [], continent: "Americas", type: "territory" }
```
`type: "country"` is the default and can be omitted for existing flags.

### localStorage keys to add
| Key | Contents |
|-----|----------|
| `flagrush_lb_countries` | Countries mode leaderboard |
| `flagrush_lb_territories` | Territories mode leaderboard |
| `flagrush_lb_everything` | Everything mode leaderboard |
| `flagrush_lb_continent_{name}` | Per-continent leaderboard |

---

## Expert Panel

Three reviewers used throughout development for playtesting and feedback:

| Reviewer | Role | Focus |
|----------|------|-------|
| **Jordan M.** | Game Advisor | Vision, roadmap, balance, what the game could become |
| **Sam R.** | QA Tester | Bugs, broken interactions, edge cases, code review |
| **Aisha K.** | Analyst | UX flow, engagement loops, retention hooks, social |

Panel runs code simulations against the actual scoring engine (Node.js), not assumptions. Useful for balance testing before shipping changes.

---

## Benchmark Scores (for balance testing)

| Scenario | Flag score | Speed bonus | Final | Stars |
|----------|-----------|-------------|-------|-------|
| Beginner — 35 min, mostly close/skips | ~8,400 | 0 | ~8,400 | ⭐⭐ |
| Intermediate — 25 min, mixed | ~13,000 | 0 | ~13,000 | ⭐⭐ |
| Peter (expert) — 14 min, 10 skips, 25 close | ~33,400 | ~773 | ~34,200 | ⭐⭐⭐⭐ |
| Elite — all perfect, 8 min | ~37,000 | ~2,000 | ~39,000 | ⭐⭐⭐⭐⭐ |
| Theoretical max — all perfect, 1 min | ~38,600 | ~4,600 | ~43,200 | ⭐⭐⭐⭐⭐ (unreachable) |

Peter's personal best: **14 minutes** on all-193 flags.

---

## File Structure (current — single file)

```
flagrush_v3.html
├── <head>          Google Fonts (Nunito, Inter), meta
├── <style>         All CSS — variables, screens, components, animations
├── <body>
│   ├── #home       Landing screen
│   ├── #mode-select Continent picker
│   ├── #game       Active gameplay screen
│   └── #results    End-of-run results screen
└── <script>
    ├── FLAGS[]     193 flag objects {code, name, alt[], continent}
    ├── Storage     localStorage get/save helpers
    ├── Scoring     calcFlagScore, calcSpeedBonus, getStarRating, renderStars
    ├── HUD         updateLiveScore, updateStreakHUD, showScorePop
    ├── Game logic  startGame, showFlag, submitAnswer, handleCorrect,
    │               triggerWrong, skipFlag, revealFlag, endGame
    ├── UI helpers  showScreen, renderLeaderboard, renderModeSelect
    └── Init        DOMContentLoaded → initTheme, initHome
```

Recommended first move in Claude Code: split into `index.html`, `style.css`, `flags.js`, `scoring.js`, `game.js` — this will make the close-answer auto-advance bug and future features much easier to isolate and test.

---

## Design Tokens (CSS variables)

```css
/* Light mode */
--bg: #F5F2EB;        /* warm off-white page background */
--surface: #FDFBF7;   /* card surface */
--text: #1A1814;      /* primary text */
--text2: #5C5850;     /* secondary text */
--text3: #9C9890;     /* tertiary / labels */
--border: #E8E4DC;
--accent: #7F77DD;    /* purple — close answers, streak active */
--correct: #1D9E75;   /* green — correct */
--close: #D4890A;     /* amber/gold — perfect answers, score pops */
--wrong: #E24B4A;     /* red — wrong answers */
--shadow-lg: 0 4px 24px rgba(0,0,0,0.08);
```

Dark mode overrides all tokens via `[data-theme="dark"]`.

---

## Flag Data Format

```js
{ code: "au", name: "Australia", alt: ["aussie"], continent: "Oceania" }
```

- `code` — ISO 3166-1 alpha-2, used for flag image URL: `https://flagcdn.com/w320/{code}.png`
- `alt` — accepted alternative spellings (exact match only, no fuzzy)
- `continent` — one of: Africa, Americas, Asia, Europe, Oceania

---

*Generated from claude.ai session — May 2026*
*Continue development in Claude Code. Paste this file at session start for full context.*
