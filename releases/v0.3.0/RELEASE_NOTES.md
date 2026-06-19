# v0.3.0 — Dynamic valve list, next-run countdown, zone-exclude dot

## New features

- **Dynamic valve list (1–5 zones)** — valves are no longer hard-coded. Each zone lives in a numbered slot backed by three helpers:
  - `input_text.garden_valve_N_entity` — the valve switch entity
  - `input_text.garden_valve_N_name` — the zone display label
  - `input_number.garden_valve_N_duration` — run time (0–60 min, step 5)

  The card iterates slots 1–5 and renders a zone row only for slots whose entity is filled, so you can start with one valve and grow to five with no card or script changes.
- **Next-run countdown** — below the next-run date/time, a live countdown shows `in 6d 22h 10m` (the days component only appears when the run is ≥ 24h away). Computed in Jinja2 from the days-ahead offset and the start time, using `//` and subtraction — no `%` modulo, consistent with the card's defensive templating.
- **Zone-exclude dot** — a red ✗ dot on each zone row sets that zone's duration to 0, excluding it from the run while keeping the row visible. The Total run line shows `15+10` for multiple zones, `1 zone` for a single zone, or `—` when none are active.

## Changes

- **`script.garden_watering_sequence` rewritten** — replaced the two hard-coded upper/lower blocks with five slot-based `if/then` steps. Each step turns on the templated valve entity via `homeassistant.turn_on`, waits the slot's duration, then turns it off. Slots with duration 0 are skipped by a `numeric_state … above: 0` guard.
- **Zone duration label width** `28px → 40px` so double-digit values like "20 min" no longer wrap.

## Removed

- `input_number.garden_water_upper_duration` / `_lower_duration` — superseded by the per-slot duration helpers. The per-zone `timer.*` helpers previously earmarked for v0.3.0 are deferred to v0.5.0 and are not required.

## Upgrade from v0.2.x

1. Create the 15 valve-slot helpers (entity / name / duration × 5). Put your old upper-lawn values in slot 1 and lower-lawn in slot 2; leave slots 3–5 blank to keep the same two-zone behaviour.
2. Replace your existing card YAML with `releases/v0.3.0/card.yaml`.
3. Update `script.garden_watering_sequence` to the slot-based version (or ask Claude to do it).

The Claude-assisted install discovers your valve switches and fills the slots for you — see INSTALLATION.md, Option A.
