# Flag Rush — Session Handover
*Rewrite this at the end of each session to brief the next one.*

---

## Last Session (May 2026)

Daily Challenge v1 shipped. Phase 2 panel audit bug-fix sprint complete. 9 commits pushed to `origin/main` (`052580e..5faf71b`). Vercel deploy live.

**What shipped:**
- Daily Challenge: seeded PRNG, one-attempt gate, home tile, results pill, share card date, partial quit score
- Panel bugs fixed: quit routing, MAX_BONUS scaling, leaderboard filter, recent runs tags, score-pop glow
- Docs restructured: CHANGELOG.md and HANDOVER.md split out from CLAUDE.md

---

## Start Here Next Session

1. **Mobile UX — keyboard/viewport fix** ← blocking usability issue on mobile. See "Mobile UX Issues" section below.
2. **Practice Mode** — `flagrush_missed` localStorage is already populated each run. Build a drill screen — no new data infrastructure needed. Entry point: "Practice missed flags" button on home or results screen.
3. **Share card visual polish** — functional (shows daily date, correct data). Needs: typography hierarchy, colour/gradient treatment, stronger brand. Mock up before coding.
4. **Home screen redesign** — not launch-quality. Design project, mockup first. Do not ship to a wider audience with the current home screen.
5. **Delete `demo_animations.html`** — animation choices are locked in. File is stale.
6. **`git push origin main`** after any session to keep Vercel in sync.

---

## Mobile UX Issues (logged May 2026 — fix next session)

Screenshots taken on iPhone in dark mode. Two issues:

### 1. Keyboard occludes game HUD ← HIGH PRIORITY
When the on-screen keyboard opens during a game, the score, flag count, and action buttons (Skip / Reveal / Submit) are pushed off-screen or partially hidden. The player can't see their score or progress while typing.

**Goal:** Everything critical must be visible with the keyboard open on common mobile viewports (iPhone SE, iPhone 14, mid-size Android).

**Approach to consider:**
- Compress the game topbar when keyboard is open (`visualViewport` resize event or CSS `dvh`/`svh` units)
- Reduce flag image height when viewport shrinks — flag can be smaller, HUD must not be
- Ensure `#answer-input` scrolls into view without pushing topbar off-screen
- Test at 375px width (iPhone SE) as the minimum target

### 2. Dark mode toggle placement — home screen & mode-select
- Too much dead space between the dark mode toggle (top-right) and the "FlagRush" h1
- Toggle feels intrusive on both the home screen and the continent picker overlay
- **Possible fix:** Move toggle into a less prominent position (e.g. footer of home card, or inside a settings area). Or reduce top padding so the h1 sits closer to the top of the viewport.

---

## Open Design Decisions

| Decision | Status | Notes |
|----------|--------|-------|
| Uncapped speed bonus (Peter's request) | Deferred | Needs new formula + recalibrated star thresholds + new benchmarks. Do not implement without a dedicated design session. |
| Wrong answer closeness hint (Marcus) | Deferred | Distinguish "really close" vs "not even close". Phase 3. |
| Pace indicator during run (Marcus) | Deferred | "Am I ahead of my PB right now?" mid-run pressure hook. Phase 3. |
| Continent mode entry point | Deferred | Tied to home screen redesign. |
| Territories mode dataset | Not started | Architecture decided (see CLAUDE.md). Need to build the flag list. |
