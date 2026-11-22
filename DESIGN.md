# DESIGN.md: Kindergarten Math Exercises

**Status: mandatory.** Every AI agent or human who creates or changes an exercise in this repo MUST read this file first and follow it. MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119. If this file conflicts with your instincts, follow this file. If it conflicts with what a real 5-year-old does on a real device, the child wins. Update this file in the same PR as any convention change.

**Reference implementation:** `equal-groups.html` ("Share the Cookies"), rebuilt after a usability review. When this document names an identifier in backticks (`Speaker`, `layoutPile`, `friendNear`, ...), it means the function or constant in that file. Copy its patterns; do not reinvent them. Section 16 lists where the reference itself falls short. There, this document wins.

The audience is children aged 4 to 6 who mostly **cannot read**. They use an iPad (landscape or portrait) or an iPhone (portrait first, 390 px wide), with fingers, often with no adult watching.

---

## 1. Core thesis (non-negotiable)

1. **A story the child owns.** Build the math around a situation a 5-year-old already understands, such as *sharing cookies fairly with animal friends*. The math is the story's goal, never an abstract statement like "3 groups of ?".
2. **One task per screen.** At any moment there is one obvious thing to do. No side panels, no mode switches, no settings.
3. **Every instruction is spoken AND pictured.** Speech says it; the screen shows it (halo, pointing hand, ghost slots). A child who can't hear, or doesn't understand the words, can still play.
4. **Text is at most a numeral or 1-2 words.** Nothing on screen may require reading. Buttons are icons or emoji. Numerals MAY appear as scaffolds (badges, picker cards).
5. **No Check button.** The activity detects completion itself and celebrates.
6. **No scores, no failure counters, no timers, no "wrong".** Progress is shown only as filled star dots.
7. **Everything is reversible.** A misplaced item can always be taken back with one tap.

---

## 2. Failures to avoid (from the v1 review)

Each row shipped in v1 of Equal Groups and hurt children. You MUST NOT reintroduce any of them.

| v1 failure | Rule now |
|---|---|
| Sentence-style abstract goal ("3 groups of ?") | Goal is a spoken story line plus picture scaffolds (ghost slots). §8 |
| Small +/- steppers to set parameters | Parameters are preset per level, or chosen by tapping big picture cards. §10 |
| About 10 competing buttons | At most 5 top-bar icon buttons (home, star dots, say-again, restart, mascot). Nothing else outside the play area. §4 |
| 18-34 px drag targets | Items 52-72 px visual with a generous hit area. Targets at least 100x120 px on a phone. §4.3 |
| Scrolling or sticky pile eating the screen | Docked, non-scrolling tray. The page never scrolls. §4 |
| Red X, shake, "missed" tally | Positive-only feedback. §6 |
| Long messages with abstract words ("leftover", "can't") | Sentences under 8 words, concrete words, kept in `COPY`. §8.4 |
| 24-30 items | 12 or fewer items on a phone. Totals start at 4-12. §10 |
| Hint hidden in header with escalating text labels | Wordless picture hints, mostly automatic, plus a big mascot button. §7 |
| Abstract inventory strip | The pile is the real objects (🍪), laid out stably. §4 |
| No audio | Speech for every instruction plus WebAudio sound effects. §8 |
| No pointer capture | `setPointerCapture`, single active pointer, palm rejection. §5.3 |
| DOM rebuild flicker | Stable cells keyed by id, FLIP animation on landing. §5.2, §5.4 |
| Page scroll or bounce during drag | `body{position:fixed}`, `touch-action` rules, `preventDefault` on pointerdown. §4.1 |
| Auto-check that punished the last drop | Evaluate only when the pile is empty, after an 800 ms grace period. An uneven result gets a gentle invitation, not a verdict. §6 |

---

## 3. Plan first (required before writing code)

Before writing any HTML you MUST write a short plan (in the PR description, or in chat with the user) covering the items below. Do not start building until the user has seen it.

