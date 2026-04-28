# Design System Narrative — v3 locked

Google Stitch 9-section format. Modern-magazine voice. Locked from prototype v3 on 2026-04-23.

## 1. Overall design language

The trip book is a **magazine** — not a web app, not a travel-service PDF. Its voice is editorial-modern: confident display type, generous white space, one strong accent color, documentary imagery with honest attribution, and drawn line-art maps rather than service screenshots. Every page should feel like it was composed, not assembled.

The modern-magazine register uses Inter for display and Source Serif 4 for body. It commits to hard-edged geometry (no rounded corners anywhere), hard-offset shadows instead of blurred drops, and a single editorial-red accent (`#d7362b`) as the only strong color. All other chromatic decisions are paper-whites, inks, and near-neutrals.

Reading-register invariants:
- **Single column** is the default for everything except sustained prose.
- **Page-references** for navigation, not deep links — the book reads the same on screen and in print.
- **Documentary imagery** only; travel-glossy is off-voice.
- **Maps: OSM tile composite + editorial SVG overlay** *(updated in v2)*. Real OSM cartography (zoom 12–17 depending on surface) carries the orientation job; the editorial overlay (numbered accent-red pins, dashed routes, white-halo label boxes, home-base markers) carries the design signature. QR codes generated via keyless `api.qrserver.com` and embedded as base64 PNG inside the SVG. Pure-skill — no `scripts/` directory, no npm deps. Self-contained artifacts, fully print-safe. Supersedes v1's pure-vector line-art-from-Overpass approach (Approach 1, rejected during prototype iteration).
- **Silent degradation** when data is absent — typography-only cards, suppressed sections, fallback strings.

## 2. Color

**Palette**: two paper tones, four inks, one accent, one accent-soft.

| Role | Value | Where |
|---|---|---|
| Paper base | `#fafaf7` | Primary ground (reading sections) |
| Paper alt | `#f2f1ec` | Time-specific primer + What-to-Eat ground |
| Paper deep | `#ebebe8` | Quick Lookups ground; SVG-map water fills |
| Ink default | `#0b0b0c` | Body + headings + map strokes |
| Ink soft | `#2a2a2c` | Secondary body |
| Ink mute | `#747479` | Captions + metadata |
| Ink faint | `#a8a8a4` | Deep secondary |
| Rule | `#dcdcd8` | Separators |
| Accent | `#d7362b` | Section marks + lens badges + CTAs + route lines |
| Accent dark | `#a6261c` | Accent-on-accent-soft text |
| Accent soft | `#fbe5e3` | Chip + today-column + yes-chip grounds |

**Color logic**: Accent is used sparingly to mark structure (eyebrows, dividers), to chip important metadata (yes-chips, page-refs), and to draw the eye through dynamic content (route lines on maps, CTAs). Accent is NEVER used for brand decoration.

**Accessibility**: Accent on paper passes WCAG AA for body text. Accent-dark is the accessible-on-accent-soft variant. No component relies on color alone — every state also has a typography or shape signal.

**Print**: Accent collapses to `#000`, accent-soft to `#eee`. The book reads correctly in pure B&W.

## 3. Typography

**Families**: Inter (sans, display + UI) and Source Serif 4 (serif, body).

**Mood**: Inter is used at 800 weight for display headings with tight negative letter-spacing. This is deliberately magazine-register (not SaaS-register) — headlines are heavy, confident, slightly italic-adjacent via tight tracking. Source Serif 4 carries reading passages; it's a modern humanist serif chosen for print + screen readability.

**Size system** is rhythm-based, not modular. Specific sizes are chosen for editorial purpose:
- Cover `clamp(120px, 18vw, 220px)` → the one "billboard" moment
- Section h2 `78px` → every top-level section except Day (72) + Checklists (68) + Appendix (64)
- Group head `40px` → Featured category groups + Lookup page titles
- Sub-head `14px` uppercase → shared sub-structure indicator
- Body `17px` serif · card-body `15px` · eyebrow `11px`

