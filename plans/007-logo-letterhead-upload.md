# Plan 007: Add an optional logo / letterhead

> **Executor instructions**: Follow step by step, verify each step, STOP on any STOP condition,
> update `plans/README.md` when done.
>
> **Drift check (run first)**: no git SHA. Confirm "Current state" excerpts match `index.html`. On
> mismatch, STOP.

## Status

- **Priority**: P3
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none
- **Category**: direction (feature)
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

A logo makes the invoice look like an official document from a real business — useful for both the
ФОП and your US company. Because the app is offline/single-file, the logo must be stored as a
base64 data URL in localStorage (no file paths, no uploads server).

## Current state

`index.html`, one inline-`<script>` IIFE.

- The invoice header is the `.inv-top` block in `render()`. It currently looks like:
  ```js
  '<div class="inv-top">' +
    '<div class="inv-title">INVOICE<small>РАХУНОК-ФАКТУРА</small></div>' +
    '<div class="inv-meta">' +
       /* number / date / currency rows */
    "</div>" +
  "</div>" +
  ```
  (Search for `class="inv-top"` to find it — around `index.html:660`.)
- `DEFAULT_FIELDS` array (top of IIFE) controls what persists. Note: it currently maps id → value
  via `$(f).value`, which works for inputs but **not** for a stored data URL that has no visible
  input value. So the logo needs its own persistence, not membership in `DEFAULT_FIELDS`.
- Storage pattern: `saveDefaults`/`loadDefaults` wrap localStorage in `try/catch`.

Conventions: ES5 `var`, double quotes, 2-space indent, escaped dynamic HTML via `esc()`.

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md`) | `SYNTAX OK` |
| File input added | `grep -n 'id="logoFile"' index.html` | one match |
| Logo storage key | `grep -n "invoice_builder_logo" index.html` | ≥1 match |
| Logo rendered | `grep -n "inv-logo" index.html` | ≥1 match |

## Scope

**In scope**: `index.html` only.

**Out of scope**:
- Image resizing/compression — store the file as-is (warn on very large files, see Step 2).
- Multiple logos / per-party logos — one logo this pass.
- Adding the logo to `DEFAULT_FIELDS` — it has its own key (data URLs don't fit that input-value
  mechanism).

## Steps

### Step 1: Add a file input + clear button to the editor

In the editor (a sensible place: the top of the "Service provider" section, or a new small block),
add:
```html
<div class="field">
  <label>Logo (optional) · Логотип</label>
  <input id="logoFile" type="file" accept="image/*" />
  <button class="btn" id="logoClear" type="button" style="margin-top:6px">Remove logo</button>
</div>
```

**Verify**: `grep -n 'id="logoFile"' index.html` → one match.

### Step 2: Read, store, and persist the logo as a data URL

Add `var LOGO_KEY = "invoice_builder_logo";` near `LS_KEY`. Add handlers in `init()`:
```js
$("logoFile").addEventListener("change", function (e) {
  var f = e.target.files && e.target.files[0];
  if (!f) return;
  if (f.size > 1024 * 1024) { alert("Logo is larger than 1 MB — please use a smaller image."); return; }
  var reader = new FileReader();
  reader.onload = function () {
    try { localStorage.setItem(LOGO_KEY, reader.result); } catch (err) { alert("Could not store logo: " + err.message); }
    render();
  };
  reader.readAsDataURL(f);
});
$("logoClear").addEventListener("click", function () {
  try { localStorage.removeItem(LOGO_KEY); } catch (e) {}
  $("logoFile").value = "";
  render();
});
```

**Verify**: `grep -n "invoice_builder_logo" index.html` → ≥1 match. Syntax gate → `SYNTAX OK`.

### Step 3: Render the logo in the invoice header

In `render()`, before building the `.inv-top` string, read the stored logo:
```js
var logo = "";
try { logo = localStorage.getItem(LOGO_KEY) || ""; } catch (e) {}
```
Then put it inside `.inv-title` (or just before it) — render an `<img>` only when present:
```js
'<div class="inv-title">' +
  (logo ? '<img class="inv-logo" src="' + esc(logo) + '" alt="logo" />' : "") +
  'INVOICE<small>РАХУНОК-ФАКТУРА</small></div>' +
```
Note: a data URL is app-generated (from the user's own file), but keep `esc()` for consistency.

**Verify**: `grep -n "inv-logo" index.html` → ≥1 match.

### Step 4: Add CSS for the logo

In `<style>`, near `.inv-title`:
```css
.inv-logo { display:block; max-height:56px; max-width:220px; margin-bottom:8px; object-fit:contain; }
```

**Verify**: `grep -n "\.inv-logo" index.html` → ≥1 match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test:

1. Open `index.html`. Upload a small PNG/JPG (<1 MB) → it appears above the "INVOICE" title in the
   preview. Reload → logo persists.
2. Click **Remove logo** → image disappears; reload → still gone.
3. Try a >1 MB image → alert shown, no logo stored.
4. Print/Save PDF → logo appears in the PDF.
5. Console: zero errors.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n 'id="logoFile"' index.html` → one match
- [ ] `grep -n "invoice_builder_logo" index.html` → ≥1 match
- [ ] `grep -n "inv-logo" index.html` → ≥2 matches (render + CSS)
- [ ] Upload / persist / remove / print all work (smoke test) with zero console errors
- [ ] `plans/README.md` status row for 007 updated

## STOP conditions

- The `.inv-top` / `.inv-title` block can't be located in `render()` — STOP; do not guess markup.
- localStorage quota errors on a normal (<1 MB) image — report it rather than silently dropping.

## Maintenance notes

- Logos near 1 MB plus a large history (Plan 006) could approach the ~5 MB localStorage quota.
  If users hit quota errors, the future fix is client-side downscaling before storing — deferred.
- The logo is per-browser, like all other stored data; it is not bundled into exported records.
