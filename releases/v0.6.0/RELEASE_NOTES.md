# v0.6.0 — Multiple independent schedules

Run more than one watering schedule, each with its own days, start time, and zones —
backed by namespaced helpers so the schedules are genuinely independent rather than two
views of the same one.

> ⚠️ **You cannot duplicate the card to get a second schedule.** Two copies of the same
> card share the same helpers and control one schedule. An independent schedule needs its
> own namespaced helpers, script, automation, and card — all provided in `multi-schedule/`.

## Added

- **`multi-schedule/` bundle** — a ready-made two-schedule (A & B) build:
  - `card-a.yaml` / `card-b.yaml` — namespaced clones of `card.yaml` (`garden_a_*` / `garden_b_*`)
  - `scripts.yaml` — `script.garden_a_watering_sequence` + `_b_`, each with the single-valve cap guard
  - `schedule-automations.yaml` — `automation.garden_a_watering_schedule` + `_b_`
  - `rain-automations.yaml` — the three shared rain automations, generalised to both schedules
  - `helpers.md` — the 50 per-schedule helpers, the shared list, and the singleton → A migration
  - `README.md` — bundle overview and install order
- **Design doc** — `docs/design-multiple-schedules.md` records the agreed design (Option 3 hybrid namespace + FIFO single-valve policy).

## Design

- **Per-schedule, namespaced (`garden_a_*` / `garden_b_*`):** day toggles, start time, valve slots 1–5, armed flag, run-started stamp, one script + one automation each.
- **Shared house-wide (single instance):** rain cancel / threshold / last-rain and the rain automations, winter shutdown, and the weather entity. Winterise once on any card and every schedule is suspended.
- **Overlap policy — FIFO single-valve cap:** at most one garden valve open system-wide (protects sprinkler pressure). Each sequence script guards every valve-open with "no garden valve currently on across both schedules (and manual/pool opens)". First command wins; a later overlapping zone is **dropped for that run**, never queued. Turn-off always wins; scripts never re-assert — enforced from live switch state, no lock entity.
- **Winterise condition:** both schedule automations carry an explicit `garden_winter_shutdown == off` condition, so the shared ❄ flag suspends every schedule regardless of each card's armed state.
- **Rain, generalised:** the recorder is unchanged; the auto-cancel check fires 30 min before *either* schedule's start; the daily reset clears 30 min after the *latest* start, so a rainy day skips both an early and a late run before clearing.
- **Card namespacing:** entity IDs find-replaced to `garden_a_*` / `garden_b_*`; the client-side countdown global `window.gwsT` is namespaced to `window.gwsT_a` / `_b` so two cards on one page don't collide; titles tagged `(A)` / `(B)`.

## Upgrading from one schedule

Going from a single schedule to two: migrate your existing `garden_*` helpers into the
`garden_a_*` namespace (renaming preserves their values), create the `garden_b_*` set, install
the A/B scripts + automations, swap in the generalised rain automations, and deploy the two
cards. Full step-by-step — both **Claude-assisted** and **manual** — in
[`INSTALLATION.md`](../../INSTALLATION.md) → *Adding a second schedule*.

> **Populate each schedule's valve slots.** A valve slot with a blank entity renders no zone
> row, so a fresh schedule's card looks empty — set `garden_b_valve_N_entity` (and `_name`) for
> every zone you want to appear, even if its duration starts at 0.

## Not included (deferred)

Master meter-valve linkage, the two-part winterise model, drain sequencing, and leak detection
remain deferred — see `docs/design-multiple-schedules.md`.
