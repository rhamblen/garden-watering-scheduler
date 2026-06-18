# v0.1.0 — Release Notes

**Released:** 2026-06-18

## What's new

First release of the Garden Watering Scheduler card.

- **Day selector** — seven toggle buttons (Mon–Sun); tap to include or exclude any day
- **Start time picker** — ‹ › step buttons cycle through all 30-minute slots from 00:00 to 23:30
- **Zone duration dots** — four dots per zone representing 5, 10, 15, and 20 minutes; tap to select
- **Total run display** — live sum of both zone durations shown alongside the start time
- **Go / Winterise / Disarm** action buttons with correct state transitions between all five modes
- **Rain cancel** — 🌧 header button manually toggles today's run skip; tap Go to override
- **Next run calculation** — Jinja2 walks forward from today to find the next enabled weekday, accounting for whether today's start time has already passed
- **Five display states** — Disarmed / Armed / Running (placeholder) / Rain skip / Winterised, each with appropriate badge, button states, and status text
- **Winter shutdown** — Winterise button suspends all scheduling and dims the day controls; Disarm exits both winter and armed states cleanly

## HA helpers required

See [INSTALLATION.md](../../INSTALLATION.md) for the full step-by-step guide.

16 helpers in total:
- 7 × `input_boolean` (day toggles)
- 1 × `input_select` (start time — 48 options)
- 2 × `input_number` (zone durations)
- 3 × `input_boolean` (armed, winter shutdown, rain cancel)
- 1 × `input_number` (rain threshold %)
- 2 × `timer` (upper and lower lawn — used from v0.3.0)

## What's coming next

- **v0.2.0** — watering sequence script + scheduling automation + next run display driven by real automation state
- **v0.3.0** — active countdown per zone, progress bar, Cancel this run button
- **v0.4.0** — automatic rain cancel via daily weather forecast check
