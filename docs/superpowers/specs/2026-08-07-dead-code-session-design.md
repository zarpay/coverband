# Dead-Code Review Session — Design Spec

Date: 2026-08-07
Repo: zarpay/coverband (fork of danmayer/coverband)

## Purpose

Facilitate dead-code removal from the Coverband web report. A reviewer starts a
"dead-code session" in the browser and progressively narrows the file list by
hiding uninteresting files/folders, and marks files/folders for deletion,
exporting the marked list as a CSV for an out-of-band code-removal PR process.

## Scope / Non-goals

- Entirely client-side: `public/application.js` + `public/application.css`.
  No Ruby changes, no new routes, no server-side storage.
- State is per-browser (localStorage). No sharing across machines/teammates.
- No actual code deletion is performed by Coverband; export only.

## Architecture

The Coverband web UI is a Rack app rendering an ERB file table which is
enhanced by DataTables 1.7 (jQuery). Rows arrive two ways: rendered server-side
in `views/file_list.erb`, or (paged mode) appended client-side from
`/report_json` via `fnAddData`. Both end in the same DataTable, so all session
logic hooks the table client-side:

1. **State module** (plain JS, pure functions where possible):
   localStorage key `coverband_deadcode_session:<base_path>` holding
   ```json
   {
     "hiddenPaths": ["app/models/", "app/foo.rb"],
     "markedPaths": [{"path": "lib/old/", "comment": "unused since v2", "markedAt": "ISO8601"}],
     "hideRuntimeUsed": false,
     "startedAt": "ISO8601"
   }
   ```
   Entries ending in `/` are folder prefixes (hide/mark everything under them);
   others are exact file paths. Full path for a row is read from the file
   link's `title` attribute.

2. **Row filter**: one function pushed onto `$.fn.dataTableExt.afnFiltering`
   (supported by bundled DataTables 1.7). A row is filtered out when:
   - its path matches any `hiddenPaths` entry (exact or prefix), or
   - its path matches any `markedPaths` entry, or
   - `hideRuntimeUsed` is on and the row's runtime % (column 2) > 0.
   State changes call `fnDraw()`.

3. **Row decoration**: on table draw, decorate column 1's path display.
   Each directory segment and the filename get, on row hover, two small
   inline icons (VS Code find-results style):
   - ✕ = hide (segment ✕ hides the folder prefix; filename ✕ hides the file)
   - 🗑 = mark for deletion (opens confirmation popup)
   Icon clicks `preventDefault`/`stopPropagation` so they never trigger the
   source-file popup link. Path text content is unchanged so DataTables
   search/sort keep working.

4. **Mark confirmation popup**: small anchored popup showing the path, an
   optional comment textarea, and Mark for deletion / Cancel. Confirming adds
   the entry to `markedPaths` and redraws (rows disappear).

5. **Dead-code toolbar** (JS-injected above the file table):
   - Checkbox: "Hide files with runtime % > 0" (rule-based; applies to data
     loaded later in paged mode too)
   - Live badge: "N hidden" (count of rows filtered out by hides/rule)
   - Button: "Marked for deletion: N" → opens the marked modal
   - Disclosure: chips for each `hiddenPaths` entry, each with ✕ to unhide
   - Button: "Reset session" — clears hides + runtime toggle ONLY
     (marks survive; they are a separate worklist)

6. **Marked modal**: large in-page modal (plain JS/CSS; colorbox not reused).
   Table of marked entries: path, inline-editable comment, marked-at,
   per-row unmark ✕. Footer:
   - **Download CSV**: columns `path,comment,marked_at`, RFC-4180 quoting,
     Blob download named `coverband-dead-code-YYYY-MM-DD.csv`
   - **Start over**: clears `markedPaths` after a confirm

## Error handling

- localStorage unavailable/corrupt JSON → start with empty state, log to
  console; the report continues to work as stock Coverband.
- Paths containing quotes/commas/newlines are CSV-escaped.
- Multiple groups/tables on the page: filter applies per-table; decoration
  applied to all `.file_list` tables.

## Testing

- Pure logic (prefix matching, state (de)serialization, CSV escaping)
  extracted into standalone functions on a `CoverbandDeadCode` namespace,
  testable in isolation.
- No JS test rig exists in the gem; manual verification against
  `test/rails8_dummy` app, both paged and non-paged reporting modes.
- Existing Ruby test suite must stay green (no Ruby files touched).

## Decisions log

- localStorage over server-side store (personal workflow, zero backend risk).
- Client-side decoration over server-rendered segments (single code path
  covers both ERB and JSON row sources).
- Reset session ≠ Start over: viewing filters and deletion worklist have
  independent lifecycles.