```
Story:            who / what / why (e.g. "Bobo's animal friends want cookies; share fairly")
Goal (child words, <8 words, spoken): "Give every friend the same number of cookies."
Math concept:     (e.g. partitioning N into G equal groups)
Objects:          item emoji (🍪), target emoji (🐶🐱🐰...), mascot (🐻 Bobo)
Interaction:      the ONE state transition (e.g. move item pile -> target), and how tap / drag / tap-select map to it
Completion rule:  what the code checks to auto-complete (e.g. pile empty AND all plates equal)
Not-yet-done states: needs-more / extra / uneven, and how each is shown positively
Hints:            what L1 / L2 / L3 look like for THIS activity
Levels:           table of preset parameters, 3 rounds each, scaffolds per level
Copy:             full COPY object draft (every spoken line)
Sizes:            max item count per round, slot grid COLS map
```

Then build, verify (§15), and run the acceptance checklist (§17).

---

## 4. Screen, layout and sizing

### 4.1 Page shell (copy from `equal-groups.html`)

- Meta viewport MUST be `width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no`.
- `body`: `position: fixed; inset: 0; overflow: hidden; overscroll-behavior: none; touch-action: manipulation; user-select: none; -webkit-user-select: none; -webkit-touch-callout: none;`
- `* { -webkit-tap-highlight-color: transparent; }`
- `#app`: `height: 100dvh`, CSS grid, padding including `env(safe-area-inset-top)` and `env(safe-area-inset-bottom)` (via `--safe-b`).
- Draggable items MUST have `touch-action: none` (`.ck`).
- Block the context menu: `document.addEventListener('contextmenu', e => e.preventDefault())`.
- Font stack: `"Nunito", "Avenir Next", "Trebuchet MS", system-ui, sans-serif`. Do not load web fonts.

### 4.2 Regions

| Orientation | Layout |
|---|---|
| Phone portrait (390 px reference) | Rows: top bar (`auto`, at least 64 px), target area (`1fr`, flexible), docked bottom tray (`auto`, `align-self: end`, never scrolls). `grid-template-areas: "top" "friends" "pile"` |
| iPad landscape and phone landscape | `@media (orientation: landscape) and (min-aspect-ratio: 23/20)`: `grid-template-areas: "top top" "pile friends"`. The tray becomes the left column and stretches to full height. |

- The page MUST NOT scroll in any orientation at any supported size. If content doesn't fit, shrink computed sizes or reduce the item count. Never add scrolling.
- Cap width with `max-width: 1200px; margin: 0 auto`.

### 4.3 Size table

| Element | Minimum | Reference |
|---|---|---|
| Icon buttons (`.ibtn`) | 56 px phone, 64 px at `min-width: 700px` | `--btn` |
| Mascot button | 72 px or more, gold border, idle `bounce` every 8 s | `#bobo` |
| Big ▶ (cover, level end) | 168 px circle | `.bigplay` |
| Draggable item, visual | 52-72 px (hard floor 48 px for new work) | `pc`, computed in `layoutPile` |
| Item hit area | equal to or larger than the visual, never smaller | `.ck` sized to its cell |
| Gaps between tappables | 12 px preferred, 8 px absolute minimum | |
| Target card (friend) on phone | 100 x 120 px or more | `.friend`, computed in `renderFriends` |
| Slot inside target | ten-frame style grid from the `COLS` map. 40 px or more for new work. | `.slot` |
| Badge numeral | 44 px circle, 26 px bold | `.badge` |
| Focus ring | `outline: 4px solid var(--accent); outline-offset: 3px` | `:focus-visible` |
| Item count per round | 12 or fewer on a phone | `LEVELS` |

### 4.4 Sizes are computed in JS

- Item and slot sizes MUST be computed from the viewport (`window.innerWidth/innerHeight`, container `clientWidth/clientHeight`). Copy the `layoutPile` and `renderFriends` pattern: clamp with `Math.max(floor, Math.min(ceiling, available / count))`, then write `gridTemplateColumns` and `gridAutoRows` in px. Set a CSS variable (`--pc`) if CSS needs it.
- Re-lay-out on `resize`, debounced to 120 ms, and never during a drag (copy the `rz` handler).
- `isLandscape()` uses `innerWidth > innerHeight * 1.15` so it matches the CSS aspect-ratio query (23/20).

---

## 5. Interaction model

### 5.1 One state transition, three equivalent inputs

All input MUST end up in a single function that changes state. In the reference this is `move(id, from, to, srcRect)`. Nothing else mutates `pile` or `plates`.

