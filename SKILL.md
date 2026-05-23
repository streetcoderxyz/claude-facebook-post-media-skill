---
name: facebook-post-from-clone
description: Publish a cloned Facebook post (caption + image produced by the facebook-clone-post skill) to either a single Facebook group OR your own timeline, via OpenClaw browser automation with a logged-in Brave profile. Handles the image-attach composer flow, Vietnamese diacritics, and post-button-disabled-while-uploading timing. Triggers on "post the clone to <group URL>", "đăng bài clone lên timeline", "publish post-<N> to group X".
---

# Publish Cloned FB Post (single target)

Takes one post directory produced by the `facebook-clone-post` skill (containing `new-caption.txt` + `new-image-logo.jpg`) and publishes it to a single Facebook target — either a group URL or your own timeline. You write the source content; this skill performs the upload step.

## When to use this skill

- After running `/facebook-clone-post`, you've reviewed `post-<N>/new-image-logo.jpg` + `new-caption.txt` and want to publish that specific post to ONE destination.
- You want the caption + image to land on either a group OR your timeline; the skill picks the right composer flow based on the target URL.

Do NOT use for:
- Posting to MANY groups in batch → use `facebook-post-to-groups` (works from a Notion list of group URLs).
- Sharing an existing FB post by URL (no regeneration) → use `facebook-share-to-groups`.
- Composing brand-new content (not pre-cloned) → use `facebook-post-to-groups` with a hand-written `post.md`.
- Scheduling for later via Business Suite → use `facebook-schedule-business-suite`.

## Prerequisites

1. `openclaw browser status` → `running: true`. If not: `openclaw browser start && sleep 4`.
2. Brave profile already logged in to Facebook. Login walls → **stop and notify**, never automate login.
3. The clone directory must already exist (`post-<N>/new-image-logo.jpg` + `post-<N>/new-caption.txt`).
4. `OPENCLAW_TIMEOUT=60000` prefix on every `openclaw browser` command (or 120000 for heavy ops).

---

## Configurable Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| **CLONE_DIR** | Yes | — | Path to a `post-<N>` directory containing `new-caption.txt` and one image (`new-image-logo.jpg` preferred, falls back to `new-image.jpg`) |
| **TARGET** | Yes | — | Either a Facebook group URL (`https://www.facebook.com/groups/<id>`) OR `timeline` (post to own profile) OR a full profile URL |
| **IMAGE_FILE** | No | auto-pick | Override the image — by default, picks `new-image-logo.jpg` if present, else `new-image.jpg` |
| **CAPTION_FILE** | No | `new-caption.txt` | Override the caption source file |
| **DRY_RUN** | No | `false` | If `true`, fills composer + verifies button enabled but doesn't click Post. Useful for previewing before commit. |

---

## Workflow

### Phase 0 — Setup + validate inputs

1. Verify `CLONE_DIR` exists and has the expected files:

   ```bash
   test -d "<CLONE_DIR>" || { echo "ERR: $CLONE_DIR not a directory"; exit 1; }
   CAPTION="<CLONE_DIR>/${CAPTION_FILE:-new-caption.txt}"
   test -f "$CAPTION" || { echo "ERR: $CAPTION missing"; exit 1; }
   # Pick image: prefer logoed version
   if [ -n "$IMAGE_FILE" ]; then IMAGE="<CLONE_DIR>/$IMAGE_FILE"; \
   elif [ -f "<CLONE_DIR>/new-image-logo.jpg" ]; then IMAGE="<CLONE_DIR>/new-image-logo.jpg"; \
   else IMAGE="<CLONE_DIR>/new-image.jpg"; fi
   test -f "$IMAGE" || { echo "ERR: no image found"; exit 1; }
   ```

2. Classify `TARGET`:
   - `groups/<id>` in URL → **group flow**
   - `timeline` keyword OR URL is facebook.com root / `facebook.com/<username>` → **timeline flow**
   - Anything else → STOP, ask user to clarify.

3. Verify browser:

   ```bash
   OPENCLAW_TIMEOUT=60000 openclaw browser status
   ```

   If `running: false` → `openclaw browser start && sleep 4`.

4. **Stage the image** to OpenClaw's uploads dir with a unique-per-target name so a retry doesn't trip FB's "same upload" dedup if it has one:

   ```bash
   STAGED=/tmp/openclaw/uploads/fbpost-$(basename "<CLONE_DIR>")-$(date +%s).jpg
   cp "$IMAGE" "$STAGED"
   ```

