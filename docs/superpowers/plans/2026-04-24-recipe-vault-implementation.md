# Recipe Vault & Meal Planning System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a working Obsidian recipe vault that runs an autonomous Sunday meal-planning job and supports interactive Claude Code tweaks, for Doug + Alisa.

**Architecture:** Single iCloud-Drive folder serves as Obsidian vault + git repo + Alisa's read-only mobile sync target. GitHub Actions invokes Claude Code headlessly each Sunday to write `plans/` and `lists/` notes from the recipe library and `preferences.md`.

**Tech Stack:** Obsidian (vault + plugins: Dataview, Templater, Obsidian Git, Recipe View), git + GitHub (private repo + Actions), Claude Code CLI / claude-agent-sdk (Sunday job + interactive), iCloud Drive (Doug↔Alisa sync), macOS launchd (deferred).

**Reference:** See `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` for the full design.

**Conventions used in this plan:**
- `[USER ACTION]` = step the human must perform manually (Finder, web UI, System Settings)
- `[CLAUDE]` = step Claude can execute via Bash/Write/Edit
- `[CONFIRM FIRST]` = destructive or unusual; per Doug's CLAUDE.md, confirm with user before running

---

## Phase 1 — Foundation (vault location, git, baseline files)

### Task 1: Confirm and execute the vault relocation

**Files:**
- Source: `/Users/dougmarch/Projects/recipes/` (currently contains only `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` and `docs/superpowers/plans/2026-04-24-recipe-vault-implementation.md` — this plan)
- Target: `~/Library/Mobile Documents/com~apple~CloudDocs/Recipes/` (canonical iCloud location)
- Symlink: `/Users/dougmarch/Projects/recipes` → target

**Why:** The spec requires the canonical vault to live in iCloud Drive so Alisa's iPhone can sync via Obsidian mobile. The Projects path becomes a convenience symlink.

- [ ] **Step 1.1: [CONFIRM FIRST] Show the user what will happen and get explicit confirmation**

State to user:
> About to:
> 1. Create `~/Library/Mobile Documents/com~apple~CloudDocs/Recipes/`
> 2. Move `~/Projects/recipes/docs/` into that new folder
> 3. Remove the now-empty `~/Projects/recipes/`
> 4. Create symlink `~/Projects/recipes` → iCloud Recipes folder
>
> The spec doc and this plan will move with `docs/`. No data loss. Confirm to proceed?

Wait for user "yes."

- [ ] **Step 1.2: [CLAUDE] Create the canonical iCloud folder**

Run:
```bash
mkdir -p ~/Library/Mobile\ Documents/com~apple~CloudDocs/Recipes
```
Expected: silent success.

- [ ] **Step 1.3: [CLAUDE] Move docs to the new location**

Run:
```bash
mv /Users/dougmarch/Projects/recipes/docs ~/Library/Mobile\ Documents/com~apple~CloudDocs/Recipes/
```
Expected: silent success.

- [ ] **Step 1.4: [CLAUDE] Remove the now-empty Projects/recipes directory**

Verify it's empty first:
```bash
ls -la /Users/dougmarch/Projects/recipes
```
Expected: only `.` and `..` entries.

Then remove:
```bash
rmdir /Users/dougmarch/Projects/recipes
```
Expected: silent success. (Use `rmdir`, not `rm -rf`; safer — refuses to delete non-empty dir.)

- [ ] **Step 1.5: [CLAUDE] Create the symlink**

Run:
```bash
ln -s ~/Library/Mobile\ Documents/com~apple~CloudDocs/Recipes /Users/dougmarch/Projects/recipes
```

Verify:
```bash
ls -la /Users/dougmarch/Projects/recipes/docs/superpowers/specs/
```
Expected: shows the spec file via the symlink.

---

### Task 2: Initialize git repo at the canonical iCloud location

**Files:**
- Create: `~/Library/Mobile Documents/com~apple~CloudDocs/Recipes/.git/` (via `git init`)

- [ ] **Step 2.1: [CLAUDE] Init git in the canonical location**

