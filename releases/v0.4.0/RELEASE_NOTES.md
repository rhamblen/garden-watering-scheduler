# v0.4.0 — Automatic rain cancel

Skips a scheduled watering run automatically when the weather doesn't warrant it — no card changes required. The detection sits entirely in three new automations that set the existing `input_boolean.garden_rain_cancel`, which the scheduler already honours. The card's 🌧 button and "Rain skip" badge work exactly as before; they just get set for you now.

## How it decides

30 minutes before each scheduled run that is actually due, the check cancels today's run if **either** is true:

- **Rained in the previous 12 h (actual)** — the weather entity's own condition (its `state`) entered a rain condition (`rainy`, `pouring`, `lightning-rainy`, `snowy-rainy`, `hail`) within the last 12 hours.
- **Rain forecast in the next 24 h (forecast)** — any hour in the next 24 h has a `precipitation_probability` at or above `input_number.garden_rain_threshold` (your % threshold).

When it cancels, it posts a Home Assistant **persistent notification** explaining the reason and reminding you that pressing **Go** overrides the skip. The flag clears automatically 30 minutes after the slot, so it never carries into the next day. A manual 🌧 skip auto-clears the same way.

## New objects

| Object | Type | Role |
|--------|------|------|
| `input_datetime.garden_last_rain` | Helper (date + time) | Records the last time it actually started raining; powers the 12 h lookback |
| `automation.garden_rain_recorder` | Automation | Stamps the helper when the weather condition turns to rain |
| `automation.garden_rain_auto_cancel_check` | Automation | T−30 min: evaluates actual + forecast, sets the flag, notifies |
| `automation.garden_rain_cancel_daily_reset` | Automation | T+30 min: clears the flag and dismisses the notification |

`T` is the time held in `input_select.garden_water_start_time`. Anchoring the checks to ±30 min of `T` means the feature works at any start time with no fixed-clock collisions and no race with the existing schedule automation.

## Weather provider

The weather entity is **not** hard-coded. Set your own `weather.*` entity in the two marked spots in [`automations.yaml`](automations.yaml) (recorder trigger + forecast service target). Any provider that exposes hourly forecasts works — Met Office, met.no, OpenWeatherMap, AccuWeather, etc. See **INSTALLATION.md → v0.4.0 rain cancel** for the manual walk-through and the Claude-assisted one-liner.

## Why a "last rain" helper instead of History Stats

Forecast/weather entities only hold current + future data, and templates can't look back in time. A small recorder automation stamping `input_datetime.garden_last_rain` gives the same "did it rain in the last 12 h" answer as a History Stats sensor, using only helpers and automations that install cleanly via the UI or Claude.

## Add it

- **Claude-assisted:** "Add automatic rain cancel (v0.4.0) to my Garden Watering Scheduler — discover my weather entity and wire up the rain automations."
- **Manual:** create the `input_datetime` helper, import the three automations from [`automations.yaml`](automations.yaml), and set your weather entity in the two marked places. Full steps in INSTALLATION.md.

No upgrade to the card or `script.garden_watering_sequence` is needed — `releases/v0.4.0/card.yaml` is identical to v0.3.0 and included only so the release folder is self-contained.
