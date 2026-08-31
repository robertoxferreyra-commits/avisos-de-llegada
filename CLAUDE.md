# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained HTML file (`Qwen_html_20260828_2ajzzp8ws.html`) that runs entirely
in the browser. It is a logistics tool for an Evergreen Line / Everbol Shipping agency that:

1. Ingests an XLSX shipping report,
2. Builds an interactive TEU/container dashboard (Chart.js), and
3. Generates single-page A4 "Aviso de Llegada" (cargo arrival notice) PDFs, individually or
   bulk-zipped per vessel.

UI and domain language are Spanish. There is **no build system, no package manager, no tests,
no lint**. To run it, open the HTML file directly in a browser:

```bash
start Qwen_html_20260828_2ajzzp8ws.html
```

All dependencies load from CDNs via `<script>`/`<link>` tags: SheetJS (`xlsx`), Chart.js,
JSZip, html2pdf.js, and (at the end of `<body>`) html2canvas + jsPDF. An internet connection
is required for the page to function.

## Code structure (all inside the one file)

- `<style>` blocks: main theme (Evergreen green palette via CSS vars), `#avisoCss` (the A4
  arrival-notice layout — fixed 794px width, mm units for print), and `#aviso-pdf-fix-css`.
- **Main IIFE** (`<script>` starting ~line 372): the whole application. Most of it hangs off a
  single `state` object (`sheets`, `baseSheet`, `records`, `d` = data-view state, `f` = active
  filters); the Demoras tab keeps its own `dem` object.
- **Second IIFE** (`#aviso-pdf-fix-js`, ~line 805): defines `window.avisoPdfFix`, an
  alternative PDF exporter. The visible "⬇️ PDF" button is wired to it via inline
  `onclick="avisoPdfFix(event)"` (line ~349), *not* to `#btnPdfAviso`'s own handler. So there
  are two PDF code paths — `btnPdfAviso`'s html2pdf path and `avisoPdfFix`'s
  html2canvas→jsPDF "fit to one A4 page" path. `avisoPdfFix`'s `findAvisoSource()` picks the
  element to rasterize; `#avisoPage` carries `data-aviso-pdf` so it matches the explicit
  selector first — without that marker the function falls back to a text heuristic that grabs
  a far-too-large ancestor (dashboard card, toolbar, clause note all end up in the PDF).

## Data pipeline

`handleFile` → `ingest` → `buildRecords` → `aggregate` → `refresh*` renderers.

- **`ingest(wb)`**: converts every sheet to `{header, rows}`. Auto-detects the base data sheet
  as the first sheet whose headers (lowercased) contain both `pod` and `20sd`.
- **`buildRecords()`**: maps base-sheet columns into record objects **by lowercased header
  text** (`H[header.toLowerCase()] → column index`). This is the main fragility: renaming a
  column in the source XLSX silently drops that field. Expected columns include: `bl`/`ir`,
  `contenedores` (container numbers, comma/`;`/`|`/newline-separated → record field
  `contNums`), `consignatario`, `pod`, `pol`, `servicio`, `a/c`, `20sd`, `40sd`, `40sh`,
  `nave principal`, `viaje`, `vslvoy`, `feeder`, `arribo arica`, `arribo iquique`, `eta pod`,
  `emision`, `emision destino`, `onward inland routing`.
- Derived per record: `cont(r)` = 20+40sd+40sh (a function — distinct from the `contNums`
  string field); `teus(r)` = 20 + (40sd+40sh)*2;
  `trade` = `"WS"` if POL contains `BUENAVENTURA` else `"FE"`; `dest` via `normDest()` which
  maps the `a/c` field's country prefix (BO→BOLIVIA, CL→CHILE, etc.).
- `parseDate()` handles Excel serials, Date objects, and several string formats; two-digit
  years are coerced to 20xx.
- Filters in `state.f` (POD / service / destination / month / vessel) are applied by
  `filteredRecords()`, consumed by every dashboard renderer. Month `__transit` = records with
  no ETA.

## Aviso de Llegada generation

`buildAviso(r)` returns an HTML string for one arrival notice. The two `.tbl-rec` charge
tables are kept to 5 columns; the bank / account-holder info goes in a separate compact
`.tbl-bank` strip table right below each — deliberately **not** `rowspan` cells, because
html2canvas (used by every PDF/ZIP export path) misrenders `rowspan` in `border-collapse`
tables (shifted text, phantom green header bleed). Keep new columns out of spanned cells.

