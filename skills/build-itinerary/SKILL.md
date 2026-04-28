---
name: build-itinerary
description: Plan your day-by-day itinerary from research
---

# Build Itinerary

Read all available research for the active trip and produce a day-by-day itinerary. LLM-driven synthesis — no scoring algorithm. Output is markdown that downstream `build-document` renders to HTML.

## Inputs

**Required:**
- `Trips/<slug>/trip.yaml`
- At least one research file (typically all five base research files; ideally also `restaurants-final.md`)

**Optional (will use any present):**
- `.myfootmarks/trips/<slug>/research/weather.md`
- `.myfootmarks/trips/<slug>/research/events.md`
- `.myfootmarks/trips/<slug>/research/local-foods.md`
- `.myfootmarks/trips/<slug>/research/attractions.md`
- `.myfootmarks/trips/<slug>/research/restaurants-final.md` (preferred over `restaurants.md`)
- `.myfootmarks/trips/<slug>/research/restaurants.md` (fallback when `restaurants-final.md` is missing)

## Output

`Trips/<slug>/itinerary.md`

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract destination, dates, travelers (with ages and interest weights), home_base, regions, pace (default `moderate`).

If `trip.yaml` is missing or malformed, bail to `/myfootmarks:intake`.

### Step 2: Load whatever research is available

Check for each potential input file. Read the ones present. Track exactly which files you read.

For restaurants: prefer `restaurants-final.md`. If absent, fall back to `restaurants.md`. If neither is present, you can still proceed but the itinerary won't include dinner picks — note this in the itinerary's output (see step 4).

If NO research files are present at all, stop and tell the user:

> No research files yet — run `/myfootmarks:research-weather` (and friends) before building the itinerary.

### Step 3: Synthesize the day-by-day plan (LLM reasoning, no subagent)

The following reasoning steps are MANDATORY and MUST be executed in the stated order. Each step's output feeds the next; do not reorder, skip, or merge steps.

1. **Extract constraints.** Trip dates → list of days (one per date). Travelers + ages → group composition. `pace` → target stops per day (relaxed=2–3, moderate=3–5, packed=5–7). Home base → starting/ending neighborhood. If any `trip.yaml` field is missing, ambiguous, or contradicts another field, record the gap explicitly in the itinerary's output header (as an HTML comment) so the user can see what was inferred and what was missing.

2. **Anchor events first.** From `events.md` (if present), find events that fall within the trip window. Lock those events to their specific dates. If an event is evening-time, that day's dinner slot is locked near it.

3. **Shape by weather.** From `weather.md` (if present), identify rainy vs. clear days. Schedule outdoor attractions on clear days; indoor (museums, galleries, indoor markets) on rainy.

4. **Shape by group composition.** Kids under ~10 in the party → start no later than 9:30 AM, dinner by 6:30 PM, mostly walking-friendly stops, plan a midday rest break. School-age kids and up → more flexibility. Adults only → late dinners and longer evenings are fine.

5. **Shape by interests.** Use traveler interest weights from `trip.yaml` to bias selection. When multiple travelers have different dominant interests (top weight within their own interests list, where the magnitude difference is ≥0.3 between first and second), alternate interest-anchored days rather than averaging. Example: if traveler A weights "hiking: 0.9, museums: 0.5" and traveler B weights "museums: 0.9, hiking: 0.5", the plan alternates hiking-anchored and museum-anchored days rather than pairing a half-hike + half-museum every day. When interest weights are flat (all ≤0.3 apart), fall back to balanced selection.

6. **Colocate geographically.** Use neighborhood and address info from `attractions.md` and `restaurants-final.md` (or `restaurants.md`). Each day's stops MUST share a neighborhood cluster. The only override reasons are: (a) an anchored event requires traveling to a different neighborhood, or (b) a specific restaurant pick from `restaurants-final.md` is in a different neighborhood for a justified reason (signature dish availability, etc.). When an override applies, state the reason in that stop's 'why' line. No distance matrix needed — reason from neighborhood proximity.

7. **Slot meals.** For each day:
   - Breakfast: near home_base or quick cafe near first activity (don't over-program)
   - Lunch: casual, near the day's main activity
   - Dinner: featured restaurant pick from `restaurants-final.md` (or fallback). Choose based on day's neighborhood, group composition, and any anchored evening events.

8. **Per-day narrative.** Write a 1–2 sentence narrative blurb for each day describing the theme (e.g., "Arrival day — settle in and explore the harbor at an easy pace" or "Coastal drive day — Sooke and back, scenic stops").

### Step 4: Render output

Write `Trips/<slug>/itinerary.md` with this structure:

```markdown
# Itinerary — <destination>, <start_date> to <end_date>

[If restaurants-final.md was missing and you fell back to restaurants.md:]
<!-- note: synth-restaurants not run; using base restaurants.md, kid-friendly and local-cuisine specializations not merged -->

## Day 1 — <Day-of-week>, <Date>

**Weather:** <one-line summary from weather.md, e.g., "Partly cloudy, high 18°C, light wind">

**Theme:** <1–2 sentence narrative blurb>

- **9:00 AM — <Activity name>** (<neighborhood>)
  - Why: <one-line reason>
- **12:30 PM — Lunch at <Restaurant name>** (<neighborhood>)
  - Why: <one-line reason>
- **2:00 PM — <Activity name>** (<neighborhood>)
  - Why: <one-line reason>
- **6:30 PM — Dinner at <Restaurant name>** (<neighborhood>)
  - Why: <one-line reason — call out signature dish if applicable>

## Day 2 — <Day-of-week>, <Date>

(... same structure ...)
```

Each `Why:` line is required — it's how the user understands why each pick was chosen.

Each day MUST have at a minimum: a morning activity (9:00-11:00 AM), lunch (12:00-13:30), an afternoon activity (14:00-17:00), and dinner (17:00-19:30 for family trips; 18:30-21:00 for adult-only trips) — UNLESS the trip's `pace` is explicitly `relaxed` (allowing half-days), or a specific constraint (travel day, anchored event, group exhaustion) overrides the requirement. The LLM MUST state any override reason as part of that day's theme blurb.

### Step 5: Log the run

Append to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "build-itinerary", "slug": "<slug>", "status": "ok", "consumed_inputs": [<actual list of files read>], "outputs_written": ["Trips/<slug>/itinerary.md"]}
```

If you fell back to base restaurants.md (synth wasn't run), include `"reason": "synth-restaurants not run; used base restaurants.md"` in the run entry.

Confirm to the user the path of the output file.
