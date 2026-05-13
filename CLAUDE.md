# Flag Rush — Project Reference

## Vision

Flag Rush is a single-page flag identification game with ambitions to become the **global benchmark for flag knowledge and speed**. The core design philosophy taps into speedrunning culture — the score ceiling is mathematically unreachable, so players always have something to chase. The goal is high UGC output on social media driven by the addictive grind for a better score.

**The three pillars of the grind:**
1. **Spelling quality** — perfect answers vs close answers vs revealed
2. **Streak consistency** — consecutive correct answers compound your multiplier
3. **Speed** — end-of-run bonus rewards faster completions

---

## Current State

- **Source file:** `flagrush_v3.html` — single self-contained HTML/CSS/JS file, ~3,120 lines
- **193 flags** covering all sovereign nations, grouped by continent
- **Three play modes:** All flags (193), by continent, or Daily Challenge (seeded, one attempt per day)
- **Persistence:** localStorage for leaderboard, recent runs, region bests, missed flags, username, daily entries
- **Theming:** Light/dark toggle, CSS custom properties throughout
- **Deployed:** Vercel via GitHub push to `main`. Latest commit: `5faf71b`
- **Demo file:** `demo_animations.html` — standalone animation sandbox, safe to delete

---

## Per-Fix Verification Workflow

Apply this workflow to **every code change** before committing:

1. **Edit** with `Edit` tool (one logical change per commit).
2. **Sync worktree** — `cp flagrush_v3.html .claude/worktrees/<name>/flagrush_v3.html` so the preview server picks up the change.
3. **Reload** — `preview_eval: window.location.reload()`.
4. **Drive the screen** — use `preview_click` / `preview_fill` / `preview_eval` to reach the relevant state.
5. **Screenshot** — `preview_screenshot`. Re-screenshot in dark mode for visual changes.
6. **Console check** — `preview_console_logs level:error`. Zero errors required.
7. **Commit** — `git add flagrush_v3.html && git commit -m "..."` (one fix per commit, Co-Authored-By footer).
8. **Smoke-test previous fix** — quick screenshot to confirm no regression.

---

## Scoring System (Locked — Do Not Change Without Discussion)

### Per-flag base points
| Result | Points |
|--------|--------|
| Perfect spelling, first attempt | 100 pts |
| Close spelling (within threshold) | 60 pts |
| Used reveal | 0 pts |
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

> ⚠️ **Under review:** Peter wants the bonus to be truly uncapped (approaching infinity as time → 0). Requires recalibrating star thresholds and benchmarks. **Do not implement without a dedicated design session.**

```js
function calcSpeedBonus(totalMs, flagCount) {
  const MAX_BONUS = Math.round(5000 * (flagCount / 193)); // scales with pool size
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
- Shown as a separate line item on results screen
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

5 stars requires near-perfect spelling + long streaks + fast time simultaneously. Peter's PB (~34k) is solid 4 stars. Only elite runs break 5 stars.

---

## Close Answer Matching

Levenshtein distance, **scaled by target word length** to prevent false positives on short names:

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
let currentMode = 'normal'; // 'normal' | 'daily'

// Timing
let timerInterval = null;   // always clearInterval this at top of startGame()
let startTime = 0;
let elapsedMs = 0;
let flagStartTime = 0;

// Counts
let correctCount = 0;   // ⚠️ incremented for BOTH perfect AND close — do not add closeCount on top
let skippedCount = 0;
let closeCount = 0;     // subset of correctCount — close answers only
let revealedCount = 0;
let perfectCount = 0;   // subset of correctCount — perfect answers only

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
let awaitingAdvance = false; // true after close answer — next Enter advances rather than submits

// Animation state
let _lastStreakTier = 0;    // 0=none, 1=1.2×, 2=1.5×, 3=2×
                             // ⚠️ capture BEFORE calling updateStreakHUD() if you need prev value
let revealAutoTimer = null; // setTimeout handle for reveal auto-advance — clear in skipFlag + startGame

// Share card (end of run)
let _lastRunScore = 0, _lastRunMs = 0, _lastRunStars = 0, _lastRunStreak = 0;
let _lastRunCorrect = 0, _lastRunTotal = 0, _lastRunContinent = null;
let _lastRunMode = 'normal', _lastRunDailyDate = null;
```

---

## localStorage Keys

| Key | Contents |
|-----|----------|
| `flagrush_leaderboard` | Array of `{name, timeMs, score}` — all-flags leaderboard (daily runs excluded) |
| `flagrush_runs` | Array of recent runs (max 20) with score, stars, continent, mode, date |
| `flagrush_region_bests` | Object keyed by continent with best time per region |
| `flagrush_missed` | Array of `{code, name, reason}` — accumulated missed flags for Practice mode |
| `flagrush_username` | Last used display name |
| `flagrush_theme` | `'light'` or `'dark'` |
| `flagrush_daily_{YYYY-MM-DD}` | Daily attempt result `{score, timeMs, stars, correct, total, quit}` |
| `flagrush_onboarded` | `true` once first-time onboarding modal has been shown |

