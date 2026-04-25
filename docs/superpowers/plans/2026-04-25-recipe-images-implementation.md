# Recipe Images Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add hero images to every recipe in the vault — automatically downloaded from the source page, embedded at the top of each recipe note in Obsidian.

**Architecture:** No code is written. The system's "implementation" is two markdown documents: `tools/process-inbox.md` (instructions Claude follows when processing the inbox) and `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` (the canonical design). Updating those docs *is* the feature. After the docs are updated, this same plan executes the new flow against (a) the one inbox file currently waiting and (b) the existing 12 recipes as a backfill — proving the docs are correct and producing the visible result the user asked for.

**Tech Stack:** `bash` (`curl`, `grep`, `sed`), Markdown editing, git. No new tools, no scripting language, no build step.

**Spec:** `docs/superpowers/specs/2026-04-25-recipe-images-design.md`

**Branch:** Already on `feature/recipe-images` from spec commit.

---

## File Structure

| File | Action | Purpose |
|---|---|---|
| `tools/process-inbox.md` | Modify | Document the new image-fetch step in the inbox-processor instructions |
| `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` | Modify | Mark §5.4 superseded; update §5 schema to include `image:` field |
| `recipes/_images/` | Create (new dir) | Holds image files |
| `recipes/_images/*.{jpg,png,webp}` | Create (new files) | One per recipe (where source has a usable image) |
| `recipes/*.md` | Modify (12 files) | Add `image:` frontmatter and embed line for backfilled recipes |
| `recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md` | Delete (after processing) | Inbox file is consumed when normalized into `recipes/` |
| `recipes/healthy-mexican-street-corn-chicken-skillet.md` | Create | Output of inbox processing |

---

## Reusable procedure: fetch image for a recipe

Several tasks below invoke this procedure. Defining it once.

**Inputs:**
- `SOURCE_URL`: the recipe's `source:` URL
- `SLUG`: the recipe filename without `.md` (e.g., `sheet-pan-chicken-fajitas`)
- `INBOX_BODY`: full text of the inbox file (only when processing inbox; absent for backfill)

**Procedure:**

1. **Try to extract the OG image URL from the source page:**

```bash
OG_URL=$(curl -sL --max-time 15 -A "Mozilla/5.0" "$SOURCE_URL" \
  | grep -oE '<meta[^>]+property=["'"'"']og:image["'"'"'][^>]*>' \
  | head -1 \
  | grep -oE 'content=["'"'"'][^"'"'"']+["'"'"']' \
  | sed -E 's/content=["'"'"']//; s/["'"'"']$//')
echo "OG image URL: ${OG_URL:-<none>}"
```

If `OG_URL` is empty, fall to step 2. Otherwise jump to step 3.

2. **Fall back to the inbox file's first Markdown image URL** (skip this for backfill):

```bash
INBOX_IMG_URL=$(grep -oE '!\[[^]]*\]\(([^)]+)\)' <<<"$INBOX_BODY" \
  | head -1 \
  | sed -E 's/.*\(([^)]+)\)/\1/')
echo "Inbox image URL: ${INBOX_IMG_URL:-<none>}"
```

If both `OG_URL` and `INBOX_IMG_URL` are empty, skip the image entirely — the recipe is still valid without one.

3. **Determine the file extension:**

```bash
IMAGE_URL="${OG_URL:-$INBOX_IMG_URL}"
# Strip query string, then take everything after the last dot
EXT=$(echo "$IMAGE_URL" | sed -E 's/\?.*$//' | grep -oE '\.[a-zA-Z0-9]+$' | tail -c +2 | tr A-Z a-z)
# Sanity: only accept image extensions
case "$EXT" in
  jpg|jpeg|png|webp|gif) ;;
  *) EXT="" ;;
esac
# If extension wasn't usable, ask the server via Content-Type
if [ -z "$EXT" ]; then
  CT=$(curl -sLI --max-time 10 -A "Mozilla/5.0" -H "Referer: $SOURCE_URL" "$IMAGE_URL" \
    | grep -i '^content-type:' | tail -1 | tr -d '\r' | awk '{print $2}')
  case "$CT" in
    image/jpeg) EXT="jpg" ;;
    image/png)  EXT="png" ;;
    image/webp) EXT="webp" ;;
    image/gif)  EXT="gif" ;;
    *) EXT="jpg" ;;  # last-resort default
  esac
fi
echo "Extension: $EXT"
```

