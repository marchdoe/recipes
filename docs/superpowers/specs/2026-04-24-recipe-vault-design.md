# Recipe Vault & Meal Planning System — Design Spec

**Date:** 2026-04-24
**Status:** Approved for implementation planning
**Owner:** Doug March
**Household:** Doug and Alisa

## 1. Purpose

Build an Obsidian-based recipe vault that automatically proposes a weekly dinner menu and grocery list every Sunday morning, while remaining fully editable through interactive Claude Code sessions during the week. Optimized for low-cholesterol eating and a two-adult household where one person (Doug) maintains the system and the other (Alisa) reads it on her phone.

## 2. Decisions summary

| # | Decision | Choice |
|---|---|---|
| Workflow model | Hybrid: autonomous Sunday cron + interactive tweaks | C |
| Obsidian usage | Dedicated vault for recipes only | A |
| Dietary baseline | Low cholesterol + adaptive preferences over time | A |
| Household | Two adults (Doug, Alisa); 2 servings default | B |
| Recipe library | Starting fresh; URL imports going forward | A |
| Automation host | GitHub Actions (cloud cron) | A |
| Spouse access | Read-only on phone; URLs sent to Doug for import (bottleneck) | 1b + 2b |
| Sync method | iCloud single-vault (Phase 1) → Obsidian Sync (Phase 2 if needed) | 3 |
| Spouse Claude Code | Not installed | 4a |
| API key | Doug's, single key in GH secret | 5a |
| Preferences file | One `preferences.md` with two named profiles | 6a |
| Notifications | None for now | 7a |
| Implementation pattern | Agentic Claude Code (CLI/SDK) over plain markdown | B |
| Recipe filename | `kebab-case.md` | — |
| Plan/list filename | `YYYY-MM-DD.md` (date of Monday) | — |
| Plugins | Dataview, Templater, Obsidian Git, Recipe View | — |
| Per-action policy | Hybrid (auto-commit low-stakes, confirm high-stakes) | A |
| Push behavior | Auto-push after every commit | A |

## 3. Architecture

### 3.1 Single folder, three roles

A single directory simultaneously serves as:

