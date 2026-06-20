# Design — Multiple Independent Schedules

**Status:** Design agreed, not yet built. Captures the decisions for the
`[Unreleased]` "multiple independent schedules" work so the build can proceed
without re-litigating them.

---

## Goal

Allow several scheduler cards to run **independent schedules** — each with its
own days, start time, and combination of valve zones — rather than every copy of
the card being a mirror of one shared schedule.

Today the card is a stateless view over **singleton** `garden_*` helpers, one
`script.garden_watering_sequence`, and one `automation.garden_watering_schedule`.
Copying the card just shows the same schedule twice (both read/write the same
helpers and trigger the same script). Independence requires the per-schedule
state to be **namespaced**.

---

## Chosen approach — Option 3 (hybrid namespace)

Namespace only the **per-schedule** state; keep **house-wide** concepts shared.

### Per-schedule (namespaced, e.g. `garden_a_*`, `garden_b_*`)
- Day-of-week toggles (`*_water_mon … _sun`)
- Start time (`*_water_start_time`)
- Valve slots 1–5 (`*_valve_N_entity` / `_name` / `_duration`)
- Armed (`*_schedule_armed`)
- Run-started stamp (`*_run_started`) — powers that card's during-run countdown
- One sequence script + one schedule automation per schedule

### Shared (house-wide, single instance)
- Rain cancel + threshold + last-rain (`garden_rain_cancel`,
  `garden_rain_threshold`, `garden_last_rain`) and the **3 rain automations** —
  rain is a property of the weather, not of a schedule, so it is **not**
  duplicated.
- Winter shutdown (`garden_winter_shutdown`) — ❄ suspends **everything at once**.
  Because this is a single shared helper, the ❄ button on **any one** card sets it
  and **all** cards immediately reflect "Winterised". You winterise once; it holds
  true across every schedule. (Un-winterising is likewise a single action on any
  card.) This is inherent to the shared-helper design — no extra wiring needed.
- Weather entity.

This is the "second helper namespace" idea from the changelog, minus the wasteful
duplication of the rain logic. If more than ~3 schedules are ever wanted, graduate
to a fully parameterised "schedule slot" model (the way valves became slots in
v0.3.0); for 2–3 named schedules, the namespaced clone is simpler.

---

## Overlap policy — FIFO, single-valve cap

Independent schedules can overlap in time and can share physical valves. We
deliberately adopt the **simplest** rule set — no contention manager, no
priorities, no queuing.

1. **One valve open system-wide, ever.** Pressure drop with two valves open stops
   the lawn sprinklers popping up, so at most one valve may be on at any instant
   across **all** schedules (and manual/pool use).

2. **First command wins.** Whatever turns a valve on first holds the single slot
   until it finishes. A later turn-on that lands while any valve is open is
   **dropped** — not queued, not deferred. The later schedule's overlapping zone
   is simply **skipped** for that run.

3. **Off always wins, no re-asserting.** A turn-off is always honored. No schedule
   re-opens a valve that another schedule (or a manual close) just shut. This is
   already true of the current design — each zone is fired exactly once
   (open → delay → close) with no re-assert loop — and must stay that way.

### Consequence (accepted)
If two schedules overlap on time, the one that fires second loses its overlapping
zones for that run; they are not watered later automatically. This is an accepted
trade-off in exchange for zero contention logic.

---

## Enforcement (build note)

The single-valve cap is enforced at the **moment of turn-on**, not by closing
valves after the fact (closing would be "fighting", which we explicitly avoid):

- Guard each `homeassistant.turn_on` step in every sequence script with a
  condition that **no garden valve is currently `on`** (check the actual switch
  states of all configured `*_valve_N_entity`, so manual/pool opens also hold the
  slot). If any is on, skip the turn-on for that zone — do **not** wait or retry.
- Turn-off steps need no guard.
- Because the cap reads live switch state (not a flag owned by one schedule), it
  naturally produces first-come-first-served behaviour without any lock entity.

---

## Phasing — what's baseline vs. deferred

**Baseline (current, valid):** the **shared-flag house-wide winterise** above is the
accepted behaviour now — ❄ on any scheduler card sets `garden_winter_shutdown` and
every scheduler card honours it. Nothing more is required for the multi-schedule
build to ship.

**Deferred to the "specific new design" phase:** everything below — the master
meter-valve linkage, the winterise drain option + close-zones-before-supply
sequencing, leak detection, and moving coordination to the watermeter — layers on
**later**, on top of the shared flag. It is recorded here so the design isn't lost,
not as part of the first build.

---

## Optional (deferred) — master meter-valve linkage (nice-to-have)

