---
name: facebook-post-from-clone
description: Publish a clone batch as ONE Facebook post (album with multiple images) to a single Facebook group or your own timeline. Reads all post-N/new-image-logo.jpg files from a facebook-clone-post batch and uploads them as an album under a single album caption. Drives the composer via OpenClaw browser with a logged-in Brave profile. Triggers on "post the album to <group>", "đăng album lên timeline", "publish batch X as one post".
---

# Publish Clone Batch as One Album Post

Takes an entire clone-batch directory (the output of `facebook-clone-post`, e.g. `/Users/binhquach/Workplace/fb-clone/<batch>/`) and creates ONE Facebook post containing all N images as an album, under a single album-level caption. Designed for the medical-infographic batches where ten ailment infographics belong together as a single themed post.

## When to use this skill

- After `/facebook-clone-post` produced N infographics (e.g. 10), you want them under ONE post with one caption, not N separate posts.
- Single-image albums (N=1) work too — pass a batch with just `post-1/`.

Do NOT use for:
- N separate posts, one per image → loop the user's manual FB upload, or use `facebook-post-to-groups` per image.
- Sharing an existing FB post → `facebook-share-to-groups`.
- Scheduling for later via Business Suite → `facebook-schedule-business-suite`.
- Posting source images instead of cloned images → adjust `--image-name` (default `new-image-logo.jpg`).

## Prerequisites

1. `openclaw browser status` → `running: true`. If not: `openclaw browser start && sleep 4`.
2. Brave profile already logged in to Facebook. Login walls → **stop and notify**, never automate login.
3. The batch directory must contain `post-1/`, `post-2/`, … each with the chosen image file.
4. The batch directory must contain `album-caption.md` (the single album caption). If missing → STOP and ask the user to create it (the skill never auto-generates marketing copy).
5. `OPENCLAW_TIMEOUT=60000` prefix on every `openclaw browser` command (120000 for upload/fill).

---

## Configurable Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| **BATCH_DIR** | Yes | — | Path to a facebook-clone-post batch directory (contains `post-1/`, `post-2/`, …) |
| **TARGET** | Yes | — | Facebook group URL (`https://www.facebook.com/groups/<id>`) OR `timeline` OR a full profile URL |
| **CAPTION_FILE** | No | `<BATCH_DIR>/album-caption.md` | Single caption used for the album |
| **IMAGE_NAME** | No | `new-image-logo.jpg` | Filename to pick from each `post-*/` (falls back to `new-image.jpg`) |
| **SLOT_ORDER** | No | numeric `1..N` | Override post order: comma-separated slot numbers, e.g. `1,3,5,2,4`. Useful if you want a specific narrative sequence |
| **MAX_IMAGES** | No | `10` | Skip slots beyond this. Facebook accepts more, but album UX degrades past ~10. |
| **DRY_RUN** | No | `false` | If `true`, fills composer + attaches all images + verifies post button enabled, but does NOT click Post. Useful for previewing. |

---

## Workflow

### Phase 0 — Setup + validate

1. Verify BATCH_DIR + caption + images:

   ```bash
   test -d "<BATCH_DIR>" || { echo "ERR: BATCH_DIR not a directory"; exit 1; }
   CAPTION="${CAPTION_FILE:-<BATCH_DIR>/album-caption.md}"
   test -f "$CAPTION" || { echo "ERR: $CAPTION missing — please create it"; exit 1; }
   IMG_NAME="${IMAGE_NAME:-new-image-logo.jpg}"
   # Discover slots
   SLOTS=$(ls -d <BATCH_DIR>/post-* 2>/dev/null | sed 's|.*post-||' | sort -n)
   echo "Slots found: $SLOTS"
   ```

2. **Build the image list** in `SLOT_ORDER` (default: numeric ascending), capped at `MAX_IMAGES`:

   ```bash
   # For each slot, pick the image (logoed preferred, falls back)
   for s in $ORDERED_SLOTS; do
     d=<BATCH_DIR>/post-$s
     if [ -f "$d/$IMG_NAME" ]; then echo "$d/$IMG_NAME"; \
     elif [ -f "$d/new-image.jpg" ]; then echo "$d/new-image.jpg"; \
     else echo "WARN: post-$s has no image, skipping" >&2; fi
   done | head -$MAX_IMAGES > /tmp/album-images.txt
   ```

   STOP if the list is empty or only 1 image where the user expected an album — surface a count to the user before proceeding.

3. **Classify TARGET** (group / timeline / profile) — same as the single-post case.

4. Verify browser:

   ```bash
   OPENCLAW_TIMEOUT=60000 openclaw browser status
   ```