1. An **Obsidian vault** (`.obsidian/` inside)
2. A **git repository** (`.git/` inside, remote on GitHub private repo)
3. An **iCloud Drive folder** (synced to Alisa's iPhone via Obsidian mobile)

**Canonical path:** `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes/` — this is the special **Obsidian iCloud container** (created by the Obsidian app itself), NOT a regular folder under `iCloud Drive/`. In Finder, it appears as "Obsidian" with the app's purple logo (distinct from any folder named "Obsidian" you might create yourself). Obsidian iOS only detects vaults stored in this container. For convenience, a symlink at `~/Projects/recipes/` points to the canonical container path, so terminal/Claude Code workflows use the familiar Projects path.

The current empty `/Users/dougmarch/Projects/recipes/` directory will be removed and replaced with this symlink during initial setup.

This conflation of roles simplifies sync logic at the cost of a few iCloud-vs-git interaction risks (mitigated; see Section 11).

### 3.2 Three flows

**A. Sunday autonomous (cloud, weekly).**
GitHub Action triggers Sundays at 7:00 AM. Spins up Claude Code headless against the repo, invokes it with `system-prompt.md` as the system prompt. Claude reads `preferences.md`, scans `recipes/*.md` frontmatter, picks 5 dinners, reads full bodies of the picks, writes `plans/YYYY-MM-DD.md` and `lists/YYYY-MM-DD.md`, commits, and pushes.

**B. Interactive (local, ad-hoc).**
Doug opens Claude Code in `~/Projects/recipes/` and chats. Recipe imports, plan tweaks, marking cooked, logging feedback, querying tonight's dinner, scaling, substitutions. Claude reads/writes vault files via tool calls. Auto-commits low-stakes actions; confirms high-stakes (per Section 8 policy table). Auto-push after every commit.

**C. Recipe ingestion (local, ad-hoc).**
Doug pastes a URL to Claude Code. Claude fetches, extracts, normalizes to the schema (Section 5), drops in `recipes/<slug>.md`, auto-commits and pushes. Alisa texts Doug new URLs; Doug runs the same flow.

## 4. Vault structure

```
recipes/
├── .git/
├── .github/
│   └── workflows/
│       └── sunday-plan.yml
├── .obsidian/
│   ├── app.json                     (committed)
│   ├── community-plugins.json       (committed)
│   ├── core-plugins.json            (committed)
│   ├── appearance.json              (committed)
│   ├── plugins/                     (committed: actual plugin code)
│   ├── workspace.json               (gitignored)
│   ├── workspace-mobile.json        (gitignored)
│   ├── workspace.json.bak           (gitignored)
│   ├── cache/                       (gitignored)
│   └── graph.json                   (gitignored)
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-04-24-recipe-vault-design.md   (this file)
├── recipes/                         one .md per recipe, kebab-case slug
├── plans/                           weekly menu, YYYY-MM-DD.md (Monday's date)
├── lists/                           weekly grocery list, same naming as plans
├── tools/
│   └── questionnaire.md             Alisa preference questionnaire template
├── preferences.md                   household profile, dietary, picker rules, feedback log
├── system-prompt.md                 Sunday agent instructions
├── README.md                        orientation note
└── .gitignore
```

### 4.1 Filename conventions

- **Recipes:** `kebab-case-of-dish.md`. Lowercase, hyphens, descriptive.
- **Plans/Lists:** `YYYY-MM-DD.md` where the date is the Monday of that week. Sortable, human-readable, easy to recall ("what did we have in March 2026 → look for `plans/2026-03-*.md`").

### 4.2 `.gitignore`

```
.obsidian/workspace*.json
.obsidian/workspace.json.bak
.obsidian/cache
.obsidian/graph.json
.DS_Store
*.icloud
*.swp
*~
```

### 4.3 Plugins (committed in `community-plugins.json`)

| Plugin | Role |
|---|---|
| Dataview | Auto-updating recipe queries (by cuisine, ingredient, rating) |
| Templater | "New recipe" template insertion |
| Obsidian Git | Optional auto-pull (deferred mitigation; see Section 11.5) |
| Recipe View | Kitchen-friendly recipe cards with scaling |

Meal Plan plugin explicitly **excluded** — its functionality is replaced by the Claude Sunday agent.

## 5. Recipe schema

### 5.1 Frontmatter

```yaml
---
title: Chicken Thighs with Lemon and Rosemary
source: https://nytcooking.com/some-url
servings: 4
cuisine: mediterranean
category: dinner
meal_type: entree
prep_time_min: 10
cook_time_min: 35
total_time_min: 45
difficulty: easy
tags: [weeknight, sheet-pan, leftover-friendly]
diet_tags: [heart-healthy, gluten-free, dairy-free]
key_ingredients: [chicken-thighs, lemon, rosemary, olive-oil, garlic]
nutrition:
  calories: 380
  cholesterol_mg: 145
  saturated_fat_g: 4
  total_fat_g: 22
  sodium_mg: 420
  carbs_g: 8
  sugar_g: 2
  fiber_g: 1
  protein_g: 28
rating: 4
last_cooked: 2026-03-15
times_cooked: 3
added: 2026-01-08
---
```

### 5.2 Field rules

- **Identity:** `title`, `source` (URL/cookbook/personal), `added` (date)
- **Sizing/timing:** `servings`, `prep_time_min`, `cook_time_min`, `total_time_min`
- **Categorization:** `cuisine`, `category` (dinner/lunch/breakfast/snack/side/dessert), `meal_type`, `difficulty` (easy/medium/hard), free-form `tags`
- **Diet:** controlled `diet_tags` vocabulary: `heart-healthy`, `low-sodium`, `low-cholesterol`, `vegetarian`, `vegan`, `gluten-free`, `dairy-free`, `high-protein`, `high-fiber`, `kid-friendly`
- **Search aid:** `key_ingredients` — 3–7 primary ingredients as kebab-case slugs
- **Nutrition:** per-serving; nine fields above. **Never fabricate** — leave blank if source lacks data. If estimated, add `nutrition_estimated: true`.
- **Adaptive:** `rating` (1–5), `last_cooked` (date), `times_cooked` (int)

### 5.3 Body

```markdown
## Ingredients

- 8 bone-in, skin-on chicken thighs
- ...

## Directions

1. Preheat oven to 425°F.
2. ...

## Notes

- Doug: doubled the garlic, much better.
- Free-form, append-only over time, by Doug or via Claude Code logging.
```

Recipe View plugin renders these conventionally.

### 5.4 Image policy

No image files in the vault. The `source` URL field is the canonical image reference — open the source link to see the original. Revisit if needed in Phase 2.

## 6. Preferences schema (`preferences.md`)

```markdown
---
last_updated: 2026-04-24
---

# Household preferences

## General

- Two adults (Doug + Alisa); default 2 servings, scale to 4 for leftovers
- Aim for 5 home dinners/week; Fri and Sun flexible (eating out, leftovers, takeout)
- Variety: don't repeat any dinner within 3 weeks
- Favor seasonal produce when possible

## Doug's profile

### Dietary targets (cardiovascular focus)
- Cholesterol: under 200mg/day (AHA <300mg; want headroom)
- Saturated fat: under 10g/day (AHA <13g)
- Sodium: under 2000mg/day (AHA <2300mg)
- Fiber: target 25g+/day
- Favor: oats, fish (salmon, sardines), legumes, nuts (unsalted), olive oil, leafy greens
- Limit: red meat (max 1x/week), butter, cheese, egg yolks, processed meats, fried foods

### Likes
- Mediterranean, Mexican, Thai, Vietnamese flavors
- Bold spice, garlic, citrus
- One-pot and sheet-pan on weeknights

### Dislikes / avoids
- Cilantro: fine
- Mushrooms: not a fan
- Very sweet entrees

## Alisa's profile

### Dietary targets
- No specific medical targets currently; benefits from same heart-healthy direction
- Lower spice tolerance than Doug

### Likes
- (to be filled via questionnaire — see tools/questionnaire.md)

### Dislikes / avoids
- (to be filled via questionnaire)

## Weekly rhythm

- Mon–Thu: weeknight constraints (≤45 min total time, easy/medium difficulty)
- Fri: flexible (pizza/takeout/something fun)
- Sat: longer-cook recipes OK (1–2 hours)
- Sun: batch-cook (soup/roast that feeds Mon lunch)

## Picker rules

- Don't pick the same protein two nights in a row
- At least one fish night per week
- At least one vegetarian night per week
- Plan around `leftover-friendly` recipes so Day-N+1 lunch is covered
- If a recipe has nutrition fields blank, deprioritize (can't verify it fits the plan)

## Recent feedback (rolling log)

<!-- Append new entries below; agent reads these to adapt picks. No cap for now. -->
```

### 6.1 Update mechanism

- Doug edits manually OR appends via Claude Code interactive sessions
- Sunday agent reads but never writes `preferences.md`
- Recent feedback length: uncapped for now, revisit when usage data is available

## 7. Sunday agent (`system-prompt.md`)

### 7.1 Execution flow

1. Read `system-prompt.md` (instructions)
2. Read `preferences.md` (profile, targets, recent feedback)
3. Glob `recipes/*.md`, read frontmatter only (cheap metadata scan)
4. Glob `plans/*.md` for last 3 weeks; extract recipe links to enforce no-repeat rule
5. Pick 5 dinners for Mon–Thu + Sat:
   - Hard rules: no repeats from last 3 weeks, weekly rhythm, dislikes
   - Soft rules: cuisine variety, recent feedback, prefer higher rating, prefer recipes with nutrition data
   - Nutrition math: averaged per-serving values across the 5 dinners must stay under Doug's daily targets
6. Read full body of just the 5 picked recipes (now have ingredients)
7. Write `plans/YYYY-MM-DD.md` (Monday's date), with rationale per pick + weekly nutrition summary
8. Write `lists/YYYY-MM-DD.md` with categorized aggregated grocery list
9. Do NOT update `last_cooked` or `times_cooked` (humans confirm cooking, not the planner)
10. `git add` + `git commit` + `git push`

### 7.2 Plan output format

`plans/YYYY-MM-DD.md`:

```markdown
---
week_start: 2026-04-27
generated: 2026-04-27T07:00Z
generated_by: claude-sonnet-4-6
---

# Week of April 27, 2026

## Monday — [[chicken-thighs-with-lemon]]
Why: heart-healthy chicken, 45 min total, leftover-friendly for Tue lunch.

## Tuesday — [[salmon-with-roasted-vegetables]]
Why: weekly fish target, omega-3s, simple weeknight prep.

## Wednesday — [[lentil-shepherds-pie]]
Why: vegetarian night, high fiber, comfort-food vibe Alisa likes.

## Thursday — [[turkey-meatballs-with-zoodles]]
Why: lean protein, low-carb, quick on a busy night.

## Saturday — [[braised-short-ribs]]
Why: longer-cook weekend recipe; using red-meat allowance for the week.

## Flexible nights
- Friday: open (pizza/takeout)
- Sunday: leftovers from Sat braise + a green salad

## Weekly nutrition (averages across 5 dinners, per person)
- Calories: 510 / dinner
- Cholesterol: 165mg / dinner (under 200mg target ✓)
- Saturated fat: 6.2g / dinner (under 10g target ✓)
- Sodium: 580mg / dinner (well under 2000mg target ✓)
- Fiber: 8g / dinner (track toward 25g/day ✓)

## Notes
- Red meat allowance used Saturday (short ribs).
- Adjusted away from mushroom-based dishes per Doug's preference.
- Included pasta-adjacent dish (Wed) per Alisa's recent feedback.
```

### 7.3 Grocery list output format

`lists/YYYY-MM-DD.md`:

```markdown
---
week_start: 2026-04-27
generated: 2026-04-27T07:00Z
plan: [[plans/2026-04-27]]
---

# Grocery list — week of April 27, 2026

## Produce
- [ ] Lemons (3)
- [ ] ...

## Proteins
- [ ] Bone-in chicken thighs (8, ~3 lb) — Monday
- [ ] ...

## Pantry / dry goods
- [ ] ...

## Dairy
- [ ] ...

## Frozen
- [ ] ...

## Other
- [ ] ...

## Already on hand (skip if you have)
- Salt, pepper, olive oil, basic herbs

## Notes
- Short ribs benefit from a day of marinating — buy by Friday for Saturday.
```

Categories: Produce, Proteins, Pantry/dry goods, Dairy, Frozen, Other. Duplicates aggregated ("2 lemons" + "1 lemon" → "3 lemons"). Each protein/major item annotated with which night it's for.

### 7.4 System prompt content

Lives at `system-prompt.md`. Editable like any note. Contains the agent's task description, style guidance, and "what NOT to do" rules. Initial draft:

```markdown
# Weekly meal planner — system prompt

You are the Sunday meal planner for Doug and Alisa. Every Sunday morning,
you propose 5 weeknight dinners and produce a grocery list.

## Your task

1. Read `preferences.md` carefully. The household profile, dietary targets,
   weekly rhythm, picker rules, and recent feedback all matter.
2. Scan all `recipes/*.md` files (frontmatter only first). Build a mental
   index of what's available.
3. Pick 5 dinners for the upcoming week (Mon–Thu + Sat, with Fri and Sun
   flexible per the household rhythm). Apply:
   - Hard rules: no repeats from the last 3 weeks (check `plans/` history),
     respect dislikes, respect weekly rhythm and difficulty constraints
   - Soft rules: variety in cuisine and protein, recent feedback signals,
     prefer higher-rated recipes, prefer recipes with nutrition data
   - Nutrition: average the 5 dinners' per-serving values; verify they
     stay under Doug's daily targets when those dinners are eaten
4. Read full bodies of the 5 picked recipes. Aggregate ingredients into a
   categorized grocery list (produce, proteins, pantry, dairy, frozen, other).
   Combine duplicates ("2 lemons" + "1 lemon" → "3 lemons").
5. Write `plans/YYYY-MM-DD.md` (Monday's date) with the menu, rationale per
   pick, and the weekly nutrition summary.
6. Write `lists/YYYY-MM-DD.md` with the categorized grocery list.
7. Commit and push.

## Style

- Explain your reasoning briefly per pick. Doug and Alisa want to see why,
  not just what.
- If a hard constraint can't be satisfied (e.g., no fish recipes available
  but the rhythm calls for one), note it explicitly in the plan rather
  than silently violating.
- If recent feedback contradicts older preferences, recent wins.
- Don't fabricate nutrition data; if a recipe lacks numbers, deprioritize
  but don't invent.

## What NOT to do

- Don't update `last_cooked` or `times_cooked` on recipes — only humans
  confirm "we made this."
- Don't modify `preferences.md` — that's a human-edited file (or updated
  via interactive sessions, not the autonomous job).
- Don't pick the same protein two nights running.
- Don't propose more than 5 dinners; Friday and Sunday are intentionally flexible.
```

This text is committed to the vault and freely editable — every Sunday job re-reads the current version.

### 7.5 Error handling

- **No matching recipes** for a slot (e.g., zero fish recipes available) → write partial plan, flag the gap explicitly in plan notes
- **Anthropic API failure** → GH Action retries 3× with exponential backoff, then fails. GitHub emails Doug on Action failure.
- **Git push conflict** (rare; only if local commit not pulled before Action runs) → Action fails loudly; Doug pulls and re-triggers manually.
- **Nutrition targets impossible to satisfy** → pick closest fit, explicitly note overage in plan ("this week is over Doug's saturated fat target by 1.2g/day; consider reducing red meat allowance")

## 8. Interactive flows

### 8.1 v1 flows (build now)

| Flow | Trigger | Per-action policy |
|---|---|---|
| Recipe ingestion (URL) | "Add this recipe: <URL>" | Auto-commit, show summary |
| Mark "we cooked this" | "We made the chicken tonight" | Auto-commit, no preamble |
| Recent feedback | "The lentil soup was bland" | Auto-commit, show what was added |
| Plan tweak | "Swap Wed for chicken thighs" | Always confirm |
| Recipe deletion | "Delete the Brussels sprouts recipe" | Always confirm |
| Bulk operations | "Estimate nutrition for all recipes" | Always confirm, batched preview |
| `preferences.md` edit | direct edit or "update Doug's profile" | Always confirm |
| `system-prompt.md` edit | "change the picker to do X" | Always confirm |
| Alisa questionnaire | "Run the Alisa questionnaire" | Build profile, confirm before write |
| Recipe browse | Dataview queries in Obsidian (no Claude needed) | n/a |
| "What's for dinner tonight?" | "What are we eating tonight?" | Read-only, no commit |
| Recipe scaling | "Scale Saturday's braise to 6 people" | Always confirm |
| Substitution help | "What can I use instead of rosemary?" | Read-only, no commit |

All commits auto-push to GitHub.

### 8.2 Phase 2 flows (later)

- Fridge inventory mode — manual inventory tracking influences picks
- Cookbook photo import — vision/OCR-based recipe creation from photos
- Monthly retrospective — first of month, agent writes summary of past 4 weeks (lowest priority)

### 8.3 Auto-commit policy summary

Low-stakes (auto-commit + auto-push): recipe imports, marking cooked, recent feedback appends.
High-stakes (always confirm + auto-push after approval): plan tweaks, deletions, bulk ops, preferences edits, system-prompt edits, questionnaire results, scaling, anything destructive or behavior-changing.

## 9. Multi-user setup

### 9.1 Doug

- Creates and owns the GitHub repo (private)
- Sole writer (recipe imports, plan tweaks, preferences updates, system prompt)
- Runs Claude Code interactively on Mac
- iCloud Drive holds the vault folder, "Keep Downloaded" enabled
- Anthropic API key in GitHub repo secret

### 9.2 Alisa

- Read-only on iPhone via Obsidian mobile pointed at the iCloud-synced vault folder
- No GitHub account, no Claude Code, no git knowledge needed
- Sends new recipe URLs to Doug via text/iMessage; Doug runs the import flow
- Profile filled via the questionnaire (see `tools/questionnaire.md`); Doug can run the questionnaire interview-style during a Claude Code session and append results to `preferences.md`

### 9.3 Questionnaire (`tools/questionnaire.md`)

Initial questions:
- What cuisines do you most enjoy?
- What cuisines do you avoid?
- Spice tolerance: mild / medium / hot?
- Top 3 favorite ingredients?
- Ingredients you dislike or avoid?
- Any food allergies or sensitivities?
- Favorite cookbook authors or food blogs?
- Comfort foods / nostalgia foods?
- Eating schedule preferences (lighter dinners? bigger weekends?)
- How adventurous on weeknights?

## 10. Sync strategy

### 10.1 Phase 1 — iCloud single-vault (now)

- Canonical vault location: `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes/` (the Obsidian-app iCloud container, NOT a generic folder in iCloud Drive — Obsidian iOS only detects vaults in this container)
- Symlink for convenience: `~/Projects/recipes/` → canonical iCloud path
- "Keep Downloaded" enabled on the folder (Finder → right-click → Keep Downloaded)
- `.gitignore` excludes per-device Obsidian state files
- Doug's Mac and Alisa's iPhone both see the vault via iCloud (Alisa via Obsidian mobile pointed at the iCloud folder)
- GitHub remote synced via Doug's Mac pushes/pulls
- Sunday Action runs against the GitHub remote (not iCloud); changes flow Action → GitHub → Doug's local pull → iCloud → Alisa

### 10.2 Phase 2 — Obsidian Sync (if triggered)

Trigger conditions:
- Six months of consistent use proving the system valuable, OR
- Any of the six iCloud failure modes (Section 11) becomes a recurring annoyance

Action when triggered: subscribe to Obsidian Sync (~$8/mo), point both devices at it, leave git for cloud-side automation only.

## 11. Failure modes & mitigations

| # | Issue | Mitigation | Residual severity |
|---|---|---|---|
| 1 | `.icloud` placeholder files (eviction) | "Keep Downloaded" on vault folder | Low |
| 2 | `.git/index` corruption | Single-Mac writer; `rm .git/index && git reset` recovery (no work lost) | Low |
| 3 | `.obsidian/workspace.json` conflicts | `.gitignore` workspace files; spouse view-only minimizes write pressure | Cosmetic |
| 4 | Simultaneous edit, silent loss | Spouse read-only; Sunday agent writes only to new files (`plans/`, `lists/`); humans own all other files | Eliminated |
| 5 | Sync lag Sunday morning | Defer auto-pull setup; revisit if pattern of "missing menu at the store" emerges | Acceptable |
| 6 | Small-file performance | Vault stays under ~500 files at projected scale; "Keep Downloaded" mitigates further | Low |

## 12. Cost estimate

- Sunday job (agentic Claude): ~$1–3/month at projected usage
- GitHub Actions: free for private repos (within 2000-min/mo quota; this uses <5 min/run)
- Obsidian: free
- iCloud: assumed already subscribed
- Optional Phase 2: Obsidian Sync ~$8/mo if triggered

Total monthly Phase 1: ~$1–3.
Total monthly Phase 2: ~$9–11.

## 13. Open questions / deferred decisions

- **Sync method long-term:** Phase 2 trigger is real but undefined in advance; Doug decides based on actual usage
- **Recent feedback length cap:** revisit once we see how much accumulates after a few months
- **Phase 2 flow timing:** inventory mode, cookbook photo import, monthly retrospective — order and priority TBD
- **Notifications:** add only if Doug starts forgetting to check the menu Sunday morning

## 14. Out of scope (explicitly)

- Recipe images stored in vault
- Real-time multi-user editing
- Public recipe sharing / blog publishing
- Calorie counting beyond the planner's nutrition awareness
- Calendar integration (Google Calendar, etc.)
- Smart home / shopping cart integration (Amazon Fresh, Instacart)
- Mobile-first Claude experience for Alisa
- Bear / non-Obsidian alternatives (researched and ruled out)

## 15. Implementation phases

**Phase 1 (now):**
1. Initialize vault directory structure (`recipes/`, `plans/`, `lists/`, `tools/`, `docs/`)
2. Set up `.git`, `.gitignore`, `.obsidian/` baseline config + plugins
3. Write `system-prompt.md`, `preferences.md` (Doug's profile, Alisa placeholder), `tools/questionnaire.md`, `README.md`
4. Set up GitHub repo (private), commit + push
5. Configure GitHub Action workflow (`.github/workflows/sunday-plan.yml`) + add Anthropic API key secret
6. Test end-to-end with 5–10 seed recipes
7. Onboard Alisa: install Obsidian on iPhone, point at iCloud folder, run questionnaire

**Phase 2 (triggered):**
- Migrate to Obsidian Sync if iCloud pain materializes
- Build inventory mode, cookbook photo import, monthly retrospective in priority order
- Add notification step to Sunday Action if Doug forgets to look

## 16. Revision history

- 2026-04-24: Initial spec from brainstorming session (Doug + Claude)
- 2026-04-24 (later): Canonical vault path moved from `iCloud Drive/Recipes/` to the Obsidian iCloud container (`~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes/`) after discovering Obsidian iOS only detects vaults stored in the app's own iCloud container, not in a generic `iCloud Drive/Obsidian/` folder. Symlink at `~/Projects/recipes` updated to point to new location. Git, GitHub repo, Sunday Action all unaffected (cloud doesn't care about local paths). One interim attempt put the vault in `iCloud Drive/Obsidian/` (a generic folder named "Obsidian" in iCloud Drive root); this doesn't work because iOS expects the Obsidian-app iCloud container specifically — distinct from any user-created folder of the same name.
