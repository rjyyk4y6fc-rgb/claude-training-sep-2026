# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`index.html` is a single-file demo web app: a Kanban board for a fictional "IT PMO"
(internal IT project management office). It is a training/demo artifact, not a production
system — it deliberately avoids any real-company branding or trademarks (text wordmark
plus a Korean subheading, generic corporate-blue-and-rainbow palette).

## Running and developing

There is no build step, package manager, or server. The entire app — HTML, `<style>`, and
`<script>` — lives in `index.html`.

- **Run it**: open `index.html` directly in a browser (double-click, or `open index.html`
  on macOS). No `npm install`, no dev server.
- **No test suite / linter is configured.** Verify changes manually in a browser: reload
  the page and exercise the feature (drag a card, add a task, apply a filter, resize below
  768px to check the mobile stacked layout).
- Quick static sanity checks (brace/paren/backtick balance, no accidental storage-API
  usage) can be done with `grep`/`python3` one-liners since there's no linter — see prior
  session history for examples, but a full manual browser pass is the real verification.

## Hard constraints (do not violate when editing)

These were explicit requirements for this app and should be preserved in any future edits:

- **Vanilla only**: no React/Vue/jQuery/Tailwind, no bundler, no npm dependencies, no
  external CDN scripts or fonts. System font stack and inline SVG/Unicode glyphs only.
- **Single file**: all markup, styles, and script stay in `index.html`. Don't split into
  separate `.css`/`.js` files — it must keep working by double-clicking the file with no
  server.
- **No persistence**: board state lives only in the in-memory `state` object in the
  `<script>` block. Never introduce `localStorage`, `sessionStorage`, `IndexedDB`,
  cookies, or any other storage API — a page refresh is *supposed* to reset the board to
  seed data, and the UI has a note saying so.
- **FormSubmit is the only backend call**: new-task notifications POST to the FormSubmit
  AJAX JSON endpoint (`FORMSUBMIT_ENDPOINT` constant near the top of the script). This is
  best-effort and optimistic — a failed/unconfirmed FormSubmit call must never remove the
  card or block the UI (see the try/catch in `notifyNewTask()`). FormSubmit requires a
  one-time confirmation-email click for a new address before notifications actually
  deliver; a "silent" failure during testing is expected, not a bug.

## Architecture

Everything is driven by one `state` object (`{ tasks, filters, deleteConfirmIds,
nextIdCounter }`) — there is no other source of truth. All rendering flows one way:
mutate `state` → call `renderBoard()` → DOM is fully regenerated from `state`. Card
markup is only ever produced inside `renderCard()`/`renderBoard()`; nothing else patches
card DOM directly.

Key functions in `index.html`'s `<script>` block:

- `seedTasks()` / `generateTaskId()` — builds the 8 demo tasks on load and issues
  sequential `ITPM-####` IDs.
- `applyFilters(tasks)` — pure filter over the in-memory array (project/assignee/priority).
- `renderBoard()` → `renderCard(task)` — the only path that writes card HTML; buckets
  filtered tasks into the 4 fixed status columns and updates counts/summary strip.
- `addTask(fields)` / `moveTask(id, status)` / `deleteTask(id)` — the only mutators of
  `state.tasks`; each ends by calling `renderBoard()`.
- `validateForm(fields)` — client-side validation for the Add Task form (no `alert()`;
  errors render inline per field).
- `notifyNewTask(task)` — the FormSubmit call, isolated so a network failure only shows a
  warning toast (`showToast`) and never touches `state.tasks`.
- Drag-and-drop and the keyboard-accessible "Move ▸" select both funnel into the same
  `moveTask()` — don't add a second, divergent code path for moving a card.
- Event handling on the board uses delegation (`boardEl.addEventListener(...)` for click/
  change/dragstart/dragend) so re-rendering card markup never requires re-binding
  per-card listeners.
- `escapeHtml()` must wrap every piece of user-supplied text before it's interpolated
  into template-literal HTML (titles, descriptions, assignee names, etc.).

Status values (`Backlog`, `In Progress`, `Blocked`, `Done`) are used both as display
strings and as `data-status` attributes; DOM element IDs can't contain the space in
`In Progress`, so `statusSlug()` converts it to `In-Progress` for `id` lookups — reuse
that helper rather than hardcoding a second slugging scheme.