Run:
```bash
cd "/Users/dougmarch/Library/Mobile Documents/com~apple~CloudDocs/Recipes" && git init -b main
```
Expected: "Initialized empty Git repository in .../Recipes/.git/"

- [ ] **Step 2.2: [CLAUDE] Verify symlink path also sees the repo**

Run:
```bash
git -C /Users/dougmarch/Projects/recipes status
```
Expected: "On branch main / No commits yet / nothing to commit"

---

### Task 3: Write `.gitignore`

**Files:**
- Create: `~/Library/Mobile Documents/com~apple~CloudDocs/Recipes/.gitignore` (also reachable as `/Users/dougmarch/Projects/recipes/.gitignore`)

- [ ] **Step 3.1: [CLAUDE] Write the file**

Use Write tool, path `/Users/dougmarch/Projects/recipes/.gitignore`, content:

```
# Per-device Obsidian state
.obsidian/workspace*.json
.obsidian/workspace.json.bak
.obsidian/cache
.obsidian/graph.json

# macOS
.DS_Store
*.icloud

# Editor scratch
*.swp
*~
```

---

### Task 4: Write `README.md`

**Files:**
- Create: `/Users/dougmarch/Projects/recipes/README.md`

- [ ] **Step 4.1: [CLAUDE] Write the file**

Use Write tool, path `/Users/dougmarch/Projects/recipes/README.md`, content:

```markdown
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
```

---

### Task 5: Initial commit

- [ ] **Step 5.1: [CLAUDE] Stage and commit foundation files**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git add .gitignore README.md docs/ && git status
```
Expected: shows `.gitignore`, `README.md`, and `docs/` files staged.

- [ ] **Step 5.2: [CLAUDE] Commit**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git commit -m "$(cat <<'EOF'
Initial vault scaffolding: gitignore, README, design docs

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```
Expected: commit succeeds with hash.

---

## Phase 2 — Vault content (preferences, system prompt, questionnaire)

### Task 6: Create directory placeholders

**Files:**
- Create: `recipes/.gitkeep`, `plans/.gitkeep`, `lists/.gitkeep`, `tools/.gitkeep`

These are empty placeholders so git tracks the empty dirs.

- [ ] **Step 6.1: [CLAUDE] Create folders + placeholders**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && mkdir -p recipes plans lists tools && touch recipes/.gitkeep plans/.gitkeep lists/.gitkeep tools/.gitkeep
```
Expected: silent success.

---

### Task 7: Write `preferences.md`

**Files:**
- Create: `/Users/dougmarch/Projects/recipes/preferences.md`

- [ ] **Step 7.1: [CLAUDE] Write the file**

Use Write tool, content:

```markdown
---
last_updated: 2026-04-24
---

# Household preferences

## General

- Two adults (Doug + Alisa); default 2 servings, scale to 4 for leftovers
- Aim for 5 home dinners per week; Fri and Sun flexible (eating out, leftovers, takeout)
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
- One-pot and sheet-pan meals on weeknights

