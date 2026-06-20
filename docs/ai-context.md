# AI Context — Garden Watering Scheduler Card

This file helps Claude (or any AI assistant) understand the repository structure so it can assist with modifications, updates, and publishing new versions.

---

## Repository purpose

This repository contains a Home Assistant Lovelace dashboard card that schedules sequential garden watering across **1–5 configurable valve zones**. A single `custom:html-template-card` provides the full UI — day selection, time picking, per-zone duration selection, armed/disarmed/winterised state management, next-run display with live countdown, header **▶ Start now / ■ Stop** controls, and a during-run **Time remaining** countdown that ticks client-side. The actual watering sequence is controlled by a separate HA script and automation. Valves are not hard-coded — each zone's switch entity, name, and duration live in slot helpers (`garden_valve_N_*`, N = 1–5), so the card adapts to however many valves are configured.

## Companion project

**Zigbee Smart Water Valve Card** — https://github.com/rhamblen/Zigbee-smart-water-valve
Same tech stack. Defines the valve entities this scheduler uses. Reference for CSS style consistency.

---

## File structure

```
/
├── README.md                        — Project description, features, entity table, sequence diagram
├── CHANGELOG.md                     — Version history (Keep a Changelog format)
├── INSTALLATION.md                  — Step-by-step setup: helpers (incl. 5 valve slots), card deployment, troubleshooting
├── hacs.json                        — HACS registry metadata (points at root card.yaml)
├── card.yaml                        — CURRENT card (version-agnostic; install docs link here)
├── automations.yaml                 — CURRENT rain-cancel automations (version-agnostic)
├── .gitignore
├── docs/
│   ├── ai-context.md                — This file. AI/Claude reference for repo navigation
│   └── design-multiple-schedules.md — Agreed design for independent schedules (Option 3 hybrid + FIFO)
├── multi-schedule/                  — [Unreleased] two-schedule bundle (A & B)
│   ├── README.md                    — Bundle overview + install order
│   ├── helpers.md                   — 50 per-schedule helpers, shared list, singleton→A migration
│   ├── card-a.yaml / card-b.yaml    — Namespaced clones of card.yaml (garden_a_* / garden_b_*)
│   ├── scripts.yaml                 — script.garden_a/b_watering_sequence (with single-valve cap guard)
│   ├── schedule-automations.yaml    — automation.garden_a/b_watering_schedule (+ shared winter-off cond)
│   └── rain-automations.yaml        — 3 shared rain automations generalised to both schedules
└── releases/
    ├── v0.1.0/ … v0.4.0/            — Earlier release snapshots (card.yaml + RELEASE_NOTES.md each)
    └── v0.5.0/
        ├── card.yaml                — Snapshot of the current card
        ├── automations.yaml         — Snapshot of the rain-cancel automations
        └── RELEASE_NOTES.md         — Release notes for v0.5.0
```

> The root `card.yaml` / `automations.yaml` are the canonical current files (install docs reference these, no version in the path). Each release also keeps a versioned snapshot under `releases/` for history.

---

## Card architecture

Single `custom:html-template-card` with `ignore_line_breaks: true`. Uses:
- **Jinja2** for all live state (helper entity reads, next-run date arithmetic, badge/class logic)
- **Inline CSS** scoped under `.gws-` prefix (Garden Watering Scheduler)
- **JavaScript IIFE onclick handlers** for all interactivity — `hass.callService()` calls via `document.querySelector('home-assistant').hass` (shadow DOM has no global `hass`); no `{% %}` block tags inside onclick strings (the `{{%` lexer trap)
- **Slot-loop rendering** — a `namespace(valves=[], total_d=0, parts=[])` accumulates non-empty slots 1–5; zone rows, total run, and the script all derive from it
- No external dependencies beyond `custom:html-template-card`

---

## Entity reference

### Helpers

