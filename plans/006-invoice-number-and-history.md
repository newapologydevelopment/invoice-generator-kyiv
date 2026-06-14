# Plan 006: Auto-increment invoice № and keep a history of issued invoices

> **Executor instructions**: Follow step by step, verify each step, STOP on any STOP condition,
> update `plans/README.md` when done. This is the largest plan — do the steps in order; the app
> stays working between steps.
>
> **Drift check (run first)**: no git SHA. Confirm "Current state" excerpts match `index.html`. On
> mismatch, STOP. **Also confirm Plan 001 is DONE** (status in `plans/README.md`) — this plan
> depends on it.

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: MED (touches persistence + adds UI)
- **Depends on**: `plans/001-stop-line-items-persisting.md` (must be DONE first)
- **Category**: direction (feature)
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

Today nothing records what was issued — each invoice overwrites the live form and the invoice
number is typed by hand (easy to duplicate or skip). Sequential numbering and a record of issued
invoices are basic bookkeeping needs for a ФОП billing a foreign client. This plan adds: (a) a
suggested next number on a fresh invoice, and (b) a lightweight "Save to history" + reload-past-
invoice capability, stored locally in the browser.

## Why it depends on 001

Plan 001 removes line items from the shared **defaults** blob (`LS_KEY`). This plan stores each
issued invoice — including its line items — under a **separate** history key. Building history
before 001 would entangle per-invoice items with the defaults again.

## Current state

`index.html`, one inline-`<script>` IIFE. Relevant anchors:

- `var LS_KEY = "invoice_builder_v1";` (top of IIFE) — the defaults storage key.
- `DEFAULT_FIELDS` / `ALL_FIELDS` arrays (top of IIFE) — `invNo` is in `ALL_FIELDS` only.
- `saveDefaults()` / `loadDefaults()` (~`index.html:720-741`) — localStorage read/write pattern,
  all wrapped in `try/catch`.
- `items` array (module-scoped) holds the current line items; `drawItems()` re-renders the editor
  rows; `render()` re-renders the invoice preview.
- The toolbar (~`index.html` near the editor bottom) has buttons:
  ```html
  <button class="btn btn-accent" id="print">⤓ Save PDF / Print</button>
  <button class="btn" id="save">Save defaults</button>
  <button class="btn" id="reset">Reset</button>
  ```
  and their handlers are wired in `init()` (~`index.html:774-784`).

Conventions: ES5 `var`, double quotes, 2-space indent, all storage access in `try/catch`, all
dynamic HTML escaped via `esc()`.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| History key added | `grep -n "invoice_builder_history" index.html` | ≥1 match |
| Save-to-history button | `grep -n 'id="saveHistory"' index.html` | one match |
| Next-number helper | `grep -n "function suggestNextNumber" index.html` | one match |

## Scope

**In scope**: `index.html` only.

**Out of scope**:
- Server-side or cloud sync — history is browser-local only (localStorage), matching the rest of
  the app. Do not add network calls.
- Editing/deleting individual history entries beyond a simple "load" — keep it minimal this pass.
- Changing the defaults persistence from Plan 001 — leave `LS_KEY` behavior as 001 left it.

## Steps

### Step 1: Add a history storage key and helpers

Near `var LS_KEY = ...`, add `var HIST_KEY = "invoice_builder_history";`. Add two helpers next to
`saveDefaults`/`loadDefaults`:
```js
function loadHistory() {
  try { return JSON.parse(localStorage.getItem(HIST_KEY)) || []; }
  catch (e) { return []; }
}
function saveHistory(list) {
  try { localStorage.setItem(HIST_KEY, JSON.stringify(list)); } catch (e) {}
}
```

**Verify**: `grep -n "invoice_builder_history" index.html` → ≥1 match. Syntax gate → `SYNTAX OK`.

### Step 2: Suggest the next invoice number on a fresh invoice

Add a helper that reads history and proposes the next number. Support the common `YYYY-NNN` pattern
the placeholder already uses (`2026-001`), falling back to a plain increment:
```js
function suggestNextNumber() {
  var hist = loadHistory();
  var year = new Date().getFullYear();
  var max = 0;
  hist.forEach(function (h) {
    var m = /(\d{4})-(\d+)/.exec(h.invNo || "");
    if (m && +m[1] === year) max = Math.max(max, +m[2]);
  });
  var pad = function (x) { return x < 10 ? "00" + x : x < 100 ? "0" + x : "" + x; };
  return year + "-" + pad(max + 1);
}
```
In `init()`, after `loadDefaults()` and the default-date block, seed the number only when empty:
```js
if (!$("invNo").value) $("invNo").value = suggestNextNumber();
```

