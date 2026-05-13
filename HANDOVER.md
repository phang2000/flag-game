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

1. **Practice Mode** ← highest priority. `flagrush_missed` localStorage is already populated each run. Build a drill screen — no new data infrastructure needed. Entry point: "Practice missed flags" button on home or results screen.
2. **Share card visual polish** — functional (shows daily date, correct data). Needs: typography hierarchy, colour/gradient treatment, stronger brand. Mock up before coding.
3. **Home screen redesign** — not launch-quality. Design project, mockup first. Do not ship to a wider audience with the current home screen.
4. **Delete `demo_animations.html`** — animation choices are locked in. File is stale.
5. **`git push origin main`** after any session to keep Vercel in sync.

---

## Open Design Decisions

| Decision | Status | Notes |
|----------|--------|-------|
| Uncapped speed bonus (Peter's request) | Deferred | Needs new formula + recalibrated star thresholds + new benchmarks. Do not implement without a dedicated design session. |
| Wrong answer closeness hint (Marcus) | Deferred | Distinguish "really close" vs "not even close". Phase 3. |
| Pace indicator during run (Marcus) | Deferred | "Am I ahead of my PB right now?" mid-run pressure hook. Phase 3. |
| Continent mode entry point | Deferred | Tied to home screen redesign. |
| Territories mode dataset | Not started | Architecture decided (see CLAUDE.md). Need to build the flag list. |