**Hierarchy signals**:
1. Size step (h2 vs. h3 vs. body)
2. Family switch (sans 800 vs. serif 400) — display moments
3. Case + tracking (uppercase 0.22em eyebrows vs. sentence-case body)
4. Color (accent-red for eyebrows + dek)
5. Rule lines (border-top 2px ink under sub-heads)

**Voice**: the book speaks in declarative present-tense, second-person-direct-address. "Get lost on purpose." "The original, unchanged since 1837." "Come at five, for the light."

## 4. Layout

**Two container widths**:
- `--page-w: 860px` — primary reading column (primer, lookup pages, checklists, appendix, dish cards, card bodies)
- `--wide-w: 1080px` — wide container (featured, day-spread, lookups section, checklists section — sections that can afford horizontal space)

**Grid pattern**: not a formal grid system. Most sections are centered single-column; specific components (day-body, lp-weather, lp-kids, cover credits) use fixed multi-column CSS Grids.

**Section rhythm**: vertical padding `80px 0 60px` or `80px 0` — these are the two conductor bars of the book.

**Page breaks**: `section { page-break-before: always }` plus explicit breaks on `.checklist-section` and `.lookup-page` — guarantees one-per-page print behavior for the dense sections.

**Reading order** (IA):
1. Cover
2. Destination Primer
3. Time-specific Primer ("When you're there")
4. Quick Lookups (5 reader-situational index pages)
5. Featured Places (grouped by trip-use: Icons / Neighbourhood anchors / Events)
6. What to Eat (dish-first)
7. Daily Plan
8. Logistics · Checklists (packing → pre-trip → day-of)
9. Appendix
10. Colophon

This order is design-deep: anticipation, orientation, reference, inspiration, food story, schedule, utility.

#### Per-doc reading order (v2 — three-doc split)

Starting with v2 (three-file split), the single IA list above is preserved for historical reference and for templates that elect to render one combined doc. The three-file default output uses three per-doc reading orders; each doc's subagent brief enforces its own order.

**`trip-itinerary.html`** (bring-along · tactical density):
1. Cover (dates + weather band + daily-schedule thumbnail)
2. Time-specific primer (weather strip + typed events this week + things-to-know-this-week)
3. Daily Plan (per-day sections, each with theme + weather band + timeline + per-slot field cards + per-slot alternatives + day overview map)
4. Quick Lookups — during-trip flavors only (By Day-of-Week, By Weather)
5. Back cover (Emergency · Lodging · Transport)

**`trip-checklists.html`** (print-and-cross-off · austere density):
1. Cover (urgency summary — four lead-time buckets)
2. Reservations & Urgency (ordered by lead time: weeks-ahead → days-ahead → day-of → walk-up)
3. Pre-trip checklist (own page)
4. Packing checklist (own page)
5. Day-of checklist (own page)

**`trip-book.html`** (keepsake · atmospheric density):
1. Cover (evocative destination hero)
2. Destination Primer (setting · history · culture · neighborhoods-at-a-glance · seasonal mood)
3. Featured Places — story cards grouped by trip-use (Icons · Neighborhood anchors · Events this week)
4. What to Eat (dish-first)
5. By-Neighborhood reference index
6. Quick Lookups — pre-trip flavors only (With Kids · Without Kids)
7. Appendix (researched long-tail places)

#### One voice, three densities

The three docs share the same template voice (tokens, typography, interaction patterns) but express three distinct reading densities:

- **Itinerary — tactical.** Dense, scannable, high information-per-inch. Field-card body drops to 13px sans.
- **Checklists — austere.** Strictly utilitarian. Sans-forward on urgency-summary cover; serif body on task lists. Native checkboxes, B&W-safe.
- **Keepsake — atmospheric.** Editorial pacing. Long-form sections with breathing room. Primer body is the ONLY section in the entire three-doc set that uses two-column layout.

