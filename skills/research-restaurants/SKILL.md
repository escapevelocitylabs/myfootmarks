---
name: research-restaurants
description: Research top restaurant recommendations for your trip
---

# Research Restaurants

Find restaurant recommendations covering a range of cuisines and price points at the active trip's destination. Write findings to `research/restaurants.md`. Optionally enriched by `research/local-foods.md` (so picks bias toward places serving the destination's signature dishes).

This is the BASE restaurants skill. Specialized variants (`research-restaurants-local-cuisine`, `research-restaurants-kid-friendly`) write separate files; `synth-restaurants` merges them all into `restaurants-final.md` for downstream consumption.

## Inputs

**Required:** `<slug>/trip.yaml`

**Optional:** `.myfootmarks/trips/<slug>/research/local-foods.md` — if present, recommendations bias toward restaurants serving identified signature dishes.

## Output

`.myfootmarks/trips/<slug>/research/restaurants.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `<slug>/trip.yaml`. Extract `destination`, `regions`, `travelers`, `budget_tier` (default `moderate`). Bail to `/myfootmarks:intake` if missing.

### Step 2: Gather enrichment context

Check whether `.myfootmarks/trips/<slug>/research/local-foods.md` exists.

- If present: read it and prepare a brief summary of the signature dishes (just the names + one-line descriptions) for the subagent.
- If absent: skip enrichment.

Track which optional inputs you actually read.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching restaurants for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Restaurants open, close, change menus, and shift quality often — verify with current sources.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>
- Budget tier: <budget_tier>

[If local-foods.md was read:]
The destination has these signature dishes worth seeking out:
<local-foods summary>

When recommending restaurants, BIAS TOWARD places that serve these specific dishes well, and CALL OUT in each entry which signature dishes the restaurant is known for. Do not exclude great non-traditional restaurants — just give signature-dish places extra weight.

Return your findings as TWO blocks: a structured `places:` YAML array AND a human-readable Markdown body. Both describe the same 10–15 restaurants.

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
    category: <string>              # e.g., "restaurant-pnw", "izakaya", "bistro", "seafood", "pastelaria"
    lens: [food]
    neighborhood: <string | null>   # free-form neighborhood label; aim for consistency with trip.yaml.regions
    address_street: <string | null>
    phone: <string | null>
    hours_structured:               # IMPORTANT — restaurant hours vary day-to-day
      mon: "HH:MM-HH:MM" | null
      tue: "HH:MM-HH:MM" | null
      wed: "HH:MM-HH:MM" | null
      thu: "HH:MM-HH:MM" | null
      fri: "HH:MM-HH:MM" | null
      sat: "HH:MM-HH:MM" | null
      sun: "HH:MM-HH:MM" | null
    reservation_url: <string | null>    # OpenTable / Resy / venue site
    reservation_required: <bool>
    lead_time_days: <int | null>    # how far ahead to book
    signature: <string | null>      # cuisine + notable-dish hint
    source_url: <string | null>
    stroller: <bool>
    with_kids: <bool>
    kid_first: <bool>               # true only if it's a kid-first environment
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

Emit `null` for unknown fields, not omitted keys. `lens: [food]` on every entry. If local-foods.md was read, the `signature` field should mention which of its dishes this restaurant is known for. Order in YAML must match Markdown order.

## Markdown body

Render the same restaurants as Markdown:

## Restaurants

For each restaurant, an H3 heading followed by a structured block:

### <Restaurant name>

- **Location:** <neighborhood> — <street address if known>
- **Cuisine:** <type — e.g., "Modern Pacific Northwest", "Izakaya">
- **Price tier:** <one of: $, $$, $$$, $$$$ matching <budget_tier>>
- **Family-friendliness:** <e.g., "kid-friendly with menu", "adults preferred", "fine dining, plan ahead with kids">
- **Why it's worth going:** <one to two sentences>
- **Signature dishes** (if local-foods context provided): <which ones this restaurant serves, if applicable>
- **Source:** <markdown link>

Order by your recommendation priority. Mix price tiers — don't recommend only the most expensive places, even at higher budget tiers.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/restaurants.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: restaurants
generated_at: <ISO 8601>
consumed_inputs: [<actual list — trip.yaml, optionally research/local-foods.md>]
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

Append to `runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-restaurants", "slug": "<slug>", "status": "ok", "consumed_inputs": [<actual>], "outputs_written": [".myfootmarks/trips/<slug>/research/restaurants.md"]}
```

On error: `"status": "error"`, `"reason"`, no output file written.
