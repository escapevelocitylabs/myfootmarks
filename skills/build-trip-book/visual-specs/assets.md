# Asset Inventory — v3 locked

Extracted from the v3 prototype source. Assets in the prototype are **placeholder-heavy by design** — the actual renderer (`build-trip-book`) fetches real images from Wikidata → Commons at generation time. This inventory distinguishes placeholder treatments from real-data slots.

---

## Hero images

### Cover hero
- **DOM**: `.cover .hero`
- **Placeholder treatment in v3**: CSS gradient `linear-gradient(135deg, #2e3b4e, #5a7189 45%, #c4873a)` with a darkening overlay and a warm highlight at the top-right.
- **Real-data slot**: the Wikidata P18 image of the destination entity (e.g., Lisbon Q597). Fetched at book-gen time, saved locally, embedded as `<img>` with alt text.
- **Attribution line**: `.hero-attrib` renders Commons credit + Wikidata entity id + P18 claim.
- **Category**: real-data-source

### Featured card hero (`.feat-hero`)
- **DOM**: one per featured card that carries an image.
- **Placeholder treatment in v3**: one of four pre-defined gradient variants (`variant-a` default, `variant-b` warm, `variant-c` olive, `variant-d` purple) with a two-layer overlay.
- **Real-data slot**: Wikidata P18 for each place's Wikidata entity (e.g., Q208786 Pena).
- **Attribution**: `.feat-attrib` row — Commons credit, license, optional photographer.
- **Category**: real-data-source
- **Degradation**: cards without a Wikidata entity (Tasca do Chico, Piriquita, Ramiro, Feira da Ladra, IndieLisboa, Freedom Day's Feira entry) render via `[data-variant="no-image"]` — no hero, horizontal map strip only. This is a first-class state.

---

## SVG maps (all inline, all drawn)

### Featured-card inset maps
- **DOM**: `.feat-map svg` — one per featured card.
- **Size**: `viewBox="0 0 300 150"` · aspect 2:1.
- **Style**: stroke-based black line-art on white — streets, water, optional buildings, and a red circle pin.
- **Variants in v3**: ~15 different map compositions drawn by hand (each card gets a different neighborhood-appropriate sketch).
- **Real-data slot**: generated from Overpass API queries at book-gen time. Source bounding box derived from each place's lat/lon ± 150m.
- **Category**: real-data-source
- **Note**: the prototype SVGs are hand-authored approximations. Production will use Overpass features (streets, water, buildings) rendered to SVG paths.

### Day-overview maps
- **DOM**: `.day-map svg` — one per day spread.
- **Size**: `viewBox="0 0 400 380"` · aspect ~1:1.
- **Style**: richer — streets, water, buildings (rectangles), compass rose, numbered pins for each timeline stop, dashed accent-red walking path, home-base marker.
- **Real-data slot**: Overpass data bounded by the day's visited coordinates (union of stop lat/lons + home-base).
- **Category**: real-data-source

### Events section maps (for events with location)
- Same structure as featured-card inset maps. Event pin in accent.
- **Category**: real-data-source (for events with a Wikidata-resolvable venue) or **fixed-ui** (for generalized event pins without a specific venue)

### Weather icons (unicode character glyphs)
- `☀` (sunny) · `☂` (rainy) · `☁` (overcast)
- **Category**: fixed-ui — these are character glyphs, not images. Font-rendered.
- **Real-data mapping**: `research-weather` → daily forecast → glyph mapping (sunny/partly-sunny/overcast/rain/snow/etc.).

### Compass roses (on day maps)
- Drawn inline: small circle with a North triangle and "N" label.
- Single consistent rendering across all day maps.
- **Category**: fixed-ui

---

## Icons

The prototype uses **zero icon assets**. All visual chrome is typography or drawn SVG. Where a conventional UI might use a chevron, plus-sign, or hamburger menu, v3 uses text or typography.

Specific cases:
- **Reservation CTAs** — use a trailing `→` (unicode glyph) via CSS `::after`.
- **State-switcher expand/collapse** — there is no collapse; the switcher is always open.
- **Walkthrough play/stop buttons** — inline SVG paths (`<svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor"><path d="M8 5v14l11-7z"/></svg>`).
  - Play triangle (Unicode `▶` used as text in v3)
  - Stop square (Unicode `■`)
- **Chip markers** — pure typography, no icons.

**Category**: fixed-ui — all glyphs are Unicode or inline SVG, not asset files.

---

## Gradient placeholders (cover + featured hero)

The v3 prototype defines 4 gradient variants for placeholder heroes:

| Variant | Gradient definition | Intended mood |
|---|---|---|
| default | `135deg, #3d4f65, #78859a 50%, #c8a070` | blue-to-amber (stone + light) |
| `variant-b` | `135deg, #8b5a3c, #c4873a 50%, #f4c878` | amber-to-honey (terracotta) |
| `variant-c` | `135deg, #2d3e3a, #5c7872 50%, #b8a878` | moss-to-sand (green) |
| `variant-d` | `135deg, #4a3b5e, #7b6993 50%, #d4b87f` | plum-to-sand (purple) |

All have a darkening overlay at the bottom and a warm highlight at the top.

**Category**: placeholder — these will be REMOVED in production. They exist solely because the prototype doesn't fetch Commons images. Production: the hero is a real `<img>` with the Commons photo.

---

## Fonts

### Display + Body families
- **Sans**: Inter (Google Fonts CDN or self-hosted)
- **Serif**: Source Serif 4 / Source Serif Pro (Google Fonts CDN or self-hosted)

### Loading strategy
- **v3 prototype**: system-font fallback chain only. No `@font-face` declarations; no CDN link. The stack resolves to system sans (system-ui) and system serif (Georgia/Charter) — a working approximation.
- **Production**: inline base64-embedded web-font files for self-containment, OR a `<link>` to Google Fonts with a preload strategy. Given the "no runtime dependencies" invariant, inline-base64 is preferred.
- **Category**: real-data-source (the production font file replaces the system-fallback stack)

### Font weights used
400 · 500 · 600 · 700 · 800 (500 only in a few spots; could be dropped if font weight loading is costly)

### Fallback chain
Documented in `tokens.json` under `font.family.sans` and `font.family.serif`.

---

## Sample-data strings

All the following are **placeholder content** generated at book-gen time per trip:

| Slot | v3 placeholder | Real-data source |
|---|---|---|
| Destination name | "Lisbon, Portugal" | `trip.yaml:destination` |
| Trip dates | "Thu 23 — Mon 27 Apr" | `trip.yaml:start_date`, `end_date` |
| Travellers | "Rita · Marco · Sofia (10) · Tiago (7)" | `trip.yaml:travelers` |
| Home-base | "Alfama" | `trip.yaml:home_base` |
| Pace | "moderate" | `trip.yaml:pace` |
| All place names | "Pena", "Castelo de São Jorge", etc. | research-skill frontmatter `places[].name` |
| All place `why` prose | "A 19th-century Romantic fantasia…" | research-skill markdown body (condensed) |
| All hours strings | "Daily · 09:30 – 18:30 · last entry 17:00" | research-skill `places[].hours_structured` |
| All lens-line prose | "Photogenic in mist — the Saturday rain forecast is actually a gift…" | research-skill markdown body (extracted per lens) |
| All weather strip days | "☀ 18° Partly sunny, breeze off the Tagus" | `research-weather` output |
| Event details (Freedom Day, etc.) | As specified | `research-events` output + Wikidata for venues |
| Dish names + stories | "Pastel de nata — Flaky puff-pastry…" | `research-local-foods` output |
| Dish options (dopt) | "Pastéis de Belém · Belém · $ · Icon · Standing" | `research-local-foods` `served_at:` cross-ref |
| Daily narratives | "Fly in tired. Let Alfama introduce itself…" | `build-itinerary` output |
| Timeline entries | "09:30 · Jerónimos · Cloisters first, church second." | `build-itinerary` output |
| Alternatives | Per-slot alts | new generation step (see PRD/Design) |
| Checklist items | "Ramiro · Sun 26 · 19:00 · party 4" | `build-checklists` + per-place `reservation_required` |
| All appendix entries | "Manteigaria · Food · Pastelaria · warm pastéis" | research-skills long-tail + `featured: false` |
| All page-refs (`p. 14`, `p. 20`) | Sample numbers | computed at render time from actual page breaks |

---

## Wikimedia Commons resolution pipeline

For each placeholder hero, the production pipeline does:

1. Look up the place's `wikidata_id` from its research-skill frontmatter.
2. Query Wikidata for `P18` (image) claim → filename on Commons.
3. Fetch the Commons file's thumbnail URL (appropriate size for cover = 1800×1200, for card hero = 600×400).
4. Download to `<slug>/assets/<place-id>.jpg`.
5. Capture the Commons metadata: author, license, source URL.
6. Replace the placeholder gradient with `<img>` pointing at the local asset.
7. Populate the `.feat-attrib` / `.hero-attrib` with author + license.

**Degradation**: if Wikidata lookup fails or P18 is empty, the card renders via `[data-variant="no-image"]` — no hero, no attribution row, horizontal map strip only.

---

## License / attribution inventory

All real-data assets will carry per-image attribution:

- **Format**: `Photo: <Title>, <Author> / Wikimedia Commons, <License>`
- **License types expected**: CC BY, CC BY-SA 2.0 / 3.0 / 4.0, CC0, Public Domain.
- **Render**: inline in `.feat-attrib` or `.hero-attrib` in `9.5px` sans uppercase letter-spaced caps.
- **Print**: retained; collapses to B&W.

---

## Asset file structure (production — not in prototype)

```
<slug>/
├── trip-book.html      # self-contained HTML (this template)
└── assets/
    ├── cover-lisbon.jpg       # real Commons image
    ├── castelo-sao-jorge.jpg  # real Commons image
    ├── jeronimos.jpg          # etc.
    └── attributions.json      # machine-readable attribution log
```

The prototype does not write `assets/`; production does.

---

## Category summary

| Category | Count | Notes |
|---|---|---|
| **real-data-source** | ~25 hero + map slots | Production fetches from Wikidata/Commons/Overpass |
| **placeholder** | 4 gradient variants | v3 only — removed at production |
| **fixed-ui** | ~10 (glyphs, compass, weather icons) | Stable across trips |
| **fonts** | 2 families × 5 weights | Self-hosted or CDN-loaded at production |

**Total placeholder count to replace at production**: every hero + every map.
**Total fixed UI assets**: glyphs only. No icon files.
