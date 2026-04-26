# Recipe Images — Design Spec

**Date:** 2026-04-25
**Status:** Approved for implementation planning
**Owner:** Doug March
**Supersedes:** Section 5.4 ("Image policy") of `2026-04-24-recipe-vault-design.md`

## 1. Purpose

Add a recipe image to every recipe in the vault. Images appear as the first thing rendered when a recipe is opened in Obsidian (desktop or mobile). Images live inside the vault (downloaded from the source page at inbox-processing time) so they sync via iCloud and are available offline.

This reverses the original "no images in vault" decision (2026-04-24 spec § 5.4), which was deferred to Phase 2.

## 2. Decisions summary

| # | Decision | Choice |
|---|---|---|
| Storage | Download into vault (vs. URL-only or hybrid) | Download |
| Image source | Source page's OG image, with inbox-URL fallback | OG-preferred |
| Folder | `recipes/_images/<slug>.<ext>` | Co-located, underscore prefix |
| Embed syntax | Obsidian wikilink with width hint | `![[_images/<slug>.<ext>\|500]]` |
| Placement | First line of body, after frontmatter, before any heading | Hero position |
| Frontmatter field | New `image:` field, value `_images/<slug>.<ext>`, absent if missing | Yes |
| Failure mode | Silent — omit embed and frontmatter field, no placeholder | Silent degradation |
| Backfill | Re-process the existing recipes once to fetch their images | Yes |
| Web Clipper config | No change | Unchanged |
| Git treatment | Images committed to git (not LFS, not gitignored) | In-tree |

## 3. Architecture

### 3.1 Where images live

```
recipes/
  _images/
    sheet-pan-chicken-fajitas.jpg
    salmon-club-salad.jpg
    ...
  sheet-pan-chicken-fajitas.md
  salmon-club-salad.md
  ...
```

One image per recipe, named `<slug>.<ext>` matching the recipe filename. The underscore prefix on `_images/` sorts the folder above the recipe files in Obsidian's file explorer and signals "support assets, not recipes."

### 3.2 How images get there

The inbox processor (`tools/process-inbox.md`) gains a new step. Existing flow:

1. Read inbox file → 2. Re-fetch source URL → 3. Extract recipe → 4. Write `recipes/<slug>.md` → 5. Delete inbox file

New flow inserts step 3.5:

1. Read inbox file → 2. Re-fetch source URL → 3. Extract recipe → **3.5. Extract & download image** → 4. Write `recipes/<slug>.md` (with `image:` frontmatter and embed line if image was downloaded) → 5. Delete inbox file

### 3.3 Image source priority

Step 3.5 in detail:

1. **Try source page's OG image first.** Fetch the source URL's HTML and extract `<meta property="og:image" content="...">`. If found, download it to `recipes/_images/<slug>.<ext>`. Preserve the original extension from the URL (or `Content-Type` header if URL has no extension). Done.
2. **Fall back to the inbox file's embedded image URL.** If OG extraction fails (no tag, fetch error, download error), parse the inbox file body for the first Markdown image (`![alt](url)`) and download that.
3. **If both fail, skip the image.** Don't write the `image:` frontmatter field, don't write the embed line. The recipe is still valid — it just has no picture. Log this in the inbox-processing summary so Doug sees it.

### 3.4 Recipe file layout

Image embed sits between frontmatter and `## Ingredients`, with one blank line on each side:

```markdown
---
title: Easy Sheet Pan Chicken Fajitas
source: https://www.halfbakedharvest.com/sheet-pan-chicken-fajitas/
image: _images/sheet-pan-chicken-fajitas.jpg
servings: 6
...
added: 2026-04-24
---

![[_images/sheet-pan-chicken-fajitas.jpg|500]]

## Ingredients
...
```

Existing parsing contract is preserved: `## Ingredients`, `## Directions`, `## Notes` headings remain at the same level and order. The Sunday meal-planner agent reads frontmatter only and is unaffected.

### 3.5 Why this layout

- **Top of body** so the image is the first rendered element when a note opens. Obsidian hides frontmatter in reading view by default; placing the embed before any heading means it leads.
- **Wikilink with width hint** (`![[...|500]]`) is the only syntax in core Obsidian that supports inline width. Standard Markdown `![alt](url)` ignores width and renders full-pane on mobile.
- **`|500`** caps desktop width at ~500 px (recipe text remains readable next to it in reading view) and mobile auto-scales to viewport.
- **Frontmatter `image:` field** is convention-paired with the embed: same path, present together or absent together. Future programmatic uses (thumbnail browse view, dashboards, the meal-planner showing pictures in plans) read frontmatter and don't have to scan the body.

