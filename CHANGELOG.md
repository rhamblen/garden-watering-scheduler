# Changelog

All notable changes to this project will be documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [Unreleased]

Nothing planned yet.

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