Business rules are **hardcoded** here and must be edited in place:

- Fee formulas, e.g. `thc = 160*n20 + 230*n40`, `vbF = 120*n20 + 200*n40`, admin/BL fees.
- Deadlines are **business days** (`addWorkingDays`, skips Sat/Sun): `arribo + 15 working
  days` = trámite deadline; `arribo + <demFreeDays()> working days` (default 21) = container
  return date. `demFreeDays()` reads the `#demFree` input on the Demoras tab, so changing the
  free-days there also moves the Aviso's devolución date — kept intentionally consistent with
  the Demoras calculator.
- Bank details (Banco BCP Bolivianos account, Banco Santander Chile), Everbol Shipping
  S.R.L. NIT, contact emails, and the Santa Cruz office address are literal strings in the
  template.
- The "premisa" line keys off whether `onward inland routing` mentions `BOLIVIA`
  (→ "CARGO IN TRANSIT TO BOLIVIA").

### PDF export

Both PDF paths share `renderSheetToPdf(el)` in the `#aviso-pdf-fix-js` IIFE: clone the sheet
→ `.aviso-pdf-sheet` → html2canvas → one jsPDF A4 page scaled to fit inside `PAGE_MARGIN_MM`.

- Single "PDF" button → `window.avisoPdfFix()` on the live `#avisoPage` (`[data-aviso-pdf]`),
  saved as `<BL>.pdf` (`avisoFileName()` reads `#blInput`).
- `btnZipAviso` (📦 ZIP masivo): with a vessel picked in `#selNaveAviso` and no BL, it builds
  `buildAviso(r)` for every record whose `r.vsl` matches, renders each via
  `window.avisoSheetHtmlToPdf(html)` (same pipeline, off an offscreen div), and packs
  `<BL>.pdf` files into `AVISOS_<NAVE>.zip`. Sequential; ~2–3 s per aviso.

The old html2pdf path (`btnPdfAviso` listener, `pdfOptions()`) is dead code — the visible
button's inline `onclick="avisoPdfFix(event)"` calls `stopImmediatePropagation()` first.

## Onward-routing clause check

`clauseStatus()` / `refreshOnward()` validate that BOLIVIA-destined records carry a
"CARGO IN TRANSIT TO BOLIVIA" (or Spanish/partial variants) clause in `onward inland routing`,
flagging each row ok / review / missing.

## Demoras (demurrage/detention) — `🧮 Demoras` tab (`#calcView`)

Per-container overdue-days + cost calculator. State in `dem = {rows, totals, bl}`.

- `btnCalcLoad`: looks up the BL, splits `r.contNums` into container numbers, and creates one
  `dem.rows` entry per 20' and per 40' container (`{num, tipo, desc, dev}`), `desc` =
  discharge date (arrival at POD).
- `calcRow(row)`: `limit = addWorkingDays(desc, demFreeDays())`; `demora = max(0, calendar
  days between limit and (dev || today))`; `cost = demora * demRate(tipo)` (`demRate20`/
  `demRate40` inputs, default 55/110). Status: EN PLAZO / POR VENCER (≤5 d) / VENCIDO /
  Con demora / Devuelto a tiempo.
- `renderCalc()` builds the editable table (inline `data-i`/`data-f` inputs, event
  delegation, re-renders on every change) and the stat chips; empty `dem.rows` shows a hint.
- `btnCalcInv`: opens a print window with a "DEM-<BL>" invoice — line per container, subtotal,
  `+8% + USD 35` transfer fee, total, and hardcoded Perú (BCP USD) + Bolivia (BCP Bs) bank
  details.
- `escAttr()` / `money()` are Demoras-local helpers; date helpers `fmtD` / `isoLocal` /
  `addWorkingDays` / `calDays` live with the other top-of-IIFE helpers and are shared with
  `buildAviso`.

## Conventions

- ES5-style code throughout the main IIFE (`var`, function expressions, no arrow functions);
  the pdf-fix IIFE uses modern syntax. Match the surrounding style when editing each.
- `$(id)` is `document.getElementById`; `el(tag, class, text)` builds elements.
- Editing is direct DOM string building + `makeTable()`; there is no framework or templating.
