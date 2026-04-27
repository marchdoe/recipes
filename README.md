# Recipes vault

A meal-planning Obsidian vault for Doug and Alisa. Recipes live in `recipes/`; each Sunday a GitHub Action proposes 5 weeknight dinners (`plans/`) and a categorized grocery list (`lists/`) tuned to our preferences (`preferences.md`).

## Structure

- `recipes/` — one Markdown file per recipe; frontmatter holds metadata + nutrition; body holds ingredients/directions/notes
- `recipes/_images/` — hero images, one per recipe, matched by slug (downloaded automatically when recipes are imported)
- `plans/YYYY-MM-DD.md` — weekly menu (date is the Monday of that week)
- `lists/YYYY-MM-DD.md` — paired grocery list
- `preferences.md` — household profile, dietary targets, weekly rhythm, picker rules, recent feedback
- `system-prompt.md` — instructions the Sunday agent reads
- `tools/questionnaire.md` — preference-interview template
- `docs/` — design spec and implementation plan
- `.github/workflows/sunday-plan.yml` — the Sunday cron Action

## How to use

- **Add a recipe:** open Claude Code in this folder, paste a URL, say "add this recipe"
- **Tweak this week's plan:** open Claude Code, say "swap Wednesday for X"
- **Mark cooked:** "we made the chicken thighs tonight"
- **Log feedback:** "the lentil soup was bland; we won't make it again"
- **Browse recipes on phone:** open Obsidian mobile (vault is in iCloud Drive)

## Where things are documented

- **`docs/README.md`** — day-to-day user guide: common tasks, the Sunday flow, troubleshooting
- **`docs/SETUP.md`** — technical setup record: what's installed where, how plugins are configured, why each decision was made, recovery cheat sheet (start here if setting up a fresh device)
- **`docs/TODO.md`** — outstanding work and deferred ideas
- **`docs/superpowers/specs/`** — design specs (the architectural why)
- **`docs/superpowers/plans/`** — implementation plans (the step-by-step how)
