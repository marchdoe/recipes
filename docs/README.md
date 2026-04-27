# Recipes vault — user guide

This is the everyday operations guide for the recipe vault: how the system works, what to do each week, common tasks, and how to set up another device. For the design rationale and architecture, see `superpowers/specs/2026-04-24-recipe-vault-design.md`.

## What this is

An Obsidian vault that automatically proposes a weekly dinner menu and grocery list every Sunday, then lets you adjust it interactively during the week. Built for Doug + Alisa, with Doug as the curator and Alisa as a read-only viewer on her phone.

## How it works (plain language)

Three things working together:

1. **A folder of recipe Markdown files** — one `.md` per recipe, with structured metadata (cuisine, servings, nutrition, etc.) in the frontmatter and the actual ingredients/directions in the body. The folder is in iCloud Drive.

2. **A git repository on GitHub** — the same folder is also a private GitHub repo. Every change is version-controlled and pushed to GitHub, where the cloud automation can see it.

3. **A GitHub Action that runs every Sunday** — at 7:00 AM Pacific, GitHub spins up a Claude Code agent in the cloud, gives it the system prompt + preferences + the recipe library, and it writes a weekly menu and grocery list into the repo. Cost: ~$1–2/month.

The same Claude Code can also be opened locally on Doug's Mac at any time to import new recipes, swap meals, log feedback, ask "what's for dinner tonight?" etc.

## How files flow between devices

```
[Sunday Action runs in the cloud]
      ↓ commits + pushes
[GitHub remote]
      ↓ Doug pulls manually (Cmd+P → "Obsidian Git: Pull", or `git pull` in terminal)
[Doug's Mac local folder = iCloud Recipes]
      ↓ iCloud Drive auto-syncs (a few minutes)
[Alisa's iPhone via Obsidian mobile]
```

Doug pulls when he wants the latest — typically once Sunday morning to get that week's plan. iCloud propagates to the phone automatically afterwards. The Obsidian Git plugin used to auto-pull on a 30-second timer, but that was disabled on 2026-04-27 because it caused authentication errors on the iPhone (which has no git credentials). See `SETUP.md` §5 for the full story.

## Folder structure

| Folder/File | What's in it |
|---|---|
| `recipes/` | One `.md` per recipe; frontmatter holds metadata + nutrition; body holds ingredients/directions/notes |
| `recipes/_images/` | Hero images, one per recipe (matched by slug). Downloaded automatically when recipes are imported. |
| `recipes-inbox/` | Web Clipper drops clipped pages here; `"Process the recipe inbox"` normalizes them into `recipes/` |
| `recipes-needs-review/` | Inbox files that couldn't be auto-processed (incomplete clip, site blocks re-fetch, etc.) — Doug fixes manually or deletes |
| `plans/YYYY-MM-DD.md` | Weekly menu (date is the Monday of that week); written by the Sunday Action |
| `lists/YYYY-MM-DD.md` | Paired grocery list; written by the Sunday Action |
| `preferences.md` | Household profile: Doug's + Alisa's dietary targets, weekly rhythm, picker rules, recent feedback |
| `system-prompt.md` | The instructions the Sunday agent reads every run; editable like any note |
| `tools/questionnaire.md` | Preference-interview template for filling in profiles |
| `.github/workflows/sunday-plan.yml` | The Sunday cron job |
| `docs/` | This guide + design spec + implementation plan |
| `.obsidian/` | Obsidian config (plugins, theme); per-device workspace state is gitignored |

## Common tasks (Doug)

All of these happen by opening Claude Code in `~/Projects/recipes/` and chatting:

| What you want | What you say |
|---|---|
| Add a new recipe from a URL | *"Add this recipe: <url>"* |
| Process recipes you clipped via the browser extension | *"Process the recipe inbox"* (see "One-click clipping" below) |
| Mark a recipe as cooked tonight | *"We made the chicken thighs tonight"* |
| Log feedback on a recent meal | *"The lentil soup was bland; we won't make it again"* |
| Swap a meal in this week's plan | *"Swap Wednesday for something with chicken thighs"* |
| Ask what's for dinner tonight | *"What are we eating tonight?"* |
| Scale a recipe for guests | *"Scale Saturday's braise to 6 people, we have guests"* |
| Get a substitution suggestion | *"What can I use instead of fresh rosemary?"* |
| Estimate nutrition for a recipe | *"Estimate nutrition for the salmon-club-salad recipe"* |
| Run the Alisa questionnaire | *"Run the Alisa questionnaire"* |

