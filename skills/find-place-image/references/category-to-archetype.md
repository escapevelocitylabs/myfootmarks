# Category → archetype mapping

The resolver's first step is to map the caller's free-form `category` string to one of seven archetypes. The archetype determines which source stack runs (see SKILL.md → "Source stacks").

## Mapping table

| Archetype | Categories |
|---|---|
| `destination` | `destination-city`, `region`, `day-trip-region`, `neighborhood`, `food-neighborhood`, `historic-neighborhood-street`, `arts-district` |
| `landmark` | `landmark`, `museum`, `art-museum`, `castle`, `historic-site`, `park`, `city-park`, `regional-park`, `nature-reserve`, `garden`, `waterfront`, `viewpoint`, `sunset-spot`, `concert-hall`, `theater-venue`, `arena`, `aquarium`, `zoo`, `hiking-trail`, `petting-zoo` |
| `restaurant` | any `restaurant-*`, any `cafe-*`, any `bistro-*`, `brewpub`, `bakery-cafe`, `seafood`, `seafood-casual`, `japanese-sushi`, `italian`, `italian-family`, `french-bistro`, `heritage-restaurant`, any `diner-*`, any `fine-dining-*`, `fishmonger-counter`, `specialty-food`, `specialty-retail`, `tearoom-breakfast`, `pub`, any `pub-*`, `wine-bar`, `wine-bar-bistro`, `cocktail-bar`, `speakeasy`, `jazz-club`, `singer-songwriter-room`, `rock-club` |
| `event` | `theater`, `theater-performance-event`, `concert`, `music-event-dated`, `exhibition-dated`, `festival`, `festival-annual`, `parade`, `parade-annual`, `sports-event`, `tour`, `water-activity`, `fair` |
| `dish` | (no `category` triggers this — dishes come exclusively via the `dishes:` array from `research-local-foods`; the caller passes `category: "dish"` explicitly) |
| `seasonal` | (no `category` triggers this today; reserved for future weather-imagery use cases) |
| `neighborhood` | (currently routed through `destination`; may split later) |

## Matching rules

1. **Exact match wins.** Compare the caller's `category` string case-insensitively against the table.
2. **Wildcard segments.** `restaurant-*`, `cafe-*`, `bistro-*`, `diner-*`, `fine-dining-*`, `pub-*` all map to `restaurant`. The `*` is a literal kebab-case suffix segment, e.g., `restaurant-fine-dining`, `cafe-third-wave`, `pub-gastro`.
3. **Unfamiliar category.** Fall back to the `landmark` archetype with `confidence: "low"` carried through the entire resolution. The audit table will flag this as a spot-check candidate.

## When to update this table

If a research skill is migrated and its `Skill-specific guidance` lists a category string not in this table, add the new string to the appropriate archetype row. Do NOT add an entirely new archetype without empirical evidence (a comparison prototype demonstrating that a new source stack outperforms an existing one for that category).

## Examples (smoke-tests for the table)

| Caller passes | Resolves to archetype |
|---|---|
| `category: "garden"` | `landmark` |
| `category: "Garden"` (case mismatch) | `landmark` (case-insensitive) |
| `category: "restaurant-fine-dining"` (wildcard) | `restaurant` |
| `category: "destination-city"` | `destination` |
| `category: "historic-neighborhood-street"` | `destination` |
| `category: "festival-annual"` | `event` |
| `category: "dish"` (explicit by caller) | `dish` |
| `category: "fictional-thing"` (unknown) | `landmark` with `confidence: "low"` |
