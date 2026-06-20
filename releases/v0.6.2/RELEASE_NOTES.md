# v0.6.2 — Accurate countdown + install-guide tidy

A small patch on top of [v0.6.1](../v0.6.1/RELEASE_NOTES.md).

## Fixed

- **During-run "Time remaining" countdown now matches the real finish.** v0.6.1 added a
  10-second settle delay after each valve closes, but the card's countdown still summed only
  the watering durations — so it hit `0:00` about 10 s per zone too early. The multi-schedule
  cards (`card-a.yaml` / `card-b.yaml`) now add `settle_s = (vns.parts|length) * 10` seconds to
  the computed run-end, so the countdown lines up with the actual finish.

  > The single-schedule root `card.yaml` is unchanged — its script has no single-valve cap
  > guard and therefore no settle delay, so its countdown was already accurate.

## Changed

- **Install guide — Claude-assisted section.** Replaced the standalone "ask Claude to add
  rain cancel" follow-up (rain cancel is already covered by the main install prompt and by
  Step 5) with an **"ask Claude to add a second independent schedule"** prompt that links to
  the *Adding a second schedule* section.

## Upgrading from v0.6.1

Re-import `multi-schedule/card-a.yaml` / `card-b.yaml` (or just add `settle_s` to the
`rem_s` / `end_ts` lines in your two scheduler cards). No helper, script, or automation
changes are required.