## 4. Frontmatter schema change

Add one optional field to the recipe schema:

```yaml
image: _images/<slug>.<ext>   # path relative to recipes/, omit if no image
```

Update `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` §5 schema example to include the field. No other fields change.

## 5. Inbox processor change

Update `tools/process-inbox.md`:

- Add a new "Image handling" subsection under "Your task per inbox file" that codifies §3.3 above (OG-first, inbox-URL fallback, silent skip)
- Add the `image:` field to the example frontmatter snippet
- Add the embed line to the example body snippet
- Add image-fetch failures to the summary table (e.g., `⚠ image fetch failed (no OG, no inbox URL)` — recipe still written, just imageless)
- Note that the `_images/` folder may need to be created on first run

## 6. Backfill

Existing 12 recipes have no images. Run a one-shot pass:

For each recipe in `recipes/*.md`:
1. Read the `source:` URL from frontmatter
2. Apply the same image-fetch logic from §3.3 (OG image, no inbox-URL fallback since there's no inbox file — go straight to skip on failure)
3. If an image was downloaded, edit the recipe file to add the `image:` frontmatter field and the embed line at the top of the body
4. If no image, leave the recipe unchanged

Commit as a single batch: `Backfill images for existing recipes`.

## 7. Web Clipper

No changes. The Clipper's contract — drop a Markdown file into `recipes-inbox/` with `source:` in frontmatter — is unchanged. All image work happens downstream in the processor.

## 8. Failure modes

| Scenario | Behavior |
|---|---|
| OG image tag missing on source page | Fall back to inbox URL |
| OG URL returns 404 / network error | Fall back to inbox URL |
| Inbox URL also fails (or no inbox URL — backfill case) | Skip image: write recipe with no `image:` field, no embed |
| Image downloaded but file is 0 bytes / truncated | Treat as failure; fall back or skip |
| Source page has no extractable image at all | Recipe is written without image; flagged in summary |
| Source page changes the image after backfill | Local copy is stale but still valid; not auto-refreshed |
| Doug wants to replace the auto-fetched image with his own photo | Drop the new file at `recipes/_images/<slug>.<ext>` (overwrite). Done. No code change needed because the embed is a convention-based wikilink. |

## 9. Out of scope (intentionally)

- Image optimization / re-encoding (no resize, no jpeg recompression — preserve source bytes)
- Image format normalization (jpg/png/webp all kept as-is)
- Multiple images per recipe (one hero only; if a recipe has step-by-step photos, that's Phase 2)
- A "no image" placeholder graphic (silent degradation chosen instead)
- Image caching / refresh strategy (download once, never refetch)
- LFS / image-only branches (small enough to live in main git history)

## 10. Risks & mitigations

- **Vault size growth.** ~50–200 KB per image × ~100 recipes ≈ 5–20 MB total. Negligible for iCloud and git. Revisit if the library grows past several hundred recipes.
- **Hotlink-protection on source CDNs.** Some sites block direct image fetches without proper `Referer` headers. Mitigation: send a `Referer: <source-page-URL>` header on the image GET request. Document in the processor instructions.
- **Stale images when source updates.** Accepted trade-off; auto-refresh adds complexity for marginal gain.
- **Filename collisions when backfilling on a recipe whose slug already collides with another file.** Recipe slugs are already unique (one recipe per file), so `<slug>.<ext>` in `_images/` cannot collide.

## 11. Acceptance criteria

The change is complete when:

1. `tools/process-inbox.md` documents the new image-fetch step
2. `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` §5.4 is updated to reflect the new policy (or marked as superseded by this spec)
3. The `Healthy Mexican Street Corn Chicken Skillet` inbox file has been processed end-to-end and the resulting recipe file in `recipes/` has both the `image:` frontmatter field and the embed line, with the image file present at `recipes/_images/<slug>.<ext>`
4. The 12 existing recipes have been backfilled where source pages provided usable OG images; recipes whose sources couldn't be fetched are left unchanged and flagged in the commit message
5. Opening any updated recipe in Obsidian (desktop) shows the image as the first rendered element
6. The Sunday meal-planner agent's behavior is unchanged (parsing contract preserved)
