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
3. A small nav bridge script is injected so relative `<a href>` clicks switch documents
   inside the same viewer instead of leaving the page.
4. The result is rendered via `srcdoc` in a sandboxed iframe
   (`allow-scripts allow-forms allow-modals allow-popups allow-downloads allow-pointer-lock`).

Because the iframe is sandboxed without `allow-same-origin`, previewed pages run in an
opaque origin: scripts work, but `localStorage`, cookies and same-origin APIs are unavailable.

## Requirements

The native pickers use the File System Access API, so **Chrome / Edge / Opera on desktop**
give the best experience (`Reload` from disk, whole-folder open). Other browsers fall back to
plain `<input type="file">` / `<input webkitdirectory>` pickers; drag & drop works everywhere.

## Files

- `index.html` — the entire app (no build step, no dependencies)
