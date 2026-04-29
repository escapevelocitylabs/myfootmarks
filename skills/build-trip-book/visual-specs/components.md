# Component Inventory — v3 locked

Extracted from the v3 prototype source. Components are organized by where they appear in reading order. Each entry lists its variants, states, and any `data-variant` attribute that modulates rendering.

---

## Required components per doc (Stage 4 must-haves)

The components below MUST appear in every Stage 4 render of the named doc. The detailed component specs continue in the sections that follow; this list is the must-have summary.

**`trip-itinerary.html` — required components:**

- `.field-card` — MUST appear on every named-place timeline stop. Variants from the v2 component spec (Free-entry, Event, Pre-booked, No-address) all emit one card per stop; they only change row content.
- `.weather-strip` — MUST appear once in the time-primer section, containing one `.wx-day` card per trip day.
- `.kpi-row` — MUST appear at the top of every `.day-spread`.
- `.lookup-page` — MUST appear at least twice in the lookups section: once wrapping the DOW lookup (`.dow-grid`), once wrapping the weather lookup (`.weather-cols`). Each `.lookup-page` carries an `.lp-num` (e.g. `01 of 02`), `.lp-title`, and italic `.lp-hint`.
- `.evt-row` — MUST be used to render every dated event in the time-primer's `.evt-list`.
- `.evt-grid` — MUST appear once in the time-primer section to show recurring events Mon→Sun in a 7-column grid.
- `.fest-card` — MUST be used to render every festival in the time-primer section.
- `.ttk` / `.ttk-row` — MUST be used to render every "Things to know" entry in the time-primer section.
- `.cover-rail`, `.cover-schedule`, `.cover-meta` — MUST appear on the itinerary cover (see "Three distinct per-doc covers" → Variant 1).
- `.emergency-rail` — MUST appear three times in the back cover, one panel each for Emergency / Lodging · Home base / Transport.

**`trip-checklists.html` — required components:**

- `.cover-rail`, `.urgency-summary`, four `.urgency-bucket` blocks — MUST appear on the cover (see "Three distinct per-doc covers" → Variant 2).

**`trip-book.html` (keepsake) — required components:**

- `.cover-rail`, `.cover-hero` (with hard-offset accent shadow), big-title, italic-serif subtitle, `.cover-meta`, `.attribution` — MUST appear on the keepsake cover (see "Three distinct per-doc covers" → Variant 3).

See `rendering-checklist.md` (sibling file) for the full Stage 4 contract, including the self-validation checks the agent runs after writing each HTML file.

## Fixed controls (persistent UI)

### top-nav
Section-navigation rail, top-left.
- **Position**: fixed, top 14px, left 14px
- **Variants**: single variant
- **States**: none (hover on anchors only)
- **Scroll**: internal scroll at max-height 85vh
- **Mobile**: hidden below 720px
- **Selectors**: `.top-nav`, `.top-nav h4`, `.top-nav a`

### state-switcher
Two-dropdown panel for scenario + length state swapping, top-right.
- **Variants**: single variant (always present)
- **States**: carries `data-state` and `data-length` on `<body>` — CSS selectors cascade
- **Scenarios** (scalar): `happy`, `no-reservations`, `no-hours`, `thin-primer`, `zero-research`
- **Lengths** (scalar): `5-day`, `1-day`
- **V3 badge**: `.badge-v3` pill — version indicator
- **Shadow**: hard offset `4px 4px 0 accent`
- **Mobile**: docks to bottom-full at 720px

### walkthrough-panel
Bottom-right CTA that opens a menu of guided tours.
- **Variants**: single
- **States**: `menu.hidden` when collapsed, expanded when open, `isPlaying` during tour
- **Uses `@ourroadmaps/web-sdk`** from CDN for cursor overlay + annotations

---

## Cover (section 01)

### cover
Full-height opening page.
- **Parts**: `.cover-top` (masthead strip), `.big-title`, `.kicker`, `.hero`, `.credits`
- **Title treatment**: accent letter inside title (`span.accent` on the `o` of LISBON)
- **Hero**: gradient placeholder with Commons attribution inset
- **Credits grid**: 4-column KV pairs (destination / dates / travellers / home-base)

### masthead
Horizontal strip used as section separator.
- **Layout**: `border-top 2px / border-bottom 1px`, flex-between with left + right labels
- **Right label**: accent-red section number (`01`, `01b`, `02`, …)
- **Variants**: used in every top-level section

---

