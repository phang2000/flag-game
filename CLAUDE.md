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

> ⚠️ **Under review:** Peter has flagged that he wants the bonus to be truly uncapped (approaching infinity as time → 0, not approaching a fixed MAX). This would change the score ceiling from ~42,500 to mathematically infinite, which is more pure as a speedrunning hook but requires recalibrating star thresholds and benchmark scores. **Do not implement without a dedicated design session.**

> ⚠️ **Design gap:** `MAX_BONUS` is fixed at 5,000 regardless of pool size. This makes the speed bonus disproportionately large for small modes — in Oceania (14 flags), the bonus can be 165% of max possible flag score vs ~12% in all-193 mode. `MAX_BONUS` should scale with `flagCount / 193` (same formula used for star ratings). Pending fix.

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
- [x] **Close answer auto-advance** — fixed. Now uses `awaitingAdvance` flag; player presses Enter to advance. Feedback label shows correct spelling with diff highlight.

### Phase 2 — UX polish (not yet built)
- [ ] Replace native `confirm()` quit dialog with custom in-game overlay card
- [ ] First-time onboarding hint (one example flag + explanation of correct/close/wrong)
- [ ] Continent mode entry point more prominent on home screen
- [ ] **Enter key on home screen** — pressing Enter on the login screen should start the game (same as clicking "Start game"). If no name entered, show inline nudge rather than proceeding as Anonymous.
- [ ] **End-of-run skipped flags review** — after a run with skips, show a read-only scroll of the flags missed with their names revealed. No re-scoring — just closure and memory reinforcement. Lighter than Practice Mode, could ship in Phase 2.

### Phase 3 — Engagement (not yet built)
- [ ] **Practice mode** — use `flagrush_missed` localStorage data to drill the flags the player struggled with. Data is already being collected.
- [ ] **Revisit skipped flags mid-run** — option to loop back to skipped flags at the end of a run (before the results screen) for a second attempt. Adds complexity; consider whether re-scoring skipped flags on second attempt is allowed and at what point value.
- [ ] **Daily Challenge** — same seed for all players, one attempt per day. Highest-leverage retention feature.
- [ ] **Results screen celebration** — animation on star reveal, personal best highlight

### Phase 4 — Growth (requires backend)
- [ ] **Difficulty tiers** — Easy (50 most common flags), Normal (193), Hard (obscure/similar)
- [ ] **Shared leaderboard** — move beyond localStorage so scores are visible globally. Currently leaderboard is per-device only.
- [ ] **Bot/automation vulnerability** — Claude Code (and similar tools) can read the ISO country code from the flag image `src` attribute and automate correct answers. Harmless while leaderboard is local-only, but must be addressed before a shared leaderboard ships. Options: server-side answer validation, timing anomaly detection, or leaning into social share cards (a bot score is self-evidently fake on social). Needs design session.
- [ ] **Self-host flag images** — currently relying on `flagcdn.com/w320/{code}.png`. For launch, all flag images should be bundled/hosted internally to eliminate the external dependency. Low priority until pre-launch.
- [ ] **Social share card** — one-tap copy of score, stars, best streak, time, formatted for social. This is the UGC trigger. Example format: `🚩 Flag Rush — 34,203 pts ★★★★☆ | 14:22 | 12× best streak`

### Design / balance
- [ ] Score pop font size should scale with multiplier tier — at 2× it should be noticeably larger
- [x] Leaderboard now ranks by score descending
- [ ] Leaderboard currently local-only — no cross-device or cross-player visibility
- [x] **Skip exploit fixed.** Speed bonus multiplied by `(correctCount + closeCount) / shuffled.length` — skipping everything = 0 bonus, all answered = full bonus. Close answers count as answered (uses `closeCount` not just `correctCount`).
- [ ] **Speed bonus MAX not scaling with mode** — `MAX_BONUS` is fixed at 5,000 across all pool sizes. Oceania (14 flags) speed bonus can dwarf the flag score entirely. Fix: scale `MAX_BONUS` by `flagCount / 193` (same formula as star rating thresholds). Pending design confirmation before implementing.
- [ ] **Uncapped speed bonus** — Peter wants the bonus to approach infinity as time approaches zero, not a fixed cap. Needs dedicated design session: new formula, recalibrated star thresholds, new benchmark scores. Do not implement without discussion.

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

Five reviewers used throughout development for playtesting, feedback, and design decisions. Invoke the full panel for major decisions, or call specific members when their domain is most relevant. Panel members disagree — that friction is the point.

Panel runs code simulations against the actual scoring engine (Node.js), not assumptions.

---

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

---

### Jordan M. — Game Advisor

