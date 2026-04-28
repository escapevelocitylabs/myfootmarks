---
name: research-nightlife
description: Research bars, cocktail lounges, and evening entertainment
---

# Research Nightlife

Run live web research on evening and adult-oriented venues at the active trip's destination — bars, cocktail lounges, beer halls, wine bars, late-night live music spots. Write findings to `research/nightlife.md`.

## Inputs

**Required:** `Trips/<slug>/trip.yaml`

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/nightlife.md` — markdown document with YAML frontmatter carrying a structured `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

(Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `regions`, `travelers`, `budget_tier`. Bail to `/myfootmarks:intake` if missing.)

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Prompt:

```
You are researching nightlife venues for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Bars open and close, change ownership, and shift programs frequently — verify with current sources.

Trip:
- Destination: <destination>
- Regions of focus: <regions or "destination-wide">
- Travelers: <names with ages>
- Budget tier: <budget_tier>

Find the top 10–15 nightlife and evening venues: bars, cocktail lounges, beer halls, wine bars, speakeasies, rooftop bars, late-night live music rooms.

Cover a range:
- Casual pub / neighborhood bar
- Craft cocktail / mixology
- Beer-focused (brewpub, beer hall, craft tap list)
- Wine bar
- Live-music-with-drinks venue (cross-reference with music venues — this can overlap)
- Late-night (open past midnight)

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
    category: <string>              # e.g., "cocktail-bar", "brewpub", "wine-bar", "beer-hall", "speakeasy", "rooftop-bar"
    lens: [nightlife]
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
    signature: <string | null>      # vibe + age policy hint
    source_url: <string | null>
    stroller: <bool>
    with_kids: <bool>                # usually false; true for brewpub/kid-welcoming-early
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

Emit `null` for unknown fields, not omitted keys. `lens: [nightlife]` on every entry. Order in YAML must match Markdown order.

## Markdown body

Render the same venues as Markdown with one H3 per recommendation:

### <Venue name>

- **Location:** <neighborhood> — <address if known>
- **Type:** <cocktail bar, brewpub, wine bar, beer hall, etc.>
- **Vibe:** <casual neighborhood, date-night, high-energy, low-key, tourist-friendly, strictly local>
- **Typical hours:** <when they open and close>
- **Price tier:** <$, $$, $$$ — match to <budget_tier>>
- **Age/kid policy:** <adults-only, 19+/21+, family-friendly-early-then-21+, welcoming to all ages until X PM>
- **Why it's worth going:** <one to two sentences>
- **Source:** <markdown link>

Order by your recommendation priority. EXPLICITLY FLAG any venue that is strictly adults-only or has a dress code. DO NOT include venues that are exclusively daytime cafes misclassified as nightlife. The goal is genuine evening/night-out options.

If the traveler group includes young children, note which venues have reasonable early-evening family windows (e.g., breweries open at 4 PM welcoming kids until 7 PM).
```

### Step 4: Render output

Write `.myfootmarks/trips/<slug>/research/nightlife.md` with frontmatter that includes both metadata and the `places:` array from the subagent:

```yaml
---
topic: nightlife
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
    lens: [nightlife]
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
  # …one entry per venue in the markdown body, same order
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

```json
{"timestamp": "<ISO 8601>", "skill": "research-nightlife", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/nightlife.md"]}
```

On error: `"status": "error"` + `"reason"`, no output file.
