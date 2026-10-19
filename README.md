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
4. The document is served from a `blob:` URL in an iframe **with no `sandbox` attribute**.

### Why no `sandbox` attribute

`blob:` URLs can only be loaded by the document that created them, so a sandboxed iframe
(which gets an opaque origin) silently fails to load *every* stylesheet, image and font —
the text still renders, so it looks like nothing is wrong. The only fix is
`allow-same-origin`, and Chrome deliberately warns about `allow-scripts` +
`allow-same-origin` ("can escape its sandboxing") because that combination is not a security
boundary: the preview can reach this page either way. Rather than ship a configuration that
emits a correct warning about a boundary it does not provide, the attribute is omitted.

Consequences, stated plainly: a previewed page runs same-origin with the viewer. It can touch
this page's DOM and its `localStorage`/cookies, and scripted `top.location = …` can navigate
you away. Only open files you trust. The nav bridge does keep `target="_top"`/`"_parent"` link
clicks inside the frame, which is what the sandbox used to block.

(If you do need true isolation, drop `allow-same-origin` and inline every asset as a
`data:` URL instead of `blob:` — measured working at 1.5 MB per document, but not at 5 MB,
so it is not viable for asset-heavy folders.)

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

- No sandbox warning is emitted: the iframe has no `sandbox` attribute at all (see
  *Why no `sandbox` attribute* above). If you add one back for any reason, Chrome will log
  `An iframe which has both allow-scripts and allow-same-origin … can escape its sandboxing`,
  and removing `allow-same-origin` instead breaks every blob-loaded style and image.
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
