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

1. Replace **all files** in your repo's root with the files in this package (including the hidden `.nojekyll`).
2. Commit and push to your default branch.
3. GitHub Pages should redeploy in ~30 seconds.
4. Open the site in an **incognito / private window** and hard-refresh (Ctrl+Shift+R / Cmd+Shift+R) to bypass any browser cache.

## Verifying a successful deployment

After hard-refresh:

- **NSSU form top-left:** "Photo Creator" button (dark gray)
- **NSSU form top-right:** "Home" + hamburger + NSSU logo
- **NSSU form right column** (scrolling down): Page 3 preview → Page 4 (when rework selected) → Save as Draft → Page 2 Safety Section
- **NSSU form left column:** ends at "DTL / Back and Forth Frequency" — no Page 1 Required Statements row
- **Photo Creator toolbar:** "← Back to Templates", "Clear Photos", "Export JPEGs", and "Current Template" (no Print button)
- **Photo Creator masthead:** "Templates" + "Save Draft" (no Print/Save PDF button)
- **WIS Photos section** below the photo grid: still-camera icons for the photo types, plus two distinctly colored video buttons at the end (green Inspection Video, blue Rework Video)

---

## Latest changes (since the May 5 baseline)

This package contains everything from the May 5 session plus the May 6 changes:

### May 6 — Print path eliminated
- Removed the "Print / Save PDF" button from the photo creator masthead
- Removed the secondary "Print" button from the template toolbar
- Stripped 115 lines of `@media print` CSS — no print path remains
- Sole export route is **Export JPEGs**

### May 6 — JPEG export right-edge clipping fix
- html2canvas was sub-pixel rounding the right edge during capture, shaving the rightmost pixel column off slot borders, caption bars, and banner inputs
- Fixed by:
  - Adding 8px right-side white guard to the capture width
  - Explicit `windowWidth`, `width`, `height`, `x`, `y`, `scrollX` parameters passed to `html2canvas`
  - Forces a clean full-width capture instead of a `boundingClientRect`-inferred capture

---

## Full feature inventory

### NSSU Input Form (`index.html`)

**Layout & form structure**
- Page 2 Safety Section moved into the right column under the Rework Steps Preview to fill dead vertical space (PDF output unchanged)
- Page 1 Required Statements section removed
- Misleading AI suggestion note removed from under Save as Draft
- Tablet/narrow-viewport CSS for checkboxes: below 900px, checkboxes stack vertically so labels don't word-wrap one letter per line
- Defensive masthead-left CSS (`min-width: 140px`, `flex: 0 0 auto`) so the Photo Creator button can't collapse

**Signatures**
- Signature canvases 20% taller for easier touch use
- Revision flow: clears only the bottom 3 NSSU Approval signatures; keeps the top 2 trained-on initials so they survive a revision
- Synchronous `signatureStates` flag reaffirmation avoids the async image-onload race condition

**Library**
- Bug fix: blank Excel cells correctly override the embedded `exactReworkAssociations` table

**Photo Creator handoff**
- Photo Creator button writes a localStorage handoff with top-section field values, kanban entry, and first sentences of inspection step 3, step 4, and rework step 1
- Auto-saves the current NSSU as a draft; auto-restores it on return via a one-shot `nssuReturnRecordId` flag

### Photo Page Creator (`nssu_photo_creator_with_rework_templates.html`)

**Handoff & captions**
- Reads `nssuPhotoHandoff` on load, populates top-section inputs, stores caption text in a separate `handoffCaptions` object (no global state mutation)
- Category-aware caption autofill: inspection templates use NSSU step text for `good` and `ng` slots; rework templates use short hardcoded labels
- Comma-split caption rendering: `"If no Splits are present, Part = Good"` renders as two centered lines
- "Back to NSSU" navigates to `index.html`

**Witness Mark slot type**
- Blue 5px photo border + black caption with white text (separate from `reference` so reference templates aren't affected)
- Three witness templates updated to use `witnessMark`
- Witness panel content editable, text "Witness Mark Location: / 1st. = Blue / 2nd. = Yellow"

**Slot interaction**
- Drag handle on each slot for insert-on-drag reordering (drop on another slot = insert before/after; drop on empty grid cell = append to end of DOM order)
- Independent row resizing: `minmax(220px, 1fr)` row tracks combined with `align-self: start` on individually-resized slots

**WIS Photos section**
- 12 photo-type buttons (Work Area, Whole Part, Good, NG, Witness Mark, Suspect Area, Kanban, Rejection Step, Unpacking, Repacking, Rework Step, Rework Tools)
- Photo-type buttons whose role is already in the current template's slots are auto-hidden
- Each WIS button uses an SVG still-camera icon, opens device's back camera via `capture="environment"`

**Inspection / Rework videos**
- Two distinctly colored buttons at the end of the WIS grid: green Inspection Video, blue Rework Video
- Use device's built-in camera app via `<input type="file" accept="video/*" capture="environment">`
- Recorded files included in Export JPEGs as separate downloads

**Export**
- "Export JPEGs" produces:
  - Single landscape JPEG of the whole photo page (banner + photo grid) at ~1600px long edge, named `Additional Photo Page - <kanban> - <sortLocation>.jpeg`
  - Each filled template slot photo as its own JPEG at ~1280px long edge, named by slot type
  - Each WIS photo as its own JPEG, named by WIS type label
  - Each recorded video as its own file (mp4/webm/mov) with proper naming
- All separate downloads, no zip
- JPEG quality 0.82, target 200–500 KB per image — comfortable for email
- Photo page rendered via html2canvas (loaded from Cloudflare CDN)

---

## Honest caveats

- **WIS photos, video recordings, and slot positions don't persist to JSON drafts** — they live in memory for the current session only
- **html2canvas is a CDN dependency** — without network the page-as-JPEG export fails; individual photo and video exports still work
- **Camera-only enforcement is best-effort** — `capture="environment"` opens the back camera on Android Chrome and iOS Safari but desktop browsers always show a file picker
- **Video file size depends on the device camera** — long high-res clips can exceed Outlook's 20 MB attachment cap; users should keep recordings short
- **No TTS narration** — was attempted but pulled because the factory floor is too loud for speaker-to-mic capture; belongs in the eventual APK build
- **Slot resize is parent-bounded but not page-bounded** — a slot resized to fill its grid container still hits the page content edge; if a future change makes the template area larger than the page, slots could push past the page edge. Currently they don't.

## Cache-busting

If after pushing you still see the old version:

1. Open the site in a fresh incognito/private window
2. Hard-refresh with Ctrl+Shift+R (Windows) or Cmd+Shift+R (Mac)
3. View page source and search for `Export JPEGs` and `video-btn`. Both should be present. If they are, the file is deployed correctly and only the cache is stale.
