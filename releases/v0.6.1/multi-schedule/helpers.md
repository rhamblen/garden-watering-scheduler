# Multi-schedule helpers (A & B)

This bundle adds a **second independent schedule**. Per the agreed design
(`docs/design-multiple-schedules.md`, Option 3 hybrid namespace), only the
**per-schedule** state is namespaced (`garden_a_*` / `garden_b_*`). House-wide
concepts stay **shared** (single instance).

---

## Per-schedule helpers — 25 per schedule

Create the full set for each of `a` and `b` (the table shows `a`; repeat with
`b`). Types/ranges are identical to the current singleton helpers.

| Helper | Type | Notes |
|--------|------|-------|
| `input_boolean.garden_a_water_mon` … `_sun` | Toggle × 7 | Day-of-week schedule |
| `input_select.garden_a_water_start_time` | Dropdown | 48 options `00:00`…`23:30` (clone the existing option list) |
| `input_text.garden_a_valve_1_entity` … `_5_entity` | Text × 5 | Valve switch entity id per slot (blank = slot unused) |
| `input_text.garden_a_valve_1_name` … `_5_name` | Text × 5 | Zone display name per slot |
| `input_number.garden_a_valve_1_duration` … `_5_duration` | Number × 5 | 0–60, step 5 (0 = excluded) |
| `input_boolean.garden_a_schedule_armed` | Toggle | Schedule A active/inactive |
| `input_datetime.garden_a_run_started` | Date+time | Stamped at run start; powers A's during-run countdown |

**Total new helpers: 25 × 2 = 50.**

---

## Shared helpers — unchanged, single instance

Do **not** namespace these. One rain skip / one winterise applies to every
schedule.

| Helper | Type | Why shared |
|--------|------|-----------|
| `input_boolean.garden_rain_cancel` | Toggle | Rain is a property of the weather, not a schedule |
| `input_number.garden_rain_threshold` | Number (0–100, step 5) | One forecast threshold |
| `input_datetime.garden_last_rain` | Date+time | One 12 h actual-rain lookback |
| `input_boolean.garden_winter_shutdown` | Toggle | ❄ suspends **everything at once** |

The weather entity (e.g. `weather.met_office_fleet`) is also shared and set in
`rain-automations.yaml`.

---

## Migration — singleton → schedule A (chosen approach)

The existing live `garden_*` singleton helpers become **schedule A** so A keeps
your current days / start time / valves; **B** is created fresh and left blank
(no days, no valves) until you configure it.

**Rename (preserve state):**

| From (singleton) | To (schedule A) |
|------------------|-----------------|
| `input_boolean.garden_water_{day}` × 7 | `input_boolean.garden_a_water_{day}` |
| `input_select.garden_water_start_time` | `input_select.garden_a_water_start_time` |
| `input_text.garden_valve_N_entity` / `_name` × 5 | `input_text.garden_a_valve_N_entity` / `_name` |
| `input_number.garden_valve_N_duration` × 5 | `input_number.garden_a_valve_N_duration` |
| `input_boolean.garden_schedule_armed` | `input_boolean.garden_a_schedule_armed` |
| `input_datetime.garden_run_started` | `input_datetime.garden_a_run_started` |

> ⚠️ Renaming a helper's **entity_id** does not change values, but it **does**
> break anything still pointing at the old id. After renaming, the old
> `script.garden_watering_sequence` and `automation.garden_watering_schedule`
> must be replaced by the A/B versions in this bundle (they reference the new
> ids). Plan to swap the script + automation in the same sitting as the rename.

**Create fresh for B:** all 25 `garden_b_*` helpers. **Populate B's valve entity + name slots**
(`garden_b_valve_N_entity` / `_name`) for every zone you want to appear — the same switches as A, or a
subset. Durations and day toggles can start at `0`/off so B waters nothing until configured on the card.

> ⚠️ A valve slot with a **blank entity renders no zone row**, so the card shows no valve to select and
> no duration dots. Always set the entity (and name) for each zone you want — even with duration 0 — or
> the second card looks empty. Set values via the helper UI or `input_text.set_value`.

**Retire (after A/B scripts + automations are in place):**
- `script.garden_watering_sequence`
- `automation.garden_watering_schedule`
- the three v0.4.0 rain automations (replaced by `rain-automations.yaml`)

---

## Objects to install alongside the helpers

| File | Creates |
|------|---------|
| `scripts.yaml` | `script.garden_a_watering_sequence`, `script.garden_b_watering_sequence` (with the single-valve cap guard) |
| `schedule-automations.yaml` | `automation.garden_a_watering_schedule`, `automation.garden_b_watering_schedule` (with the shared winter-off condition) |
| `rain-automations.yaml` | Replacements for the 3 shared rain automations (recorder unchanged; check/reset generalised to both schedules) |
| `card-a.yaml`, `card-b.yaml` | The two dashboard cards (namespaced clones of `card.yaml`) |