| Entity | Type | Purpose |
|--------|------|---------|
| `input_boolean.garden_water_mon` … `_sun` | Toggle × 7 | Day-of-week schedule |
| `input_select.garden_water_start_time` | Dropdown | Start time (48 options: 00:00–23:30) |
| `input_text.garden_valve_1_entity` … `_5_entity` | Text × 5 | Valve switch entity ID per slot (blank = slot unused) |
| `input_text.garden_valve_1_name` … `_5_name` | Text × 5 | Zone display name per slot |
| `input_number.garden_valve_1_duration` … `_5_duration` | Number (0–60, step 5) × 5 | Per-zone minutes; 0 = excluded from run |
| `input_boolean.garden_schedule_armed` | Toggle | Schedule active/inactive |
| `input_boolean.garden_winter_shutdown` | Toggle | Seasonal suspension |
| `input_boolean.garden_rain_cancel` | Toggle | Skip today's run (manual or auto) |
| `input_number.garden_rain_threshold` | Number (0–100, step 5) | Forecast precipitation % threshold for auto rain cancel |
| `input_datetime.garden_last_rain` | Date+time | Last actual-rain onset; powers the 12 h lookback (rain cancel) |
| `input_datetime.garden_run_started` | Date+time | Stamped at run start by the script; powers the live during-run countdown |

No `timer.*` helpers are used — both countdowns are computed (next-run in Jinja2, during-run client-side from `garden_run_started`).

### Valve entities

Not hard-coded. Each slot's valve is whatever `switch.*` entity the user stores in `input_text.garden_valve_N_entity`. Zones run in slot order (1 → 5), skipping blank/zero slots. Reference install populated slot 1 = `switch.tap_lhs_upper_lawn_blue`, slot 2 = `switch.tap_lhs_lower_lawn_green`, with slots 3–5 available for fence/veg-trug valves.

---

## Card state logic

| Condition | Badge | Go button | Status area |
|-----------|-------|-----------|-------------|
| `winter == on` | Winterised (blue) | Dim / no-op | Suspension message |
| `winter == off, armed == off` | Disarmed (grey) | Active | "Press Go to arm" |
| `armed == on, rain == on` | Rain skip (blue) | Active (overrides rain) | Struck-through next run + rescheduled date |
| `armed == on, rain == off, any_day` | Armed (green) | Active | Next run date/time **+ live countdown** (`in 6d 22h 10m`) |
| `armed == on, no days selected` | Armed (green) | Active | "No days selected" |
| `script.garden_watering_sequence == on` | — | ▶ Start now dims | Status shows **Time remaining** countdown (ticks client-side) |

---

## Button behaviour

Header group order: **▶ Start now · ■ Stop · ❄ Winterise · 🌧 Rain · badge**. The **Go** button is full-width in the body (arms the schedule).

| Button | Action when clicked |
|--------|-------------------|
| ▶ Start now (header) | `script.turn_on(garden_watering_sequence)` — runs immediately. No-op if winterised or already running; lights up (`gws-rb-on`) while running |
| ■ Stop (header) | `script.turn_off(garden_watering_sequence)` + `homeassistant.turn_off` on **every** configured `garden_valve_N_entity` (templated `{% for valve in vns.valves %}`) + `turn_off(armed)` + `turn_off(winter_shutdown)` |
| ❄ Winterise (header, off→on) | `turn_on(winter_shutdown)` + `turn_off(armed)` |
| ❄ Winterise (header, on→off) | `turn_off(winter_shutdown)` |
| 🌧 Rain (header) | Toggles `input_boolean.garden_rain_cancel` |
| ▶ Go (body) | `turn_on(armed)` + `turn_off(rain_cancel)`. No-op if winterised. |
| Duration dot (5/10/15/20) | `input_number.set_value` on that slot's `garden_valve_N_duration` |
| ✗ dot (per zone) | Sets that slot's `garden_valve_N_duration` to 0 (excludes the zone) |

---

## Next-run date calculation (Jinja2)

Uses `namespace` to walk 0–7 days ahead from today's weekday (0=Mon, 6=Sun):
1. If today's day is enabled AND current `HH:MM` < start_time → offset = 0 (today). Current time is built manually from `now().hour`/`now().minute` (zero-padded), **not** `strftime` — defensive `%`-avoidance.
2. Otherwise scan days 1–7 forward and take the first enabled one
3. `now() + timedelta(days=offset)` gives the date; formatted as `"Fri 20 Jun — 06:30"`

A second namespace (`ns2`) calculates the rain-skip "next after" date by always starting from offset 1 (never today).

**Next-run countdown:** in the same `ns.off >= 0 and any_day` branch, `rem_min = (ns.off*1440 + sh*60 + sm) - (now().hour*60 + now().minute)`, then split into days/hours/minutes with `//` and subtraction (no `%`). Rendered as `in 6d 22h 10m` (days component dropped when < 24h) on a `.gws-skipn` sub-line under the next-run date. This one is server-side, so it refreshes on HA's ~per-minute `now()` cadence — fine for a far-off date.