**Verify**: `grep -n "function suggestNextNumber" index.html` → one match. Syntax gate → `SYNTAX OK`.

### Step 3: Add a "Save to history" button

In the toolbar, add after the `save` button:
```html
<button class="btn" id="saveHistory">Save to history</button>
```
Wire it in `init()`:
```js
$("saveHistory").addEventListener("click", function () {
  var g = function (id) { return $(id).value; };
  var entry = { savedAt: Date.now(), invNo: g("invNo"), invDate: g("invDate"),
    dueDate: g("dueDate"), currency: g("currency"), clName: g("clName"), items: items };
  var hist = loadHistory();
  hist.unshift(entry);
  saveHistory(hist);
  renderHistory();
  flash("Saved to history ✓");  // flash() exists; it briefly relabels the #save button
});
```
(`flash` currently relabels the `#save` button — that's fine; reuse it.)

**Verify**: `grep -n 'id="saveHistory"' index.html` → one match. Syntax gate → `SYNTAX OK`.

### Step 4: Render the history list and allow loading an entry

Add a small history panel. Put a container in the editor (e.g. a new `<details class="section">`
titled `History · Історія` with `<div id="historyList"></div>` inside). Add:
```js
function renderHistory() {
  var box = $("historyList"); if (!box) return;
  var hist = loadHistory();
  if (!hist.length) { box.innerHTML = '<p class="hint" style="padding:0">No saved invoices yet.</p>'; return; }
  box.innerHTML = hist.map(function (h, i) {
    return '<div class="item" style="display:flex;justify-content:space-between;align-items:center">' +
      '<span>' + esc(h.invNo || "—") + ' · ' + esc(h.clName || "") + '</span>' +
      '<button class="btn" data-load="' + i + '">Load</button></div>';
  }).join("");
  box.querySelectorAll("[data-load]").forEach(function (b) {
    b.addEventListener("click", function () {
      var h = loadHistory()[+b.getAttribute("data-load")];
      if (!h) return;
      ["invNo","invDate","dueDate","currency","clName"].forEach(function (k){ if (h[k] != null) $(k).value = h[k]; });
      items = Array.isArray(h.items) ? h.items.slice() : [];
      if (!items.length) addItem(); else drawItems();
      render();
    });
  });
}
```
Call `renderHistory()` once at the end of `init()`.

**Verify**: `grep -n "function renderHistory" index.html` → one match;
`grep -n 'id="historyList"' index.html` → one match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Open a fresh `index.html` (cleared storage). Invoice № auto-fills (`2026-001`).
2. Fill a client + one line item, click **Save to history** → entry appears in the History panel.
3. Click **Reset**, reload → invoice № now suggests `2026-002` (incremented from history).
4. Click **Load** on the saved entry → client, number, and line items repopulate; preview updates.
5. Reload the page → history entries still listed (persisted).
6. Console: zero errors throughout.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n "invoice_builder_history" index.html` → ≥1 match
- [ ] `grep -n 'id="saveHistory"' index.html` → one match
- [ ] `grep -n "function suggestNextNumber" index.html` → one match
- [ ] `grep -n "function renderHistory" index.html` → one match
- [ ] Smoke test steps 1–5 pass with zero console errors
- [ ] `plans/README.md` status row for 006 updated

## STOP conditions

- Plan 001 is not DONE (line items still load from the defaults blob) — STOP; do 001 first.
- The toolbar markup or `init()` handler-wiring block doesn't match the excerpts — STOP.
- localStorage is unavailable (e.g. `file://` blocked in the test browser) and history can't
  persist — report it; the feature is inherently storage-backed, don't fabricate a workaround.

## Maintenance notes

- History is local to one browser/profile; it is **not** a backup. If the user needs durable
  records, that's a future export feature (e.g. download JSON) — out of scope here.
- `Date.now()` is used only as an opaque sort/id stamp on entries; it doesn't drive any display.
- If list size grows large, consider capping or adding delete — deferred this pass.
