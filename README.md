# UOB IT PMO — Project Kanban

A single-file Kanban board for tracking IT project tasks across four statuses, built as an
internal **demo and training tool**.

**Live: https://jonbowden.github.io/kanban1/**

> **Not an official UOB system.** This is a training exercise. It uses a plain "UOB IT PMO"
> text wordmark and a generic corporate blue palette — no UOB logo, trademark, or branding,
> and it does not reproduce or connect to any real UOB system. All task data is invented.

![The board on desktop: a deep red header, four large KPI figures (8 tasks, 7 open, 2 blocked, 2 overdue on a solid red tile), a "Work by project" bar chart across six workstreams, and four task columns below](docs/board-desktop.png)

<details>
<summary>On a phone (390px) — columns stack</summary>

![The same board at 390px wide: KPIs reflow to a 2x2 grid, the project chart rows stack, and the four columns stack vertically](docs/board-mobile.png)

</details>

---

## Running it

Open `index.html` — double-click the file, or drag it into a browser. That is the whole
process. There is no server, no build step, no dependency install, and no npm.

The live link above serves the same file over HTTPS.

## What it does

- **KPI strip** — tasks on the board, still open, blocked and overdue, sized to be read at a
  glance. These always report the whole board, never the filtered view.
- **Work by project** — a bar per workstream, segmented by status and carrying that project's
  overdue count. Each row is a button: selecting one filters the board to that project and
  selecting it again clears the filter, so the chart is navigation rather than decoration.
- **Four columns** — Backlog, In Progress, Blocked, Done — side by side on desktop, stacked
  below 768px, each with a live count.
- **Drag and drop** cards between columns using the native HTML5 API, with a drop-target
  highlight on the column you are over.
- **Keyboard equivalent** — every card carries a `Move` control, so the board is fully
  operable without a mouse. Drag-and-drop is mouse-only, so this is the accessible path,
  not a convenience.
- **Cards** show a severity code (P1-P4), task ID, title, project, owner, due date and
  category. Priority reads three ways — the spine colour, the code, and the written word —
  so colour is never the only signal. Overdue tasks (due date passed, not yet Done) are
  flagged. Finished work is deliberately muted: a closed task should not signal risk.
- **Add task** modal with inline validation — no `alert()`, no `confirm()`. Deleting a card
  uses an inline `Delete? Yes / No` row inside the card itself.
- **Filters** by project, owner (case-insensitive contains) and priority, plus a live
  summary strip showing totals per status and the overdue count.
- **Email notification** on new tasks via FormSubmit, sent as a background AJAX call so the
  page never navigates away.
- **IT Support widget** — a floating button, bottom right, opening a panel that links to the
  service desk on WhatsApp with a prefilled message. Keyboard operable: Escape closes it and
  focus returns to the button.

## What the chart does not show

"Work by project" reports **current board state only**, and says so on the chart along with
the sample size. It is deliberately not a flow dashboard: this board records no
state-transition history — moving a card just changes `task.status` — so aging WIP, cycle
time percentiles, throughput, cumulative flow and forecasting cannot be computed from it.
Showing them would mean inventing the numbers, and a wrong metric is worse than a missing
one because people act on it. Adding them would mean recording a timestamp on every status
change, and persisting it.

## Refreshing resets the board — by design

There is no persistence of any kind: no `localStorage`, `sessionStorage`, IndexedDB or
cookies. Board state is a JavaScript array in memory, so **a page refresh restores the eight
seeded demo tasks**. That is intended behaviour for a training tool, and the header says so.

If you add a card and reload, it is gone. Nothing is broken.

## Configuration

Two constants sit at the top of the script.

**IT Support contact** — the WhatsApp widget:

```js
const SUPPORT_WHATSAPP = "6591397490";      // digits only, country code first
const SUPPORT_DISPLAY  = "+65 9139 7490";   // how it is shown on screen
```

Set `SUPPORT_WHATSAPP` to `""` to remove the widget entirely. This repository is public, so
whatever number sits there is readable by anyone and harvestable by scrapers — prefer a
service-desk line over a personal mobile.

**FormSubmit endpoint:**

```js
const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/YOUR_EMAIL@example.com";
```

It currently holds a placeholder, so the Add Task form always reports
`Card added locally — email notification failed`. The board itself works regardless — a
failed notification never removes the card or blocks anything.

To enable email:

1. Replace the address in that constant.
2. Submit one task. FormSubmit requires a **one-time activation**: the first submission to a
   new address triggers a confirmation email, and nothing is delivered until you click the
   link in it.

Because this repository is public, an address placed there is publicly readable. FormSubmit
also issues a random token string that works in place of the address in the endpoint URL —
use that if you would rather not expose an inbox.

## Contributing

The constraints below are requirements of the exercise, not preferences, and nothing in the
toolchain enforces them:

- Vanilla HTML, CSS and JavaScript only — no framework, bundler, or npm.
- Everything stays in `index.html`: one `<style>` block, one `<script>` block.
- No external resources — no CDN, no web fonts, no image files. System font stack, inline
  SVG or Unicode glyphs for icons. Two outbound URLs exist, both navigation rather than
  loaded resources: the FormSubmit endpoint and the `wa.me` support link.
- No storage APIs, no `alert()`/`confirm()`, no `!important`.

`CLAUDE.md` in this repository documents the internal architecture — the `state` object as
single source of truth, the re-render-from-state rule, the `escapeHtml()` requirement, and
how the board is verified with headless Chrome.
