# Clipboard Paste for Pinpoint — Design

**Date:** 2026-07-16
**Status:** Approved (pending spec review)
**File affected:** `pinpoint-feedback-app.html` (single self-contained HTML app)

## Goal

Let a reviewer paste a screenshot from the clipboard directly into Pinpoint, removing the
"save screenshot to disk, then import it" step. The intended flow becomes:

1. Take a screenshot to the clipboard (`Cmd+Ctrl+Shift+4` on macOS, `Win+Shift+S` on Windows).
2. Open the Pinpoint bookmark.
3. Press `Cmd+V` / `Ctrl+V` anywhere on the page.
4. The screenshot appears as a new screen, ready to annotate.

This is a prerequisite for the later GitHub Pages hosting step: the hosted app's primary
entry flow depends on paste working.

## Non-Goals (parked)

- localStorage auto-save / session restore — revisit only if identified as a real pain point.
- PWA install.
- Chrome/Edge extension.
- GitHub Pages hosting — this is the **next** step, after paste is built and tested locally.

## Architecture

One new global `paste` event listener on `document`. It reuses the app's existing
`importFiles()` pipeline rather than duplicating any logic.

- The `paste` event is user-initiated, so reading pasted data needs **no** clipboard
  permission prompt. This is simpler and more broadly compatible than the async
  `navigator.clipboard.read()` API, so we use the event approach.
- On paste, iterate `e.clipboardData.items`, collect any whose `type` starts with `image/`
  via `getAsFile()`, and pass them through `importFiles()`.
- `importFiles()` already handles: image compression (WebP/JPEG, max 2000px wide),
  new-screen creation, thumbnail rendering, switching to the newest screen, and the
  "Screen added" / "N screens added" toast. Paste inherits all of it for free.

## Behavior & Data Flow

- Pasted image(s) become new screen(s), identical to drag/drop and file-picker import.
- Multiple images in a single paste are all imported.
- Pasted images typically have no filename. Drag/drop derives a screen name from the file
  name; paste falls back to a default name: **"Pasted screen"**.

## Correctness Nuance — Do Not Hijack Comment Editing

The paste handler must not interfere with editing a comment:

- If focus is in a **comment `textarea`** (or any editable input) **and** the clipboard is
  **text**, let the native paste happen. Do nothing special; show no toast.
- If the clipboard contains an **image**, import it as a new screen regardless of focus
  (an image cannot paste into a plain `textarea` anyway).
- If the paste is a **non-image and focus is NOT in an editable field**, show a gentle hint
  toast (see below).

## Edge Cases

| Case | Behavior |
|------|----------|
| Review mode (`state.mode === "review"`) | Paste blocked, consistent with the existing drag/drop handler. |
| Non-image paste, not in an editable field | Gentle hint toast: **"Copy an image or screenshot first."** |
| Non-image paste, focus in a comment field | Native paste proceeds; no toast. |
| Empty clipboard | Treated as non-image paste (hint toast unless in an editable field). |
| Image compression/decode failure | Reuses existing "Couldn't read that image" error toast (already in `importFiles`). |

## Discoverability (UI copy)

Chosen approach: "Add paste, keep hero."

- Keep the existing empty-state headline: **"Drop a design to review."**
- Update the empty-state body copy to mention pasting a screenshot in addition to drag/drop.
- Add a small keyboard hint for paste (`⌘V` / `Ctrl+V`) alongside the existing "or drag it in"
  affordance.

No changes to the headline, layout, or the stage-level "Click anywhere…" hint.

## Testing

The app is a single self-contained HTML file with no test framework, and adding one is out of
scope. Verification is manual, driving the real flow in a browser:

1. Copy an image/screenshot to the clipboard, paste into the app → a new screen appears with
   the "Screen added" toast.
2. Pin a comment on the pasted screen, then export a handoff file → confirm the screen and
   comment are present.
3. Paste text (not an image) with nothing focused → hint toast appears, canvas unchanged.
4. Paste text while editing a comment → text goes into the comment; no new screen, no toast.
5. Paste an image while a screen already exists → appended as a new screen and selected.
6. In an exported review-mode file, paste an image → nothing happens (blocked).

## Implementation Notes

- Add the listener in the existing `bindEvents()` function, next to the drag/drop handlers.
- Route through `importFiles()`; add only the default-name fallback ("Pasted screen") where a
  pasted `File` lacks a usable `name`.
- Keep the change minimal and consistent with the existing plain-ES5-style IIFE code in the file.
