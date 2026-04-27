# Setup & configuration record

This document is the technical record of how this vault is set up: what's installed where, what's configured to what value, and **why each decision was made**. Pair this with `docs/README.md` (the day-to-day user guide) and `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` (the original design rationale).

If you're setting up a fresh Mac, a fresh iPhone, or trying to understand why something is wired the way it is, start here.

**Last updated:** 2026-04-27 (after the obsidian-git auto-pull change)

---

## 1. Architecture at a glance

```
                 ┌─────────────────────────────┐
                 │  GitHub repo (origin/main)  │
                 │   marchdoe/recipes          │
                 └────────────┬────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
   ┌─────────────────┐ ┌────────────┐ ┌──────────────┐
   │ Sunday cron     │ │   Mac      │ │   iPhone     │
   │ (GH Actions)    │ │            │ │              │
   │ writes plans/   │ │ git push   │ │ READ-ONLY    │
   │ + lists/        │ │ + git pull │ │ (no git)     │
   └─────────────────┘ └─────┬──────┘ └──────▲───────┘
                             │               │
                             └─── iCloud ────┘
                                  Drive
```

Two sync mechanisms, intentionally split:

- **Git (GitHub ↔ Mac)** — version control, history, the channel that the Sunday GitHub Action uses. Mac is the only device that authenticates to git.
- **iCloud Drive (Mac ↔ iPhone)** — file sync between devices. iPhone never talks to git; everything reaches it via iCloud.

The vault's git repo and its iCloud folder are **the same directory** — `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes/`. There's a symlink at `~/Projects/recipes/` for terminal convenience.

---

## 2. Vault paths

| Path | What it is |
|---|---|
| `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes/` | **The actual vault.** Special Obsidian iCloud container — NOT a regular folder under `iCloud Drive/Obsidian/`. Created by Obsidian Mac the first time you make an iCloud vault; only Obsidian iOS detects vaults stored in this container. |
| `~/Projects/recipes/` | Symlink → the path above. Use this for `cd`, Claude Code sessions, and CLI work. |
| `<vault>/.git/` | Git repo. Lives inside the iCloud folder. Yes, that's slightly unusual. iCloud has a few quirks (see §10). |
| `<vault>/.obsidian/` | Obsidian config — plugins, themes, settings. Mostly tracked in git so all devices share the same plugins. Per-device state is gitignored (see §6). |

If you ever need to move the vault, the design spec (§3.1, §11) covers the iCloud-container constraint and the migration path.

---

## 3. Mac setup (fresh machine)

The order matters in a couple of places. Do this top-to-bottom.

### 3.1 Software

| Tool | Why | Install |
|---|---|---|
| Obsidian (Mac) | Editor + viewer for the vault | https://obsidian.md (or `brew install --cask obsidian`) |
| `gh` (GitHub CLI) | Git authentication; PR creation; reading run logs | `brew install gh` |
| Claude Code CLI | Interactive recipe import, processing, plan tweaks | https://claude.com/claude-code |
| `git` | Comes with macOS Xcode CLT; just run `git --version` to trigger CLT install if needed | (built-in or via CLT) |
| Node.js (for Sunday cron only) | The Sunday GitHub Action installs Claude Code CLI via npm — Node is needed there. Mac doesn't strictly need it. | Optional |

### 3.2 Authenticate `gh`

```bash
gh auth login
# choose: GitHub.com → HTTPS → "Login with a web browser"
```

This stores credentials in macOS Keychain. Git operations (clone/fetch/pull/push) automatically use these — no SSH keys, no manual tokens.

### 3.3 Clone the repo into the iCloud container

```bash
# Make sure Obsidian's iCloud container exists. The easiest way is to open
# Obsidian once and create any vault — it'll create the container path.
ls ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/

# Then clone INTO that container, NOT into ~/Projects:
cd ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/
git clone https://github.com/marchdoe/recipes.git Recipes

# Create the convenience symlink:
ln -s ~/Library/Mobile\ Documents/iCloud~md~obsidian/Documents/Recipes ~/Projects/recipes
```

Verify:

```bash
cd ~/Projects/recipes
git status   # should show "On branch main" and clean
ls           # should list recipes/, plans/, lists/, docs/, .obsidian/, etc.
```

