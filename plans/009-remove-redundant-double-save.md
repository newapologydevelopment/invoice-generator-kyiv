# Plan 009: Remove the redundant localStorage write on every keystroke

> **Executor instructions**: Follow step by step, verify, STOP on any STOP condition, update
> `plans/README.md` when done.
>
> **Drift check (run first)**: no git SHA. Confirm the excerpts match `index.html`. On mismatch, STOP.

## Status

- **Priority**: P3
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: tech-debt / perf
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

`render` is wrapped at module scope so that **every** call to `render()` also calls
`saveDefaults(true)`. The per-field input listeners then call `render()` **and** `saveDefaults(true)`
again — two serialize-and-write-to-localStorage operations per keystroke. It's harmless but wasteful
and confusing. Remove the duplicate so there's exactly one save path: `render()` saves.

## Current state

Two relevant spots in `index.html`.

1. The input listeners in `init()` (`index.html:766-772`):
   ```js
   ALL_FIELDS.forEach(function (f) {
     var el = $(f);
     el.addEventListener("input", function () {
       render();
       saveDefaults(true);
     });
   });
   ```
2. The render wrapper at module scope (`index.html:791-793`):
   ```js
   // save items whenever they change (render is called on item edits)
   var _render = render;
   render = function () { _render(); saveDefaults(true); };
   ```
Because the wrapper already calls `saveDefaults(true)` on every `render()`, the second call inside
the listener is redundant.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| One save path | `grep -c "saveDefaults(true)" index.html` | `1` |

(After the fix, `saveDefaults(true)` should appear exactly **once** — inside the render wrapper.)

## Scope

**In scope**: the input-listener block in `init()` in `index.html`.

**Out of scope**:
- The render wrapper (`index.html:791-793`) — leave it; it is the single intended save path.
- `saveDefaults(false)` calls (the explicit **Save defaults** button) — different arg, keep them.

## Steps

### Step 1: Drop the duplicate save from the input listener

Change the listener body so it only calls `render()` (the wrapper saves):
```js
ALL_FIELDS.forEach(function (f) {
  var el = $(f);
  el.addEventListener("input", function () {
    render();
  });
});
```

**Verify**: `grep -c "saveDefaults(true)" index.html` → `1`. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Open `index.html`. Type into the provider-name field, then reload.
   - **Expected**: the value persisted (autosave still works — proves the wrapper's save path runs).
2. Click **Save defaults** → still flashes "Saved ✓" (the `saveDefaults(false)` path is untouched).
3. Console: zero errors.

## Done criteria

- [ ] `grep -c "saveDefaults(true)" index.html` → `1`
- [ ] Syntax gate prints `SYNTAX OK`
- [ ] Typing a field then reloading still restores the value (autosave intact)
- [ ] **Save defaults** button still shows the "Saved ✓" flash
- [ ] `plans/README.md` status row for 009 updated

## STOP conditions

- After the change, `grep -c "saveDefaults(true)" index.html` is `0` — you removed the wrapper's
  save too. Restore so exactly one remains (in the wrapper). 
- The render wrapper at `index.html:791-793` is not present (code drifted) — STOP; without it,
  removing the listener's save would break autosave.

## Maintenance notes

- If Plan 006 (history) added a `saveHistory` call, that's a different function and unaffected.
- The single remaining save path is the render wrapper; any future "save on change" need should
  rely on `render()` rather than re-adding explicit `saveDefaults` calls in listeners.
