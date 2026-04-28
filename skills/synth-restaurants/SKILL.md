---
name: synth-restaurants
description: Merge restaurant research into a final canonical list
---

# Synth Restaurants

Merge the base restaurant research (`restaurants.md`) and any specialization files (`restaurants-*.md`) into a single canonical `restaurants-final.md` that downstream build skills consume.

This skill does NOT dispatch a subagent. It reads local files and runs LLM-driven merge reasoning.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml`
- `.myfootmarks/trips/<slug>/research/restaurants.md` (the base)

**Optional:**
- `.myfootmarks/trips/<slug>/research/restaurants-local-cuisine.md`
- `.myfootmarks/trips/<slug>/research/restaurants-kid-friendly.md`
- (Any future `restaurants-*.md` specialization files.)

## Output

`.myfootmarks/trips/<slug>/research/restaurants-final.md` — markdown document with YAML frontmatter carrying a merged/deduplicated `places:` array plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Bail to `/myfootmarks:intake` if missing.

### Step 2: Read base restaurants research

Read `.myfootmarks/trips/<slug>/research/restaurants.md`. If MISSING, stop and tell the user:

> No base restaurants research yet — run `/myfootmarks:research-restaurants` first.

### Step 3: Read any specialization files

List files in `.myfootmarks/trips/<slug>/research/` matching the pattern `restaurants-*.md` (excluding `restaurants-final.md` itself if it happens to exist from a prior run).

For each one found, read its contents. Track which files you read — you'll declare them in `consumed_inputs`.

If no specialization files exist, that's fine — the merge still produces a valid final file using just the base.

### Step 4: Merge into a canonical recommendation list

Reason through the merge directly (no subagent). The merge rules:

1. **Dedupe by restaurant name.** If the same restaurant appears in multiple sources, it's one entry in the final list.

2. **When merging multi-source entries, COMBINE the strongest justifications and notes from each source.** Example: if `restaurants.md` says a place is "great Pacific Northwest cuisine" and `restaurants-kid-friendly.md` says "has a kids menu", the merged entry mentions BOTH.

3. **When sources disagree, the specialization wins on conflict.** If `restaurants.md` says a place is "adults-only ambiance" but `restaurants-kid-friendly.md` says it's actually kid-friendly with the right reservation, trust the specialization.

4. **Preserve source URLs.** Each merged entry keeps a citation link from at least one source.

5. **Preserve recommendation rank loosely.** Top picks from any source land near the top. Ties broken by appearance in more sources.

6. **Image-resolution fields tie-break.** When merging two entries for the same restaurant that carry different image-resolution tuples, prefer the higher confidence (`high` > `medium` > `low`). On ties, prefer source priority `wikidata-verified` > `rest-summary` > `wikivoyage-banner` > `openverse` > `commons-file-search` > `commons-category` > `subpage-og-image` > `homepage-og-image`. Carry forward all six new fields (`wikidata_id`, `commons_filename`, `image_url`, `source`, `archetype`, `match_confidence`, `match_reason`) on every merged entry. If both entries have `match_confidence: null` (both unresolved), keep the one with a non-null `archetype`; otherwise keep the first-seen entry's tuple.

### Frontmatter merge

Merge the `places:` arrays from the input research files into a single deduplicated `places:` array on `restaurants-final.md`:

- De-dup by `id` (if IDs match exactly) OR by `wikidata_id` (if both entries have one and they match).
- When a place appears in multiple input files, produce ONE merged entry. Union the `lens:` arrays (all three inputs emit `lens: [food]` so the union is `[food]` — keep this single-element form).
- For conflicting scalar fields (e.g., two different phone numbers or opening hours), apply the same source-priority rule used for the Markdown body: `research-restaurants-local-cuisine` > `research-restaurants-kid-friendly` > `research-restaurants`. Specializations are more curated than the base.
- Carry `wikidata_id`, `commons_filename`, `image_url`, `source`, `archetype`, `match_confidence`, `match_reason`, `lat`, `lon`, `address_street`, `phone`, `hours_structured`, `reservation_url`, `reservation_required`, `lead_time_days`, `source_url`, `stroller`, `with_kids`, `kid_first` from the winning source.
- For `signature`, MERGE the strings when informative — e.g., "serves Bouzy Bouillabaisse; has a kids menu".
- The final array's order must match the order of H3 entries in the merged Markdown body.

Write the merged list as Markdown:

```markdown
## Restaurants — final canonical list

For each merged restaurant, an H3 entry:

### <Restaurant name>

- **Location:** <neighborhood + address>
- **Cuisine + atmosphere:** <synthesized from sources>
- **Price tier:** <$, $$, $$$, $$$$>
- **Why it's worth going:** <combined justification, with both general and specialization-specific reasons if applicable>
- **Notable for:** <free-form synthesis — e.g., "kid-friendly with menu", "serves authentic local dish X", "best brunch spot">
- **Source:** <markdown link>

## Sources merged

A short list of which input files contributed to this output:
- restaurants.md
- restaurants-local-cuisine.md (if present)
- restaurants-kid-friendly.md (if present)
```

### Step 5: Render output and log

Write `.myfootmarks/trips/<slug>/research/restaurants-final.md` with frontmatter that includes metadata and the merged `places:` array:

```yaml
---
topic: restaurants-final
generated_at: <ISO 8601>
consumed_inputs: [<actual files read — at minimum trip.yaml and restaurants.md, plus any specializations>]
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
  # …one entry per canonical restaurant, same order as merged Markdown body
---
```

Below the frontmatter, write the merged Markdown body.

Append to `runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "synth-restaurants", "slug": "<slug>", "status": "ok", "consumed_inputs": [<actual>], "outputs_written": [".myfootmarks/trips/<slug>/research/restaurants-final.md"]}
```
