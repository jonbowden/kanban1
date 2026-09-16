# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Kanban board for a fictional internal IT PMO ("UOB IT PMO"), built as a demo/training tool.
The entire app is one file: `index.html` (~1,370 lines). There is no package.json, no git repo,
no dependencies, and no build step.

It is **not** an official UOB system. Keep the neutral `UOB IT PMO` text wordmark and the
corporate blue palette — never add a real UOB logo, trademark, or anything that imitates an
official UOB product.

## Hard constraints

These are requirements of the brief, not preferences. They are easy to violate accidentally
because nothing in the toolchain enforces them:

- **Vanilla HTML/CSS/JS only.** No React, Vue, jQuery, Tailwind, bundler, npm, or any framework.
- **Single file.** All markup, one `<style>` block, one `<script>` block, in `index.html`.
  It must run by double-clicking the file — never introduce anything that needs a server.
- **Zero external resources.** No CDN scripts, no Google Fonts, no image files. System font
  stack, inline SVG or Unicode glyphs for icons. The only outbound URL in the file is the
  FormSubmit endpoint.
- **No persistence of any kind.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies.
  Board state is a plain array in memory; refreshing resets to seed data, which is intended
  behaviour that the header note explains to the user.
- **No `alert()` or `confirm()`.** Validation errors render inline under each field; deleting a
  card uses an inline `Delete? Yes / No` row inside the card.
- **No `!important`** anywhere in the CSS.

Before finishing any change, sweep for regressions:

```bash
grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important' index.html
grep -oE 'https?://[^"'"'"' )]+' index.html | sort -u   # should only ever print the FormSubmit URL
```

## Architecture

The file is organised into numbered banner-comment sections (`1. CONFIG`, `2. STATE`, … through
`13. INIT` in the script; `1. DESIGN TOKENS` … `9. RESPONSIVE` in the style block). Grep for
`^   [0-9]+\. ` to jump between them.

Three invariants hold the design together — breaking any of them is the main way this app rots:

**1. `state` is the single source of truth and the UI is always re-rendered from it.**

```js
const state = { tasks: [], filters: {…}, nextId, pendingDelete };
```

`renderBoard()` rebuilds all four columns wholesale from `applyFilters()`. Nothing outside
`renderBoard()`/`renderCard()` mutates card contents — a mutating action changes `state` and
calls `renderBoard()` again. Only `state.tasks` mutators are `addTask()`, `moveTask()` and
`deleteTask()`. The only DOM writes outside the render path are the toast region and the inline
form errors. Transient per-card UI lives in state too, not in the DOM: `state.pendingDelete`
holds the id of the card currently showing its delete confirmation.

**2. Every user-supplied string passes through `escapeHtml()` before reaching `innerHTML`.**

`renderCard()` builds HTML strings, so this is the only thing standing between a task title and
script injection. If you add a field to a card, escape it.

**3. Card event handlers are delegated from `#board`, never bound to individual cards.**

Because `renderBoard()` replaces the board's innerHTML on every change, per-card listeners would
be destroyed constantly. Cards carry `data-action` / `data-id` attributes; `handleBoardClick()`
and `handleBoardChange()` dispatch on them. Add new card interactions the same way.

### Two ways to move a card, one code path

Drag-and-drop (native HTML5 DnD) and the per-card `Move ▸` `<select>` both funnel into
`moveTask(id, status)`. The select is the keyboard-accessible fallback — DnD is mouse-only, so
it is load-bearing for accessibility, not decoration. Keep both wired to the same function.

### Optimistic add

`handleSubmit()` validates, then calls `addTask()` so the card is on the board **before** the
network call starts, then fires `notifyNewTask()` in parallel. A FormSubmit failure must never
remove the card or break the board — it only raises the warning toast
`Card added locally — email notification failed`. Note this differs from what a persistent
backend would need (a rejected write would have to roll back); see the persistence note below.

## FormSubmit

`index.html:664` is the single config constant:

```js
const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/YOUR_EMAIL@example.com";
```

Currently a placeholder. FormSubmit requires a **one-time activation per address**: the first
submission to a new address triggers a confirmation email, and nothing is delivered until the
link in it is clicked. The board works regardless — an unactivated address just yields the
warning toast.

`notifyNewTask()` is specified verbatim by the brief (`_subject`, `_template: "table"`,
`_captcha: "false"`, the nine field keys). Don't restructure its payload without being asked.
Note it only rejects on a non-2xx status — FormSubmit answers a bad address with a 2xx, so a
"success" toast does not prove an email was sent.

**Never send the user's email address anywhere except this endpoint.** When testing the submit
path, stub `window.fetch` rather than hitting the live service.

## Testing

There is no test framework. Verification is done by driving the file in headless Chrome. On this
WSL setup the Windows Chrome binary is used, and paths must be Windows-style `file:///C:/...`:

```bash
CHROME="/mnt/c/Program Files/Google/Chrome/Application/chrome.exe"

# Render and inspect the resulting DOM
"$CHROME" --headless --disable-gpu --virtual-time-budget=4000 \
  --dump-dom "file:///C:/Users/jon/Projects/kanban/index.html" > /tmp/dom.html

# Screenshot (writes to a Windows path)
"$CHROME" --headless --disable-gpu --hide-scrollbars --force-device-scale-factor=1 \
  --window-size=1440,1100 --virtual-time-budget=4000 \
  --screenshot='C:\Users\jon\Projects\kanban\.shot.png' \
  "file:///C:/Users/jon/Projects/kanban/index.html"
```

To test behaviour, copy `index.html` to a temp file **inside the project directory** (Windows
Chrome cannot reach WSL `/tmp`), append a `<script>` harness before `</body>` that calls the
app's functions and writes results into a `<pre>`, run `--dump-dom`, extract the `<pre>`, then
delete the temp file. The app's functions are all globals, so a harness can call `moveTask()`,
`validateForm()`, `addTask()` etc. directly.

Gotchas learned the hard way:

- Write the harness's own closing tag as a literal `</script>`; only escape it as `<\/script>`
  when it appears *inside* a JS string. Getting this backwards silently kills the whole block.
- Wrap the harness body in `try/catch` and write results in a `finally`, or one thrown assertion
  loses the entire report.
- Headless Chrome clamps the window to roughly a 485px viewport here, so `--window-size=400`
  produces a *cropped screenshot of a wider page*, which looks exactly like a horizontal-overflow
  bug but isn't. To genuinely test narrow layouts, load `index.html` in an `<iframe width="360">`
  (with `--allow-file-access-from-files`) and measure `getBoundingClientRect()` inside it.
- Syntax-check the script block without a browser:
  `sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' > /tmp/app.js && node --check /tmp/app.js`

Clean up any `.shot-*.png` / temp harness files from the project directory — `index.html` should
be the only tracked file besides this one.

## Making the board persistent

If asked to add persistence, note that the current failure semantics are deliberately wrong for
a backend: today a failed notification *keeps* the card by design. With a real API, a rejected
write would need to roll back instead, and ID generation must move server-side so two users
can't both mint `UOB-ITPM-0009`. `state` being the single source of truth means the rest of the
change is small.
