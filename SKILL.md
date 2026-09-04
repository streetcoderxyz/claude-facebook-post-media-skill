---
name: facebook-post-media
description: Publish media (one video OR an N-image album) from a clone-batch folder as ONE Facebook post to a single target (your timeline OR a Facebook group). Reads media files from a facebook-clone-post style batch (`post-N/<media>`) and drives the composer via OpenClaw browser with a logged-in Brave profile. Triggers on "post the album to <group>", "publish video to my timeline", "đăng video lên nhóm <name>", "publish batch as one post". For posting to MANY targets, hand off to facebook-post-to-groups instead.
allowed-tools: Bash(openclaw browser:*), Bash(sleep:*), Bash(ps:*), Bash(kill:*), Bash(cat:*), Bash(ls:*), Bash(cp:*), Bash(mkdir:*), Bash(python3:*), Bash(sips:*), Bash(test:*), Bash(date:*), Bash(basename:*), Bash(dirname:*), Bash(file:*)
---

# Publish Media from a Clone-Batch Folder

Takes a single piece of media (one video at `post-N/video/<file>.mp4`, OR a multi-image album at `post-N/<image>.jpg`) from a facebook-clone-post style batch directory and creates ONE Facebook post on a single target (your timeline OR a Facebook group). Designed for two flavors:

- **Album mode** — the original use case: a batch of N infographics (e.g. ten ailment cards) published as one album with one caption.
- **Video mode** — a single rendered video (e.g. `post-1/video/post-1.mp4` from the `video-prep` `post2video` pipeline) published as one post with one caption.

The composer mechanics are the same; only the file type and timing differ.

## When to use this skill

- After `/facebook-clone-post` produced N infographics and you want one album post with one caption.
- After `/video-prep post2video` produced one `post-1/video/post-1.mp4` and you want one video post.
- Single-image case (N=1 with one `post-1/new-image-logo.jpg`) — works too.

Do NOT use for:
- N separate posts, one per image → use `facebook-post-to-groups` per image.
- Posting the SAME content to MANY targets → use `facebook-post-to-groups` (it iterates group URLs from a Notion list). This skill is one target per invocation.
- Sharing an existing FB post → `facebook-share-to-groups`.
- Scheduling for later via Business Suite → `facebook-schedule-business-suite`.
- Posting source images instead of cloned images → set `MEDIA_NAME=source-image.jpg`.

## Prerequisites

1. `openclaw browser status` → `running: true`. If not: `openclaw browser start && sleep 4`.
2. Brave profile already logged in to Facebook. Login walls → **stop and notify**, never automate login.
3. The batch directory must contain `post-1/`, `post-2/`, … each with the chosen media file.
4. A caption file at one of:
   - `<BATCH_DIR>/album-caption.md` — used by default (album-style)
   - `<BATCH_DIR>/post-<slot>/video-caption.md` — used when MODE=video and the slot is specified
   - Override via `CAPTION_FILE=<path>` env var
   If missing → STOP and ask the user to create it (the skill never auto-generates marketing copy).
5. `OPENCLAW_TIMEOUT=60000` prefix on every `openclaw browser` command (120000 for upload/fill).

---

## Configurable Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| **BATCH_DIR** | Yes | — | Path to a facebook-clone-post (or video-prep post2video) batch directory containing `post-1/`, `post-2/`, … |
| **TARGET** | Yes | — | Facebook group URL (`https://www.facebook.com/groups/<id>`) OR `timeline` OR a full profile URL |
| **MODE** | No | auto | `album` \| `video` \| `auto`. `auto` picks `video` if `post-<SLOT>/video/*.mp4` exists, else `album`. |
| **SLOT** | No | `1` (video mode) / all (album mode) | In video mode, which slot's video to post. In album mode, ignored. |
| **CAPTION_FILE** | No | mode-specific (see above) | Caption used for the post. |
| **MEDIA_NAME** | No | album mode: `new-image-logo.jpg` (fallback `new-image.jpg`); video mode: first `*.mp4` in `post-<SLOT>/video/` | Filename to pick from each slot. |
| **SLOT_ORDER** | No | numeric `1..N` (album mode only) | Comma-separated slot order override, e.g. `1,3,5,2,4`. |
| **MAX_IMAGES** | No | `10` (album mode only) | Cap the album slot count. FB albums degrade past ~10. |
| **DRY_RUN** | No | `false` | Fill composer + attach media + verify Post button enabled, but DON'T click Post. Recommended for a first run against a new group. |

