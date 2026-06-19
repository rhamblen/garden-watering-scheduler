# Garden Watering Scheduler — Home Assistant Dashboard Card

A Home Assistant Lovelace dashboard card that schedules sequential garden watering across 1–5 configurable valve zones. Select which days of the week to water, set a start time, choose how many minutes each zone runs, and press Go to arm the schedule — or ▶ Start now to run immediately. Zones run in order; any zone with duration set to zero is skipped automatically.

> Companion project to [Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve) — uses the same valve entities and `custom:html-template-card` stack.

---

## Features

- **Day selector** — toggle any combination of Mon–Sun
- **Start time** — step through 30-minute slots from 00:00 to 23:30
- **Dynamic valve list** — 1–5 zones configured at install time via HA helpers; add another zone later by populating the next slot, no card changes needed
- **Zone duration** — 5, 10, 15, or 20 minutes per zone; red ✗ dot excludes a zone from the sequence without removing it from the card
- **Total run display** — live sum of non-zero durations; shows `15+10` / `1 zone` / `—` depending on configuration
- **Go / Disarm** — arm or disarm the schedule in one tap
- **Start now / Stop** — header ▶ runs a watering sequence immediately, outside the schedule; ■ stops an active run and closes every valve
- **Live run countdown** — while watering, a "Time remaining" timer counts down second-by-second (client-side, so it ticks smoothly rather than jumping each minute)
- **Winterise** — suspend all scheduling with a single button; re-enable in spring
- **Rain cancel** — 🌧 header button skips today's run manually; automatic rain detection cancels a run when it rained in the last 12 h or rain is forecast in the next 24 h, and posts a notification
- **Next run display** — calculated live in Jinja2; shows the actual next scheduled date and time, with a live countdown (`in 6d 22h 10m`)
- **Five card states** — Disarmed / Armed / Rain skip / Winterised / Running
- **HA theme aware** — uses CSS variables for colours

---

## Versions

