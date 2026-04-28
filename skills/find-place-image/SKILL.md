---
name: find-place-image
description: Find a good representative image for a place, dish, or event.
---

# Find Place Image

Given an entity's name and category (and optionally a region, coordinates, or a pre-resolved Wikidata ID), find a single representative image from a category-aware multi-source stack and return a structured JSON envelope. Used by:

- Every research skill's subagent (after resolving name + category) — invoked as part of populating the `places:` (or `dishes:`) frontmatter.
- `build-trip-book` Stage 2 (`fetch-assets`) — invoked in validate-mode for pre-resolved Wikidata IDs and in fresh-resolve-mode when no ID is present.
- The trip planner directly via `/myfootmarks:find-place-image` — diagnostic, read-only, prints a trace + envelope.

## Inputs

```yaml
name: <string>          # required — entity name
category: <string>      # required — free-form (e.g., "garden", "restaurant-fine-dining", "concert", "dish")
region: <string>        # optional — e.g., "Victoria, BC"; used for region verification + Openverse query
coordinates: [<lat>, <lon>]  # optional — tie-breaker only
wikidata_id: <Qnnnnn>   # optional — when present, runs validate-mode instead of fresh-resolve
```

When invoked as a slash command, the user passes these as a JSON or YAML block in the prompt body.

## Output (JSON envelope)

```json
{
  "status": "found" | "no-image",
  "archetype": "destination" | "landmark" | "restaurant" | "event" | "dish" | "seasonal" | "neighborhood",
  "source": "rest-summary" | "wikidata-verified" | "wikivoyage-banner" | "openverse" | "commons-file-search" | "commons-category" | "subpage-og-image" | "homepage-og-image" | null,
  "wikidata_id": "Qnnnnn" | null,
  "commons_filename": "..." | null,
  "image_url": "https://...",
  "author": "...",
  "license": "...",
  "source_page_url": "...",
  "confidence": "high" | "medium" | "low" | null,
  "match_reason": "rest-title-match" | "wikidata-name+category+region" | "wikivoyage-banner-extract" | "openverse-cc-match" | "commons-file-image" | "commons-category-first" | "subpage-og-link-text-match" | "homepage-og-filtered" | "category-mismatch" | "region-mismatch" | "no-p18-on-verified-entity" | "no-match",
  "rejected_candidates": [ { "source": "...", "reason": "..." } ]
}
```

For `status: "no-image"`, `source` and `image_url` are null; `match_reason` carries the exhaustion reason; `rejected_candidates` summarizes what was tried.

## References (read these first)

- `references/category-to-archetype.md` — free-form category → one of seven archetypes.
- `references/category-to-p31.md` — archetype → canonical Wikidata P31 Q-IDs for verification.
- `references/og-image-heuristics.md` — filename reject list + sub-page link-text matching rules.
- `references/openverse-query.md` — query construction + result filtering.
- `references/attribution-extract.md` — Commons `imageinfo.extmetadata`, Openverse field mapping, og:image domain credit fallback.
- `references/user-agent.md` — Wikimedia UA policy + exact UA string.

## Source stacks per archetype

| Archetype | Source stack (in priority order) |
|---|---|
| `destination` | wikivoyage-banner → rest-summary → wikidata-verified → openverse → commons-file-search |
| `landmark` | rest-summary → wikidata-verified → openverse → commons-file-search → homepage-og-image |
| `restaurant` | openverse → subpage-og-image → commons-file-search → homepage-og-image |
| `event` | subpage-og-image → commons-file-search → openverse → homepage-og-image |
| `dish` | openverse → commons-category |
| `seasonal` | openverse → commons-category |
| `neighborhood` | (treated as `destination`) |

`destination` deliberately omits `homepage-og-image` (Wikivoyage banners cover the surface). Unfamiliar categories fall back to the `landmark` stack with `confidence: "low"`.

When the input `category` resolves to the `neighborhood` archetype, the resolver runs the `destination` source stack but **returns `archetype: "neighborhood"`** in the envelope (input-faithful, not execution-faithful). Stage 2 consumers can filter on the input archetype without losing neighborhood entities to a `destination` alias.

## Steps

### Step 0: Decide mode

If the caller supplied `wikidata_id`: run **validate-mode** (Step V below).

Otherwise: run **fresh-resolve mode** (Steps 1–N below).

### Step V: Validate-mode (caller-supplied Q-ID)

