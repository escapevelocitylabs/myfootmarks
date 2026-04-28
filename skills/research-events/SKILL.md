---
name: research-events
description: Find dated events during your trip window
---

# Research Events

Find events happening at the active trip's destination during the trip window. Write findings to `research/events.md`.

## Inputs

**Required:** `<slug>/trip.yaml`

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/events.md` — markdown document with YAML frontmatter carrying a structured `places:` array (with a `type:` discriminator per entry) plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

(Same as research-weather. Read `.myfootmarks/current-trip` → slug → `<slug>/trip.yaml`. Extract `destination`, `start_date`, `end_date`, `travelers`. Bail to `/myfootmarks:intake` if missing.)

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching dated events for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Do not rely on training data — event schedules change constantly and most fall outside training cutoffs.

Trip:
- Destination: <destination>
- Dates: <start_date> to <end_date> (inclusive)
- Travelers: <names with ages>

Find events happening AT THE DESTINATION between (or partially overlapping) the trip dates: festivals, markets, concerts, theater, sports events, seasonal happenings, exhibitions, parades.

Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body.

## Shared schema: `places:` frontmatter

In addition to the existing markdown body, the subagent MUST emit a structured `places:` YAML array at the top of the output file alongside the existing `topic`, `generated_at`, `consumed_inputs` fields.

Schema per entry:

```yaml
places:
  - id: <slug>                      # stable slug: name lowercased, punctuation stripped, spaces hyphenated
    name: <string>
    wikidata_id: <Qnnnnn | null>    # Wikidata entity id; populated by find-place-image when source ∈ {wikidata-verified, rest-summary, wikivoyage-banner}
    commons_filename: <string | null>   # Commons filename when source is Wikimedia-backed
    image_url: <string | null>          # absolute thumbnail/image URL — populated for every source
    source: <string | null>             # rest-summary | wikidata-verified | wikivoyage-banner | openverse | commons-file-search | commons-category | subpage-og-image | homepage-og-image | null
    archetype: <string | null>          # destination | landmark | restaurant | event | dish | seasonal | neighborhood
    match_confidence: high | medium | low | null
    match_reason: <string | null>       # see find-place-image SKILL.md "Output" envelope
    lat: <float | null>             # venue coords
    lon: <float | null>
    category: <string>              # e.g., "festival", "parade", "farmers-market", "concert", "sports-match"
    lens: [event]
    type: dated | recurring | festival  # REQUIRED — see discriminator below
    neighborhood: <string | null>
    address_street: <string | null>
    phone: <string | null>
    hours_structured:               # day-keyed open/close or null for closed/unknown
      mon: "HH:MM-HH:MM" | null
      tue: "HH:MM-HH:MM" | null
      wed: "HH:MM-HH:MM" | null
      thu: "HH:MM-HH:MM" | null
      fri: "HH:MM-HH:MM" | null
      sat: "HH:MM-HH:MM" | null
      sun: "HH:MM-HH:MM" | null
    reservation_url: <string | null>    # ticketing URL
    reservation_required: <bool>
    lead_time_days: <int | null>    # 0 = walk-up; null = no reservation expected
    signature: <string | null>          # one-line event hook
    source_url: <string | null>
    stroller: <bool>                # age-fit triad
    with_kids: <bool>
    kid_first: <bool>
