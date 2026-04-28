# Interaction Catalog — v3 locked

Extracted from the v3 prototype source. The book is an intentionally-static reading artifact; interactions are sparse by design. Every interaction listed below is actually-implemented in the v3 source.

---

## Reading interactions

### Scroll
- The book is a single long scrollable document. There is no pagination scroller, no virtual list, no lazy loading.
- Every section is anchored (`#cover`, `#primer`, `#time-primer`, `#lookups`, `#lp-neighborhood`, `#lp-dow`, `#lp-weather`, `#lp-kids`, `#lp-res`, `#featured`, `#featured-hoods`, `#featured-events`, `#eat`, `#day-1`, `#day-2`, `#packing`, `#pre-trip`, `#day-of`, `#appendix`). Anchors are linkable from the top-nav and from the walkthrough SDK.

### Anchor navigation (`.top-nav a`)
- **Hover**: text color transitions to `accent.base` (no timing specified; browser default).
- **Click**: native anchor scroll (instant in CSS; smooth if `behavior: smooth` is applied by the scripted walkthrough).

### Catalogue-style card hover (catalogue index page only — outer `source/index.html` in multi-option modes)
- `transition: transform 0.15s, box-shadow 0.15s`
- On hover: `transform: translateY(-3px)`; box-shadow deepens
- Not used in the main book (v3 uses the standalone template, not the catalogue wrapper)

---

## State switcher

### `#state-scenario` and `#state-length` dropdowns
- **Change event**: JavaScript handler updates `document.body.dataset.state` and `document.body.dataset.length` scalars.
- CSS selectors cascade off `[data-state="..."]` / `[data-length="..."]` on the body.
- **No animation** — the state swap is immediate.

### State-driven visibility
CSS rules that toggle on body state:
- `[data-length="1-day"] .day-spread:not([data-day="1"]) { display: none }` — hide days 2+
- `[data-state="zero-research"]` hides `.featured`, `.lookups`, `.eat`, `.appendix`, most primer sub-blocks, Time-primer events + things-to-know, and all `.alternatives` blocks
- `[data-state="zero-research"] .primer .thin-primer { display: block }` shows the thin-primer fallback
- `[data-state="thin-primer"]` is the same but keeps everything except the thin-primer fallback

### Label updates
- The `#state-label` text re-renders each time a dropdown changes, with format: `{scenario label} · {length label}`

---

## Walkthrough SDK (`@ourroadmaps/web-sdk`)

The prototype loads the SDK from CDN (`https://unpkg.com/@ourroadmaps/web-sdk`) and wires a bottom-right CTA to play scripted tours.

### Walkthrough panel
- `#walkthrough-toggle` — CTA button. Red ground, black 2px border, hard-offset 3px/3px shadow.
- On click: toggles `#walkthrough-menu.hidden`.
- When playing: toggle becomes `■ Stop`; click stops the overlay.

### Walkthrough menu
- Appears above the toggle. White ground, black 2px border, hard-offset shadow in accent color.
- Each entry is a vertical stack of `.wt-title` + `.wt-desc`.
- Hover on entry: `background: accent.soft`.
- Click entry: invokes `playWalkthrough(key)`.

### Walkthrough script (`playWalkthrough`)
Sequence for each walkthrough:
1. Hide the menu.
2. Set playing state and button.
3. `overlay.play(script)` from the SDK.
4. On completion, restore the button.

The v3 prototype defines one walkthrough:
- **v3-tour** — 5-step tour annotating the new v3 patterns (Neighborhoods-at-a-glance → Things to know → Quick Lookups → lens badges → What to Eat).

### Script action types (used by the SDK)
- `scrollTo { target: { selector } }` — smooth scroll to the element.
- `wait { duration }` — pause (ms).
- `showAnnotations { annotations: [{ type: "label", text, target, position }] }` — render pointer annotation.
- `hideAnnotations` — dismiss.

### Script timing conventions in v3
- Wait after scrollTo: 1000–1200 ms (let the reader register the viewport change).
- Wait during annotation display: 3000 ms (enough to read one short label).
- No click / type / select actions used in v3 (no forms).

### Fallback
If the SDK fails to load, `playWalkthrough()` falls back to a `document.getElementById('cover').scrollIntoView({behavior:'smooth'})` and closes the menu — the book still works without the SDK.

---

## Outside-click-closes pattern
Both `#walkthrough-menu` and `#walkthrough-panel` close when any click lands outside them:
```js
document.addEventListener('click', (e) => {
  if (!e.target.closest('#walkthrough-panel')) menu.classList.add('hidden');
});
```

---

## Checklists

### Native checkboxes
- Standard HTML `<input type="checkbox">`.
- `accent-color: #000` (ink) — checked state is black, visible both digital and printed.
- No JavaScript — toggling is native browser behavior.
- State persists for the session but not to storage (intentional — checklists are ephemeral per reading).

### Label click
- Native label-for-checkbox pattern: clicking the label text toggles its checkbox.
- Grid is `22px | 1fr`; hit target is the full row.

---

## Reservation CTAs (`.reserve a`)
- Simple `<a href="#">` links (placeholder).
- Hover color: inherits (no hover rule); `text-decoration: none` baseline.
- Style includes a trailing `→` glyph via `::after`.
- In production, these resolve to actual reservation URLs from `reservation_url` on each place card.

---

