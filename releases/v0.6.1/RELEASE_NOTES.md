# v0.6.1 — Bug fix: settle delay between valves

A patch on top of [v0.6.0](../v0.6.0/RELEASE_NOTES.md) (multiple independent schedules).

## Fixed

- **Second (and subsequent) zone skipped mid-run.** Each sequence script
  (`script.garden_a_watering_sequence` / `_b_`) now waits **10 seconds after every
  `turn_off`** before moving on to the next zone.

  **Why it happened:** every zone is guarded by the single-valve cap ("don't open if any
  garden valve is already on"). A Zigbee valve doesn't report `off` back to Home Assistant
  the instant it's switched off, so the *next* zone's guard could still see the just-closed
  valve as `on` and skip that zone entirely — the symptom was "the second valve never
  opened". The settle delay lets the valve's state propagate so the following zone opens
  normally.

## Upgrading from v0.6.0

Re-import the two sequence scripts from
[`multi-schedule/scripts.yaml`](multi-schedule/scripts.yaml) (each zone now ends with a
`delay: { seconds: 10 }` after its `turn_off`), or add that delay step yourself after each
`homeassistant.turn_off`. No helper, automation, or card changes are required.

> If you ever add a schedule C/D, the same settle delay belongs in every sequence script.

## Note

The card's during-run "Time remaining" countdown sums only the watering durations, so it now
reaches `0:00` roughly 10 s × (number of zones) before the script actually finishes. This is
cosmetic only — the watering itself is correct.