---

## Workflow

### Phase 0 — Setup + validate

1. Verify BATCH_DIR exists. Resolve MODE:

   ```bash
   test -d "<BATCH_DIR>" || { echo "ERR: BATCH_DIR not a directory"; exit 1; }
   SLOT="${SLOT:-1}"
   if [ "$MODE" = "auto" ] || [ -z "$MODE" ]; then
     if ls "<BATCH_DIR>/post-$SLOT/video/"*.mp4 2>/dev/null | head -1 >/dev/null; then
       MODE=video
     else
       MODE=album
     fi
   fi
   echo "Mode: $MODE"
   ```

2. Resolve caption:

   ```bash
   if [ -z "$CAPTION_FILE" ]; then
     if [ "$MODE" = "video" ]; then
       CAPTION_FILE=<BATCH_DIR>/post-$SLOT/video-caption.md
       [ -f "$CAPTION_FILE" ] || CAPTION_FILE=<BATCH_DIR>/album-caption.md
     else
       CAPTION_FILE=<BATCH_DIR>/album-caption.md
     fi
   fi
   test -f "$CAPTION_FILE" || { echo "ERR: $CAPTION_FILE missing — please create it"; exit 1; }
   ```

3. **Build the media list:**

   - **Album mode** — iterate slots in `SLOT_ORDER` (default numeric), pick `MEDIA_NAME` per slot (fallback `new-image.jpg`), cap at `MAX_IMAGES`.
   - **Video mode** — single file `<BATCH_DIR>/post-$SLOT/video/${MEDIA_NAME:-$(ls <BATCH_DIR>/post-$SLOT/video/*.mp4 | head -1)}`.

   STOP if the list is empty.

4. **Classify TARGET** (group / timeline / profile).

5. Verify browser:

   ```bash
   OPENCLAW_TIMEOUT=60000 openclaw browser status
   ```

6. **Stage media files** with unique per-batch filenames:

   ```bash
   BATCH_ID=$(basename "<BATCH_DIR>")
   STAGED_DIR=/tmp/openclaw/uploads
   mkdir -p "$STAGED_DIR"
   STAGED_LIST=""
   for src in $MEDIA_LIST; do
     EXT="${src##*.}"
     dest="$STAGED_DIR/post-${BATCH_ID}-$(basename $(dirname $src))-$(date +%s%N | tail -c 8).$EXT"
     cp "$src" "$dest"
     STAGED_LIST="$STAGED_LIST $dest"
   done
   echo "Staged: $STAGED_LIST"
   ```

   Per-file unique filenames matter — FB's composer occasionally dedups identical filenames within a session.

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
| Own timeline | `<Name> ơi, bạn đang nghĩ gì thế?` / `What's on your mind, <Name>?` / `Bạn đang nghĩ gì?` |
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

### Phase 3 — Attach media

Click the "Ảnh/video" / "Photo/video" toggle inside the composer to mount the file input (`<input type=file multiple>` accepts both images AND video). **Do NOT click an "Add photos" / "Choose file" button afterwards — that opens the native OS file picker dialog, which OpenClaw can't drive.** Instead, upload directly to the input element via `--element`:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Photo/video|Ảnh/video" | head -3
OPENCLAW_TIMEOUT=60000 openclaw browser click <photo_video_ref>
sleep 2

