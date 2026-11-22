# AGENTS.md

Guidance for AI coding agents working in this repo.

- **Scope:** new exercises and `equal-groups.html` ("Share the Cookies") are for kindergarten children (ages 4-6) and MUST follow **[DESIGN.md](DESIGN.md)**. Read it in full first, and write the plan it asks for (§2) before building. The three older pages (`number-rods.html`, `place-value-blocks.html`, `missing-number-rods.html`) are legacy: exempt except for the integration steps (DESIGN.md §13). Do not convert them unless asked.
- **Format:** one self-contained HTML file per exercise at the repo root. No dependencies, build, network or external assets.
- **Integration:** button in `index.html` (kindergarten labels start with an emoji), link in each legacy page's `.topnav`, one line in `README.md`, same name everywhere.
- **Persistence:** only the level reached, in `localStorage` under a versioned key, inside try/catch.
- **Git:** work on a `claude/<short-name>` branch and open a PR into `main` (Pages deploys from `main`; never push to it directly).
- **Honesty:** run the verification script and checklist (DESIGN.md §14, §16). Do not claim checks you did not run; say what you could not test.
- If you change a convention, update DESIGN.md in the same PR.
