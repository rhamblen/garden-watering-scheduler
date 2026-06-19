# v0.5.0 — Start now / Stop header controls + live ticking countdown

## New features

- **▶ Start now (header)** — runs the watering sequence immediately, independent of the schedule, so anyone can trigger a run outside the scheduled start time. Disabled while a run is already in progress or while winterised.
- **■ Stop (header)** — properly halts an active run: stops `script.garden_watering_sequence`, closes **every configured valve**, then disarms the schedule.
- **Live "Time remaining" countdown** — while watering, the status area shows a countdown that ticks **second-by-second in the browser**, instead of jumping on Home Assistant's ~per-minute template refresh. The script stamps `input_datetime.garden_run_started` at the start of a run; the card computes the run-end time and a small client-side interval counts down to it (self-syncing and self-clearing).

## New helper

- `input_datetime.garden_run_started` (Date+time) — records run start; powers the countdown. Create it before running a sequence (see INSTALLATION.md → step 1p). The watering script stamps it automatically as its first step.

## Header layout

The header control set is now: **▶ Start now · ■ Stop · ❄ Winterise · 🌧 Rain · status badge**.

## Removed

- The **Test button** (and its `script.garden_test_watering` reference) — superseded by ▶ Start now.
- The interim in-card progress bar — superseded by the ticking Time-remaining countdown.

## Packaging

- `card.yaml` and `automations.yaml` now live at the **repository root** as the canonical current files, so install instructions no longer reference a version-specific path. `hacs.json` points at the root `card.yaml`. Versioned snapshots remain under `releases/` for history.

## Upgrade from v0.4.0

1. Create the `input_datetime.garden_run_started` helper.
2. Re-deploy the card from `card.yaml` (repo root), or let Claude update it in place.
3. The watering script gains a first step that stamps `garden_run_started` — re-import it, or ask Claude to update it.

No change to the rain-cancel automations or any other helper.
