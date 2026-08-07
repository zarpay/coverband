# Dead-Code Review Session Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a client-side "dead-code session" to Coverband's web report: hide files/folders (manually or by runtime % > 0 rule), mark files/folders for deletion with comments, and export the marked list as CSV.

**Architecture:** Entirely client-side. A new pure-logic module `public/dead_code_session.js` (state, path matching, CSV) is exercised by Node tests and consumed by UI wiring added to `public/application.js` (DataTables 1.7 custom filter, row decoration, toolbar, modal). State lives in localStorage. No Ruby logic changes; only `views/layout.erb` gains a script tag.

**Tech Stack:** jQuery 1.x + DataTables 1.7 (bundled in `public/dependencies.js`), plain JS/CSS, `node --test` for pure-logic tests.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-08-07-dead-code-session-design.md`
- No Ruby behavior changes; existing Ruby test suite stays green.
- No new runtime dependencies; no build step. JS must run in old-jQuery world (ES5-friendly in `application.js`; `dead_code_session.js` may use ES5 + `Array.prototype` methods only).
- localStorage key: `coverband_deadcode_session:<coverageurl>`.
- Folder entries end with `/` and match by prefix; file entries match exactly.
- "Reset session" clears hides + runtime toggle only. "Start over" (modal) clears marks only.
- CSV columns: `path,comment,marked_at`, RFC-4180 quoting, CRLF line endings.
- Work on branch `feature/dead-code-session` in `/Users/obie/projects/coverband`.

---

### Task 1: Pure logic module `dead_code_session.js` + Node tests

**Files:**
- Create: `public/dead_code_session.js`
- Test: `test/javascript/dead_code_session.test.mjs`

**Interfaces:**
- Produces global `CoverbandDeadCode` (browser) / module export (Node) with:
  - `emptyState() -> state` — `{hiddenPaths: string[], markedPaths: {path,comment,markedAt}[], hideRuntimeUsed: bool, startedAt: string|null}`
  - `parseState(json|null) -> state` (never throws; invalid → empty)
  - `pathMatches(entry, path) -> bool`
  - `isExcluded(state, path, runtimePercent|null) -> bool`
  - `addHidden(state, entry)`, `removeHidden(state, entry)`, `clearHides(state)` (also resets `hideRuntimeUsed`)
  - `addMark(state, path, comment, markedAt)` (replaces existing mark for same path), `removeMark(state, path)`, `updateComment(state, path, comment)`, `clearMarks(state)`
  - `toCSV(markedPaths) -> string`
  - `segmentPrefixes(path) -> [{label, prefix, isFile}]` e.g. `"app/models/foo.rb"` → `[{label:"app", prefix:"app/", isFile:false}, {label:"models", prefix:"app/models/", isFile:false}, {label:"foo.rb", prefix:"app/models/foo.rb", isFile:true}]`

- [ ] **Step 1: Write the failing tests**

```js
// test/javascript/dead_code_session.test.mjs
import { test } from "node:test";
import assert from "node:assert/strict";
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const D = require("../../public/dead_code_session.js");

test("parseState returns empty state on garbage", () => {
  assert.deepEqual(D.parseState(null), D.emptyState());
  assert.deepEqual(D.parseState("{not json"), D.emptyState());
  assert.deepEqual(D.parseState('{"hiddenPaths": "nope"}').hiddenPaths, []);
});

test("parseState keeps valid entries and normalizes marks", () => {
  const s = D.parseState(JSON.stringify({
    hiddenPaths: ["app/", 5, "lib/foo.rb"],
    markedPaths: [{ path: "old/" }, { nope: true }],
    hideRuntimeUsed: true
  }));
  assert.deepEqual(s.hiddenPaths, ["app/", "lib/foo.rb"]);
  assert.deepEqual(s.markedPaths, [{ path: "old/", comment: "", markedAt: "" }]);
  assert.equal(s.hideRuntimeUsed, true);
});

test("pathMatches: folder prefix vs exact file", () => {
  assert.ok(D.pathMatches("app/models/", "app/models/foo.rb"));
  assert.ok(!D.pathMatches("app/models/", "app/models2/foo.rb"));
  assert.ok(D.pathMatches("app/foo.rb", "app/foo.rb"));
  assert.ok(!D.pathMatches("app/foo.rb", "app/foo.rb.bak"));
});