1. Map `category` → `archetype` per `references/category-to-archetype.md`.
2. Fetch P18, P31, P131, P17 for the supplied Q-ID:

   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://www.wikidata.org/w/api.php?action=wbgetentities&ids=${wikidata_id}&props=claims|labels&format=json"
   ```

3. Check the candidate's P31 against the archetype's canonical P31 set (`references/category-to-p31.md`). If no overlap → return:
   ```json
   { "status": "no-image", "archetype": "<X>", "wikidata_id": "<Q...>", "match_reason": "category-mismatch", "rejected_candidates": [{"source": "wikidata-verified", "reason": "P31 ${actual} ∉ archetype canonical set"}] }
   ```
4. Check P131/P17 against the trip's `region` (if provided). If region was provided AND verification fails → return `status: "no-image"`, `match_reason: "region-mismatch"`. If region was NOT provided → skip region check, downgrade `confidence` to `"medium"`.
5. If P18 is absent on the verified entity → return `status: "no-image"`, `match_reason: "no-p18-on-verified-entity"`.
6. Otherwise: extract P18 filename → construct Commons thumbnail URL → fetch attribution per `references/attribution-extract.md` §1 → return:
   ```json
   { "status": "found", "archetype": "<X>", "source": "wikidata-verified", "wikidata_id": "<Q...>", "commons_filename": "<file>", "image_url": "<thumb-url>", "author": "...", "license": "...", "source_page_url": "https://commons.wikimedia.org/wiki/File:<file>", "confidence": "high", "match_reason": "wikidata-name+category+region", "rejected_candidates": [] }
   ```

Validate-mode never falls through to fresh-resolve. If the caller's Q-ID is wrong, returning `no-image` is the right outcome — the caller (Stage 2 of build-trip-book) will, on a subsequent run with the bad ID cleared from research output, fall through to fresh-resolve naturally.

### Steps 1–N: Fresh-resolve mode

Map `category` → `archetype` per `references/category-to-archetype.md`. For each source in the archetype's stack (in order), run the source's resolution. Stop at the first `found` result. If the entire stack exhausts → return `status: "no-image"`, `match_reason: "no-match"`, `rejected_candidates` populated.

#### Source: `wikivoyage-banner` (destination archetype)

1. Fetch raw Wikivoyage page:
   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://en.wikivoyage.org/wiki/${name_url_encoded}?action=raw"
   ```
2. Regex-extract the pagebanner filename:
   ```bash
   filename=$(echo "$wv_page" | grep -oE -i '\{\{\s*pagebanner\s*\|\s*[^|}]+' | head -1 | sed -E 's/.*\|\s*//')
   ```
3. If no banner found → record rejected `{source: "wikivoyage-banner", reason: "no-pagebanner-template"}`, continue.
4. Construct the Commons thumb URL (1200px) using the MD5-hash path scheme — see `skills/build-trip-book/SKILL.md` Stage 2's "Compute the Commons thumbnail URL" step for the canonical formula.
5. Verify the thumb URL returns 200 + image MIME. If 404 → record rejected, continue.
6. Fetch attribution per `references/attribution-extract.md` §1.
7. Return `source: "wikivoyage-banner"`, `confidence: "high"`, `match_reason: "wikivoyage-banner-extract"`.

#### Source: `rest-summary` (destination + landmark archetypes)

1. Fetch Wikipedia REST summary, attempting region-qualified title first if `region` was provided:
   ```bash
   # Try "Butchart Gardens, British Columbia" first, then bare "Butchart Gardens"
   for title_candidate in "${name}, ${region}" "${name}"; do
     enc=$(jq -rn --arg t "$title_candidate" '$t | @uri')
     resp=$(curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
       "https://en.wikipedia.org/api/rest_v1/page/summary/${enc}")
     ...
   done
   ```
2. From the response, prefer `originalimage.source` over `thumbnail.source`. If neither exists → continue to next title or record rejected.
3. Verify the image MIME is `image/*`.
4. Extract the Commons filename from the URL path (e.g., `https://upload.wikimedia.org/.../File.jpg` → `File.jpg`).
5. Fetch attribution per `references/attribution-extract.md` §1.
6. Return `source: "rest-summary"`, `confidence: "high"`, `match_reason: "rest-title-match"`.

#### Source: `wikidata-verified` (destination + landmark archetypes)

1. Search Wikidata for the name (top 5 candidates):
   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://www.wikidata.org/w/api.php?action=wbsearchentities&search=${name_url_encoded}&language=en&limit=5&format=json"
   ```
2. For each candidate Q-ID, fetch P31 + P131 + P17 + P18:
   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://www.wikidata.org/w/api.php?action=wbgetentities&ids=${candidate_qid}&props=claims|labels&format=json"
   ```
3. Apply verification per Step V (same algorithm).
4. The first candidate that passes both P31 and region check → fetch P18 filename → construct Commons thumb URL → fetch attribution.
5. Return `source: "wikidata-verified"`, `confidence: "high"`, `match_reason: "wikidata-name+category+region"`.
6. If all candidates fail → record `rejected_candidates` listing each Q-ID + rejection reason; continue.

#### Source: `openverse` (most archetypes)