| Input | Behavior | Reference |
|---|---|---|
| **Tap a target** | Deals one item: the selected one, or the next from the pile | `tapFriend(g)` |
| **Drag an item** | Picks up after a 10 px threshold. Floats 48 px above the finger. Drops on the *nearest* target (forgiving). | `onDown` / `onMove` / `onUp`, `friendNear` |
| **Tap item, then tap target** | Tapping a pile item selects it (gold glow plus bob); tapping again deselects; tapping a target delivers it | `selected`, `.ck.sel` |
| **Tap a placed item** | Returns it to the pile | `onUp` branch `d.from !== 'p'` |
| **Drop on nothing** | The item glides back silently. No sound, no message. | `onUp` final `else` |

- Placed items MUST auto-snap into the next free slot. The child never positions items precisely.
- Forgiving drop (`friendNear`): inflate each target rect by 24 px; points inside the inflated rect get distance x 0.3; accept the nearest target within 140 px of its center. Use the *floated item's* position (`clientY - 48`), not the finger's.
- Highlight the would-be drop target live during a drag (`.hot`, sky border, scale 1.05). Highlight the tray when returning (`#pilewrap.hot`).

### 5.2 Stable layout

- The pile renders **one cell per item id**, in a fixed order (`renderPile` loops `i < N` and leaves empty cells). Items never reflow when a neighbor leaves.
- Every item element carries `data-id` and `data-from` so it can be found again after a re-render.

### 5.3 Touch robustness (all MUST)

| Concern | Pattern |
|---|---|
| Palm rejection | `if (e.pointerType === 'touch' && e.width > 60) return;` |
| Single active pointer | `if (mode !== 'deal' \|\| drag) return;` at the top of `onDown`. Ignore moves and ups whose `pointerId !== drag.pid`. |
| Pointer capture | `t.setPointerCapture(e.pointerId)` in try/catch. Listen on the item for `pointermove/pointerup/pointercancel/lostpointercapture`. |
| No page scroll | `e.preventDefault()` in `onDown`, plus `touch-action: none` on items |
| Cancel safely | `onCancel` on `pointercancel`, `lostpointercapture`, `visibilitychange` (hidden), `orientationchange`. Cancelling returns the item and restarts timers. |
| Click suppression | `lastDragEnd = Date.now()` in `endDrag`. Target click handlers ignore clicks within 350 ms. Items `stopPropagation` on click. |
| Cleanup | `endDrag()` removes listeners, the float and all `.hot` classes, then sets `drag = null` |

### 5.4 Animation of landings (FLIP)

After `render()`, find the moved element by `data-id`, measure its new rect, and animate from `translate(old - new) scale(1.2)` to `none` over 350 ms ease-out (see the end of `move`). Skip under reduced motion. Pass `srcRect` from wherever the item visually was: the float, the pile cell, or the plate.

---

## 6. Feedback (positive only)

| State | Visual | Sound / speech | Never |
|---|---|---|---|
| Item placed | Target face `munch`, badge `pop` | `Sfx.pop()` plus count aloud of the new total (`Speaker.count`) | |
| Item returned to pile | FLIP glide | `Sfx.boing()` (soft, low) | |
| Target needs more | Amber/gold border, dashed amber ghost slots, `pulse`, 👆 | (hint speech only) | red |
| Target has extra (uneven) | Gentle `wobble` (±2°) | "Hmm, let's make it fair!" | shake, buzz, X |
| Success | Green border on all targets, badges shown, star dot fills, mascot cheers (🤗 + `jump`), confetti | `Sfx.chime()` plus "Everybody has three! That's fair!" | |
| Level complete | Full-screen overlay: mascot, ⭐⭐⭐, big ▶ | chime plus "You did it!" | score, time |

Rules:
- You MUST NOT use red, ❌, ✖, shake animations, buzzer sounds, "wrong", "no", "try again", "missed", or any tally of mistakes. The CSS MUST NOT define a `--bad` color.
- No timers, countdowns or speed pressure. The only timers are idle-hint timers (§7) and celebration pacing.
- Color is never the only signal. Pair it with shape (dashed ghost slots), an icon (👆, ⭐) and motion (pulse or wobble, or a static ring under reduced motion).
- Confetti: 24 emoji pieces (`🎉 ⭐ 🍪 ✨`), the whole effect MUST finish within 1.6 s (§16 notes the reference slightly exceeds this). Remove the nodes on finish.
- Evaluate completion only when nothing is left to place, after an 800 ms grace period (`setTimeout(evaluate, 800)`), so the last count can be spoken and seen. If uneven, raise the hint to level 2 and show the wobble. Don't announce a verdict.
- After success, hold the celebration about 3.2 s, then advance automatically (`nextPile` or `showPicker`).

