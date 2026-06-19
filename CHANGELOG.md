# Changelog

All notable changes to this project will be documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

- v0.5.0 — active per-zone countdown, progress bar, Cancel this run
- Dual-card support — second helper namespace for independent schedules
- Design review — pool fill target max vs default; valve-card timer handoff vs self-managed delay

---

## [0.4.0] — 2026-06-19

### Added
- **Automatic rain cancel** — three new automations skip a scheduled run when the weather doesn't warrant it, without any card change (they set the existing `input_boolean.garden_rain_cancel`, which the scheduler already honours):
  - `automation.garden_rain_recorder` — state trigger on the weather entity's rain conditions (`rainy`, `pouring`, `lightning-rainy`, `snowy-rainy`, `hail`); stamps `input_datetime.garden_last_rain`.
  - `automation.garden_rain_auto_cancel_check` — fires 30 min before the start time (only when a run is actually due). Calls `weather.get_forecasts` (`type: hourly`) and cancels if it **rained in the last 12 h** (now − `garden_last_rain` < 12 h) **OR** any hour in the **next 24 h** has `precipitation_probability ≥ input_number.garden_rain_threshold`. On cancel it sets `garden_rain_cancel` and posts a persistent notification with the reason.
  - `automation.garden_rain_cancel_daily_reset` — fires 30 min after the start time; clears `garden_rain_cancel` and dismisses the notification so a skip (auto or manual) doesn't carry to the next day.
- **`input_datetime.garden_last_rain`** helper — records the last actual-rain onset, powering the 12 h lookback.
- Detection uses **two signals**: *actual* rain from the weather entity's `state` (condition), and *forecast* rain from the hourly `precipitation_probability`.

### Notes
- **Weather entity is parameterised, not hard-coded** — set your own `weather.*` entity in the two `# << SET WEATHER ENTITY` spots in `releases/v0.4.0/automations.yaml` (recorder trigger + forecast service target). Any provider with hourly forecasts works. The probability-scan template is provider-agnostic (`fc.values() | first`) so it needs no editing.
- **History Stats avoided on purpose** — that helper type isn't creatable through the HA config-flow API and templates can't look back in time, so an `input_datetime` + recorder automation provides the same 12 h answer using only UI/Claude-installable objects.
- The card and `script.garden_watering_sequence` are unchanged; `releases/v0.4.0/card.yaml` is identical to v0.3.0 and shipped only to keep the release folder self-contained.

### Anchoring / design decisions
- All three automations are anchored to ±30 min of `input_select.garden_water_start_time`, so the feature works at any start time, avoids fixed-clock collisions, and never races the existing schedule automation (which still evaluates `rain_cancel` at the start time).
- The auto-check only ever turns the flag **on** (so a manual 🌧 skip is never overridden by a dry forecast); the separate daily reset turns it off.
- Template conditions for the T±30 / weekday checks are unavoidable — they compare a derived time against the `input_select`/`input_boolean` helper values, which native `condition: time` can't reference — matching the existing schedule automation's pattern.

---

## [0.3.0] — 2026-06-19

### Added
- **Dynamic valve list (1–5 zones)** — valves are no longer hard-coded. Each zone lives in a numbered slot backed by three helpers: `input_text.garden_valve_N_entity` (the switch entity), `input_text.garden_valve_N_name` (display label), and `input_number.garden_valve_N_duration` (0–60 min, step 5). The card iterates slots 1–5 and renders a zone row only for slots whose entity is filled, so you can start with one valve and grow to five with no card or script changes.
- **Next-run countdown** — below the next-run date/time, a live countdown shows `in 6d 22h 10m` (days component appears only when ≥ 24h away). Computed in Jinja2 from `ns.off` (days ahead) and the start time, using `//` and subtraction — no `%` modulo and no `strftime`, consistent with the card's defensive templating.
- **Zone-exclude dot** — a red ✗ dot on each zone row sets that zone's duration to 0, excluding it from the run while keeping the row visible. Total run shows `15+10` for multiple zones, `1 zone` for a single zone, or `—` when none are active.

### Changed
- **`script.garden_watering_sequence` rewritten** — replaced the two hard-coded upper/lower blocks with five slot-based `if/then` steps. Each step turns on the templated valve entity from `input_text.garden_valve_N_entity` via `homeassistant.turn_on`, waits the slot's duration, then turns it off. Slots with duration 0 are skipped by a `numeric_state … above: 0` guard.
- **Zone duration label width** `28px → 40px` (`.gws-dl`) — double-digit values like "20 min" no longer wrap.

### Removed
- **`input_number.garden_water_upper_duration` / `_lower_duration`** — superseded by the per-slot duration helpers. (The named per-zone `timer.*` helpers previously earmarked for v0.3.0 are deferred to v0.5.0 and are not required.)

### Migration from v0.2.x
- Create the 15 valve-slot helpers (entity/name/duration × 5). Put your old upper-lawn values in slot 1 and lower-lawn in slot 2; leave slots 3–5 blank to keep the same two-zone behaviour.
- Re-deploy the card from `releases/v0.3.0/card.yaml` (or let Claude update it in place).

---

## [0.2.1] — 2026-06-19

