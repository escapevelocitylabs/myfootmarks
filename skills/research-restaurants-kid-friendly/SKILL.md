---
name: research-restaurants-kid-friendly
description: Find family-friendly restaurants for your trip
---

# Research Restaurants — Kid Friendly

Specialization of the restaurants research domain. Finds family-friendly restaurants when the trip includes children (any traveler under age 13). Output is a separate file (`restaurants-kid-friendly.md`) that `synth-restaurants` merges with the base `restaurants.md`.

This skill bails gracefully if its trigger condition is not met.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml` (TRIGGER — must include at least one traveler with `age_years < 13`)

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/restaurants-kid-friendly.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 0: Trigger check

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Parse the `travelers` list and check whether any traveler has `age_years < 13`.

If NO traveler is under 13:
- Append to `runs.jsonl`:
  ```json
  {"timestamp": "<ISO 8601>", "skill": "research-restaurants-kid-friendly", "slug": "<slug>", "status": "skipped", "reason": "no travelers under age 13", "consumed_inputs": ["trip.yaml"], "outputs_written": []}
  ```
- Tell the user:
  > No travelers under age 13 — kid-friendly specialization not needed.
- Stop. Do NOT proceed to step 1.

If at least one traveler is under 13, proceed.

### Step 1: Resolve the active trip

(Already done in step 0 — keep the parsed trip.yaml in context.)

Extract: `destination`, `regions`, `travelers` (all of them, with ages), `budget_tier`. Identify which specific travelers are kids (age < 13) — you'll mention them in the brief.

### Step 2: Gather context

This skill has no other optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching family-friendly restaurants for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Adults in the party: <count>
- Children to accommodate: <list with ages — e.g., "Charlie (8), Daisy (2)">
- Budget tier: <budget_tier>

Find restaurants that genuinely accommodate families with children of THESE specific ages. Look for:
- Kids' menus or willingness to make simple kid-friendly dishes
- Reasonable noise tolerance (kids can be loud)
- Reasonable pricing — fine dining isn't usually the right choice with kids
- Early-evening service (5–7pm reservations available)
- Easy-access seating (highchairs, booster seats, stroller-friendly entryways for very young kids)

EXPLICITLY AVOID: formal/fine-dining venues, bar-focused places where kids aren't welcome, places with notoriously long waits.

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
    category: <string>              # e.g., "family-casual", "pizzeria", "bakery-cafe", "brunch-spot"
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
    signature: <string | null>      # why it works for kids — specific accommodation
    source_url: <string | null>
    stroller: <bool>                # usually true
    with_kids: <bool>                # always true for this skill
    kid_first: <bool>                # true when kids are an explicit target audience (e.g., kids menu advertised)
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

Emit `null` for unknown fields, not omitted keys. `lens: [food]` on every entry. `with_kids: true` is the default for this skill. Order in YAML must match Markdown order.

## Markdown body

Render the same restaurants as Markdown:

For each restaurant:

### <Restaurant name>

- **Location:** <neighborhood + address if known>
- **Why it works for kids:** <specific accommodations — kids menu, casual atmosphere, etc.>
- **Best for:** <which of the kids' ages this most suits>
- **Cuisine + atmosphere:** <one line each>
- **Price tier:** <$, $$, $$$>
- **Source:** <markdown link>

Aim for 5–10 picks. Variety of cuisines.
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/restaurants-kid-friendly.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: restaurants-kid-friendly
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
{"timestamp": "<ISO 8601>", "skill": "research-restaurants-kid-friendly", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/restaurants-kid-friendly.md"]}
```

On error: `"status": "error"`, `"reason"`, no output file.
