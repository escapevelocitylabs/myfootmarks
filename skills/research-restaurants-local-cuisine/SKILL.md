---
name: research-restaurants-local-cuisine
description: Find restaurants serving your destination's signature dishes
---

# Research Restaurants — Local Cuisine

Specialization of the restaurants research domain. Finds restaurants that actively serve the destination's signature dishes (sourced from `research/local-foods.md`). Output is a separate file (`restaurants-local-cuisine.md`) that `synth-restaurants` merges with the base `restaurants.md`.

This skill bails gracefully if its trigger condition is not met — it does NOT produce an output file in that case.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml`
- `.myfootmarks/trips/<slug>/research/local-foods.md` (TRIGGER — must exist)

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/restaurants-local-cuisine.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 0: Trigger check

Verify `.myfootmarks/trips/<slug>/research/local-foods.md` exists.

If MISSING:
- Append to `runs.jsonl`:
  ```json
  {"timestamp": "<ISO 8601>", "skill": "research-restaurants-local-cuisine", "slug": "<slug>", "status": "skipped", "reason": "no local-foods.md yet", "consumed_inputs": [], "outputs_written": []}
  ```
- Tell the user:
  > No local-foods research yet — run `/myfootmarks:research-local-foods` first, then re-run this skill.
- Stop. Do NOT proceed to step 1.

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `regions`, `travelers`, `budget_tier`. Bail to `/myfootmarks:intake` if missing.

### Step 2: Gather context

Read `.myfootmarks/trips/<slug>/research/local-foods.md` (verified to exist in step 0). Prepare a list of signature dishes from it.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching restaurants that serve specific signature dishes for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Verify each recommendation with current sources.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>
- Budget tier: <budget_tier>

The destination has these signature dishes:
<numbered list of signature dish names from local-foods.md>

Find restaurants in this destination that ACTIVELY SERVE these specific dishes. Emphasize authenticity over tourist-trap appeal — places where locals actually go for these dishes, not chains catering to visitors.

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
    lat: <float | null>             # decimal degrees
    lon: <float | null>
    category: <string>              # e.g., "traditional-taberna", "heritage-restaurant", "signature-dish-specialist"
    lens: [food]
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
    reservation_url: <string | null>
    reservation_required: <bool>
    lead_time_days: <int | null>
    signature: <string | null>      # MUST list which signature dishes this restaurant serves (by dish_id from local-foods.md)
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

Emit `null` for unknown fields, not omitted keys. `lens: [food]` on every entry. The `signature` field MUST call out specific dish names this restaurant is known for (these will be cross-referenced downstream by `research-local-foods`'s `served_at:` back-refs). Order in YAML must match Markdown order.

## Markdown body

For each restaurant, render as Markdown:

### <Restaurant name>

- **Location:** <neighborhood + address if known>
- **Signature dishes served:** <which specific dishes from the list above this restaurant is known for>
- **What makes them noteworthy:** <one to two sentences explaining authenticity, tradition, or specialization>
- **Price tier:** <$, $$, $$$, $$$$>
- **Source:** <markdown link>

Aim for 5–10 picks. It's fine if you can't fill 10 — quality over quantity. If a particular dish has no clear standout, say so.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/restaurants-local-cuisine.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: restaurants-local-cuisine
generated_at: <ISO 8601>
consumed_inputs: [trip.yaml, research/local-foods.md]
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
    lens: [food]
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
  # …one entry per restaurant in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

```json
{"timestamp": "<ISO 8601>", "skill": "research-restaurants-local-cuisine", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml", "research/local-foods.md"], "outputs_written": [".myfootmarks/trips/<slug>/research/restaurants-local-cuisine.md"]}
```

On error: `"status": "error"`, `"reason"`, no output file.
