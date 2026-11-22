# DESIGN.md: Kindergarten Math Exercises

**Status: mandatory for the pages in scope (below).** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119. If this file conflicts with your instincts, follow this file. If it conflicts with what a real 5-year-old does on a real device, the child wins. Update this file in the same PR as any convention change.

## 0. Scope and how to read this

- **In scope:** `equal-groups.html` ("Share the Cookies") and every NEW exercise for children aged 4 to 6 (pre-readers, finger on iPad/iPhone, often unsupervised).
- **Out of scope (legacy):** `number-rods.html`, `place-value-blocks.html`, `missing-number-rods.html` are for older children. They are exempt from everything here except §13 (integration). Do not convert them unless asked.
- **Structure:** Part A (§1-§9) are *invariants* for every kindergarten exercise. Part B (§10) is the *pattern* used by Equal Groups (distribute items into targets) and §11 says how to adapt for other exercise shapes. §12 holds the constants. §13-§16 are process.
- **Reference implementation:** `equal-groups.html`. Identifiers in backticks (`Speaker`, `layoutPile`, `move`, ...) are functions or constants in that file. Copy its patterns, **except** where §15 lists a deviation. The rationale comes from the usability review that produced PR #6 (rewrite of Equal Groups).
- **Verification honesty:** do NOT tick a checklist item you did not execute. If you cannot run a browser or a device, say so in the PR (§14).

---

# Part A: Invariants (all kindergarten exercises)

## 1. Core thesis

1. **A story the child owns** (sharing cookies fairly with animal friends). The math is the story's goal, never an abstract statement like "3 groups of ?".
2. **One task per screen.** One obvious thing to do. No side panels, no settings, no mode switches.
3. **Every instruction is spoken AND pictured** (halo, pointing hand, ghost slots). A child who cannot hear or understand the words can still play.
4. **Words are never required.** Buttons are icons/emoji. Numerals 0-20 are *content*, not text, and MAY be used as labels and answer choices (matching numeral to quantity is a kindergarten goal). Any numeral the child must act on is also spoken when tapped. Word labels MUST NOT appear in the play UI (`aria-label` is fine).
5. **No Check button.** The activity detects completion itself and celebrates.
6. **No scores, failure counters, countdowns or speed pressure.** Idle-hint timers (§6) are allowed. Progress is shown only as star dots.
7. **Everything is reversible** with one tap.

## 2. Plan first (required before code)

Write this short plan in the PR description (or chat) and show it before building:

```
Story / goal in child words (<= 8 words, spoken)
Math concept
Objects (item emoji, target emoji, mascot)
Interaction: the ONE state transition and how tap / drag / tap-select map to it
Completion rule (auto-detected)
Not-yet-done states and how each is shown positively
Hints: what L1 / L2 / L3 look like for THIS activity
Levels: parameters per level, 3 rounds each, scaffolds per level
COPY: every spoken line (functions for lines with numbers)
Sizes: max items per round, slot grid
Exercise shape: distribute (Part B) or other (§11)?
```

## 3. Layout and sizing

