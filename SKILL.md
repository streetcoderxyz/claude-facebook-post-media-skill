---
name: facebook-post-from-clone
description: Publish a clone batch as ONE Facebook post (album with multiple images) to a single Facebook group or your own timeline. Reads all post-N/new-image-logo.jpg files from a facebook-clone-post batch and uploads them as an album under a single album caption. Drives the composer via OpenClaw browser with a logged-in Brave profile. Triggers on "post the album to <group>", "đăng album lên timeline", "publish batch X as one post".
allowed-tools: Bash(openclaw browser:*), Bash(sleep:*), Bash(ps:*), Bash(kill:*), Bash(cat:*), Bash(ls:*), Bash(cp:*), Bash(mkdir:*), Bash(python3:*), Bash(sips:*)
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

The composer trigger label depends on the target surface:

| Surface | Trigger label |
|---|---|
| Own timeline | `<Name> ơi, bạn đang nghĩ gì thế?` / `What's on your mind, <Name>?` |
| Page | `Bạn đang nghĩ gì?` / `What's on your mind?` |
| Group | `Bạn viết gì đi...` / `Write something...` |

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Write something|Viết gì đi|Viết gì đó|What's on your mind|Bạn đang nghĩ gì|nghĩ gì thế" | head -3
OPENCLAW_TIMEOUT=60000 openclaw browser click <trigger_ref>
sleep 3
```

A modal composer opens with an empty textbox.

> **Privacy note for own-timeline:** FB defaults the audience to "Friends" (`Bạn bè`). If you want public, click the privacy button (`Chỉnh sửa quyền riêng tư...`) and switch BEFORE filling content — switching after often resets the composer.

### Phase 3 — Attach all images in one shot

Click the "Ảnh/video" / "Photo/video" toggle inside the composer to mount the file input (`<input type=file multiple>`). **Do NOT click an "Add photos" / "Choose file" button afterwards — that opens the native OS file picker dialog, which OpenClaw can't drive.** Instead, upload directly to the input element via `--element`:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Photo/video|Ảnh/video" | head -3
OPENCLAW_TIMEOUT=60000 openclaw browser click <photo_video_ref>
sleep 2

# Programmatic upload — no popup. Target the multi-file input inside the dialog.
OPENCLAW_TIMEOUT=120000 openclaw browser upload --element '[role=dialog] input[type=file][multiple]' $STAGED_LIST
sleep 12  # FB needs time to process thumbnails for an album
```

> **Why `--element` and not `--ref`:** `--ref <button>` arms a file chooser triggered by clicking that button — that opens the native OS picker, which the agent cannot interact with. `--element <selector>` sets `input.files` directly on the hidden input via DOM, then dispatches `change` — no popup, no user interaction needed.

**Verify all thumbnails rendered.** FB's composer accepts only 5 images via the first programmatic upload; the rest are silently dropped. So after sleep, count *unique* attached photos and add the remainder via the Edit-all flow if short:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var dlgs = document.querySelectorAll("[role=dialog]"); var t = Array.from(dlgs).find(function(d){return d.getAttribute("aria-label")==="Tạo bài viết" && d.offsetWidth>0;}); var blobs = t ? Array.from(t.querySelectorAll("img")).filter(function(i){return i.src.startsWith("blob:");}) : []; var u = {}; blobs.forEach(function(i){u[i.src]=true;}); return JSON.stringify({unique_blobs: Object.keys(u).length}); }'
```

If `unique_blobs < N`:
1. Click "Chỉnh sửa tất cả" / "Edit all" button in the composer (re-snapshot to find its ref).
2. In the editor view, click "Thêm ảnh/video" / "Add photo/video" button — **but as `--element`**, not as a UI click. Re-snapshot to get the file input ref/selector for the editor's input.
3. Upload the missing files via `--element` on that input.
4. Count again. The editor shows ~2 blob views per photo (main + thumbnail-strip) so the unique count there is `≈ N × 2` — divide by 2 to compare against `N`.
5. Click "Xong" / "Done" to exit editor mode.

> ⚠️ Closing the editor with "Xong" can trigger a "Lưu bài viết này làm bản nháp?" (save as draft?) confirmation if FB thinks the user is leaving. Click "Đóng" (Close) on THAT dialog — not "Xóa bản nháp" (which deletes everything) and not "Lưu làm bản nháp" (which buries it in drafts). After closing, re-snapshot the editor view and click Xong again — the draft prompt should not re-appear.

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

Then **verify the caption stuck** by reading the Lexical contenteditable's text content. Don't rely on the snapshot showing it — the editor sometimes mounts outside the dialog DOM tree:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var tbs = document.querySelectorAll("[contenteditable=true]"); var best = Array.from(tbs).reduce(function(a,b){return b.innerText.length > a.innerText.length ? b : a;}, tbs[0] || {innerText:""}); return JSON.stringify({len: best.innerText.length, preview: best.innerText.slice(0,80)}); }'
```

If `len === 0` after fill, either the ref was stale OR your `evaluate` was scoped too narrowly — Lexical's actual editor sometimes mounts outside `[role=dialog]`. Try a global `document.querySelectorAll("[contenteditable=true]")` and read the longest one; if that has the caption, you're done. Otherwise re-snapshot the textbox and re-fill.

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
| Only 5 of N images attached after `upload --element` | FB silently drops uploads past 5 on the first programmatic batch. Click "Chỉnh sửa tất cả" / "Edit all" to enter the editor view, find the editor's `input[type=file]` (its own hidden input, distinct from the composer's), then `upload --element` the missing images to it. Then click "Xong" / "Done". |
| "Lưu bài viết này làm bản nháp?" prompt appears after clicking "Xong" | FB thinks you're closing the composer. Click "Đóng" (Close) in THAT prompt — never "Xóa bản nháp" (deletes work) or "Lưu làm bản nháp" (sends to drafts). After closing, re-snapshot the editor and re-click Xong. |
| Caption fill reports `filled 1 field(s)` but text doesn't appear | The ref pointed at a sibling/empty textbox. Lexical's actual editor is sometimes outside the dialog DOM — verify via `document.querySelectorAll("[contenteditable=true]")` globally and read the largest one. Re-snapshot the textbox ref and re-fill. |
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