## 5. Components

28 distinct components; see `components.md` for the full inventory with variants, states, and selectors. Key patterns:

- **Masthead strip** — reusable section-separator element (border-top 2px + border-bottom 1px) with left label + accent-red right label. Every section opens with one.
- **Eyebrow** — 11px sans-800 uppercase accent-red tagline below mastheads.
- **Sub-head** — 14px sans-800 uppercase with accent-red underline. The "section within section" indicator.
- **Lens badge** — 9.5px sans-800 uppercase pill. Three variants: default (ink border, ink text), primary (accent ground, white text), hood (ink ground, white text, for neighbourhood labels).
- **Age-fit chip row** — three chips (Stroller · With kids · Kid-first), each either neutral (empty) or `.yes` (accent-soft ground, accent-dark text).
- **Page-ref chip** — `.tl-ref` and `.ref` — small accent-red typographic reference used across timelines and lookup entries.
- **Hard-offset shadow** — `3px 3px 0 ink` or `4px 4px 0 accent` — hard geometric shadow, never blurred. A voice-signature element.
- **v2 maps (this iteration):** four additional components — tile-mosaic-svg, qr-place, qr-day, destination-map. See `components.md` "v2 maps — tile mosaic + QR linkouts" for full specs.

## 6. Motion

**Minimal.** The book is a reading artifact, not a reactive app.

- Catalogue card hover: 0.15s transform + box-shadow (only on the outer catalogue index, not in the book proper)
- Anchor navigation: browser-default (instant or smooth based on CSS preference)
- State-switcher swap: instant — CSS visibility cascade, no transition
- Walkthrough SDK: controlled by `@ourroadmaps/web-sdk`; smooth scroll + annotation fade

**No ambient animation.** No pulses, fades, carousels, or auto-advance. This is a deliberate voice decision — a keepsake book is not alive while it waits for you.

## 7. Do's and Don'ts

### Confirmed patterns (from iteration)
**Do:** Use editorial red (`#d7362b`) as the sole strong color. Never introduce a second brand hue.
**Do:** Treat accent-dark as the only accessible-contrast text on accent-soft grounds. Never use base-accent on accent-soft for body text.
**Do:** Ship every card with lens badges when multiple research lenses surfaced it. Never render a place twice across sections.
**Do:** Use single column everywhere. Two columns ONLY for sustained prose (primer body). Never use multi-column for lists, cards, or metadata.
**Do:** Render checklists one-per-page with `page-break-before: always`. Never use 3-column checklist grids.
**Do:** Use hard-offset shadows (3px/3px or 4px/4px, zero blur). Never use soft-blurred shadows — they read SaaS, not editorial.
**Do:** Use drawn SVG maps with black line strokes + accent-red pins. Never use tile-server screenshots or satellite imagery.
**Do:** Degrade silently on missing data. Typography-only cards are a first-class state.

### Rejected decisions (from the v1→v3 iteration log)
**Don't:** Two-column cards (two featured cards per row). *Why not:* v1 review found it cramped. v2 moved to full-width cards.
**Don't:** Three-column checklists. *Why not:* v1 review found them unprintable and hard to scan.
**Don't:** Categorise Featured by research-file taxonomy (`Places / Food / Activities / Events`). *Why not:* mirrors the researcher's silos, not the reader's questions. v3 replaced with trip-use taxonomy.
**Don't:** Mix dated events, recurring events, and multi-day festivals into a single list. *Why not:* readers use them differently; they need different layouts. v3 split into three patterns.
**Don't:** Treat Featured as the only place taxonomy. *Why not:* readers also need situational re-indexing (neighbourhood, day-of-week, weather, kids, urgency). v3 added Quick Lookups as a first-class section.
**Don't:** Duplicate place cards across research lenses (e.g., Butchart in attractions + outdoor + scenic as separate cards). *Why not:* the reader encounters the same place multiple times with different framings; confusing. v3 unified into one canonical card with lens badges and layered insight lines.
**Don't:** Hide time-sensitive facts inside individual cards. *Why not:* Things like "Sat 25 is a holiday with reduced hours" or "AGGV free Thursday only May 14" are reader-level gotchas that need to surface. v3 added "Things to know this week" sub-section.