# Programmatic upload — no popup. Target the file input inside the dialog.
OPENCLAW_TIMEOUT=120000 openclaw browser upload --element '[role=dialog] input[type=file]' $STAGED_LIST
```

Wait for processing — video needs longer:

```bash
# Album: sleep ~3s per image (FB processes thumbnails)
# Video: sleep 10-15s for the first frame to render in the composer
sleep $([ "$MODE" = "video" ] && echo 12 || echo $((3 * N_IMAGES)))
```

> **Why `--element` and not `--ref`:** `--ref <button>` arms a file chooser triggered by clicking that button — that opens the native OS picker, which the agent cannot interact with. `--element <selector>` sets `input.files` directly on the hidden input via DOM, then dispatches `change` — no popup, no user interaction needed.

> **Note on groups vs timeline:** in some group composers, the upload temporarily opens an unlabeled secondary dialog (showing the file input dialog itself) alongside the `Tạo bài viết` composer. Don't panic — the file still binds correctly. Verify via a broader scan (see verification below).

**Verify the media attached.** Different checks for the two modes:

```bash
# Album mode — count unique blob: imgs across all visible dialogs
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var dlgs = Array.from(document.querySelectorAll("[role=dialog]")).filter(d => d.offsetWidth > 0); var imgs = []; dlgs.forEach(d => Array.from(d.querySelectorAll("img")).forEach(i => { if (i.src.startsWith("blob:")) imgs.push(i.src); })); var u = {}; imgs.forEach(s => u[s] = true); return JSON.stringify({unique_blobs: Object.keys(u).length}); }'

# Video mode — count <video> elements across all visible dialogs
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var dlgs = Array.from(document.querySelectorAll("[role=dialog]")).filter(d => d.offsetWidth > 0); var n = 0; dlgs.forEach(d => { n += d.querySelectorAll("video").length; }); return JSON.stringify({videos: n}); }'
```

**Album mode 5-image cap:** if `unique_blobs < N`, FB silently dropped uploads past the first 5. Recover via the editor:
1. Click "Chỉnh sửa tất cả" / "Edit all" button in the composer (re-snapshot to find its ref).
2. In the editor view, find the editor's own `input[type=file]` (a separate hidden input).
3. Upload the missing files via `--element` on that input.
4. Re-count (editor shows ~2 blob views per photo: main + strip — so unique blobs there is `≈ N × 2`; divide by 2).
5. Click "Xong" / "Done" to exit editor mode.

> ⚠️ Closing the editor with "Xong" can trigger a "Lưu bài viết này làm bản nháp?" (save as draft?) confirmation if FB thinks the user is leaving. Click "Đóng" (Close) on THAT dialog — not "Xóa bản nháp" (which deletes everything) and not "Lưu làm bản nháp" (which buries it in drafts). After closing, re-snapshot the editor view and click Xong again — the draft prompt should not re-appear.

### Phase 4 — Fill the caption

**No-dash rule (mandatory).** The user does not want ANY dash character in the caption. This is **authoring-level, not just stripping** — never write a caption (here, or any `video-caption.md` / `post.md` you author upstream) that contains a dash in the first place. Remove **all** of these: hyphen-minus `-`, double hyphen `--`, en-dash `–`, em-dash `—`, minus sign `−`, horizontal bar `―`. Replace each with whatever reads naturally:
- A dash used as a separator/clause break (`A — B`, `A - B`, `A -- B`) → `A, B` (comma + space).
- A hyphen joining a compound token (`OTC-dosing`, `anti-histamin`) → a single space or joined, whichever reads correctly in Vietnamese.
- Leading list dashes (`- item`) → drop the dash, keep the item.

Collapse any double spaces created by removal. Middle dot `·` and other non-dash separators are fine — leave them. **Verify zero dash characters remain** before filling (the fill snippet below strips them as a mechanical backstop, but the caption should already be dash-free).

Re-snapshot to find the contenteditable textbox ref (it's the modal's main composer field, not any per-image alt-text field):

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Create a public post|Tạo bài viết|What's on your mind|Bạn đang nghĩ gì|textbox" | head -5
```

Fill via `fill --fields` (never `type`):

```bash
# Reads caption, strips ALL dash chars (backstop for the no-dash rule), JSON-encodes.
PJSON=$(cat "$CAPTION_FILE" | python3 -c '
import sys, re, json
t = sys.stdin.read()
t = re.sub(r"\s*[-‐‑‒–—―−]+\s*", lambda m: ", " if m.group(0).strip() else " ", t)  # dash (with spaces) -> ", "
t = re.sub(r"(?m)^,\s*", "", t)          # a line that started with a list dash: drop the leading comma
t = re.sub(r"[ \t]{2,}", " ", t)          # collapse doubled spaces
t = re.sub(r"\s+,", ",", t)               # no space before comma
t = re.sub(r",\s*,", ",", t)              # no doubled commas
print(json.dumps(t))
')
OPENCLAW_TIMEOUT=120000 openclaw browser fill --fields "[{\"ref\":\"<textbox_ref>\",\"value\":$PJSON}]"
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
  | grep -iE 'button "Post"|button "Đăng"' | head -3
```