## Destination Primer (section 01)

### primer.full-primer.body
Two-column prose body (the explicit "sustained prose" exception to single-column default).
- **First-letter**: drop cap — 72px sans-800 accent color
- **Column count**: `2`, gap `40px`, max-width `880px`

### primer.neighborhoods
Stacked list of neighborhood sketches. One per home-base region.
- **Entry layout**: grid `180px | 1fr` — left: name + character chars · right: prose + anchors
- **`.hood-chars`**: eyebrow-style character summary (e.g. "Moorish · Vertical · Fado")
- **`.anchor` lines** (two per neighborhood):
  - `<b>Walk this</b>` — typographic route suggestion
  - `<b>Pairs with</b>` — suggested day-use pairing
- **Variant**: single-column when data is thin (no specific `data-variant`; renders same)

### primer.pullquote
Editorial pull-quote block.
- **Treatment**: sans-800 42px, left border accent 4px, padding-left 28px
- **Inline highlight**: nested `<span>` in accent color

---

## Time-specific Primer "When you're there" (section 01b)

### weather-strip
5-day forecast horizontal grid.
- **Layout**: `grid-template-columns: repeat(5, 1fr)`, gap 14px
- **wx-day entry**: date chip (accent) · icon · temp · note
- **Mobile**: 2-col below 720px, 3-col below 900px

### sub-head
Reusable h3 treatment for sub-sections within Time-primer and Lookups.
- **Style**: sans-800 14px uppercase, 0.2em tracking, accent-red border-bottom

### events-section (three sub-patterns)

**evt-dated** (one-off events)
- Layout: `grid 120px | 100px | 1fr` (date | tag | body)
- Date in accent 800-20px with day-of-week subtitle
- `.tag` on ink background (white text)
- Renders as a vertical timeline row

**evt-recurring** (weekly grid)
- `grid-template-columns: repeat(7, 1fr)`
- One `.day` cell per weekday · `.dname` header · `.evt-chip` entries
- Each `.evt-chip` has background `accent-soft`
- Empty days render blank (no placeholder)

**evt-festival** (multi-day)
- Bordered box with `.fest-run` accent range
- `.fest-highlights` sub-grid (2-col) with labeled KVs

### things-to-know
Named-observation list, one observation per row.
- **Layout**: grid `150px | 1fr` — tag column + h4+p column
- **Tags**: `In season`, `Reduced hours`, `Free evening`, `Book ahead`, `Weather-sensitive`, `New opening`
- **Border**: `ink-3` top + bottom, `ink` between rows
- **Variants**: none — extensible tag set

---

## Quick Lookups (section 02) — 5 index pages

Each lookup page is a `.lookup-page` block with ink border, 36px padding, and a `.lp-head` with number + title + hint.

### lp-hoods (By Neighborhood)
- **Layout**: `grid 1fr 1fr` — neighborhoods sit in two columns
- **Entry**: `.lp-hood` with `.hood-note` italic subtitle, `.hood-group` sub-sections, `.hood-entry` rows with name + refs + lenses

### lp-dow (By Day of Week)
- **Layout**: `grid-template-columns: repeat(7, 1fr)`
- **`.dcol`** per day · `.today` variant highlights current day with accent-soft background + accent border

