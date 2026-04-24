# TODO — recipe vault project

Outstanding work from the original build, plus deferred ideas. Excludes Alisa's setup (handled separately).

Last updated: 2026-04-24

---

## Should do soon (quick + worth doing)

### 1. Phase 8 — exercise the v1 interactive flows once
**Why:** Confirm everything works on a normal weekday before you need it to. ~15 minutes.

Open Claude Code in `~/Projects/recipes` and try each, in any order:
- "What are we eating tonight?" → reads this week's plan, shows the recipe and ingredients
- "We made the chicken thighs tonight" (or whatever you actually cooked) → marks cooked, increments times_cooked
- "The lentil soup was bland" (or any recipe-specific feedback) → appends to recent feedback in preferences.md
- "Swap Wednesday for something with chicken thighs" → plan tweak with confirmation
- "Scale Saturday's pork shoulder to 6 people, we have guests" → recipe scaling
- "What can I substitute for fresh rosemary?" → substitution help

If any of these misbehave, edit `system-prompt.md` to add a clarifying rule, commit, push.

---

## Could do (improves quality, not blocking)

### 2. Grow the recipe library
**Why:** With only 11 recipes, the Sunday picker is constrained. With 25–30 you'll get meaningfully better variety each week.

- Aim for 20+ recipes total
- Prioritize gaps in current library:
  - More vegetarian recipes (currently only Turkish eggs, which is `category: breakfast`)
  - More fish recipes beyond salmon (e.g., halibut, cod, tuna, shrimp)
  - More Asian recipes
  - At least one vegan recipe
- Use the Web Clipper extension while browsing (Chrome or Safari) — it auto-drops to `recipes-inbox/`
- Periodically run "process the recipe inbox" in Claude Code to normalize them

### 3. Run the Alisa preference questionnaire (when she's available)
**Why:** Currently `## Alisa's profile` in `preferences.md` has placeholder "(to be filled via questionnaire)". The picker doesn't account for her tastes yet.

When she's around: open Claude Code and say *"Run the Alisa questionnaire"*. Claude reads the questions from `tools/questionnaire.md`, you ask them aloud, type her answers back to Claude, and Claude updates `preferences.md`.

(This was originally Phase 7 of the build plan; just hasn't happened yet because she wasn't around.)

---

## Only if needed (don't build proactively)

### 4. Auto-pull launchd job
**Trigger:** You notice that the Sunday plan isn't appearing in Obsidian when you expect it to (e.g., you opened Obsidian Sunday morning but the new plan wasn't there yet).

**Fix:** ~10-line macOS launchd job that runs `git -C ~/Projects/recipes pull` every Sunday at 7:30 AM. Fires whether Obsidian is open or not.

### 5. Sunday notifications
**Trigger:** You find yourself forgetting to look at the Sunday plan / grocery list.

**Fix:** Add a step to `.github/workflows/sunday-plan.yml` that sends an email or push notification when the plan commits. A few extra lines.

### 6. Migration to Obsidian Sync ($8/mo)
**Trigger:** Any of the iCloud failure modes (see spec section 11) becomes a recurring annoyance — most likely candidates are sync lag on the phone or `.obsidian/workspace.json` conflicts.

**Fix:** Subscribe, point both devices at it, leave git purely for cloud automation.

---

## Phase 2 features (build when proven valuable)

In rough priority order. Each is a real feature requiring real design — bring back to brainstorming when ready.

### 7. "Suggest mode" — Claude finds new recipes via web search
**Value:** Easier to grow the library; you can ask "find 3 weeknight chicken recipes I don't have" and get options to approve.

**Effort:** Small. Mostly a Claude prompt. No new infrastructure.

### 8. Fridge inventory mode
**Value:** Sunday plan factors in what you already have, reducing food waste.

**Effort:** Medium. Requires you to occasionally update an inventory file, plus updates to `system-prompt.md`. Most maintenance overhead per use.

### 9. Cookbook photo import
**Value:** Type-able cookbooks become importable.

**Effort:** Larger. Requires Claude vision API setup, page-extraction prompts, error handling for blurry photos.

### 10. Monthly retrospective
**Value:** Patterns over time become visible (e.g., "you made fish 3× this month, never made the lentil soup we said we'd try").

**Effort:** Small to medium. New scheduled Action firing first of every month.

---

## Polish / nits noticed during build

### 11. Recent feedback length cap in `preferences.md`
**Why:** The "Recent feedback (rolling log)" section will grow indefinitely. Eventually the Sunday agent's context will bloat reading it.

**When:** Revisit after a few months when you can see how fast it actually grows.

**Fix:** Probably keep last 20 entries; archive older to `preferences-feedback-archive.md`.

### 12. Weekly nutrition summary table when most recipes lack data
**Why:** The table looks ugly with mostly empty dashes. Now mostly fixed because we estimated nutrition for all 10 original recipes. New recipes added without nutrition could re-introduce the issue.

**Fix:** Edit `system-prompt.md` to either show only available columns, or skip the table entirely when fewer than ~3 picks have full nutrition data. Defer until it actually annoys.

---

## Reference

- Design spec: `docs/superpowers/specs/2026-04-24-recipe-vault-design.md`
- Implementation plan: `docs/superpowers/plans/2026-04-24-recipe-vault-implementation.md`
- User guide: `docs/README.md`
- Inbox processor instructions: `tools/process-inbox.md`