5. **Stage all images** to OpenClaw's uploads dir with unique per-batch filenames:

   ```bash
   BATCH_ID=$(basename "<BATCH_DIR>")
   STAGED_DIR=/tmp/openclaw/uploads
   mkdir -p "$STAGED_DIR"
   STAGED_LIST=""
   while read src; do
     dest="$STAGED_DIR/album-${BATCH_ID}-$(basename $(dirname $src))-$(date +%s%N | tail -c 8).jpg"
     cp "$src" "$dest"
     STAGED_LIST="$STAGED_LIST $dest"
   done < /tmp/album-images.txt
   echo "Staged: $STAGED_LIST"
   ```

   Per-image unique filenames matter — FB's composer occasionally dedups identical filenames within a session.

### Phase 1 — Navigate to target

For group:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser open "<TARGET>" && sleep 5
```

For timeline:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser open "https://www.facebook.com/me" && sleep 5
```

Verify not a login wall:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => JSON.stringify({title: document.title.slice(0,80), url: location.href})'
```

If title contains "Log in to Facebook" → STOP.

### Phase 2 — Open the composer

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Write something|Viết gì đó|What's on your mind|Bạn đang nghĩ gì" | head -3
OPENCLAW_TIMEOUT=60000 openclaw browser click <trigger_ref>
sleep 3
```

A modal composer opens with an empty textbox.

### Phase 3 — Attach all images in one shot

Open the inline Photo/Video picker:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Photo/video|Ảnh/video" | head -3
OPENCLAW_TIMEOUT=60000 openclaw browser click <photo_video_ref>
sleep 2
```

Find the "Add photos" / "Chọn file" trigger and pass ALL staged paths to `upload`. OpenClaw's `upload` accepts variadic paths (`upload [options] <paths...>`) so a single call sets every file at once on the input:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Add photos|Add photo|Thêm ảnh|Choose file|Chọn file" | head -5
# Pass every staged path on one line:
OPENCLAW_TIMEOUT=120000 openclaw browser upload --ref <add_photos_ref> $STAGED_LIST
sleep 10  # FB needs time to process all thumbnails
```

**Verify all thumbnails rendered.** Count visible image previews in the composer:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var dlg = document.querySelector("[role=dialog]"); var imgs = dlg ? Array.from(dlg.querySelectorAll("img")).filter(function(i){ return /blob:|scontent|fbcdn/.test(i.src) && i.width > 60 && i.width < 600; }) : []; return JSON.stringify({thumb_count: imgs.length}); }'
```

Expected `thumb_count === N` (number of images you uploaded). If it's lower:
- Some uploads still processing → wait 5s and recount.
- Persistently lower → some files failed. Identify which by comparing thumbnail captions/order, then use the inline "+" / "Add more photos" button to retry the missing ones individually.

> ⚠️ FB silently caps album size at certain limits (varies by surface — historically ~80 photos but UI degrades past ~10). If `thumb_count` plateaus below the requested N, lower `MAX_IMAGES` and re-run.

### Phase 4 — Fill the caption

Re-snapshot to find the contenteditable textbox ref (it's the modal's main composer field, not any per-image alt-text field):

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Create a public post|Tạo bài viết|What's on your mind|Bạn đang nghĩ gì|textbox" | head -5
```

Fill via `fill --fields` (never `type`):

```bash
P=$(cat "$CAPTION") && PJSON=$(printf '%s' "$P" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))') \
  && OPENCLAW_TIMEOUT=120000 openclaw browser fill --fields "[{\"ref\":\"<textbox_ref>\",\"value\":$PJSON}]"
sleep 3
```

### Phase 5 — Submit + verify

Re-snapshot the Post button:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "button \"Post\"|button \"Đăng\"" | head -3
```

If `[disabled]`, the album upload isn't finished. **Wait longer than single-image** — 10 images can take 30 s+ to process FB-side. Loop with 5s sleeps, up to 6 retries. If still disabled → flag INCOMPLETE.

If `DRY_RUN=true`: stop here. Report `READY_BUT_NOT_POSTED` with `thumb_count` and caption preview.

Otherwise click and verify:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser click <post_ref>
sleep 10  # album posts take longer to commit than single-image
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var modal = document.querySelector("[role=dialog][aria-label*=\"Create\" i], [role=dialog][aria-label*=\"Tạo\" i]"); return JSON.stringify({modal_still_open: !!modal, url: location.href}); }'
```

If `modal_still_open: true` after 10 s, wait another 10 s. If still open, FB is processing or showing a confirmation — STOP and notify user; do NOT click anywhere automatically.

### Phase 6 — Summary