**During-run "Time remaining" countdown (client-side ticker):** when `seq_on`, the template computes `end_ts = run_ts + total_d*60` (where `run_ts = state_attr('input_datetime.garden_run_started','timestamp')`) and renders `<span class="gwscd">MM:SS</span>` (server-side fallback) plus a hidden `<img src="x" onerror="…">`. The `onerror` handler (browser-executed, like the `onclick` handlers) starts a `setInterval(…,1000)` stored on `window.gwsT` that updates the span every second from `end_ts*1000 - Date.now()`. It clears any prior interval first (so re-renders don't stack), self-syncs to `end_ts` on every re-render, and `clearInterval`s at 0. **No `%`/`{%`/`%}` in the JS** — minutes/seconds split via `Math.floor` and subtraction — preserving the `{{%` defensive rule. This is the only way to get true second-by-second ticking inside `html-template-card`, whose content is otherwise re-rendered server-side.

---

## CSS class reference (`.gws-` prefix)

| Class | Purpose |
|-------|---------|
| `.gws-hdr` | Header flex row |
| `.gws-ttl` | Card title — uppercase, secondary colour |
| `.gws-rb` / `.gws-rb-on` | Rain button — default / active (blue tint) |
| `.gws-badge` | Badge container |
| `.gws-barmed` | Armed badge — green |
| `.gws-bdisarm` | Disarmed badge — grey |
| `.gws-bwinter` | Winterised badge — blue |
| `.gws-brain` | Rain skip badge — blue |
| `.gws-day` / `.gws-don` / `.gws-ddim` | Day button — default / on (green) / winter-dimmed |
| `.gws-ctl` | Mid-row control box background |
| `.gws-sv` | Value text (start time, total run) |
| `.gws-zone` / `.gws-zn` / `.gws-dots` | Per-zone row / zone name / dot container |
| `.gws-dot` / `.gws-don2` / `.gws-don-x` | Duration dot — default / selected (blue) / off-dot active (red) |
| `.gws-dl` | Per-zone duration label (right-aligned, `width:40px` to fit "20 min") |
| `.gws-bgo` / `.gws-bgo-dim` | Go button — active / disabled |
| `.gws-stat` | Status area background box |
| `.gws-stl` / `.gws-stv` | Status label / value (also used for "Time remaining" + countdown) |
| `.gwscd` | Countdown span updated by the client-side ticker (no own CSS; inherits `.gws-stv`) |
| `.gws-skip` / `.gws-skipn` | Rain-skip struck-through date / replacement date & countdown sub-line |
| `.gws-ntcr` / `.gws-ntcw` | Rain / winter notice box |

---

## HA script and automation (v0.2.0)

### `script.garden_watering_sequence`

Step 0: `input_datetime.set_datetime` stamps `garden_run_started` with `{{ now().timestamp() | int }}` (powers the during-run countdown). Then five slot-based `if/then` steps (one per slot 1–5), run in order. Each step:
1. Guard: `numeric_state` on `input_number.garden_valve_N_duration` `above: 0` (skips the slot if 0)
2. `homeassistant.turn_on` with `entity_id: "{{ states('input_text.garden_valve_N_entity') }}"`
3. `delay` of `{{ states('input_number.garden_valve_N_duration') | int }}` minutes
4. `homeassistant.turn_off` on the same templated entity

Mode: `single`. Adding/removing a valve = editing the slot helpers; the script needs no changes (it always covers slots 1–5). The script is started both by the schedule automation and by the card's ▶ Start now button; ■ Stop turns it off and closes valves.

### `automation.garden_watering_schedule`

Triggers: `time_pattern` at minutes `0` and `30` (covers all 48 half-hour slots).

Conditions (all must pass):
1. `input_boolean.garden_schedule_armed` == `on`
2. `input_boolean.garden_rain_cancel` == `off`
3. Template: `now().strftime('%H:%M') == states('input_select.garden_water_start_time')`
4. Template: today's weekday index maps to the matching `input_boolean.garden_water_{day}` == `on`

Action: `script.garden_watering_sequence`. Mode: `single`.

---

## Rain cancel automations (v0.4.0)

Detection-only — no card or script change. Three automations set the existing `input_boolean.garden_rain_cancel`. Weather entity is **parameterised** (set in two spots in `releases/v0.4.0/automations.yaml`); the live install uses Met Office (`weather.met_office_fleet`), with `weather.forecast_home_weather` (met.no) as the other available provider.

### `automation.garden_rain_recorder`
State trigger on the weather entity → `to:` `[rainy, pouring, lightning-rainy, snowy-rainy, hail]`. Action: `input_datetime.set_datetime` stamps `garden_last_rain = now()`. This is the **actual**-rain source (the weather entity's `state`/condition). Mode `single`.

### `automation.garden_rain_auto_cancel_check`
Triggers: `time_pattern` minutes 0/30. Conditions (gate to a run actually due): armed on, winter off, rain_cancel off, `(now()+30min) HH:MM == start_time`, and the run-day (`now()+30min` weekday) day-boolean is on. Actions: `weather.get_forecasts` (`type: hourly`, `response_variable: fc`) → `variables` compute `max_prob` (highest `precipitation_probability` over hours within next 24 h; provider-agnostic via `fc.values() | first`), `rained_recently` (`now − garden_last_rain < 43200 s`) → `choose`: if `rained_recently or max_prob >= threshold`, `input_boolean.turn_on` rain_cancel + `persistent_notification.create` (id `garden_rain_cancel`). Mode `single`.

### `automation.garden_rain_cancel_daily_reset`
Triggers: `time_pattern` minutes 0/30. Conditions: `(now()-30min) HH:MM == start_time` and rain_cancel on. Actions: `input_boolean.turn_off` rain_cancel + `persistent_notification.dismiss` (id `garden_rain_cancel`). Mode `single`.

**Design:** anchored to ±30 min of `start_time` (no fixed-clock collisions, no race with the schedule automation). Auto-check only turns the flag **on** so a manual 🌧 skip is never overridden; the reset turns it off. Notify target = HA persistent notification. History Stats was avoided because that helper type isn't creatable via the config-flow API and templates can't query history — hence the `input_datetime` + recorder pattern. Forecast decision is on **probability %**, not mm (providers often report 0 mm at moderate probability). See [[project-v040-weather]] in Claude's memory for the as-built record.

---

## Multiple independent schedules (`multi-schedule/`)

Status: `[Unreleased]`. Two independent schedules built on **Option 3 hybrid namespace** + a
**FIFO single-valve** overlap policy. Design rationale: `docs/design-multiple-schedules.md`.
Bundle: `multi-schedule/` — `card-a.yaml`, `card-b.yaml`, `scripts.yaml`,
`schedule-automations.yaml`, `rain-automations.yaml`, `helpers.md`, `README.md`.

### Namespacing
- **Per-schedule (`garden_a_*` / `garden_b_*`):** 7 day toggles, start time, valve slots 1–5
  (entity/name/duration), `schedule_armed`, `run_started` stamp; one
  `script.garden_X_watering_sequence` + one `automation.garden_X_watering_schedule` each.
- **Shared house-wide (single instance, NOT namespaced):** `garden_rain_cancel`,
  `garden_rain_threshold`, `garden_last_rain` + the 3 rain automations; `garden_winter_shutdown`;
  the weather entity.

### Single-valve cap (FIFO) — the key new logic
Every `homeassistant.turn_on` in each sequence script is guarded by a template that is true only
when **no** garden valve is currently on across **both** namespaces' `*_valve_N_entity` (so
manual/pool opens hold the slot too). The whole zone block (turn_on + delay + turn_off) is wrapped,
so a blocked zone is **skipped, not queued**. First command wins; off always wins; scripts never
re-assert — no lock entity. The guard hardcodes the namespace list `['a','b']`; **extend it per new
schedule** (the only cross-schedule edit besides the rain templates).

### Schedule-automation change vs single-schedule
Both `automation.garden_X_watering_schedule` carry an explicit
`input_boolean.garden_winter_shutdown == off` condition. The single-schedule design relied on ❄
disarming the one schedule; per-schedule armed flags break that, so the shared ❄ flag is now a hard
condition on every schedule (winterise once → all suspended).

### Rain automations, generalised (recorder unchanged)
- **Auto-cancel check:** singleton armed+start+day gate replaced by an **OR across both schedules**
  (fires 30 min before *either* due schedule); still only ever turns the shared flag **on**.
- **Daily reset:** clears 30 min after the **latest** of the schedules' start times (a rainy day
  skips both an early and a late run before clearing).