If `[disabled]`, the media upload isn't finished. **Wait longer for video and large albums:**
- Single image: 5-10 s
- Album (10 images): 30-60 s
- Video: 20-90 s (FB processes the video into multiple resolutions before allowing Post)

Loop with 5 s sleeps, up to 12 retries (60 s for albums, 60-90 s for video). If still disabled → flag INCOMPLETE.

If `DRY_RUN=true`: stop here. Report `READY_BUT_NOT_POSTED` with media count, caption preview, and the Post button ref.

Otherwise click and verify:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser click <post_ref>
sleep 10  # album/video posts take longer to commit than single-image
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var dlgs = Array.from(document.querySelectorAll("[role=dialog]")).filter(d => d.offsetWidth > 0); var labels = dlgs.map(d => d.getAttribute("aria-label")); var composer_open = labels.some(l => l && /Tạo bài viết|Create post/i.test(l)); return JSON.stringify({composer_open, modal_labels: labels, url: location.href}); }'
```

If `composer_open: true` after 10 s, wait another 10 s. If still open, FB is processing or showing a confirmation — STOP and notify user; do NOT click anywhere automatically.

> **Video post-publish quirk:** after a successful video post, FB sometimes redirects to a media viewer URL like `/photo/?fbid=<id>` (yes, even for video). This is normal — the post is published. Modal labels like `Trình xem ảnh` / `Photo viewer` confirm this state.

### Phase 6 — Summary

```
Posted!
  MODE: <album|video>
  BATCH: <BATCH_ID>
  TARGET: <TARGET>
  MEDIA: <N> image(s)  OR  1 video (<filename>)
  CAPTION: <first 80 chars>…
  RESULT_URL: <location.href after publish>
```

For DRY_RUN:
```
DRY RUN — composer populated, NOT posted.
  Click Post manually in the browser, or re-run without DRY_RUN.
