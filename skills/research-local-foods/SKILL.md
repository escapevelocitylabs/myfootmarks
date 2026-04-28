---
name: research-local-foods
description: Research signature dishes and local cuisine of your destination
---

# Research Local Foods

Research the food culture of the active trip's destination — what locals actually eat, signature dishes worth seeking out, the food traditions of the place. Write findings to `research/local-foods.md`. Used as enrichment by `research-restaurants` and as the trigger for `research-restaurants-local-cuisine`.

## Inputs

**Required:** `Trips/<slug>/trip.yaml`

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/local-foods.md` — markdown document with YAML frontmatter carrying a structured `dishes:` array AND a `places:` array for food markets/neighborhoods, plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

(Same as research-weather. Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `regions`. Bail to `/myfootmarks:intake` if missing.)

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching local cuisine and signature dishes for a destination.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Local food traditions change less than events, but recent trends, new specialties, and shifting traditions still matter — verify with current sources.

Destination: <destination>
Regions of interest (if specified): <regions>

Research the food culture of this destination. Return your findings as TWO structured YAML blocks — a `dishes:` array and a `places:` array for food markets/neighborhoods — plus a human-readable Markdown body.

## Structured YAML blocks

### `dishes:` array (the PRIMARY output of this skill)

```yaml
dishes:
  - id: <slug>                     # stable slug for the dish (e.g., "pacific-wild-salmon", "pastel-de-nata")
    name: <string>                 # human-readable dish name
    wikidata_id: <Qnnnnn | null>   # Wikidata entity id; populated by find-place-image
    commons_filename: <string | null>   # Commons filename when source is Wikimedia-backed
    image_url: <string | null>          # absolute thumbnail/image URL — populated for every source
    source: <string | null>             # rest-summary | wikidata-verified | wikivoyage-banner | openverse | commons-file-search | commons-category | subpage-og-image | homepage-og-image | null
    archetype: <string | null>          # always "dish" for entries here
    match_confidence: high | medium | low | null
    match_reason: <string | null>       # see find-place-image SKILL.md "Output" envelope
    description: <string>          # one-sentence "what-it-is"
    cultural_context: <string>     # one to two sentences "why it matters here"
    served_at: [<place_id>, …]     # cross-refs to places[].id in restaurant research files;
                                   # leave empty (`served_at: []`) if this skill runs before restaurants research —
                                   # the references will resolve downstream, not here.
    source_url: <string | null>
```

Aim for 5–10 signature dishes covering breakfast/snack/main/dessert breadth. Use stable slug conventions consistent across research files so `served_at` back-refs resolve downstream.

### `places:` array (secondary — food markets and food-scene neighborhoods)

```yaml
places:
  - id: <slug>                      # stable slug: name lowercased, punctuation stripped, spaces hyphenated
    name: <string>                 # market/neighborhood name
    wikidata_id: <Qnnnnn | null>    # Wikidata entity id; populated by find-place-image when source ∈ {wikidata-verified, rest-summary, wikivoyage-banner}
    commons_filename: <string | null>   # Commons filename when source is Wikimedia-backed
    image_url: <string | null>          # absolute thumbnail/image URL — populated for every source
    source: <string | null>             # rest-summary | wikidata-verified | wikivoyage-banner | openverse | commons-file-search | commons-category | subpage-og-image | homepage-og-image | null
    archetype: <string | null>          # destination | landmark | restaurant | event | dish | seasonal | neighborhood
    match_confidence: high | medium | low | null
    match_reason: <string | null>       # see find-place-image SKILL.md "Output" envelope
    lat: <float | null>
    lon: <float | null>
    category: <string>             # e.g., "public-market", "food-hall", "food-neighborhood"
    lens: [food-neighborhood]
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
    reservation_url: <string | null>
    reservation_required: <bool>
    lead_time_days: <int | null>    # 0 = walk-up; null = no reservation expected
    signature: <string | null>
    source_url: <string | null>
    stroller: <bool>                # age-fit triad
    with_kids: <bool>
    kid_first: <bool>
```

3–5 entries. `lens: [food-neighborhood]` on every entry. Emit `null` for unknown fields, not omitted keys.

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

## Markdown body

Render the human-readable body:

## Signature dishes

For each notable dish, an H3 heading followed by a structured block:

### <Dish name>

- **What it is:** <one-sentence description>
- **Cultural context:** <when it's traditionally eaten, occasion, history — one to two sentences>
- **Where to find it:** <neighborhood or category of place — e.g., "izakaya in Gion", "fish-and-chips shops along the waterfront">
- **Source:** <markdown link to a description of the dish>

Aim for 5–10 signature dishes covering breakfast/snack/main/dessert breadth. The order in the `dishes:` YAML array MUST match the Markdown order.

## Notable food markets and food neighborhoods

A short list (3–5) of markets, food halls, or neighborhoods particularly known for food, with one-line descriptions and source links. These correspond to the `places:` array entries — preserve order across both representations.

If you find no clearly distinctive local cuisine (rare but possible for very generic destinations), return a single paragraph saying so and citing what you searched. In that case emit empty `dishes: []` and `places: []` arrays.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/local-foods.md` with frontmatter that includes both metadata and the `dishes:` + `places:` arrays from the subagent:

```yaml
---
topic: local-foods
generated_at: <ISO 8601>
consumed_inputs: [trip.yaml]
dishes:
  - id: <slug>
    name: <string>
    wikidata_id: <Qnnnnn | null>
    commons_filename: <string | null>
    image_url: <string | null>
    source: <string | null>
    archetype: <string | null>
    match_confidence: high | medium | low | null
    match_reason: <string | null>
    description: <string>
    cultural_context: <string>
    served_at: [<place_id>, …]
    source_url: <string | null>
  # …one entry per dish in the markdown body, same order
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
    lens: [food-neighborhood]
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
  # …one entry per food market/neighborhood in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

Append to `runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-local-foods", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/local-foods.md"]}
```

On error: log `"status": "error"` + `"reason"`, no output file written.
