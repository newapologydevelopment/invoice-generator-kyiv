# Plan 002: Show the total amount in words («сума прописом»)

> **Executor instructions**: Follow step by step. Run every verification command and confirm the
> expected result before moving on. If a "STOP condition" occurs, stop and report. When done,
> update the status row in `plans/README.md`.
>
> **Drift check (run first)**: no git SHA exists. Confirm the "Current state" excerpts still match
> `index.html`. On mismatch, STOP.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: LOW
- **Depends on**: none (land Plan 004 first if doing both — soft overlap, see `plans/README.md`)
- **Category**: direction (feature)
- **Planned at**: no VCS (not a git repo), 2026-06-14

## Why this matters

Ukrainian invoices and acts of completed work customarily restate the total in words
(«сума прописом») — it is an expected convention on ФОП paperwork and reduces tampering/typo
disputes. This is the single most impactful domain feature for this exact use case (UA sole
proprietor billing a US client). After this plan, the invoice prints e.g.
`Five thousand four hundred US dollars 00 cents / П'ять тисяч чотириста доларів США 00 центів`.

## Current state

Single file `index.html`, one IIFE in the inline `<script>`. Relevant pieces:

- `index.html:570-575` — money helper:
  ```js
  function money(n, cur) {
    var sym = CUR[cur] || "";
    var v = isFinite(n) ? n : 0;
    return sym + v.toLocaleString("en-US", { minimumFractionDigits: 2, maximumFractionDigits: 2 });
  }
  ```
- The `CUR` map (near the top of the IIFE): `var CUR = { USD: "$", EUR: "€", UAH: "₴" };`
- `index.html:589` — `total` accumulates the numeric grand total (a Number).
- `index.html:699-702` — the totals block in the `render()` template string:
  ```js
  '<div class="totals"><table>' +
    '<tr><td class="lbl">Subtotal · Проміжний підсумок</td><td class="r">' + money(total, cur) + "</td></tr>" +
    '<tr class="grand"><td>TOTAL DUE · ДО СПЛАТИ</td><td class="r">' + money(total, cur) + "</td></tr>" +
  "</table></div>" +
  ```