### lp-weather (By Weather)
- **Layout**: `grid 1fr 1fr` — sunny column / rainy column
- **`.lp-wcol h3.sunny`**: orange (#d97706)
- **`.lp-wcol h3.rainy`**: blue (#2563eb)
- **Entries**: name · sub-reason · accent-red page-ref

### lp-kids (With Kids / Without Kids)
- **Layout**: `grid 1fr 1fr`
- Entries carry `.ktags` meta line showing stroller/with-kids/kid-first flags

### lp-res (Reservations & Urgency)
- **Layout**: vertical flex stack of 4 `.lp-rgroup` blocks
- **Groups** (lead-time buckets): `Weeks ahead`, `Days ahead`, `Day-of`, `Walk-up`
- **Entry**: grid `1fr | 180px | auto` — name / when / ref

---

## Featured Places (section 03) — trip-use taxonomy

### cat-head
Top-level category header within Featured.
- **Layout**: flex baseline row, `cat-label` roman numeral + `cat-title` 40px + `cat-count` right-aligned
- **Groups used**: `I. Icons of Lisbon`, `II. Neighbourhood anchors`, `III. Events this week`

### cat-intro
One-paragraph italic serif intro below each cat-head.

### feat-card
Canonical place card.
- **Layout**: `grid 280px | 1fr` — visual column / body column
- **Variants**:
  - `data-variant="no-image"` → no hero, visual column collapses to horizontal map-only row
- **Parts**:
  - `.feat-hero` — aspect-ratio 3/2 gradient (variant a/b/c/d)
  - `.feat-map` — inline SVG (30×15 viewBox hairline map)
  - `.feat-attrib` — Commons attribution caption
  - `.feat-body h3` — place name (30px sans-800)
  - `.lenses` — badge row (see component below)
  - `.why` — editorial why (19px sans-600 with accent-3 left border)
  - `.layered-insights .lens-line` — per-lens insight row (label + serif body)
  - `.rows` — metadata rows
  - `.reserve` — typographic CTA

### lens badge (.lens)
Small pill indicating research-lens relevance.
- **Variants**:
  - default — ink border, ink text, white background
  - `.lens.primary` — accent background, white text (the "primary" lens the card was promoted under)
  - `.lens.hood` — ink background, white text (neighborhood label; always present)
- **Use**: one card typically shows 3–5 lenses

### age-fit chip row
Inside `.row.agefit .val`.
- **Three chips in fixed order**: Stroller · With kids · Kid-first
- **Variants per chip**:
  - default (empty) — neutral, capability absent
  - `.yes` — accent-soft background, accent-dark text, capability confirmed

---

## What to Eat (section 04) — dish-first food

### dish
Boxed entry for a signature dish.
- **Layout**: `.dhead` (name + origin label) + `.dbody` (serif story) + `.doptions` (auto-fit grid of `.dopt`)

### dopt (dish option)
Compact card for a restaurant/place that serves the dish.
- **Left border**: accent 3px
- **Parts**: `.dpname` (sans-700 14px) · `.dphood` (10px eyebrow) · `.dptier` (accent badge) · `.dpwhy` (serif) · `.dpref` (accent ref)

---

## Daily Plan (section 05)

### day-spread
Full-page day block.
- **Parts**:
  - `.kpi-row` — grid `auto | auto | 1fr | auto` (day-no · date · weather · meta)
  - `h2.display` — day title
  - `.narrative` — serif lede with `:first-line` sans-700 override
  - `.day-body` — grid `1fr 1fr` (timeline left · day-map right)
  - `.timeline-label` — sub-head style
  - `.tl-item` — grid `62px | 1fr` (time · body)
  - `.tl-ref` — accent-soft chip with page reference
  - `.day-map` — white border-ink box containing SVG
  - `.alternatives` — appended block after the day body (see below)

### alternatives
Per-day pivot catalog, slot-grouped.
- **Parts**:
  - `.alt-head` — uppercase sub-head
  - `.alt-slots` — vertical stack of slots
  - `.alt-slot` — accent-3 left border, `h4` slot label, `.slot-instead` ("Instead of X") with accent X highlight
  - `.alt-item` — grid `auto | 1fr` (pin · body)
  - `.alt-pin` — black square 22px with letter (a/b/c)

---

## Checklists (section 06) — one per page

### checklist-section
Per-list page-break container.
- **Background**: `#fff` (escapes paper tone for print fidelity)
- **Parts**:
  - `.chk-head` — flex row: left title column (`.chk-num` accent + `h2.chk-title`) · right `.chk-note`
  - `.chk-groups` — max-width 860 stack of groups
  - `.chk-group` — `h4` accent uppercase sub-head + `ul` with native checkbox labels
- **Three instances**: Packing · Pre-trip · Day-of

### checkbox row
Native HTML — `<label><input type="checkbox"> text</label>`.
- Accent-color on checkbox is ink (`accent-color: #000`)
- Grid `22px | 1fr`, gap 12px

---

## Appendix (section 07)

### appendix.entry
Compact long-tail row.
- **Layout**: `grid auto | 1fr`
- **Parts**: `.lead` (name + cat eyebrow) · `.loc` (serif contextual line)

---

## Reused patterns

### Page-ref chip (`.tl-ref`, `.ref`)
Small accent-colored page-number reference. Used in day timelines AND in every Quick Lookup entry. Signals "see canonical card at page X."

### Section eyebrow (`.eyebrow`)
11px sans-800 uppercase accent-red — appears below every section's masthead.

### Sub-head (`.sub-head`, `.timeline-label`, `.chk-group h4`, `.lp-hood .hood-group`)
14px (or 10–11px in compact contexts) sans-800 uppercase with 0.2em letter-spacing — the book's "section within section" label style.

---

## `data-reuses` annotations
*No existing-app components to reuse — this is a new artifact type with no predecessor in the repo. `data-reuses` attributes are omitted throughout. A future lock-in that iterates on an existing `build-document` HTML may populate these.*

---

## Component count summary

| Category | Count |
|---|---|
| Fixed controls | 3 (top-nav · state-switcher · walkthrough-panel) |
| Cover | 2 (cover · masthead) |
| Primer | 3 (body · neighborhoods · pullquote) |
| Time-primer | 4 (weather-strip · events-{dated,recurring,festival} · things-to-know) |
| Quick Lookups | 5 pages × shared patterns (lp-hood · lp-dow · lp-weather · lp-kids · lp-res) |
| Featured | 5 (cat-head · cat-intro · feat-card · lens · age-fit-chip) |
| What to Eat | 2 (dish · dopt) |
| Daily Plan | 2 (day-spread · alternatives) |
| Checklists | 1 (checklist-section — three instances) |
| Appendix | 1 (appendix.entry) |
| **Total distinct** | **~28 components** |

All components are inline-CSS styled, stateful via `body[data-state]` and `body[data-length]`. No JavaScript beyond the state switcher and the walkthrough SDK integration.

---

## v2 — New components for three-doc split

These four components are introduced by the three-file split (itinerary / checklists / keepsake) — see spec `docs/superpowers/specs/2026-04-24-three-file-trip-book-design.md`. The 28 components documented above are inherited unchanged.

### Field card — `.field-card`

Role: in-itinerary place essentials. Embedded inline inside each `.timeline-slot` that corresponds to a named place, indented to match the timeline's content column.

Appears in: `trip-itinerary.html` only.

Structure:
- Container `<div class="field-card">`: left-bordered 3px ink, paper-alt ground, sans family, 13px body.
- Fixed row set, in order: `Address`, `Open today`, `Cost`, `Get there`. Rows separated by 1px dotted rule in `rule` color; last row has no separator.
- Row labels: uppercase sans, 10.5px, `ink.mute` color. Row values: 13px sans, `ink.default` color.
- The `Open today` value is the only row value that uses `accent.dark` color with `font-weight: 600` — today-specific hours are visually distinguished.

Variants:
- **Default** — all four rows populated.
- **Free-entry** — `Cost` simplifies to "Free" or "Free · cash & card at vendors."
- **Event** — `Address` becomes `Route` (e.g., a parade route); `Open today` becomes a time window.
- **Pre-booked** — `Cost` becomes "Pre-booked · tickets required"; `Get there` notes walk time.
- **No-address** (ferry crossings, drives) — `Address` row omitted.

Invariants:
- Never carries narrative content (history, cultural framing, hero imagery) — that belongs to the keepsake's story card.
- Never contains an `href` pointing to another doc in the set.
- If a place is visited multiple times in one trip, the field card repeats in place — no cross-slot deduplication. Each visit may have different today-hours or purpose.

### Urgency-summary cover

Role: the first-page at-a-glance view of what's due and when, organized by lead-time buckets. Replaces the traditional magazine-cover hero image for the checklists doc.

Appears in: `trip-checklists.html` only (as the first page).

Structure:
- `cover-rail` (cross-cover identifier) at top — see "Three distinct per-doc covers" below.
- Big-title (`To do.` or equivalent) + short subtitle line.
- `urgency-summary` container with four `urgency-bucket` blocks stacked vertically:
  - Bucket 1: **Weeks ahead** (blocker-level tasks, 4+ weeks out).
  - Bucket 2: **Days ahead** (2–3 weeks out).
  - Bucket 3: **Night before**.
  - Bucket 4: **Day-of arrival**.

Per-bucket:
- Header: `<h3>` sans 800 uppercase 18px. Two-column grid: bucket name (accent-red) + count chip (ink, 28px). Border-top 2px ink, 12px padding-top.
- Ordered list of tasks. Each `<li>` is a two-column grid: task text (left, sans 14px ink) + due tag (right, uppercase sans 10.5px, `ink.mute`). 1px rule between list items.

Variants:
- **Full** (all four buckets populated) — default.
- **Sparse** (short trip) — some buckets may have just one or two tasks; header still shows count.

States: static (no interactive state).

### Three distinct per-doc covers

Role: each doc's first page must identify the doc at a glance and orient the reader to the doc's reading mode. All three covers share a `cover-rail` identifier strip at the top; body content below the rail differs per doc.

Cross-cover invariant (shared by all three):
- `cover-rail` at top: `<div class="cover-rail">` with two children:
  - `<div class="doc-mark">` — accent-red sans 800 uppercase 12px, letter-spacing 0.25em. Single-word doc type (`Itinerary` / `Checklists` / `Keepsake`).
  - `<div class="doc-desc">` — soft-ink sans 600 uppercase 12px, letter-spacing 0.18em. One-line purpose (`Bring this one with you · Day-by-day` / `Print this one · Cross off over time` / `Read this one ahead · Or leave it at the hotel`).
- Cover-rail bordered top + bottom with 3px ink, 16px padding top+bottom.

Variant 1: **Itinerary cover (tactical)** — for `trip-itinerary.html`:
- Body: `cover-rail` → big-title (trip destination, sans 800 clamp(84px, 14vw, 160px)) → subtitle paragraph (sans 600 18px, max-width 52ch) → `cover-schedule` (at-a-glance list of days with theme labels, 2-column ordered list) → `cover-meta` (dates · home base · travelers · weather band · pace).
- **No hero photo.** The itinerary cover is tactical orientation, not atmospheric.
- `cover-schedule` list items: each a 2-column grid: day label (`Sat 05·09`, accent-red sans 800 12px tabular-nums) + theme (ink sans 600 13px).

Variant 2: **Checklists cover (austere)** — for `trip-checklists.html`:
- Body: `cover-rail` → big-title (`To do.`, sans 800 clamp(72px, 13vw, 140px)) → subtitle paragraph → urgency-summary (4 buckets as defined above).
- **No hero photo.** B&W-safe. Only color comes from accent-red bucket-name labels, which collapse to black in print mode.

Variant 3: **Keepsake cover (evocative)** — for `trip-book.html`:
- Body: `cover-rail` → `cover-hero` (`<div class="cover-hero">` with `<img>` at 58vh, bordered 3px ink, hard-offset 4px/4px/0 accent shadow) → big-title (destination name, sans 800 clamp(100px, 17vw, 200px)) → italic-serif subtitle paragraph (serif italic 22px, max-width 54ch) → `cover-meta` (dates · home base · travelers · season) → `attribution` line (sans 9.5px `ink.mute`).
- **Hero image is required.** A keepsake cover without the hero reads as broken.

### Back cover — itinerary final page

Role: emergency + lodging + transport info in one place. The page to flip to when something goes sideways.

Appears in: `trip-itinerary.html` only, as the last page.

Structure:
- `bc-eyebrow` (accent-red sans 800 uppercase 11px, letter-spacing 0.25em): `Keep this one · Back cover`.
- `<h2>`: `In case you need it.`
- `section-lede` (serif 17px ink-soft, max-width 52ch): one-sentence framing.
- Three `emergency-rail` panels stacked vertically:
  - Panel 1: **Emergency** — 911, poison control, non-emergency police, pediatric urgent care, walk-in clinic, 24-hr pharmacy.
  - Panel 2: **Lodging · home base** — neighborhood, rental address, host contact, Wi-Fi, lockbox / key.
  - Panel 3: **Transport** — car rental, ferry notes, taxi, transit, float plane (or locale-specific equivalents).

Per-panel:
- Container: 3px solid ink border, hard-offset 4px/4px/0 accent shadow, paper ground, 24px padding.
- `<h3>`: sans 800 uppercase 14px, letter-spacing 0.22em, accent-red. 16px margin-bottom.
- Label-value rows: two-column grid — 160px label (sans 700 uppercase 11px `ink.mute`) + value (sans 15px ink). 1px rule between rows; last row no rule.

Invariants:
- Back cover is the densest-information page of the itinerary. No photos, no stories — just phone numbers, addresses, and contacts.
- Value strings are filled with real data at build time (from `book-data.json` → lodging + trip context). Unfilled slots render as `[from check-in]` / `[from booking]` style placeholders.

---

## v2 maps — tile mosaic + QR linkouts

These four components are introduced by the Stage 3 rewrite (Approach 3 — OSM tile composite + editorial SVG overlay + QR linkouts) — see spec `docs/superpowers/specs/2026-04-24-trip-book-maps-design.md`. They supersede v1's pure-vector line-art maps for all in-book map surfaces.

### Tile mosaic SVG — `.tile-mosaic-svg`

Role: the base layer of every map artifact. Real OpenStreetMap cartography fetched at build time and embedded as base64 PNG data URLs inside an SVG.

Appears in: every `.svg` file under `.myfootmarks/trips/<slug>/maps/` (day-overview, place-card inset, destination-level).

Structure:
- Outer `<svg viewBox="0 0 {cols*256} {rows*256}" preserveAspectRatio="xMidYMid meet">` — viewBox matches the tile mosaic's pixel dimensions exactly so the editorial overlay can project lat/lon into the same coordinate system.
- Inside the SVG: one `<image>` element per tile with `href="data:image/png;base64,..."`, positioned via `x` `y` `width="256"` `height="256"` (in viewBox pixel units).
- Below the tile mosaic `<image>` elements, the editorial overlay group `<g class="map-overlay">` carries pins, route, labels, home base.
- For day-overview and place-inset SVGs, the QR `<image href="data:image/png;base64,..." width="..." height="..." class="qr-day|qr-place">` is positioned in a margin slot OUTSIDE the tile mosaic but INSIDE the SVG so the artifact is fully self-contained.

Zoom levels:
- Day-overview: zoom 15 (4×3 mosaic for typical city day, ~3km spread).
- Place-card inset: zoom 16–17 (single 1×1 or 2×2 mosaic for tight crop).
- Destination-level: zoom 12–13 (broad city/region shape).

Invariants:
- The SVG is fully self-contained — no remote `href`s, no external CSS. Tiles + QR are base64-embedded.
- `preserveAspectRatio="xMidYMid meet"` — locks overlay to tile grid at any container width.
- Visible `Map data © OpenStreetMap contributors` attribution rendered as a `<text>` element near the bottom of the SVG. Required by OSM fair-use; never hide in HTML comments.

### QR — per place — `.qr-place`

Role: phone handoff for a specific place. Reader scans → Google Maps opens at that place's listing card.

URL pattern: `https://www.google.com/maps/search/?api=1&query={url-encoded "name, destination"}` with optional `&query_place_id={googlePlaceId}`. **Never** use `?query={lat},{lon}` — hides the listing card.

Generated by: WebFetch to `https://api.qrserver.com/v1/create-qr-code/?data={url-encoded url}&size=200x200&ecc=M`. Returns PNG. Embed as base64 data URL via `<image>`.

Embed pattern inside the place's inset SVG:

```html
<image href="data:image/png;base64,{base64}" x="..." y="..." width="80" height="80" class="qr-place"/>
<text x="..." y="..." class="qr-label">Scan for directions</text>
```

ECC level: `M`. Print physical size: ≥ 2cm wide.

Failure fallback: if the API call fails, the SVG slot becomes `<text class="qr-fallback">Search "{name}" on Google Maps</text>` instead of the `<image>`.

### QR — per day — `.qr-day`

Role: phone handoff for a whole day's stops. Reader scans → Google Maps opens with multi-waypoint directions.

URL pattern: `https://www.google.com/maps/dir/?api=1&origin={homeBase.lat},{homeBase.lon}&destination={homeBase.lat},{homeBase.lon}&waypoints={url-encoded "lat1,lon1|lat2,lon2|..."}`. **Always** use lat/lon waypoints — named-place waypoints are unreliable (chain-restaurant collisions, 5+ name truncation).

Generated via: same QR API, with `&size=240x240&ecc=L`.

ECC level: `L` (lower than place QR because waypoint URL is longer; `H` would push module count too high for photocopy survivability).

Print physical size: ≥ 2.5cm wide.

Failure fallback: `<text class="qr-fallback">Open Google Maps for today's stops</text>`.

### Destination-level editorial map — `destination.svg`

Role: the keepsake's atmospheric destination-level map. Conveys character and shape of the place. Read on a couch, in a plane, before the trip.

Appears in: `.myfootmarks/trips/<slug>/maps/destination.svg`, inlined into the keepsake's Destination Primer section.

Structure:
- Same tile mosaic + overlay pattern as day-overview, but at zoom 12–13 (broad city/region shape).
- Sparse overlay: NO route line, NO numbered pins. Optional small dots marking featured-place neighborhoods (when Overpass returns `place=neighbourhood|suburb|quarter|city_district` nodes in the bbox). NO QR.
- Optional editorial overlay labels for major neighborhoods, rendered in caps with wide letter-spacing, accent-red with white-halo stroke (used elsewhere in the keepsake's neighborhood-block spec).

Pad: bbox padFrac 0.25 (more breathing room than the day-overview's 0.18).
