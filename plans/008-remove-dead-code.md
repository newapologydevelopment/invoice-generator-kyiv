# Plan 008: Remove dead code (`origAdd`)

> **Executor instructions**: Follow step by step, verify, STOP on any STOP condition, update
> `plans/README.md` when done.
>
> **Drift check (run first)**: no git SHA. Confirm the excerpt matches `index.html`. On mismatch, STOP.

## Status

- **Priority**: P3
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: tech-debt
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

`init()` contains a leftover assignment that is never used. Dead code misleads future readers (and
executor models) into thinking it matters. Removing it is free and risk-free.

## Current state

`index.html:786-788`:
```js
    // autosave items too
    var origAdd = addItem;
    render();
```
`origAdd` is assigned and never referenced anywhere. Confirm with a grep before editing:
`grep -n "origAdd" index.html` should return exactly **one** line (the assignment).

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Pre-check single use | `grep -n "origAdd" index.html` | exactly one match (the assignment) |
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| Post-check removed | `grep -n "origAdd" index.html` | **no match** (exit 1) |

## Scope

**In scope**: the three lines above in `init()` in `index.html`.

**Out of scope**: the `render();` call on the last line — it is **required** (it draws the initial
preview). Keep it. Only remove the comment and the `var origAdd` line.

## Steps

### Step 1: Confirm it's truly unused

Run `grep -n "origAdd" index.html`. If it returns more than one line, **STOP** — it's referenced
somewhere and this plan's premise is wrong.

### Step 2: Delete the dead line and its misleading comment

Change:
```js
    // autosave items too
    var origAdd = addItem;
    render();
```
to:
```js
    render();
```

**Verify**: `grep -n "origAdd" index.html` → no match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Open `index.html` → the invoice preview renders on load (proves the kept `render()` still runs).
2. Type in a field → preview updates live.
3. Console: zero errors.

## Done criteria

- [ ] `grep -n "origAdd" index.html` → no match
- [ ] `grep -n "render();" index.html` still present in `init()` (initial render intact)
- [ ] Syntax gate prints `SYNTAX OK`
- [ ] Preview renders on load with zero console errors
- [ ] `plans/README.md` status row for 008 updated

## STOP conditions

- Step 1 grep returns more than one match — `origAdd` is actually used; STOP and report.

## Maintenance notes

- None. Pure deletion.