### 6.1 Reduced motion

Check `REDUCED = matchMedia('(prefers-reduced-motion: reduce)').matches`. When true:
- Set `animation: none` on every looping or bouncy animation (mascot, needy pulse, wobble, bob, point, pop, munch, bigplay).
- Replace the pulse with a **static glow ring**: `box-shadow: 0 0 0 6px color-mix(in srgb, var(--gold) 50%, transparent)`.
- Skip confetti, the ghost flyer and the FLIP animation.
- Sound and speech **still play**.

---

## 7. Hints: wordless, escalating, mostly automatic

| Level | Trigger | Picture | Speech (`COPY`) |
|---|---|---|---|
| 0 | default | none (ghost slots per level scaffold) | |
| 1 | idle 6 s, or mascot tap | Halo/pulse plus 👆 on the **neediest** target (`neediest()`) | "Who needs a cookie?" |
| 2 | idle 12 s, or mascot tap | Ghost item flies from the pile to the neediest target (`flyGhost`, 900 ms). Ghost slots show the target count on every target. | "Give a cookie to this friend." |
| 3 | idle 20 s, or mascot tap | Auto-demo: "Watch me!", then after 1.2 s deals one item every 700-900 ms (`autoDeal`, 900 ms) with counting aloud | "Watch me!" |

- Idle timers are set in `setTimers()` (6000, 12000, 20000 ms). They reset on every placement and after every cancelled drag, and are cleared outside `deal` mode.
- **Any touch cancels the demo**: a capture-phase `pointerdown` listener on `document` calls `stopAuto(); setTimers()`.
- **Hints fade.** After a correct placement `hintLevel` drops by one (and resets to 0 after a demo): `hintLevel = hintLevel > 2 ? 0 : max(0, hintLevel - 1)`.
- **Mascot button** (`#bobo`, 🐻 Bobo): always in the same place (top bar, far right), at least 72 px, idle bounce. A tap raises the hint one level (`boboTap`). Outside `deal` mode it repeats the current instruction.
- The first round of level 1 starts at hint level 1, so the first action is pointed at.
- Hints are NEVER scored, counted, or allowed to block progress.

---

## 8. Audio and speech

### 8.1 Speech: copy the `Speaker` module

- Web Speech API. `lang = 'en-US'`, `rate = 0.85`, `pitch = 1.15`.
- Voice preference: `/Samantha|Ava|Karen|Allison|Susan|Zira/i`, then any `en-US`, then any `en`. Re-pick on `speechSynthesis.onvoiceschanged`.
- Call `speechSynthesis.cancel()` before every `speak` so lines never queue up.
- Speak numbers as **words** (`WORDS[n]`, "three", not "3") via `Speaker.count(n)`.
- Every `say(t)` also writes `t` into the `aria-live="polite"` element `#live`, even when speech is unsupported.
- If `'speechSynthesis' in window` is false every call is a no-op. **The game MUST be fully playable silently.** Pictures carry every instruction.
- Cancel speech on `pagehide` and when `document.hidden`. Wrap every speech call in try/catch.

### 8.2 Sound effects: copy `unlockAudio`, `tone`, `Sfx`

- Create the WebAudio context **only after a user gesture** (`unlockAudio()` in the ▶ handler) and `resume()` it if suspended.
- Gain about 0.15 with exponential decay.
- `pop`: 520 to 880 Hz, 80 ms (placement). `chime`: C5-E5-G5 (523/659/784 Hz) at 0, 150 and 300 ms (success). `boing`: 400 to 200 Hz, 150 ms (return).
- No buzzers or descending "fail" stingers. `boing` is the only falling tone and is used only for a neutral return.

### 8.3 iOS audio unlock cover

The page MUST open behind a full-screen overlay (`#cover`) showing the mascot and a 168 px pulsing ▶. Tapping it unlocks audio, initializes speech, says `COPY.start` and starts the level. No game state is visible or interactive behind it.