4. **Download the image with a Referer header** (some CDNs require this):

```bash
mkdir -p recipes/_images
DEST="recipes/_images/${SLUG}.${EXT}"
curl -sL --max-time 30 -A "Mozilla/5.0" -H "Referer: $SOURCE_URL" "$IMAGE_URL" -o "$DEST"
# Verify it's a real image and non-empty
SIZE=$(stat -f%z "$DEST" 2>/dev/null || stat -c%s "$DEST" 2>/dev/null)
if [ "${SIZE:-0}" -lt 1024 ]; then
  echo "Image too small (${SIZE} bytes), discarding"
  rm "$DEST"
fi
```

5. **Result:** either `recipes/_images/<slug>.<ext>` exists and is ≥1 KB (success), or no file exists (skip — the recipe will be written without `image:` field and embed line).

---

## Task 1: Update `tools/process-inbox.md` with the new image-fetch step

**Files:**
- Modify: `tools/process-inbox.md`

- [ ] **Step 1: Read the file to find the right insertion points**

Run: `wc -l "tools/process-inbox.md"` (expected: ~80 lines)

Identify two regions to edit:
- "Your task per inbox file" numbered list (currently steps 1–9)
- "What NOT to do" section

- [ ] **Step 2: Insert a new step 4 ("Image handling") into the numbered list**

Renumber existing 4–9 to 5–10. Insert this between current step 3 and step 4 (Nutrition handling):

```markdown
4. **Image handling:**
   - Try to extract the source page's OG image first: fetch the source URL and look for `<meta property="og:image" content="...">`. Download it to `recipes/_images/<slug>.<ext>` (preserve the original extension; fall back to Content-Type header if URL has none).
   - If OG extraction fails, fall back to the first Markdown image URL embedded in the inbox file body.
   - Always send a `Referer: <source-page-URL>` header on the image request — some CDNs reject hotlinks without it.
   - If both OG and inbox-URL fail, skip the image: write the recipe without an `image:` frontmatter field and without the embed line. Don't write a placeholder.
   - On success, the recipe gets two additions: an `image: _images/<slug>.<ext>` line in frontmatter, and `![[_images/<slug>.<ext>|500]]` as the very first body line (one blank line above and below it, before any `##` heading).
   - See `docs/superpowers/specs/2026-04-25-recipe-images-design.md` §3.3 for the exact procedure and `docs/superpowers/plans/2026-04-25-recipe-images-implementation.md` for the bash commands.
```

- [ ] **Step 3: Add image-failure handling to the summary table example**

In the existing summary table block in step 8 (was step 7), add an extra row that shows what image-failure looks like:

```
   ✓ recipes-inbox/Recipe With Image.md
     → recipes/recipe-with-image.md (+ image)
   ⚠ recipes-inbox/Recipe Without OG Image.md
     → recipes/recipe-without-og.md (no image — source page had no OG tag)
```

- [ ] **Step 4: Add a "Don't fabricate images" line to "What NOT to do"**

Append to the bulleted list under "What NOT to do":

```markdown
- Don't generate or fabricate images. If neither OG nor the inbox URL produces a usable image, the recipe is written without one — silent degradation, no placeholder.
```

- [ ] **Step 5: Verify the doc still reads cleanly**

Run: `head -100 "tools/process-inbox.md"` and skim. Renumbering must be correct (no duplicate numbers, no gaps). The new step 4 must precede the old step 4 (Nutrition handling).

- [ ] **Step 6: Commit**

```bash
git add tools/process-inbox.md
git commit -m "Document image-fetch step in inbox processor

