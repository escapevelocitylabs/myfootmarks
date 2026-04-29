# Rendering checklist

Stage 4 (`render-html`) MUST satisfy every item below before writing the corresponding HTML file. This document consolidates the must-haves; the spec at `docs/superpowers/specs/2026-04-28-trip-book-html-regression-fix-design.md` carries the full rationale.

## Cross-doc invariants

- Dates never render as raw ISO timestamps. Acceptable formats: `May 9 – 20, 2026`, `Sat · 9 May`, `9 May`, `Sat 05·09`, `2026-05-09/2026-05-17` (date-range chips for festivals only).
- No `Invalid Date` substring anywhere in the rendered output.
- No empty `tl-why`, `tl-name`, or `field-card` row values in the itinerary. Missing source data is reported to the user, not silently elided.

## `trip-itinerary.html`

1. `<style>` block contains `visual-specs/itinerary-canonical.css` verbatim. Read the canonical CSS, copy its full contents into `<style>...</style>`. Do not paraphrase, reorder, or simplify rules. Do not omit selectors that appear unused — the markup vocabulary depends on the full set.
2. **Cover** has:
   - `cover-rail` (`doc-mark="Itinerary"` + `doc-desc="Bring this one with you · Day-by-day"`).
   - Big-title with the destination word + accent-colored period (e.g. `Vict<span class="accent">o</span>ria.`).
   - Subtitle paragraph (sans 600 18px, max-width 52ch).
   - `cover-schedule` — one `<li>` per trip day, 2-column grid: day label (`Sat 05·09`) + theme.
   - `cover-meta` — Dates, Home base, Travelers, Weather band. Dates as `May 9 – 20, 2026` (never raw ISO).
3. **Time-primer section** has masthead strip, eyebrow + `h2.display` + dek + sub-heads, AND:
   - `weather-strip` with one `wx-day` card per trip day. Each card has `dchip` (e.g. `Sat 05·09`), `icon` from `⛅ ☁ ☂ ☀ ☂`, `temp` (e.g. `17° / 8°`), `note` (one-line summary).
   - `evt-list` for dated events — 3-col grid: date column with day name, ink tag chip (e.g. `concert`, `parade`), h4 + body.
   - `evt-grid` for recurring events — 7-col Mon→Sun grid.
   - `fest-card` blocks for festivals — date-range chip + h4 + body.
   - `ttk` rows for "Things to know" — tag column + h4 + body, ink rules between rows.
4. **Daily section** — every day-spread has:
   - `kpi-row` — 4 cells: `Day NN`, `Sat · 9 May`, weather-with-icon-and-summary, pace meta.
   - `h2.day-title` — short (day-of-week + date + optional parenthetical, e.g. `Saturday, May 9 (Arrival)`). NOT the full narrative.
   - `day-narrative` paragraph — separate from h2.
   - Two-column `day-body` grid — timeline left (1.15fr), day-map-wrap right (1fr).
   - Every `tl-item` has `tl-time`, `tl-name`, `tl-hood`, **non-empty `tl-why`**, and a `field-card` with the four rows `Address` / `Open today` / `Cost` / `Get there`. Field-card variants from `components.md` (Free-entry, Event, Pre-booked, No-address) are the only allowed deviations — they change row content but still emit one card per stop.
5. **Lookups section** has masthead + eyebrow + `h2.display` + dek, plus TWO `lookup-page` containers:
   - DOW lookup — 7-col `dow-grid` with Open/On/Closed lists per day.
   - Weather lookup — `weather-cols` with sunny + rainy columns, each entry has neighborhood subtitle.
   - Each `lookup-page` has `lp-num` (`01 of 02`), `lp-title`, italic `lp-hint`.
6. **Back cover** has three stacked `emergency-rail` panels:
   - Emergency (911, poison control, non-emergency police, pediatric urgent care, walk-in clinic, 24-hr pharmacy).
   - Lodging · Home base.
   - Transport.
7. `book-footer` with the standard tagline.

## `trip-checklists.html`

1. `<style>` block contains `visual-specs/checklists-canonical.css` verbatim.
2. Cover follows the **urgency-summary** pattern: four `urgency-bucket` blocks (Weeks ahead / Days ahead / Night before / Day-of arrival).
3. Body contains every component documented in `components.md` for the checklists doc.

## `trip-book.html`

1. `<style>` block contains `visual-specs/keepsake-canonical.css` verbatim.
2. Cover follows the **evocative** pattern: `cover-rail` + `cover-hero` (image with hard-offset accent shadow) + big-title + italic-serif subtitle + cover-meta + attribution.
3. Body contains every component documented in `components.md` for the keepsake doc.

## Self-validation contract

After writing each HTML file, the agent runs the following checks **on itself** using Read/Grep — no external script.

### Substring / pattern guards (all three files)

- `Invalid Date` — forbidden.
- ISO timestamp pattern (regex `[0-9]{4}-[0-9]{2}-[0-9]{2}T`) — forbidden in rendered text. Catches `T00:00:00.000Z` and all timezone variants.
- `>undefined<` — forbidden.
- `[object Object]` — forbidden.
- `>NaN` — forbidden.

### `trip-itinerary.html` structural checks

- `<style>` block ≥ 400 lines.
- `weather-strip` substring present.
- count of `<div class="wx-day"` equals trip-day count from `trip.yaml`.
- `kpi-row` substring present in every day-spread.
- `lookup-page` substring present at least twice (DOW + weather).
- `emergency-rail` substring present.
- `cover-schedule` substring present.
- count of `<div class="tl-item"` equals count of `<div class="field-card"` (1:1).
- No `<div class="tl-why"></div>` or whitespace-only `tl-why`.
- No `<p class="day-narrative">` whose text equals the preceding `<h2 class="day-title">` text. (This is a Read-and-compare check — read the file, walk paired elements, string-compare. Not a regex.)

### `trip-checklists.html` structural checks

- `<style>` block ≥ 250 lines.
- `urgency-bucket` substring present.

### `trip-book.html` structural checks

- `<style>` block ≥ 400 lines.
- `cover-hero` substring present.

### Self-correction loop

If any check fails: describe which check failed, re-emit the affected file with the fix, re-run validation. Cap at **3 self-correction passes per file**; on the 4th attempt stop and report unresolved violations to the user. The run-log entry records `"status": "validator_failed"` with the violation list.
