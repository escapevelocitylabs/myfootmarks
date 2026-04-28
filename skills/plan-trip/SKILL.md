---
name: plan-trip
description: Run the full trip-planning pipeline end-to-end
---

# Plan Trip — Orchestrator

Drive the trip-planning pipeline from intake through final HTML. Reads filesystem state, determines the current phase, dispatches the next skill(s), pauses for user review between phases. Idempotent — re-invoking picks up wherever things are.

This is a state machine OVER THE FILESYSTEM. The orchestrator is itself stateless; persistence is in the on-disk file structure.

## Inputs

- `.myfootmarks/current-trip` (the active trip slug)
- The state of files under `<slug>/` and `.myfootmarks/trips/<slug>/`

## Outputs

- Dispatches other skills (each writes its own outputs)
- Surfaces phase progression and failures to the user

## Steps

### Step 0: Preflight — verify web access

Concrete probe:
- Attempt one trivial WebSearch query (something simple about the destination once we know it; or "today's date" if we don't yet).
- Attempt one WebFetch against a known-stable URL (e.g., `https://www.wikipedia.org`).

If either tool is unavailable or fails:
- Tell the user research skills can't run without live web access.
- Stop. Do NOT proceed.

### Step 1: Intake check

Verify `.myfootmarks/current-trip` exists and is non-empty. Read the slug.

Verify `<slug>/trip.yaml` exists and parses as YAML.

If either fails:
- Tell the user to run `/myfootmarks:intake` first.
- Stop.

Extract from trip.yaml: `travelers` list (specifically each traveler's `age_years`, used for specialization triggers).

### Step 2: Detect current phase

Determine the current pipeline phase by checking which files exist:

| Files present in `.myfootmarks/trips/<slug>/research/` and `<slug>/` | Phase |
|---|---|
| Fewer than all 12 of: `weather.md`, `events.md`, `local-foods.md`, `attractions.md`, `restaurants.md`, `outdoor.md`, `scenic.md`, `daytrips.md`, `shopping.md`, `nightlife.md`, `music.md`, `arts.md` | `base_research` |
| All 12 base research files present, but specializations have not been evaluated | `specializations` |
| Specializations evaluated; `restaurants-*.md` exists but `restaurants-final.md` does not | `synth` |
| `restaurants-final.md` exists (or no specializations ran) but `<slug>/itinerary.md` is missing | `itinerary` |
| `itinerary.md` exists but any of the three checklist files is missing | `checklists` |
| All checklists exist but `.myfootmarks/trips/<slug>/research/destination-primer.md` is missing | `primer` |
| Primer exists but `<slug>/trip-book.html` is missing | `document` |
| All outputs exist | `done` |

(For "specializations evaluated" — heuristic: check whether you previously logged a `skipped` entry for either specialization in `runs.jsonl` for the current trip, OR whether any specialization output file exists. If neither: still in specializations phase.)

### Step 3: Dispatch the current phase

Each phase has its own dispatch logic and pause behavior:

#### Phase: `base_research`

Identify which of the 12 base research files are missing. For each missing one, dispatch the corresponding skill. The full set is:

- MVP base skills: `weather.md`, `events.md`, `local-foods.md`, `attractions.md`, `restaurants.md`
- Expanded base skills: `outdoor.md`, `scenic.md`, `daytrips.md`, `shopping.md`, `nightlife.md`, `music.md`, `arts.md`

CRITICAL: dispatch in PARALLEL via the Agent tool. Issue ONE message containing multiple Agent tool calls — do NOT call them sequentially. Cap at 4 concurrent dispatches. With 12 base skills, a full first-time run completes in 3 waves of 4 (issue wave 1 as one message with 4 Agent calls, wait, then wave 2, wait, then wave 3).

Wait for all dispatches to complete. Inspect `runs.jsonl` entries from each. Surface any failures.

After all base research is complete (success or surfaced failure): pause for user review. Tell the user which files were generated and let them inspect anything before proceeding.

If the user says "continue", advance to the next phase. If they say "stop", end the orchestrator (they can re-invoke later).

#### Phase: `specializations`

Evaluate specialization triggers:

- If `.myfootmarks/trips/<slug>/research/local-foods.md` exists AND `restaurants-local-cuisine.md` does NOT exist:
  - Dispatch `research-restaurants-local-cuisine` via the Agent tool.
- If `trip.yaml`'s travelers include any with `age_years < 13` AND `restaurants-kid-friendly.md` does NOT exist:
  - Dispatch `research-restaurants-kid-friendly` via the Agent tool.

Dispatch any applicable specializations IN PARALLEL (single message with multiple Agent calls).

Wait for completion. Inspect `runs.jsonl` for `status: skipped` (specialization opted not to run, fine) vs. `status: error` (real failure, surface).

Pause for user review. Continue or stop on user response.

#### Phase: `synth`

If `restaurants-*.md` specialization files exist (besides `restaurants-final.md`) AND `restaurants-final.md` is missing:
- Dispatch `synth-restaurants` via the Agent tool.

If no specialization files exist, skip synth — the base `restaurants.md` will be used directly by `build-itinerary` as a fallback. Note this and proceed.

No user pause for synth — it's a fast, non-controversial merge.

#### Phase: `itinerary`

If `<slug>/itinerary.md` is missing:
- Dispatch `build-itinerary` via the Agent tool.

Wait for completion. Surface any error. Pause for user review (they may want to inspect the itinerary).

#### Phase: `checklists`

If any of `packing.md`, `pre-trip.md`, `day-of.md` is missing in `<slug>/`:
- Dispatch `build-checklists` via the Agent tool.

Wait. Surface any error. Pause.

#### Phase: `primer`

If `.myfootmarks/trips/<slug>/research/destination-primer.md` is missing:
- Dispatch `research-destination-primer` via the Agent tool.

The primer is the narrative gateway for the trip book — destination setting, history, cultural notes, per-home-base-region blocks, and seasonal mood. It also resolves the `hero_wikidata_id` that seeds the Cover image.

Wait. Surface any error. Pause for user review — the user may want to inspect the primer before the final render since it sets the book's voice.

#### Phase: `document`

If any of `<slug>/trip-itinerary.html`, `<slug>/trip-checklists.html`, or `<slug>/trip-book.html` is missing:
- Dispatch `build-trip-book` via the Agent tool. Unconditional — there is no fallback path. `build-trip-book`'s Stage 4 applies an all-or-nothing check: re-renders all three HTML outputs if any is missing or any input is newer than any output.

`build-trip-book` internally orchestrates a 4-stage subagent pipeline (normalize → fetch-assets → generate-maps → render-html). Expect Wikidata/Commons/Overpass network calls during the middle stages; a first-time end-to-end render for a typical 5-day trip takes 60–180 seconds of wall-time.

Wait. Surface any error. Pause for user review.

#### Phase: `done`

All outputs are present. Tell the user:

> Trip planning complete. Three documents are ready in `<slug>/`:
>
> - **`trip-itinerary.html`** — bring this one with you during the trip. Daily plan, per-slot field cards, during-trip quick lookups, back cover with emergency + lodging + transport info.
> - **`trip-checklists.html`** — print this one before the trip and cross off tasks. Reservations by urgency, then pre-trip / packing / day-of lists, each on its own page. B&W-safe.
> - **`trip-book.html`** — the keepsake. Read ahead or leave at the hotel. Destination primer, featured places as story cards, what-to-eat, by-neighborhood, with/without kids, appendix.
>
> Use your browser's print-to-PDF feature to save or print each doc independently. Each doc stands fully alone — no cross-file links.
>
> To regenerate any specific stage, invoke that skill directly (e.g., `/myfootmarks:research-weather`, `/myfootmarks:research-destination-primer`, `/myfootmarks:build-trip-book`). Re-invoking `plan-trip` is a no-op when everything is in place.

### Step 4: Handle failures

Across all phases: if any dispatched skill reports `status: error` in `runs.jsonl`, surface the failure to the user with a clear menu:

```
Failed skills this phase:
  - research-events: <error reason>

Options:
  [1] Retry failed
  [2] Skip and continue
  [3] Inspect errors (show full runs.jsonl entries)
  [4] Stop
```

Act on the user's choice. Stay within the current phase until either: all skills succeeded OR user chose Skip OR user chose Stop.

### Step 5: Loop or exit

After acting on the current phase, return to step 2 (re-detect phase) and process the next one. Continue until the phase is `done` or the user stopped.

## Notes

- This skill is IDEMPOTENT. Re-invoking after any partial completion safely picks up at the next incomplete phase.
- This skill does NOT force re-runs. To force a fresh run of an individual skill, the user invokes that skill directly.
- Filesystem state is the only state. No memory or session state persists.
- Parallel dispatches in `base_research` and `specializations` phases require a single LLM message with multiple Agent tool calls. Sequential dispatches would be much slower and miss the point.