Adds a new step 4 (Image handling) to the per-file task list, plus
matching summary-table and What-NOT-to-do entries. Recipe normalization
now downloads the source page's OG image into recipes/_images/<slug>.<ext>
when available."
```

---

## Task 2: Update the original design spec

**Files:**
- Modify: `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` (§5 schema, §5.4 image policy)

- [ ] **Step 1: Update §5 frontmatter schema example to include `image:`**

Find the YAML block in §5 (around line 130–160) that shows the recipe frontmatter schema. After the `source:` field, add an `image:` field with a comment:

```yaml
image: _images/<slug>.<ext>   # Optional: relative to recipes/, omit if no image. See 2026-04-25-recipe-images-design.md.
```

- [ ] **Step 2: Replace §5.4 ("Image policy") body with a superseded notice**

Find the §5.4 paragraph (currently: *"No image files in the vault. The `source` URL field is the canonical image reference — open the source link to see the original. Revisit if needed in Phase 2."*) and replace its body with:

```markdown
### 5.4 Image policy

**Superseded 2026-04-25** — see `docs/superpowers/specs/2026-04-25-recipe-images-design.md`.

Each recipe now has a hero image stored in-vault at `recipes/_images/<slug>.<ext>`. Images are downloaded from the source page's OG image at inbox-processing time (with the inbox file's embedded image URL as a fallback). The `source:` URL remains the canonical link to the publisher; the local image is a cached snapshot for offline viewing in Obsidian.
```

Don't touch the heading text or §5.4's position — only the paragraph.

- [ ] **Step 3: Verify both edits**

Run: `grep -n "image:" "docs/superpowers/specs/2026-04-24-recipe-vault-design.md"` — expect at least one new match in the §5 frontmatter block.

Run: `grep -n "Superseded 2026-04-25" "docs/superpowers/specs/2026-04-24-recipe-vault-design.md"` — expect one match in §5.4.

- [ ] **Step 4: Commit**

```bash
git add docs/superpowers/specs/2026-04-24-recipe-vault-design.md
git commit -m "Update vault spec for image policy reversal

§5 schema gains optional image: field. §5.4 marked superseded by
2026-04-25-recipe-images-design.md."
```

---

## Task 3: End-to-end test with the inbox file

This validates the new processor instructions against a real inbox file. The file currently sitting at `recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md` is the test case — its `source:` URL points to a Punchfork wrapper around a halfbakedharvest-style food blog.

**Files:**
- Read: `recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md`
- Create: `recipes/healthy-mexican-street-corn-chicken-skillet.md`
- Create: `recipes/_images/healthy-mexican-street-corn-chicken-skillet.<ext>`
- Delete: `recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md`

- [ ] **Step 1: Read the inbox file and capture source URL + slug**

```bash
INBOX="recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md"
SOURCE_URL=$(grep -E '^source:' "$INBOX" | head -1 | sed -E 's/^source: *"?([^"]+)"?/\1/')
SLUG="healthy-mexican-street-corn-chicken-skillet"
echo "SOURCE: $SOURCE_URL"
echo "SLUG: $SLUG"
```

Expected: `SOURCE: https://www.punchfork.com/recipe/Healthy-Mexican-Street-Corn-Chicken-Skillet-Food-Meanderings`

- [ ] **Step 2: Resolve the publisher's actual URL**

Punchfork is an aggregator — the real recipe is on `foodmeanderings.com`. The inbox file body contains the publisher link. Extract it:

```bash
PUBLISHER_URL=$(grep -oE 'https?://[^[:space:])]+foodmeanderings\.com[^[:space:])]*' "$INBOX" | head -1)
echo "PUBLISHER: $PUBLISHER_URL"
```

If non-empty, use the publisher URL for OG extraction (better quality image). If empty, fall back to using the Punchfork URL.

```bash
EFFECTIVE_URL="${PUBLISHER_URL:-$SOURCE_URL}"
echo "Will fetch OG image from: $EFFECTIVE_URL"
```

- [ ] **Step 3: Run the reusable image-fetch procedure**

Execute the procedure from the "Reusable procedure" section above, with `SOURCE_URL=$EFFECTIVE_URL` and `INBOX_BODY=$(cat "$INBOX")`.

Expected: `recipes/_images/healthy-mexican-street-corn-chicken-skillet.jpg` (or `.webp`/`.png`) exists and is >1 KB.

- [ ] **Step 4: Re-fetch the publisher page and extract recipe content**

Use WebFetch (or curl) on `$EFFECTIVE_URL` to get the full recipe ingredients, directions, servings, time, etc. The Punchfork clip in the inbox file has empty ingredient lines, so a re-fetch is required.

