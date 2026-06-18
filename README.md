# Garden Watering Scheduler — Home Assistant Dashboard Card

A Home Assistant Lovelace dashboard card that schedules sequential two-zone garden watering. Select which days of the week to water, set a start time, choose how many minutes each zone runs, and press Go. The upper lawn waters first; when its timer expires the lower lawn starts automatically.

> Companion project to [Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve) — uses the same valve entities and `custom:html-template-card` stack.

---

## Features

- **Day selector** — toggle any combination of Mon–Sun
- **Start time** — step through 30-minute slots from 00:00 to 23:30
- **Zone duration** — 5, 10, 15, or 20 minutes per zone (upper and lower independently)
- **Total run display** — live sum shown as you adjust durations
- **Go / Disarm** — arm or disarm the schedule in one tap
- **Winterise** — suspend all scheduling with a single button; re-enable in spring
- **Rain cancel** — 🌧 header button skips today's run manually; automatic rain detection added in v0.4.0
- **Next run display** — calculated live in Jinja2; shows the actual next scheduled date and time
- **Five card states** — Disarmed / Armed / Rain skip / Winterised / Running (v0.3.0)
- **HA theme aware** — uses CSS variables for colours

---

## Versions

| Version | Status | Features |
|---------|--------|---------|
| v0.1.0 | ✅ | Card UI, all helpers, day/time/duration controls, next run display |
| v0.2.0 | planned | Watering sequence script + scheduling automation |
| v0.3.0 | planned | Active countdown per zone, progress bar, Cancel this run |
| v0.4.0 | planned | Automatic rain cancel via daily weather forecast |
| v1.0.0 | planned | Full release — polish, complete docs |

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Home Assistant | Any recent version with Lovelace in storage (UI) mode |
| `custom:html-template-card` | Install via HACS → Frontend → search "HTML Template Card" ([GitHub](https://github.com/PiotrMachowski/lovelace-html-template-card)) |
| Zigbee water valves | `switch.tap_lhs_upper_lawn_blue` and `switch.tap_lhs_lower_lawn_green` paired via Zigbee2MQTT |

---

## Installation

See **[INSTALLATION.md](INSTALLATION.md)** for full step-by-step instructions including all 16 helper definitions.

**Quick path — Claude-assisted:**

> "Help me install the Garden Watering Scheduler from https://github.com/rhamblen/garden-watering-scheduler — add it to my [dashboard name] dashboard."

Claude will create all required helpers and deploy the card.

**Quick path — manual:**

1. Install `custom:html-template-card` via HACS
2. Create the 16 HA helpers (see [INSTALLATION.md](INSTALLATION.md))
3. Download [`releases/v0.1.0/card.yaml`](releases/v0.1.0/card.yaml)
4. Add as a Manual card on your dashboard

---

## Entities used

| Entity | Role |
|--------|------|
| `input_boolean.garden_water_mon` … `_sun` | Day schedule toggles (×7) |
| `input_select.garden_water_start_time` | Start time — 48 options, 00:00 to 23:30 |
| `input_number.garden_water_upper_duration` | Upper lawn duration in minutes (5–20) |
| `input_number.garden_water_lower_duration` | Lower lawn duration in minutes (5–20) |
| `input_boolean.garden_schedule_armed` | Schedule armed / disarmed |
| `input_boolean.garden_winter_shutdown` | Winter suspension toggle |
| `input_boolean.garden_rain_cancel` | Rain skip flag (manual or auto) |
| `input_number.garden_rain_threshold` | Precipitation % threshold for auto rain cancel (v0.4.0) |
| `timer.garden_upper_lawn` | Countdown timer for upper zone (v0.3.0) |
| `timer.garden_lower_lawn` | Countdown timer for lower zone (v0.3.0) |
| `switch.tap_lhs_upper_lawn_blue` | Upper lawn valve |
| `switch.tap_lhs_lower_lawn_green` | Lower lawn valve |

---

## Watering sequence

```
[Start time reached on a scheduled day]
        ↓
  Upper lawn opens
  Timer counts down (5–20 min)
        ↓
  Upper lawn closes
        ↓
  Lower lawn opens
  Timer counts down (5–20 min)
        ↓
  Lower lawn closes
```

The sequence is managed by a single HA script called by a time-pattern automation (added in v0.2.0).

---

## Related project

**[Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve)** — per-valve control card with flow rate display, consumption tracking, countdown timer, and pool fill mode. The scheduler uses the same valve switch entities.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

---

## Licence

MIT — free to use, adapt, and share. Attribution appreciated.