### 3.4 Open the vault in Obsidian

- Open Obsidian → "Open existing vault" → pick the `Recipes` folder under iCloud
- Trust the vault when prompted (allows community plugins to load)
- The plugins listed in `.obsidian/community-plugins.json` will install/load automatically:
  - **Dataview** — frontmatter queries (used in some views)
  - **Templater** — recipe-template scaffolding
  - **Recipe View** — pretty rendering of recipe notes
  - **Obsidian Git** — git operations from inside Obsidian (configured to be silent — see §5)

### 3.5 Verify end-to-end

```bash
cd ~/Projects/recipes

# Pull latest (test git auth)
git pull

# Open a known recipe and check the image renders
open recipes/sheet-pan-chicken-fajitas.md
```

In Obsidian, open the same recipe — the hero image should appear at the top.

---

## 4. iPhone setup (Alisa or any read-only viewer)

iPhone is **read-only** by design. It never authenticates to git. iCloud handles all sync.

1. Install **Obsidian** from the App Store (free)
2. First launch → tap **"Open existing vault"** (do **not** "Create new vault")
3. The `Recipes` vault should appear under iCloud — tap it
4. First sync takes 30–60 seconds (downloading from iCloud)
5. Verify by opening `plans/<this-week>.md` and `lists/<this-week>.md`

### What NOT to do on iPhone

- **Don't toggle plugins via Settings → Community plugins.** That writes to `.obsidian/community-plugins.json`, which iCloud-syncs back to Mac and changes Mac's plugin state too. Per-device plugin enable/disable does not work cleanly with iCloud-synced vaults. (See §5.2 for what we do instead.)
- **Don't sign into GitHub on the phone.** The Obsidian Git plugin doesn't need to run on the phone at all — see §5.

### Optional: Web Clipper for clipping recipes from Safari

App Store → "Obsidian Web Clipper" (same listing as the Mac extension). Once installed, the Safari share sheet gets a "Clip to Obsidian" action that drops a Markdown file into `recipes-inbox/`. iCloud propagates it to Mac, where Doug runs *"process the recipe inbox"* in Claude Code to normalize.

---

## 5. The Obsidian Git plugin — configured to be silent

This was the source of two real headaches; the configuration is deliberate.

### 5.1 What the plugin is doing

By default the plugin auto-fetches and auto-pulls from `origin` on Obsidian launch and on a 30-second timer. On Mac, that just works (creds are in Keychain). On iPhone, every launch fired an `Authentication failed. Please try with different credentials` toast, because iOS has no git credentials and we don't want to add them.

### 5.2 Current configuration (as of 2026-04-27)

The plugin is **enabled vault-wide** (in `community-plugins.json`) but its auto-behaviors are **all turned off** in `.obsidian/plugins/obsidian-git/data.json`:

```json
{
  "autoPullOnBoot": false,
  "autoPullInterval": 0,
  "autoPushInterval": 0,
  "autoSaveInterval": 0,
  "autoBackupAfterFileChange": false,
  "disablePopups": true,
  "disablePopupsForNoChanges": true,
  "showErrorNotices": true
}
```

`data.json` is gitignored (see §6) but **iCloud still syncs it** to iPhone — which is exactly what we want. iPhone gets the same "do nothing automatically" config, so no auth attempts on launch.

### 5.3 How to actually pull on Mac

Since the plugin no longer auto-pulls, you have three options when the Sunday cron has committed and you want it locally:

