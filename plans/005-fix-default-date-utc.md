# Plan 005: Fix default invoice date off-by-one in western timezones

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
- **Category**: bug
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

`init()` seeds today's date with `toISOString()`, which returns the date in **UTC**. For users
west of UTC (e.g. US Pacific/Eastern in the evening), this lands on **tomorrow's** date. On a
financial document the issue date should reflect the user's local "today."

## Current state

`index.html:756-760`:
```js
// default dates
if (!$("invDate").value) {
  var t = new Date();
  $("invDate").value = t.toISOString().slice(0, 10);
}
```
An `<input id="invDate" type="date">` expects a `YYYY-MM-DD` value. Convention: ES5 `var`, 2-space
indent, double quotes.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| UTC seed removed | `grep -n "toISOString().slice(0, 10)" index.html` | **no match** (exit 1) |
| Local helper present | `grep -n "getFullYear()" index.html` | ≥1 match |

## Scope

**In scope**: the default-date block in `init()` in `index.html` only.

**Out of scope**: `fmtDate()` (the display formatter at `index.html:577-582`) — it already builds
the date from the input string with an explicit local-midnight `T00:00:00` and is correct. Do not
change it.

## Steps

### Step 1: Build the default date from local components

Replace the `toISOString()` seed with local year/month/day so it matches the user's calendar day:
```js
// default dates (local, not UTC)
if (!$("invDate").value) {
  var t = new Date();
  var pad = function (x) { return (x < 10 ? "0" : "") + x; };
  $("invDate").value = t.getFullYear() + "-" + pad(t.getMonth() + 1) + "-" + pad(t.getDate());
}
```

**Verify**: `grep -n "toISOString().slice(0, 10)" index.html` → no match;
`grep -n "getFullYear()" index.html` → ≥1 match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test (timezone-sensitive bug, so test the mechanism, not the clock):

1. Clear browser storage for the file (or use a fresh profile), open `index.html`.
   - **Expected**: the Date field shows **your local** today's date.
2. In DevTools console, run:
   ```js
   var t=new Date(); var pad=x=>(x<10?"0":"")+x; t.getFullYear()+"-"+pad(t.getMonth()+1)+"-"+pad(t.getDate());
   ```
   - **Expected**: the printed string equals the value in the Date field.
3. Console: zero errors.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n "toISOString().slice(0, 10)" index.html` → no match
- [ ] Default date equals local today (smoke test step 2 matches the field)
- [ ] `plans/README.md` status row for 005 updated

## STOP conditions

- The default-date block doesn't match the excerpt (drifted) — STOP.

## Maintenance notes

- Any future "due date in N days" helper should use the same local-components approach, not
  `toISOString()`, to avoid reintroducing this bug.