```

---

## Rules

1. **`OPENCLAW_TIMEOUT=60000`** prefix on every browser command (120000 for upload/fill of large content).
2. **Use `fill --fields`, never `type`, for Vietnamese captions.** Diacritics mangle otherwise.
3. **Re-snapshot after every mutation** (click, upload, fill). Refs change.
4. **Stage every file with a unique filename** under `/tmp/openclaw/uploads/`. FB sometimes dedups identical filenames within a session, dropping silently.
5. **One target per skill invocation.** Don't loop this to target multiple groups — use `facebook-post-to-groups` for that.
6. **Never automate login.** Login wall → STOP, notify user.
7. **Honor DRY_RUN.** First-time use against a new group → always DRY_RUN first.
8. **Default to `new-image-logo.jpg` (album) or the first `*.mp4` in `post-<SLOT>/video/` (video).** Override via `MEDIA_NAME`.
9. **Don't auto-generate captions.** The user authors the caption file. If missing, STOP — never invent marketing copy.
10. **Verify media count matches expected before clicking Post.** A partially-uploaded album posts the wrong subset and there's no undo without deleting + reposting.
11. **Wait longer for video and large albums.** Video upload + processing can take 60-90 s before Post becomes enabled.
12. **Verify composer modal closed after submit.** A still-open dialog post-click is the surest sign FB didn't accept the post. The `/photo/?fbid=...` redirect for video is normal.

---

## Error recovery

| Symptom | Action |
|---|---|
| FB redirects to /login | STOP. User must log in manually in Brave. |
| Caption file missing | STOP. Tell the user to create `album-caption.md` (or `post-<SLOT>/video-caption.md`). Don't auto-generate. |
| `unique_blobs < N` (album mode) after upload | Some files failed. Use the "Edit all" recovery flow above. Retry once. If still short → STOP and report which slots are missing. |
| `videos === 0` after upload (video mode) | Either the file is corrupt or FB rejected the format. Verify with `file <staged>.mp4`; FB accepts H.264 MP4. If valid, the upload may still be in flight — wait 15 more seconds and recheck. If still 0 → STOP. |
| Post button stays `[disabled]` for > 90 s (video) or > 60 s (album) | Likely a stalled upload or rate limit. Re-snapshot once. If still disabled → flag INCOMPLETE and don't retry; FB may have shadow-rejected the post. |
| Modal closes mid-upload | A click cascaded too fast or you bumped focus. Re-open composer, click Photo/video, re-arm upload. Use `sleep 3` between trigger clicks. |
| Only 5 of N images attached after `upload --element` (album mode) | FB silently drops uploads past 5 on the first programmatic batch. Use the "Chỉnh sửa tất cả" / "Edit all" recovery flow. |
| "Lưu bài viết này làm bản nháp?" prompt appears after clicking "Xong" | Click "Đóng" (Close) — NEVER "Xóa bản nháp" (deletes work) or "Lưu làm bản nháp" (sends to drafts). |
| Caption fill reports `filled 1 field(s)` but text doesn't appear | The ref pointed at a sibling/empty textbox. Lexical's actual editor is sometimes outside the dialog DOM — verify via `document.querySelectorAll("[contenteditable=true]")` globally and read the largest one. Re-snapshot the textbox ref and re-fill. |
| Group composer opens TWO dialogs (one unlabeled with the file input, one labeled `Tạo bài viết`) | This is normal in some groups. The file is bound to the unlabeled dialog's input but appears in the `Tạo bài viết` composer after a few seconds. Verify via a broader scan across `document.querySelectorAll("[role=dialog]")`. |
| Album order in the post differs from `SLOT_ORDER` | FB sometimes reorders by EXIF timestamp. Rename staged files with a numeric prefix (`01-<slot>.jpg`, `02-<slot>.jpg`, …) before uploading — FB respects filename sort for tied timestamps. |
| Browser redirects to `/photo/?fbid=...` after video publish | Normal for video posts. The post IS published. Modal closed. Done. |
| Rate limit dialog | STOP. Notify user. Wait ≥ 15 minutes before retrying. |
| Album image quality degraded in feed | FB compresses albums more aggressively than single images. Compare to `image-1-full.png` source — if loss is severe, post as separate single-image posts instead. |
| Video held for group moderation | Some pharmacy/medical groups require admin approval. The post will show as "Pending" in your group activity. Nothing to retry — wait. |

---

## What NOT to do

- **Don't post a partial album.** If any image failed to attach, STOP and let the user decide — never publish a subset silently.
- **Don't auto-retry after a failed post.** May have published silently; reposting risks a duplicate flagged by FB.
- **Don't `type` Vietnamese text.** Always `fill --fields`.
- **Don't paste the caption via clipboard.** Diacritic risk.
- **Don't run `openclaw browser stop` mid-flow.** It logs out FB.
- **Don't modify the caption file from inside this skill.** If a tweak is needed, edit it externally and re-run.
- **Don't loop this skill to target multiple groups.** For batch-to-groups, hand off to `facebook-post-to-groups` (it can iterate group URLs from a Notion list).
- **Don't reorder slots silently.** If `SLOT_ORDER` overrides the default numeric order, surface that in the summary.
- **Don't auto-compose a video caption from the album caption.** The album caption describes a multi-topic album; a single video covers ONE topic and needs its own caption. Ask the user.

---

## Future hooks (not implemented yet)

- `--auto-caption` mode that synthesizes the caption by feeding `_topics.json` (album) or the design.json (video) through ChatGPT.
- Multi-target mode: after a successful post, navigate to additional group URLs and clone-paste the same composer state. (For now: use `facebook-post-to-groups`.)
- Permalink capture: after publish, grab the new post's URL and write it back to `<BATCH_DIR>/published.json` for tracking.
- Hashtag injection: read `<BATCH_DIR>/hashtags.txt` and append to the caption automatically.
- Reel mode: detect short-form vertical video (9:16, < 60 s) and route through FB's Reel composer instead of the standard post composer.
