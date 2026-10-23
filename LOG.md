# texpresso fork log

This fork of TeXpresso (github.com/let-def/texpresso, MIT, by Frédéric Bour) adds one command for the live page of research-editor. The upstream code stays as it is, so a rebase stays easy. The license and the author notices do not change. Newest entry first.

## Open

- **(user) Make the GitHub fork and push.** Origin is still upstream (`let-def/texpresso`). The user makes the fork on GitHub, sets origin to it and pushes `main`. Then the registry entry in `shared/scripts/workspace.json` gets the new remote.

## 2026-10-23: render-page, a page as a PNG image for an editor with no window

- Host mini, session `f8428dc3-33fe-4fe2-867b-efb087a4b336` (agent latex-texpresso-fork), claude-opus-5-5 at effort high. Base: upstream `e8df770`.
- **Why.** The editor protocol of TeXpresso sends text, logs and SyncTeX lines, but no page. TeXpresso draws its pages only in its own window. research-editor shows the PDF in a web page, so it needs the page itself.
- **The command.** `(render-page id page width)` asks for the page `page` (0 is the first page) as a PNG image `width` pixels wide. The reply is one line: `(page id page count done width height "base64-png" draw-ms png-ms)`. With `-json`: `["page", id, page, count, done, width, height, "base64-png", draw-ms, png-ms]`.
  - `count` is the number of pages that the engine has now. `done` is true when the engine is at the end of the document, so `count` is final.
  - `page` is -1 when the document has no such page, or when the draw failed. The image is then empty.
  - `draw-ms` and `png-ms` are the times to draw the page into pixels and to encode the PNG.
- **How it works.** The command sets the page of the view, as the page keys do. So the engine runs TeX up to that page, and not further. The reply goes when the engine has shipped the page. An edit rolls the engine back to a snapshot before the change, so a page below the page count always has the new text. A new request replaces a request that waits: only the last one gets a reply. The command applies the delayed changes first.
- **The PNG.** The encoder of mupdf (`fz_new_buffer_from_pixmap_as_png`) took about 22 ms for a page 1200 pixels wide. The fork writes the PNG itself, with no row filter and zlib at level 1: about 9 ms, and a file about 10 % bigger. The draw of the page takes about 1 ms.
- **The end of the input.** A client of `render-page` has no window to close. So, after the first `render-page`, the process ends when its input ends, and no orphan engine stays. Without `render-page`, TeXpresso keeps its window open as before.
- **Times on the mini** (Apple M6), a page 1200 pixels wide, a revtex paper of 10 pages, the median of 8 one-word edits, from the write of the command to the reply: start of the paper 17 ms, middle 7 ms, end 17 ms. Of this, the PNG is 5 to 10 ms and TeX the rest. The first edit after the start takes about 90 to 130 ms.
- **Changed files.** `src/frontend/editor.h` (the command and the reply), `src/frontend/editor.c` (the parse and the reply), `src/frontend/main.c` (the request, the PNG and the end of the input). Each change has the comment "research-editor fork".
- **Not changed.** The window still opens. The client sets `SDL_VIDEODRIVER=dummy` (as the upstream tests do), so no window shows.
- **Build.** `brew install mupdf pkgconf sdl2` (macOS), then `make all`. The program is `build/texpresso`. research-editor finds it through `RESEARCH_EDITOR_TEXPRESSO`.
- **Known issue (not found again).** In the work of 2026-10-23 before the fork, TeXpresso stopped when several commands came in one read. With the fork, bursts of 8 to 20 commands in one write did not stop it (page moves and edits, several requests, 20 changes). The driver of research-editor still writes each command apart, and sends a request again when no reply comes.