- Weather entity preserved on the live install: `weather.met_office_fleet`.

### Card namespacing (find-replace from `card.yaml`)
`input_boolean.garden_water_` → `…garden_a_water_`; `input_select.garden_water_start_time` →
`…garden_a_water_start_time`; `input_text.garden_valve_` / `input_number.garden_valve_` →
`…garden_a_valve_`; `garden_schedule_armed` → `garden_a_schedule_armed`;
`script.garden_watering_sequence` → `script.garden_a_watering_sequence`;
`input_datetime.garden_run_started` → `…garden_a_run_started`; **`window.gwsT` → `window.gwsT_a`**
(the client-side countdown global — MUST be namespaced or two cards on one page collide); title
`Garden watering (A)`. Same set with `b`. Shared helper names left untouched. No `{%`/`%}`
introduced — preserves the `{{%` defensive-templating rule.

### As-built / migration (live on My Home)
Singleton `garden_*` helpers renamed → `garden_a_*` (values + history preserved via entity-registry
rename); 25 `garden_b_*` created fresh (blank, durations 0). Old `script.garden_watering_sequence` +
`automation.garden_watering_schedule` removed. Cards live at **My Home → Garden → "Garden Watering
Schedules"** (`views[5].sections[9].cards[1]` = A, `cards[2]` = B; section heading retitled to plural).
Dashboard edit done as a server-side string transform of the existing card (→ A) plus an appended
copy (→ B), so the deployed cards match the `multi-schedule/` files exactly.

