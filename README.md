# HTML Viewer

A single-file, fullscreen HTML previewer. Open local `.html` files or whole folders
(so relative CSS/JS/images resolve) and preview them in a sandboxed iframe.

**Live:** https://matrix-research-ai.github.io/html-viewer/

## Usage

Open the live URL, then:

- **Open file** — `O` — pick a single `.html`/`.htm` file
- **Open folder** — `Shift`+`O` — pick a folder; `index.html` is loaded if present and
  relative assets resolve against the folder
- **Drag & drop** — drop one or many files anywhere on the window
- **Reload** — `R` — re-reads from disk (only when opened via the native picker)
- **Zoom** — `+` / `-` / `0` (reset)
- **Fullscreen** — `F`
- **Help** — `?`

## How it works

Everything runs client-side; no file ever leaves the machine.

1. Files are collected into an in-memory path → `File` map and each becomes a `blob:` URL.
2. The entry HTML is parsed with `DOMParser`, then every `src`/`href`/`srcset`/`poster`/
   `data`/`action` and `style` attribute is rewritten to the matching blob URL, plus
   `<style>` blocks and linked stylesheets (with recursive `@import` and `url()` rewriting).
3. A small nav bridge script is injected as the **first** script in `<head>` so relative
   `<a href>` clicks switch documents inside the same viewer instead of leaving the page.
4. The document is served from a `blob:` URL in a sandboxed iframe
   (`allow-scripts allow-same-origin allow-forms allow-modals allow-popups allow-downloads allow-pointer-lock`).

`allow-same-origin` is **required**: Chrome only lets a document load `blob:` URLs created by
its own origin, so without it every stylesheet, image and font in the preview silently fails.
The trade-off is that previewed pages share this page's origin — treat the preview as you
would the file itself, and only open files you trust. localStorage/cookies written by a
preview belong to the viewer's origin.

### Why `blob:` and not `srcdoc`

A rendered document gets a real document URL (`blob:https://<host>/<uuid>`) instead of
`about:srcdoc`. Chrome refuses to retarget a `blob:`/`about:srcdoc` document to any other URL,
so `history.replaceState`/`pushState` behave like this inside a preview:

| call | `about:srcdoc` | `blob:` |
|---|---|---|
| `replaceState(null,'','#slide-2')` | throws `SecurityError` | works |
| `pushState(null,'','#slide-2')` | throws `SecurityError` | works |
| `replaceState(null,'','https://<host>/page#slide-2')` | throws | throws |
| `location.hash = '#slide-2'` | works | works |

An uncaught `SecurityError` inside a slide router's `show()` aborts that function's caller —
the slide updates but the counter/nav state after the call never runs, and the console fills
with errors. The injected guard therefore wraps `replaceState`/`pushState`: it tries the real
call first, and on failure applies the URL's fragment through `location.hash` when the target
is same-origin (fragment-only URLs are handled directly). Hash routing in previewed decks
keeps working; nothing is ever thrown into the page.

## Console messages you may see

- `An iframe which has both allow-scripts and allow-same-origin for its sandbox attribute
  can escape its sandboxing.` — **expected, deliberate.** Chrome always emits this when those
  two flags are combined, and `allow-same-origin` is required for `blob:` subresources to
  load at all. Dropping it silently breaks every stylesheet/image/font; dropping
  `allow-scripts` breaks scripted decks. The attribute is kept for the restrictions it still
  enforces (no top-level navigation, no popup escape).
- `Error with Permissions-Policy header: Unrecognized feature: 'run-ad-auction'` (and
  `join-ad-interest-group`, `private-aggregation`, `attribution-reporting`) — **not from this
  page.** GitHub Pages sends no `Permissions-Policy` header, and this file contains none of
  those strings. Those lines are logged for whichever *other* document in the browser received
  a header naming legacy Privacy Sandbox features that current Chrome has renamed or removed.
  Check the Console source column / the request's Response Headers to find the culprit.

## Requirements

The native pickers use the File System Access API, so **Chrome / Edge / Opera on desktop**
give the best experience (`Reload` from disk, whole-folder open). Other browsers fall back to
plain `<input type="file">` / `<input webkitdirectory>` pickers; drag & drop works everywhere.

## Files

- `index.html` — the entire app (no build step, no dependencies)
