# myfootmarks

A Claude Code plugin that turns a few sentences about a trip into a full, magazine-style trip book — itinerary, checklists, and a self-contained HTML keepsake — all from public, keyless data sources.

This repository is both:

- The plugin itself (manifest at [`.claude-plugin/plugin.json`](.claude-plugin/plugin.json), skills under [`skills/`](skills/)).
- A single-plugin Claude Code marketplace (manifest at [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json)).

## Install

Add the marketplace and install the plugin:

```text
/plugin marketplace add escapevelocitylabs/myfootmarks
/plugin install myfootmarks@myfootmarks
```

To pin to a specific tag or branch, append `@<ref>`:

```text
/plugin marketplace add escapevelocitylabs/myfootmarks@v0.1.0
```

## What the plugin does

The skills compose into a trip-planning pipeline. The typical flow is `/myfootmarks:intake` → `/myfootmarks:plan-trip`. Skills can also be invoked individually for fine-grained control.

### Pipeline phases

1. **Intake** — Start a trip; collect destination, dates, travelers, interests; scaffold the trip directory.
2. **Base research** (parallel) — Weather, events, local foods, attractions, restaurants, outdoor, scenic spots, day trips, shopping, nightlife, music, arts.
3. **Specialized research** — Local-cuisine restaurants and kid-friendly restaurants when triggers apply.
4. **Synthesis** — Merge restaurant research streams into a canonical file.
5. **Build** — Itinerary, checklists, destination primer, and the final magazine-style trip book.

### Orchestrator

`plan-trip` runs the pipeline end-to-end: reads filesystem state, dispatches the next phase automatically, and pauses for review between phases.

### External services (all keyless)

- **Wikidata + Wikimedia Commons** — hero images and source URLs.
- **OpenStreetMap tiles** — base64-embedded into self-contained inline SVG maps (subject to OSM fair-use).
- **Overpass API** — optional neighborhood-label nodes.
- **QR Server API** — per-place and per-day QR codes that bridge the printed book to live navigation on the reader's phone via Google Maps.

## Filesystem outputs

When a user runs the pipeline from a working directory `cwd`:

- `cwd/Trips/<slug>/` — user-facing outputs (`trip.yaml`, `itinerary.md`, `packing.md`, `pre-trip.md`, `day-of.md`, `trip-itinerary.html`, `trip-checklists.html`, `trip-book.html`, `assets/<place-id>.jpg`). The three HTML documents are independent and printable standalone.
- `cwd/.myfootmarks/trips/<slug>/research/` — hidden research markdown.
- `cwd/.myfootmarks/trips/<slug>/book-data.json` — Stage 1 normalized book model.
- `cwd/.myfootmarks/trips/<slug>/asset-manifest.json` — Stage 2 image-cache manifest.
- `cwd/.myfootmarks/trips/<slug>/maps/*.svg` — Stage 3 inline SVG maps.
- `cwd/.myfootmarks/current-trip` — active-trip pointer (slug).
- `cwd/.myfootmarks/trips/<slug>/runs.jsonl` — append-only log of skill invocations.

`trip.yaml` carries a required `template:` field (v1: `modern-magazine`) that selects which visual voice `build-trip-book` renders.

## Development

The plugin is developed in a separate private monorepo. Releases land here. To test the marketplace locally before pushing a release:

```text
/plugin marketplace add ./path/to/this/checkout
/plugin install myfootmarks@myfootmarks
```

Validate the marketplace and plugin manifests:

```bash
claude plugin validate .
```

## License

Copyright (c) 2026 Benjamin Graner / Escape Velocity Labs. All rights reserved. See [LICENSE](LICENSE). Personal, non-commercial use is permitted; contact the author for any other use.