## Source links (`.source a`)
- Underlined (`text-decoration: underline`).
- No hover transitions.
- Production: resolve to the Wikidata / Commons / official source URL captured during research.

---

## Page-reference chips (`.tl-ref`, `.ref`)
- Static typography (no hover, no click interaction).
- In a production HTML→PDF rendering, these would be internal anchor links that resolve to the canonical card's page number.
- In the v3 web prototype they are decorative typographic chips.

---

## Image "interactions"
- None. Hero placeholders are CSS gradients; they carry no click/zoom/lightbox behavior.
- When real Commons images are fetched at build time, they are rendered as `<img>` elements with native browser behavior (right-click-save, drag, etc.). No custom lightbox is planned.

---

## Map interactions
- None. Inline SVG maps are static.
- They do not zoom, pan, or respond to pointer events.
- This is deliberate — the map is a drawing, not an interactive widget.
- A future enhancement could add simple pin tooltips on hover; v3 does not include this.

---

## Print-mode behavior

### `@media print`
- `.top-nav`, `.state-switcher`, `#walkthrough-panel` → `display: none`
- Body background forced to `#fff`, text to `#000`.
- `--accent` collapses to `#000`; `--accent-soft` to `#eee`.
- `.checklist-section` and `.lookup-page` get `page-break-before: always`.

### Implications
- Reservation CTAs render as text (without the accent-red glyph emphasis).
- Lens badges render but all colors collapse — no visual distinction between primary-accent and default-ink lenses.
- Day maps render as line-art B&W — unchanged, since SVG is already black strokes.

---

## Motion / animation summary

| Interaction | Duration | Easing | Scope |
|---|---|---|---|
| State-switcher swap | instant | — | CSS visibility cascade |
| Walkthrough cursor move | `scrollTo` duration varies | SDK-default | SDK-controlled |
| Walkthrough annotation in/out | SDK-default | SDK-default | SDK-controlled |
| Catalogue card hover (catalogue only) | `0.15s` | default | `transform + box-shadow` |
| Anchor navigation | instant | — | browser-default |

**The book has no ambient animation.** Nothing pulses, fades, rotates, or auto-advances. This is a first-class voice decision — a keepsake book is not alive while it waits for you.

---

## Keyboard

### Focus
- Native browser focus visuals apply to `<a>`, `<button>`, `<input>`, `<select>`.
- No custom focus ring is defined — we inherit the browser default.
- No focus trap anywhere.

### Tab order
- Natural DOM order. State switcher is near the top of the DOM so Tab from the top-nav reaches it early.
- Walkthrough toggle sits near the end of the DOM so Tab-cycling through the body reaches it last.

### Shortcuts
- None. No key handlers beyond the browser default.

---

## Accessibility notes (extracted from the source; not from an axe run)

- All interactive elements are native (`<a>`, `<button>`, `<select>`, `<input>`). No custom ARIA is needed.
- `aria-label` is set on the top-nav and state-switcher containers.
- SVG maps have `aria-label` per day/per card for screen-reader context.
- Hero and card images would need meaningful `alt` in production; prototype uses decorative gradient ground with placeholder `aria-label`.

**Follow-up**: an `axe-core` run is deferred (validator prereq not met for v3 lock-in). The actual accessibility audit should be run against the template used in production against real data.

---

## v2 maps — print-mode + remote-API rules

### Print-mode CSS for QR codes

Stage 4's main print-mode block (in the Stage 4 subagent brief) MUST include these rules so QR codes survive print + photocopy + phone-camera scan:

```css
@media print {
  .qr-place { width: 2cm !important; height: 2cm !important; }
  .qr-day   { width: 2.5cm !important; height: 2.5cm !important; }
  .qr-place text, .qr-day text { font-size: 7pt; }
}
```

Physical print size ≥ 2cm is non-negotiable. Smaller QRs fail photocopy compression and phone-camera handoff.

### Remote-API fair-use — `api.qrserver.com`

Stage 3 generates QR codes via `https://api.qrserver.com/v1/create-qr-code/`. Free, keyless, no account required. Politeness rules:

- **Cache aggressively.** Cache key: `qr-{type}-{id}-{ecc}.png` under `.myfootmarks/trips/<slug>/cache/qr/`. Re-runs MUST not re-fetch.
- **Rate limit:** ~1 QR/sec, no concurrent calls. Be a polite client; the API is free.
- **Failure handling:** on HTTP non-200 or timeout, fall back to a `<text>` element inside the SVG with the appropriate fallback string (per `qr-place` / `qr-day` component spec). Build does not error.
- **Provider swap:** if `api.qrserver.com` permanently goes away, the brief swaps the endpoint to `quickchart.io/qr` or `goqr.me/api` — same shape of URL pattern. Brief is decoupled from one specific provider.

### OSM tile fetching — fair-use (existing, called out for v2 maps)

OSM tiles continue to require:
- Descriptive `User-Agent` (`myfootmarks/0.x (trip book generator; contact graner@gmail.com)` or equivalent).
- Valid `Referer` (any well-formed origin).
- ≤ 1 tile per 1.2 seconds, serial, never parallel.
- Aggressive disk cache: `tile-{z}-{x}-{y}.png` under `.myfootmarks/trips/<slug>/cache/tiles/`. Cap ~100 tiles per build.
- On HTTP 429/403: back off 60s, retry once, then fall back to text-only placeholder.
- Visible attribution `Map data © OpenStreetMap contributors` in every map SVG (not in HTML comments).
