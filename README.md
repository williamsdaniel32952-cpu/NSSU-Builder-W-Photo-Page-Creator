# NSSU Input Portal — Deployment Package

Complete GitHub Pages deployment for the NSSU Input Portal and the Additional Photo Page Creator.

## What's in this package

| File | Purpose |
|------|---------|
| `index.html` | NSSU Input Form (the main app) |
| `nssu_photo_creator_with_rework_templates.html` | Additional Photo Page Creator |
| `NSSU_Common_Practice_Library_v3_2.xlsx` | Common defect/rework rules library (source) |
| `NSSU_Common_Practice_Library_v3_2.json` | Library used at runtime by the app |
| `.nojekyll` | Tells GitHub Pages to serve files as-is, no Jekyll processing |
| `scripts/regenerate_library_json.py` | Optional helper to rebuild the JSON from the XLSX |
| `README.md` | This file |

## Deploying to GitHub Pages

1. Replace **all files** in your repo's root with the files in this package.
2. Commit and push to your default branch (the one Pages serves from).
3. GitHub Pages should redeploy in ~30 seconds.
4. Open the site in an **incognito / private window** and hard-refresh (Ctrl+Shift+R / Cmd+Shift+R) to bypass any browser cache. Aggressive caching is the most common reason changes don't appear right away.

## Verifying a successful deployment

After hard-refresh, you should see:

- **Top-left of the masthead:** a "Photo Creator" button (dark gray)
- **Top-right of the masthead:** a "Home" button + hamburger menu + the NSSU logo
- **Right column** (scrolling down): Page 3 Work Instructions Preview, then Page 4 Rework Steps Preview (when rework is selected), then a Save as Draft button, then **Page 2 — Safety Section** below all of that
- **Left column** (Page 1 form): ends at "DTL / Back and Forth Frequency" — no "Page 1 Required Statements" row, no AI suggestion note under Save as Draft

If after pushing you still see the old version, view page source (right-click → "View Page Source") and search for `toPhotoCreatorBtn`. If the string is in the source, the file is deployed correctly and any visual regression is a browser-cache issue. If it's not in the source, the file isn't deployed.

---

## Feature inventory — full session changes layered

### NSSU Input Form (`index.html`)

**Layout & form structure**
- Page 2 Safety Section moved into the right column under the Rework Steps Preview to fill dead vertical space (PDF output unchanged — page builder reads safety rows from the new mount via `safetyMount` div)
- Page 1 Required Statements section removed (the three yellow notice rows about emailing the copy, green ink pen, and SRP/TM trained on defect)
- Misleading AI suggestion note removed from under Save as Draft (was implying drafts only available for Rework)
- Tablet/narrow-viewport CSS for checkboxes: below 900px, checkboxes stack vertically (boxmark on top, label below) so labels don't word-wrap one letter per line
- Defensive CSS on the masthead-left actions (`min-width: 140px`, `flex: 0 0 auto`) so the Photo Creator button can't collapse under any layout condition

**Signatures**
- Signature canvases 20% taller (48 → 58px canvas height, 72 → 86px sign-box min-height) for easier use on touch devices
- Revision flow: clears only the bottom 3 NSSU Approval signatures (SRP, RI/TM, RI/GL); keeps the top 2 trained-on initials so they survive a revision. New `clearApprovalSignatures()` function with synchronous flag reaffirmation to avoid the async image-onload race condition

**Library & defects**
- Library bug fix: blank Excel cells in the loaded library now correctly override the embedded `exactReworkAssociations` table (removed `STANDARD_FALLBACK_ENABLED`, `builtInBaseBehavior`, `buildStandardFallbackRule`, and `result.standardFallback` so the library is the sole source of truth when loaded)

**Photo Creator handoff**
- "Photo Creator" button in the masthead writes a localStorage handoff (`nssuPhotoHandoff`) with the top-section field values, Kanban entry, and the first sentences of inspection step 3, step 4, and rework step 1 (with leading step numbers/parens/punctuation stripped)
- Auto-saves the current NSSU as a draft (or preserves "initiated" status) when navigating to Photo Creator, then auto-restores it on return via `nssuReturnRecordId` localStorage flag (one-shot — flag deleted after restore so a normal page refresh shows a blank form)

### Photo Page Creator (`nssu_photo_creator_with_rework_templates.html`)

**NSSU handoff reception**
- Reads `nssuPhotoHandoff` on load, populates top-section inputs, and stores caption text in a `handoffCaptions` object (one-shot — handoff key cleared after read)
- "Back to NSSU" button correctly navigates to `index.html`