---

## UI Architecture

### Screens (shown/hidden via `showScreen(id)`)
- `#home` — landing, username input, leaderboard, daily tile
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
Appended to `.flag-img-wrap`, rises from centre of flag. Size and glow scale with streak tier:
- `score-pop-tier-1x` — 1rem, no glow
- `score-pop-tier-12x` — 1.3rem
- `score-pop-tier-15x` — 1.65rem + soft glow
- `score-pop-tier-2x` — 2.1rem + bright glow (perfect: `#FF9F0A`, close: purple halo)

Gold (`var(--close)`) for perfect, purple (`var(--accent)`) for close.

### Streak HUD
Fixed height (`1.6rem`) — never shifts layout. Pill appears at tier 1+. Two animations:
- **Tier-up shimmer** — class `tier-up`, fires when `_lastStreakTier` increases (removed after 620ms)
- **Streak-break** — class `streak-break`. `signalStreakBreak()` fires on the intact pill. ⚠️ Always capture `_lastStreakTier` BEFORE calling `updateStreakHUD()`.

---

## Roadmap

### Phase 3 — Engagement
- [ ] **Practice mode** ← **next priority** — drill missed flags using `flagrush_missed` localStorage. Data already collected every run. No new data infrastructure needed.
- [ ] **Revisit skipped flags mid-run** — loop back before results screen. Decide if re-scoring is allowed and at what value.
- [ ] **Results screen celebration** — animation on star reveal, personal best highlight.
- [ ] **Pace indicator during run** — "Am I ahead of my PB?" mid-run pressure hook (Marcus).
- [ ] **Wrong answer closeness hint** — distinguish "really close" vs "not even close" (Marcus).

### Phase 4 — Growth (requires backend)
- [ ] **Difficulty tiers** — Easy (50 most common), Normal (193), Hard (obscure/similar).
- [ ] **Shared leaderboard** — replace localStorage so scores are visible globally.
- [ ] **Bot/automation vulnerability** — must address before shared leaderboard ships. Options: server-side answer validation, timing anomaly detection, lean into share cards.
- [ ] **Self-host flag images** — currently on `flagcdn.com/w320/{code}.png`. Bundle for launch.
- [ ] **Share card visual polish** — functional but rough. Needs typography hierarchy, colour treatment, brand presence. Mock up before coding.
- [ ] **Share card v2 (server-rendered)** — Vercel/Cloudflare OG image generation. Requires backend.
- [ ] **Home screen full redesign** — not launch-quality. Design project, mockup first. Do not ship to a wider audience with current home screen.

### Code / Architecture
- [ ] **Split `flagrush_v3.html`** (~3,120 lines) into `index.html`, `style.css`, `flags.js`, `scoring.js`, `game.js` — recommended before Phase 3 complexity lands.
- [ ] **Delete `demo_animations.html`** — animation choices locked in, file is no longer needed.

### Design / Balance (needs dedicated session before touching)
- [ ] **Uncapped speed bonus** — Peter wants bonus approaching infinity as time → 0. New formula + recalibrated star thresholds + new benchmarks required.
- [ ] **Continent mode entry point** — deferred to home screen redesign.

---

## Territories Mode — Planned Feature

### Three game modes
| Mode | Pool | Notes |
|------|------|-------|
| **Countries** | 193 sovereign nations | Current default |
| **Territories** | Dependencies, overseas territories, autonomous regions | New |
| **Everything** | Countries + Territories combined | New |

### Design decisions (locked)
- **Palestine** → Countries pool. ISO code `ps`.
- **Territories scope** — inclusive. Puerto Rico, Greenland, Hong Kong, Macau, Gibraltar, Faroe Islands, Bermuda, Guam, New Caledonia, French Polynesia, Aruba, Curaçao, Cayman Islands, Cook Islands, and more. Exact list TBD.
- **Separate leaderboard per mode** — Countries, Territories, and Everything do not share leaderboards.

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
```js
{ code: "pr", name: "Puerto Rico", alt: [], continent: "Americas", type: "territory" }
```
`type: "country"` is the default and can be omitted for existing entries.

### localStorage keys to add
| Key | Contents |
|-----|----------|
| `flagrush_lb_countries` | Countries mode leaderboard |
| `flagrush_lb_territories` | Territories mode leaderboard |
| `flagrush_lb_everything` | Everything mode leaderboard |
| `flagrush_lb_continent_{name}` | Per-continent leaderboard |

---

## Expert Panel

Five reviewers for playtesting, feedback, and design decisions. Invoke the full panel for major decisions; call specific members when their domain is most relevant. Panel members disagree — that friction is the point. Panel runs code simulations against the actual scoring engine (Node.js), not assumptions.

### When to invoke whom