### 8.4 Copy rules

- **All spoken text lives in one `COPY` object** at the top of the script. Templated lines (with numbers) SHOULD be functions in `COPY` too, e.g. `COPY.share(n, g)` and `COPY.fair(t)` (see §16).
- Each sentence MUST be under 8 words where possible and never more than 8. Use concrete, present-tense words a 4-year-old uses: give, share, friend, same, more, fair, watch, again.
- Banned words: leftover, remainder, can't, wrong, no, incorrect, missed, error, groups-of, divide, equal (say "same"), try again.
- The mascot speaks in first person ("Watch me!"). Praise is about the outcome ("That's fair!"), not the child's intelligence.

---

## 9. Visual language

- Palette (copy the `:root` variables): `--bg #fbf7ee`, `--panel #fff`, `--ink #26303b`, `--muted #8a8272`, `--line #e3dccb`, `--accent #1e5bd8`, `--good #1e8e3e`, `--gold #f6b93b`, `--sky #4aa8e8`. There is no red.
- Objects and characters are **emoji**; no image assets. Animal faces come from a fixed list (`FACES = ['🐶','🐱','🐰','🐸','🦊','🐼']`). Targets per round MUST NOT exceed `FACES.length`.
- Rounded everything (26 px card radius, circular buttons, 3-4 px borders).
- Numerals MAY appear only as scaffolds: target badges (levels 3+ and at success) and picker cards. Word labels MUST NOT appear in the play UI (`aria-label`s are fine).
- Top bar icons: 🏠 (hold to exit), ⭐⭐⭐ (round dots, grayscale until done), 🔊 (say again), 🔄 (restart this round), 🐻 (mascot help).

---

## 10. Progression

- **Levels with preset parameters come first.** The child makes no choices. Example: `LEVELS = {1:{piles:[4,6,8],G:2}, 2:{piles:[6,9,12],G:3}, 3:{piles:[6,8,12],G:0}, 4:{piles:[12,10,8],G:0,needAll:true}}`.
- **Then a picture-card choice.** Offer **only valid options** (`ways()` returns only divisors). No dead ends; every option has a solution. Picker cards show the crowd of faces plus a small numeral. The next unfound option pulses (`.pick.next`); found options get a ⭐ (`.pick.done`).
- **Then a find-all-ways goal** (`needAll`). Show a ▶ card once the round goal is met.
- **Small totals first** (4-12). Never more than 12 items on a phone.
- **Scaffolds fade across levels:** ghost slots always (L1-2), then ghost slots only on empty targets plus numeral badges (L3), then badges only, with ghosts returning only via hint level 2 or an uneven state (L4). See `ghosts` and the badge condition in `renderFriends`.
- **3 rounds per level**, shown as 3 star dots. Level complete shows the celebration overlay with a big ▶ to continue.
- **Save progress** in `localStorage` under a versioned key (`<prefix>_v<n>`, e.g. `eg_v2`), always in try/catch (private mode throws). Save only the level reached. Never save failures.
- **Non-divisible / leftover cases** MAY exist only as an optional late level with positive framing: the mascot happily eats the leftover. No penalty, no "can't". (Not yet built, §16.)
- 🔄 restarts only the current round. It never resets levels.

---

## 11. Safety, parent and accessibility

- **Hold-to-exit home:** 🏠 MUST need a 600 ms hold, shown as a ring/fill animation (`#home.holding::after`, `holdStart`/`holdEnd` on pointerdown and pointerup/leave/cancel). A quick tap does nothing.
- Kindergarten pages MUST NOT show the `.topnav` link bar. Children tap it by accident. The hold-home button replaces it.
- No destructive actions anywhere a child can reach: no reset-progress, no delete, no external links, no purchases, no ads.
- Everything is reversible (tap a placed item to return it, 🔄 to restart the round).
- Keyboard and screen-reader support is secondary but required. Interactive non-buttons get `role="button"` and `tabIndex = 0` and respond to Enter/Space (see friend cards). Every icon button has an `aria-label`. Use `:focus-visible` with a 4 px outline. `#live` mirrors speech.
- Parent-area features (skin toggle, speech toggle, level override, reset progress) are **not built**. If you add them they MUST sit behind an adult gate (e.g. long-press plus a 2-finger gesture) and MUST NOT be reachable by a child tapping around.