**Captions**
- Category-aware caption autofill: inspection templates pull autofilled captions from NSSU steps; rework templates always use the short hardcoded labels
- Comma-split rendering: when an autofilled caption contains a comma, splits at the first comma into two centered lines (e.g., `"If no Splits are present,"` / `"Part = Good"`)
- Doesn't mutate the global `labels` defaults — autofill is per-render, no leakage between template selections

**Witness Mark**
- New `witnessMark` slot type with blue 5px photo border and **black caption bar with white text** (separate from `reference` so existing reference templates aren't affected)
- Three witness templates updated to use `witnessMark` instead of `reference` for the photo slot paired with the witness panel
- Witness panel content made editable with text "Witness Mark Location: / 1st. = Blue / 2nd. = Yellow"
- Witness panel edits persist to JSON drafts (write-only for now)

**Slot interaction**
- Insert-on-drag reordering: drag handle in top-left of each slot, drop on another slot to insert before/after, or drop into empty grid cell to append to end of DOM order (CSS Grid auto-flow places it in the next empty cell)
- Independent row resizing: rows use `minmax(220px, 1fr)` so the initial layout fills the canvas, but resizing one slot only grows that slot's row track (siblings keep their natural height via `align-self: start` set by the resize handler)

**WIS Photos section**
- 12 buttons appear below the template area (Work Area, Whole Part, Good Photo, NG Photo, Witness Mark Photo, Suspect Area, Kanban, Rejection Step, Unpacking, Repacking, Rework Step, Rework Tools)
- Photo-type buttons whose role is already in the current template's slots are hidden (e.g., "Good Photo" hidden when template has a `good` slot)
- Each WIS photo button uses an SVG still-camera icon, signaling camera-only intent (`capture="environment"` opens the back camera directly on Android Chrome)
- Once a photo is taken, button turns green with a checkmark and shows the thumbnail; small red X clears it

**Inspection / Rework videos**
- Two extra buttons appended to the WIS Photos grid: **Inspection Video** (green-bordered, green film-camera icon) and **Rework Video** (blue-bordered, blue film-camera icon)
- Visually distinct from photo buttons (colored borders, colored icons, colored text)
- Use the device's built-in camera app via `<input type="file" accept="video/*" capture="environment">`
- Recorded video metadata + blob stored in memory; included in Export JPEGs as separate files

**Export**
- "Export JPEGs" button produces a single JPEG of the photo page (landscape, ~1600px long edge, named `Additional Photo Page - <kanban> - <sortLocation>.jpeg`) plus individual JPEGs for each filled template slot photo (named by slot type) and each WIS photo (named by type label)
- Videos export with their original mime/extension as `Inspection Video - <kanban> - <sortLocation>.mp4` (or `.webm`/`.mov`) and similarly for Rework Video
- All separate downloads, no zip
- JPEG quality 0.82, target ~200–500 KB per image — comfortable for email
- Photo page rendered via html2canvas (loaded from Cloudflare CDN — first export needs network, ~30 KB library)

**Print**
- Fit-to-page print CSS: landscape US Letter, 0.3" margins, transform-scale to fit the printable area
- All colored borders/captions preserved (`-webkit-print-color-adjust: exact`)
- Drag handles, resize handles, upload controls hidden in print
- Slots use `break-inside: avoid` so they don't split across pages

---

## Honest caveats

- **WIS photos and slot positions don't currently persist to JSON drafts** — they live in memory for the current session only. Refresh the photo creator and uploads are gone.
- **html2canvas is a CDN dependency** — without network connectivity the page-as-JPEG export won't work, but individual photo and video exports still will (they don't need html2canvas).
- **`capture="environment"` is a hint** — on Android Chrome and iOS Safari it opens the back camera directly. Some devices/browsers still show a chooser. On desktop, the file picker always opens. The visual cues (camera icons, tooltips) communicate intent; true enforcement requires the eventual APK wrapper to use Android's `MediaStore.ACTION_IMAGE_CAPTURE` intent.
- **Video file size depends on the device** — the user's tablet camera produces the file. On most Android tablets you'll get MP4 H.264 at the device's default quality. Long high-resolution clips can exceed Outlook's 20 MB attachment cap. Until APK wrap, the user should keep recordings short.
- **No TTS narration in this build** — was attempted but pulled because it required either a 25 MB ffmpeg.wasm library to merge audio onto silent video, or relying on the device speaker (won't work on a noisy factory floor). The right place for proper narrated video is the APK wrap, which can use Android's native `TextToSpeech.synthesizeToFile()` + `MediaMuxer` for clean audio in any environment.

## Cache-busting

If you push but still see the old version:

1. Open the site in a fresh incognito/private window
2. Hard-refresh with Ctrl+Shift+R (Windows) or Cmd+Shift+R (Mac)
3. If still stale, view page source and search for `toPhotoCreatorBtn` and `video-btn`. Both should be present. If they are, the file is deployed correctly and only the cache is stale.