```
Posted album!
  BATCH: <BATCH_ID>
  TARGET: <TARGET>
  IMAGES: <N> attached (in order: post-1, post-2, …, post-N)
  CAPTION: <first 80 chars of caption>…
```

For DRY_RUN:
```
DRY RUN — composer populated with <N> images + caption, NOT posted.
  Click Post manually in the browser, or re-run without DRY_RUN.
```

---

## Rules

1. **`OPENCLAW_TIMEOUT=60000`** prefix on every browser command (120000 for upload/fill of large content).
2. **Use `fill --fields`, never `type`, for Vietnamese captions.** Diacritics mangle otherwise.
3. **Re-snapshot after every mutation** (click, upload, fill). Refs change.
4. **Stage every image with a unique filename** under `/tmp/openclaw/uploads/`. FB sometimes dedups identical filenames within a session, dropping silently.
5. **One album per skill invocation.** Don't loop this to target multiple groups — use `facebook-post-to-groups` for that.
6. **Never automate login.** Login wall → STOP, notify user.
7. **Honor DRY_RUN.** First-time use against a new group → always DRY_RUN first.
8. **Default to `new-image-logo.jpg`.** The logoed image is the intended publish artifact; un-logoed is a fallback.
9. **Don't auto-generate the album caption.** The user authors `album-caption.md`. If missing, STOP — never invent marketing copy.
10. **Verify thumb_count matches expected N before clicking Post.** A partially-uploaded album posts the wrong subset and there's no undo without deleting + reposting.
11. **Wait longer for album upload + commit than single image.** ~3 s/image for upload processing, ~10 s for final commit after clicking Post.
12. **Verify modal closed after submit.** A still-open dialog post-click is the surest sign FB didn't accept the post.

---

## Error recovery

| Symptom | Action |
|---|---|
| FB redirects to /login | STOP. User must log in manually in Brave. |
| `album-caption.md` missing | STOP. Tell the user to create it at `<BATCH_DIR>/album-caption.md`. Don't auto-generate. |
| `thumb_count` < expected N after upload | Some files failed. Click "+" / "Add more" in the composer, re-arm upload with the missing files (compare `STAGED_LIST` against the rendered thumbs). Retry once. If still short → STOP and report which slots are missing. |
| Post button stays `[disabled]` for > 60 s | Likely a stalled upload or rate limit. Re-snapshot once. If still disabled → flag INCOMPLETE and don't retry; FB may have shadow-rejected the album. |
| Modal closes mid-upload | A click cascaded too fast or you bumped focus. Re-open composer, click Photo/video, re-arm upload. Use `sleep 3` between trigger clicks. |
| Variadic `upload` only attaches 1 file | OpenClaw build doesn't support multi-path upload. Fall back: click "Add more photos" once per image, `--ref` the "+" button each time, `upload` one path at a time. Use a separate snapshot for each iteration since the "+" ref moves. |
| Album order in the post differs from `SLOT_ORDER` | FB sometimes reorders by EXIF timestamp. If order matters, rename staged files with a numeric prefix (`01-<slot>.jpg`, `02-<slot>.jpg`, …) before uploading — FB respects filename sort for tied timestamps. |
| Rate limit dialog | STOP. Notify user. Wait ≥ 15 minutes before retrying. |
| Album posted but image quality looks reduced in feed | FB compresses albums more aggressively than single images. Compare the feed image to `image-1-full.png` source — if loss is severe, post as separate single-image posts instead. |

---

## What NOT to do

- **Don't post a partial album.** If any image failed to attach, STOP and let the user decide — never publish a subset silently.
- **Don't auto-retry after a failed post.** May have published silently; reposting risks a duplicate flagged by FB.
- **Don't `type` Vietnamese text.** Always `fill --fields`.
- **Don't paste the caption via clipboard.** Diacritic risk.
- **Don't run `openclaw browser stop` mid-flow.** It logs out FB.
- **Don't modify `album-caption.md` from inside this skill.** If a tweak is needed, edit it externally and re-run.
- **Don't loop this skill to target multiple groups.** For batch-to-groups, hand off to `facebook-post-to-groups` (it can iterate group URLs from a Notion list).
- **Don't reorder slots silently.** If `SLOT_ORDER` overrides the default numeric order, surface that in the summary.

---

## Future hooks (not implemented yet)

- `--auto-caption` mode that synthesizes the album caption by feeding `_topics.json` through ChatGPT (currently the user authors `album-caption.md`).
- Multi-target mode: after a successful album post, navigate to additional group URLs and clone-paste the same composer state.
- Permalink capture: after publish, grab the new post's URL and write it back to `<BATCH_DIR>/album-published.json` for tracking.
- Hashtag injection: read `<BATCH_DIR>/hashtags.txt` and append to the caption automatically.
