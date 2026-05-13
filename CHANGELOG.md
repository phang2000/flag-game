# Flag Rush — Changelog
*Historical record of fixes, implementations, and session notes.*

---

## Phase 2 — Post-Daily Panel Audit Bug-Fix Sprint (May 2026)

9 commits. All fixes verified in-browser (screenshot + zero console errors) before commit.

| Bug / Polish | Fix | Commit |
|---|---|---|
| `executeQuit()` routed to mode-select | Changed to `showScreen('home')` so locked daily tile is immediately visible | `1801833` |
| Quit-daily saved `score: 0` | Now saves `currentScore` (partial flag score, no speed bonus) + correct stars | `cf30f8b` |
| Daily quit dialog copy was punitive | Softened to "Heads up — quitting ends today's Daily run. Your progress so far will be saved." | `b00bbf6` |
| `MAX_BONUS` fixed at 5000 for all modes | Now `Math.round(5000 * flagCount / 193)` — Oceania caps at ~363 | `7addb4a` |
| Daily runs polluted all-flags leaderboard | `endGame()` skips leaderboard push when `currentMode === 'daily'` | `2826ff3` |
| Recent runs list had no daily indicator | Daily rows now prefixed `⭐ Daily ·`; continent rows prefixed with continent name | `69f7557` |
| Share card didn't show daily context | Mode label in canvas now renders `⭐ Daily · Wed, 13 May` for daily runs | `510efc5` |
| `score-pop-tier-2x` glow invisible in light mode | Increased opacity + radius; added white backing shadow for contrast independence | `a7f58c4` |
| Reveal scoring spec mismatch | Spec updated: reveals = 0 pts (matches code). No "type after reveal for 20 pts" path. | — |
| Docs: panel audit findings + verification workflow | CLAUDE.md + TASKS.md updated | `5faf71b` |

---

## Daily Challenge v1 (May 2026)

- `mulberry32` PRNG seeded from `YYYY-MM-DD` string — same flag order for all players on same day.
- One attempt per day via `flagrush_daily_{date}` localStorage key.
- Home screen tile: available (purple gradient) or locked (score + countdown).
- Quit during daily consumes attempt; partial score saved (not zero).
- Results screen daily pill; share card shows daily date.
- Daily runs excluded from all-flags leaderboard; tagged `⭐ Daily ·` in recent runs list.

---

## Phase 2 — UX Polish (completed)

- Custom quit dialog — replaces native `confirm()`. Timer pauses; resumes on "Keep playing".
- First-time onboarding modal — colour-coded rows (Perfect/Close/Revealed), stored in `flagrush_onboarded`.
- Enter on home screen starts game; inline nudge if no name entered.
- Skipped flags review on results screen — scrollable list, max-height 280px, SKIPPED/REVEALED badges.
- Score pop scales with streak tier — size + glow at 1×/1.2×/1.5×/2×.
- Streak tier transition animation — shimmer on tier-up, red flash + shake on break.
- Auto-advance delay tightened to 250ms (was 400ms).
- Reveal auto-advances after 1.2s. Enter skips the wait.
- Streak-break micro-signal fires on intact pill (not empty HUD).
- Close answer auto-advance — `awaitingAdvance` flag; Enter advances; diff highlight on feedback label.
- Leaderboard ranks by score descending.
- Skip exploit fixed — speed bonus × `correctCount / shuffled.length`.

---

## Marcus Playtest Review (May 2026)

Marcus did a full All-Flags competitive run after the Phase 2 animation changes.

### Signed off
- Score pop scaling: *"That lands. I can feel the difference between 1× and 2× now."*
- Streak break shake: *"Good — I felt every one I dropped."*
- Reveal auto-advance: *"Fixed. No more sitting there waiting."*
- Overall: *"The animations are doing real informational work now, not just decoration."*

### Still open (tracked in Roadmap)
| Issue | Priority | Notes |
|-------|----------|-------|
| Wrong answer shakes input AND HUD simultaneously | Low | Two things shaking at once. Not bad, slightly noisy. Worth monitoring. |
| No pace indicator during run | Phase 3 | "Am I ahead of my PB right now?" — mid-run pressure hook. High ceiling on competitive experience. |
| Wrong answer gives no closeness hint | Phase 3 | "Not quite — keep trying" is silent on how far off you were. Could distinguish "really close" vs "not even close". |

---

## Streak-Break Animation — Implementation Notes (May 2026)

Resolved a sequencing issue where `updateStreakHUD()` was stripping pill classes before the animation could fire.

**Architecture:**
1. Call site captures `hadStreak = currentStreak`
2. Sets `currentStreak = 0` and `_lastStreakTier = 0`
3. If `hadStreak > 0`: calls `signalStreakBreak()` only — it owns the full lifecycle
4. If `hadStreak === 0`: calls `updateStreakHUD()` directly

**Inside `signalStreakBreak()`:**
- Fires `streak-break` CSS class immediately on the intact pill (classes + text still present)
- Defers `updateStreakHUD()` to a 460ms `setTimeout` — clears the DOM after animation ends
- `@keyframes streakBreak` fades `color` to `transparent` so text disappears with the pill background simultaneously (not after a DOM update delay)

```css
@keyframes streakBreak {
  /* no 0% — interpolates from pill's current colour */
  30%  { color: var(--wrong); background-color: rgba(226,75,74,0.22); }
  100% { color: transparent; background-color: transparent; }
}
.streak-hud.streak-break {
  animation: streakBreak 0.45s ease forwards, shake 0.35s ease;
}
```

**Callers:** `triggerWrong()`, `skipFlag()`, `revealFlag()` (!wasClose branch). All follow the same pattern.

---

## Phase 1 — Core Bugs Fixed

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
