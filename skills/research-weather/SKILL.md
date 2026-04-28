---
name: research-weather
description: Research weather and climate for your trip window
---

# Research Weather

Run live web research on the climate and weather forecast for the active trip's destination during the trip dates. Write the findings to `research/weather.md`.

## Inputs

**Required:** `<slug>/trip.yaml` (resolved via the active-trip pointer)

**Optional:** none

## Output

`.myfootmarks/trips/<slug>/research/weather.md` — markdown document with YAML frontmatter carrying a single-entry `places:` array (representing "weather for these trip dates") plus the human-readable body.

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` to get the slug. Read `<slug>/trip.yaml` and parse it. Extract `destination`, `start_date`, `end_date`, and `travelers`.

If `.myfootmarks/current-trip` is missing or `trip.yaml` is missing/malformed, stop and tell the user to run `/myfootmarks:intake` first.

### Step 2: Gather enrichment context

This skill has no optional inputs. Proceed.

### Step 3: Dispatch the research subagent

Use the Agent tool with `subagent_type: general-purpose`. Pass this prompt (substituting the bracketed values):

```
You are researching weather and climate for a trip.

USE WEBSEARCH AND WEBFETCH FOR LIVE DATA. Do not rely on training data — weather forecasts and seasonal patterns change.

Trip:
- Destination: <destination>
- Dates: <start_date> to <end_date> (inclusive)
- Travelers: <names with ages>

Return your findings as TWO blocks: a structured `places:` YAML array (single synthetic entry) with a nested `daily_forecast:` array, AND a human-readable Markdown body.

## Shared schema: `places:` frontmatter (single synthetic entry)

Unlike other research skills, research-weather emits exactly ONE `places:` entry per trip. This entry represents "weather for these dates at this destination."

```yaml
places:
  - id: weather-<slug-of-destination>    # e.g., "weather-victoria-bc"
    name: "Weather for <destination>, <start_date>–<end_date>"
    wikidata_id: null
    lat: null
    lon: null
    category: weather-forecast
    lens: [weather]
    neighborhood: null
    address_street: null
    phone: null
    hours_structured:
      mon: null
      tue: null
      wed: null
      thu: null
      fri: null
      sat: null
      sun: null
    reservation_url: null
    reservation_required: false
    lead_time_days: null
    signature: <one-line summary of the overall weather pattern for the trip>
    source_url: <primary forecast source URL>
    stroller: true
    with_kids: true
    kid_first: false
    daily_forecast:                     # NON-STANDARD field unique to research-weather
      - date: YYYY-MM-DD
        icon: sunny | partly-sunny | overcast | rain | snow | mixed
        high_c: <int>                   # degrees Celsius
        low_c: <int>
        summary: <string>               # one-line forecast — e.g., "Showers in the afternoon"
      # … one entry per day from start_date through end_date (inclusive)
```

`lens: [weather]`. `daily_forecast` is a required non-standard field: every day from `start_date` to `end_date` inclusive must appear. If live forecast data isn't available for a given day, set `icon: "mixed"` and write a climatology-based `summary` (typical weather for that date), not `null`.

## Markdown body

Render the same weather information as human-readable Markdown:

## Summary

A concise structured summary at the top:
- Typical high/low temperature range for the trip window (specify units)
- Precipitation likelihood (e.g., "rain expected on ~3 of 4 days")
- Notable patterns (afternoon storms, coastal fog, wind, smoke season, etc.)

## Detail

Narrative paragraphs covering:
- What layers and gear travelers should plan to bring
- When the weather typically shifts during a day
- Anything unusual for this season (heat waves, storm season, smoke from wildfires, etc.)

Cite sources inline as Markdown links: e.g., "Average highs in early May are 18°C ([Environment Canada](https://...))."
```

### Step 4: Render output

Take the subagent's response (both the single-entry `places:` array with `daily_forecast` and the Markdown body) and write it to `.myfootmarks/trips/<slug>/research/weather.md` with this YAML frontmatter:

```yaml
---
topic: weather
generated_at: <ISO 8601 UTC timestamp, e.g., 2026-04-22T18:30:00Z>
consumed_inputs: [trip.yaml]
places:
  # inline the single-entry places: array (with nested daily_forecast) from the subagent verbatim
---
```

Below the frontmatter, write the Markdown body unchanged.

### Step 5: Log the run

Append one JSON line to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "research-weather", "slug": "<slug>", "status": "ok", "consumed_inputs": ["trip.yaml"], "outputs_written": [".myfootmarks/trips/<slug>/research/weather.md"]}
```

If the subagent failed or returned malformed output:
- Do NOT write the output file.
- Append a JSON line with `"status": "error"` and a `"reason": "<short message>"` field instead.
- Tell the user what happened and offer to retry.

Confirm to the user the path of the output file written.