### Deferred (NOT built — recorded in the design doc)
Master meter-valve linkage, two-part winterise model, drain sequencing
(close-zones-before-supply), leak detection. Likely owned by the meter project later.

---

## Build phases

| Phase | Version | Feature | HA additions required |
|-------|---------|---------|----------------------|
| 1 | v0.1.0 ✅ | Card UI, all helpers, next-run display | helpers |
| 2 | v0.2.0 ✅ | Scheduling engine — script + automation | 1 script, 1 automation |
| 3 | v0.3.0 ✅ | Dynamic valve list (1–5 zones), next-run countdown, zone-exclude dot, Test button | 15 valve-slot helpers; script rewritten |
| 4 | v0.4.0 ✅ | Automatic rain cancel (12 h actual + 24 h forecast, with notification) | 1 input_datetime helper + 3 automations (recorder, auto-check, daily reset) |
| 5 | v0.5.0 ✅ | ▶ Start now / ■ Stop header controls; client-side ticking Time-remaining countdown (Test button + interim progress bar removed) | 1 input_datetime helper (`garden_run_started`); script stamps it as step 0 |
| — | Unreleased ✅ (deployed) | Multiple independent schedules — namespaced `garden_a_*` / `garden_b_*`, FIFO single-valve cap, shared rain + winterise (`multi-schedule/`) | 50 helpers (25 per schedule), 2 scripts, 2 schedule automations, 3 rain automations generalised, 2 cards |
| — | v1.0.0 | Full polish + complete docs | None |

---

## How to publish a new version

1. Update the canonical root `card.yaml` (and `automations.yaml` if changed) to the current build
2. Copy them into a new `releases/vX.Y.Z/` snapshot alongside a `RELEASE_NOTES.md`
3. `hacs.json` already points at root `card.yaml` — no path change needed per release
4. Update `CHANGELOG.md` — move content from `[Unreleased]` to the new version section
5. Update `README.md` versions table to mark the new version complete
6. Commit: `git commit -m "feat: vX.Y.Z — description"`
7. Tag: `git tag -a vX.Y.Z -m "vX.Y.Z — description"`
8. Push: `git push origin master --tags`
9. On GitHub: create a Release from the tag, paste `RELEASE_NOTES.md` as body

## How to modify the card via Claude

Tell Claude:

> "Update the Garden Watering Scheduler card. [describe change]. Reference https://github.com/rhamblen/garden-watering-scheduler for context."

Claude should:
1. Read `docs/ai-context.md` for repo context
2. Read the root `card.yaml` for current card source
3. Apply the change
4. Update `CHANGELOG.md` under `[Unreleased]`
5. Commit and push

## How to add the card to a HA dashboard via Claude

Tell Claude:

> "Add the Garden Watering Scheduler card to my [dashboard] dashboard, [view] view, [section] section."

Claude should:
1. Call `ha_config_get_dashboard(url_path='...')` to locate the target section
2. Note the `config_hash`
3. Build the card YAML from the root `card.yaml`
4. Call `ha_config_set_dashboard(python_transform='...', config_hash='...')` to insert the card
