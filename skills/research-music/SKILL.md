---
name: research-music
description: Research live music venues, concerts, and performance calendars
---

# Research Music

Run live web research on live music at the active trip's destination — venues, upcoming concerts during the trip window, and music-specific districts. Write findings to `research/music.md`.

## Inputs

**Required:** `<slug>/trip.yaml`

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/music.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

(Read `.myfootmarks/current-trip` → slug → `<slug>/trip.yaml`. Extract `destination`, `start_date`, `end_date`, `regions`, `travelers`. Bail to `/myfootmarks:intake` if missing.)

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching live music for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Tour schedules and concert calendars are entirely date-sensitive — you MUST verify against current venue listings and ticket sites. Training data is useless for this.

Trip:
- Destination: <destination>
- Dates: <start_date> to <end_date> (inclusive)
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>

Research the music scene. Cover both:
1. **Venues** — the iconic and notable music rooms (jazz clubs, small rock venues, classical concert halls, outdoor concert spaces, intimate singer-songwriter rooms)
2. **Concerts and shows during the trip window** — specific dated events at those venues or elsewhere that overlap the trip dates

Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body. Both venues and dated events become `places:` entries (for dated events, use the venue's lat/lon).

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
    lat: <float | null>             # decimal degrees
    lon: <float | null>
    category: <string>              # e.g., "jazz-club", "concert-hall", "rock-club", "outdoor-pavilion", "singer-songwriter-room", "music-event-dated"
    lens: [music]
    neighborhood: <string | null>   # free-form neighborhood label; aim for consistency with trip.yaml.regions
    address_street: <string | null>
    phone: <string | null>
    hours_structured:
      mon: "HH:MM-HH:MM" | null
      tue: "HH:MM-HH:MM" | null
      wed: "HH:MM-HH:MM" | null
      thu: "HH:MM-HH:MM" | null
      fri: "HH:MM-HH:MM" | null
      sat: "HH:MM-HH:MM" | null
      sun: "HH:MM-HH:MM" | null
    reservation_url: <string | null>    # ticket link for dated events
    reservation_required: <bool>
    lead_time_days: <int | null>
    signature: <string | null>      # for dated events, include the date here; for venues, a one-line vibe summary
    source_url: <string | null>
    stroller: <bool>
    with_kids: <bool>
    kid_first: <bool>
```

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

Emit `null` for unknown fields, not omitted keys. `lens: [music]` on every entry. Order in YAML must match Markdown order.

## Markdown body

Render the same venues/events as Markdown with one H3 per recommendation:

### <Venue or Event name>

- **Location:** <neighborhood> — <address if known>
- **Type:** <jazz club, concert hall, rock club, outdoor pavilion, singer-songwriter room, etc.>
- **If a venue — typical programming:** <one to two sentences on what plays there, vibe, typical crowd>
- **If a venue — shows during the trip window:** <if you can find specific shows between <start_date> and <end_date>, list them as a nested list with dates>
- **If an event — Date:** <specific date within the trip window>
- **Price tier:** <$, $$, $$$ — match to budget>
- **Age/kid policy:** <note any age restrictions, all-ages matinees, etc.>
- **Why it's worth going:** <one to two sentences>
- **Source:** <markdown link to venue site, ticketing, or official calendar>

Order by your recommendation priority, leading with specific concerts dated within the trip window (highest relevance) then iconic venues. If the destination has a specific music district, call it out as a section intro.

Balance venue listings vs. dated events — both are useful, but dated events are the higher-value find when they exist.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/music.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: music
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
    lens: [music]
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
  # …one entry per venue/event in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

```json
{"timestamp": "<ISO 8601>", "skill": "research-music", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/music.md"]}
```

On error: `"status": "error"` + `"reason"`, no output file.
