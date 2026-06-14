# Plan 003: Add an optional contract reference field («Договір №»)

> **Executor instructions**: Follow step by step. Run each verification and confirm the expected
> result. STOP and report on any STOP condition. Update `plans/README.md` when done.
>
> **Drift check (run first)**: no git SHA. Confirm "Current state" excerpts match `index.html`. On
> mismatch, STOP.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: direction (feature)
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

ФОП invoices to foreign clients normally cite the underlying service agreement
(«Договір про надання послуг №… від …»). It ties the payment to a contract for both his Ukrainian
tax reporting and your US records. The field must be **optional** (some payments have no contract).

## Current state

`index.html`, one inline-`<script>` IIFE. Two patterns you must mirror exactly:

1. **A persisted form field** — fields whose names are listed in `DEFAULT_FIELDS` are auto-saved
   and reloaded. Find `DEFAULT_FIELDS` near the top of the IIFE (an array of id strings such as
   `"clEmail"`, `"bkPurpose"`, `"notesUa"`, `"currency"`). The invoice-specific fields `invNo`,
   `invDate`, `dueDate` are in `ALL_FIELDS` only (not persisted). A contract reference is usually
   stable per client, so treat it like a **persisted default** (add to `DEFAULT_FIELDS`).
2. **The invoice meta block** in `render()` (~`index.html:660-690` area). Locate the existing meta
   rows that render invoice number / date / currency — they look like:
   ```js
   '<div><span class="lbl">Currency · Валюта:</span> ' + esc(cur) + "</div>" +
   ```
   and the provider helper `g` is defined at the top of `render()` as
   `var g = function (id) { return $(id).value.trim(); };`

Convention: every form field is a `.field` with an uppercase `<label>` containing an EN · UA caption
and an `<input>` with a matching `id`. See any existing field in the "Invoice details" `<details>`
section for the exact markup to copy.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| Input exists | `grep -n 'id="contractRef"' index.html` | one match |
| Persisted | `grep -n '"contractRef"' index.html` | ≥2 (DEFAULT_FIELDS + input) |

## Scope

**In scope**: `index.html` only.

**Out of scope**: do not add date-of-contract as a separate field unless trivial — one free-text
`contractRef` input (e.g. user types `№ 12 від 01.01.2026`) is the agreed scope. Do not touch
unrelated form sections.

## Steps

### Step 1: Add the input to the "Invoice details" section

Inside the `<details>` section whose summary is `Invoice details · Деталі рахунку`, add a new
`.field` after the currency/due-date row:
```html
<div class="field">
  <label>Contract ref · Договір (optional)</label>
  <input id="contractRef" type="text" placeholder="№ 12 від 01.01.2026" />
</div>
```

**Verify**: `grep -n 'id="contractRef"' index.html` → one match.

### Step 2: Persist it

Add the string `"contractRef"` to the `DEFAULT_FIELDS` array (so it auto-saves/reloads like the
business details). Do **not** add it to `ALL_FIELDS` separately — `ALL_FIELDS` is built as
`DEFAULT_FIELDS.concat([...])`, so it's included automatically.

**Verify**: `grep -n '"contractRef"' index.html` → at least two matches (array + input id).

### Step 3: Render it in the invoice meta (only when filled)

In `render()`, in the `.inv-meta` block, add a conditional row after the currency row. Mirror the
existing `dueDate` conditional pattern:
```js
(g("contractRef") ? '<div><span class="lbl">Contract · Договір:</span> ' + esc(g("contractRef")) + "</div>" : "") +
```

**Verify**: `grep -n "Contract · Договір" index.html` → one match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Open `index.html`. Leave Contract ref empty → invoice meta shows **no** contract line.
2. Type `№ 12 від 01.01.2026` → a `Contract · Договір: № 12 від 01.01.2026` line appears in the
   top-right meta block. Reload → value persists (it's a default).
3. Console: zero errors.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n 'id="contractRef"' index.html` → one match
- [ ] `grep -n '"contractRef"' index.html` → ≥2 matches
- [ ] Empty field renders no contract line; filled field renders one (smoke test)
- [ ] `plans/README.md` status row for 003 updated

## STOP conditions

- `DEFAULT_FIELDS` is not a plain array of id strings as described (code drifted) — STOP.
- The `.inv-meta` block can't be located in `render()` — STOP and report; do not guess placement.

## Maintenance notes

- It persists as a default because contracts are usually stable per client. If you later make it
  per-invoice (cleared each time), move `"contractRef"` out of `DEFAULT_FIELDS` and into the
  invoice-specific list alongside `invNo`.
