---
name: build-checklists
description: Generate packing, pre-trip, and day-of checklists
---

# Build Checklists

Produce three checklist files in one skill invocation: `packing.md`, `pre-trip.md`, `day-of.md`. LLM-driven, no subagent dispatch. Tailored by trip parameters and weather; events shape packing if relevant (e.g., formal wear for a gala).

## Inputs

**Required:** `Trips/<slug>/trip.yaml`

**Optional (will use any present):**
- `.myfootmarks/trips/<slug>/research/weather.md`
- `.myfootmarks/trips/<slug>/research/events.md`

## Outputs

- `Trips/<slug>/packing.md`
- `Trips/<slug>/pre-trip.md`
- `Trips/<slug>/day-of.md`

## Steps

### Step 1: Resolve the active trip

Read `.myfootmarks/current-trip` → slug → `Trips/<slug>/trip.yaml`. Extract destination, dates, travelers, home_base, regions. Bail to `/myfootmarks:intake` if missing.

### Step 2: Load enrichment

Read `weather.md` if present — note temperature range, precipitation, special weather considerations.

Read `events.md` if present — scan for events implying special attire or gear.

### Step 3: Generate the three checklists (LLM reasoning, no subagent)

Reason through each checklist with cross-references. Use `- [ ]` markdown checkboxes. Group items by category within each file.

### Step 4: Write `Trips/<slug>/packing.md`

Structure:

```markdown
# Packing list — <destination> (<start_date> to <end_date>)

[Brief intro — 1 sentence noting weather expectations or special considerations]

## Documents
- [ ] Passports / IDs (one per traveler)
- [ ] Travel insurance card
- [ ] Booking confirmations (printed or saved offline)
- (... etc.)

## Clothing
[Tailored by weather + group composition. Layers if weather is variable. Rain gear if precipitation expected. Formal wear if events.md mentions a black-tie occasion.]
- [ ] <items>

## Toiletries
- [ ] <items>

## Electronics
- [ ] <items>

## Kids gear (only if travelers include children)
- [ ] <stroller, car seat, kid-specific snacks, etc.>

## Optional / nice-to-have
- [ ] <items>
```

### Step 5: Write `Trips/<slug>/pre-trip.md`

Structure:

```markdown
# Pre-trip checklist — <destination> (<start_date> to <end_date>)

[Brief intro — when to start ticking these off, e.g., "Most items are 1–2 weeks out; flagged items are months out."]

## Bookings (months out)
- [ ] Book accommodations
- [ ] Book flights / transportation
- [ ] Reserve any restaurants requiring advance booking (cross-reference `itinerary.md`)
- [ ] Book any tickets for events listed in events.md

## Documents (1–2 months out)
- [ ] Verify passport expiration (must be valid 6+ months from return for many destinations)
- [ ] Check visa requirements for the destination
- (... etc.)

## Logistics (1–2 weeks out)
- [ ] Notify bank of travel dates (avoid card freezes)
- [ ] Arrange pet/plant care
- [ ] Set up mail hold / package delivery pause
- (... etc.)

## Downloads (a few days out)
- [ ] Download offline maps
- [ ] Download streaming content for travel days
- (... etc.)
```

Cross-reference `packing.md` where appropriate ("packing list says bring passport — check expiry here").

### Step 6: Write `Trips/<slug>/day-of.md`

Structure:

```markdown
# Day of travel — <destination> (<start_date>)

## Morning before departure
- [ ] Charge devices to 100%
- [ ] Confirm flight / transit status
- [ ] Final packing pass (compare against packing.md)
- (... etc.)

## At the airport / station
- [ ] Carry-on essentials within reach: passport, boarding pass, phone, charger, headphones, snacks for kids
- (... etc.)

## On arrival
- [ ] Connect to wifi / activate eSIM
- [ ] Get cash if needed (ATM at airport often has best rate)
- [ ] Confirm hotel / home_base directions
- [ ] Send "we landed safely" message
- (... etc.)
```

### Step 7: Log the run

Append one entry to `.myfootmarks/trips/<slug>/runs.jsonl`:

```json
{"timestamp": "<ISO 8601>", "skill": "build-checklists", "slug": "<slug>", "status": "ok", "consumed_inputs": [<actual files read>], "outputs_written": ["Trips/<slug>/packing.md", "Trips/<slug>/pre-trip.md", "Trips/<slug>/day-of.md"]}
```

Confirm to the user the three paths.
