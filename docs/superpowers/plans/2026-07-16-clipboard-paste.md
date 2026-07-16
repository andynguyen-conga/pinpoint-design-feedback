# Clipboard Paste Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a reviewer paste a screenshot from the clipboard directly into Pinpoint, so the flow becomes screenshot → open bookmark → ⌘V → annotate.

**Architecture:** Add one global `paste` listener on `document` inside the existing `bindEvents()`. It extracts image files from the clipboard and routes them through the existing `importFiles()` pipeline (compression, screen creation, thumbnail, toast). A small optional `nameHint` argument on `importFiles()` gives pasted images a sensible default name. A second change updates the empty-state copy to advertise paste.

**Tech Stack:** Plain ES5-style JavaScript in a single self-contained HTML file (`pinpoint-feedback-app.html`). No build step, no framework, no external dependencies.

## Global Constraints

- Single self-contained HTML file: `pinpoint-feedback-app.html`. No new files, no new runtime dependencies.
- Match the existing code style: ES5-flavored (`var`, function expressions), inside the existing IIFE.
- Paste is **blocked in review mode** (`state.mode === "review"`), consistent with the existing drag/drop handler.
- Pasted images with no usable filename get the default screen name **"Pasted screen"**.
- Do **not** hijack comment editing: a text paste while focus is in a `textarea`/`input` must fall through to native paste with no toast.
- Non-image paste outside an editable field shows the gentle toast **"Copy an image or screenshot first"**.
- Keep the empty-state headline **"Drop a design to review"** unchanged; only the body copy and the `.or` hint change.
- No test framework exists or is being added; verification is manual in a browser (Chrome/Edge, macOS/Windows).

---

### Task 1: Clipboard paste handler + `importFiles` name hint

**Files:**
- Modify: `pinpoint-feedback-app.html:469` (`importFiles` signature + default name)
- Modify: `pinpoint-feedback-app.html:800` (add `paste` listener in `bindEvents`, after the `drop` handler)

**Interfaces:**
- Consumes: existing `importFiles(fileList)`, `state.mode`, `toast(msg, isErr)`.
- Produces: `importFiles(fileList, nameHint)` — `nameHint` is an optional string used as the screen base name when provided (overrides per-file names). Pasted images call it with `"Pasted screen"`.

- [ ] **Step 1: Add the optional `nameHint` parameter to `importFiles`**

Change the signature at `pinpoint-feedback-app.html:469`:

```javascript
    function importFiles(fileList){
```

to:

```javascript
    function importFiles(fileList, nameHint){
```

- [ ] **Step 2: Use `nameHint` for the screen base name**

Change the base-name line at `pinpoint-feedback-app.html:478`:

```javascript
            var base = (file.name || "Screen").replace(/\.[^.]+$/,"");
```

to:

```javascript
            var base = (nameHint || file.name || "Screen").replace(/\.[^.]+$/,"");
```

- [ ] **Step 3: Add the `paste` listener in `bindEvents`**

Immediately after the closing `});` of the `window.addEventListener("drop", ...)` block (currently ending at `pinpoint-feedback-app.html:800`), insert:

```javascript
      // clipboard paste -> import image(s) as new screen(s)
      document.addEventListener("paste", function(e){
        if(state.mode === "review") return;
        var items = (e.clipboardData && e.clipboardData.items) || [];
        var imgs = [];
        for(var i=0; i<items.length; i++){
          if(items[i].kind === "file" && /^image\//.test(items[i].type)){
            var f = items[i].getAsFile();
            if(f) imgs.push(f);
          }
        }
        if(imgs.length){
          e.preventDefault();
          importFiles(imgs, "Pasted screen");
          return;
        }
        // no image on the clipboard
        var el = e.target || document.activeElement;
        var editable = el && (el.tagName === "TEXTAREA" || el.tagName === "INPUT");
        if(editable) return;            // let native paste into a comment happen
        toast("Copy an image or screenshot first", true);
      });
```

