# Plan 004: Format money per currency locale

> **Executor instructions**: Follow step by step, verify each step, STOP on any STOP condition,
> update `plans/README.md` when done.
>
> **Drift check (run first)**: no git SHA. Confirm the "Current state" excerpt matches `index.html`.
> On mismatch, STOP.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: i18n / bug
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

`money()` always formats with `en-US` grouping (`5,400.00`), even when the currency is UAH, where
Ukrainian convention is space-grouped thousands and a comma decimal (`5 400,00`). The invoice is
bilingual and may be denominated in UAH, so amounts should read naturally for the currency.

## Current state

`index.html:570-575`:
```js
function money(n, cur) {
  var sym = CUR[cur] || "";
  var v = isFinite(n) ? n : 0;
  return sym + v.toLocaleString("en-US", { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
```
`CUR` (top of IIFE): `var CUR = { USD: "$", EUR: "€", UAH: "₴" };`. `money()` is called from the
items table and the totals block in `render()`.

Convention: ES5 `var`, double quotes, 2-space indent.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| Locale map present | `grep -n "uk-UA" index.html` | one match |

## Scope

**In scope**: the `money()` function in `index.html` only.

**Out of scope**: changing the currency symbols in `CUR`; changing where the symbol sits relative
to the number (keep symbol-prefix as today, e.g. `₴5 400,00`) — repositioning the symbol is not
in scope and risks layout churn in the totals table.

## Steps

### Step 1: Pick locale by currency

Rewrite `money()` to choose a locale: UAH → `"uk-UA"`, everything else → `"en-US"`. Keep the symbol
prefix and the 2-decimal rule:
```js
function money(n, cur) {
  var sym = CUR[cur] || "";
  var v = isFinite(n) ? n : 0;
  var locale = cur === "UAH" ? "uk-UA" : "en-US";
  return sym + v.toLocaleString(locale, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
}
```
Note: `uk-UA` grouping renders as a narrow no-break space — that's expected and correct.

**Verify**: `grep -n "uk-UA" index.html` → one match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Currency USD, line item 1 × 5400 → totals show `$5,400.00`.
2. Switch currency to UAH → totals show `₴5 400,00` (space-grouped, comma decimal).
3. Switch to EUR → `€5,400.00` (en-US grouping retained, by design).
4. Console: zero errors.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n "uk-UA" index.html` → one match
- [ ] UAH renders space-grouped comma-decimal; USD/EUR unchanged (smoke test)
- [ ] `plans/README.md` status row for 004 updated

## STOP conditions

- The running environment's `toLocaleString("uk-UA", ...)` ignores the locale (very old engine) and
  output is identical to en-US — report it; do not hand-roll a formatter.
- `money()` no longer matches the excerpt (drifted) — STOP.

## Maintenance notes

- If Plan 002 (amount-in-words) is also done, the spelled-out words come from the numeric `total`,
  not from this formatted string, so the two are independent. Confirm digits and words still agree.
- Adding a new currency means deciding its locale here as well as adding it to `CUR`.