- **Shell (copy from `equal-groups.html`):** viewport `width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no` (the last only suppresses double-tap zoom; iOS ignores it for pinch, which is accepted). `body{position:fixed; inset:0; overflow:hidden; overscroll-behavior:none; touch-action:manipulation; user-select:none; -webkit-touch-callout:none}`, tap highlight off, context menu blocked. `#app` is a CSS grid with `height:100vh; height:100dvh;` and padding that includes **all four** safe-area insets (`max(12px, env(safe-area-inset-left/right))`, `env(safe-area-inset-top/bottom)`). Font stack `"Nunito","Avenir Next","Trebuchet MS",system-ui,sans-serif`; no web fonts.
- **Supported browsers:** iOS/iPadOS 16.2+ Safari. Write a plain-color declaration immediately before every `color-mix()` so older engines degrade. Use plain `var`/`function` or modern syntax consistently; do not assume pre-2022 Safari.
- **Regions:** portrait = top bar, flexible target area (`1fr`), docked bottom tray (`align-self:end`, never scrolls). Landscape (`@media (orientation:landscape) and (min-aspect-ratio:23/20)`) = tray becomes the left column. `isLandscape()` MUST use `>=` and the same 1.15 ratio.
- **The page never scrolls** at any size. If content does not fit, compute smaller sizes down to the floors in §12, then use more rows, then reduce the item count. Never scroll.
- **Sizes are computed in JS** from measured containers (copy `layoutPile` / `renderFriends`): subtract every padding and border in the chain (e.g. the tray's `clientWidth - 20`), clamp between the §12 floor and ceiling, reduce columns (more rows) before going below a floor, and write `gridTemplateColumns`/`gridAutoRows` in px. Re-lay-out on `resize` (debounced 120 ms), never during a drag.
- **Colors:** palette variables in `:root` (`--bg --panel --ink --muted --line --accent --good --gold --gold-dark --sky`). There is no red and no `--bad`. Off-palette literals are limited to fallbacks and shadows.

## 4. Interaction invariants

- **One state-transition function** (`move(id, from, to, srcRect)`) is the only code that changes placement state. Tap, drag and tap-select all call it. DOM is derived from state (`render()` is idempotent).
- **Inputs:** tap a target to deliver one item; drag (10 px threshold, item floats 48 px above the finger, drop on the *nearest* target within 140 px using a 24 px inflated rect with distance x0.3: `friendNear`); tap an item then tap a target (selected item glows). Tap a placed item to return it. A drop on nothing returns the item to where it was, silently, with no message.
- **Dropping an item on the card it already sits on MUST still `render(); setTimers();`** (see `move`).
- **Auto-snap** into the next free slot; the child never positions precisely.
- **Stable layout:** the pile has one cell per item id (`renderPile`); items never reflow when a neighbor leaves. Items carry `data-id`/`data-from`.
- **Touch robustness (all MUST):**
  - Single active pointer (`if (mode !== 'deal' || drag) return`); ignore events whose `pointerId !== drag.pid`.
  - Palm rejection `e.pointerType==='touch' && e.width>60` is **best-effort** (iOS may not report a real width); single-pointer locking is the real protection.
  - `setPointerCapture` in try/catch, but **attach `pointermove/pointerup/pointercancel` to `window`**, filtered by `pointerId`, never to the item: `render()` recreates items and would strand the drag.
  - **Any code that changes state or re-renders while `drag` is set MUST call `cancelDrag()` first** (`evaluate`, `startRound`, `startPile`, `levelDone`, `autoDeal`). The capture-phase `document` `pointerdown` listener MUST NOT re-render when the touch is on an item.
  - Cancel safely on `pointercancel`, `visibilitychange` (hidden), `orientationchange`.
  - Click suppression: ignore target clicks within 350 ms of `lastDragEnd`; items `stopPropagation` on click.
  - `touch-action:none` on items; `e.preventDefault()` in `onDown`.
- **FLIP landing:** after `render()`, animate the moved element from its old rect (`translate(old-new) scale(1.2)` to `none`, 350 ms ease-out). Skip under reduced motion.

## 5. Feedback (positive only)

| State | Visual | Sound / speech |
|---|---|---|
| Item placed | face `munch`, badge `pop` | `Sfx.pop()` + count aloud |
| Item returned | FLIP glide | `Sfx.boing()` (soft) |
| Needs more | amber dashed ghost slot + `pulse` + 👆 | hint speech only |
| Too many (uneven) | gentle `wobble` (±2°) | "Hmm, let's make it fair!" |
| Success | green borders, badges, star dot, mascot 🤗, confetti, `Sfx.chime()` | praise line (`COPY.fair`) |
| Level complete | overlay: mascot, ⭐⭐⭐, big ▶ | chime + "You did it!" |

- MUST NOT use red, ❌, ✖, shake, buzzers, or the words wrong / no / try again / missed, or tally mistakes. No `--bad`.
- Color is never the only signal: pair with shape (dashed ghost), icon (👆, ⭐) and motion, **and under reduced motion with a static marker** (`.wobble` gets a dashed `--accent` outline; the "needs/next" ring is `--gold-dark`, at least 3:1 against `--bg`).
- **Reduced motion** (`REDUCED` + the `@media` block): no looping/bouncy animations, no confetti, no ghost flyer, no FLIP; static rings instead. Sound and speech still play.
- **Confetti** is 24 emoji and finishes within 1.6 s total (duration 1000-1250 ms + delay up to 200 ms in the reference).
- Evaluate completion only when nothing is left to place, after an 800 ms grace (`setTimeout(evaluate, 800)`). An uneven result raises the hint to level 2; it never announces a verdict. After success hold 3.2 s, then advance.

## 6. Hints: wordless, escalating, mostly automatic

Timings are `HINT_MS = [8000, 16000, 30000]` (idle). Levels:

| Level | Picture | Speech |
|---|---|---|
| 1 | halo + 👆 on the **neediest** target (`neediest()`) | "Who needs a cookie?" |
| 2 | ghost item flies to the neediest target (`flyGhost`); ghost slots show the target count | "Give a cookie to this friend." |
| 3 | demo (`autoDeal`): "Watch me!", after 1.2 s one item every 900 ms with counting aloud | "Watch me!" |

- **The L3 demo shows ONE pass only** (at most one item per target), then stops, resets timers and hands control back. It never finishes the round on its own (so the star is the child's).
- **Uneven state** (nothing left in the pile): L1 = wobble + 👆 on the neediest; L2 = the ghost flies from the **fullest** target to the neediest; L3 = the demo moves exactly **one** item fullest to neediest, then hands back.
- `autoDeal` MUST keep its start delay in `demoT` and `stopAuto()` MUST clear both `demoT` and `autoTimer`; `escalate` MUST return when `autoTimer || demoT` is set. Any touch cancels the demo (capture-phase listener).
- **Hints fade:** after *any* placement (including a return) `hintLevel` drops by one (0 after a demo).
- **Mascot button** (`#bobo`): same place always (top bar, far right), at least 72 px, idle bounce; a tap raises the level one step; outside `deal` mode it repeats `Speaker.last`. Round 1 of level 1 starts at hint level 1.
- Hints are never scored or counted.

## 7. Audio and speech

- **`Speaker`** (copy it): Web Speech, `en-US`, rate 0.85, pitch 1.15. Voice order: a preferred name (`Samantha|Ava|Karen|Allison|Susan|Zira`) that is en-US, else any en-US, else a preferred name, else any `en`; re-pick on `voiceschanged`. Numbers are spoken as words (`WORDS`). Everything is wrapped in try/catch; if speech is unsupported all calls are no-ops. **The game MUST be fully playable silently.**
- **`cancel()` before each utterance:** counting may interrupt instructions, but instructions MUST NOT interrupt each other. Anything that must be heard in order uses `Speaker.tell(text, done)` which calls `done` on `onend` (with a 4 s fallback), e.g. the opening line then `startLevel`. `Speaker.tell` also records `Speaker.last`; the 🔊 button replays `Speaker.last`.
- `#live` (`aria-live="polite"`) mirrors every spoken line; a zero-width toggle makes identical repeated text re-announce.
- **Sound effects** (`unlockAudio`, `tone`, `Sfx`): context created only after a user gesture; `resumeAudio()` on every `pointerdown` and when the page becomes visible again (iOS suspends it). Gain 0.15 or less. `pop` 520→880 Hz 80 ms, `chime` 523/659/784 Hz at 0/150/300 ms, `boing` 400→200 Hz 150 ms (the only falling tone, neutral). Note: the iPhone silent switch may mute WebAudio but not speech; check on a device.
- **Cover:** the page opens behind a full-screen ▶ (`#cover`, 168 px) so audio unlocks on the first gesture. Nothing behind it is interactive.
- **Copy rules:** all spoken text is in one `COPY` object (lines with numbers are functions: `COPY.share(n,g)`, `COPY.fair(t)`). **Each line is 8 words or fewer.** Concrete words a 4-year-old uses (give, share, friend, same, more, fair, watch). Banned in `COPY`: leftover, remainder, can't, wrong, no, incorrect, missed, error, divide, equal, try again. The mascot speaks in first person; praise the outcome, not the child.
- **Localization:** `COPY`, `WORDS` and `lang` are English-only today. To add a language, key `COPY`/`WORDS` by locale and set `u.lang`; if no voice exists for it, fall back to silent play (the pictures carry the instructions).

## 8. Progression and persistence

- Levels with **preset parameters first** (no choice), then **picture-card choice offering only valid options** (no dead ends), then a **find-all-ways** goal. Totals start at 4-12; never more than 12 items on a phone. 3 rounds per level shown as 3 star dots (`role="img"`, `aria-label` updated). Level complete shows the overlay with a big ▶; `levelDone` saves the next level; **after the last level the last level simply repeats** (free play).
- **Scaffolds fade:** ghost slots always (L1-2), then only on empty targets plus numeral badges (L3), then badges only with ghosts returning via hint L2 or the uneven state (L4).
- **One obvious action:** when a round's goal is met in a picker, the ▶ card pulses and unfound options do not.
- **Non-divisible/leftover levels** MAY exist only late and positively (the mascot eats the leftover). Not built (§15).
- Save only the level reached in `localStorage` under a versioned key (`eg_v2`) inside try/catch; bump the version when redesigning. 🔄 restarts only the current round.

## 9. Safety, accessibility, parent

- **Hold-to-exit home** (`HOLD_MS = 1500`, radial/fill animation, `holdStart`/`holdEnd`); `Enter`/`Space` on 🏠 also exits (keyboard/switch users). No `.topnav` on kindergarten pages. No destructive or child-reachable settings, no external links, purchases or ads.
- Parent features (skin toggle, speech toggle, level override, reset) are not built; if added they MUST sit behind an adult gate.
- Custom tappables get `role="button"`, `tabIndex=0`, Enter/Space; every icon button has an `aria-label`; `:focus-visible` is a 4 px outline.
- Home Screen / offline: out of scope today (no manifest, no service worker). Pages are plain HTML served over Pages.

---

# Part B: The distribute pattern and other exercise shapes

## 10. Distribute pattern (Equal Groups)

- State: `pile` and `plates` (arrays of item ids) are the single source of truth. `mode ∈ {cover, deal, celebrate, picker, levelend}`; input is ignored outside `deal`.
- Flow: `startLevel` → `startPile` → (`startRound` | picker) → `move`… → `evaluate` → `success` → `nextPile` / `showPicker` → `levelDone`. `render()` = `layoutPile(); renderPile(); renderDots(); renderFriends() | renderPicker()`.
- Helpers: `counts()`, `neediest()`, `isFair()`, `ways()`, `el()`, `$`. Keep timer handles (`idleTimers`, `autoTimer`, `demoT`) and clear them on every mode change.
- Slot grids use the `COLS` map as a compact array (rows of 2-5) capped by `floor((cardW - 28 + gap) / (40 + gap))` so slots never go below 40 px. Targets per round MUST NOT exceed `FACES.length`.
- Top bar: 🏠 (hold), star dots, 🔊, 🔄, 🐻. That is **four buttons plus the dots**.

## 11. Other exercise shapes (no pile / plates)

The invariants in Part A still apply. Adapt as follows:
- **Choice tasks** (count and pick the numeral, more/less, which shape): a non-matching pick gently returns to its place; the mascot counts the quantity aloud with the child and the matching option glows (L1), ghost-taps it (L2) and taps it for them (L3). Never a verdict. Completion = the matching option chosen.
- **Sequence / path tasks** (number-line hops, patterns): one item or character moves step by step; each step is spoken; "stuck" is handled with the same three hint levels pointing at the next valid step.
- **Sorting tasks:** like distribute, with target = category; the uneven rule becomes "items in the wrong group wobble gently" with no red.
- In every case define the single state transition, an auto-completion rule, and positive "not yet" states in the plan (§2). Replace "pile empty" in the completion rule with the activity's own condition.

---

# Process

## 12. Constants (single source of truth; other places refer here)

| Name | Value |
|---|---|
| Icon button `--btn` | 56 px; 64 px at `min-width:700px` |
| Mascot / ▶ big | 72 px+ / 168 px |
| Item visual (`pc`) | ceiling 72 px, floor 48 px (prefer 52+; use more rows before going lower) |
| Gap between tappables | 12 px preferred, 8 px minimum |
| Target card (phone) | 100 x 120 px or more |
| Slot | floor 40 px, ceiling 56 px |
| Badge | 44 px circle, 26 px numeral |
| Focus ring | 4 px solid `--accent`, offset 3 px |
| Items per round | 12 or fewer on a phone |
| Drag threshold / lift / drop radius | 10 px / 48 px / 140 px (inflate 24 px, x0.3 inside) |
| Click suppression / FLIP / evaluate grace | 350 ms / 350 ms / 800 ms |
| Idle hints `HINT_MS` | 8000 / 16000 / 30000 ms |
| Demo start delay / cadence | 1200 ms / 900 ms |
| Celebration hold / confetti total | 3200 ms / 1.6 s or less |
| Hold-to-exit `HOLD_MS` | 1500 ms |
| Resize debounce | 120 ms |
| Speech | `en-US`, rate 0.85, pitch 1.15 |
| Sfx gain | 0.15 or less |
| Words per spoken line | 8 or fewer |
| Storage key | `eg_v2` (versioned) |

## 13. Repo integration and shipping

1. Create `<exercise-name>.html` at the repo root (kebab-case). One self-contained file: inline CSS and JS, no dependencies, build, network, external fonts or images.
2. Add a button to `index.html` with its own color class. For kindergarten pages the label **starts with a large emoji** (e.g. `🍪 Share the Cookies`), because the home page is where a non-reading child lands. Use the **same name** in `index.html`, the page `<title>`, README and any `.topnav`.
3. Add a link to the `.topnav` of every page that has one (the three legacy pages), marked `aria-current="page"` on the current page. Kindergarten pages use hold-home instead of a topnav.
4. Add one line to `README.md` describing the story and interaction; update the "linking to all N" count.
5. Branch `claude/<short-name>`, PR into `main` (Pages deploys from `main`; never push to `main`). One exercise per PR. Bump the `localStorage` key when redesigning.

## 14. Verification

- Serve with `python3 -m http.server`. Test with touch emulation at iPhone portrait 390x844, iPhone landscape 844x390, iPad landscape 1180x820 and iPad portrait, and on a real iPad/iPhone when possible. Console must be clean.
- Script: (1) ▶ cover then the opening line completes before the first instruction; (2) finish a round by tapping only, dragging only, and tap-select then tap; (3) drop on empty space and drop an item back on its own card (it must not stay faded; hints must continue); (4) tap a placed item (returns); (5) make an uneven result (wobble + "make it fair", nothing red; the L3 demo moves one item); (6) idle: L1/L2/demo at `HINT_MS`; touch during the demo stops it; double-tap the mascot (no second demo); (7) rotate / switch tabs mid-drag, and let a timer fire mid-drag (no stuck float, no frozen input); (8) two fingers; (9) reduced motion (static rings, wobble outline, no confetti, sound still plays); (10) speech unsupported (still playable, `#live` updates); (11) reload restores the level; private mode does not crash; (12) quick tap on 🏠 does nothing, a 1.5 s hold or Enter exits; (13) no page scroll or text selection; (14) iPhone landscape with a notch (pile not under the notch); (15) background the app and return (sound still works).
- **If you cannot run a browser or device, state that explicitly in the PR and do not tick checklist boxes you did not run.** A headless Playwright smoke test (load, tap ▶, complete a round via `tapFriend`) is welcome.
- Grep (against the file, not `COPY` wording): `grep -nEi "(#e53935|#f00\b|#ff0000|\bred\b|crimson|--bad|@keyframes shake|\bwrong\b|\bmissed\b|try again)" file.html` must return nothing. Check banned words in `COPY` by reading it.

## 15. Known gaps / reference deviations

Not built: parent area, stickers (if added: collection-only, nothing lost), leftovers level, a picture speech bubble on the mascot, an automated browser test, keyboard operation of pile/placed items, dark mode (a light theme is acceptable here), Home Screen/offline support.

Reference notes (fix when touching the file, do not copy blindly): `render()` rebuilds `innerHTML` each time (stable pile cells hide most flicker; keyed updates would be better); `REDUCED` is read once at load; a few off-palette literals (`#136b2c`, `#fff3d1`, `rgba(30,91,216,.3)`); placed items are 8 px apart which is the minimum gap; portrait `layoutPile` and the slot floors can still force a layout taller than the screen on very small phones, in which case reduce the item count rather than the floors.

## 16. Acceptance checklist

Run before shipping. Every applicable box MUST be true and executed.

**Concept**
- [ ] §2 plan exists in the PR. Story-based goal, no abstract sentence goals on screen. One task per screen; at most four buttons plus star dots; no Check; completion automatic.
- [ ] No reading needed (numerals are content; no word labels in the play UI).

**Feedback**
- [ ] §14 grep returns nothing; no red/✖/shake/buzz/"wrong"/tally/countdown.
- [ ] Needs-more = amber dashed + pulse + 👆; extra = wobble (+ static outline under reduced motion); success = green/gold + star + confetti (1.6 s) + chime + praise.
- [ ] Color never the only signal; reduced motion honored; sound still plays.

**Interaction**
- [ ] Tap, drag and tap-select all call the one state-transition function; auto-snap; missed drop returns silently; same-card drop re-renders and restarts hint timers.
- [ ] Pointer listeners on `window`; `cancelDrag()` before any state change during a drag; single pointer; palm rejection best-effort; click suppression; cancel on visibility/orientation.
- [ ] Stable pile cells; FLIP landing; completion evaluated only when nothing is left, after 800 ms.

**Hints**
- [ ] `HINT_MS` ladder; demo is one pass, or one fix when uneven, then hands back; `demoT` + `autoTimer` both cleared; touch cancels; hints fade after any placement; mascot >= 72 px.

**Audio**
- [ ] ▶ cover; `Speaker` (voice order, `cancel()`, `tell` with `done`, `last`); numbers as words; counting aloud; `Sfx` with `resumeAudio`; `#live` mirrors; `COPY` lines <= 8 words; playable silently.

**Layout**
- [ ] No scroll at 390x844, 844x390, iPad portrait and landscape; all four safe-area insets; `100vh` + `100dvh`; `color-mix` fallbacks.
- [ ] Sizes meet §12 floors (more rows before smaller items); computed from measured containers; re-laid-out on resize.

**Progression and safety**
- [ ] Preset, then picker with valid options only, then find-all; 3 rounds/level; ▶ pulses when the goal is met; progress saved at level completion in try/catch; hold-to-exit (1.5 s) + Enter; no topnav; icon buttons have `aria-label`.

**Repo**
- [ ] Single self-contained file; `index.html` button starts with an emoji; same name everywhere; legacy `.topnav` links; README line and count; branch + PR; §14 script executed (or the PR says what was not).

**The real test**
- [ ] A 5-year-old who has never seen it completes level 1 within 60 seconds, with no adult speech, **without any L3 demo**, and never meets a red or "wrong" element. If you cannot test with a child, say so in the PR and have the user run this test before merging.
