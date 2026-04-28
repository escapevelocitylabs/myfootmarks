---
name: research-scenic
description: Research scenic viewpoints, overlooks, and photo spots
---

# Research Scenic

Run live web research on scenic viewpoints, scenic drives, photogenic stops, sunset/sunrise spots, and architectural landmarks worth a detour. Write findings to `research/scenic.md`.

## Inputs

**Required:** `Trips/<slug>/trip.yaml`

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/scenic.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

(Same as research-outdoor. Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `regions`, `travelers`, `home_base`. Bail to `/myfootmarks:intake` if missing.)

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching scenic spots for a trip — viewpoints, photogenic stops, sunset/sunrise locations, and architectural landmarks worth a detour.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Seasonal considerations (fall colors, spring blooms) and access conditions (road closures on scenic drives) change with the calendar — verify with current sources.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>
- Home base: <home_base>

Find the top 10–15 scenic spots: classic viewpoints, lookouts, scenic drives, photogenic architectural landmarks, famous neighborhoods to wander through for their aesthetic, sunset and sunrise viewing spots.

Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body. Both describe the same spots; the YAML is machine-consumed by downstream renderers, the Markdown is what the reader sees.

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

Render the same spots as Markdown with one H3 per recommendation:

### <Spot name>

- **Location:** <neighborhood> — <specific address, intersection, or trailhead>
- **How to reach:** <walk from home_base / short drive / scenic drive detour / stop along highway X>
- **Best time of day:** <e.g., "Golden hour, 30 min before sunset", "Early morning to avoid crowds", "Clear afternoons">
- **Age fit:** <e.g., "All ages, stroller OK", "Short uphill walk">
- **Why it matters:** <one to two sentences — what's iconic about this view or spot>
- **Source:** <markdown link>

Order by your recommendation priority. Prioritize:
1. Proximity to home_base (walkable > short drive > longer detour)
2. Photographic value (iconic, unique to this destination)
3. Accessibility for the travelers given ages
4. Best-time-of-day variety (not all sunsets) so they can fit multiple spots into different time slots
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/scenic.md` with frontmatter that includes both metadata and the `places:` array returned by the subagent:

```yaml
---
topic: scenic
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
    lens: [scenic]
    neighborhood: <string | null>
    address_street: <string | null>
  # …one entry per scenic location in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

```json
{"timestamp": "<ISO 8601>", "skill": "research-scenic", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/scenic.md"]}
```

On error: `"status": "error"` + `"reason"`, no output file written.
