# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo/training IT PMO Kanban board for a **fictitious** bank, built as one self-contained file: `index.html` (markup + `<style>` + `<script>`). There is no build, package manager, test suite or linter.

## Running / checking

- Run: open `index.html` directly in a browser (double-click, or `Start-Process index.html`). No server needed.
- JS syntax check (no test framework exists):
  ```bash
  sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' > "$TEMP/pmo_check.js" && node --check "$TEMP/pmo_check.js"
  ```
- Constraint check — should print nothing except the FormSubmit URL:
  ```bash
  grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important' index.html
  grep -noE 'https?://[^" ]+' index.html
  ```

## Hard constraints (from the original brief — keep them)

- Vanilla HTML/CSS/JS only; no frameworks, CDNs, web fonts, image files, or build step. Icons are inline SVG / Unicode.
- Stay a single file.
- **No persistence**: no localStorage/sessionStorage/IndexedDB/cookies. Refresh resetting to seed data is intended (the header note says so).
- No `alert()`/`confirm()`; errors are inline, delete confirmation is an inline "Delete? Yes / No" on the card.
- No `!important`; colours and spacing come from CSS custom properties in `:root`.
- Branding: neutral "IT PMO" text wordmark, corporate blue palette, no real bank logos. The `UOB-ITPM-####` task ID format and `[UOB IT PMO]` email subject are kept as specified by the user.
- The only network call is FormSubmit; nothing else may be sent anywhere.

## Script architecture

The `<script>` is split into numbered, commented sections (config → state → helpers → seed → rendering → mutations → board events → filters → form → FormSubmit → toasts → init). Key invariants:

- **Single source of truth**: `state = { tasks, filters, pendingDelete, openMoveMenu, focusAfter }`. Transient UI (open Move menu, pending delete confirm) lives in state too, so it survives re-renders.
- **Render from state only**: `renderBoard()` is the only code that writes card DOM; it rebuilds every column's list via `renderCard()` (returns an HTML string) and updates column badges and the header summary. Mutations (`addTask`, `moveTask`, `deleteTask`) change state, then call `renderBoard()`.
- **Escaping**: every user-supplied value inserted into HTML strings goes through `escapeHtml()`, including attribute values. Toasts and error messages use `textContent`.
- **Focus restoration**: because cards are re-rendered, keyboard focus is kept by setting `state.focusAfter = { id, action }` before rendering; `restoreFocus()` refocuses `[data-id][data-action]` afterwards (or the board if the card is gone).
- **Event delegation**: one `click`/`keydown` handler on `#board` dispatches on `data-action` (`move-toggle`, `move-to`, `delete-ask`, `delete-yes`, `delete-no`). Drag-and-drop (native HTML5) is also delegated on the board; `.is-drop-target` gives the column highlight.
- **Column badges show filtered counts; the header summary shows totals over all tasks.** Cards within a column are sorted by priority, then due date.
- **Dates** are local `YYYY-MM-DD` strings compared lexically (`todayISO()`); overdue = due < today and status ≠ Done. Seed due dates are relative to today, so the board always has overdue examples.
- **IDs** come from `nextId()` using a module-level counter (seeds take 0001–0008).

## FormSubmit integration

- `FORMSUBMIT_ENDPOINT` at the top of the script is the one place to set the recipient email (currently the `YOUR_EMAIL@example.com` placeholder, so sends fail by design until changed).
- FormSubmit requires one-time activation: the first submission emails a confirmation link to that address; nothing delivers until it is clicked.
- Submission is optimistic: the card is added and rendered first, then `notifyNewTask()` runs with the submit button disabled showing "Sending…". Any failure (non-2xx, `success: "false"` body, 15s timeout) is caught and shown as a warning toast; it must never affect the board.
- From `file://` the request origin is `null`, which FormSubmit may reject; serve over HTTP if real delivery needs testing.