Conventions: ES5 `var`, double quotes, 2-space indent, all dynamic strings escaped via `esc()`
(but the words helper output is generated from numbers, so it's already safe — no user text).

## Commands you will need

| Purpose | Command | Expected |
|---|---|---|
| Syntax gate | (see `plans/README.md` node one-liner) | `SYNTAX OK` |
| Assert helper added | `grep -n "function amountInWords" index.html` | one match |
| Assert rendered | `grep -n "amount-words" index.html` | at least one match (CSS + render) |

## Scope

**In scope**: `index.html` only.

**Out of scope**:
- The `money()` formatting itself — that's Plan 004.
- Currencies beyond USD/EUR/UAH — only those three exist in `CUR`; handle exactly those.
- Kopecks/cents grammar perfection in Ukrainian declension — keep it pragmatic (see Step 2 note).

## Steps

### Step 1: Add an `amountInWords(n, cur)` helper

Add a new function directly **below** `money()` (after `index.html:575`). It converts a number to
words in English and Ukrainian and labels the currency + fractional unit per currency.

Required behavior:
- Split `n` into integer part and fractional (`Math.round((n - int) * 100)` cents/kopecks).
- Convert the integer part to words with a self-contained integer-to-words routine (no library —
  this is a zero-dependency project). Support values up to at least 9,999,999.
- Currency words:
  - USD → EN `"US dollars"` / fractional `"cents"`; UA `"доларів США"` / `"центів"`
  - EUR → EN `"euros"` / `"cents"`; UA `"євро"` / `"центів"`
  - UAH → EN `"hryvnias"` / `"kopiykas"`; UA `"гривень"` / `"копійок"`
- Return an object `{ en: "...", ua: "..." }`, each like
  `"Five thousand four hundred US dollars 00 cents"` and
  `"П'ять тисяч чотириста доларів США 00 копійок"`.
- Capitalize the first letter of each string.
- Pad the fractional part to two digits (e.g. `00`, `05`).

Target shape (illustrative — produce equivalent, fill in the two word-lists):
```js
function amountInWords(n, cur) {
  var v = isFinite(n) ? n : 0;
  var intPart = Math.floor(v);
  var frac = Math.round((v - intPart) * 100);
  var enOnes = ["zero","one","two", /* ... up to nineteen */];
  var enTens = ["","","twenty","thirty", /* ... ninety */];
  // ... scale words: thousand, million ...
  function enWords(num) { /* returns words for an integer */ }
  var uaOnes = ["нуль","один","два", /* ... */];   // feminine forms ok for гривень
  // ... ua tens / scales (тисяча/тисячі/тисяч with simple plural) ...
  function uaWords(num) { /* returns words for an integer */ }
  var units = {
    USD: { en: ["US dollars","cents"], ua: ["доларів США","центів"] },
    EUR: { en: ["euros","cents"],      ua: ["євро","центів"] },
    UAH: { en: ["hryvnias","kopiykas"], ua: ["гривень","копійок"] }
  }[cur] || { en: ["",""], ua: ["",""] };
  function cap(s){ return s.charAt(0).toUpperCase() + s.slice(1); }
  var pad = (frac < 10 ? "0" : "") + frac;
  return {
    en: cap((enWords(intPart) + " " + units.en[0] + " " + pad + " " + units.en[1]).trim()),
    ua: cap((uaWords(intPart) + " " + units.ua[0] + " " + pad + " " + units.ua[1]).trim())
  };
}
```
Pragmatic-scope note: full Ukrainian grammatical declension of number+noun is complex. Use the
fixed genitive-plural noun forms above (`доларів`, `гривень`, `центів`, `копійок`) — that reads
correctly for the vast majority of amounts and is the common simplification. Do **not** attempt
full declension; if tempted, STOP and report instead (see STOP conditions).

**Verify**: `grep -n "function amountInWords" index.html` → one match. Run the syntax gate → `SYNTAX OK`.

### Step 2: Render the words under the total

In the totals block (`index.html:699-702`), after the closing `</table></div>` of `.totals`,
insert a new line that computes the words and renders a bilingual row. Compute before the template
or inline:
```js
var words = amountInWords(total, cur);
```
and add this markup immediately after the `.totals` div in the concatenated `html` string:
```js
'<div class="amount-words">' +
  '<span class="lbl">Amount in words · Сума словами</span>' +
  '<div>' + esc(words.en) + '</div>' +
  '<div style="color:#6b7280">' + esc(words.ua) + '</div>' +
"</div>" +
```
(`esc` already exists in the IIFE; the input is numeric-derived so this is belt-and-suspenders.)

**Verify**: `grep -n "amount-words" index.html` → at least one match.

### Step 3: Add minimal CSS for `.amount-words`

In the `<style>` block, near the `.totals` rules, add:
```css
.amount-words { margin: -8px 0 22px; font-size: 10.5px; }
.amount-words .lbl { display:block; font-size:9px; text-transform:uppercase; letter-spacing:.05em; color:var(--muted); margin-bottom:3px; }
```

**Verify**: `grep -n "\.amount-words" index.html` → at least one match. Syntax gate → `SYNTAX OK`.

## Test plan

Manual smoke test (no test framework):

1. Open `index.html`. Add a line item qty `120`, price `45` (total 5,400.00), currency USD.
   - **Expected** under the total: `Five thousand four hundred US dollars 00 cents` and the
     Ukrainian equivalent `П'ять тисяч чотириста доларів США 00 копійок`.
2. Change currency to UAH → noun words switch to `гривень` / `гривні`-style and `hryvnias`.
3. Set price `45.50`, qty `1` → cents/копійок show `50`.
4. Edge: price `0`, qty `0` → `Zero ... 00 ...`, no console error.
5. Console: **zero errors**.

## Done criteria

- [ ] Syntax gate prints `SYNTAX OK`
- [ ] `grep -n "function amountInWords" index.html` → one match
- [ ] `grep -n "amount-words" index.html` → ≥2 matches (render + CSS)
- [ ] Smoke test steps 1–4 produce correct words with zero console errors
- [ ] `plans/README.md` status row for 002 updated

## STOP conditions

- The integer-to-words routine would need to exceed millions for real invoices and you're unsure
  of correctness past 9,999,999 — report the cap rather than shipping wrong words.
- You find yourself implementing full Ukrainian case declension (one/two/five → різні форми) — STOP;
  the fixed genitive-plural simplification is the agreed scope.
- `total` is not a Number at `index.html:589` (code drifted) — STOP.

## Maintenance notes

- If a new currency is added to `CUR`, add a matching entry to the `units` map here or it falls
  back to empty unit words.
- Plan 004 changes `money()` formatting only; it does not affect `amountInWords` (which reads the
  numeric `total`). If both land, confirm the displayed digits and the words agree.
