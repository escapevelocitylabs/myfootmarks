---
name: build-trip-book
description: Render the final magazine-style trip book HTML from research + primer + itinerary + checklists + chosen template. Orchestrates a 4-stage subagent pipeline (normalize → fetch-assets → generate-maps → render-html) with resumable intermediate state.
---

# Build Trip Book

Produce three self-contained HTML artifacts in `Trips/<slug>/` — itinerary, checklists, and keepsake — using the user's chosen template.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml` (must include `template:` field; defaults to `modern-magazine` when absent)
- `.myfootmarks/trips/<slug>/research/destination-primer.md`
- `.myfootmarks/trips/<slug>/research/*.md` (all 14 research outputs + `restaurants-final.md`)
- `Trips/<slug>/itinerary.md`
- `Trips/<slug>/packing.md`
- `Trips/<slug>/pre-trip.md`
- `Trips/<slug>/day-of.md`

## Output

- `Trips/<slug>/trip-itinerary.html` — during-trip bring-along (tactical density, field cards for in-hand logistics)
- `Trips/<slug>/trip-checklists.html` — before-trip print-and-cross-off (austere, B&W-safe, native checkboxes, one-per-page sections)
- `Trips/<slug>/trip-book.html` — keepsake (read-ahead / hotel). **Note the semantic shift: starting in v2 this file is no longer "the whole book"; it is keepsake-only, and the itinerary + checklists have moved into the two new files above.**
- `Trips/<slug>/assets/` — image cache from Wikidata/Commons (cross-run-persistent)
- `.myfootmarks/trips/<slug>/book-data.json` — Stage 1 output
- `.myfootmarks/trips/<slug>/asset-manifest.json` — Stage 2 output
- `.myfootmarks/trips/<slug>/maps/*.svg` — Stage 3 outputs (one `day-<N>.svg` per itinerary day, one `<place-id>.svg` per featured place, and one `destination.svg` editorial overview)

## Resumability

Each stage's primary output file is checked at the top of that stage. If the output exists AND every upstream input is older (by mtime) than the output, the stage is skipped.

Mtime comparisons per stage:

| Stage | Output | Triggers re-run when newer |
|---|---|---|
| 1 — normalize | `book-data.json` | any `research/*.md`, `destination-primer.md`, `itinerary.md`, or `trip.yaml` |
| 2 — fetch-assets | `asset-manifest.json` | `book-data.json` |
| 3 — generate-maps | each `day-<N>.svg`, each `<place-id>.svg`, and `destination.svg` (per-file resumability — any single missing output triggers re-render of that file only, not the whole stage) | `book-data.json`, `itinerary.md`, `trip.yaml` (day-overview maps depend on itinerary's timeline; per-place insets and destination map are book-data driven) |
| 4 — render-html | `trip-itinerary.html` AND `trip-checklists.html` AND `trip-book.html` (all three must exist AND all must be newer than every input to skip) | `book-data.json`, `asset-manifest.json`, any `maps/*.svg`, `packing.md`, `pre-trip.md`, `day-of.md`, `itinerary.md` |

*Stage 4 output is an all-or-nothing set of three files. Any input newer than any of the three outputs OR any of the three outputs missing → re-render all three.*

An explicit `--rebuild` flag is NOT implemented in v1. To force a re-run of a specific stage, delete that stage's intermediate output; to force a full rebuild, delete `.myfootmarks/trips/<slug>/book-data.json` (cascades through the pipeline).

## Steps

### Step 1: Resolve the active trip and verify inputs

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `start_date`, `end_date`, `travelers`, `home_base`, `regions`, `pace`, `template` (default `modern-magazine` when absent).

Verify upstream artifacts exist:
- `.myfootmarks/trips/<slug>/research/destination-primer.md`
- `Trips/<slug>/itinerary.md`, `packing.md`, `pre-trip.md`, `day-of.md`
- At least one `.myfootmarks/trips/<slug>/research/*.md` file with `places:` frontmatter

If anything required is missing, stop and tell the user which upstream skill needs to run first. Do not proceed.

### Step 2: Stage 1 — normalize

If `.myfootmarks/trips/<slug>/book-data.json` exists AND is newer than every research file + destination-primer.md + itinerary.md + trip.yaml, skip this stage.

Otherwise, dispatch a subagent with the following brief.

#### Stage 1 subagent brief

You normalize the raw research outputs into a single structured book model. Read:

- `Trips/<slug>/trip.yaml`
- `.myfootmarks/trips/<slug>/research/destination-primer.md` frontmatter (`hero_wikidata_id`, `places:` array with `kind: destination | neighborhood`)
- Every `.myfootmarks/trips/<slug>/research/*.md` frontmatter (14 research files + `restaurants-final.md`)
- `Trips/<slug>/itinerary.md` (for day structure and stop IDs)

Produce a single JSON file at `.myfootmarks/trips/<slug>/book-data.json` with this shape:

```json
{
  "trip": {
    "destination": "...",
    "start_date": "YYYY-MM-DD",
    "end_date": "YYYY-MM-DD",
    "travelers": [...],
    "home_base": "...",
    "regions": [...],
    "pace": "...",
    "template": "modern-magazine"
  },
  "primer": {
    "hero_wikidata_id": "Qnnnnn",
    "neighborhoods": [
      { "id": "...", "name": "...", "character": "...", "walk_this": "...", "pairs_with": "..." }
    ]
  },
  "time_primer": {
    "weather": { "daily_forecast": [...] },
    "events": {
      "dated": [...],
      "recurring": [...],
      "festivals": [...]
    },
    "things_to_know": [
      { "tag": "In season | Reduced hours | Free evening | Book ahead | Weather-sensitive | New opening", "title": "...", "body": "..." }
    ]
  },
  "places": [
    {
      "id": "castelo-sao-jorge",
      "name": "...",
      "wikidata_id": "Qnnnnn",
      "commons_filename": "...",
      "image_url": "...",
      "source": "...",
      "archetype": "...",
      "match_confidence": "...",
      "match_reason": "...",
      "lat": 38.714,
      "lon": -9.133,
      "lens": ["attraction", "scenic", "daytrip"],
      "neighborhood": "...",
      "category": "...",
      "address_street": "...",
      "phone": "...",
      "hours_structured": { "mon": "...", "tue": "...", "...": "..." },
      "reservation_url": "...",
      "reservation_required": false,
      "lead_time_days": null,
      "signature": "...",
      "source_url": "...",
      "stroller": true,
      "with_kids": true,
      "kid_first": false
    }
  ],
  "dishes": [
    { "id": "...", "name": "...", "description": "...", "wikidata_id": null, "commons_filename": null, "image_url": "...", "source": "...", "archetype": "dish", "match_confidence": "...", "match_reason": "...", "cultural_context": "...", "served_at": ["place_id", "..."] }
  ],
  "lookups": {
    "by_neighborhood": { "<neighborhood-name>": [{ "place_id": "...", "lens_summary": "..." }] },
    "by_day_of_week": {
      "mon": { "open": ["place_id", ...], "closed": ["place_id", ...], "recurring_events": [...] },
      "tue": { "...": "..." },
      "wed": { "...": "..." },
      "thu": { "...": "..." },
      "fri": { "...": "..." },
      "sat": { "...": "..." },
      "sun": { "...": "..." }
    },
    "by_weather": { "sunny": ["place_id", ...], "rainy": ["place_id", ...] },
    "with_kids": ["place_id", ...],
    "without_kids": ["place_id", ...],
    "reservations": { "weeks_ahead": [...], "days_ahead": [...], "day_of": [...], "walk_up": [...] }
  },
  "featured_groups": {
    "icons": ["place_id", ...],
    "neighborhood_anchors": { "<region>": ["place_id", ...] },
    "events_this_week": ["place_id", ...]
  },
  "daily_plan": [
    {
      "day": 1,
      "date": "YYYY-MM-DD",
      "theme": "...",
      "weather": { "icon": "...", "high_c": 18, "low_c": 10, "summary": "..." },
      "narrative": "3-5 sentence lede",
      "timeline": [
        { "time": "09:30", "place_id": "...", "note": "..." }
      ],
      "alternatives": [
        { "slot": "morning | lunch | dinner", "instead_of": "place_id", "options": [{ "place_id": "...", "label": "..." }] }
      ]
    }
  ],
  "checklists": { "packing": "...", "pre_trip": "...", "day_of": "..." },
  "appendix": ["place_id", ...],
  "_conflicts": []
}
```

### De-duplication rules

Group the `places:` entries across all research files into one canonical per-place record:

- Group by `id` (exact match) OR by `wikidata_id` (if both entries have one and they match).
- When grouping, UNION the `lens:` arrays. A grouped place carries the union of all lenses from every research file that surfaced it. Example: a place surfaced by `research-attractions` + `research-scenic` + `research-daytrips` ends up with `lens: ["attraction", "scenic", "daytrip"]`.
- For conflicting scalar fields (different phone numbers, different `lat`/`lon` to 3 decimal places, different `category`), keep the first-seen value using this read order: `restaurants-final.md` (pre-merged) → `research-attractions.md` → `research-scenic.md` → `research-outdoor.md` → `research-daytrips.md` → `research-arts.md` → `research-music.md` → `research-nightlife.md` → `research-shopping.md` → `research-events.md` → `research-local-foods.md` → `research-weather.md`. Log every conflict to `_conflicts[]` with `{ place_id, field, kept_value, rejected_value, rejected_source }` for debugging.
- For image-resolution fields (`wikidata_id`, `commons_filename`, `image_url`, `source`, `archetype`, `match_confidence`, `match_reason`) on a grouped place: prefer the entry with **higher confidence** (`high` > `medium` > `low`); on tie, prefer source priority `wikidata-verified` > `rest-summary` > `wikivoyage-banner` > `openverse` > `commons-file-search` > `commons-category` > `subpage-og-image` > `homepage-og-image`; on further tie, fall back to the existing first-seen-by-file-priority rule. Log conflicts to `_conflicts[]` with `{place_id, field: "image-resolution-tuple", kept_source, rejected_source, rejected_file}`.

### Featured-group curation heuristics

- `featured_groups.icons`: places tagged with `lens.length >= 2` (surfaced by multiple research lenses — a proxy for "matters multiple ways"). Cap at 6–8. Prefer higher `lens.length` and entries with `wikidata_id` set. Example: Butchart Gardens surfaced by `[attraction, scenic, outdoor, daytrip]` is a clear icon.
- `featured_groups.neighborhood_anchors[region]`: for each home-base region in `trip.regions`, pick 2–4 places where `neighborhood == region`, preferring ones NOT already in `icons` (avoid double-counting). Order by `lens.length` descending.
- `featured_groups.events_this_week`: every entry from `research-events`'s `places:` whose dates overlap the trip window (check `start_date`, `festival_run`, or `recurring_days` intersected with trip day-of-week set).

### Lookup-index construction

- `lookups.by_neighborhood[region]`: every place with `neighborhood == region`, with a `lens_summary` string summarizing its lens list (e.g., `"Attraction · Scenic · Kid-powered"`). Sort by `lens.length` descending.
- `lookups.by_day_of_week[day]`: for each short day key (`mon` … `sun`):
  - `open`: place_ids where `hours_structured[day] != null`.
  - `closed`: place_ids where `hours_structured[day] == null` AND at least one other day has hours (indicating intentional closure, not unknown).
  - `recurring_events`: place_ids from `research-events` where `type: recurring` and `recurring_days` contains this day.
- `lookups.by_weather.sunny`: places with any of `[outdoor, scenic, daytrip]` in `lens`. `by_weather.rainy`: places with `[attraction, arts, food, shopping, nightlife]` in `lens` AND explicit indoor `category` (museum, gallery, restaurant, market, bar, concert-hall).
- `lookups.with_kids`: places where `kid_first == true`. `without_kids`: places inferred adults-only from category (`cocktail-bar`, `speakeasy`, `wine-bar`, `fine-dining`) or from `with_kids == false` AND `stroller == false`.
- `lookups.reservations`: bucket places by `lead_time_days`:
  - `> 14` → `weeks_ahead`
  - `3 <= x <= 14` → `days_ahead`
  - `1 <= x <= 2` → `day_of`
  - `0` (walk-up) → `walk_up`
  - Omit places with `null` lead_time (no reservation expected).

### Daily plan extraction

Parse `itinerary.md` to extract per-day structure. For each day:
- Extract date, theme/title, narrative lede (3–5 sentences).
- Parse the timeline items. For each time-stamped stop, match the stop name against the deduplicated `places[]` list to resolve a `place_id`. If no match, keep the stop as a free-text `note` with `place_id: null`.
- Extract per-day alternatives if itinerary structure surfaces them; otherwise emit an empty `alternatives: []` for that day.

Weather for each day comes from `research-weather.md`'s `daily_forecast` array matched by date.

### Time-primer `things_to_know` extraction

Scan every research-file's Markdown body for time-sensitive gotchas that match the trip window. For each one you find, emit a `{ tag, title, body }` entry using ONE of these canonical tags (pick the best fit):

- `In season` — ingredient at peak, bloom window, migration, etc.
- `Reduced hours` — shoulder-season schedule, partial closures.
- `Free evening` — free admission night or half-price night that falls within the trip window.
- `Book ahead` — a specific restaurant/event within the trip week that requires urgency.
- `Weather-sensitive` — an activity that depends on weather holding.
- `New opening` — something new since the last guidebook edition that the reader might miss.

Cap at 8 entries. Prioritize ones that actually fall within the trip dates, not just "sometime in May."

### Output

Write `.myfootmarks/trips/<slug>/book-data.json` as pretty-printed JSON (2-space indent). Do not emit anything else.

### Step 3: Stage 2 — fetch-assets

If `.myfootmarks/trips/<slug>/asset-manifest.json` exists AND is newer than `book-data.json`, skip this stage.

Otherwise, dispatch a subagent with the following brief.

#### Stage 2 subagent brief

You resolve and download one representative image per entity in `book-data.json` (`places[]` and `dishes[]` and the destination hero), honoring `trip.yaml.image_overrides` first and falling back to the `find-place-image` resolver. You produce `asset-manifest.json` with enriched per-entry metadata and print a per-entity audit table to stdout.

### Your tools
- WebFetch — for HTTP fetches (where applicable)
- Bash — `curl` for image downloads, `jq` for JSON manipulation, `md5` for Commons hash computation, `sleep` for pacing
- The `/myfootmarks:find-place-image` skill — invoke it directly when fresh resolution is needed

### Inputs to read

- `Trips/<slug>/trip.yaml` — for `destination` (passed as `region` to find-place-image) and `image_overrides` (top-level key, optional).
- `.myfootmarks/trips/<slug>/book-data.json` — `places[]`, `dishes[]`, and `primer.hero_wikidata_id`.

### Steps

1. **Read inputs.** Parse `trip.yaml.destination` (use as `region`). Parse `trip.yaml.image_overrides` (default to empty map if absent). Parse `book-data.json` and collect every entity that needs an image: each `places[]` entry (key by `places[].id`), each `dishes[]` entry (key by `dishes[].id`), plus a synthetic `hero` entity keyed by the destination-primer's place id, with `wikidata_id: <book-data.primer.hero_wikidata_id>`.

2. **Validate `image_overrides` keys.** For each key in `trip.yaml.image_overrides`, check that it matches an entity id in the collected set. Unknown keys produce a warning row in the audit table but the build continues.

3. **For each entity (in book-data order), resolve the image:**

   **Step 0: Override.** If `image_overrides[<entity_id>]` is present:
   - `"skip"` → record manifest entry `{status: "no-image", source: "override-skip", reason: "manual-skip", confidence: "high", match_reason: "manual-skip"}`. Skip download. Continue.
   - Commons URL or raw Commons filename → parse filename, construct Commons thumb URL (1200px) using the MD5 hash-path scheme described later in this brief, download, fetch attribution per `skills/find-place-image/references/attribution-extract.md` §1. Record `source: "override-commons"`, `confidence: "high"`. On failure → `status: "fetch-failed"`, populate `reason`. Continue.
   - Other HTTPS URL → direct download with `curl -sS -L -o`. Verify the response Content-Type is `image/*`; if not, record `status: "fetch-failed"`, `reason: "non-image-mime: <mime>"`. On success → record `source: "override-url"`, `author: "unknown"`, `license: "user-provided"`, `confidence: "high"`. Continue.
   - Anything else (malformed) → record `status: "fetch-failed"`, `reason: "malformed-override-value"`. Continue.

   **Step 1: Pre-resolved Q-ID validation.** Else if the entity has non-null `wikidata_id` AND non-null `commons_filename` AND non-null `image_url`:
   - If `source == "wikidata-verified"` (already verified upstream): download `image_url`, fetch attribution per `attribution-extract.md` §1, record `source: "wikidata-verified"`, `confidence: "high"`. Continue.
   - Else (entity from pre-migration research output, `source == null` or any non-`wikidata-verified` value): **invoke `find-place-image` in validate-mode** by passing `wikidata_id` along with `name`, `category`, `region`. On the envelope's `status: "found"` → download the envelope's `image_url`, write all envelope fields to the manifest. On `status: "no-image"` (validation rejected the Q-ID) → record `status: "no-image"`, `reason: "stale-or-mismatched-id"`, copy `match_reason` from the envelope. **Do NOT fall through to fresh-resolve in this run** — the bad ID needs to be cleared from research output first.

   **Step 2: Fresh resolve.** Else (no override, no usable Q-ID): invoke `find-place-image` with `{name, category, region}` (and `coordinates` if present). On `found` → download `image_url`, write all envelope fields to manifest. On `no-image` → record `status: "no-image"`, copy `match_reason` from envelope.

4. **Image download mechanics.** Per source:
   - **Wikimedia (Commons-backed) sources** (`rest-summary`, `wikidata-verified`, `wikivoyage-banner`, `commons-file-search`, `commons-category`, `override-commons`): the `image_url` from find-place-image is already a 1200px thumbnail URL (`https://upload.wikimedia.org/wikipedia/commons/thumb/<a>/<ab>/<filename>/1200px-<filename>`). For raw Commons filenames (override path), construct the URL using:
     ```bash
     filename_underscored=$(echo "$filename" | tr ' ' '_')
     md5=$(echo -n "$filename_underscored" | md5 -q)
     prefix1=${md5:0:1}
     prefix2=${md5:0:2}
     thumb_url="https://upload.wikimedia.org/wikipedia/commons/thumb/${prefix1}/${prefix2}/${filename_underscored}/1200px-${filename_underscored}"
     # If filename ends in .png/.svg/etc., append .jpg to force JPG transcoding
     ```
   - **Openverse**: download the result's `url` directly (Openverse returns hot-linkable image URLs).
   - **og:image (sub-page or homepage)**: download the og:image URL with browser-like UA + Referer set to the originating venue page.
   - **Override-URL**: direct `curl -sS -L -o`, verify `Content-Type: image/*` afterward via `curl -sI`.

   ```bash
   mkdir -p "Trips/<slug>/assets"
   curl -sS -L -A "${UA}" -o "Trips/<slug>/assets/<entity-id>.jpg" "${image_url}"
   sleep 0.25
   ```

   If `curl` returns non-zero or the output file is under 1KB → `status: "fetch-failed"`, `reason: "download-failed"`.

5. **Idempotency.** Before each download, check if `Trips/<slug>/assets/<entity-id>.jpg` already exists AND `book-data.json[<entity_id>]` mtime is unchanged from the prior manifest entry's `generated_at` timestamp AND no override change has occurred for that entity. If all three are true, skip the entire find-place-image call AND the download — reuse the prior manifest entry verbatim (still print its row in the audit table).

6. **Cache `find-place-image` results.** When invoking the resolver from this stage (not from the slash command), key the cache on SHA-1 of `{name, category, region, wikidata_id?}` JSON-canonical and write to `.myfootmarks/trips/<slug>/cache/find-place-image/<hash>.json`. On a re-run with unchanged inputs, reuse the cache hit (do NOT re-invoke the resolver).

7. **Write `asset-manifest.json`.** Emit `.myfootmarks/trips/<slug>/asset-manifest.json` with this enriched schema:

```json
{
  "generated_at": "<ISO-8601>",
  "images": [
    {
      "place_id": "...",
      "status": "cached" | "no-image" | "fetch-failed",
      "local_path": "Trips/<slug>/assets/<id>.jpg" | null,

      "source": "rest-summary" | "wikidata-verified" | "wikivoyage-banner" | "openverse" | "commons-file-search" | "commons-category" | "subpage-og-image" | "homepage-og-image" | "override-commons" | "override-url" | "override-skip",
      "archetype": "destination" | "landmark" | "restaurant" | "event" | "dish" | "seasonal" | "neighborhood" | null,
      "confidence": "high" | "medium" | "low" | null,
      "match_reason": "...",

      "source_filename": "..." | null,
      "image_url": "..." | null,
      "source_url": "...",
      "author": "...",
      "license": "...",
      "reason": "..."
    }
  ],
  "hero_image": { "place_id": "<destination-id>", "...": "(same shape as an entry in images[])" }
}
```

8. **Print the audit table to stdout.** One aligned monospace block, one row per entity (in book-data order), with these columns: ENTITY, ARCHETYPE, SOURCE, CONFIDENCE, NOTES.

```
ENTITY                                ARCHETYPE    SOURCE              CONFIDENCE  NOTES
butchart-gardens                      landmark     rest-summary        high        Sunken Garden, CC BY-SA 3.0
royal-bc-museum                       landmark     rest-summary        high        Main entrance, CC BY 3.0
red-fish-blue-fish                    restaurant   openverse           high        Flickr CC BY-NC
belfry-casey-and-diana                event        subpage-og-image    medium      /shows/casey-and-diana
victoria-highland-games-2026          event        commons-file-search high        Category:Highland games
obscure-local-cafe                    restaurant   —                   —           no-image (stack-exhausted)
bad-override-example                  landmark     override-url        —           fetch-failed: 404 at <url>
```

NOTES column carries (for `cached`) a short "<image subject>, <license>" hint; (for `no-image`) the `match_reason` (e.g., `no-match`, `category-mismatch`, `stale-or-mismatched-id`); (for `fetch-failed`) `<reason>: <detail>`. Pad columns with spaces to align — no box-drawing characters.

Append warning rows for any unknown override keys flagged in step 2.

### Rate-limit etiquette

- Wikimedia (combined): max 4 req/s; 250ms sleep between requests; 30s back-off on HTTP 429.
- Openverse: 250ms sleep between requests.
- Venue og:image fetches: serial, no concurrency.
- Total Stage 2 wall-time target for a 50-place trip: under 4 minutes. Abort if it exceeds 10 minutes.

### Failure-mode summary

| Failure | manifest behavior | audit row |
|---|---|---|
| Resolver returns `no-image` (stack exhausted) | `status: "no-image"`, `match_reason: "no-match"` | `—  —  no-image (stack-exhausted)` |
| Resolver returns `no-image` (category-mismatch on validate-mode) | `status: "no-image"`, `match_reason: "category-mismatch"` or `region-mismatch` | `—  —  no-image (stale-or-mismatched-id)` |
| Override invalid (404, malformed, non-image) | `status: "fetch-failed"`, populated `reason` | `override-*  —  fetch-failed: <reason>` |
| Override key matches no entity | (no manifest entry written for that key) | warning row at end of audit |
| Wikimedia 429 / 5xx | retry per rate-limit etiquette; persistent → `status: "fetch-failed"`, `reason: "upstream-unavailable"` | `<source>  —  fetch-failed: upstream-unavailable` |

### Output

Return a short status summary: `"Stage 2 complete: <N cached>, <M no-image>, <K fetch-failed>, <L overridden>."`

### Step 4: Stage 3 — generate-maps

If `.myfootmarks/trips/<slug>/maps/` exists AND every expected SVG is present AND each is newer than `book-data.json`, skip this stage.

Otherwise, dispatch a subagent with the following brief.

#### Stage 3 subagent brief

You generate self-contained inline-SVG maps backed by real OpenStreetMap tile imagery and editorial overlays, plus QR codes for live-navigation handoff. Three map surfaces: day-overview maps in the itinerary doc, place-card inset maps shared by itinerary + keepsake, and a destination-level editorial map in the keepsake doc.

The base layer is OSM tile cartography fetched at build time and embedded as base64 PNG inside the SVG. The overlay layer (numbered pins, dashed accent-red route, white-halo label boxes, home-base square) carries the book's editorial signature. QR codes are generated via the keyless `api.qrserver.com` API and embedded as base64 PNG.

**STOP — read this before doing anything else.** If `.myfootmarks/trips/<slug>/maps/` already contains SVG files from a prior build, do NOT open them, inspect their contents, or use their visual style as a reference. Per-file resumability skips files that are still current; for any file you DO render, follow this brief exactly — base64-embedded OSM tile mosaic + editorial overlay + QR. Never pattern-match the new output to legacy line-art files left over on disk. If the existing files look stylistically different from what this brief describes, that is expected — they are stale outputs from v1 and will be regenerated whenever their inputs change.

### Your tools
- WebFetch — for OSM tiles (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`), QR API (`https://api.qrserver.com/v1/create-qr-code/`), and optional Overpass queries for neighborhood-label nodes
- Bash — for file writes, base64 encoding (`base64 -w 0`), file-existence cache checks, sleep-based rate limiting, simple math via `bc` or `node -e`

### Inputs to read

Runtime data:
- `Trips/<slug>/trip.yaml` — for `destination` string used in QR URL `, {destination}` suffix
- `.myfootmarks/trips/<slug>/book-data.json` (output of Stage 1) — authoritative data model. Specifically you need:
  - `daily_plan[].timeline[]` — per-day stop sequence (name, lat, lon, place_id)
  - `daily_plan[].home_base` (or trip-level `home_base`) — anchor for route start/end
  - `featured_groups.icons[]`, `featured_groups.neighborhood_anchors[]`, `featured_groups.events_this_week[]` — places that get inset maps
  - `places[].lat`, `places[].lon`, `places[].name`, `places[].google_place_id` (optional)

Visual contract (canonical source of truth — MUST be read):
- `skills/build-trip-book/visual-specs/components.md` → "v2 maps — tile mosaic + QR linkouts" for the four new components (tile-mosaic-svg, qr-place, qr-day, destination-map)
- `skills/build-trip-book/visual-specs/interactions.md` → "v2 maps — print-mode + remote-API rules" for fair-use, rate limits, fallback behavior
- `skills/build-trip-book/visual-specs/tokens.json` for accent-red `#d7362b`, paper `#fafaf7`, ink `#0b0b0c`, fonts (Inter sans-800 for pin numbers, sans-700 for labels)

Read all three before writing any SVG.

### Outputs

Three categories of `.svg` files written to `.myfootmarks/trips/<slug>/maps/`:

- `day-<N>.svg` — one per day in `daily_plan`. Contains zoom-15 tile mosaic + numbered pins for each timeline stop + home-base square + dashed accent-red route + per-day QR (in margin) + OSM attribution. Inlined by Stage 4 inside each per-day section in the itinerary.
- `<place-id>.svg` — one per featured place across all `featured_groups.*[]`. Contains zoom-16/17 tile mosaic + single accent-red pin + per-place QR (in margin) + OSM attribution. Inlined by Stage 4 inside place field cards (itinerary) and place story cards (keepsake).
- `destination.svg` — one file per trip. Contains zoom-12/13 tile mosaic covering the full trip bounding box + sparse editorial overlay (no route, optional neighborhood labels) + OSM attribution. NO QR. Inlined by Stage 4 inside the keepsake's Destination Primer section.

All three artifact types are fully self-contained — base64-embedded tiles + base64-embedded QR PNG + inline SVG primitives. No remote `href`, no external CSS, no separate asset files.

### Step 1: Compute bboxes per surface

For each map artifact, compute the lat/lon bounding box and pad it. Pad fractions:

- Day-overview: 0.18 (padFrac applied to lat/lon span; minimum dLat 0.003, dLon 0.005)
- Place-inset: 0.05 (tighter; single-point bbox is the place lat/lon ± a small amount)
- Destination-level: 0.25 (more breathing room — the keepsake's destination map should feel atmospheric, not cropped)

Bbox formula (pseudocode, run in Bash via `node -e` or equivalent):

```javascript
function computeBbox(points, padFrac) {
  const lats = points.map(p => p.lat), lons = points.map(p => p.lon);
  const minLat = Math.min(...lats), maxLat = Math.max(...lats);
  const minLon = Math.min(...lons), maxLon = Math.max(...lons);
  const dLat = Math.max((maxLat - minLat) * padFrac, 0.003);
  const dLon = Math.max((maxLon - minLon) * padFrac, 0.005);
  return { s: minLat - dLat, w: minLon - dLon, n: maxLat + dLat, e: maxLon + dLon };
}
```

For day-overview: `points = [home_base, ...timeline.map(slot => slot.place)]`.
For place-inset: `points = [{lat, lon}]` of the single place; result is just `{lat ± dLat, lon ± dLon}`.
For destination: `points = [home_base, ...all featured places, ...all daily_plan stops]` — the union.

### Step 2: Fetch and cache OSM tiles

For each bbox + zoom level, compute the tile coordinates spanning the bbox via Web Mercator:

```javascript
function lonToTileX(lon, z) { return (lon + 180) / 360 * Math.pow(2, z); }
function latToTileY(lat, z) {
  return (1 - Math.log(Math.tan(lat * Math.PI / 180) + 1 / Math.cos(lat * Math.PI / 180)) / Math.PI) / 2 * Math.pow(2, z);
}
```

`tx0 = floor(lonToTileX(bbox.w, z))`, `tx1 = floor(lonToTileX(bbox.e, z))`, `ty0 = floor(latToTileY(bbox.n, z))`, `ty1 = floor(latToTileY(bbox.s, z))`. The mosaic is `cols = tx1 - tx0 + 1` wide and `rows = ty1 - ty0 + 1` tall.

For each `(tx, ty)` in the grid:
1. Check cache: `.myfootmarks/trips/<slug>/cache/tiles/tile-{z}-{tx}-{ty}.png`.
2. If missing, WebFetch `https://tile.openstreetmap.org/{z}/{tx}/{ty}.png` with:
   - Header `User-Agent: myfootmarks/0.x (trip book generator; contact graner@gmail.com)` (descriptive, plugin-identifying — NOT blank, NOT a generic curl-style UA)
   - Header `Referer: https://github.com/escapevelocitylabs/myfootmarks` (any valid origin)
3. Write the response bytes to the cache file.
4. Sleep 1.2 seconds before the next tile fetch (serial, never parallel — be polite).
5. Base64-encode the cached PNG via `base64 -w 0 < {file}` for embedding in the SVG.

**On HTTP 429 or 403:** back off 60 seconds, retry the failing tile once. If it fails again, fall back to a text-only placeholder for that map (`<text>Map unavailable</text>` inside the SVG) and continue with the rest of the maps.

Tile budget per trip: ~12 tiles per day-overview × 12 days + ~1–4 tiles per place-inset × 30 places + ~16 tiles for destination = up to ~250 tiles. Aggressive cache; re-runs on the same data should fetch zero tiles.

### Step 3: Compose the tile mosaic SVG

For each map artifact, write an `<svg>` whose `viewBox` matches the mosaic's pixel space:

```html
<svg viewBox="0 0 {cols*256} {rows*256}" xmlns="http://www.w3.org/2000/svg"
     preserveAspectRatio="xMidYMid meet">
  <!-- Tile mosaic layer -->
  <image href="data:image/png;base64,{base64-of-tile-tx0-ty0}"
         x="0" y="0" width="256" height="256"/>
  <image href="data:image/png;base64,{base64-of-tile-tx1-ty0}"
         x="256" y="0" width="256" height="256"/>
  <!-- ... one <image> per tile, positioned at (gridX * 256, gridY * 256) ... -->

  <!-- Editorial overlay group (Step 4) -->
  <g class="map-overlay">
    <!-- pins, route, labels, home base -->
  </g>

  <!-- QR + attribution (Step 5) -->
  <g class="qr-slot">
    <image class="qr-day|qr-place" href="data:image/png;base64,{qr-base64}"
           x="..." y="..." width="..." height="..."/>
    <text class="qr-label" x="..." y="...">Scan for directions</text>
  </g>
  <text class="map-attribution" x="..." y="..." font-size="9">Map data © OpenStreetMap contributors</text>
</svg>
```

**Critical:** The SVG `viewBox` MUST be `0 0 {cols*256} {rows*256}` — the pixel dimensions of the tile mosaic. Pins (Step 4) project lat/lon into this same coordinate space via the mosaic projector. Project against the mosaic extent, NOT the requested bbox (mosaic is tile-aligned and may exceed the bbox).

### Step 4: Render the editorial overlay

For each named place / home base, project lat/lon into mosaic pixel space:

```javascript
function makeMosaicProjector(m, w, h) {
  return ({ lat, lon }) => {
    const xPx = (lonToTileX(lon, m.zoom) - m.tx0) / m.cols * w;
    const yPx = (latToTileY(lat, m.zoom) - m.ty0) / m.rows * h;
    return { x: xPx, y: yPx };
  };
}
```

Where `m = { zoom, tx0, ty0, cols, rows }` from Step 2 and `w = cols * 256`, `h = rows * 256`.

Then emit overlay primitives:

**Route line** (day-overview only — sequence is `[home_base, ...stops, home_base]`):
```html
<path d="M {x0} {y0} L {x1} {y1} L {x2} {y2} ..." fill="none"
      stroke="#d7362b" stroke-width="3"
      stroke-dasharray="7 5"
      stroke-linecap="round" stroke-linejoin="round"/>
```

**Home-base marker** (day-overview only — black square, distinguishes from circular stop pins):
```html
<rect x="{hx - 7}" y="{hy - 7}" width="14" height="14"
      fill="#0b0b0c" stroke="#fafaf7" stroke-width="2"/>
```

**Numbered pin** (day-overview: i ranges 1..N for stops; place-inset: just one pin, i=1 or unnumbered):
```html
<g>
  <circle cx="{x}" cy="{y}" r="14"
          fill="#d7362b" stroke="#fafaf7" stroke-width="2.5"/>
  <text x="{x}" y="{y + 4}" text-anchor="middle"
        font-family="Inter" font-weight="800" font-size="13"
        fill="#fafaf7">{i}</text>
</g>
```

**Pin label** (right of pin, day-overview only — place-insets and destination map don't need labels):
```html
<g>
  <rect x="{x + 18}" y="{y - 8}" width="{w}" height="16"
        fill="#fafaf7" stroke="#0b0b0c" stroke-width="0.5"/>
  <text x="{x + 22}" y="{y + 4}"
        font-family="Inter" font-weight="700" font-size="11"
        fill="#0b0b0c">{name}</text>
</g>
```

`width = max(80, name.length * 6.5)`. Always position right of the pin (x + 18).

**Destination map overlay:** sparse — no route, no numbered pins. Optional small dots for featured-place neighborhoods if Overpass returns `place=neighbourhood` nodes in the bbox. NO labels unless the trip has a clear handful of named neighborhoods.

**All strokes ≥ 0.75px** for print-grayscale safety. Route at 3px is fine.

### Step 5: Generate and embed QR codes

QR codes use the keyless `api.qrserver.com` API. **NOT** an npm dep — pure WebFetch.

**Per-place QR (L1):**

URL pattern (the URL the QR encodes — what the phone scanner opens):
```
https://www.google.com/maps/search/?api=1&query={url-encoded "name, destination"}
```

If the place has `google_place_id`, append `&query_place_id={url-encoded id}`.

Example for "Beacon Hill Park" in Victoria, BC:
```
https://www.google.com/maps/search/?api=1&query=Beacon%20Hill%20Park%2C%20Victoria%2C%20BC
```

Then fetch the QR PNG:
```
WebFetch https://api.qrserver.com/v1/create-qr-code/?data={url-encoded-target}&size=200x200&ecc=M
```

Cache as `.myfootmarks/trips/<slug>/cache/qr/qr-place-{place-id}-M.png`. Base64-encode for embedding.

**Per-day QR (L2):**

URL pattern:
```
https://www.google.com/maps/dir/?api=1&origin={home_base.lat},{home_base.lon}&destination={home_base.lat},{home_base.lon}&waypoints={url-encoded "lat1,lon1|lat2,lon2|..."}
```

ALWAYS use lat/lon waypoints. Named-place waypoints are unreliable.

Example for Day 3 of a Victoria trip:
```
https://www.google.com/maps/dir/?api=1&origin=48.4101,-123.3656&destination=48.4101,-123.3656&waypoints=48.5635%2C-123.4708%7C48.5566%2C-123.4622
```

Fetch with `&size=240x240&ecc=L`. Cache as `qr-day-{N}-L.png`.

**Failure handling:** on HTTP non-200 or timeout, substitute a text fallback inside the SVG slot:

- Per-place: `<text class="qr-fallback" x="..." y="...">Search "{name}" on Google Maps</text>`
- Per-day: `<text class="qr-fallback" x="..." y="...">Open Google Maps for today's stops</text>`

Build does not error. Page renders fine. Logged in `runs.jsonl`.

**Rate limit:** 1 QR per second, no concurrent calls. ~30–50 QRs per trip max.

**No QR on the destination map.** That's by design — the keepsake's destination-level map is atmospheric, not navigational.

### Step 6: Self-check QR URL strings

For each generated QR, verify the encoded URL contains the expected destination string. This is the eyeball replacement for an automated decode test.

**Per-place QR check:** the URL passed to `api.qrserver.com` (before URL-encoding) must contain `{name}, {destination}` exactly. Example check via Bash:

```bash
# Pseudo-check: the URL passed to the QR API should encode "Beacon Hill Park, Victoria, BC"
expected_substring="Beacon%20Hill%20Park%2C%20Victoria%2C%20BC"
[[ "$qr_api_url" == *"$expected_substring"* ]] || echo "QR URL mismatch for Beacon Hill Park"
```

**Per-day QR check:** the URL must contain the home_base lat/lon as both origin and destination, AND each timeline stop's lat/lon as a waypoint. Example:

```bash
# Day 3 origin/destination = home base lat/lon, both encoded
[[ "$qr_api_url" == *"origin=48.4101,-123.3656"* ]] || echo "Day 3 origin mismatch"
[[ "$qr_api_url" == *"destination=48.4101,-123.3656"* ]] || echo "Day 3 destination mismatch"
[[ "$qr_api_url" == *"48.5635%2C-123.4708"* ]] || echo "Day 3 missing Butchart Gardens waypoint"
```

If any self-check fails, regenerate the QR with corrected input. If still failing after one retry, fall back to text inside the SVG and log the failure in `runs.jsonl`.

This catches encoding bugs (wrong place name, wrong waypoint order, URL-encoding errors) without an automated decode harness. Pixel-level decode failures (corrupted QR after print) are caught by manual print-test post-merge.

### Failure modes

- **OSM tile fetch returns 429 or 403:** wait 60s, retry once. If still failing, render a text-only `<text>Map unavailable</text>` placeholder inside the SVG. Page renders fine. **QR is unaffected** (independent code path).
- **OSM tile timeout (no response in 30s):** treat as failure; fallback as above.
- **`api.qrserver.com` returns non-200 or times out:** fallback to text inside the SVG (per Step 5 fallback strings). Build does not error.
- **Both fail simultaneously:** both placeholders render; page is still readable. Stage 4 inlines whatever Stage 3 produced.
- **Subagent emits one fewer SVG than expected:** per-file resumability catches this (the missing file is detected next build by the resumability check). User sees stale content for one cycle. Acceptable.
- **QR self-check (Step 6) detects a wrong URL:** retry generation once. If still wrong, fall back to text. Log in `runs.jsonl`.
- **No internet at build time (rare):** cache layer means re-runs on the same data work fine. First-time builds without internet fail loudly with all maps placeholdered.

### Composition rules — Agent Prompt Guide

10 imperative rules for this subagent pass:

1. **Always** write three categories of SVG: `day-<N>.svg` per day, `<place-id>.svg` per featured place, exactly one `destination.svg` per trip. Never collapse them; never invent additional categories.
2. **Always** use the keyless `api.qrserver.com` for QR generation. Never use a different URL scheme (`?query={lat},{lon}` was rejected — hides the listing card). Never add an npm dep.
3. **Never** use coordinate-based place QR URLs. Per-place QR MUST be `?query={name}, {destination}` (with optional `&query_place_id={id}`). Per-day QR MUST be the multi-waypoint directions URL with lat/lon waypoints.
4. **Always** embed tiles and QR images as base64 PNG data URLs inside the SVG via `<image href="data:...">`. Never use remote `href`s — the SVG must be fully self-contained for offline + print viewing.
5. **Always** project pin coordinates against the mosaic extent (`makeMosaicProjector`), not the requested bbox. Mosaic is tile-aligned and may exceed the bbox; using the bbox produces misaligned pins.
6. **Always** use `viewBox="0 0 {cols*256} {rows*256}"` and `preserveAspectRatio="xMidYMid meet"` on the outer `<svg>`. The viewBox MUST match the mosaic pixel space exactly.
7. **Always** include the visible `Map data © OpenStreetMap contributors` `<text>` element near the bottom of every map SVG. OSM fair-use requires it. Never hide in HTML comments.
8. **Always** sleep 1.2 seconds between OSM tile fetches and 1 second between QR API fetches. Cache aggressively (key: `tile-{z}-{x}-{y}.png` and `qr-{type}-{id}-{ecc}.png`). Re-runs on the same data must fetch zero new tiles or QRs.
9. **Always** use accent-red `#d7362b` for route lines and pin fills, paper `#fafaf7` for pin label backgrounds and stroke insets, ink `#0b0b0c` for label text and home-base square. Strokes ≥ 0.75px for print safety.
10. **Always** run the QR URL self-check (Step 6) before declaring done. If a check fails, retry once with corrected input; if still failing, fall back to text and log it.
11. **Never** read or inspect existing files in `.myfootmarks/trips/<slug>/maps/` for style cues. They may be stale v1 line-art outputs from before this brief existed. Render every file you write according to this brief, regardless of what is already on disk. "Match the existing style" is a failure mode, not a feature.

### Output

Return a short status summary to the caller:
`"Stage 3 complete: <N day maps>, <M place insets>, 1 destination map, <K fallback maps>."`

### Step 5: Stage 4 — render-html

If `Trips/<slug>/trip-itinerary.html`, `Trips/<slug>/trip-checklists.html`, AND `Trips/<slug>/trip-book.html` ALL exist AND ALL are newer than `book-data.json`, `asset-manifest.json`, every `maps/*.svg`, `itinerary.md`, `packing.md`, `pre-trip.md`, AND `day-of.md`, skip this stage. If ANY of the three output files is missing OR ANY input is newer than ANY of the three outputs, re-render all three.

Otherwise, dispatch a subagent with the following brief.

#### Stage 4 subagent brief

You compose three fully-independent self-contained HTML documents (itinerary / checklists / keepsake) by inlining all prepared assets (images as local `<img>` src references, SVG maps inlined as `<svg>`, copy, CSS) and applying the `modern-magazine` template. All three docs share one template voice but express three distinct reading densities — see `visual-specs/designs.md` → "One voice, three densities" for the mood requirements per doc.

### Your tools
- Read — for reading the Visual Specifications files listed below
- Bash — for file writes, reads, and glob expansion

### Inputs to read

Runtime data produced by earlier stages:
- `Trips/<slug>/trip.yaml`
- `.myfootmarks/trips/<slug>/book-data.json` (output of Stage 1 — authoritative data model)
- `.myfootmarks/trips/<slug>/asset-manifest.json` (output of Stage 2)
- `.myfootmarks/trips/<slug>/maps/*.svg` (outputs of Stage 3 — inline each)
- `Trips/<slug>/itinerary.md` (for per-day narrative)
- `Trips/<slug>/packing.md`, `pre-trip.md`, `day-of.md` (checklist content)

Visual contract — **the canonical source of truth** for the `modern-magazine` template. MUST be read:
- `skills/build-trip-book/visual-specs/designs.md` — 9-section design narrative (especially Section 9: Agent Prompt Guide, the imperative rules)
- `skills/build-trip-book/visual-specs/tokens.json` — full DTCG design tokens
- `skills/build-trip-book/visual-specs/tokens.md` — human-readable token summary
- `skills/build-trip-book/visual-specs/components.md` — 28-component inventory with HTML/CSS patterns
- `skills/build-trip-book/visual-specs/copy.md` — canonical copy strings, tone, and voice rules
- `skills/build-trip-book/visual-specs/interactions.md` — print mode behavior, state rules (prototype-only states do NOT apply in the production render)
- `skills/build-trip-book/visual-specs/assets.md` — confirms all glyphs inline, no icon files

Read all seven files before writing any HTML. When Visual Specifications conflict with anything in this brief, the Visual Specifications win.

### Output

Write exactly three HTML files to `Trips/<slug>/`. Each file must be fully self-contained (own `<head>`, own inline CSS, own cover, own footer). No file in the set may reference another file in the set via `<a href>` — zero cross-file links. The reader must be able to use any one of the three docs without the other two.

- `Trips/<slug>/trip-itinerary.html` — during-trip bring-along. Tactical density. Field cards for in-hand logistics.
- `Trips/<slug>/trip-checklists.html` — before-trip print-and-cross-off. Austere. B&W-safe. Native checkboxes. One-per-page checklist sections.
- `Trips/<slug>/trip-book.html` — keepsake. Atmospheric, editorial pacing, long-form narrative, destination hero.

### IA — per doc (in reading order)

Each doc has its own reading order. See `visual-specs/designs.md` → "Per-doc reading order (v2 — three-doc split)" for the canonical list.

**`trip-itinerary.html`:**

1. **Cover (tactical)** — destination, dates, travelers, weather band, daily-schedule thumbnail (2-column list: day label + theme). `cover-rail` at top with `doc-mark="Itinerary"` + `doc-desc="Bring this one with you · Day-by-day"`. **No hero photo** on this cover.
2. **Time-specific Primer** — weather strip, typed events this week (3 patterns from `book-data.time_primer.events`), "Things to know this week" list.
3. **Daily Plan** — one per-day section per day in `book-data.daily_plan`. Each day carries: theme, weather band, timeline with per-slot alternatives, day-overview SVG map (inline from `maps/day-<N>.svg`), and — **new in v2** — an inline field card embedded directly inside each named-place timeline slot. Field card rows: `Address`, `Open today`, `Cost`, `Get there`. See `visual-specs/components.md` → "Field card" for the component spec.
4. **Quick Lookups — during-trip flavors only** — two index pages: By Day-of-Week (what's open/closed today) + By Weather (indoor picks for rain, outdoor picks for sun). With-Kids/Without-Kids and By-Neighborhood do NOT appear in the itinerary — they are keepsake-curation pages.
5. **Back cover** — three `emergency-rail` panels: Emergency (911, poison control, non-emergency police, pediatric urgent care, walk-in clinic, 24-hr pharmacy) · Lodging · Transport. See `visual-specs/components.md` → "Back cover".

**`trip-checklists.html`:**

1. **Cover (austere)** — urgency-summary as the first page. `cover-rail` at top with `doc-mark="Checklists"` + `doc-desc="Print this one · Cross off over time"`. Four `urgency-bucket` blocks stacked vertically: Weeks-ahead · Days-ahead · Night-before · Day-of-arrival. Each bucket shows its task list with a due-tag chip per task. **No hero photo.** B&W-safe throughout.
2. **Reservations & Urgency** — place-based reservation list ordered by lead time (weeks-ahead → days-ahead → day-of (non-bookable) → walk-up safe).
3. **Pre-trip checklist** — own page (`page-break-before: always`). Native `<input type="checkbox">`.
4. **Packing checklist** — own page.
5. **Day-of checklist** — own page.

**`trip-book.html` (keepsake):**

1. **Cover (evocative)** — `cover-rail` at top with `doc-mark="Keepsake"` + `doc-desc="Read this one ahead · Or leave it at the hotel"`. Large bordered hero image (from `asset-manifest.hero_image`, 58vh, 3px ink border, 4/4 accent offset shadow). Big-title (destination name) + italic-serif subtitle. Cover-meta + attribution row.
2. **Destination Primer** — 5 sub-sections (setting, history, cultural notes, neighborhoods-at-a-glance, seasonal mood) from `book-data.primer`. Neighborhoods sub-section renders one block per home-base region. The setting sub-section opens with the destination-level editorial map inlined from `maps/destination.svg` — atmospheric, no QR, no route, sparse overlay.
3. **Featured Places** — trip-use groups (Icons / Neighborhood anchors / Events this week) from `book-data.featured_groups`. Each place appears as a **story card** (narrative, history, hero imagery, cultural framing) — NOT a field card. One canonical story card per place; lens badges + layered-insight lines per lens. No in-field logistics (address, hours today, etc.) leak into story cards — that belongs to the itinerary's field card.
4. **What to Eat** — dish-first section. Each `book-data.dishes[]` entry lists its `served_at` places as sub-cards referencing the canonical story cards.
5. **By-Neighborhood reference index** — all places indexed by home-base region. One neighborhood per sub-section.
6. **Quick Lookups — pre-trip flavors only** — With-Kids / Without-Kids curation page. By-Day-of-Week, By-Weather, and Reservations & Urgency do NOT appear in the keepsake — By-Day and By-Weather are itinerary during-trip aids; Reservations is checklists territory.
7. **Appendix** — compact cards for `book-data.appendix[]` place IDs — long-tail researched places that didn't make the Featured cut.

### Composition rules — Agent Prompt Guide

15 imperative rules for this subagent pass. Rules marked **[v2]** are new or rewritten for the three-doc split; unmarked rules are inherited from v1 and apply uniformly across all three docs.

1. **[v2] Always** emit three distinct HTML files per `plan-trip` run into `Trips/<slug>/`: `trip-itinerary.html`, `trip-checklists.html`, `trip-book.html`. Never emit a single combined document.
2. **[v2] Always** make each doc fully self-contained (own `<head>`, own inline CSS, own cover). Never use `<link rel="stylesheet">` pointing across docs; no shared JS file.
3. **[v2] Never** emit an `href` attribute targeting another doc in the set. Zero cross-file links — each doc must be usable by a reader who only has that one doc in hand. Grep the produced files before declaring done: `grep -c 'href="trip-'` must return 0 for each of the three files.
4. **[v2] When** a place exists in both the itinerary's daily plan and the keepsake's Featured Places, emit two different content types: a **field card** (logistics: address, hours today, cost, transport hint) in the itinerary day-spread slot, and a **story card** (narrative: why the place matters, cultural framing, hero imagery) in the keepsake. Same `<slug>` on both; disjoint content. Never copy the keepsake's story card into the itinerary or vice versa.
5. **[v2] Always** render the three doc covers with the `cover-rail` pattern at top (`doc-mark` + `doc-desc`) so readers recognize which of the three docs they're holding. See `visual-specs/copy.md` → "Doc-rail pattern" for the exact strings per doc.
6. **[v2] Always** include the schedule-at-a-glance thumbnail on the itinerary cover (2-column ordered list of days with theme labels). **Never** substitute with a hero image — the itinerary cover is tactical orientation, not atmospheric.
7. **[v2] Always** render the checklists cover as the urgency-summary (four lead-time buckets: Weeks ahead / Days ahead / Night before / Day-of arrival). Never lead the checklists doc with a hero image. Keep it B&W-safe throughout.
8. **[v2] Always** embed a field card inline inside each named-place timeline slot in the itinerary. See `visual-specs/components.md` → "v2 — New components for three-doc split" → "Field card" for the component spec.
9. **[v2] When** the itinerary has quick-lookup pages, render only the during-trip flavors (By Day-of-Week, By Weather). Never include With/Without Kids or By-Neighborhood in the itinerary — those are keepsake pre-trip-curation pages.
10. **[v2] Always** include the back cover on the itinerary (three `emergency-rail` panels: Emergency · Lodging · Transport). Never omit it — this is the "something went sideways" page.
11. **[v2] Always** include a footer attribution line on the itinerary and keepsake: `<Doc mark> · <Destination> · <Dates> · This document is one of three.` Never use the footer to link to the other docs.
12. **Always** use `color.accent.base` (`#d7362b`) for section eyebrows, primary lens badges, page-refs, and route lines on day maps. Never substitute with a secondary brand hue — there is no secondary brand hue. Apply inherited tokens uniformly across all three docs.
13. **Always** use 800-weight Inter with tight negative letter-spacing (`-0.03em` to `-0.045em`) for display h2 and larger. **Never** use Inter below 700 for body content — use Source Serif 4 instead.
14. **Always** use single-column layout for everything except sustained prose. The ONLY sanctioned two-column layout in the three-doc set is the keepsake's primer body paragraphs. **Never** use multi-column for cards, lists, or metadata rows.
15. **[v2] Always** render the checklists doc's Pre-trip, Packing, and Day-of sections with `page-break-before: always` on each, so each checklist prints on its own page. Never use a multi-column checklist grid — this was rejected in v1 and the three-doc split doubles down on print-first checklists.

**Inherited rules that still apply** (carried forward from v1, unchanged):

- One canonical place card per place, with lens badges (rule 2 in v1). Now applies **within** each doc: keepsake's Featured has one canonical story card per place; itinerary's Daily Plan has one field card per visit (a place visited twice gets two field cards at their respective slots — this is not "card duplication," it is the surface re-rendering per visit).
- Featured places grouped by trip-use taxonomy (Icons / Neighborhood anchors / Events this week). Applies to the keepsake's Featured Places section.
- Events with `type` field render using the matching pattern (`dated` → timeline row, `recurring` → weekly grid chip, `festival` → bordered mini-guide). Applies to the itinerary's time-specific primer (for dated events falling this week) AND to the keepsake's Featured Places → "Events this week" group.
- Places lacking a `wikidata_id` or with a failed P18 lookup render with `data-variant="no-image"` (no hero, typography-only). Applies uniformly across all three docs' place references.
- Time-specific primer includes all three sub-sections (Weather · Events · Things-to-know) in that order. Applies to the itinerary only (this primer is in the itinerary's IA, not the keepsake's).
- Hard-offset shadows only (`3px 3px 0 ink` or `4px 4px 0 accent`), never soft-blurred. Applies uniformly.

### Core design tokens (pin the most-used values — full set in `visual-specs/tokens.json`)

- **Paper**: `#fafaf7` (base), `#f2f1ec` (alt — time-primer + what-to-eat), `#ebebe8` (deep — quick-lookups)
- **Ink**: `#0b0b0c` (default), `#2a2a2c` (soft), `#747479` (mute), `#a8a8a4` (faint)
- **Accent**: `#d7362b` (base), `#a6261c` (dark), `#fbe5e3` (soft — timeline-ref chips, today column, yes-chips)
- **Rule line**: `#dcdcd8`
- **Fonts**: Inter (sans, `Inter, 'Helvetica Neue', Helvetica, Arial, system-ui, sans-serif`), Source Serif 4 (body, `'Source Serif 4', 'Source Serif Pro', Charter, 'Iowan Old Style', Georgia, serif`)
- **Container widths**: `860px` (primary reading column, page-w), `1080px` (wide — featured/day/lookups/checklists, wide-w)
- **Body size**: `17px` serif; primer body `16.5px`; card body `15px`
- **Display sizes**: cover big-title `clamp(120px, 18vw, 220px)`; h2-display `78px`; h2-day `72px`; feat-card-title `30px`
- **Letter-spacing**: `-0.045em` (cover), `-0.035em` (tighter), `-0.03em` (display), `0.22em` (eyebrow uppercase)
- **Line-height**: `0.82` (cover), `0.95` (display-2), `1.05` (tight), `1.55` (body), `1.65` (primer)
- **Space ramp** (CSS vars): `--x1: 4px` → `--x11: 80px` (see tokens.json for full scale)
- **Borders**: `1px solid #dcdcd8` (rule), `1px solid #0b0b0c` (ink), `2px solid #0b0b0c` (ink-2), `3px solid #0b0b0c` (ink-3), `3px solid #d7362b` (accent-3)
- **Shadows**: `3px 3px 0 #0b0b0c` (button), `4px 4px 0 #d7362b` (menu/switcher)
- **Radii**: `0` everywhere — all corners sharp by design

### Images

Each card references its image via `<img src="assets/<place-id>.jpg" alt="<place name>">` (relative path — relative to the trip directory). For places with `asset-manifest.images[].status != "cached"`, render the card with `data-variant="no-image"` — no `<img>`, typography-only treatment.

Each image gets an attribution row below or within the card: `Photo: <author> / Wikimedia Commons, <license>` — styled per `visual-specs/components.md` (usually `font-size: 9.5px`, `color: #747479`).

### SVG maps

Inline each `<svg>` from `.myfootmarks/trips/<slug>/maps/` directly into the HTML. DO NOT link to external files. The trip book must be self-contained.

Day-overview maps (`maps/day-<N>.svg`) inline inside each day spread. Inset maps (`maps/<place-id>.svg`) inline inside each canonical place card that has a corresponding inset.

### Print mode CSS

Apply the print-specific overrides from `visual-specs/interactions.md`: body white, accent collapses to `#000`, `--accent-soft` collapses to `#eee`. Checklists page-break-before. No page chrome (no top-nav, no state-switcher, no walkthrough SDK — none of those are rendered in production anyway).

**v2 maps QR sizing (REQUIRED):** include the following inside the document's `@media print` block so QR codes survive print + photocopy + phone-camera scan. Physical print size ≥ 2cm wide is non-negotiable.

```css
@media print {
  .qr-place { width: 2cm !important; height: 2cm !important; }
  .qr-day   { width: 2.5cm !important; height: 2.5cm !important; }
  .qr-place text, .qr-day text { font-size: 7pt; }
}
```

### DO NOT include

- State switcher (`#state-switcher`, `#top-nav`) — prototype-only dev tool.
- Walkthrough SDK (`<script src="…walkthrough.js">` or `#walkthrough-panel`) — prototype-only.
- Any `data-screen`/`data-state` selectors in CSS — those drive the prototype's state switcher, irrelevant to production.

### Output size target

Approximately 2500–4500 lines of HTML **across the three files combined** (not per-file). The keepsake typically accounts for the largest share; checklists the smallest. Do not chase a line count — hit the IA and let the content determine length. No external asset references in any of the three files (images are local paths under `assets/`; SVGs are inline).

### Final step

Write all three files: `Trips/<slug>/trip-itinerary.html`, `Trips/<slug>/trip-checklists.html`, `Trips/<slug>/trip-book.html`. Return a short status summary to the caller: `"Stage 4 complete: itinerary <N1> lines, checklists <N2> lines, keepsake <N3> lines, <M KB total>."`

### Step 6: Log run

Append to `.myfootmarks/trips/<slug>/runs.jsonl` a single line with stage-by-stage status:

```json
{"timestamp": "<ISO-8601>", "skill": "build-trip-book", "slug": "<slug>", "status": "ok", "stages": {"normalize": "ran" | "skipped" | "error", "fetch-assets": "...", "generate-maps": "...", "render-html": "..."}, "outputs_written": ["Trips/<slug>/trip-itinerary.html", "Trips/<slug>/trip-checklists.html", "Trips/<slug>/trip-book.html"]}
```

On error in any stage, log `"status": "error"`, `"reason": "<short message>"`, and include the failing stage in the `stages` object. Do not proceed to downstream stages.