### Phase 1 — Navigate to target

For group flow:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser open "<TARGET>" && sleep 5
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => JSON.stringify({title: document.title.slice(0,80), url: location.href})'
```

For timeline flow (own profile):

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser open "https://www.facebook.com/me" && sleep 5
```

**Verify not a login wall.** If `document.title` contains "Log in to Facebook" OR the URL redirects to `/login` → **STOP**, notify the user to log in manually in Brave.

### Phase 2 — Open the composer

The composer trigger button differs by target:

| Target | Trigger button text (bilingual) |
|---|---|
| Group | "Write something..." / "Viết gì đó..." / "Discussion" tab → composer |
| Timeline | "What's on your mind?" / "Bạn đang nghĩ gì?" |

Re-snapshot to find the trigger ref:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Write something|Viết gì đó|What's on your mind|Bạn đang nghĩ gì|Discussion|Thảo luận" | head -5
```

Click the trigger:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser click <trigger_ref>
sleep 3
```

A modal composer opens with an empty contenteditable textbox.

### Phase 3 — Attach the image

Unlike the file-upload flow in `facebook-post-to-groups` (which uses "More post options" → "File" for zips), images go through the inline **Photo/Video** button. Re-snapshot:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Photo/video|Ảnh/video|Add to your post|Thêm vào bài viết" | head -10
```

Click the **"Photo/video"** (or **"Ảnh/video"**) button:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser click <photo_video_ref>
sleep 2
```

Now click the **"Add photos/videos"** trigger (the visible "+" or "Add" button inside the inline photo picker) WITH upload armed:

```bash
OPENCLAW_TIMEOUT=120000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Add photos|Add photo|Thêm ảnh|Photo|Choose file|Chọn file" | head -10
# Then:
OPENCLAW_TIMEOUT=120000 openclaw browser upload --ref <add_photos_ref> "$STAGED"
sleep 6
```

**Verify the thumbnail rendered** in the composer:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var imgs = Array.from(document.querySelectorAll("img")).filter(function(i){ return /blob:|scontent|fbcdn/.test(i.src) && i.width > 80 && i.width < 600; }); return JSON.stringify({thumbs: imgs.length, firstSrc: imgs[0]?imgs[0].src.slice(0,60):""}); }'
```

If `thumbs === 0` → upload may have failed silently or modal closed. Recover: re-snapshot, find the **"Photo/Video"** trigger again, click it, re-arm upload with `--ref` and retry once. If still 0 → STOP and notify user.

### Phase 4 — Fill the caption (Vietnamese-safe)

Re-snapshot to find the contenteditable textbox ref:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "Create a public post|Tạo bài viết|What's on your mind|Bạn đang nghĩ gì|textbox" | head -5
```

Read caption + fill via `fill --fields` (never `type` — diacritics get mangled):

```bash
P=$(cat "$CAPTION") && PJSON=$(printf '%s' "$P" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read()))') \
  && OPENCLAW_TIMEOUT=120000 openclaw browser fill --fields "[{\"ref\":\"<textbox_ref>\",\"value\":$PJSON}]"
sleep 2
```

### Phase 5 — Submit + verify

Re-snapshot to find the Post button:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser snapshot --interactive --compact 2>&1 \
  | grep -iE "button \"Post\"|button \"Đăng\"|Publish|Xuất bản" | head -3
```

If `[disabled]` appears next to the button name, the image upload isn't finished — wait 4 s and re-snapshot. Retry up to 3 times. If still disabled, flag as INCOMPLETE.

If `DRY_RUN=true`: stop here. Report `READY_BUT_NOT_POSTED` with composer state.

Otherwise click:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser click <post_ref>
sleep 6
```

**Verify publication.** The composer modal should close, and:
- Group flow: URL stays on `/groups/<id>` and the new post appears at top of feed
- Timeline flow: URL stays on `/me` and the new post appears at top

Quick check:

```bash
OPENCLAW_TIMEOUT=60000 openclaw browser evaluate --fn '() => { var modal = document.querySelector("[role=dialog][aria-label*=\"Create\" i], [role=dialog][aria-label*=\"Tạo\" i]"); return JSON.stringify({modal_still_open: !!modal, url: location.href}); }'
```