---

## 12. Code architecture

- **One self-contained HTML file** per exercise: inline `<style>` and an inline `<script>` in an IIFE. No dependencies, build, network, external fonts or images.
- Plain ES5-compatible JavaScript (`var`, `function`) matches house style.
- **State** lives in module-level variables. `pile` and `plates` (arrays of item ids) are the single source of truth. The DOM is always derived from them.
- **Modes**: `mode ∈ {cover, deal, celebrate, picker, levelend}`. Every handler checks the mode first; input is ignored outside `deal`.
- **Flow:** `startLevel` → `startPile` → (`startRound` | picker) → `move`… → `evaluate` → `success` → `nextPile` / `showPicker` → `levelDone`.
- **Render:** `render()` = `layoutPile(); renderPile(); renderDots(); renderFriends()|renderPicker()`. Rendering MUST be idempotent and derive everything from state.
- **Helpers to mirror:** `counts()`, `neediest()`, `isFair()`, `ways()`, `el(tag, cls, text)`, `$ = id => getElementById`.
- **Timers:** keep handles (`idleTimers`, `autoTimer`) and always clear them on mode change (`clearTimers`, `stopAuto`).

---

## 13. Copy-paste patterns from `equal-groups.html`

| Need | Copy | Notes |
|---|---|---|
| Fixed, non-scrolling, safe-area shell | `body` and `#app` CSS, plus the landscape media query | top of the `<style>` |
| Tray / target area layout | `#pilewrap`, `#friends`, `.friend`, `.plate`, `.slot.ghost` | |
| Viewport-computed sizes | `layoutPile()`, the sizing math in `renderFriends()`, `isLandscape()` | clamp pattern |
| Ten-frame slot grids | `COLS` map | `{1:1,2:2,3:3,4:2,5:3,6:3,7:4,8:4,9:3,10:5,11:4,12:4}` |
| Stable pile | `renderPile()` | one cell per id |
| Single state transition plus FLIP | `move(id, from, to, srcRect)` | |
| Auto-complete | `evaluate()`, `isFair()`, `setTimeout(evaluate, 800)` | |
| Drag | `onDown`, `onMove`, `floatAt`, `onUp`, `endDrag`, `onCancel` | 10 px threshold, 48 px lift |
| Forgiving drop | `friendNear(x, y)`, `overPile(x, y)` | 24 px inflate, x0.3, 140 px max |
| Tap-to-deal / select | `tapFriend(g)`, `selected` | |
| Hint ladder | `setTimers`, `escalate(n)`, `boboTap`, `flyGhost`, `autoDeal`, `stopAuto` | |
| Demo cancel on touch | capture-phase `document` `pointerdown` listener | |
| Speech | `Speaker` (`init`, `say`, `count`), `WORDS` | |
| Sound effects | `unlockAudio`, `tone`, `Sfx` | |
| Audio unlock cover | `#cover` + `.bigplay` + `$('play')` handler | |
| Celebration | `success()`, `confetti()`, `#bobo.cheer`, `#levelend` | |
| Progress | `save()` and the `try { JSON.parse(localStorage.getItem('eg_v2')) }` block | |
| Hold-to-exit | `holdStart`, `holdEnd`, `#home.holding::after` | 600 ms |
| Reduced motion | `REDUCED` and the `@media (prefers-reduced-motion)` block | static glow ring |
| Lifecycle safety | `visibilitychange`, `pagehide`, `orientationchange`, `contextmenu` listeners, debounced `resize` | |

Key snippets (reproduce exactly, adapt names only):

```js
// Speaker.say: mirror to aria-live, cancel, then speak
$('live').textContent = t;
if (!this.ok) return;
try {
  speechSynthesis.cancel();
  var u = new SpeechSynthesisUtterance(t); u.lang = 'en-US'; u.rate = 0.85; u.pitch = 1.15;
  if (this.voice) u.voice = this.voice;
  speechSynthesis.speak(u);
} catch (e) {}
```

```js
// onDown guards
if (mode !== 'deal' || drag) return;                       // single pointer, mode check
if (e.pointerType === 'touch' && e.width > 60) return;     // palm
e.preventDefault(); stopAuto(); clearTimers();
try { t.setPointerCapture(e.pointerId); } catch (x) {}
```