| Version | Status | Features |
|---------|--------|---------|
| v0.1.0 | ✅ | Card UI, all helpers, day/time/duration controls, next run display |
| v0.2.0 | ✅ | Watering sequence script + scheduling automation |
| v0.2.1 | ✅ | Bug fixes: buttons, blank card, layout — Winterise/Disarm in header |
| v0.3.0 | ✅ | Dynamic valve list (1–5 zones), next-run countdown, zone-exclude dot, Test button |
| v0.4.0 | ✅ | Automatic rain cancel — 12 h actual + 24 h forecast check, with notification |
| v0.5.0 | ✅ | Start-now / Stop header controls, live ticking time-remaining countdown (Test button removed) |
| v1.0.0 | planned | Full release — polish, complete docs |

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Home Assistant | Any recent version with Lovelace in storage (UI) mode |
| `custom:html-template-card` | Install via HACS → Frontend → search "HTML Template Card" ([GitHub](https://github.com/PiotrMachowski/lovelace-html-template-card)) |
| Zigbee water valves | 1–5 valve `switch.*` entities paired via Zigbee2MQTT (any switch entities work; the entity IDs are stored in helpers, not hard-coded) |

---

## Installation

See **[INSTALLATION.md](INSTALLATION.md)** for full step-by-step instructions including all helper definitions.

**Quick path — Claude-assisted (recommended):**

> "Help me install the Garden Watering Scheduler from https://github.com/rhamblen/garden-watering-scheduler — discover my Zigbee valve switches, set them up as zones, and add the card to my [dashboard name] dashboard."

Claude discovers your valve switch entities, creates all helpers (pre-filling the valve slots), deploys the watering script + automation, and adds the card.

**Quick path — manual:**

1. Install `custom:html-template-card` via HACS
2. Create the HA helpers (see [INSTALLATION.md](INSTALLATION.md)) — fill in 1–5 valve slots with your switch entity IDs
3. Download [`card.yaml`](card.yaml)
4. Add as a Manual card on your dashboard
5. (Optional) Add automatic rain cancel — see [INSTALLATION.md](INSTALLATION.md) → *Automatic rain cancel*

---

## Entities used

| Entity | Role |
|--------|------|
| `input_boolean.garden_water_mon` … `_sun` | Day schedule toggles (×7) |
| `input_select.garden_water_start_time` | Start time — 48 options, 00:00 to 23:30 |
| `input_text.garden_valve_1_entity` … `_5_entity` | Valve switch entity ID for each slot (×5) — empty slots are ignored |
| `input_text.garden_valve_1_name` … `_5_name` | Display name shown on the zone row (×5) |
| `input_number.garden_valve_1_duration` … `_5_duration` | Per-zone duration in minutes, 0–60 step 5 (×5); 0 = excluded |
| `input_boolean.garden_schedule_armed` | Schedule armed / disarmed |
| `input_boolean.garden_winter_shutdown` | Winter suspension toggle |
| `input_boolean.garden_rain_cancel` | Rain skip flag (manual or auto) |
| `input_number.garden_rain_threshold` | Forecast precipitation % threshold for auto rain cancel |
| `input_datetime.garden_last_rain` | Timestamp of last actual rain — powers the 12 h lookback |
| `input_datetime.garden_run_started` | Stamped when a run begins — drives the live "Time remaining" countdown |
| `weather.*` (your provider) | Forecast + current condition source for rain detection; set in the automations, not hard-coded |

---

## Watering sequence

```
[Start time reached on a scheduled day]
        ↓
  For each configured slot (1 → 5):
        ↓
    Duration > 0 ?  ── no ──▶ skip slot
        │ yes
    Valve opens
    Wait <slot duration> minutes
    Valve closes
        ↓
  (next slot)
```

The sequence is managed by a single HA script (`script.garden_watering_sequence`) called by a time-pattern automation. The script loops over all five slots; each slot reads its valve entity from `input_text.garden_valve_N_entity` and its run time from `input_number.garden_valve_N_duration`, skipping any slot whose duration is 0. It stamps `input_datetime.garden_run_started` as its first step, which drives the live countdown.

---

## Running outside the schedule

The header has two manual controls so anyone can run — or stop — a sequence without waiting for the scheduled time:

- **▶ Start now** — runs the full sequence immediately using the current zone durations. Disabled while watering or while winterised.
- **■ Stop** — halts an active run, closes every configured valve, and disarms the schedule.

While a run is in progress (scheduled or manual), the status area shows a **Time remaining** countdown that ticks down second-by-second in the browser.

---

## Automatic rain cancel

Three automations decide, 30 minutes before each scheduled run, whether to skip it:

```
[T − 30 min, on a day a run is due]
        ↓
  Rained in the last 12 h?  ──┐   (actual: weather CONDITION was rainy/pouring/…,
        │ no                  │    recorded into input_datetime.garden_last_rain)
        ↓                     │
  Any hour in next 24 h ≥ ────┤   (forecast: precipitation_probability from
  garden_rain_threshold %?    │    weather.get_forecasts, type: hourly)
        │ no                  │ yes (either)
        ↓                     ↓
   run proceeds        set garden_rain_cancel = on  +  notify
                              ↓
                  existing schedule automation skips the run
                              ↓
              [T + 30 min] flag cleared, notification dismissed
```

- **Actual rain** comes from the weather entity's `state` (its current condition). **Forecast rain** comes from the hourly `precipitation_probability` returned by `weather.get_forecasts`.
- The **weather provider is not hard-coded** — you point it at your own `weather.*` entity. See [INSTALLATION.md](INSTALLATION.md) → *Automatic rain cancel* for exactly which two places to set it.
- Pressing **Go** (or ▶ Start now) overrides any skip and waters anyway.

---

## Related project

**[Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve)** — per-valve control card with flow rate display, consumption tracking, countdown timer, and pool fill mode. The scheduler uses the same valve switch entities.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

---

## Licence

MIT — free to use, adapt, and share. Attribution appreciated.
