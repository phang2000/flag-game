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

**Key design decision:** Auto-focus `#answer-input` at the start of `startGame()` so the keyboard is already open when the first flag appears. Player never sees the layout shift, and no time is lost tapping the field. Timer starts; keyboard is already up. Single line fix: `document.getElementById('answer-input').focus()` — call it after the first flag renders.

**`visualViewport` resize still needed** as a graceful recovery — players will dismiss/re-open the keyboard (autocorrect, distraction). But it's a fallback, not the primary experience.

**Full implementation plan:**
1. `startGame()` — call `answer-input.focus()` after `showFlag()` so keyboard opens immediately
2. `syncGameHeight()` — hook `visualViewport.resize` + `scroll` events, write `--game-h` CSS var
3. `#game { height: var(--game-h, 100dvh); overflow: hidden; }` — container tracks real visible height
4. `#flag-img-wrap` — set `flex: 1; min-height: 80px; max-height: 240px` — flag is the sacrificial flex element, everything else is fixed height
5. `#game { transition: height 0.15s ease; }` — smooth resize when keyboard dismissed/re-opened
6. Clean up `visualViewport` listener in `showScreen()` when leaving `#game` (Sam's note)
7. Test viewports: 375×667 (iPhone SE), 390×844 (iPhone 14), 360px wide (Android mid)

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

---

## Food for Thought (logged May 2026 — discuss next session)

### 1. Accepted name review — abbreviations & shortened names
Currently some flags accept informal names via the `alt[]` array (e.g. "NZ" for New Zealand, "PNG" for Papua New Guinea, "Macedonia" for North Macedonia, "Congo" for DRC/Republic of Congo). Need a full audit and a consistent policy:
- Should abbreviations (NZ, PNG, UAE, USA, UK) be accepted? They're fast to type but may undermine the spelling challenge.
- Ambiguous shortened names (Congo = DRC or Republic of Congo?) need a decision — accept both for one, neither, or require disambiguation.
- Invoke panel for this: Sam (edge cases), Jordan (does accepting shortcuts cheapen the skill?), Marcus (competitive feel).

### 2. Mode expansion
- **Split Americas into North America + South America** — current "Americas" pool mixes both. Splitting gives more granular continent runs and cleaner leaderboards. Check flag counts: ~23 North, ~12 South roughly.
- **Territories mode** — architecture already decided in CLAUDE.md. Build the flag dataset (Puerto Rico, Greenland, HK, Macau, Gibraltar, Faroe Islands, Bermuda, Guam, New Caledonia, French Polynesia, Aruba, Curaçao, Cayman Islands, Cook Islands etc.). Separate leaderboard per mode.
- Both changes touch mode-select UI, leaderboard keys, and `startGame()` routing — plan carefully before implementing.

### 3. Results screen — % complete stat & animated stat cards
- **% complete** — show what % of the total pool the player answered correctly (e.g. "You named 87% of all flags"). More intuitive than raw correct/total for casual players.
- **Animated stat cards** — currently stats appear statically. Consider counting-up animations on the numbers (score, streak, correct count) as they reveal on results screen. Adds a reward moment.
- Could also explore a "personal progress" arc — showing this run vs. your personal best visually rather than just numbers.
- Discuss with Aisha (casual feel) + Marcus (competitive satisfaction) before building.