```js
// card click after drag suppression
card.addEventListener('click', function () { if (Date.now() - lastDragEnd < 350) return; tapFriend(g); });
```

---

## 14. Repo integration and shipping

1. Create `<exercise-name>.html` at the repo root (kebab-case).
2. Add a button to `index.html`: an `<a>` with its own color class (follow `a.green` / `a.orange` / `a.purple`). The label is 1-3 words (it is for adults and older siblings).
3. Add a link to the `.topnav` of **every page that has one** (`number-rods.html`, `place-value-blocks.html`, `missing-number-rods.html`). Mark the current page with `aria-current="page"`. Kindergarten-style pages use hold-home instead of a topnav (§11).
4. Add one line to `README.md` describing the story and interaction in one sentence. Update the "linking to all N" count.
5. Work on a branch named `claude/<short-name>` and open a PR into `main`. GitHub Pages deploys from `main`; never push directly to `main`.
6. One exercise per PR. If you redesign an existing exercise, bump its `localStorage` key version so stale saves don't break it.

---

## 15. Verification (before opening the PR)

- Serve locally with `python3 -m http.server` and open the page.
- Test in device emulation at minimum: **iPhone portrait 390x844**, **iPhone landscape 844x390**, **iPad landscape 1180x820 / 1024x768**, **iPad portrait**. Use touch emulation, not just a mouse. Test on a real iPad or iPhone in Safari whenever possible.
- Walk through this script:
  1. ▶ cover appears; tapping it brings speech ("Let's share the cookies!" then the goal).
  2. Complete a round **by tapping only**, then **by dragging only**, then **by tap-select then tap target**.
  3. Drop an item on empty space: it glides back silently.
  4. Tap a placed item: it returns to the pile.
  5. Make an uneven result: you see a wobble plus "make it fair", and nothing red.
  6. Wait idle: L1 at about 6 s, L2 at about 12 s, demo at about 20 s. Touch during the demo and confirm it stops.
  7. Rotate mid-drag, and switch tabs mid-drag. No stuck floats.
  8. Two fingers at once: only one item moves.
  9. Turn on reduced motion: static rings, no confetti, sound still plays.
  10. Disable speech (or use a browser without it): still playable from pictures alone, and `#live` updates.
  11. Reload: the level is restored. Private mode doesn't crash.
  12. A quick tap on 🏠 does nothing; a 600 ms hold goes to `index.html`.
  13. The page never scrolls or bounces, and no text selection or callout appears on long-press.
- Check the console: no errors.

---

## 16. Known gaps / not yet built

Acknowledged, and not blockers unless your change touches them.

- **Parent area**: no skin toggle (cookies vs other objects), no speech on/off, no level override, no reset-progress UI. Needs an adult gate (§11).
- **Stickers / rewards collection**: not built. If added, it MUST stay collection-only, with nothing lost and no streaks.
- **Leftovers level** (non-divisible totals where the mascot eats the leftover): designed, not built.
- **Picture speech bubble**: the mascot doesn't show a pictorial bubble (e.g. 🍪➡️🐶) alongside speech. For now the hints carry this.
- **No automated browser test.** Verification is manual (§15). A headless smoke test (load, tap ▶, complete a round via `tapFriend`) would be welcome.
- **Reference deviations to fix, not copy:**
  - Landscape `layoutPile` floor is 34 px and the plate `slot` floor is 24 px. New work MUST keep items at least 48 px and slots at least 40 px, reducing counts instead.
  - Confetti runs 1.4-1.8 s plus up to 250 ms delay. Target is 1.6 s total or less.
  - Templated lines (`'Share ' + WORDS[N] + ...`, `'Everybody has ' + ...`) are built inline instead of living in `COPY`.
  - `render()` rebuilds `innerHTML` on every change. Stable cells hide most flicker, but keyed in-place updates would be better.
  - `render()` contains a leftover no-op comment block.
  - Placed items and the pile are not keyboard-operable (only friend cards are).
  - No dark mode, unlike the older pages. A light theme is acceptable for kindergarten pages.

---

## 17. Acceptance checklist (run before shipping; every box MUST be checked)