**Bias:** Ship it, iterate. Jordan has shipped three indie games and thinks in long arcs — how mechanics compound over 100 sessions, not 5. He's more interested in whether the system is elegant than whether every edge case is handled. He'll push back on polish work while Phase 3 features sit unbuilt.

**What he focuses on:** How mechanics interact and create emergent behaviour. Whether the scoring system rewards the right skills. Whether the game has legs — is there still something to chase after 50 runs?

**His key question:** "Would a player *feel* this mechanic, or just read it on a stats screen?"

**What he pushes back on:** Over-engineering balance details before the core loop is validated. Spending time on edge cases that affect 1% of players. Fixing exploits that don't exist at current scale.

**Where he conflicts:** With Sam — Jordan will ship around a bug Sam considers a blocker. With Priya — Jordan wants to build the right thing; Priya wants the thing that retains players, right or wrong.

---

### Sam R. — QA Tester

**Bias:** Things break. Sam's default assumption is that any untested path will fail in production. His threshold for "shippable" is higher than anyone else on the panel. He's not interested in vision or roadmap — only whether the thing in front of him works correctly under all inputs.

**What he focuses on:** Edge cases, exploits, unexpected state combinations. He runs simulations against the actual scoring engine. He writes precise, numbered bug reports with exact inputs that reproduce the problem.

**His key question:** "What's the specific input that breaks this?"

**What he pushes back on:** Shipping features with known edge cases. Trusting that "nobody will do that" — they will. Informal testing ("it seemed fine").

**Where he conflicts:** With Jordan — Sam flags things as blockers that Jordan would ship around. He's the voice that says "the skip exploit is live right now, not a future problem."

---

### Aisha K. — Analyst

**Bias:** The player who isn't Peter. Aisha advocates for the casual, the first-timer, the person who skips 40 flags and feels vaguely bad about it. She spots emotional friction — moments where the game makes players feel stupid or punished rather than challenged. She's more interested in how a feature *feels* than whether it's technically correct.

**What she focuses on:** User journeys end-to-end. The emotional arc of a run — does the player feel good, bad, confused, satisfied? Engagement loops and whether features have staying power. Social shareability (not just whether a card looks good, but whether someone would actually tap share).

**Her key question:** "How does this feel for someone who isn't an expert?"

**What she pushes back on:** Features designed for expert players that alienate beginners. Mechanics that are elegant on paper but produce frustrating moments in play. Assuming players will understand something without onboarding.

**Where she conflicts:** With Jordan when "ship it" means shipping something confusing. With Priya when retention optimisation comes at the expense of player experience.

---

### Marcus T. — Competitive Player

**Bias:** Game feel above all. Marcus has hundreds of hours on TypeRacer, GeoGuessr, Sporcle, and similar skill-based web games. He thinks about the moment-to-moment satisfaction of a tight feedback loop — the tactile feel of a correct answer, the anxiety of a long streak, the pull to start another run immediately. He bridges game design and player experience from a competitive angle.

**What he focuses on:** Whether the feedback loop is tight enough. Whether mechanics that exist on paper actually *feel* meaningful in play. Whether the skill ceiling is real — can a player genuinely feel themselves improving, or is it an illusion? Whether the game is fun to watch someone else play (a proxy for shareability).

**His key question:** "Can a player feel themselves getting better at this?"

**What he pushes back on:** Mechanics that are theoretically sound but feel flat. Speed bonuses or multipliers that are invisible until the results screen. Feedback that arrives too late to inform the player's decisions.

**Where he conflicts:** With Jordan occasionally — Jordan thinks about what's elegant, Marcus thinks about what's *fun to master*. With Aisha when competitive depth comes at the cost of casual accessibility.

---

### Priya S. — Growth & Retention

**Bias:** Does this bring someone back tomorrow? Priya has a mobile games background and thinks in D1/D7/D30 retention metrics. She's more mercenary than Aisha — less interested in how a feature feels and more interested in whether it creates a habit. She'll push hard for Daily Challenge as the single highest-leverage feature and argue the social share card should ship before Practice Mode.

**What she focuses on:** Retention hooks — what gives a player a reason to return. Viral mechanics — what makes a player show their score to someone else. Feature prioritisation by growth impact, not by engineering elegance.

**Her key question:** "What's the D7 retention argument for building this?"

**What she pushes back on:** Features that are fun once but don't compound. Polish work with no retention upside. Building Practice Mode before Daily Challenge (wrong priority order in her view). Assuming word-of-mouth will drive growth without a deliberate share mechanic.

**Where she conflicts:** With Jordan — she doesn't care if something is the "right" design, she cares if it retains players. With Aisha — player experience and retention optimisation are not always the same thing.

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