```

### `lens` values per skill (single-entry arrays)
- `research-attractions` → `[attraction]`
- `research-outdoor` → `[outdoor]`
- `research-scenic` → `[scenic]`
- `research-daytrips` → `[daytrip]`
- `research-shopping` → `[shopping]`
- `research-nightlife` → `[nightlife]`
- `research-music` → `[music]`
- `research-arts` → `[arts]`
- `research-restaurants` + `research-restaurants-kid-friendly` + `research-restaurants-local-cuisine` → `[food]`
- `research-events` → `[event]`
- `research-weather` → `[weather]` (one synthetic entry per trip; see Task A.7)
- `research-local-foods` → emits a `dishes:` array, NOT `places:` (see Task A.6)

### De-dup contract
`build-trip-book` Stage 1 merges per-place `lens` across all surfacing files (union). Downstream files do NOT need to know about cross-file aggregation — each file emits only its own lens.

### Missing data
Missing `wikidata_id`, `commons_filename`, `image_url`, `source`, `archetype`, `match_confidence`, `match_reason`, `hours_structured` entries, `phone`, `reservation_url`, etc. are ALL valid. The render handles absence as a first-class state. Emit `null` rather than omitting keys. When `find-place-image` returns `status: "no-image"`, set `wikidata_id` / `commons_filename` / `image_url` / `source` to `null`, but DO populate `archetype`, `match_confidence` (typically `null`), and `match_reason` so the audit table can reflect what was tried.

## Image resolution: delegate to `find-place-image`

For each entry in your `places:` array (or `dishes:` array, for `research-local-foods`), populate the image-resolution fields by invoking the `find-place-image` skill. **Do NOT** perform unverified `wbsearchentities` lookups inline — that path produced systemic misresolutions (Butchart Gardens → German Christmas store; BC Parliament → Bow Wow Wow band).

For each entity:

1. After resolving `name`, `category`, and (if available) `region` from `trip.yaml.destination`, invoke `find-place-image` with that triple. If you also have a confident `wikidata_id` from another source (e.g., a Wikipedia article that links to a Wikidata entity), pass it as well — `find-place-image` will run validate-mode and either confirm or reject it.

2. Take the resolver's JSON envelope and copy these fields directly into the corresponding `places:` (or `dishes:`) entry:
   - `wikidata_id` (may be null)
   - `commons_filename` (may be null)
   - `image_url` (may be null when status is `no-image`)
   - `source` (may be null when status is `no-image`)
   - `archetype` (always populated)
   - `match_confidence` → write to the entry's `match_confidence` field
   - `match_reason` → write to the entry's `match_reason` field

3. If the resolver returns `status: "no-image"`, that's a valid state — emit the entry with `wikidata_id: null`, `commons_filename: null`, `image_url: null`, `source: null`, but DO populate `archetype` and `match_reason`. The render handles `no-image` entries with a typography-only card (`data-variant="no-image"`).

4. **Do not retry or attempt manual lookups** when the resolver returns `no-image`. The trip planner can correct any single resolution after the first build via `trip.yaml.image_overrides`.

## Why this delegation matters

`find-place-image` is the one place where image-resolution logic lives. Improvements to it (e.g., better archetype heuristics, an additional source in a stack, a tightened P31 verifier) compound across every research skill automatically. The legacy per-skill inline Wikidata code is **deleted** during this migration; do not leave it commented out.

### Event-type discriminator (research-events only)

Every `places:` entry for this skill carries a `type:` field that determines which additional fields are required:

- `type: dated` — one-off event. Additional fields:
  - `start_date: YYYY-MM-DD` (required)
  - `end_date: YYYY-MM-DD` (optional — omit for single-day events)

- `type: recurring` — happens on a weekly schedule during the trip window. Additional fields:
  - `recurring_days: [tue, sat]` (short day names, lowercased, array)

- `type: festival` — multi-day event spanning several days with a program. Additional fields:
  - `festival_run: "YYYY-MM-DD/YYYY-MM-DD"` (required)
  - `sub_events: [{ name: <string>, date: YYYY-MM-DD }, …]` (optional — notable program items)

Classify each event honestly: a Saturday farmers-market is `recurring` (not `dated`); a 5-day Highland Games is `festival`; a one-night concert is `dated`. Dated events falling inside a festival window should be modeled as `sub_events` of the parent festival, not separate entries.

Emit `null` for unknown optional fields, not omitted keys. `lens: [event]` on every entry. Order in YAML must match Markdown order.

## Markdown body

Render the same events as Markdown. Use this structure:

## Events during the trip window

For each event, an H3 heading followed by a structured block:

### <Event name>

- **Dates:** <start>–<end> (or single date)
- **Venue:** <name + neighborhood>
- **What:** <one-sentence description>
- **Source:** <markdown link to the official source>

Order events by date. Prioritize events that overlap the trip window directly. Include events that fall within ±1 day of the trip if they're particularly notable.

If you find no events, return a single paragraph stating that and citing the source you searched (e.g., the city's events calendar URL).
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/events.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: events
generated_at: <ISO 8601>
consumed_inputs: [trip.yaml]
places:
  - id: <slug>
    name: <string>
    wikidata_id: <Qnnnnn | null>
    commons_filename: <string | null>
    image_url: <string | null>
    source: <string | null>
    archetype: <string | null>
    match_confidence: high | medium | low | null
    match_reason: <string | null>
    lat: <float | null>
    lon: <float | null>
    category: <string>
    lens: [event]
    type: dated | recurring | festival
    neighborhood: <string | null>
    address_street: <string | null>
    phone: <string | null>
    hours_structured:
      mon: ...
      # …
    reservation_url: <string | null>
    reservation_required: <bool>
    lead_time_days: <int | null>
    signature: <string | null>
    source_url: <string | null>
    stroller: <bool>
    with_kids: <bool>
    kid_first: <bool>
  # …one entry per event in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

Append to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-events", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/events.md"]}
```

On error: write `"status": "error"` + `"reason": "..."` and do NOT write the output file. Tell the user.
