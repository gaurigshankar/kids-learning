# AGENTS.md

Guidance for AI coding agents working in this repo.

## Mandatory reading
Before creating, redesigning, or changing ANY exercise, read **[DESIGN.md](DESIGN.md)** in full and follow it. It is the binding standard for kindergarten (ages 4-6) touch exercises. It covers story-first design, positive-only feedback, wordless hints, speech and audio, touch handling, sizing, progression, and a final acceptance checklist you MUST run before shipping.

The reference implementation is `equal-groups.html`. Copy its patterns (`Speaker`, `Sfx`, `move`, `layoutPile`, `friendNear`, `escalate`, `holdStart`, ...) rather than reinventing them.

## Repo conventions
- One self-contained HTML file per exercise at the repo root (inline CSS and JS). No dependencies, no build step, no network, no external assets. Emoji serve as graphics.
- Target devices: iPad (landscape and portrait) and iPhone (portrait first, 390 px wide), Safari, touch.
- Every new exercise MUST:
  - get a button in `index.html` (its own color class),
  - get a link in the `.topnav` of every page that has one (`aria-current="page"` on the current page),
  - get a one-line description in `README.md`.
- Kindergarten pages use a hold-to-exit 🏠 button instead of a topnav.
- Save progress only via `localStorage` with a versioned key (e.g. `eg_v2`), always inside try/catch.

## Workflow
1. Plan first: write the short plan described in DESIGN.md §3 (story, goal in child words, interaction, hints, levels, COPY) and show it before building.
2. Build in a single HTML file following DESIGN.md.
3. Verify with `python3 -m http.server` plus device emulation (iPhone 390x844 portrait/landscape, iPad landscape/portrait), and on a real device when possible. Use the manual script in DESIGN.md §15.
4. Run the acceptance checklist in DESIGN.md §17. Every item must pass.
5. Work on a `claude/<short-name>` branch and open a PR into `main`. Never push to `main` directly, because GitHub Pages deploys from it.

## Hard rules (summary; DESIGN.md is authoritative)
- No reading needed, no Check button, no scores or failure counters, no timers.
- Never use red, X, shake, buzz, or "wrong". Feedback is positive only.
- Every instruction is spoken AND pictured. The game is fully playable silently.
- Touch targets: buttons 56 px or more, items 48-72 px, 12 or fewer items on a phone. The page never scrolls.

If you change a convention, update DESIGN.md in the same PR.