The garden supply runs through a **water meter with its own valve** (the
Zemismart meter-valve combo, companion project, e.g. `switch.garden_water_meter`
+ `sensor.garden_water_meter_flow`). Because that meter sees **all** garden water,
it enables two optional safety features. Both are **off by default** and gated by
an enable toggle; the master-valve entity is **parameterised** (discovered at
install, never hard-coded — same pattern as the weather entity), held in e.g.
`input_text.garden_master_valve_entity` (blank = feature off).

### The two-part winterise model
Winterise has **two distinct layers** that can be set independently but are
designed to cascade from the meter:

**Part 1 — Schedule winterise (scheduler side).** The shared
`garden_winter_shutdown` flag — turns the schedules off for winter. Typically set
in **autumn**, and can be done on its own, independently of the meter.

**Part 2 — Meter winterise (meter side, the orchestrator).** A deliberate
**"winterised" mode on the meter — which is NOT the same as the master valve simply
being `off`.** A valve that is merely closed for some operational reason is *off*;
*winterised* is an explicit mode that drives the whole sequence. When the meter is
put into winterise mode it:
- **winterises the valves correctly** — runs the physical drain sequence (close
  master supply, optionally open zone valves to drain; see sequencing below), and
- **confirms the schedules are winterised** — ensures `garden_winter_shutdown` is
  on (belt-and-braces: even if Part 1 wasn't done in autumn, meter winterise sets
  it).

**Un-winterising from the meter is asymmetric:** turning winterise **off** at the
meter **sorts the valves out correctly** (the spring exit sequence: close zones →
open master), **but does NOT turn the schedules back on.** Re-arming the schedule is
a **separate, deliberate act** on a scheduler card. (Consistent with the
"no surprise re-arming / off always wins" rule.)

> This supersedes the earlier looser "master valve *closed* → auto-winterise"
> phrasing: the trigger is the explicit **winterise mode**, not the valve's on/off
> state.

The master valve is the **upstream supply**, not a zone, so it is **excluded from
the single-valve pressure cap** — it is expected to be open whenever any zone runs.

### Winterise drain option + valve sequencing (safety-critical)
Winterise may optionally **open the zone valves and leave them open** through
winter (master supply closed) to drain/depressurise the lines and reduce burst
risk. This is an option on winterise, not the default.

- **Entry order (winterise):** close the master supply **first**, *then* open the
  zone valves to drain residual water. Opening several zones at once here is fine
  and is **exempt from the single-valve pressure cap** — the supply is closed, so
  there is no pressure and no sprinkler-popup concern; it's a gravity drain.
- **Exit order (spring) — the constraint:** **all zone valves must be closed
  before the master supply valve opens.** Opening the supply while zones are open
  would flood every open zone simultaneously. So the spring "exit winter" action is
  always **close all zones → then open master** (and only then, separately and
  deliberately, un-winterise/re-arm the schedules — re-open still does not
  auto-re-arm, per above).
- **Enforcement:** if the master meter-valve is HA-controllable (the Zemismart
  switch), provide this as a single ordered "exit winter" sequence so the order
  can't be got wrong. As a backstop for a **manual/physical** supply open, a safety
  automation should watch the master valve: if it becomes open while any zone valve
  is open, immediately close the zone valves.

### Leak detection (related, separable)
Because the meter is garden-dedicated, flow at the meter while **no** zone valve is
open means a leak. A separate optional automation: if `meter flow > 0` AND all
configured zone valves are `off` for a short debounce, raise a persistent
notification. This is **independent** of the schedule feature and could live in the
meter project instead; recorded here so the idea isn't lost.

---

## Future review — winterise ownership in a mixed environment

**Revisit when the meter + zone valves + scheduler are all deployed together.**
The winterise process (close master supply, optional zone-drain, the spring
exit sequence, auto-winterise on master close, leak detection) spans all three
systems. The likely direction is to **coordinate winterise from the watermeter
project, not from the schedules** — the master valve and the meter live there, so
the sequencing/safety logic belongs next to the hardware it drives, with the
scheduler just honouring the shared `garden_winter_shutdown` flag it already reads.

This doc currently describes those features from the scheduler's side because that
is where the design conversation started; on review, expect the
auto-winterise / drain-sequencing / leak-detection automations to **move to (or be
owned by) the meter project**, leaving the scheduler as a consumer of the
house-wide winterise state. Decide the split at that point.

---

## Open items (not blockers)

- **Blocked-zone behaviour within a run:** current decision is *skip and move on*.
  Whether the script should advance immediately or still serve its delay is a
  build detail; skipping the delay too is cleaner.
- **How many schedules:** start with two (`a` / `b`); revisit the slot-model
  refactor only if more are wanted.