| Situation | Who to call |
|-----------|-------------|
| Scoring system changes, mechanic design | Jordan, Sam, Marcus |
| Prioritisation / roadmap decisions | Jordan, Priya |
| New feature UX or player experience | Aisha, Marcus |
| Bug triage, edge cases, code correctness | Sam |
| Retention, social, growth features | Priya, Aisha |
| Exploit or balance issue | Sam, Jordan |
| "Would this feel good to play?" | Marcus, Aisha |
| "Should we ship this now or later?" | Jordan, Priya (expect disagreement) |

### Jordan M. — Game Advisor
**Bias:** Ship it, iterate. Thinks in long arcs — how mechanics compound over 100 sessions. More interested in elegance than edge cases. Pushes back on polish while Phase 3 features sit unbuilt.
**Key question:** "Would a player *feel* this mechanic, or just read it on a stats screen?"
**Conflicts with:** Sam (Jordan ships around bugs Sam considers blockers). Priya (right design vs. retaining players).

### Sam R. — QA Tester
**Bias:** Things break. Highest "shippable" threshold on the panel. Writes precise numbered bug reports with exact reproduction inputs.
**Key question:** "What's the specific input that breaks this?"
**Conflicts with:** Jordan (Sam flags blockers Jordan would ship around).

### Aisha K. — Analyst
**Bias:** The player who isn't Peter. Advocates for casual players, first-timers, people who skip 40 flags. Spots emotional friction — moments that make players feel stupid rather than challenged.
**Key question:** "How does this feel for someone who isn't an expert?"
**Conflicts with:** Jordan when "ship it" means shipping something confusing. Priya when retention optimisation hurts player experience.

### Marcus T. — Competitive Player
**Bias:** Game feel above all. Hundreds of hours on TypeRacer, GeoGuessr, Sporcle. Thinks about moment-to-moment satisfaction — the tactile feel of a correct answer, the anxiety of a long streak.
**Key question:** "Can a player feel themselves getting better at this?"
**Conflicts with:** Jordan (elegant vs. fun to master). Aisha (competitive depth vs. casual accessibility).

### Priya S. — Growth & Retention
**Bias:** Does this bring someone back tomorrow? Mobile games background, thinks in D1/D7/D30 metrics. More mercenary than Aisha — less interested in how a feature feels, more in whether it creates a habit.
**Key question:** "What's the D7 retention argument for building this?"
**Conflicts with:** Jordan (retention vs. right design). Aisha (retention optimisation ≠ player experience).

---

## Key Functions Reference

### `showScorePop(pts, type)`
Reads `currentStreak` to pick tier class. `type` is `'perfect'` or `'close'`.

### `updateStreakHUD()`
Updates HUD text + class. Fires `tier-up` shimmer when `_lastStreakTier` increases. Updates `_lastStreakTier`. ⚠️ Call sites that need prev tier must capture `_lastStreakTier` BEFORE calling this.

### `signalStreakBreak()`
**No parameter.** Fires red flash + shake on the intact pill — call BEFORE `updateStreakHUD()` clears classes. Internally defers `updateStreakHUD()` to 460ms so pill fades with background. Set `currentStreak = 0` and `_lastStreakTier = 0` first, then call `signalStreakBreak()` if `hadStreak > 0`. ⚠️ Do NOT call `updateStreakHUD()` separately if you already called `signalStreakBreak()` — it will double-fire.

### `revealFlag()`
Sets `awaitingAdvance = true`, starts `revealAutoTimer` (1200ms). Enter advances early. `skipFlag()` and `startGame()` both clear `revealAutoTimer`.

### `startGame(continent)`
Must reset: `currentStreak = 0`, `bestStreak = 0`, `perfectCount = 0`, `_lastStreakTier = 0`, `clearTimeout(revealAutoTimer)`, `updateStreakHUD()`. All currently present.

---

## Benchmark Scores (for balance testing)

| Scenario | Flag score | Speed bonus | Final | Stars |
|----------|-----------|-------------|-------|-------|
| Beginner — 35 min, mostly close/skips | ~8,400 | 0 | ~8,400 | ⭐⭐ |
| Intermediate — 25 min, mixed | ~13,000 | 0 | ~13,000 | ⭐⭐ |
| Peter (expert) — 14 min, 10 skips, 25 close | ~33,400 | ~773 | ~34,200 | ⭐⭐⭐⭐ |
| Elite — all perfect, 8 min | ~37,000 | ~2,000 | ~39,000 | ⭐⭐⭐⭐⭐ |
| Theoretical max — all perfect, 1 min | ~38,600 | ~4,600 | ~43,200 | ⭐⭐⭐⭐⭐ (unreachable) |

Peter's personal best: **14 minutes** on all-193 flags → ~34,200 pts.

---

## File Structure

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

*See also: [CHANGELOG.md](CHANGELOG.md) — full fix history | [HANDOVER.md](HANDOVER.md) — current session priorities*
