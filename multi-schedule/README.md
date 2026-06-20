# Multi-schedule bundle (schedules A & B)

Independent watering schedules — each with its own days, start time, and valve
zones — built on the agreed **Option 3 hybrid namespace** with a **FIFO,
single-valve** overlap policy. Design rationale: [`../docs/design-multiple-schedules.md`](../docs/design-multiple-schedules.md).

> **Status:** `[Unreleased]`. This is the **first** multi-schedule build — two
> schedules (`a`, `b`). The deferred meter / winterise-drain / leak-detection
> work in the design doc is **not** included here.

## What's in this folder

| File | Purpose |
|------|---------|
| [`helpers.md`](helpers.md) | The 50 per-schedule helpers, the shared helpers, and the singleton → A migration |
| [`scripts.yaml`](scripts.yaml) | `script.garden_a_watering_sequence` + `_b_`, each with the single-valve cap guard |
| [`schedule-automations.yaml`](schedule-automations.yaml) | `automation.garden_a_watering_schedule` + `_b_` (adds shared winter-off condition) |
| [`rain-automations.yaml`](rain-automations.yaml) | Shared rain automations, generalised to both schedules (replaces the v0.4.0 root set) |
| [`card-a.yaml`](card-a.yaml) / [`card-b.yaml`](card-b.yaml) | The two dashboard cards (namespaced clones of `../card.yaml`) |

## Design in one screen

**Namespaced per schedule (`garden_a_*` / `garden_b_*`):** days, start time,
valve slots 1–5, armed flag, run-started stamp, one script, one schedule
automation.

**Shared house-wide (single instance):** rain cancel + threshold + last-rain and
the rain automations; winter shutdown; the weather entity.

**Overlap = FIFO, one valve open system-wide.** Every `turn_on` in a sequence
script is guarded by "no garden valve currently on across **both** schedules
(and manual/pool opens of those valves)". First command holds the slot; a later
overlapping zone is **dropped for that run** — not queued. Turn-off always wins;
scripts never re-assert. No lock entity — the guard reads live switch state.

**Winterise is house-wide.** ❄ on either card sets the shared
`garden_winter_shutdown`; both schedule automations now carry an explicit
`winter_shutdown == off` condition, so winterising once suspends **every**
schedule regardless of each card's armed state.

## Suggested install order (deployment pass — not done yet)

1. Create the 50 `garden_a_*` / `garden_b_*` helpers (see `helpers.md`); migrate
   the live singleton helpers into `garden_a_*`.
2. Install `scripts.yaml` (two scripts).
3. Install `schedule-automations.yaml` (two automations).
4. Replace the three live v0.4.0 rain automations with `rain-automations.yaml`
   (set your `weather.*` entity in the two marked spots).
5. Retire the old singleton `script.garden_watering_sequence` and
   `automation.garden_watering_schedule`.
6. Deploy `card-a.yaml` and `card-b.yaml` side by side on the dashboard.

## One adaptation worth a look (review note)

The design doc says the shared rain automations are reused "unchanged", but they
reference helpers that are now per-schedule. Two of the three therefore had to be
**generalised** (the recorder is genuinely unchanged):

- **Auto-cancel check** fires 30 min before **either** schedule's start (OR
  across A and B); still only ever turns the shared flag **on**.
- **Daily reset** clears the shared flag 30 min after the **latest** of the two
  start times, so a rainy day skips both an early and a late run before
  clearing.

If you'd prefer a different rule (e.g. a single fixed pre-dawn check, or
resetting per-schedule), that's the spot to change.
