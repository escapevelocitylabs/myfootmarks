---
name: research-destination-primer
description: Produce the trip book's destination primer — setting, history, culture, per-neighborhood blocks, food scene, seasonal mood — plus hero image Wikidata ID.
---

# Research: Destination Primer

Produce `.myfootmarks/trips/<slug>/research/destination-primer.md` — the narrative primer content that the trip book's Cover + Destination Primer section render from.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml` (for destination, dates, home_base, regions)

## Output

`.myfootmarks/trips/<slug>/research/destination-primer.md` — markdown document with YAML frontmatter carrying `hero_wikidata_id` and a `places:` array of per-home-base-region entries, plus the five-section primer body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract `destination`, `start_date`, `end_date`, `home_base`, `regions`.

If `.myfootmarks/current-trip` is missing or `trip.yaml` is missing/malformed, stop and tell the user to run `/myfootmarks:intake` first.

### Step 2: Dispatch subagent

Use the Agent tool with `subagent_type: general-purpose`. Pass the Subagent brief below, substituting the bracketed trip-context values.

### Step 3: Render subagent response to markdown

The subagent returns a complete markdown document (frontmatter + body). Write it verbatim to `.myfootmarks/trips/<slug>/research/destination-primer.md`.

### Step 4: Log run

Append to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO-8601>", "skill": "research-destination-primer", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/destination-primer.md"]}
```

On error: `"status": "error"` + `"reason"`, no output file written.

## Subagent brief

You are a research subagent producing the destination primer for a trip book. The primer is the narrative gateway — it sets the reader's expectations before they encounter specific places. Tone-setting matters as much as fact-gathering.

### Your tools
- WebFetch — Wikipedia city article, Wikivoyage destination page, local tourism authorities, travel-guide sources
- WebSearch — fallback for specific facts you can't locate via WebFetch

### Trip context
- Destination: <destination>
- Dates: <start_date> → <end_date>
- Home base: <home_base>
- Home-base regions: <regions — expanded list>

### Research objectives

1. **Resolve the destination's Wikidata ID** — look up the destination on Wikipedia; the Wikidata sidebar link gives the `Q…` ID. This will be the `hero_wikidata_id` and seeds the Cover image pipeline (Stage 2 fetches the P18 claim to pull the hero photo from Wikimedia Commons).

2. **Resolve each home-base region's Wikidata ID** when it exists. Neighborhoods of major cities usually have Wikidata entries; smaller regions may not — emit `null` in that case, do not invent.

3. **Gather the five-section content** described below.

### Content to produce

Write a markdown document with YAML frontmatter. The frontmatter:

```yaml
---
topic: destination-primer
generated_at: <ISO-8601>
consumed_inputs: [trip.yaml]
hero_wikidata_id: <Qnnnnn>            # Wikidata entity for the destination itself
places:
  - id: <destination-slug>            # e.g., "victoria-bc"
    kind: destination                 # discriminator unique to this skill
    name: <destination>
    wikidata_id: <Qnnnnn>             # same as hero_wikidata_id — confirmed by find-place-image
    commons_filename: <string | null> # Commons filename when source is Wikimedia-backed
    image_url: <string | null>        # absolute thumbnail/image URL — populated for every source
    source: <string | null>           # rest-summary | wikidata-verified | wikivoyage-banner | openverse | commons-file-search | commons-category | subpage-og-image | homepage-og-image | null
    archetype: destination
    match_confidence: high | medium | low | null
    match_reason: <string | null>
    lens: [destination-primer]
  - id: <neighborhood-slug-1>         # e.g., "james-bay"
    kind: neighborhood
    name: <neighborhood name>
    wikidata_id: <Qnnnnn | null>      # confirmed by find-place-image; null if not found
    commons_filename: <string | null>
    image_url: <string | null>
    source: <string | null>
    archetype: neighborhood
    match_confidence: high | medium | low | null
    match_reason: <string | null>
    lens: [destination-primer]
  # … one entry per home-base region in the same order they appear in trip.yaml.regions
---
```

The `kind:` field on each entry (`destination` | `neighborhood`) is specific to this skill — Stage 1 of `build-trip-book` uses it to route primer content to the Cover vs. the Neighborhoods-at-a-glance sub-section. Other research skills don't emit `kind:`.

### Missing data
Missing `commons_filename`, `image_url`, `source`, `match_confidence`, `match_reason` are ALL valid. The render handles absence as a first-class state. Emit `null` rather than omitting keys. When `find-place-image` returns `status: "no-image"`, set `commons_filename` / `image_url` / `source` to `null`, but DO populate `archetype` and `match_reason` so the audit table can reflect what was tried.

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

### Markdown body (five sections in this exact order)

1. **Setting & geography** (1 paragraph) — where the place sits, what the land looks like, what the light does. Evocative, specific. Ground the reader geographically and atmospherically.

2. **Brief history** (1 paragraph) — the minimum a reader needs to understand the place's layers. Not encyclopedic. Lead with the one detail that explains why the place feels the way it does today.

3. **Cultural notes** (1 paragraph) — language quirks, social protocol, a characteristic local attitude. The things a guidebook usually skips but a seasoned friend would mention. Tipping norms, table manners, greeting customs, queue culture — whatever is actually distinctive here.

4. **Neighborhoods** — one H3 sub-section per home-base region (in the same order as the frontmatter `places:` entries). Each H3 contains:
   - A one-sentence **character line** — evocative, specific. "James Bay is the city's inner-harbour bedroom — painted-lady Victorians, retirees, a twenty-minute walk to anywhere downtown."
   - `**Walk this:**` — a 3-5 stop suggested walking line through the neighborhood, with specific anchors. "From Parliament, cut south on Menzies to Fisherman's Wharf, coffee at Red Fish Blue Fish, then wind north through Beacon Hill Park to the peacocks."
   - `**Pairs with:**` — one sentence saying what this neighborhood pairs with in the trip (a meal, an activity, a day-trip direction, time of day). "Best paired with a mid-afternoon pause — the light on the harbour is unmatched at 4pm."

5. **Seasonal mood** (1 paragraph) — what the trip's dates will feel like. Weather texture, seasonal events or blooms, what's in season food-wise, what locals are doing that time of year. Don't repeat weather.md content; give the reader the vibe.

### Tone
Literary-editorial, second-person direct, present-tense. Short paragraphs over long ones. Specific over generic. "You'll find café tables set out by six; by seven the light turns amber and the buskers drift toward Bastion Square" beats "the city is atmospheric in the evening."

Avoid: breathless marketing ("bustling", "vibrant", "a must-see"), checklist syntax ("don't miss…"), generic lists of attractions (those go in the research skills, not here).

### Length target
~600-900 words total across all five sections. Tight is better than long. Neighborhoods section will have the most content because it scales with the region list.

### Citation

Inline markdown links in the body are fine for specific facts. At the very end of the markdown body, also include a hidden HTML comment listing the primary sources you read:

```html
<!-- sources: https://en.wikipedia.org/wiki/…, https://en.wikivoyage.org/wiki/…, … -->
```

### Output shape

Return the complete markdown file as a single string: frontmatter delimited by `---` lines, then the five-section body, then the sources comment. The calling skill writes this verbatim to disk.