- [ ] **Step 4: Manual verification — paste an image**

Open `pinpoint-feedback-app.html` in a browser. Copy any image to the clipboard (macOS `Cmd+Ctrl+Shift+4` region shot, or copy an image from another app). Click the page, press `Cmd+V` / `Ctrl+V`.
Expected: a new screen appears in the rail named "Pasted screen", the stage shows it, and the "Screen added" toast fires. Clicking the stage drops a pin.

- [ ] **Step 5: Manual verification — non-image paste, nothing focused**

Copy plain text (e.g. from a text editor). Click empty canvas area (not a comment field), press `Cmd+V` / `Ctrl+V`.
Expected: toast "Copy an image or screenshot first"; no new screen; canvas unchanged.

- [ ] **Step 6: Manual verification — text paste while editing a comment**

With at least one screen and one pinned comment, click into the comment's textarea, then paste plain text.
Expected: the text is inserted into the comment; no new screen; no toast.

- [ ] **Step 7: Manual verification — review mode blocked**

Use "Save for handoff" to export a review file, open it, and press `Cmd+V` / `Ctrl+V` with an image on the clipboard.
Expected: nothing happens (paste blocked in review mode).

- [ ] **Step 8: Commit**

```bash
git add pinpoint-feedback-app.html
git commit -m "feat: paste screenshots from clipboard into Pinpoint"
```

---

### Task 2: Advertise paste in the empty-state copy

**Files:**
- Modify: `pinpoint-feedback-app.html:374` (empty-state body copy)
- Modify: `pinpoint-feedback-app.html:379` (`.or` hint line)

**Interfaces:**
- Consumes: nothing new. Pure copy/markup change to the `#empty` block.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Update the empty-state body paragraph**

Change `pinpoint-feedback-app.html:374`:

```html
        <p>Drag &amp; drop a screenshot of a UI or prototype anywhere here. Click on it to pin comments, then save the file to send your feedback back.</p>
```

to:

```html
        <p>Paste a screenshot (⌘V / Ctrl+V), or drag &amp; drop one anywhere here. Click it to pin comments, then save the file to send your feedback back.</p>
```

- [ ] **Step 2: Update the `.or` hint line**

Change `pinpoint-feedback-app.html:379`:

```html
        <div class="or">or drag it in</div>
```

to:

```html
        <div class="or">paste ⌘V &middot; or drag it in</div>
```

- [ ] **Step 3: Manual verification**

Open `pinpoint-feedback-app.html` in a browser with no screens loaded.
Expected: the empty-state headline still reads "Drop a design to review"; the body now mentions pasting a screenshot; the hint below the button reads "paste ⌘V · or drag it in".

- [ ] **Step 4: Commit**

```bash
git add pinpoint-feedback-app.html
git commit -m "docs: advertise clipboard paste in the empty-state copy"
```

---

## Self-Review

**1. Spec coverage:**
- Paste → new screen via `importFiles`: Task 1, Steps 1–4. ✓
- Multiple images in one paste: `importFiles` already loops the list; the paste handler passes all image files. ✓
- Default name "Pasted screen": Task 1, Steps 1–2. ✓
- Don't hijack comment editing: Task 1, Step 3 (`editable` check) + Step 6. ✓
- Non-image → gentle toast: Task 1, Step 3 + Step 5. ✓
- Review mode blocked: Task 1, Step 3 + Step 7. ✓
- Compression failure reuses existing error toast: inherited from `importFiles` (unchanged path). ✓
- Discoverability copy (keep hero, add paste + hint): Task 2. ✓
- Manual testing over the real flow: verification steps in both tasks. ✓

**2. Placeholder scan:** No TBD/TODO; every code step shows complete code. ✓

**3. Type consistency:** `importFiles(fileList, nameHint)` is defined in Task 1 and called with `("Pasted screen")` in the same task. `toast(msg, isErr)` matches the existing signature. ✓