**Concept**
- [ ] A written plan (§3) exists in the PR description: story, goal in child words, interaction, hints, levels, COPY.
- [ ] Story-based goal. No abstract sentence goals on screen.
- [ ] One task per screen. At most 5 top-bar buttons. No Check button. Completion is automatic.
- [ ] On-screen text is at most numerals or 1-2 words. Nothing requires reading.

**Feedback**
- [ ] No red, ❌, shake, buzz, "wrong/no/try again/missed", score, failure count or timer anywhere (grep for `red`, `#e53935`, `--bad`, `shake`, `wrong`, `missed`).
- [ ] Needs-more = amber dashed ghost plus pulse plus 👆. Extra = wobble. Success = green/gold plus star plus confetti (1.6 s or less) plus chime plus spoken praise.
- [ ] Color is never the only signal.
- [ ] Reduced motion: static glow rings, no confetti/bobbing/FLIP, sound still plays.

**Interaction**
- [ ] Tap target, drag, and tap-select-then-target all call the single state-transition function.
- [ ] Tapping a placed item returns it. Missed drops glide back silently. Items auto-snap to slots.
- [ ] 10 px drag threshold, 48 px lift, forgiving nearest-target drop with live highlight.
- [ ] Palm rejection, single pointer, `setPointerCapture`, cancel on pointercancel / lostpointercapture / visibilitychange / orientationchange, click suppression (350 ms).
- [ ] Stable pile cells by id. FLIP landing animation.
- [ ] Completion is evaluated only when nothing is left, after an 800 ms grace period.

**Hints**
- [ ] Idle hints at 6 / 12 / 20 s: halo plus hand, then ghost flyer plus ghost slots, then auto-demo (700-900 ms cadence, counting aloud).
- [ ] Mascot button at least 72 px, always in the same place, idle bounce, raises the hint level.
- [ ] Any touch cancels the demo. Hints fade by one after a correct placement. Hints are never scored.

**Audio**
- [ ] Full-screen ▶ cover unlocks audio on the first gesture.
- [ ] `Speaker` with en-US, rate 0.85, pitch 1.15, voice preference, `voiceschanged`, `cancel()` before each line, cancel on pagehide/hidden.
- [ ] Numbers spoken as words. Counting aloud on each placement.
- [ ] WebAudio `pop`/`chime`/`boing` at gain 0.15 or less, context created after the gesture.
- [ ] All spoken text is in `COPY`, each line under 8 words, no banned words. `#live` mirrors speech.
- [ ] Fully playable with speech unsupported and with sound muted.

**Layout**
- [ ] No scrolling or bouncing at 390x844, 844x390, iPad landscape, or iPad portrait.
- [ ] Portrait: top bar, flexible targets, docked tray. Landscape (aspect ratio 23/20 or more): tray in the left column.
- [ ] `body` fixed with overflow hidden, `100dvh`, safe-area insets, `touch-action` rules, `user-select: none`, no callout, context menu blocked.
- [ ] Buttons 56 px (64 px on iPad). Items 48-72 px visual (52+ preferred). Targets at least 100x120 px on a phone. Gaps at least 8 px (12 preferred). 12 or fewer items per round.
- [ ] Sizes computed in JS from the viewport, re-laid-out on a debounced resize.

**Progression and safety**
- [ ] Preset levels first, then a picker with only valid options, then find-all-ways. Totals start at 4-12. 3 rounds per level with star dots. Level-end overlay with big ▶.
- [ ] Scaffolds fade (ghost slots, then numerals, then nothing, plus hints).
- [ ] Progress saved in `localStorage` under a versioned key, inside try/catch.
- [ ] Hold-to-exit home (600 ms fill). No topnav on the kindergarten page. No destructive or child-reachable settings.
- [ ] `aria-label` on icon buttons, `role=button` + `tabIndex` + Enter/Space on custom tappables, 4 px `:focus-visible`.

**Repo**
- [ ] Single self-contained HTML file. No dependencies, network calls or build.
- [ ] Button added to `index.html`. Link added to every existing `.topnav`. Line added to `README.md`.
- [ ] Branch `claude/<name>`, PR into `main`. The manual verification script (§15) passed with no console errors.

**The real test**
- [ ] A 5-year-old who has never seen it completes level 1 in **60 seconds or less**, with **no adult speech**, and **never meets a red or "wrong" element**. If you can't test with a child, say so explicitly in the PR and have the user run this test before merging.