test("isExcluded combines hides, marks, and runtime rule", () => {
  const s = D.emptyState();
  D.addHidden(s, "app/hidden/");
  D.addMark(s, "lib/dead.rb", "bye", "2026-08-07T00:00:00Z");
  assert.ok(D.isExcluded(s, "app/hidden/x.rb", 0));
  assert.ok(D.isExcluded(s, "lib/dead.rb", 0));
  assert.ok(!D.isExcluded(s, "app/live.rb", 12.5));
  s.hideRuntimeUsed = true;
  assert.ok(D.isExcluded(s, "app/live.rb", 12.5));
  assert.ok(!D.isExcluded(s, "app/unused.rb", 0));
  assert.ok(!D.isExcluded(s, "app/unknown.rb", null));
});

test("addHidden dedupes; removeHidden removes; clearHides resets toggle", () => {
  const s = D.emptyState();
  D.addHidden(s, "app/");
  D.addHidden(s, "app/");
  assert.deepEqual(s.hiddenPaths, ["app/"]);
  D.removeHidden(s, "app/");
  assert.deepEqual(s.hiddenPaths, []);
  s.hideRuntimeUsed = true;
  D.addHidden(s, "x.rb");
  D.clearHides(s);
  assert.deepEqual(s.hiddenPaths, []);
  assert.equal(s.hideRuntimeUsed, false);
});

test("addMark replaces same path; updateComment; removeMark; clearMarks", () => {
  const s = D.emptyState();
  D.addMark(s, "a.rb", "one", "t1");
  D.addMark(s, "a.rb", "two", "t2");
  assert.deepEqual(s.markedPaths, [{ path: "a.rb", comment: "two", markedAt: "t2" }]);
  D.updateComment(s, "a.rb", "three");
  assert.equal(s.markedPaths[0].comment, "three");
  D.removeMark(s, "a.rb");
  assert.deepEqual(s.markedPaths, []);
  D.addMark(s, "b.rb", "", "t");
  D.clearMarks(s);
  assert.deepEqual(s.markedPaths, []);
});

test("toCSV escapes per RFC 4180", () => {
  const csv = D.toCSV([
    { path: "app/a.rb", comment: 'has "quotes", commas\nand newline', markedAt: "2026-08-07T01:02:03Z" },
    { path: "lib/", comment: "", markedAt: "" }
  ]);
  assert.equal(csv,
    'path,comment,marked_at\r\n' +
    'app/a.rb,"has ""quotes"", commas\nand newline",2026-08-07T01:02:03Z\r\n' +
    'lib/,,\r\n');
});