### Dislikes / avoids
- Cilantro: fine
- Mushrooms: not a fan, can substitute
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
- If a recipe has nutrition fields blank, deprioritize (we can't verify it fits the plan)

## Recent feedback (rolling log)

<!-- Append new entries below; agent reads these to adapt picks. No cap for now. -->
```

---

### Task 8: Write `system-prompt.md`

**Files:**
- Create: `/Users/dougmarch/Projects/recipes/system-prompt.md`

- [ ] **Step 8.1: [CLAUDE] Write the file**

Use Write tool, content:

```markdown
# Weekly meal planner — system prompt

You are the Sunday meal planner for Doug and Alisa. Every Sunday morning, you propose 5 weeknight dinners and produce a grocery list.

## Your task

1. Read `preferences.md` carefully. The household profile, dietary targets, weekly rhythm, picker rules, and recent feedback all matter.
2. Scan all `recipes/*.md` files (frontmatter only first). Build a mental index of what's available.
3. Pick 5 dinners for the upcoming week (Mon–Thu + Sat, with Fri and Sun flexible per the household rhythm). Apply:
   - Hard rules: no repeats from the last 3 weeks (check `plans/` history), respect dislikes, respect weekly rhythm and difficulty constraints
   - Soft rules: variety in cuisine and protein, recent feedback signals, prefer higher-rated recipes, prefer recipes with nutrition data
   - Nutrition: average the 5 dinners' per-serving values; verify they stay under Doug's daily targets when those dinners are eaten
4. Read full bodies of the 5 picked recipes. Aggregate ingredients into a categorized grocery list (produce, proteins, pantry, dairy, frozen, other). Combine duplicates ("2 lemons" + "1 lemon" → "3 lemons").
5. Write `plans/YYYY-MM-DD.md` (Monday's date) with the menu, rationale per pick, and the weekly nutrition summary.
6. Write `lists/YYYY-MM-DD.md` with the categorized grocery list.
7. Commit and push.

## Style

- Explain your reasoning briefly per pick. Doug and Alisa want to see why, not just what.
- If a hard constraint can't be satisfied (e.g., no fish recipes available but the rhythm calls for one), note it explicitly in the plan rather than silently violating.
- If recent feedback contradicts older preferences, recent wins.
- Don't fabricate nutrition data; if a recipe lacks numbers, deprioritize but don't invent.

## What NOT to do

- Don't update `last_cooked` or `times_cooked` on recipes — only humans confirm "we made this."
- Don't modify `preferences.md` — that's a human-edited file (or updated via interactive sessions, not the autonomous job).
- Don't pick the same protein two nights running.
- Don't propose more than 5 dinners; Friday and Sunday are intentionally flexible.

## Output format

### Plan (`plans/YYYY-MM-DD.md`)

```yaml
---
week_start: YYYY-MM-DD
generated: <ISO timestamp>
generated_by: <model id>
---
```

Followed by a section per night with `## <Day> — [[<recipe-slug>]]` and a one-line "Why:" rationale, then a "Flexible nights" section, then a "Weekly nutrition" summary table per Doug's targets, then a "Notes" section for any constraint flags or tradeoffs.

### Grocery list (`lists/YYYY-MM-DD.md`)

```yaml
---
week_start: YYYY-MM-DD
generated: <ISO timestamp>
plan: [[plans/YYYY-MM-DD]]
---
```

Followed by category sections in this order: Produce, Proteins, Pantry / dry goods, Dairy, Frozen, Other. Use `- [ ]` checklist syntax. Annotate proteins and major items with the night they're for. End with "Already on hand (skip if you have)" and "Notes" sections.
```

---

### Task 9: Write `tools/questionnaire.md`

**Files:**
- Create: `/Users/dougmarch/Projects/recipes/tools/questionnaire.md`

- [ ] **Step 9.1: [CLAUDE] Write the file**

Use Write tool, content:

```markdown
# Preference questionnaire

Run this to fill in a household member's profile in `preferences.md`. Doug can read these aloud to Alisa, or anyone can fill them in directly.

## Questions

1. What cuisines do you most enjoy?
2. What cuisines do you avoid?
3. Spice tolerance: mild / medium / hot?
4. Top 3 favorite ingredients?
5. Ingredients you dislike or avoid?
6. Any food allergies or sensitivities?
7. Favorite cookbook authors or food blogs?
8. Comfort foods / nostalgia foods?
9. Eating schedule preferences (lighter dinners? bigger weekend meals?)
10. How adventurous on weeknights — try new recipes, or stick to known winners?

## How to use with Claude Code

Open Claude Code in this folder and say: *"Run the Alisa questionnaire."* Claude will ask one question at a time, capture the answers, and propose updates to the `## Alisa's profile` section of `preferences.md` for confirmation.
```

---

### Task 10: Commit Phase 2 content

- [ ] **Step 10.1: [CLAUDE] Stage and commit**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git add preferences.md system-prompt.md tools/ recipes/ plans/ lists/ && git status
```
Expected: shows the new files staged.

- [ ] **Step 10.2: [CLAUDE] Commit**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git commit -m "$(cat <<'EOF'
Add household preferences, Sunday agent prompt, questionnaire, folder scaffold

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 3 — Obsidian setup

### Task 11: Open vault in Obsidian and install plugins

- [ ] **Step 11.1: [USER ACTION] Open the vault**

Tell user:
> 1. Open Obsidian on your Mac
> 2. Click "Open folder as vault"
> 3. Navigate to `~/Projects/recipes` (or the iCloud Recipes folder directly)
> 4. Click Open
> 5. When prompted "Trust author and enable plugins?" — say yes (you wrote the system prompt; you're the author)

- [ ] **Step 11.2: [USER ACTION] Install community plugins**

Tell user:
> In Obsidian: Settings → Community plugins → Browse. Install and enable each of:
> - Dataview
> - Templater
> - Obsidian Git
> - Recipe View
>
> After install, return here and confirm.

- [ ] **Step 11.3: [USER ACTION] Enable "Keep Downloaded" on the vault folder**

Tell user:
> In Finder, navigate to `~/Library/Mobile Documents/com~apple~CloudDocs/`. Right-click the `Recipes` folder → "Keep Downloaded." This pins the folder local; iCloud won't evict files.

- [ ] **Step 11.4: [CLAUDE] Verify Obsidian config files appeared**

Run:
```bash
ls -la /Users/dougmarch/Projects/recipes/.obsidian/ 2>/dev/null || echo "No .obsidian/ folder yet"
```
Expected: shows `app.json`, `community-plugins.json`, etc. If "No .obsidian/ folder yet," user hasn't completed Step 11.1; pause and wait.

---

### Task 12: Commit Obsidian baseline config

**Files:**
- Modify (stage): `.obsidian/app.json`, `.obsidian/community-plugins.json`, `.obsidian/core-plugins.json`, `.obsidian/appearance.json`, plugin folders under `.obsidian/plugins/`

- [ ] **Step 12.1: [CLAUDE] Verify gitignore is excluding the right files**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git status --ignored | head -40
```
Expected: workspace*.json files appear under "Ignored files." If they appear under "Untracked," the gitignore is wrong — fix before committing.

- [ ] **Step 12.2: [CLAUDE] Stage and commit**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git add .obsidian/ && git status
```
Expected: shows committed Obsidian config files; no `workspace*.json`.

```bash
cd /Users/dougmarch/Projects/recipes && git commit -m "$(cat <<'EOF'
Add Obsidian baseline config and community plugins

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 4 — GitHub remote

> **Throughout Phase 4–7:** `<user>` in commands means the implementer's GitHub username. Find it once with `gh api user --jq .login` and substitute everywhere it appears below. `<run-id>` means a workflow run ID returned by the immediately preceding `gh run list` command.

### Task 13: Create the private GitHub repository

- [ ] **Step 13.1: [USER ACTION] Verify gh CLI is installed and authenticated**

Tell user to run in their terminal:
```bash
gh auth status
```
Expected: "Logged in to github.com as <username>". If not, run `gh auth login`.

- [ ] **Step 13.2: [CLAUDE] Create the private repo via gh**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && gh repo create recipes --private --source=. --remote=origin --description "Personal recipe vault and weekly meal planner for Doug & Alisa"
```
Expected: "Created repository <user>/recipes on GitHub" + "Added remote https://github.com/<user>/recipes.git"

- [ ] **Step 13.3: [CLAUDE] Push the existing commits**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git push -u origin main
```
Expected: pushes 3+ commits, sets upstream tracking.

- [ ] **Step 13.4: [CLAUDE] Verify the remote**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git remote -v && git log --oneline -5
```
Expected: shows origin URL + recent commits.

---

## Phase 5 — Sunday GitHub Action

### Task 14: Write the workflow file

**Files:**
- Create: `/Users/dougmarch/Projects/recipes/.github/workflows/sunday-plan.yml`

- [ ] **Step 14.1: [CLAUDE] Create the workflow directory**

Run:
```bash
mkdir -p /Users/dougmarch/Projects/recipes/.github/workflows
```

- [ ] **Step 14.2: [CLAUDE] Write the workflow**

Use Write tool, path `/Users/dougmarch/Projects/recipes/.github/workflows/sunday-plan.yml`, content:

```yaml
name: Sunday meal plan

on:
  schedule:
    # 7:00 AM US Pacific = 14:00 UTC (PST) or 15:00 UTC (PDT).
    # Cron is in UTC. Pick 14:00 UTC; spec accepts ±1hr drift across DST.
    - cron: '0 14 * * 0'
  workflow_dispatch:  # allow manual trigger from GitHub UI

jobs:
  plan-the-week:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: write  # so the action can commit + push back

    steps:
      - name: Checkout vault
        uses: actions/checkout@v4
        with:
          fetch-depth: 50  # need recent plans/ history for no-repeat rule

      - name: Set up Node (for Claude Code CLI)
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Claude Code CLI
        run: npm install -g @anthropic-ai/claude-code

      - name: Configure git identity
        run: |
          git config user.name "Sunday Planner Bot"
          git config user.email "actions@github.com"

      - name: Run the Sunday agent
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          claude \
            --print \
            --model claude-sonnet-4-6 \
            --append-system-prompt "$(cat system-prompt.md)" \
            --max-turns 30 \
            --permission-mode acceptEdits \
            "Run the Sunday meal planning task as described in your system prompt. Today is $(date -u +%Y-%m-%d). The upcoming week starts $(date -d 'next monday' +%Y-%m-%d)."

      - name: Commit and push results
        run: |
          if [ -n "$(git status --porcelain)" ]; then
            git add plans/ lists/
            git commit -m "Sunday plan: week of $(date -d 'next monday' +%Y-%m-%d)"
            git push
          else
            echo "No changes produced — agent may have failed or produced empty plan."
            exit 1
          fi

      - name: Notify on failure
        if: failure()
        run: echo "Sunday plan generation failed; see job logs."
```

---

### Task 15: Add the Anthropic API key as a GitHub secret

- [ ] **Step 15.1: [USER ACTION] Add the secret**

Tell user:
> Go to https://github.com/<your-username>/recipes/settings/secrets/actions
> Click "New repository secret"
> Name: `ANTHROPIC_API_KEY`
> Value: your Anthropic API key (from https://console.anthropic.com/settings/keys)
> Click "Add secret"
> Confirm here when done.

- [ ] **Step 15.2: [CLAUDE] Verify the secret name (cannot read value)**

Run:
```bash
gh secret list --repo <user>/recipes
```
(Substitute actual username; the user can supply or you can derive from `git remote -v`.)

Expected: shows `ANTHROPIC_API_KEY` in the list.

---

### Task 16: Commit and push the workflow

- [ ] **Step 16.1: [CLAUDE] Stage, commit, push**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git add .github/ && git commit -m "$(cat <<'EOF'
Add Sunday meal planning GitHub Action

Cron runs Sundays at 14:00 UTC (~7am Pacific), invokes Claude Code
headlessly with system-prompt.md, commits plans/ and lists/ outputs.
Manual dispatch enabled for testing.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)" && git push
```

---

### Task 17: Manually trigger the Action and verify failure mode (no recipes yet)

- [ ] **Step 17.1: [CLAUDE] Trigger the workflow manually**

Run:
```bash
gh workflow run sunday-plan.yml --repo <user>/recipes
```
Expected: "Created workflow_dispatch event for sunday-plan.yml"

- [ ] **Step 17.2: [CLAUDE] Wait for completion and inspect logs**

Run:
```bash
sleep 30 && gh run list --workflow=sunday-plan.yml --limit 1 --repo <user>/recipes
```

Get the run ID, then:
```bash
gh run view <run-id> --log --repo <user>/recipes | tail -100
```

Expected: agent runs successfully but produces a plan that flags "no recipes available" since `recipes/` is empty (only `.gitkeep`). The plan file may still be written with explicit "no recipes; cannot plan" content. **Either outcome is acceptable for this test** — it confirms the pipeline works.

If agent crashes with an actual error (auth failure, model unreachable, missing dependency): debug, fix, re-run.

---

## Phase 6 — Seed recipes and end-to-end validation

### Task 18: Add 5–10 seed recipes via interactive Claude Code

- [ ] **Step 18.1: [USER ACTION] Provide 5–10 recipe URLs**

Tell user:
> Open Claude Code in `~/Projects/recipes`. Provide 5–10 recipe URLs (NYT Cooking, Serious Eats, AllRecipes, etc.) — ideally a mix of cuisines, including at least one fish and one vegetarian recipe. Tell Claude: *"Add these recipes."*

The interactive Claude Code session will use the recipe ingestion flow from the spec (Section 8.1) — fetch each URL, normalize to the schema, drop in `recipes/`, auto-commit and push.

- [ ] **Step 18.2: [CLAUDE in the new interactive session] For each URL, perform recipe ingestion**

Per recipe URL, the recipe-ingestion flow:
1. Fetch the URL via WebFetch
2. Extract: title, ingredients, directions, servings, source URL
3. Estimate (best guess from content): cuisine, category, difficulty, prep_time_min, cook_time_min, total_time_min, key_ingredients (3–7 slugs), tags, diet_tags
4. **Leave `nutrition` fields empty** unless the source page lists them; never fabricate
5. Pick a kebab-case filename: lowercase, hyphens, descriptive
6. Write `recipes/<slug>.md` with the full schema (see spec Section 5)
7. Show user the frontmatter for review; on approval, `git add` + `commit` + `push`

- [ ] **Step 18.3: [CLAUDE] Verify the recipes are committed and pushed**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && ls recipes/ && git log --oneline -10
```
Expected: 5–10 `.md` files, recent commits per recipe.

---

### Task 19: Trigger Sunday Action against real recipes

- [ ] **Step 19.1: [CLAUDE] Trigger the workflow**

Run:
```bash
gh workflow run sunday-plan.yml --repo <user>/recipes
```

- [ ] **Step 19.2: [CLAUDE] Wait and inspect**

Wait ~3–5 minutes, then:
```bash
gh run list --workflow=sunday-plan.yml --limit 1 --repo <user>/recipes
gh run view <run-id> --log --repo <user>/recipes | tail -200
```
Expected: agent reads recipes, picks 5, writes `plans/<monday-date>.md` and `lists/<monday-date>.md`, commits, pushes.

- [ ] **Step 19.3: [CLAUDE] Pull and inspect outputs**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git pull && cat plans/$(ls plans/*.md | tail -1) && echo "---" && cat lists/$(ls lists/*.md | tail -1)
```
Expected: well-formed plan and list per the spec's output format.

- [ ] **Step 19.4: [USER ACTION] Sanity-check the output**

Tell user:
> Open the generated `plans/YYYY-MM-DD.md` and `lists/YYYY-MM-DD.md` in Obsidian. Read them. Things to look for:
> - Did it follow the weekly rhythm (5 dinners, Mon–Thu + Sat)?
> - Did it respect Doug's nutrition targets in the summary?
> - Is the rationale per pick reasonable?
> - Is the grocery list categorized and de-duplicated?
> - Are there any obvious mistakes (wrong category, fabricated nutrition, etc.)?
>
> Report back any issues — they likely indicate `system-prompt.md` needs tightening.

---

### Task 20: Iterate on the system prompt if needed

- [ ] **Step 20.1: [CLAUDE-via-interactive-session] Apply user's feedback to `system-prompt.md`**

If user reports issues with the Sunday output, edit `system-prompt.md` to add explicit rules addressing the problem. Per the spec's per-action policy, this is a high-stakes edit — show the diff, confirm with user, then commit + push.

- [ ] **Step 20.2: [CLAUDE] Re-trigger the Action and re-inspect**

Repeat Task 19 to verify the fix. Iterate until user is satisfied.

---

## Phase 7 — Alisa onboarding

### Task 21: Install Obsidian on Alisa's iPhone

- [ ] **Step 21.1: [USER ACTION] Install Obsidian mobile**

Tell user:
> On Alisa's iPhone:
> 1. App Store → search "Obsidian" → install
> 2. Open Obsidian → "Create new vault"... wait — instead, "Open existing vault"
> 3. Choose "iCloud Drive" → navigate to `Recipes/` → select
> 4. Vault opens. May take a minute to download files first time.

- [ ] **Step 21.2: [USER ACTION] Verify Alisa can read a plan and a list**

Tell user:
> Have Alisa navigate to `plans/<this-week>.md` and `lists/<this-week>.md` on her phone. Confirm she can read both clearly.

---

### Task 22: Run the Alisa preference questionnaire

- [ ] **Step 22.1: [CLAUDE-via-interactive-session] Conduct the interview**

In Claude Code on Doug's Mac, say: *"Run the Alisa questionnaire."*

Claude reads `tools/questionnaire.md`, asks each of the 10 questions one at a time. Doug reads the question to Alisa, captures her answers, types them back to Claude.

- [ ] **Step 22.2: [CLAUDE-via-interactive-session] Update preferences.md**

Build out the `## Alisa's profile` section of `preferences.md` with her answers. Per the per-action policy, `preferences.md` edits are high-stakes — show Doug the proposed diff, confirm, then auto-commit + push.

- [ ] **Step 22.3: [CLAUDE] Verify push**

Run:
```bash
cd /Users/dougmarch/Projects/recipes && git log --oneline -3
```
Expected: most recent commit is the preferences update.

---

## Phase 8 — Final verification

### Task 23: Confirm the full hybrid loop works

- [ ] **Step 23.1: Verify all v1 interactive flows are usable**

Tell user to try each, in any order, in a Claude Code session:
- *"What are we eating tonight?"* → Claude reads this week's plan, names the recipe, shows ingredients
- *"Scale Saturday's recipe to 6 people"* → Claude rewrites that recipe's ingredients
- *"What can I substitute for fresh rosemary?"* → Claude proposes substitutes (read-only, no commit)
- *"Add this recipe: <URL>"* → ingestion works
- *"We made the chicken thighs tonight"* → marks cooked, increments times_cooked
- *"The lentil soup was bland"* → appends to recent feedback in preferences.md
- *"Swap Wednesday for something with chicken"* → plan tweak with confirmation

If any flow misbehaves, edit `system-prompt.md` to clarify the expected behavior, commit, push.

- [ ] **Step 23.2: Confirm Sunday Action is scheduled**

Run:
```bash
gh workflow list --repo <user>/recipes
```
Expected: `sunday-plan.yml` appears with status "active."

- [ ] **Step 23.3: Document any deviations from the spec**

If implementation surfaced anything that requires updating the spec (different file paths, additional fields, refined picker rules), edit `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` and commit.

---

## Self-review checklist

Done before declaring this plan complete:

- [x] Every spec section has a corresponding task
  - Section 3 (architecture) → Tasks 1, 2, 14
  - Section 4 (vault structure) → Tasks 3, 6
  - Section 5 (recipe schema) → Task 18 (ingestion uses the schema)
  - Section 6 (preferences) → Task 7
  - Section 7 (Sunday agent) → Tasks 8, 14, 17, 19
  - Section 8 (interactive flows) → Tasks 18, 23
  - Section 9 (multi-user) → Tasks 21, 22
  - Section 10 (sync) → Tasks 1 (iCloud), 11.3 (Keep Downloaded), 22 (Alisa)
  - Section 11 (failure modes) → mitigations baked into Tasks 3 (gitignore), 11.3 (Keep Downloaded), 14 (workflow retry via re-run)
  - Section 12 (cost) → no task; informational only
  - Section 13 (open questions) → no tasks; deferred by design
  - Section 14 (out of scope) → not implemented; correct
  - Section 15 (implementation phases) → mirrors plan phases
- [x] No placeholders ("TBD", "implement later", etc.)
- [x] Every step has either exact content or exact commands
- [x] Destructive operations have [CONFIRM FIRST] markers (Task 1)
- [x] User-action and Claude-action steps are distinguished
- [x] Filename and path references are consistent throughout (`/Users/dougmarch/Projects/recipes` everywhere)
- [x] Frequent commits (every Phase ends with a commit)

---

## Out of scope for this plan (deferred to Phase 2 of the spec)

- Fridge inventory mode
- Cookbook photo import (vision/OCR)
- Monthly retrospective
- Auto-pull launchd job (only if Sunday lag pain emerges)
- Migration to Obsidian Sync (only if iCloud failure modes recur)
- Notifications when Sunday plan drops

These are documented in spec Section 13 and can be planned separately when triggered.
