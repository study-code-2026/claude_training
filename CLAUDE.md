# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo/training IT PMO Kanban board for a **fictitious** bank. There are two self-contained pages (each markup + `<style>` + `<script>`) with identical behaviour:

- `index.html`: the **original design**. The user wants it kept as is; change it only when asked.
- `redesign.html`: the **redesign** (ink-navy header with a proportional status ribbon, Bahnschrift display type, flat lanes, owner initials, slide-in Add task drawer, filters collapsible on mobile).

The two scripts are separate copies, so a behaviour fix usually has to be made in both files. There is no build, package manager, test suite or linter.

## Running / checking

- Run: open `index.html` or `redesign.html` directly in a browser. No server needed. (The Playwright MCP browser blocks `file:` URLs; for it, serve with `python -m http.server 8765 --bind 127.0.0.1`.)
- Live (GitHub Pages, from `main`): https://study-code-2026.github.io/claude_training/ and `/redesign.html`.
- JS syntax check (no test framework exists):
  ```bash
  for f in index.html redesign.html; do sed -n '/<script>/,/<\/script>/p' $f | sed '1d;$d' > "$TEMP/c.js" && node --check "$TEMP/c.js" && echo "$f OK"; done
  ```
- Constraint check — should print nothing except the FormSubmit URL:
  ```bash
  grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important' index.html redesign.html
  grep -noE 'https?://[^" ]+' index.html redesign.html
  ```

## Hard constraints (from the original brief — keep them)

- Vanilla HTML/CSS/JS only; no frameworks, CDNs, web fonts, image files, or build step. Icons are inline SVG / Unicode.
- Each page stays a single self-contained file.
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

- `FORMSUBMIT_ENDPOINT` at the top of each page's script is the one place per page to set the recipient email (currently the `YOUR_EMAIL@example.com` placeholder, so sends fail by design until changed).
- FormSubmit requires one-time activation: the first submission emails a confirmation link to that address; nothing delivers until it is clicked.
- Submission is optimistic: the card is added and rendered first, then `notifyNewTask()` runs with the submit button disabled showing "Sending…". Any failure (non-2xx, `success: "false"` body, 15s timeout) is caught and shown as a warning toast; it must never affect the board.
- From `file://` the request origin is `null`, which FormSubmit may reject; serve over HTTP if real delivery needs testing.