| Option | How |
|---|---|
| **Manual via Obsidian** | Cmd+P → `Obsidian Git: Pull` (or any of the plugin's commands) |
| **Manual via terminal** | `cd ~/Projects/recipes && git pull` |
| **Automated via launchd** | See `docs/TODO.md` item #4 — a ~10-line plist that runs `git pull` every Sunday morning. Not built yet; build only if you actually miss the auto-pull. |

### 5.4 Rejected alternatives (and why)

- **Disable the plugin on iPhone via Community Plugins toggle.** Tried 2026-04-27 — `community-plugins.json` is iCloud-synced, so disabling on iPhone disabled it on Mac too. Whack-a-mole.
- **Set up GitHub credentials on iPhone.** Doable (personal access token in plugin settings) but adds a token-rotation maintenance task and gains nothing — iCloud already syncs files to the phone.
- **Add `.obsidian/community-plugins.json` to `.gitignore`.** Doesn't help — iCloud still propagates the file regardless of git's view.

The current config (plugin on, auto-everything off) is the one that survives both sync mechanisms.

---

## 6. What's gitignored, what's iCloud-synced, what's neither

`.gitignore`:

```
# Per-device Obsidian state
.obsidian/workspace*.json
.obsidian/workspace.json.bak
.obsidian/cache
.obsidian/graph.json

# Per-device plugin settings
.obsidian/plugins/obsidian-git/data.json

# macOS
.DS_Store
*.icloud

# Editor scratch
*.swp
*~
```

What this means in practice:

| File / pattern | Tracked in git? | Synced via iCloud? | Why |
|---|---|---|---|
| `recipes/*.md`, `recipes/_images/*` | Yes | Yes | The actual content — needs to be everywhere |
| `plans/*`, `lists/*` | Yes | Yes | Same |
| `preferences.md`, `system-prompt.md` | Yes | Yes | Same |
| `.obsidian/community-plugins.json` | Yes | Yes | Plugin enable/disable list — all devices use the same set |
| `.obsidian/plugins/<name>/data.json` (most plugins) | Yes | Yes | Plugin settings live here — shared across devices |
| `.obsidian/plugins/obsidian-git/data.json` | **No (gitignored)** | Yes (via iCloud) | Marked per-device but iCloud overrides — used to mean "Mac-specific" but in practice device-shared. The current settings are written so this works either way. |
| `.obsidian/workspace*.json`, `.obsidian/cache`, `.obsidian/graph.json` | No | Yes (best effort) | Truly per-device — pane layouts, cached indexes. Gitignoring keeps them out of commits. iCloud sometimes syncs them anyway, which is harmless because Obsidian regenerates them. |
| `.DS_Store` | No | Yes | macOS Finder metadata. Gitignored to keep commits clean. |

---

## 7. GitHub Actions — Sunday cron

`.github/workflows/sunday-plan.yml` runs every Sunday at 14:00 UTC (= 7:00 AM PT, with ±1hr DST drift the spec accepts). What it does:

1. Checks out the vault
2. Installs Node + Claude Code CLI
3. Runs Claude Code with the vault's `system-prompt.md`, `preferences.md`, and recipe library
4. Claude writes `plans/<Monday's-date>.md` and `lists/<Monday's-date>.md`
5. Commits and pushes back to `main`

### Required secrets

In the GitHub repo settings → Secrets and variables → Actions:

- `ANTHROPIC_API_KEY` — Doug's API key (used by Claude Code in the runner)

That's the only secret. The runner's default `GITHUB_TOKEN` handles the commit + push; permissions are set in the workflow (`contents: write`).

### Manually triggering

```bash
gh workflow run sunday-plan.yml
gh run list --workflow=sunday-plan.yml --limit 5    # see recent runs
gh run view <run-id> --log                           # full logs
```

Cost is roughly **$1–2/month** depending on recipe library size.

---

## 8. Recipe ingestion pipeline

Two paths in, both end up at `recipes/<slug>.md`:

### Path A: paste-a-URL

1. Open Claude Code in `~/Projects/recipes/`
2. Say: *"Add this recipe: <url>"*
3. Claude fetches the page, normalizes to the recipe schema, downloads the OG image to `recipes/_images/<slug>.<ext>`, writes the recipe file, commits, pushes

### Path B: Web Clipper inbox

1. While browsing, click the Obsidian Web Clipper extension → it drops a Markdown file in `recipes-inbox/` (synced via iCloud + git)
2. Whenever you want, open Claude Code and say *"Process the recipe inbox"*
3. Claude follows `tools/process-inbox.md` — re-fetches the source URL, normalizes, downloads the image, writes the recipe, deletes the inbox file, commits

Failed inbox files (incomplete clip, source 404'd, etc.) move to `recipes-needs-review/` instead of being deleted, so nothing is silently lost.

The image-handling specifics live in `docs/superpowers/specs/2026-04-25-recipe-images-design.md`.

---

## 9. Plugin reference

| Plugin | What it does | Notes |
|---|---|---|
| Dataview | Frontmatter queries (`dataview` code blocks) | Used sparingly; could be removed if it ever gets flaky |
| Templater | Recipe scaffolding template | Lightweight |
| Recipe View | Renders recipe notes more attractively | Pure UI, no data side effects |
| Obsidian Git | Git ops from command palette | Configured silent — see §5 |

If you add a plugin: install it on Mac (it gets added to `community-plugins.json`), commit, push. iCloud propagates to iPhone, Obsidian iOS auto-installs it from the community registry on next launch.

---

## 10. iCloud quirks (and what to do about them)

iCloud Drive is mostly fine but has a few rough edges with git repos and active editors.

| Symptom | Cause | Fix |
|---|---|---|
| `.git/index corrupt` error | iCloud raced with a git operation mid-write | `cd ~/Projects/recipes && rm .git/index && git reset` |
| Untracked `*.icloud` files | iCloud tombstones for files not yet downloaded | Already gitignored; harmless. To force download: `cd <folder> && find . -name "*.icloud" -exec brctl download {} \;` |
| Random `Untitled.base` files | Obsidian created a "base" view and forgot about it | Just delete them. If recurring, gitignore `*.base`. |
| Phone shows old version of a file | iCloud sync lag | Pull-to-refresh in Obsidian iOS, or wait a few minutes |
| Mac and phone disagree about a recipe | iCloud sync conflict (rare) | Resolve in Obsidian on Mac, save, re-sync; check git log for unexpected commits |

If iCloud quirks become persistently painful, the documented escape hatch is **Obsidian Sync** ($8/month) — see `docs/TODO.md` item #6.

---

## 11. Decision log

Material configuration decisions, in date order. If you're wondering "why is X this way?" check here first.

| Date | Decision | Rationale |
|---|---|---|
| 2026-04-24 | Vault lives in Obsidian's iCloud container, not a generic iCloud Drive folder | Only the special container is detected by Obsidian iOS |
| 2026-04-24 | Symlink at `~/Projects/recipes` | Terminal/Claude Code convenience without breaking Obsidian's path detection |
| 2026-04-24 | Per-action commit policy: low-stakes auto-commit, high-stakes confirm | Spec §10 — balance speed against blast radius |
| 2026-04-24 | Sunday cron at 14:00 UTC | 7am Pacific with ±1hr DST drift acceptable |
| 2026-04-25 | Recipes get hero images stored in-vault at `recipes/_images/<slug>.<ext>` | Reverses the original "no images in vault" decision; offline-capable; ~150KB × 100 recipes is acceptable git size |
| 2026-04-25 | Image source priority: source-page OG, then inbox-URL, then skip silently | OG is higher-quality and standard; inbox URL is the safety net; placeholder graphics add clutter |
| 2026-04-27 | Obsidian Git auto-pull / auto-fetch / auto-push all set to 0 | iPhone has no git creds; auto-pull caused failed-auth toast every launch; iCloud-synced settings can't be device-specific. Manual pull or future launchd job. |

---

## 12. Recovery cheat sheet

For when something is broken and you need to reset to a known-good state.

| Problem | Recovery |
|---|---|
| Obsidian Git plugin throws "branch not set" or weird state | `rm .obsidian/plugins/obsidian-git/data.json && restart Obsidian`. Plugin will recreate with defaults. |
| Plugin behavior changed unexpectedly | Check `.obsidian/community-plugins.json` for unexpected diff (probably propagated from another device). Revert and commit. |
| Vault won't open on phone | Force-quit Obsidian iOS, reopen. If still broken, sign in/out of iCloud in iOS Settings. |
| Sunday cron failed | `gh run list --workflow=sunday-plan.yml --limit 5` → `gh run view <id> --log`. Most failures are transient API issues; re-trigger with `gh workflow run sunday-plan.yml`. |
| `.git/index corrupt` | `rm .git/index && git reset` |
| Unsure whether Mac is current | `cd ~/Projects/recipes && git pull && git status` |

---

## See also

- `docs/README.md` — day-to-day operations guide for non-setup tasks
- `docs/TODO.md` — outstanding work + deferred ideas
- `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` — original architectural design
- `docs/superpowers/specs/2026-04-25-recipe-images-design.md` — image-system design
- `tools/process-inbox.md` — inbox-processor instructions Claude follows
