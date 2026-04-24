# Recipes vault

A meal-planning Obsidian vault for Doug and Alisa. Recipes live in `recipes/`; each Sunday a GitHub Action proposes 5 weeknight dinners (`plans/`) and a categorized grocery list (`lists/`) tuned to our preferences (`preferences.md`).

## Structure

- `recipes/` — one Markdown file per recipe; frontmatter holds metadata + nutrition; body holds ingredients/directions/notes
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

See `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` for the full design.
