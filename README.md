# Pinpoint — Design Feedback

A lightweight, zero-install tool for pinning comments on screenshots and sending
the feedback back to a designer or developer. Built for managers and stakeholders
to review UI/prototypes without any accounts, servers, or setup.

## How it's used

1. Take a screenshot to the clipboard — `Cmd+Ctrl+Shift+4` (macOS) or `Win+Shift+S` (Windows).
2. Open the app (bookmark the hosted URL, below).
3. Press `Cmd+V` / `Ctrl+V` to drop the screenshot in — or drag/drop a file, or use "Choose image".
4. Click anywhere on the image to drop numbered pin comments.
5. **Save for handoff** → produces a self-contained, read-only HTML file you can email or share
   in Teams. The recipient opens it in any browser; no install needed.
   (**Working copy** saves an editable version you can reopen and continue later.)

## Hosting

Hosted on **GitHub Pages** from the `main` branch, root folder.

- Live URL: `https://andynguyen-conga.github.io/pinpoint-design-feedback/`
- `index.html` is the file Pages serves at the clean root URL.
- The whole app is one self-contained HTML file — 100% client-side, no backend, no build step.
  Exports are generated in the browser and downloaded locally.

## Files

| File | Purpose |
|------|---------|
| `index.html` | Copy served by GitHub Pages at the root URL. |
| `pinpoint-feedback-app.html` | The original app file, kept alongside `index.html`. |
| `docs/superpowers/specs/` | Design spec(s). |
| `docs/superpowers/plans/` | Implementation plan(s). |

> **Maintenance note:** `index.html` and `pinpoint-feedback-app.html` are intentional
> duplicates. Any change to the app must be applied to **both** files, or pick one as the
> single source of truth and regenerate the other (`cp pinpoint-feedback-app.html index.html`).

Local screenshots (`*.png`) are git-ignored and never pushed.

## Key decisions

- **Distribution = a bookmarkable hosted URL, not an install.** The tool's whole value is
  spreading with zero friction: no accounts, cross-platform, works offline after first load,
  and the handoff output is an emailable HTML file. Anything that adds an install step works
  against adoption.
- **Chrome/Edge extension was considered and shelved.** An extension could capture screenshots
  in-context (removing the screenshot→switch step), but it would require installation, trip
  enterprise security review, be Chrome/Edge-only, and sacrifice the emailable-file model.
  Verdict: technically feasible, but the install/distribution cost outweighs the capture win
  for enterprise adoption. Revisit only if in-context capture becomes a proven need.
- **Clipboard paste** was added instead — it removes the "save screenshot to disk then import"
  friction while keeping every advantage of the hosted-file model.

## Parked ideas (revisit if a real need emerges)

- **localStorage auto-save / session restore** — reopening the bookmark starts a fresh, empty
  session today. Add persistence only if "come back to a review later" proves to be a pain point.
- **PWA install** — a manifest + service worker would give a Dock/launcher icon and full offline
  support. Nice-to-have, not needed for the current flow.
- **Chrome/Edge extension** — see Key decisions above.
