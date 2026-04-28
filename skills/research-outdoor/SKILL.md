---
name: research-outdoor
description: Research outdoor activities — hiking, parks, beaches, and nature experiences
---

# Research Outdoor

Run live web research on outdoor activities at the active trip's destination — hiking trails, local and regional parks, beaches, nature reserves, walking and biking paths, and water activities. Write findings to `research/outdoor.md`.

## Inputs

**Required:** `<slug>/trip.yaml` (resolved via the active-trip pointer)

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/outdoor.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` to get the slug. Read `<slug>/trip.yaml` and parse it. Extract `destination`, `start_date`, `end_date`, `travelers`, `regions`, `interests`.

If `.myfootmarks/current-trip` is missing or `trip.yaml` is missing/malformed, stop and tell the user to run `/myfootmarks:intake` first.

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Pass this prompt (substituting the bracketed values):

```
You are researching outdoor activities for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Do not rely on training data — trail conditions, park hours, seasonal closures, and permit requirements change.

Trip:
- Destination: <destination>
- Dates: <start_date> to <end_date> (inclusive)
- Travelers: <names with ages>
- Regions of focus: <regions or "destination-wide">
- Interests (with weights): <interests summary>

Find the top 10–15 outdoor activities worth including: hiking trails (with rough distance/elevation/difficulty), local and regional parks, beaches, nature reserves, walking or biking paths, water activities (kayaking, paddleboarding, swimming spots).

Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body. Both describe the same activities; the YAML is machine-consumed by downstream renderers, the Markdown is what the reader sees in research/outdoor.md.

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

## Markdown body

Render the same activities as Markdown with one H3 per recommendation:

### <Activity or Place name>

- **Location:** <trailhead / park entrance — neighborhood + address if known>
- **Hours / Season:** <typical hours; note seasonal closures or ranger station hours>
- **Age fit:** <who this suits — e.g., "Stroller-accessible, all ages", "Moderate hike, best for 10+", "Calm beach with lifeguards">
- **Distance / Effort:** <for trails: rough distance + elevation + difficulty; for water activities: typical duration>
- **Why it matters:** <one to two sentences explaining the appeal>
- **Permits / bookings:** <note any permits needed, parking fees, advance reservations — otherwise omit>
- **Source:** <markdown link>

Order by your recommendation priority. Prioritize based on:
1. Match to traveler ages and interest weights (weight "hiking", "beaches", "nature", "outdoor" heavily)
2. Age-appropriateness — flag anything inappropriate for the youngest traveler
3. Geographic relevance to home_base and regions
4. Current-season accessibility (closed trails or seasonal parks should be excluded or explicitly flagged)

The order in the `places:` YAML array MUST match the order in the Markdown body.
```

### Step 4: Render output

Take the subagent's response (both the `places:` YAML array and the Markdown body) and write it to `.myfootmarks/trips/<slug>/research/outdoor.md` with this YAML frontmatter:

```yaml
---
topic: outdoor
generated_at: <ISO 8601 UTC timestamp>
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
    lens: [outdoor]
    neighborhood: <string | null>
  # …one entry per outdoor activity in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

Append one JSON line to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-outdoor", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/outdoor.md"]}
```

If the subagent failed or returned malformed output:
- Do NOT write the output file.
- Append a JSON line with `"status": "error"` and a `"reason": "<short message>"` field instead.
- Tell the user what happened and offer to retry.

Confirm to the user the path of the output file written.
