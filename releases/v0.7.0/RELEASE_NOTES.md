# v0.7.0 — Arm-state persistence

## What's new

**Schedules now survive HA reboots.** Previously, pressing Go to arm a schedule would reset to disarmed after every Home Assistant restart, requiring a manual re-arm.

### The fix: intent-helper pattern

Two new helpers (`input_boolean.garden_a_schedule_arm_intent` / `garden_b_schedule_arm_intent`) record the user's *intent* to have a schedule armed. A `homeassistant.start` automation restores the live armed flag from the intent helper on every HA boot.

### Button behaviour changes

| Button | Before | After |
|--------|--------|-------|
| **Go** (body) | Arms schedule | Arms schedule; also sets intent = on |
| **Go** (body, when armed) | — | Now shows **◼ Disarm** — disarms schedule and sets intent = off |
| **■ Stop** (header) | Halts run + **disarms** | Halts run only; armed state and intent are preserved |
| **❄ Winterise** (header, on) | Activates winter mode + disarms | Activates winter mode + disarms + **clears arm intent** (so a reboot during winter won't re-arm) |

### Migration

If you have existing `garden_a_*` / `garden_b_*` helpers:

1. Create two new `input_boolean` helpers:
   - Name: **Garden A Schedule Arm Intent** → `input_boolean.garden_a_schedule_arm_intent`
   - Name: **Garden B Schedule Arm Intent** → `input_boolean.garden_b_schedule_arm_intent`
2. Set each to `on` if the corresponding schedule is currently armed.
3. Add the *Garden Watering Restore Armed State* automation from `multi-schedule/schedule-automations.yaml`.
4. Replace `card-a.yaml` and `card-b.yaml` on your dashboard with the updated versions.

## Files changed

- `multi-schedule/card-a.yaml` — Go/Disarm toggle; Stop no longer disarms; Winterise disarms + clears arm intent
- `multi-schedule/card-b.yaml` — same
- `multi-schedule/schedule-automations.yaml` — new startup restore automation appended