### Fixed
- **Buttons not responding** — `hass` is not in global scope inside `html-template-card`'s shadow DOM; all service calls now use `document.querySelector('home-assistant').hass.callService(...)` (pattern from companion Zigbee Smart Water Valve card)
- **Blank card (Jinja2 `unexpected '%'`)** — `(function(){{% if` in Go button onclick created the sequence `{{%`, which Jinja2's WebSocket subscription lexer treated as an unclosed variable expression; replaced with a JS-level guard `{{ 'true' if winter else 'false' }}` to eliminate `{% %}` block tags inside `onclick` IIFEs entirely
- **CSS `border-radius:50%`** → `50px`; `strftime('%H:%M')` → manual `now().hour`/`now().minute` construction; `%7` modulo → conditional arithmetic — defensive `%` removal

### Changed
- **Winterise ❄ and Disarm ■ moved to card header** as compact icon buttons alongside the rain toggle; infrequent seasonal controls belong at the top, not alongside the daily Go button
- **Go button is now full-width** in its own row below the zone duration controls

---

## [0.2.0] — 2026-06-18

### Added
- **`script.garden_watering_sequence`** — turns on upper lawn valve, waits `input_number.garden_water_upper_duration` minutes, turns it off, then turns on lower lawn valve, waits `input_number.garden_water_lower_duration` minutes, turns it off. Mode `single` prevents double-run.
- **`automation.garden_watering_schedule`** — `time_pattern` trigger at every :00 and :30; four conditions: schedule armed, rain cancel off, current HH:MM matches `input_select.garden_water_start_time`, current weekday's `input_boolean.garden_water_{day}` is on. Calls `script.garden_watering_sequence` when all pass.

### Design decisions
- `time_pattern` at minutes 0 and 30 covers all 48 start-time slots without polling every minute
- Template conditions used for time and day checks because no native HA condition handles dynamic values from `input_select` or a runtime-indexed `input_boolean` array
- Automation mode `single` matches script mode — a second trigger while watering is in progress is safely ignored
- `rain_cancel` condition already wired in anticipation of v0.4.0; the manual 🌧 toggle on the card already exercises this path

---

## [0.1.0] — 2026-06-18

### Added
- **Day selector** — seven toggle buttons (Mon–Sun); reads from `input_boolean.garden_water_{day}` helpers and calls `input_boolean.turn_on/off` on tap
- **Start time stepper** — ‹ › buttons call `input_select.select_previous/next` on `input_select.garden_water_start_time`; 48 options covering 00:00–23:30 in 30-minute steps
- **Zone duration dots** — four dots (5/10/15/20 min) per zone; tap calls `input_number.set_value`; selected dot highlighted in green
- **Total run display** — Jinja2 sum of both durations shown in the mid row alongside start time
- **Go button** — arms the schedule (`input_boolean.garden_schedule_armed = on`) and clears any active rain cancel flag
- **Winterise button** — toggles `input_boolean.garden_winter_shutdown`; when enabling also disarms; day buttons dim and Go is disabled while winterised
- **Disarm button** — turns off both `garden_schedule_armed` and `garden_winter_shutdown` to return cleanly to disarmed state
- **Rain cancel toggle** — 🌧 header button toggles `input_boolean.garden_rain_cancel` manually; Go overrides it
- **Next run calculation** — Jinja2 walks 0–7 days ahead from today's weekday to find the next enabled day, comparing current HH:MM against start time to handle same-day logic
- **Rain skip display** — when rain cancel is active and schedule is armed, next run shows struck-through with the following scheduled date beneath
- **Five badge states** — Disarmed / Armed / Rain skip / Winterised, each with distinct colour treatment
- **Winter status message** — replaces the next run area with a suspension notice while winterised
- **CSS prefix `.gws-`** — all classes scoped to avoid collision with HA built-in and other card styles

### HA helpers (create before deploying)
- `input_boolean.garden_water_mon` through `_sun` × 7
- `input_select.garden_water_start_time` (48 options)
- `input_number.garden_water_upper_duration` (5–20, step 5)
- `input_number.garden_water_lower_duration` (5–20, step 5)
- `input_boolean.garden_schedule_armed`
- `input_boolean.garden_winter_shutdown`
- `input_boolean.garden_rain_cancel`
- `input_number.garden_rain_threshold` (0–100, step 5, default 60)
- `timer.garden_upper_lawn` (used from v0.3.0 — create now)
- `timer.garden_lower_lawn` (used from v0.3.0 — create now)

### Design decisions
- `input_select` chosen for start time over `input_datetime` so the ‹ › step UI wraps correctly around 00:00 / 23:30 without custom JS
- `timedelta(days=n)` in Jinja2 used for next-run date arithmetic — avoids JS date manipulation and stays server-side
- Separate `namespace` objects (`ns`, `ns2`) used for next-run and rain-skip offsets because Jinja2 loop variables require `namespace` for mutation
- Go button always calls both `turn_on(armed)` and `turn_off(rain_cancel)` — idempotent when already armed and no rain, so double-tapping has no side effect
- Winterise disarms on activation so automations (added in v0.2.0) cannot fire while winterised even if a stale armed state exists
- HTML entities (`&#9654;` ▶, `&#10052;` ❄, `&#9632;` ■) used for symbols to avoid encoding issues in YAML multiline scalars
