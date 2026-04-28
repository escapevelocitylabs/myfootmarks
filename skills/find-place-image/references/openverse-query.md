# Openverse query construction + result filtering

`api.openverse.org` is keyless and aggregates Flickr (CC) + Wikimedia Commons + others. Empirically dominant for restaurants (6/6 in the v2 prototype) and dishes (4/6).

## Endpoint

```
GET https://api.openverse.org/v1/images/
```

Query string params:

| Param | Value | Notes |
|---|---|---|
| `q` | full-text search query (URL-encoded) | See "Query construction" below per archetype |
| `page_size` | `3` | Top 3 results; rarely need more for first-image-good-enough |
| `license_type` | `commercial,modification` | All trip-book uses are commercial-adjacent (a sold/shareable keepsake); modification is irrelevant but pairs well |
| `mature` | `false` | Default; keep explicitly to be safe |
| `format` | `json` | Default; explicit |

Example request:

```bash
curl -sS \
  "https://api.openverse.org/v1/images/?q=Red%20Fish%20Blue%20Fish%20Victoria&page_size=3&license_type=commercial,modification&mature=false&format=json"
```

## Query construction per archetype

| Archetype | Query template | Example |
|---|---|---|
| `restaurant` | `"<name> <region-or-city>"` | `"Red Fish Blue Fish Victoria"` |
| `dish` | `"<dish-name>"` (no region — dishes generalize) | `"halibut tacos"` |
| `landmark` (fallback after Wikidata) | `"<name>"` (specific landmarks rarely need region) | `"Butchart Gardens"` |
| `event` | `"<name>"` (events frequently don't have region in the search target) | `"Victoria Highland Games"` |
| `seasonal` | `"<season> <topic>"` | `"spring cherry blossom"` |
| `destination` (last-resort) | `"<destination> skyline"` or `"<destination> waterfront"` | `"Victoria BC harbour"` |

Use the caller's `region` only for the `restaurant` template — it disambiguates restaurants with common names (multiple restaurants named "Bistro 21" exist; Openverse's full-text search picks them apart by region context).

## Result filtering

After receiving results, apply in order:

1. **Aspect-ratio filter.** Reject images with `width / height < 0.5` (very tall portraits) or `width / height > 3.0` (very wide panoramas). The trip-book card crops are landscape-leaning; extreme aspects look broken. Use `width` and `height` fields in the result.
2. **Minimum-resolution filter.** Reject images with `width < 800`. Cards render at ~600px wide; below 800 source-width starts to look soft on retina prints.
3. **License filter.** Already done by `license_type=commercial,modification`, but double-check the result's `license` field is one of: `cc0`, `pdm`, `by`, `by-sa`, `by-nc` (BY-NC is acceptable because the trip-book is a non-commercial keepsake; if commerciality changes later this list narrows).
4. **Take the first survivor.**

## Attribution mapping (passes through to manifest)

| Openverse result field | Manifest field |
|---|---|
| `url` | `image_url` |
| `creator` | `author` |
| `license` (e.g., `by-sa`) + `license_version` | combined into `license` (e.g., `CC BY-SA 4.0`) |
| `foreign_landing_url` | `source_url` |

The renderer credit line for Openverse-sourced images: `Photo: <creator> via Openverse, <CC license>`.

## Empirical evidence

v2 prototype (18 entities × 7 sources):

| Archetype | Hits via Openverse | Notes |
|---|---|---|
| restaurant | 6/6 | Dominant |
| dish | 4/6 | Strong |
| landmark | 3/6 | Behind Wikipedia REST + verified Wikidata |
| event | 2/6 | Behind sub-page og:image |
| destination | 1/6 | Behind Wikivoyage banner |

## Failure modes

- HTTP 503 / timeout → skip this source; fall through.
- Zero results after filtering → skip this source; fall through.
- Result set returned but all filtered out → skip this source.

## When to update query templates

If a category-specific search consistently returns generic stock photos (e.g., a `restaurant` query for "Café Beachside" returns generic beach photos), narrow the template (e.g., add `restaurant` to the query: `"Café Beachside Victoria restaurant"`). Validate the change against ≥ 5 known-good cases before committing.