- [ ] **Step 5: Write the normalized recipe file**

Write `recipes/healthy-mexican-street-corn-chicken-skillet.md` following the schema in `docs/superpowers/specs/2026-04-24-recipe-vault-design.md` §5 and the image conventions in `2026-04-25-recipe-images-design.md` §3.4. The exact structure:

```markdown
---
title: Healthy Mexican Street Corn Chicken Skillet
source: <publisher URL or punchfork URL>
image: _images/healthy-mexican-street-corn-chicken-skillet.<ext>
servings: <from source>
cuisine: mexican
category: dinner
meal_type: entree
prep_time_min: <from source>
cook_time_min: <from source>
total_time_min: 30
difficulty: easy
tags: [...]
diet_tags: [gluten-free, high-protein]
key_ingredients: [chicken, corn, ...]
nutrition:
  calories: 330
  protein_g: 29
  total_fat_g: 11
  carbs_g: 33
  cholesterol_mg:
  saturated_fat_g:
  sodium_mg:
  sugar_g:
  fiber_g:
nutrition_estimated: false
rating:
last_cooked:
times_cooked: 0
added: 2026-04-25
---

![[_images/healthy-mexican-street-corn-chicken-skillet.<ext>|500]]

## Ingredients

<from source>

## Directions

<from source>

## Notes

- **Heart-health tweaks for Doug:** <relevant tweaks>
- Imported via Web Clipper inbox flow on 2026-04-25.
```

If the image fetch in step 3 failed, OMIT both the `image:` frontmatter line and the `![[...]]` embed line. Otherwise include both, with the matching extension.

- [ ] **Step 6: Delete the inbox file**

```bash
rm "recipes-inbox/Healthy Mexican Street Corn Chicken Skillet.md"
```

- [ ] **Step 7: Visual verification in Obsidian**

Open `recipes/healthy-mexican-street-corn-chicken-skillet.md` in Obsidian. In reading view, the image must be the first rendered element, sized around 500px wide on desktop. Take a screenshot and report what you saw.

If the image doesn't render: check that the file path in the wikilink matches the actual file in `_images/` (extension matters), and that the wikilink syntax uses `[[` not `[`. Both are common mistakes.

- [ ] **Step 8: Commit**

```bash
git add recipes-inbox recipes/healthy-mexican-street-corn-chicken-skillet.md "recipes/_images/healthy-mexican-street-corn-chicken-skillet.*"
git commit -m "Process inbox: healthy-mexican-street-corn-chicken-skillet

End-to-end validation of the new image-fetch flow. Image downloaded
from source page's OG tag."
```

---

## Task 4: Backfill images for the 12 existing recipes

The 12 recipes in `recipes/` predate the image policy. Each has a valid `source:` URL. Run the image-fetch procedure for each, edit the recipe file to add the `image:` frontmatter line and the embed line, and commit as a single batch.

**Files:**
- Modify: all 12 recipes in `recipes/*.md` (where image fetch succeeds)
- Create: `recipes/_images/<slug>.<ext>` for each successful fetch

**The 12 recipes and their source URLs** (already verified, all reachable from frontmatter):

| Slug | Source |
|---|---|
| baja-fish-tacos-chipotle-mango-salsa | halfbakedharvest.com |
| cheddar-chicken-chili | halfbakedharvest.com |
| chicken-tzatziki-avocado-salad | halfbakedharvest.com |
| lemony-pesto-chicken-noodle-soup | halfbakedharvest.com |
| protein-bowls-viral-tiktok | spoonforkbacon.com |
| salmon-club-salad | halfbakedharvest.com |
| sheet-pan-chicken-fajitas | halfbakedharvest.com |
| slow-roasted-apple-cider-pork-shoulder | thenoshery.com |
| sticky-asian-bbq-baked-chicken-wings | halfbakedharvest.com |
| streak-o-lean | lanascooking.com |
| thai-basil-beef-noodles | halfbakedharvest.com |
| turkish-eggs-with-chile-butter | halfbakedharvest.com |

- [ ] **Step 1: Loop over recipes and download images**

