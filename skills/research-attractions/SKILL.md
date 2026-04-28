---
name: research-attractions
description: Research tourist attractions, museums, and landmarks
---

# Research Attractions

Find the top tourist attractions, museums, and landmarks at the active trip's destination. Write findings as a structured list of recommendations to `research/attractions.md`. Optionally enriched by `research/events.md` (so attractions can avoid scheduling crashes with events).

## Inputs

**Required:** `<slug>/trip.yaml`

**Optional:** `.myfootmarks/trips/<slug>/research/events.md` — if present, the subagent will consider scheduled events when prioritizing attractions.

## Output

`.myfootmarks/trips/<slug>/research/attractions.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `<slug>/trip.yaml`. Extract `destination`, `regions`, `travelers` (with ages and interests). Bail to `/myfootmarks:intake` if missing.

### Step 2: Gather enrichment context

Check whether `.myfootmarks/trips/<slug>/research/events.md` exists.

- If present: read it and prepare a concise summary (event names + dates) to include in the subagent brief.
- If absent: skip enrichment.

Track which optional inputs you actually used; you'll declare them in `consumed_inputs`.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching tourist attractions for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Hours, seasonal closures, and notable temporary exhibitions change frequently — verify with current sources.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>
- Interests (with weights): <interests summary>

[If events.md was read:]
Scheduled events during this trip:
<event summary — name + date>

Use this only as light context for prioritization (e.g., note if any famous attraction is closed for an event setup, or if a free festival overlaps with normal museum hours).

Find the top 10–15 attractions worth visiting. Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body. Both describe the same attractions; the YAML is machine-consumed by downstream renderers, the Markdown is what the reader sees in research/attractions.md.

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
    category: <string>              # free-form; e.g. "museum-castle", "viewpoint", "pastelaria"
    lens: [<this-skill-lens>]       # single-entry array identifying THIS skill (see below)
    neighborhood: <string | null>   # free-form neighborhood label; aim for consistency with trip.yaml.regions
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
    reservation_url: <string | null>
    reservation_required: <bool>
    lead_time_days: <int | null>    # 0 = walk-up; null = no reservation expected
    signature: <string | null>      # one-line distinguishing note
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

### Skill-specific guidance
Emit `lens: [attraction]` on every entry. Choose categories that reflect the attraction type: `museum`, `castle`, `monument`, `park`, `garden`, `aquarium`, `historic-site`, etc. Be consistent — don't mix `museum` and `museum-castle` unless there's a real distinction worth preserving.

## Markdown body

Render the same 10-15 attractions as Markdown. Each attraction gets an H3 heading followed by a structured block:

### <Attraction name>

- **Location:** <neighborhood> — <street address if known>
- **Hours:** <typical hours — note seasonal variations>
- **Age fit:** <e.g., "All ages", "best for teens and adults", "kid-friendly with stroller access">
- **Why it matters:** <one to two sentences explaining the appeal — historical significance, exceptional collection, photogenic, etc.>
- **Source:** <markdown link>

Order by your recommendation priority. Prioritize based on:
1. Match to the travelers' interests (heavier weight on higher-weighted interests)
2. Age-appropriateness for the travelers
3. Geographic relevance to the home base / regions
4. General reputation

If `regions` was specified, weight attractions in those areas higher. The order in the `places:` YAML array MUST match the order in the Markdown body.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/attractions.md` with frontmatter containing both the existing metadata AND the `places:` array returned by the subagent:

```yaml
---
topic: attractions
generated_at: <ISO 8601>
consumed_inputs: [<list of files actually read — e.g., trip.yaml, or trip.yaml, research/events.md>]
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
    lens: [attraction]
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
  # …one entry per attraction in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body produced by the subagent unchanged.

Be honest in `consumed_inputs`: include `research/events.md` only if you actually read it.

### Step 5: Log the run

Append to `runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-attractions", "slug": "<slug>", "status": "ok", "consumed_inputs": [<actual list>], "outputs_written": [".myfootmarks/trips/<slug>/research/attractions.md"]}
```

On error: `"status": "error"`, `"reason"`, no output file written.
