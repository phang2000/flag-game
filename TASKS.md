# Flag Rush — Tasks
*Last updated: 2026-05-13*

---

## Verification Workflow (apply to every code fix)

1. Edit with `Edit` tool (one logical change per commit)
2. Sync worktree: `cp flagrush_v3.html .claude/worktrees/<name>/flagrush_v3.html`
3. Reload: `preview_eval: window.location.reload()`
4. Drive the relevant screen via `preview_click` / `preview_fill` / `preview_eval`
5. Screenshot: `preview_screenshot` (re-screenshot in dark mode for visual changes)
6. Console check: `preview_console_logs level:error` — zero errors required
7. Commit: `git add flagrush_v3.html && git commit -m "..."` (one fix per commit)
8. Smoke-test previous fix to confirm no regression

---

## In progress

- [ ] **Phase 3 Engagement** — active focus area

---

## Phase 3 — Engagement

- [ ] **Practice mode** ← **next priority** — drill missed flags using `flagrush_missed` localStorage (data already being collected). Build a drill screen. No new data infrastructure needed.
- [ ] Revisit skipped flags mid-run — loop back to skipped flags before results screen; decide if re-scoring is allowed and at what value
- [ ] Daily Challenge — streak counter (consecutive days played)
- [ ] Results screen celebration — animation on star reveal, personal best highlight

---

## Phase 4 — Growth (requires backend)

- [ ] Difficulty tiers — Easy (50 most common), Normal (193), Hard (obscure/similar)
- [ ] Shared leaderboard — replace localStorage so scores are visible globally across devices/players. Daily Challenge leaderboard comes with this.
- [ ] Social share card — v1 functional, needs visual polish (typography hierarchy, colour treatment, brand presence). Mock up before coding.
- [ ] Home screen full redesign — not launch-quality. Design project (mockup first), not an incremental code change.

---

## Territories Mode

- [ ] Design the territories flag dataset (Puerto Rico, Greenland, HK, Macau, Gibraltar, Faroe Islands, Bermuda, Guam, New Caledonia, French Polynesia, Aruba, Curaçao, Cayman Islands, Cook Islands + more)
- [ ] Add `type: "territory"` field to flag data format
- [ ] Build mode selector UI: Countries / Territories / Everything
- [ ] Implement separate leaderboards per mode
- [ ] Apply scaled star rating formula across all modes

---

## Code / Architecture

- [ ] Split `flagrush_v3.html` (~3,120 lines) into `index.html`, `style.css`, `flags.js`, `scoring.js`, `game.js` — recommended before Phase 3 complexity lands
- [ ] Delete `demo_animations.html` — animation choices are locked in, file is no longer needed
- [ ] Self-host flag images — currently relying on `flagcdn.com/w320/{code}.png`

---

## Deferred / Design needed

- [ ] **Uncapped speed bonus** — Peter wants bonus to approach infinity as time → 0. Needs dedicated design session: new formula, recalibrated star thresholds, new benchmarks. Do not implement without discussion.
- [ ] Share card visual polish — functional but rough. Needs design pass before wider launch.
- [ ] Wrong answer gives no closeness hint — Marcus flagged: "Not quite — keep trying" is silent on how far off you were. Could distinguish "really close" vs "not even close".
- [ ] No pace indicator during run — "Am I ahead of my PB right now?" Mid-run pressure hook.

---

## Done ✓

### Phase 1 — Core bugs
- [x] Stale timer interval on back-navigation
- [x] Close-match threshold on short names (scaled by target word length)
- [x] Hardcoded hex in theme toggle → CSS variables
- [x] Skip data not persisted → `missedFlags[]` array saved to localStorage
- [x] Wrong answers not resetting streak
- [x] Duplicate `revealedCount` declaration
- [x] Score not visible during gameplay → redesigned topbar
- [x] Score pop appearing top-right → now appends to `flag-img-wrap`
- [x] Close answer auto-advance → `awaitingAdvance` flag, Enter advances, diff highlight
- [x] Leaderboard ranking by score descending

### Phase 2 — UX polish
- [x] Custom quit dialog — replaces native `confirm()`
- [x] First-time onboarding hint — one-time modal, stored in `flagrush_onboarded`
- [x] Enter key on home screen starts game; inline nudge if no name entered
- [x] Skipped flags review on results screen
- [x] Score pop scales with multiplier tier (size + glow)
- [x] Streak tier transition animation (shimmer on tier-up, red flash + shake on break)
- [x] Auto-advance delay tightened to 250ms
- [x] Reveal auto-advances after 1.2s
- [x] Streak-break micro-signal fires on intact pill

### Daily Challenge (Phase 3 v1)
- [x] Date-seeded shuffle (mulberry32 PRNG) — same flag order for all players on same day
- [x] One attempt per day via `flagrush_daily_{YYYY-MM-DD}` localStorage key
- [x] Home screen tile: available (purple gradient) or locked (score + countdown)
- [x] Quit during daily consumes the attempt and saves partial score
- [x] Results screen daily pill

### Phase 2 panel audit (post-daily)
- [x] `executeQuit()` routes to home (not mode-select) — `1801833`
- [x] Quit-daily preserves partial score (not zero) — `cf30f8b`
- [x] Quit dialog copy softened — `b00bbf6`
- [x] `MAX_BONUS` scales with flag count — `7addb4a`
- [x] Daily runs excluded from all-flags leaderboard — `2826ff3`
- [x] Daily runs tagged in recent runs list (`⭐ Daily ·`) — `69f7557`
- [x] Share card shows daily date on canvas — `510efc5`
- [x] `score-pop-tier-2x` glow visible in light mode — `a7f58c4`
- [x] Skip exploit fixed — speed bonus × `correctCount / shuffled.length`