If `modal_still_open: true` after 6 s → posting in progress or stuck. Wait another 6 s. If still open, FB may be showing an "are you sure" / "rate limit" dialog — STOP and notify user.

### Phase 6 — Summary

```
Posted!
  CLONE: <CLONE_DIR>
  TARGET: <TARGET>
  IMAGE: <basename of IMAGE>
  CAPTION: <first 80 chars>...
```

For DRY_RUN:
```
DRY RUN — composer populated, NOT posted.
  Click Post manually in the browser to publish, or re-run without DRY_RUN.
```

---

## Rules

1. **`OPENCLAW_TIMEOUT=60000`** prefix on every browser command (120000 for upload/fill of large content).
2. **Use `fill --fields`, never `type`, for Vietnamese captions.** Diacritics mangle otherwise.
3. **Re-snapshot after every mutation** (click, upload, fill). Refs change.
4. **Stage the image with a unique filename** under `/tmp/openclaw/uploads/` (include the clone batch + timestamp). FB occasionally dedups identical uploads in the same session.
5. **Never automate login.** Login wall → STOP, notify user.
6. **Honor DRY_RUN.** It exists so the user can sanity-check before publishing.
7. **One post per skill invocation.** This isn't a batch skill — for many groups, hand off to `facebook-post-to-groups`.
8. **Default to `new-image-logo.jpg`** when present. The clone batch's brand-stamped image is the intended publish version; `new-image.jpg` is the un-logoed fallback.
9. **Don't auto-edit the caption.** The user already accepted `new-caption.txt`. Don't trim, reformat, or add hashtags unless explicitly asked.
10. **Verify modal closed after submit.** A "still-open dialog" is the surest sign FB didn't accept the post.

---

## Error recovery

| Symptom | Action |
|---|---|
| FB redirects to /login | STOP. User must log in to Facebook manually in Brave. |
| "Write something..." trigger not found on a group | The group may require approval to post, or FB UI is in a variant. Re-snapshot wider; look for "Anyone can post" vs "Admin only" notices. If admin-only → STOP and notify. |
| Image thumb doesn't appear after upload | Re-click "Photo/video", re-arm upload with `--ref`. If still missing after 2 retries → STOP. Common cause: modal closed between snapshot and click — re-open composer first. |
| Post button stays `[disabled]` after fill + image | Image still uploading. Re-snapshot every 4 s for up to 3 retries. If still disabled, the upload likely failed silently — flag INCOMPLETE. |
| Modal closes prematurely after `Photo/video` click | The trigger button click cascaded too fast. Re-open composer, click "Photo/video" with `sleep 3` before next action. |
| `fill` returns "filled 1 field" but textbox visually empty | Wrong ref — the snapshot picked the search box or a stale composer. Re-snapshot, the active composer's textbox usually appears after the modal opens. Look for `role=dialog` ancestors. |
| Post succeeds but no confirmation visible | Refresh the target URL and inspect the top of feed. FB sometimes silently posts without UI feedback. Search for the caption's first line in the feed. |
| Rate limit / "you're posting too fast" dialog | STOP. Notify user. Wait ≥ 15 minutes before retrying. Do NOT automate the dismiss-and-retry. |

---

## What NOT to do

- **Don't loop this skill over many targets.** It posts to ONE. For batch → `facebook-post-to-groups`.
- **Don't auto-retry a failed post by reposting.** If a post may have published silently, refresh and check feed — never post twice; FB will flag duplicates and may temp-ban posting.
- **Don't `type` Vietnamese text.** Diacritics get mangled. Always `fill --fields`.
- **Don't paste the caption via clipboard.** Same diacritic risk.
- **Don't modify the caption file inline.** If you need a tweak, edit `new-caption.txt` first, then re-run.
- **Don't run `openclaw browser stop` mid-flow.** It logs out FB.
- **Don't skip DRY_RUN for first-time use against a new group.** Verify the composer fills correctly before committing to a public post.

---

## Future hooks (not implemented yet)

- `EDIT_CAPTION_BEFORE_POST=true` — open the caption file in `$EDITOR` for last-minute tweaks before publish.
- Multi-image support (FB groups accept albums via the same Photo/video flow). Would need to iterate `--ref` clicks of the "Add more" plus button.
- Auto-tag accounts mentioned in caption (`@dược sĩ Bình` → tag suggestion picker).
- After publish, append a row to a Notion log table with timestamp + permalink for tracking.