```bash
cd "/Users/dougmarch/Library/Mobile Documents/iCloud~md~obsidian/Documents/Recipes"
mkdir -p recipes/_images

declare -A RESULTS  # slug -> ext or "skip"

for f in recipes/*.md; do
  SLUG=$(basename "$f" .md)
  SOURCE_URL=$(grep -E '^source:' "$f" | head -1 | sed -E 's/^source: *"?([^"]+)"?/\1/' | tr -d ' ')
  echo "=== $SLUG ==="
  echo "Source: $SOURCE_URL"

  # OG extraction
  OG_URL=$(curl -sL --max-time 15 -A "Mozilla/5.0" "$SOURCE_URL" \
    | grep -oE '<meta[^>]+property=["'"'"']og:image["'"'"'][^>]*>' \
    | head -1 \
    | grep -oE 'content=["'"'"'][^"'"'"']+["'"'"']' \
    | sed -E 's/content=["'"'"']//; s/["'"'"']$//')

  if [ -z "$OG_URL" ]; then
    echo "  no OG image, skipping"
    RESULTS[$SLUG]="skip"
    continue
  fi
  echo "  OG: $OG_URL"

  # Extension
  EXT=$(echo "$OG_URL" | sed -E 's/\?.*$//' | grep -oE '\.[a-zA-Z0-9]+$' | tail -c +2 | tr A-Z a-z)
  case "$EXT" in jpg|jpeg|png|webp|gif) ;; *) EXT="" ;; esac
  if [ -z "$EXT" ]; then
    CT=$(curl -sLI --max-time 10 -A "Mozilla/5.0" -H "Referer: $SOURCE_URL" "$OG_URL" \
      | grep -i '^content-type:' | tail -1 | tr -d '\r' | awk '{print $2}')
    case "$CT" in
      image/jpeg) EXT="jpg" ;;
      image/png)  EXT="png" ;;
      image/webp) EXT="webp" ;;
      image/gif)  EXT="gif" ;;
      *) EXT="jpg" ;;
    esac
  fi

  # Normalize jpeg -> jpg
  [ "$EXT" = "jpeg" ] && EXT="jpg"

  DEST="recipes/_images/${SLUG}.${EXT}"
  curl -sL --max-time 30 -A "Mozilla/5.0" -H "Referer: $SOURCE_URL" "$OG_URL" -o "$DEST"
  SIZE=$(stat -f%z "$DEST" 2>/dev/null || stat -c%s "$DEST" 2>/dev/null)
  if [ "${SIZE:-0}" -lt 1024 ]; then
    echo "  too small ($SIZE bytes), discarding"
    rm -f "$DEST"
    RESULTS[$SLUG]="skip"
  else
    echo "  saved $DEST ($SIZE bytes)"
    RESULTS[$SLUG]="$EXT"
  fi
done

# Print summary
echo
echo "=== SUMMARY ==="
for slug in "${!RESULTS[@]}"; do
  echo "  $slug: ${RESULTS[$slug]}"
done
```

Expected: most or all recipes return a valid extension; any that returned "skip" should be noted.

- [ ] **Step 2: Edit each recipe file to add `image:` frontmatter and the embed line**

For each recipe where step 1 produced a file in `_images/`, edit the recipe markdown:

1. In the frontmatter (between the two `---` lines), insert `image: _images/<slug>.<ext>` directly after the `source:` line.
2. Immediately after the closing `---` of frontmatter, on a new line, insert a blank line, then `![[_images/<slug>.<ext>|500]]`, then a blank line. This must come BEFORE any `##` heading.

Example diff (for `recipes/sheet-pan-chicken-fajitas.md`):

```diff
 ---
 title: Easy Sheet Pan Chicken Fajitas
 source: https://www.halfbakedharvest.com/sheet-pan-chicken-fajitas/
+image: _images/sheet-pan-chicken-fajitas.jpg
 servings: 6
 cuisine: mexican
 ...
 added: 2026-04-24
 ---
+
+![[_images/sheet-pan-chicken-fajitas.jpg|500]]

 ## Ingredients
```

For recipes with `RESULTS[$slug] == "skip"` from Task 4 step 1, do NOT edit — leave as-is.