The agent follows a per-action policy: low-stakes things (recipe imports, feedback logging) auto-commit; high-stakes things (plan edits, preferences updates, system prompt changes) always ask for confirmation first. Either way, every commit auto-pushes to GitHub immediately.

## The Sunday weekly flow

1. **Sunday morning** — GitHub Action runs at 7:00 AM Pacific, takes ~5 minutes
2. **Doug pulls when he sits down with Obsidian** — Cmd+P → `Obsidian Git: Pull`, or `git pull` in terminal
3. **iCloud syncs to Alisa's phone** — within minutes of the Mac pulling
4. **At the grocery store** — Alisa opens Obsidian, taps `lists/YYYY-MM-DD.md` for this week's grocery list with checkboxes
5. **At dinner time** — open `plans/YYYY-MM-DD.md`, tap the linked recipe for ingredients and directions

The pull is manual now (was auto-pull until 2026-04-27 — see `SETUP.md` §5). One keystroke or one `git pull`. If you forget, the plan still appears the next time you do pull — as long as that's before the grocery run, you're fine.

## Setting up Obsidian on Alisa's iPhone

One-time setup. Takes about 5 minutes including initial sync.

1. **Install Obsidian:**
   - Open the App Store
   - Search "Obsidian"
   - Install (it's free) and open it

2. **Open the existing iCloud vault** (do NOT "Create new vault"):
   - On the first screen, tap **"Open existing vault"** or **"Add iCloud vault"** (wording varies by version)
   - The "Recipes" vault should appear in the iCloud vaults list
   - **Important:** Obsidian iOS only detects vaults stored in the app's own iCloud container — that's a special location, NOT a regular folder you'd see in the Files app under "iCloud Drive". Our vault lives in that container (real path: `iCloud~md~obsidian/Documents/Recipes/`), which the app finds automatically.
   - Tap **Recipes**

3. **Wait for files to download:**
   - First-open takes 30–60 seconds while iCloud downloads everything to the phone
   - This is a one-time cost — subsequent opens are instant

4. **Verify it works:**
   - Tap the file explorer icon (top-left, or swipe from left)
   - Navigate to `plans/` → tap the most recent file (e.g., `2026-04-27.md`) → should show this week's menu
   - Navigate to `lists/` → tap the most recent file → should show the grocery list with checkable boxes
   - Both should render with proper headings, tables, and links

5. **Optional — pin the vault for quick access:**
   - In iOS, you can add the Obsidian app to the home screen for one-tap access
   - You can also long-press the Obsidian icon → "Recipes" appears as a quick-action shortcut

If something goes wrong (vault not appearing, files not downloading, formatting broken), the most common fixes are:
- Wait longer — first sync is genuinely slow on cellular
- Force-quit Obsidian and reopen
- Check that the iPhone is signed into the same iCloud account that owns the Recipes folder

## One-click clipping (browser extension)

For recipes you find while browsing, instead of copying URLs to Claude Code, you can use the **Obsidian Web Clipper** extension. It's the official browser extension from Obsidian.

**The flow:**
1. While browsing, see a recipe you want → click the Web Clipper extension icon → it saves the page to `recipes-inbox/<title>.md` in the vault
2. Periodically (whenever you want, no schedule), open Claude Code in the vault and say: *"Process the recipe inbox"*
3. Claude reads each inbox file, re-fetches the source URL, normalizes to the recipe schema (with cuisine, key_ingredients, diet_tags, etc.), writes to `recipes/`, deletes the inbox entry, commits

**Where to install:**
- **Mac (Chrome/Brave/Edge/Arc):** [Chrome Web Store](https://chromewebstore.google.com/detail/obsidian-web-clipper/cnjifjpddelmedmihgijeibhnjfabmlf)
- **Mac (Safari):** [Mac App Store](https://apps.apple.com/us/app/obsidian-web-clipper/id6720708363)
- **iPhone (Safari):** same App Store listing — works as both an app and a Safari extension

**How Alisa contributes recipes:** she installs Web Clipper on her iPhone Safari, taps the share/extension button on any recipe she likes, the clip lands in the vault's inbox via iCloud sync. Doug runs *"process the recipe inbox"* later to normalize and import them.

**Rule of thumb:** clip when you're browsing, paste-URL-to-Claude-Code when you've already got the URL in hand. Either flow eventually produces the same normalized recipe in `recipes/`.

## What Alisa can do

In v1, Alisa is read-only on her phone. She can:
- Open `plans/<this-week>.md` to see what's for dinner each night
- Open `lists/<this-week>.md` and check off groceries as she shops
- Tap any `[[recipe-link]]` to see the full recipe (ingredients, directions, notes)
- Browse `recipes/` to see the full library

She can't directly add recipes or edit anything that gets pushed back. To add a recipe she likes, she sends Doug the URL and he runs the import flow.

A v2 enhancement (see "Open ideas" below) is a "suggest mode" where Doug can ask Claude to find recipes matching constraints, with Alisa contributing search criteria.

## Maintaining the system

| When | Do |
|---|---|
| Whenever you find a recipe you want to try | Send Doug the URL or run the import flow |
| After cooking a meal | Mark it cooked + log any feedback |
| When tastes/needs change | Edit `preferences.md` (or ask Claude to update it) |
| When the agent does something dumb | Edit `system-prompt.md` to add a clarifying rule, or tell Claude in conversation |
| Monthly-ish | Skim recent plans to see if variety is good; if not, add more recipes in under-covered categories |

## Costs

- **Sunday Claude API call:** ~$1–2/month depending on recipe library size
- **GitHub:** free for private repos with this level of Action minutes
- **Obsidian:** free
- **iCloud Drive:** assumes you already have it
- **Optional later:** Obsidian Sync (~$8/mo) if iCloud sync starts being painful

Total: about a buck or two a month.

## Open ideas (Phase 2, when needed)

Things deliberately not built yet but available to add later:

- **Fridge inventory mode** — track what's in the fridge; Sunday agent factors it in to reduce waste
- **Cookbook photo import** — snap a photo of a cookbook page; Claude OCRs and creates the recipe
- **Monthly retrospective** — first of every month, agent summarizes what was eaten and feedback patterns
- **"Suggest mode"** — Claude searches the web for recipes matching constraints; you approve which to import
- **Auto-pull launchd job** — fires `git pull` even when Obsidian is closed (only matters if you find yourself missing Sunday plans)
- **Notifications** — push notification or email when Sunday plan drops
- **Migration to Obsidian Sync** — if iCloud sync becomes painful (e.g., persistent conflict files, slow on phone)

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Sunday plan didn't appear in Obsidian | Manual pull hasn't run yet | Cmd+P → `Obsidian Git: Pull`, or `git pull` in terminal |
| Alisa's phone not showing new files | iCloud sync lag (only fires after Mac pulls) | Make sure Mac has pulled; pull-to-refresh in Obsidian; check WiFi |
| `Authentication failed` toast on iPhone Obsidian | Obsidian Git plugin trying to fetch | Should be impossible after 2026-04-27 config; if it returns, see `SETUP.md` §5 |
| `Aborted, branch not set` error | Plugin in stale state | `rm .obsidian/plugins/obsidian-git/data.json` and restart Obsidian (see `SETUP.md` §12) |
| Sunday Action failed | API key issue, model error, etc. | Check https://github.com/marchdoe/recipes/actions for the failed run logs |
| Recipe imported with wrong cuisine/category | Claude best-guess was off | Open the recipe `.md`, edit the frontmatter, commit |
| Plan picked something that violates preferences | System prompt needs tightening | Edit `system-prompt.md` to add an explicit rule for that case |
| `.git/index corrupt` error | iCloud + git race condition | `cd ~/Projects/recipes && rm .git/index && git reset` |
| `Untitled.base` or other random files appear | Obsidian auto-created them | Delete them; consider gitignoring `*.base` if it keeps happening |

## Where to learn more

- **Setup & configuration record:** `docs/SETUP.md` — what's installed where, how plugins are configured, why each decision was made, recovery cheat sheet
- **Design spec:** `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` — full architectural reasoning, all decisions made and why
- **Image system design:** `docs/superpowers/specs/2026-04-25-recipe-images-design.md` — recipe-image storage and OG-fetching policy
- **Implementation plans:** `docs/superpowers/plans/` — step-by-step build records
- **Vault root README:** `~/Projects/recipes/README.md` — short pointer to this guide
