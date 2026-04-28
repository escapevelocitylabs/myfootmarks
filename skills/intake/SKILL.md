---
name: intake
description: Start a new trip — collect destination, dates, travelers, and interests
---

# Intake

Collect the parameters for a new trip and write the canonical `trip.yaml` plus directory scaffolding so the rest of the pipeline can run.

This skill is interactive — it prompts the user for input. It does NOT dispatch a subagent.

## Inputs

- User responses to interactive prompts.
- Working directory (cwd) — all filesystem operations are relative to it.

## Outputs

- `Trips/<slug>/trip.yaml`
- `.myfootmarks/current-trip` (plain text containing the slug)
- `.myfootmarks/trips/<slug>/research/` (empty directory)
- `.myfootmarks/trips/<slug>/runs.jsonl` (empty file)

## Steps

### Step 1: Collect trip parameters interactively

Ask the user for each of the following, one at a time. Validate as you go:

1. **Destination** — natural-language string (e.g., "Victoria, BC", "Kyoto, Japan").
2. **Start date** — ISO 8601 date (e.g., `2026-05-03`). Validate as a real calendar date.
3. **End date** — ISO 8601 date, inclusive. Must be on or after `start_date`.
4. **Travelers** — for each traveler:
   - Name (string, can be a nickname)
   - Age in years (integer)
   - Interests (list of strings; ask for an optional weight per interest, default 0.5 if omitted)
   Loop until the user says they have no more travelers to add.
5. **Home base** — natural-language description of where the travelers will stay (neighborhood, hotel, address — whatever is known).

Then ask the optional questions (with the defaults clearly noted):

6. **Regions to include** — optional list of areas (e.g., `["downtown", "Oak Bay", "Sooke"]`). Default: empty (research will use destination-wide scope).
7. **Pace** — `relaxed`, `moderate`, or `packed`. Default: `moderate`.
8. **Budget tier** — `shoestring`, `moderate`, `comfortable`, or `luxurious`. Default: `moderate`.
9. **Include research appendix in the final HTML?** — boolean. Default: `false`.
10. **Trip-book template** — which visual voice to render the final trip-book in. v1 ships a single template:
    - `modern-magazine` (default) — editorial-magazine voice: red accent, Inter + Source Serif 4, single-column, hard-offset shadows.

    Write the selected value as `template:` on the trip.yaml. Future versions will expand the catalog and this becomes a multi-choice picker presenting each voice's one-line summary before the user chooses.

### Step 2: Derive the trip slug

Build a slug from `destination` + `start_date`. Lowercase, replace spaces and commas with hyphens, strip non-alphanumeric (except hyphens), and append `-YYYY-MM` from the start date.

Examples:
- `"Victoria, BC"` + `2026-05-03` → `victoria-bc-2026-05`
- `"Kyoto, Japan"` + `2026-09-15` → `kyoto-japan-2026-09`

If `Trips/<slug>/` already exists, ask the user to choose:
- **Reactivate the existing trip** — set `.myfootmarks/current-trip` to this slug and stop without overwriting trip.yaml.
- **Pick a different slug** — let the user provide a custom slug (validate it's `kebab-case`).
- **Overwrite** — proceed and overwrite the existing trip.yaml. Confirm explicitly first.

### Step 3: Write `Trips/<slug>/trip.yaml`

Create `Trips/<slug>/` if it doesn't exist. Write `trip.yaml` with this shape:

```yaml
destination: "<destination>"
start_date: <start_date>
end_date: <end_date>
travelers:
  - name: <name>
    age_years: <integer>
    interests:
      - <interest>: <weight>
home_base: "<home_base>"
template: modern-magazine     # Required. In v1, the only supported value. Future versions will accept other templates from the catalog.
# Optional fields — include only if user provided non-default values:
regions: [<list>]
pace: <pace>
budget_tier: <budget_tier>
include_research_appendix: <bool>
```

Quote string values that contain spaces, commas, or colons.

### Optional: `image_overrides:`

`intake` does NOT interactively prompt for `image_overrides:`. The field is added by hand by the trip planner **after** the first build, as a residual-correction mechanism for places where the automated resolver picked a wrong image (or none at all).

Documented here for discoverability — when a trip planner asks "how do I fix this card's photo?", the answer lives in the documentation that produced their `trip.yaml`.

#### Field shape

```yaml
image_overrides:
  <place-or-dish-id>: <override-value>
```

Place IDs are the stable kebab-case slugs that appear in `book-data.json` `places[].id` (e.g., `butchart-gardens`, `bc-parliament-buildings`, `red-fish-blue-fish`). Dish IDs come from `dishes[].id` in the same file.

#### The four override-value forms

```yaml
image_overrides:
  # 1. Commons file URL → resolved to thumbnail + Commons attribution.
  butchart-gardens: "https://commons.wikimedia.org/wiki/File:Sunken_Garden_at_Butchart_Gardens.jpg"

  # 2. Raw Commons filename (must end in an image extension) → same as above without URL parsing.
  bc-parliament-buildings: "British_Columbia_Parliament_Buildings_at_night.jpg"

  # 3. Arbitrary HTTPS URL to any image → direct download, attribution = "user-provided".
  red-fish-blue-fish: "https://example.com/our-photo-of-red-fish.jpg"

  # 4. The literal string "skip" → forces typography-only rendering (no image, no attempted resolution).
  obscure-local-cafe: "skip"
```

#### What happens on the next `build-trip-book` run

- **Override present and valid** → image is downloaded (or skipped, for `"skip"`); audit table prints `source: override-commons | override-url | override-skip`; resolver is NOT invoked for that entity.
- **Override present and invalid** (404, malformed URL, non-image MIME) → audit table prints `fetch-failed: <reason>`. **The build does NOT silently fall back to auto-resolution** — the user's intent is load-bearing.
- **Override removed** (entry deleted from `image_overrides:`) → next build re-runs auto-resolution for that place.
- **Override key matches no place ID** → audit table prints a warning row indicating the unknown key. The override is ignored.

#### Re-running after adding overrides

```bash
/myfootmarks:build-trip-book
```

Stage 2 (`fetch-assets`) is the only stage affected. Stages 1, 3, and 4 are skipped if their inputs haven't otherwise changed (Stage 2 caches per-entity downloads, so unaffected places are not re-fetched).

#### Diagnostic: why did this place resolve the way it did?

```
/myfootmarks:find-place-image
{ "name": "<entity>", "category": "<category>", "region": "<destination>" }
```

Read-only — prints the resolver trace + JSON envelope. Use this before adding an override to understand what the automation tried.

### Step 4: Create directory scaffolding

Create these paths if missing:

- `.myfootmarks/trips/<slug>/research/` (empty directory)
- `.myfootmarks/trips/<slug>/runs.jsonl` (empty file — `touch` it)

### Step 5: Set the active-trip pointer

Write the slug (just the slug, no trailing newline rules — keep simple) to `.myfootmarks/current-trip`. This file is the active-trip pointer that subsequent skills read.

### Step 6: Confirm and hint the next step

Tell the user the trip is set up at `Trips/<slug>/trip.yaml` and that they can run `/myfootmarks:plan-trip` to begin the research and build pipeline.

## Notes

- Do NOT dispatch any subagent.
- Do NOT touch existing files outside `Trips/<slug>/`, `.myfootmarks/trips/<slug>/`, and `.myfootmarks/current-trip`.
- If the user aborts mid-prompt, do not write any files.
