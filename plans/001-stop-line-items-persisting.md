# Plan 001: Line items no longer carry over from the previous invoice

> **Executor instructions**: Follow this plan step by step. Run every verification command and
> confirm the expected result before moving on. If anything in "STOP conditions" occurs, stop and
> report — do not improvise. When done, update the status row for this plan in `plans/README.md`.
>
> **Drift check (run first)**: there is no git SHA for this project. Instead, open `index.html`
> and confirm the excerpts under "Current state" still match the live code. On any mismatch,
> treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: bug
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

The tool saves the current line items into the same browser-storage blob it uses for the
provider/client/banking **defaults**. So when the paymaster opens the tool to create a *new*
invoice, the previous invoice's line items are restored. If he updates the amount but forgets a
leftover row, he bills the wrong total — a money error on a financial document. Business and
banking details *should* persist (they rarely change); line items should not (they're per-invoice).

## Current state

Single file: `index.html`. All logic is one IIFE in the inline `<script>`. Relevant pieces:

- `index.html:724` — `saveDefaults()` writes line items into the defaults blob:
  ```js
  data.__items = items;
  ```
- `index.html:738-740` — `loadDefaults()` restores them on every load:
  ```js
  if (Array.isArray(data.__items) && data.__items.length) {
    items = data.__items;
  }
  ```
- `index.html:752-763` — `init()` loads defaults, then seeds one empty row only if none were restored:
  ```js
  function init() {
    loadDefaults();
    // default dates ...
    if (!items.length) addItem();
    else drawItems();
  ```

Convention to match: the file uses ES5-style `var`, 2-space indent, double-quoted strings,
and `try/catch` around all `localStorage` access (see `saveDefaults`/`loadDefaults`). Match it.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Syntax gate | (see `plans/README.md` → node one-liner) | `SYNTAX OK` |
| Assert items not saved | `grep -n "data.__items = items" index.html` | **no match** (exit 1) |
| Assert restore removed | `grep -n "items = data.__items" index.html` | **no match** (exit 1) |

## Scope

**In scope** (only file to modify): `index.html`

**Out of scope** (do NOT touch):
- `DEFAULT_FIELDS` / `ALL_FIELDS` arrays — they correctly exclude per-invoice fields already.
- The `reset` button handler — it already clears items; leave it.
- Plan 006's history feature — do not build invoice history here.

## Steps

### Step 1: Stop writing line items into the defaults blob

In `saveDefaults()` (~`index.html:721-729`), delete the line:
```js
data.__items = items;
```

**Verify**: `grep -n "data.__items = items" index.html` → no match.

### Step 2: Stop restoring line items on load

In `loadDefaults()` (~`index.html:737-740`), delete the block:
```js
if (Array.isArray(data.__items) && data.__items.length) {
  items = data.__items;
}
```
Leave the line above it (`DEFAULT_FIELDS.forEach(...)`) intact.

**Verify**: `grep -n "data.__items" index.html` → no match.

### Step 3: Confirm a fresh invoice starts with one empty row

No code change expected — `init()` already calls `addItem()` when `items` is empty (`index.html:762`).
After steps 1–2, `items` is always empty at load, so this path runs. Just confirm the line is still there.

**Verify**: `grep -n "if (!items.length) addItem();" index.html` → exactly one match.

## Test plan

No unit-test framework (see `plans/README.md`). Manual smoke test:

1. Open `index.html` in a browser. Fill provider name, client name, and one line item. Reload.
   - **Expected**: provider/client persist; the line item is **gone**, one empty row shown.
2. Refill a line item, click **Save defaults**, reload.
   - **Expected**: still no line items restored (only business/banking defaults persist).
3. Console shows **zero errors** throughout.

## Done criteria

ALL must hold:

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n "data.__items" index.html` → no matches
- [ ] `grep -n "if (!items.length) addItem();" index.html` → one match
- [ ] Manual smoke test step 1 passes (line items do not survive reload; business details do)
- [ ] `plans/README.md` status row for 001 updated

## STOP conditions

- The `saveDefaults`/`loadDefaults` functions don't match the excerpts above (code drifted).
- After removing the restore block, reloading throws a console error referencing `items` being
  undefined — report instead of patching around it.

## Maintenance notes

- Plan 006 (invoice history) will reintroduce per-invoice persistence, but under a **separate**
  storage key — not the defaults blob. Keep these concerns separate.
- A reviewer should confirm business/banking defaults still persist (we only removed `__items`).