## 8. Accessibility

- **Color alone never carries state.** Yes-chips have both accent-soft ground AND bold text weight; lens badges use text + border + ground combinations.
- **Native interactive elements** everywhere (`<a>`, `<button>`, `<select>`, `<input>`). No custom ARIA patterns.
- **Focus visuals** are browser-default; the book reads fine with keyboard navigation.
- **Maps** have `aria-label` per map describing the day or place.
- **Hero images** will carry meaningful `alt` text in production (describing the photograph, not "image of Lisbon"). Placeholders in the prototype use decorative `aria-label`.
- **Print fidelity**: the entire book reads correctly in pure B&W — accent collapses to black, accent-soft to pale grey. Checklists are the first-class print case and survive every viewport.
- **Text contrast**: body at `#2a2a2c` on `#fafaf7` = 13.4:1 (WCAG AAA). Accent at `#d7362b` on `#fafaf7` = 4.6:1 (WCAG AA). Accent-dark at `#a6261c` on `#fbe5e3` = 4.7:1 (WCAG AA).
- **Axe run deferred** — a full axe audit should run against production render with real data before ship.

## 9. Agent Prompt Guide

These are imperative instructions for the build LLM. Grounded in extracted token values and iteration decisions.

1. **Always** use `color.accent.base` (`#d7362b`) for section eyebrows, primary lens badges, page-refs, and route lines on day maps. Never substitute with a secondary brand hue — there is no secondary brand hue.
2. **Always** render place cards as ONE canonical card per place, with a `.lenses` row showing every research-lens badge that surfaced the place. Never duplicate a place across Featured groups; use references from other groups to the canonical card's page.
3. **Always** group Featured places by trip-use taxonomy — `Icons of [destination]` · `Neighbourhood anchors` · `Events this week`. Never group by research-file taxonomy (Places/Food/Activities/Events was explicitly rejected in iteration).
4. **Always** use 800-weight Inter with tight negative letter-spacing (`-0.03em` to `-0.045em`) for display h2 and larger. **Never** use Inter below 700 for body content — use Source Serif 4 instead.
5. **Always** render checklists one section per page with `page-break-before: always`. Never use a multi-column checklist grid — this was rejected in v1.
6. **When** multiple research files surface the same place, emit lens badges for each file and layered-insight lines in `.layered-insights` with per-lens prose excerpts. **Never** emit separate cards for the same place.
7. **When** an event has a type field (`dated` / `recurring` / `festival`), render it with the matching pattern: timeline row for `dated`, weekly grid chip for `recurring`, bordered mini-guide for `festival`. Never render all three types in a single list.
8. **When** a place lacks a `wikidata_id` or the P18 lookup fails, render the card with `data-variant="no-image"` (no hero, horizontal map strip only). This is a first-class state, not an error.
9. **Always** use single-column layout for everything except sustained prose. The only sanctioned two-column layout in the book is the primer body paragraphs. **Never** use multi-column for cards, lists, or metadata rows.
10. **When** rendering the Time-specific primer, always include the three sub-sections in this order: Weather-across-your-dates · Events-this-week · Things-to-know-about-this-week. Never skip "Things to know" — it consolidates time-sensitive facts that would otherwise scatter.
11. **When** rendering Quick Lookups, render all five index pages (By Neighbourhood · By Day · By Weather · With Kids/Without Kids · Reservations & Urgency) even if some pages are thin. A thin page is better than a missing page.
12. **Always** use hard-offset shadows (`3px 3px 0 ink` or `4px 4px 0 accent`), never soft-blurred shadows. The editorial register depends on this.
