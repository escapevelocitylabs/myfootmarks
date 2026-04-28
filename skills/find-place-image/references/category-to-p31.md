# Archetype → canonical Wikidata P31 Q-IDs

When the resolver runs the `wikidata-verified` source — either as part of a stack or in validate-mode for a caller-supplied `wikidata_id` — it MUST verify that the candidate's P31 (instance-of) claim includes at least one Q-ID from the archetype's canonical set. Otherwise the candidate is rejected with `match_reason: "category-mismatch"`.

## Canonical Q-IDs per archetype

These lists are deliberately broad — Wikidata's P31 taxonomy is messy, with many subclasses. The verifier accepts the candidate if **any** P31 value on the entity is in the archetype's set. Subclass-of (P279) is NOT walked; that's a deliberate simplification — entities whose direct P31 doesn't include a canonical type are routed to `no-image` rather than risk false positives via transitive subclasses.

### `destination`
- `Q515` city
- `Q1093829` city in the United States
- `Q188509` suburb
- `Q3957` town
- `Q5119` capital
- `Q1549591` big city
- `Q35657` U.S. state (when the caller explicitly passes a state-name as a destination)
- `Q133442` neighborhood
- `Q123705` neighborhood (alt)
- `Q1334932` administrative territorial entity
- `Q15642541` human-geographic territorial entity

### `landmark`
- `Q33506` museum
- `Q207694` art museum
- `Q23413` castle
- `Q22746` historic site
- `Q570116` tourist attraction
- `Q839954` archaeological site
- `Q22698` urban park
- `Q44613` city park
- `Q188114` botanical garden
- `Q174782` public square
- `Q39614` cemetery (heritage)
- `Q179700` statue
- `Q24398318` religious building
- `Q16970` church building
- `Q1656682` event venue
- `Q41176` building
- `Q11303` skyscraper
- `Q4989906` monument
- `Q43229` organization (rare; for specific landmark institutions)
- `Q43501` zoo
- `Q11315` shopping mall (when surfaced as a landmark)

### `restaurant`
- `Q11707` restaurant
- `Q30022` café
- `Q1141366` bar (establishment)
- `Q41349` brewery
- `Q204194` pub
- `Q3406554` bistro
- `Q66301044` brewpub
- `Q1798526` bakery
- `Q19844914` jazz club
- `Q187456` nightclub

### `event`
- `Q24716587` recurring event
- `Q1190554` occurrence (events)
- `Q132241` festival
- `Q15275719` music festival
- `Q3329581` theatre festival
- `Q132834` concert
- `Q15275719` exhibition
- `Q210272` parade

### `dish`
- `Q746549` dish (food)
- `Q2095` food
- `Q21034321` regional dish
- `Q3343352` ingredient

### `seasonal`
- (open — no canonical Wikidata Q-IDs for seasonal mood imagery; the resolver should not invoke `wikidata-verified` for this archetype.)

### `neighborhood`
- (currently routed through `destination`; uses the `destination` Q-ID set.)

## Matching algorithm (reference pseudocode)

```javascript
function archetypeMatchesP31(archetype, p31Values, p31TableForArchetype) {
  // p31Values: array of Q-IDs from candidate.claims.P31[].mainsnak.datavalue.value.id
  // p31TableForArchetype: array of canonical Q-IDs from this file for the archetype
  return p31Values.some(q => p31TableForArchetype.includes(q));
}
```

## Region match (P131 / P17)

The resolver also gates on region match. For a candidate to pass:

- If the trip's `region` parses to a country (e.g., "BC" → Canada → Q16) → the candidate's P17 (country) must equal the country Q-ID, OR P131 (located in) must transitively reach a country-named entity. The verifier walks P131 up to depth 5; if no match, reject as `region-mismatch`.
- If the trip's `region` is a city/region within a country (e.g., "Victoria, BC") → both P17 and P131 are checked; P131 should resolve to a Q-ID whose label contains the region string OR an ancestor entity whose label does.

When the trip's `region` is missing or ambiguous (e.g., a non-geocodable string), region verification is **skipped** — `wikidata-verified` candidates pass on P31-only with `confidence: "medium"` (downgraded from default `high`).

## When to update this table

When a real-world place is misresolved by the verifier because its P31 isn't in the table, add the missing Q-ID to the appropriate archetype. Always cite the misresolved entity in the commit message so future readers understand the empirical motivation.
