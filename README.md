# Lexxy editors painted before they're defined

Lexxy defines its custom elements with `setTimeout(defineElements, 0)` at the end of `src/index.js`. A timer runs in a later task, and the browser may paint between the task that ran the page's scripts and that one. A `<lexxy-editor>` that's already in the HTML can then be painted before it's defined: an empty box, without its toolbar or placeholder, for a frame or two, then the editor.

An app can't avoid it on its side. `blocking="render"` on its entry module (Chrome, Edge, Safari 18.2+) holds the first paint until that module and everything it imports have run. Lexxy's timer comes after that hold has ended.

The fix proposed in [basecamp/lexxy#1314](https://github.com/basecamp/lexxy/pull/1314) defines the elements with `queueMicrotask(defineElements)` instead: still after a `configure()` call that follows the import, and before the browser can paint. Its second commit keeps a later `configure()` call working, from another module script or after an await: until the next task, `configure()` rebuilds the editors that already started. `patched.html` loads Lexxy with both commits, from [idanci/lexxy#2](https://github.com/idanci/lexxy/pull/2).

## Run it

```sh
python3 -m http.server 8000
```

Open both in Chrome, Edge or Safari 18.2+:

- http://localhost:8000/current.html: Lexxy from `basecamp/lexxy` main
- http://localhost:8000/patched.html: Lexxy from the PR

The two pages are the same except for which `lexxy.js` the import map loads. Each loads Lexxy from a render-blocking module, calls `configure()`, and has one editor in its HTML.

## What to look for

1. **The line at the top.** The page writes it from a microtask queued after `configure()`, which is the last moment before the browser may paint:
   - `current.html`: "lexxy-editor was not defined yet when the browser could paint."
   - `patched.html`: "lexxy-editor was defined before the browser could paint."
2. **The editor.** On a fast machine the gap is a frame or less, so slow the CPU down. Open DevTools, then the Performance panel, click the gear icon and set CPU to 6× slowdown. Keep DevTools open and reload a few times:
   - `current.html`: the editor flashes as an empty box, with no toolbar and no "Write something…", before the toolbar appears.
   - `patched.html`: the editor appears whole every time.

## Measured

Recorded from the screen in Chrome 155 on macOS, CPU slowed 6×, 20 reloads of each page:

| Page | Reloads where the empty box was painted |
|---|---|
| `current.html` | 20 of 20, for 1–2 frames each |
| `patched.html` | 0 of 20 |

Recordings are in the PR. They were made with the first commit alone. The second one does nothing on these pages, because they call `configure()` before the elements are defined.

## Files

- `current.html`, `patched.html`: the two pages.
- `current/lexxy.js`: built from `basecamp/lexxy` main at `b384704c`.
- `patched/lexxy.js`: built from `idanci/lexxy` `define-elements-in-microtask-reconfigure` at `81ae9091`.
- `lexxy*.css`: Lexxy's stylesheets, shared by both pages.
- `bin/build`: rebuilds both `lexxy.js` files from source.

Lexxy is MIT licensed; its license is in `LICENSE-lexxy`.