Use the Edit tool, one recipe at a time. Verify each edit by re-reading the top 25 lines of the file and confirming the embed line appears between the `---` and `## Ingredients`.

- [ ] **Step 3: Spot-check three recipes by opening them in Obsidian**

Open three recipes in Obsidian (any three with images, e.g., `salmon-club-salad`, `sheet-pan-chicken-fajitas`, `turkish-eggs-with-chile-butter`). For each:
- The image must be the first rendered element in reading view
- It must be sized around 500px wide on desktop
- The `## Ingredients` heading must follow it

If any fails, check the wikilink syntax in the recipe and the file extension in `_images/`.

- [ ] **Step 4: Commit**

```bash
git add recipes/_images recipes/*.md
git commit -m "Backfill images for existing recipes

Downloaded OG images for the 12 recipes that predate the image policy.
Added image: frontmatter and the wikilink embed at the top of each
recipe's body. Recipes whose source pages had no OG tag were left
unchanged (none in this batch / N skipped — fill in from summary)."
```

If any recipe was skipped, replace the `(none in this batch / N skipped...)` text with the actual count and recipe slugs.

---

## Task 5: Open PR and merge

**Files:** none (git operations only)

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feature/recipe-images
```

- [ ] **Step 2: Open the PR**

```bash
gh pr create --title "Add recipe images: design, processor update, and backfill" --body "$(cat <<'EOF'
## Summary

- Adds a hero image to every recipe in the vault, downloaded from the source page's OG image at inbox-processing time
- Updates `tools/process-inbox.md` with the new image-fetch step (OG-preferred, inbox-URL fallback, silent-skip on failure)
- Backfills the 12 existing recipes with images where the source page provided one
- Reverses the original "no images in vault" decision (2026-04-24 spec §5.4)

## Spec & Plan

- Design: `docs/superpowers/specs/2026-04-25-recipe-images-design.md`
- Plan: `docs/superpowers/plans/2026-04-25-recipe-images-implementation.md`

## Test plan

- [ ] Open `recipes/healthy-mexican-street-corn-chicken-skillet.md` in Obsidian — image is the first rendered element
- [ ] Open three backfilled recipes in Obsidian — images render correctly at ~500px wide
- [ ] `## Ingredients`/`## Directions`/`## Notes` parsing contract preserved (Sunday agent unaffected)
- [ ] No `image:` field on recipes whose source page had no OG image (silent degradation)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Merge if everything looks clean**

User has authorized merge ("merge if you think it is good to go"). Before merging, sanity-check:
- All commits on the branch are spec/plan/processor/recipes — nothing unrelated
- No accidentally-committed binaries beyond the `recipes/_images/` files
- Recipes that were "skipped" are correctly left without `image:` field (not stuck halfway with embed but no file)

If clean:

```bash
gh pr merge --squash --delete-branch
```

If anything is off, do NOT merge — report findings instead.

- [ ] **Step 4: Pull merged main**

```bash
git checkout main && git pull
```

---

## Self-review notes

Before handing this plan to the executor:

- **Spec coverage:** §1 Purpose ✓ (Tasks 3 & 4 produce visible images). §3 Architecture ✓ (Task 1 documents §3.1–3.5 in process-inbox.md; reusable procedure encodes §3.3). §4 Frontmatter schema ✓ (Task 2). §5 Inbox processor change ✓ (Task 1). §6 Backfill ✓ (Task 4). §8 Failure modes ✓ (procedure handles each branch). §11 Acceptance criteria 1–6 all map: AC1 = Task 1; AC2 = Task 2; AC3 = Task 3; AC4 = Task 4; AC5 = Task 4 step 3; AC6 = preserved by construction (no `## Ingredients` reordering anywhere).

- **Placeholder scan:** No "TODO", "TBD", "fill in details". The only template in Task 3 step 5 is the recipe body, which is necessarily filled in from the publisher page at execution time — that's not a planning gap, that's the work.

- **Type/name consistency:** `_images` everywhere (not `images/`, not `attachments/`). `recipes/_images/<slug>.<ext>` everywhere. Wikilink form `![[_images/<slug>.<ext>|500]]` everywhere. `image:` frontmatter field everywhere (not `image_path:` or `picture:`).
