# Garden Watering Scheduler — Home Assistant Dashboard Card

A Home Assistant Lovelace dashboard card that schedules sequential garden watering across 1–5 configurable valve zones. Select which days of the week to water, set a start time, choose how many minutes each zone runs, and press Go. Zones run in order; any zone with duration set to zero is skipped automatically.

> Companion project to [Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve) — uses the same valve entities and `custom:html-template-card` stack.

---

## Features

- **Day selector** — toggle any combination of Mon–Sun
- **Start time** — step through 30-minute slots from 00:00 to 23:30
- **Dynamic valve list** — 1–5 zones configured at install time via HA helpers; add another zone later by populating the next slot, no card changes needed
- **Zone duration** — 5, 10, 15, or 20 minutes per zone; red ✗ dot excludes a zone from the sequence without removing it from the card
- **Total run display** — live sum of non-zero durations; shows `15+10` / `1 zone` / `—` depending on configuration
- **Go / Disarm** — arm or disarm the schedule in one tap
- **Winterise** — suspend all scheduling with a single button; re-enable in spring
- **Rain cancel** — 🌧 header button skips today's run manually; automatic rain detection added in v0.4.0
- **Next run display** — calculated live in Jinja2; shows the actual next scheduled date and time, with a live countdown (`in 6d 22h 10m`)
- **Test button** — runs the sequence on demand with a 60-second grace countdown before it starts
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
| v0.4.0 | planned | Automatic rain cancel via daily weather forecast |
| v0.5.0 | planned | Active countdown per zone, progress bar, Cancel this run |
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
3. Download [`releases/v0.3.0/card.yaml`](releases/v0.3.0/card.yaml)
4. Add as a Manual card on your dashboard

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
| `input_number.garden_rain_threshold` | Precipitation % threshold for auto rain cancel (v0.4.0) |

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

The sequence is managed by a single HA script (`script.garden_watering_sequence`) called by a time-pattern automation (added in v0.2.0). The script loops over all five slots; each slot reads its valve entity from `input_text.garden_valve_N_entity` and its run time from `input_number.garden_valve_N_duration`, skipping any slot whose duration is 0.

---

## Related project

**[Zigbee Smart Water Valve Card](https://github.com/rhamblen/Zigbee-smart-water-valve)** — per-valve control card with flow rate display, consumption tracking, countdown timer, and pool fill mode. The scheduler uses the same valve switch entities.

---

## Changelog

See [CHANGELOG.md](CHANGELOG.md)

---

## Licence

MIT — free to use, adapt, and share. Attribution appreciated.