test("segmentPrefixes splits path into cumulative prefixes", () => {
  assert.deepEqual(D.segmentPrefixes("app/models/foo.rb"), [
    { label: "app", prefix: "app/", isFile: false },
    { label: "models", prefix: "app/models/", isFile: false },
    { label: "foo.rb", prefix: "app/models/foo.rb", isFile: true }
  ]);
  assert.deepEqual(D.segmentPrefixes("foo.rb"), [{ label: "foo.rb", prefix: "foo.rb", isFile: true }]);
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/obie/projects/coverband && node --test test/javascript/`
Expected: FAIL (cannot find module `../../public/dead_code_session.js`)

- [ ] **Step 3: Write the implementation**

```js
// public/dead_code_session.js
// Pure logic for the dead-code review session. Loaded in the browser as
// window.CoverbandDeadCode and in Node (tests) via require.
(function (root, factory) {
  if (typeof module === "object" && module.exports) {
    module.exports = factory();
  } else {
    root.CoverbandDeadCode = factory();
  }
})(typeof self !== "undefined" ? self : this, function () {
  function emptyState() {
    return { hiddenPaths: [], markedPaths: [], hideRuntimeUsed: false, startedAt: null };
  }

  function parseState(json) {
    var raw;
    try { raw = JSON.parse(json); } catch (e) { raw = null; }
    var state = emptyState();
    if (!raw || typeof raw !== "object") return state;
    if (Object.prototype.toString.call(raw.hiddenPaths) === "[object Array]") {
      state.hiddenPaths = raw.hiddenPaths.filter(function (p) {
        return typeof p === "string" && p.length > 0;
      });
    }
    if (Object.prototype.toString.call(raw.markedPaths) === "[object Array]") {
      state.markedPaths = raw.markedPaths.filter(function (m) {
        return m && typeof m.path === "string" && m.path.length > 0;
      }).map(function (m) {
        return {
          path: m.path,
          comment: typeof m.comment === "string" ? m.comment : "",
          markedAt: typeof m.markedAt === "string" ? m.markedAt : ""
        };
      });
    }
    state.hideRuntimeUsed = raw.hideRuntimeUsed === true;
    state.startedAt = typeof raw.startedAt === "string" ? raw.startedAt : null;
    return state;
  }

  function pathMatches(entry, path) {
    if (entry.charAt(entry.length - 1) === "/") return path.indexOf(entry) === 0;
    return path === entry;
  }

  function matchesAny(entries, path) {
    for (var i = 0; i < entries.length; i++) {
      if (pathMatches(entries[i], path)) return true;
    }
    return false;
  }

  function isMarked(state, path) {
    for (var i = 0; i < state.markedPaths.length; i++) {
      if (pathMatches(state.markedPaths[i].path, path)) return true;
    }
    return false;
  }

  function isExcluded(state, path, runtimePercent) {
    if (matchesAny(state.hiddenPaths, path)) return true;
    if (isMarked(state, path)) return true;
    if (state.hideRuntimeUsed && typeof runtimePercent === "number" && runtimePercent > 0) return true;
    return false;
  }

  function addHidden(state, entry) {
    if (state.hiddenPaths.indexOf(entry) === -1) state.hiddenPaths.push(entry);
  }

  function removeHidden(state, entry) {
    var i = state.hiddenPaths.indexOf(entry);
    if (i !== -1) state.hiddenPaths.splice(i, 1);
  }

  function clearHides(state) {
    state.hiddenPaths = [];
    state.hideRuntimeUsed = false;
  }

  function addMark(state, path, comment, markedAt) {
    removeMark(state, path);
    state.markedPaths.push({ path: path, comment: comment || "", markedAt: markedAt || "" });
  }

  function removeMark(state, path) {
    for (var i = state.markedPaths.length - 1; i >= 0; i--) {
      if (state.markedPaths[i].path === path) state.markedPaths.splice(i, 1);
    }
  }

  function updateComment(state, path, comment) {
    for (var i = 0; i < state.markedPaths.length; i++) {
      if (state.markedPaths[i].path === path) state.markedPaths[i].comment = comment;
    }
  }

  function clearMarks(state) {
    state.markedPaths = [];
  }

  function csvField(value) {
    var s = String(value == null ? "" : value);
    if (/[",\n\r]/.test(s)) return '"' + s.replace(/"/g, '""') + '"';
    return s;
  }

  function toCSV(markedPaths) {
    var lines = ["path,comment,marked_at"];
    for (var i = 0; i < markedPaths.length; i++) {
      var m = markedPaths[i];
      lines.push([csvField(m.path), csvField(m.comment), csvField(m.markedAt)].join(","));
    }
    return lines.join("\r\n") + "\r\n";
  }

  function segmentPrefixes(path) {
    var parts = path.split("/");
    var out = [];
    var prefix = "";
    for (var i = 0; i < parts.length; i++) {
      var isFile = i === parts.length - 1;
      prefix += parts[i] + (isFile ? "" : "/");
      out.push({ label: parts[i], prefix: prefix, isFile: isFile });
    }
    return out;
  }

  return {
    emptyState: emptyState,
    parseState: parseState,
    pathMatches: pathMatches,
    isExcluded: isExcluded,
    addHidden: addHidden,
    removeHidden: removeHidden,
    clearHides: clearHides,
    addMark: addMark,
    removeMark: removeMark,
    updateComment: updateComment,
    clearMarks: clearMarks,
    toCSV: toCSV,
    segmentPrefixes: segmentPrefixes
  };
});
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/obie/projects/coverband && node --test test/javascript/`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add public/dead_code_session.js test/javascript/dead_code_session.test.mjs
git commit -m "feat: dead-code session pure logic module with node tests"
```

---

### Task 2: Load module, session state wiring, DataTables filter, toolbar

**Files:**
- Modify: `views/layout.erb` (add script tag, line 6-7 area)
- Modify: `public/application.js` (append dead-code session section; hook `tableOptions`)
- Modify: `public/application.css` (append `.dcs-*` styles)

**Interfaces:**
- Consumes: `CoverbandDeadCode` from Task 1.
- Produces (used by Tasks 3-4, all inside the `initDeadCodeSession` closure in `application.js`):
  - `state` — the live session state object
  - `saveState()` — persists to localStorage
  - `redrawTables()` — calls `fnDraw()` on every `.file_list` DataTable and then `updateBadges()`
  - `updateBadges()` — recomputes hidden/marked counts into `#dcs-hidden-count` / `#dcs-marked-count`, and rebuilds the hidden-chips list `#dcs-chips`
  - `extractPath(cellHtml)` — full path from a file cell's `title` attribute
  - Toolbar DOM: `#dcs-toolbar` with `#dcs-runtime-toggle` (checkbox), `#dcs-hidden-count`, `#dcs-marked-btn` (with `#dcs-marked-count`), `#dcs-chips-toggle`, `#dcs-chips`, `#dcs-reset`

- [ ] **Step 1: Add script tag to layout**

In `views/layout.erb`, before the `application.js` line, add:

```erb
    <script src='<%= assets_path('dead_code_session.js') %>' type='text/javascript'></script>
```

- [ ] **Step 2: Append the session section to `application.js`**

Add inside the existing `$(document).ready(function () { ... })` — insert a call `initDeadCodeSession();` immediately after `$(".file_list").dataTable(tableOptions);`, and add to `tableOptions` a draw callback (before the `$(".file_list").dataTable(tableOptions);` line):

```js
  tableOptions.fnDrawCallback = function (oSettings) {
    if (window.deadCodeOnDraw) window.deadCodeOnDraw(oSettings);
  };
```

Then append at the bottom of the ready handler (before the closing `});`):

```js
  function initDeadCodeSession() {
    if (!window.CoverbandDeadCode || !$(".file_list").length) return;
    var D = window.CoverbandDeadCode;
    var storageKey = "coverband_deadcode_session:" + ($(".file_list").data("coverageurl") || "/");
    var state;
    try {
      state = D.parseState(window.localStorage.getItem(storageKey));
    } catch (e) {
      console && console.warn && console.warn("coverband dead-code session: localStorage unavailable", e);
      state = D.emptyState();
    }
    if (!state.startedAt) state.startedAt = new Date().toISOString();

    function saveState() {
      try {
        window.localStorage.setItem(storageKey, JSON.stringify(state));
      } catch (e) { /* private mode etc.; session still works in-memory */ }
    }
    saveState();

    function extractPath(cellHtml) {
      var m = String(cellHtml).match(/title="([^"]*)"/);
      return m ? m[1] : "";
    }

    function rowRuntime(aData) {
      var n = parseFloat(String(aData[2]));
      return isNaN(n) ? null : n;
    }

    // Custom filter: hide rows excluded by the session (DataTables 1.7 API)
    $.fn.dataTableExt.afnFiltering.push(function (oSettings, aData, iDataIndex) {
      if (!$(oSettings.nTable).hasClass("file_list")) return true;
      return !D.isExcluded(state, extractPath(aData[0]), rowRuntime(aData));
    });

    function redrawTables() {
      $(".file_list").each(function () {
        $(this).dataTable().fnDraw();
      });
      updateBadges();
    }

    function updateBadges() {
      var hidden = 0;
      $(".file_list").each(function () {
        var data = $(this).dataTable().fnGetData();
        for (var i = 0; i < data.length; i++) {
          if (D.isExcluded(state, extractPath(data[i][0]), rowRuntime(data[i]))) hidden++;
        }
      });
      $("#dcs-hidden-count").text(hidden + " hidden");
      $("#dcs-marked-count").text(state.markedPaths.length);
      var chips = $("#dcs-chips").empty();
      $.each(state.hiddenPaths, function (_i, entry) {
        var chip = $('<span class="dcs-chip"></span>').text(entry);
        $('<button class="dcs-chip-x" title="Unhide">&times;</button>')
          .appendTo(chip)
          .on("click", function () {
            D.removeHidden(state, entry);
            saveState();
            redrawTables();
          });
        chips.append(chip);
      });
      chips.toggle(state.hiddenPaths.length > 0 && chips.data("open") === true);
      $("#dcs-chips-toggle").toggle(state.hiddenPaths.length > 0)
        .text("Hidden items (" + state.hiddenPaths.length + ")");
    }

    // Toolbar
    var toolbar = $(
      '<div id="dcs-toolbar">' +
      '  <strong>Dead-code session</strong>' +
      '  <label><input type="checkbox" id="dcs-runtime-toggle"> Hide files with runtime % &gt; 0</label>' +
      '  <span class="dcs-badge" id="dcs-hidden-count">0 hidden</span>' +
      '  <button type="button" id="dcs-marked-btn">Marked for deletion: <span id="dcs-marked-count">0</span></button>' +
      '  <button type="button" id="dcs-chips-toggle">Hidden items</button>' +
      '  <button type="button" id="dcs-reset">Reset session</button>' +
      '  <div id="dcs-chips"></div>' +
      '</div>'
    );
    $("#content").prepend(toolbar);

    $("#dcs-runtime-toggle")
      .prop("checked", state.hideRuntimeUsed)
      .on("change", function () {
        state.hideRuntimeUsed = this.checked;
        saveState();
        redrawTables();
      });

    $("#dcs-chips-toggle").on("click", function () {
      var chips = $("#dcs-chips");
      chips.data("open", chips.data("open") !== true);
      updateBadges();
    });

    $("#dcs-reset").on("click", function () {
      if (!confirm("Reset session? This unhides everything (marked-for-deletion list is kept).")) return;
      window.CoverbandDeadCode.clearHides(state);
      $("#dcs-runtime-toggle").prop("checked", false);
      saveState();
      redrawTables();
    });

    // Exposed for Tasks 3-4 and the shared fnDrawCallback
    window.deadCodeSession = {
      D: D, state: state, saveState: saveState,
      redrawTables: redrawTables, updateBadges: updateBadges,
      extractPath: extractPath
    };
    window.deadCodeOnDraw = function (oSettings) {
      if (window.deadCodeDecorate) window.deadCodeDecorate(oSettings);
      updateBadges();
    };

    redrawTables();
  }
```

- [ ] **Step 3: Append toolbar styles to `application.css`**

```css
/* Dead-code review session */
#dcs-toolbar { margin: 8px 0 12px 0; padding: 8px 10px; background: #f4f4f4; border: 1px solid #ddd; border-radius: 4px; font-size: 0.85em; }
#dcs-toolbar label { margin: 0 10px; cursor: pointer; }
#dcs-toolbar button { margin: 0 4px; cursor: pointer; }
.dcs-badge { background: #666; color: #fff; border-radius: 8px; padding: 1px 8px; margin: 0 6px; }
#dcs-chips { margin-top: 6px; display: none; }
.dcs-chip { display: inline-block; background: #e8e8e8; border: 1px solid #ccc; border-radius: 10px; padding: 1px 6px; margin: 2px; font-family: monospace; }
.dcs-chip-x { border: none; background: transparent; cursor: pointer; font-weight: bold; margin-left: 4px; }
```

- [ ] **Step 4: Verify no syntax errors**

Run: `node --check public/application.js && node --check public/dead_code_session.js`
Expected: both exit 0

- [ ] **Step 5: Manual smoke check (see Task 5 harness)** — defer full check; just confirm Ruby suite untouched:

Run: `cd /Users/obie/projects/coverband && bundle exec rake test 2>&1 | tail -5` (skip if bundle not installed; nothing Ruby-facing changed except an ERB asset tag)
Expected: same pass/fail state as `main`

- [ ] **Step 6: Commit**

```bash
git add views/layout.erb public/application.js public/application.css
git commit -m "feat: dead-code session toolbar, state, and row filtering"
```

---

### Task 3: Row decoration — per-segment hide (✕) affordance

**Files:**
- Modify: `public/application.js` (add `window.deadCodeDecorate` + delegated click handlers, inside/after `initDeadCodeSession`)
- Modify: `public/application.css` (segment/icon styles)

**Interfaces:**
- Consumes: `window.deadCodeSession` (`D`, `state`, `saveState`, `redrawTables`), `CoverbandDeadCode.segmentPrefixes`.
- Produces: `window.deadCodeDecorate(oSettings)`; DOM structure per file link: `span.dcs-seg[data-prefix]` wrapping each segment, containing `button.dcs-hide` and `button.dcs-mark` inside `span.dcs-actions`. (`.dcs-mark` buttons are rendered here; their click behavior lands in Task 4.)

- [ ] **Step 1: Add the decorator (append inside `initDeadCodeSession`, before `redrawTables();` at the end)**

```js
    function escapeHtml(s) {
      return String(s).replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
    }

    window.deadCodeDecorate = function (oSettings) {
      $(oSettings.nTable).find("tbody td a.src_link").not(".dcs-done").each(function () {
        var link = $(this);
        var fullPath = link.attr("title");
        if (!fullPath) return;
        var segs = D.segmentPrefixes(fullPath);
        var html = "";
        for (var i = 0; i < segs.length; i++) {
          html += '<span class="dcs-seg" data-prefix="' + escapeHtml(segs[i].prefix) + '">' +
            escapeHtml(segs[i].label) +
            '<span class="dcs-actions">' +
            '<button type="button" class="dcs-hide" title="Hide ' + escapeHtml(segs[i].prefix) + (segs[i].isFile ? "" : " and everything under it") + '">&times;</button>' +
            '<button type="button" class="dcs-mark" title="Mark ' + escapeHtml(segs[i].prefix) + ' for deletion">&#128465;</button>' +
            '</span></span>' + (segs[i].isFile ? "" : "/");
        }
        link.html(html).addClass("dcs-done");
      });
    };
```

- [ ] **Step 2: Add delegated click handler for hide (append after the decorator)**

```js
    $(document).on("click", ".dcs-hide", function (e) {
      e.preventDefault();
      e.stopPropagation();
      D.addHidden(state, $(this).closest(".dcs-seg").attr("data-prefix"));
      saveState();
      redrawTables();
      return false;
    });
```

Note: DataTables 1.7 filtering reads its internally stored row data (the
original cell HTML), not the mutated DOM — decoration does not affect
search/sort. Decoration must re-run every draw because DataTables re-renders
rows from stored data; that is why it hangs off `fnDrawCallback`.

- [ ] **Step 3: Add segment/icon styles to `application.css`**

```css
.dcs-seg { position: relative; }
.dcs-actions { display: none; white-space: nowrap; }
.dcs-seg:hover > .dcs-actions { display: inline; }
.dcs-actions button { border: none; background: #ddd; border-radius: 3px; cursor: pointer; font-size: 0.85em; padding: 0 3px; margin-left: 2px; line-height: 1.2; }
.dcs-actions button:hover { background: #bbb; }
```

- [ ] **Step 4: Syntax check**

Run: `node --check public/application.js`
Expected: exit 0

- [ ] **Step 5: Commit**

```bash
git add public/application.js public/application.css
git commit -m "feat: per-segment hide affordance on file paths"
```

---

### Task 4: Mark-for-deletion popup, marked modal, CSV download

**Files:**
- Modify: `public/application.js` (mark popup, modal, CSV; inside `initDeadCodeSession`)
- Modify: `public/application.css` (popup/modal styles)

**Interfaces:**
- Consumes: `window.deadCodeSession` internals (`D`, `state`, `saveState`, `redrawTables`), `.dcs-mark` buttons from Task 3, `#dcs-marked-btn` from Task 2, `D.toCSV`.
- Produces: user-visible mark flow; no further consumers.

- [ ] **Step 1: Add mark popup (append inside `initDeadCodeSession`)**

```js
    function closeMarkPopup() { $("#dcs-mark-popup").remove(); }

    function openMarkPopup(prefix, x, y) {
      closeMarkPopup();
      var popup = $(
        '<div id="dcs-mark-popup">' +
        '  <div class="dcs-popup-path"></div>' +
        '  <textarea id="dcs-mark-comment" rows="2" placeholder="Optional comment (why is this dead?)"></textarea>' +
        '  <div class="dcs-popup-actions">' +
        '    <button type="button" id="dcs-mark-confirm">Mark for deletion</button>' +
        '    <button type="button" id="dcs-mark-cancel">Cancel</button>' +
        '  </div>' +
        '</div>'
      );
      popup.find(".dcs-popup-path").text(prefix);
      popup.css({ left: Math.min(x, $(window).width() - 340) + "px", top: (y + 8) + "px" });
      $("body").append(popup);
      $("#dcs-mark-comment").focus();
      $("#dcs-mark-cancel").on("click", closeMarkPopup);
      $("#dcs-mark-confirm").on("click", function () {
        D.addMark(state, prefix, $("#dcs-mark-comment").val(), new Date().toISOString());
        saveState();
        closeMarkPopup();
        redrawTables();
      });
    }

    $(document).on("click", ".dcs-mark", function (e) {
      e.preventDefault();
      e.stopPropagation();
      openMarkPopup($(this).closest(".dcs-seg").attr("data-prefix"), e.pageX, e.pageY);
      return false;
    });
```

- [ ] **Step 2: Add the marked modal + CSV download (append inside `initDeadCodeSession`)**

```js
    function closeMarkedModal() { $("#dcs-modal-overlay").remove(); }

    function downloadCSV() {
      var blob = new Blob([D.toCSV(state.markedPaths)], { type: "text/csv;charset=utf-8" });
      var a = document.createElement("a");
      a.href = URL.createObjectURL(blob);
      a.download = "coverband-dead-code-" + new Date().toISOString().slice(0, 10) + ".csv";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(a.href);
    }

    function openMarkedModal() {
      closeMarkedModal();
      var overlay = $('<div id="dcs-modal-overlay"><div id="dcs-modal">' +
        '<div id="dcs-modal-head"><strong>Marked for deletion</strong>' +
        '<button type="button" id="dcs-modal-close">&times;</button></div>' +
        '<table id="dcs-modal-table"><thead><tr>' +
        '<th>Path</th><th>Comment</th><th>Marked at</th><th></th>' +
        '</tr></thead><tbody></tbody></table>' +
        '<div id="dcs-modal-foot">' +
        '<button type="button" id="dcs-csv">Download CSV</button>' +
        '<button type="button" id="dcs-start-over">Start over</button>' +
        '</div></div></div>');
      var tbody = overlay.find("tbody");
      if (state.markedPaths.length === 0) {
        tbody.append('<tr><td colspan="4"><em>Nothing marked yet.</em></td></tr>');
      }
      $.each(state.markedPaths.slice(), function (_i, m) {
        var tr = $("<tr></tr>");
        $('<td class="dcs-mono"></td>').text(m.path).appendTo(tr);
        var commentInput = $('<input type="text" class="dcs-comment-edit">').val(m.comment)
          .on("change", function () {
            D.updateComment(state, m.path, $(this).val());
            saveState();
          });
        $("<td></td>").append(commentInput).appendTo(tr);
        $('<td class="dcs-mono"></td>').text(m.markedAt).appendTo(tr);
        $("<td></td>").append(
          $('<button type="button" title="Unmark">&times;</button>').on("click", function () {
            D.removeMark(state, m.path);
            saveState();
            redrawTables();
            openMarkedModal(); // rebuild
          })
        ).appendTo(tr);
        tbody.append(tr);
      });
      $("body").append(overlay);
      $("#dcs-modal-close").on("click", closeMarkedModal);
      overlay.on("click", function (e) { if (e.target === this) closeMarkedModal(); });
      $("#dcs-csv").on("click", downloadCSV);
      $("#dcs-start-over").on("click", function () {
        if (!confirm("Clear the entire marked-for-deletion list?")) return;
        D.clearMarks(state);
        saveState();
        redrawTables();
        openMarkedModal();
      });
    }

    $("#dcs-marked-btn").on("click", openMarkedModal);
```

- [ ] **Step 3: Add popup/modal styles to `application.css`**

```css
#dcs-mark-popup { position: absolute; z-index: 10000; background: #fff; border: 1px solid #999; border-radius: 4px; box-shadow: 0 2px 8px rgba(0,0,0,0.3); padding: 10px; width: 320px; }
#dcs-mark-popup .dcs-popup-path { font-family: monospace; font-weight: bold; margin-bottom: 6px; word-break: break-all; }
#dcs-mark-popup textarea { width: 100%; box-sizing: border-box; }
#dcs-mark-popup .dcs-popup-actions { margin-top: 6px; text-align: right; }
#dcs-modal-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 10001; }
#dcs-modal { background: #fff; border-radius: 6px; width: 80%; max-width: 900px; max-height: 80%; overflow: auto; margin: 5% auto; padding: 16px; }
#dcs-modal-head { display: flex; justify-content: space-between; margin-bottom: 10px; }
#dcs-modal-table { width: 100%; border-collapse: collapse; }
#dcs-modal-table th, #dcs-modal-table td { border-bottom: 1px solid #eee; padding: 4px 8px; text-align: left; }
#dcs-modal-foot { margin-top: 12px; text-align: right; }
#dcs-modal-foot button { margin-left: 8px; }
.dcs-mono { font-family: monospace; }
.dcs-comment-edit { width: 100%; box-sizing: border-box; }
```

- [ ] **Step 4: Syntax check**

Run: `node --check public/application.js`
Expected: exit 0

- [ ] **Step 5: Commit**

```bash
git add public/application.js public/application.css
git commit -m "feat: mark-for-deletion popup, marked modal, and CSV export"
```

---

### Task 5: Manual verification harness + README section

**Files:**
- Create: `scratch/config.ru` (git-ignored harness; add `scratch/` to `.gitignore` if not ignored)
- Modify: `README.md` (short "Dead-code review session" section under the web UI docs)

**Interfaces:**
- Consumes: the full feature from Tasks 1-4.

- [ ] **Step 1: Create a rackup harness with fake coverage data**

```ruby
# scratch/config.ru — local demo only, not shipped
$LOAD_PATH.unshift File.expand_path("../lib", __dir__)
require "coverband"

store = Coverband::Adapters::FileStore.new("/tmp/coverband_demo/coverage.json")
Coverband.configure do |config|
  config.store = store
  config.root = File.expand_path("..", __dir__)
end

# Seed fake coverage: mix of runtime-used and dead files
files = {
  "#{Coverband.configuration.root}/lib/coverband.rb" => [1, 1, 0, 5, nil, 0],
  "#{Coverband.configuration.root}/lib/coverband/reporters/web.rb" => [1, 0, 0, nil, 0, 0],
  "#{Coverband.configuration.root}/lib/coverband/utils/result.rb" => [0, 0, 0, 0, nil, 0]
}
store.type = Coverband::EAGER_TYPE
store.save_report(files)
store.type = Coverband::RUNTIME_TYPE
store.save_report({ "#{Coverband.configuration.root}/lib/coverband.rb" => [0, 3, 0, 2, nil, 0] })

run Coverband::Reporters::Web
```

- [ ] **Step 2: Run and verify in browser**

Run: `cd /Users/obie/projects/coverband && bundle exec rackup scratch/config.ru -p 9292`
Open http://localhost:9292/ and verify each item:
1. Toolbar renders above the table with 0 hidden / 0 marked.
2. Hovering a path segment shows ✕ and 🗑; hovering a directory segment vs filename shows correct tooltips.
3. Clicking directory ✕ hides all files under it; badge and "Hidden items" chips update; chip ✕ unhides.
4. "Hide files with runtime % > 0" hides `lib/coverband.rb` only; reload page → state persists.
5. 🗑 opens comment popup; confirming removes the row and increments the marked count.
6. Marked modal lists entries, comment edits persist across reloads, unmark restores the row.
7. Download CSV produces correct `path,comment,marked_at` file.
8. "Start over" empties marks; "Reset session" clears hides but keeps marks.
9. Source popup (clicking the path text itself) still opens colorbox; DataTables column sort and search box still work.

- [ ] **Step 3: Fix anything found; re-verify; run full checks**

Run: `node --test test/javascript/ && node --check public/application.js`
Expected: PASS

- [ ] **Step 4: Add README section**

Append near the web UI docs in `README.md`:

```markdown
### Dead-code review session

The web report includes a client-side "dead-code session" toolbar to help drive code-removal work:

- Hide files or whole folders from the report via the ✕ icon that appears when hovering path segments, or hide everything with runtime % > 0 with one toggle.
- Mark files or folders for deletion via the 🗑 icon (with an optional comment). Marked items disappear from the view and are listed in the "Marked for deletion" modal.
- Export the marked list as a CSV (`path,comment,marked_at`) to feed a code-removal PR process.

Session state lives in your browser's localStorage: "Reset session" clears the hidden set; "Start over" (in the modal) clears the marked list.
```

- [ ] **Step 5: Commit**

```bash
git add README.md .gitignore
git commit -m "docs: dead-code review session usage"
```

---

## Self-Review Notes

- Spec coverage: state module (T1), filter+toolbar+runtime toggle+reset+badges+chips (T2), segment hide affordance (T3), mark popup+modal+CSV+start-over (T4), manual verification+docs (T5). Error handling (corrupt state, localStorage absent, CSV escaping) covered in T1/T2.
- Names cross-checked: `deadCodeSession`, `deadCodeOnDraw`, `deadCodeDecorate`, `.dcs-seg[data-prefix]`, `.dcs-hide`, `.dcs-mark`, `#dcs-marked-btn` consistent across tasks.
- Paged mode: rows added via `fnAddData` trigger a draw → `fnDrawCallback` decorates and the filter applies; no special-casing needed.
