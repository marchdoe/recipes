# Process recipe inbox

Instructions Claude follows when Doug says *"Process the recipe inbox"* in an interactive Claude Code session.

## What's in `recipes-inbox/`

Files clipped from the web by the Obsidian Web Clipper extension. Each clipped file:
- Has a `source` URL in its frontmatter (this is the critical field — never lose it)
- Contains the raw clipped page content as Markdown in the body
- May include a `title`, `clipped` date, and other Web-Clipper-default fields
- Has a filename derived from the page title (probably not yet in our recipe-slug convention)

## Your task per inbox file

For each `.md` file in `recipes-inbox/` (excluding `.gitkeep`):

1. **Read the file.** Find the `source:` URL in frontmatter. If missing, skip the file and tell Doug; we don't trust an inbox file without provenance.
2. **Re-fetch the source URL** via WebFetch. The Web Clipper sometimes captures incomplete content, sidebar junk, or nav menus. A fresh fetch with explicit "extract the recipe" prompting is more reliable than the raw clip.
3. **Extract the recipe** in the same way you would for a normal "add this recipe" flow:
   - Title (clean, human-readable)
   - Servings, prep/cook/total time
   - Cuisine, category, meal_type, difficulty (best-guess from content)
   - Free-form `tags` and controlled `diet_tags` (`heart-healthy`, `low-sodium`, `low-cholesterol`, `vegetarian`, `vegan`, `gluten-free`, `dairy-free`, `high-protein`, `high-fiber`, `kid-friendly`)
   - `key_ingredients` (3–7 kebab-case slugs)
   - Full ingredients list in `## Ingredients` body section
   - Step-by-step `## Directions`
   - `## Notes` section with heart-health caveats relevant to Doug's targets (cholesterol <200mg/day, sat fat <10g/day, sodium <2000mg/day)
4. **Image handling:**
   - Try to extract the source page's OG image first: fetch the source URL and look for `<meta property="og:image" content="...">`. Download it to `recipes/_images/<slug>.<ext>` (preserve the original extension; fall back to Content-Type header if URL has none).
   - If OG extraction fails, fall back to the first Markdown image URL embedded in the inbox file body.
   - Always send a `Referer: <source-page-URL>` header on the image request — some CDNs reject hotlinks without it.
   - If both OG and inbox-URL fail, skip the image: write the recipe without an `image:` frontmatter field and without the embed line. Don't write a placeholder.
   - On success, the recipe gets two additions: an `image: _images/<slug>.<ext>` line in frontmatter, and `![[_images/<slug>.<ext>|500]]` as the very first body line (one blank line above and below it, before any `##` heading).
   - See `docs/superpowers/specs/2026-04-25-recipe-images-design.md` §3.3 for the exact procedure and `docs/superpowers/plans/2026-04-25-recipe-images-implementation.md` for the bash commands.
5. **Nutrition handling:**
   - If the source provides nutrition data, populate the `nutrition:` block with real values, leave `nutrition_estimated` unset (or `false`)
   - If the source provides ONLY calories (typical for many food blogs), populate just `calories:` and leave the rest blank
   - **Do NOT estimate nutrition automatically.** Doug will run the estimate-nutrition flow separately for batches he wants estimated. Inbox processing is just normalization.
6. **Filename:** Convert the title to kebab-case (e.g., "Sticky Asian BBQ Boneless Oven Baked Chicken Wings" → `sticky-asian-bbq-baked-chicken-wings.md`). Drop unnecessary words for length.
7. **Write the normalized file** to `recipes/<slug>.md` using the schema from `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` Section 5.
8. **Delete the inbox file** once the recipe is successfully written.
9. **Show Doug a summary table** of what was processed:
   ```
   Processed 3 inbox items:
     ✓ recipes-inbox/Half Baked Harvest - Salmon Bowls.md
       → recipes/spicy-thai-basil-salmon-bowls.md
     ✓ recipes-inbox/NYT Cooking - Lentil Soup.md
       → recipes/red-lentil-soup-with-lemon.md
     ✓ recipes-inbox/Recipe With Image.md
       → recipes/recipe-with-image.md (+ image)
     ⚠ recipes-inbox/Recipe Without OG Image.md
       → recipes/recipe-without-og.md (no image — source page had no OG tag)
     ⚠ recipes-inbox/some-broken-clip.md
       → skipped: source URL returned 404; left in inbox for review
   ```
10. **Commit** in one batch: `Add N recipes from inbox` — auto-commit per the per-action policy (recipe ingestion is auto-commit). Auto-push.

## When a clip can't be cleanly processed

If processing fails for any reason — re-fetch returns 403/404, the clip is missing key sections (e.g., no ingredients list and re-fetch failed too), the source URL points to a non-recipe page, or the content is otherwise unusable — **move the file to `recipes-needs-review/`** instead of deleting it or trying to write a partial recipe.

```bash
mv "recipes-inbox/<filename>" "recipes-needs-review/<filename>"
```

The failed file should keep its original frontmatter (source URL, title, clipped date) so Doug can later either:
- Manually fix the recipe by visiting the source URL and filling in missing fields
- Delete it if the recipe wasn't worth keeping
- Re-trigger processing if the underlying issue (e.g., site temporarily down) was transient

Include the file in the summary table with a `⚠ moved to needs-review` status and a one-sentence reason.

## What NOT to do

- Don't fabricate nutrition data
- Don't fabricate ingredient quantities — if quantities are missing from the clip and the source can't be re-fetched, move to `recipes-needs-review/` rather than guessing
- Don't process files without a `source:` URL — move them to `recipes-needs-review/` so Doug can inspect
- Don't delete inbox files that failed to process — move them to `recipes-needs-review/` instead
- Don't write a recipe if the page content seems wrong (e.g., the URL points to a navigation page, not a recipe). Move to needs-review and tell Doug.
- Don't combine inbox processing with other vault operations in the same commit — keep the history clean
- Don't generate or fabricate images. If neither OG nor the inbox URL produces a usable image, the recipe is written without one — silent degradation, no placeholder.

## When to run

Doug runs this whenever there's stuff in `recipes-inbox/`. Could be once a day, once a week, or just before a Sunday plan needs more variety. There's no schedule.

## Inspection command

To see what's in the inbox without processing:
```
ls recipes-inbox/
```
