# Design Tokens — Modern-magazine voice · v3 locked

Extracted from the v3 prototype source at 2026-04-23. See `tokens.json` for the canonical DTCG format.

## Color

### Paper (backgrounds)
- `paper.base`    `#fafaf7` — primary paper ground (reading sections)
- `paper.alt`     `#f2f1ec` — Time-specific primer + What-to-Eat ground
- `paper.deep`    `#ebebe8` — Quick Lookups section ground; water fills in SVG maps

### Ink (text)
- `ink.default`   `#0b0b0c` — body text, primary headings, map strokes
- `ink.soft`      `#2a2a2c` — secondary body, lede paragraphs
- `ink.mute`      `#747479` — captions, metadata, lookup hints
- `ink.faint`     `#a8a8a4` — deep secondary

### Accent (editorial red)
- `accent.base`   `#d7362b` — section marks, primary lens badges, route lines, CTAs
- `accent.dark`   `#a6261c` — accent-on-accent-soft text (accessibility)
- `accent.soft`   `#fbe5e3` — pale accent ground (timeline-ref chips, today column, yes chips)

### Rule
- `rule`          `#dcdcd8` — horizontal separators and card borders

### Print mode
In `@media print`, `--accent` collapses to `#000` and `--accent-soft` to `#eee`. All color-bearing elements degrade to B&W-safe equivalents. Checklist sections retain full fidelity.

---

## Typography

### Families
- **Sans**  — Inter (fallback: Helvetica Neue, Helvetica, Arial, system-ui)
- **Serif** — Source Serif 4 / Source Serif Pro (fallback: Charter, Iowan Old Style, Georgia)

### Weights
- 400 body · 500 medium · 600 semibold · 700 bold · 800 display

### Sizes (in reading order)

| Token | Size | Usage |
|---|---|---|
| cover-big | `clamp(120px, 18vw, 220px)` | Cover title `LISBON` with `o` in accent |
| h2-display | `78px` | Primer / Time-primer / Lookups / Featured / Eat / Daily / Checklists |
| h2-day | `72px` | Day-spread title |
| h2-checklist | `68px` | Per-checklist page title |
| h2-appendix | `64px` | Appendix h2 |
| cat-title | `40px` | Featured trip-use group title (Icons / Anchors / Events) |
| lp-title | `40px` | Lookup page title |
| h3-primer | `34px` | Primer sub-headings |
| feat-card-title | `30px` | Featured card place name |
| dek | `20px` | Section dek (sans 400) |
| day-narrative | `19px` | Day lede (serif, first-line sans-700) |
| feat-why | `19px` | Editorial *why* on featured cards (sans 600) |
| body | `17px` | Book body (serif) |
| primer-body | `16.5px` | Primer prose body |
| card-body | `15px` | Card body text |
| tl-time | `16px` | Day-spread timeline time column |
| sub-head | `14px` | Sub-section head (sans 800 uppercase) |
| card-rows | `13px` | Card row content |
| lens-label | `9.5px` | Lens badges / attribution lines |
| eyebrow | `11px` | Accent-red section eyebrows |

### Letter spacing
- `tight` `-0.045em` (cover big-title)
- `tighter` `-0.035em` (day-spread h2)
- `display` `-0.03em` (cat-title, lp-title)
- `body` `0`
- `caps` `0.14em` (small-caps labels)
- `eyebrow` `0.22em` (most uppercase labels)
- `eyebrow-x` `0.25em` (cat-label on featured groups)

### Line height
- `display` `0.82` (cover)
- `display-2` `0.95` (h2)
- `tight` `1.05` (h3)
- `body` `1.55` (default)
- `primer` `1.65` (primer body)

---

## Layout

- `size.page-w` `860px` — primary reading column max-width
- `size.wide-w` `1080px` — wide container (featured, day, lookups, checklists)
- **Default: single column.** Two columns only for sustained prose (primer body).

### Spacing (section padding multiples)
- `x2 8px · x3 12px · x4 16px · x5 20px · x6 24px · x7 32px · x8 40px · x9 48px · x10 60px · x11 80px`
- Sections use `x11 / 48px` pattern for vertical/horizontal rhythm.

### Radii
- **No border-radius** anywhere. All corners are sharp. This is a deliberate voice choice — "magazine, not app."

### Borders
- `rule` `1px #dcdcd8` — faint separators between card rows
- `ink` `1px #0b0b0c` — card + lookup-page borders
- `ink-2` `2px #0b0b0c` — sub-head underlines
- `ink-3` `3px #0b0b0c` — category / alternatives / checklist dividers
- `accent-3` `3px #d7362b` — why-rule + alt-slot left borders

### Shadows (hard-offset, editorial)
- `switcher` `4px 4px 0 #d7362b` — state switcher + walkthrough menu
- `button` `3px 3px 0 #0b0b0c` — walkthrough CTA button

---

## Motion

- `motion.hover` `0.15s` — subtle hover transforms (catalogue cards rise `-3px`)
- No other transitions defined; the book is intentionally static.

---

## Print mode overrides

```css
@media print {
  body { background: #fff; color: #000 }
  .top-nav, .state-switcher, #walkthrough-panel { display: none }
  .checklist-section, .lookup-page { page-break-before: always }
  :root { --accent: #000; --accent-soft: #eee; --accent-dk: #000 }
}
```

The book is digital-first; checklists are the non-negotiable print case. Quick Lookup pages also print well because they're inherently tabular.

---

## Working-copy note

These tokens are inlined verbatim into the Design doc's `## Visual Specifications` section. Local `locked/extracted/tokens.json` and `tokens.md` stay for local tooling; the MCP-backed Design is the downstream source of truth.