1. Construct query per `references/openverse-query.md`.
2. Fetch top 3 results.
3. Apply aspect-ratio + minimum-resolution + license filters per the reference.
4. First survivor → return `source: "openverse"`, `confidence` from `references/openverse-query.md` (medium for most; high for dish/seasonal where Openverse is dominant), `match_reason: "openverse-cc-match"`.

#### Source: `commons-file-search` (most archetypes, MIME-filtered)

1. Search Commons File namespace:
   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://commons.wikimedia.org/w/api.php?action=query&list=search&srnamespace=6&srsearch=${name_url_encoded}&srlimit=5&format=json"
   ```
2. For each result title, fetch `imageinfo.mime`. Reject any non-`image/*` (Commons returns PDFs of historical newspapers for common-noun queries — must filter).
3. First survivor → fetch attribution per `references/attribution-extract.md` §1 → return `source: "commons-file-search"`, `confidence: "medium"`, `match_reason: "commons-file-image"`.

#### Source: `commons-category` (dish + seasonal archetypes)

1. Query the category's members:
   ```bash
   curl -sS -A "myfootmarks/0.0.1 (graner@gmail.com)" \
     "https://commons.wikimedia.org/w/api.php?action=query&list=categorymembers&cmtitle=Category:${term_url_encoded}&cmtype=file&cmlimit=5&format=json"
   ```
2. MIME-filter same as `commons-file-search`.
3. First survivor → fetch attribution → return `source: "commons-category"`, `confidence: "medium"`, `match_reason: "commons-category-first"`.

#### Source: `subpage-og-image` (event + restaurant archetypes)

Apply the link-text matching algorithm in `references/og-image-heuristics.md` §2. On hit → fetch the sub-page → extract its `<meta property="og:image">` → apply filename reject filter → return `source: "subpage-og-image"`, `confidence: "medium"`, `match_reason: "subpage-og-link-text-match"`. Attribution per `references/attribution-extract.md` §3.

#### Source: `homepage-og-image` (last-resort for landmark + restaurant + event)

Fetch the venue homepage with browser-like UA + Referer. Extract `<meta property="og:image">`. Apply filename reject filter (`references/og-image-heuristics.md` §1). Return `source: "homepage-og-image"`, `confidence: "low"` ALWAYS, `match_reason: "homepage-og-filtered"`. Attribution per `references/attribution-extract.md` §3.

### Step Final: Diagnostic-mode trace

When invoked as `/myfootmarks:find-place-image` (slash command), print a human-readable trace BEFORE the JSON envelope:

```
[find-place-image] entity: <name>
[find-place-image] category: <category>
[find-place-image] resolved archetype: <X> (matched | fallback)
[find-place-image] running stack: <source1> → <source2> → ...
[find-place-image] <source1>: <hit | miss | rejected — reason>
[find-place-image] <source2>: <hit | miss | rejected — reason>
[find-place-image] result: <found | no-image>
[find-place-image] envelope:
{ "status": ..., "source": ..., ... }
```

The trace is the diagnostic surface — it's how the trip planner sees why a specific place resolved (or didn't) the way it did. The trace MUST cite the reference file when a decision was driven by it (e.g., `[find-place-image] homepage-og-image: rejected — filename matches reject filter (og-image-heuristics.md §1)`).

The slash command is **read-only**: it does not write `book-data.json`, `asset-manifest.json`, or any other file. The JSON envelope is printed to stdout for the user to inspect.

## Rate-limit etiquette

- Wikimedia (Wikipedia REST, Wikidata, Commons, Wikivoyage): max 4 req/s combined; 250ms sleep between requests; 30s back-off on HTTP 429; UA per `references/user-agent.md`.
- Openverse: serial fetches with 250ms sleep.
- Venue og:image: serial; one homepage fetch + (events) one sub-page fetch per entity. Browser-like UA + Referer per `references/user-agent.md`.

## Failure handling

| Failure | Behavior |
|---|---|
| HTTP 429 | sleep 30s, retry once |
| HTTP 5xx / timeout | retry once after 10s; on second failure, record rejected, continue |
| HTTP 403 / 404 | record rejected, continue |
| Stack exhausted | return `status: "no-image"`, `match_reason: "no-match"`, `rejected_candidates` populated |

The resolver NEVER throws an exception or returns an undefined envelope — every code path produces a valid JSON envelope with `status` of either `found` or `no-image`.

## Caching

Cache the entire envelope keyed on `(name, category, region, wikidata_id?)` tuple under `.myfootmarks/trips/<slug>/cache/find-place-image/<hash>.json` when invoked from Stage 2. The cache key is the SHA-1 of the canonicalized JSON `{name, category, region, wikidata_id?}` — including the optional Q-ID matters because validate-mode and fresh-resolve produce different envelopes for the same `(name, category, region)` triple. Slash-command invocations are NOT cached (diagnostic, by design).
