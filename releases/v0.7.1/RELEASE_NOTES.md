# v0.7.1 — Winterise fix

## What changed

**Winterise now correctly disarms the schedule and clears arm intent.**

In v0.7.0, the Winterise button was decoupled from the armed state — it only toggled `winter_shutdown`. This was wrong: Winterise is a deliberate seasonal shutdown and should disable the schedule entirely.

**Corrected behaviour:**

| Button | v0.7.0 | v0.7.1 |
|--------|--------|--------|
| ❄ Winterise (activate) | Sets `winter_shutdown = on` only | Sets `winter_shutdown = on` + `armed = off` + `arm_intent = off` |
| ❄ De-winterise | Clears `winter_shutdown` | Clears `winter_shutdown`; schedule stays disarmed until you press Go |

Clearing `arm_intent` means the startup automation will not re-arm the schedule if HA reboots while the system is winterised.

## Files changed

- `multi-schedule/card-a.yaml`
- `multi-schedule/card-b.yaml`
