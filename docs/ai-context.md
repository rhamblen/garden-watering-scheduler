# AI Context — Garden Watering Scheduler Card

This file helps Claude (or any AI assistant) understand the repository structure so it can assist with modifications, updates, and publishing new versions.

---

## Repository purpose

This repository contains a Home Assistant Lovelace dashboard card that schedules sequential garden watering across **1–5 configurable valve zones**. A single `custom:html-template-card` provides the full UI — day selection, time picking, per-zone duration selection, armed/disarmed/winterised state management, next-run display with live countdown, and an on-demand Test button. The actual watering sequence is controlled by a separate HA script and automation (added in v0.2.0). Valves are not hard-coded — each zone's switch entity, name, and duration live in slot helpers (`garden_valve_N_*`, N = 1–5), so the card adapts to however many valves are configured.

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
├── hacs.json                        — HACS registry metadata
├── .gitignore
├── docs/
│   └── ai-context.md                — This file. AI/Claude reference for repo navigation
└── releases/
    ├── v0.1.0/ … v0.2.1/            — Earlier releases (card.yaml + RELEASE_NOTES.md each)
    └── v0.3.0/
        ├── card.yaml                — Current Lovelace card YAML (copy-paste ready)
        └── RELEASE_NOTES.md         — Release-specific notes for v0.3.0
```

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
| `input_number.garden_rain_threshold` | Number (0–100, step 5) | Auto rain cancel % threshold |

Per-zone active-countdown `timer.*` helpers are deferred to v0.5.0 and not currently created.

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
| `script.garden_watering_sequence == on` | — | — | Test button shows "⚫ Running…" |

---

## Button behaviour

| Button | Action when clicked |
|--------|-------------------|
| 🌧 (header) | Toggles `input_boolean.garden_rain_cancel` |
| ▶ Go | `turn_on(armed)` + `turn_off(rain_cancel)`. No-op if winterised. |
| ❄ Winterise (off→on) | `turn_on(winter_shutdown)` + `turn_off(armed)` |
| ❄ Winterise (on→off) | `turn_off(winter_shutdown)` |
| ■ Disarm | `turn_off(armed)` + `turn_off(winter_shutdown)` |
| Duration dot (5/10/15/20) | `input_number.set_value` on that slot's `garden_valve_N_duration` |
| ✗ dot (per zone) | Sets that slot's `garden_valve_N_duration` to 0 (excludes the zone) |
| 🔧 Test | `script.turn_on(garden_test_watering)` — 60 s grace countdown, then runs the sequence; dims while running |
| Cancel this run (v0.5.0) | Stop script + close active valve |

---

## Next-run date calculation (Jinja2)

Uses `namespace` to walk 0–7 days ahead from today's weekday (0=Mon, 6=Sun):
1. If today's day is enabled AND current `HH:MM` < start_time → offset = 0 (today). Current time is built manually from `now().hour`/`now().minute` (zero-padded), **not** `strftime` — defensive `%`-avoidance.
2. Otherwise scan days 1–7 forward and take the first enabled one
3. `now() + timedelta(days=offset)` gives the date; formatted as `"Fri 20 Jun — 06:30"`

A second namespace (`ns2`) calculates the rain-skip "next after" date by always starting from offset 1 (never today).

**Live countdown:** in the same `ns.off >= 0 and any_day` branch, `rem_min = (ns.off*1440 + sh*60 + sm) - (now().hour*60 + now().minute)`, then split into days/hours/minutes with `//` and subtraction (no `%`). Rendered as `in 6d 22h 10m` (days component dropped when < 24h) on a `.gws-skipn` sub-line under the next-run date.

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
| `.gws-tst` / `.gws-tst-busy` | Test button — default (amber) / running (dim) |
| `.gws-stat` | Status area background box |
| `.gws-skip` / `.gws-skipn` | Rain-skip struck-through date / replacement date & countdown sub-line |
| `.gws-ntcr` / `.gws-ntcw` | Rain / winter notice box |

---

## HA script and automation (v0.2.0)

### `script.garden_watering_sequence`

Five slot-based `if/then` steps (one per slot 1–5), run in order. Each step:
1. Guard: `numeric_state` on `input_number.garden_valve_N_duration` `above: 0` (skips the slot if 0)
2. `homeassistant.turn_on` with `entity_id: "{{ states('input_text.garden_valve_N_entity') }}"`
3. `delay` of `{{ states('input_number.garden_valve_N_duration') | int }}` minutes
4. `homeassistant.turn_off` on the same templated entity

Mode: `single`. Adding/removing a valve = editing the slot helpers; the script needs no changes (it always covers slots 1–5).

### `automation.garden_watering_schedule`

Triggers: `time_pattern` at minutes `0` and `30` (covers all 48 half-hour slots).

Conditions (all must pass):
1. `input_boolean.garden_schedule_armed` == `on`
2. `input_boolean.garden_rain_cancel` == `off`
3. Template: `now().strftime('%H:%M') == states('input_select.garden_water_start_time')`
4. Template: today's weekday index maps to the matching `input_boolean.garden_water_{day}` == `on`

Action: `script.garden_watering_sequence`. Mode: `single`.

---

## Build phases

| Phase | Version | Feature | HA additions required |
|-------|---------|---------|----------------------|
| 1 | v0.1.0 ✅ | Card UI, all helpers, next-run display | helpers |
| 2 | v0.2.0 ✅ | Scheduling engine — script + automation | 1 script, 1 automation |
| 3 | v0.3.0 ✅ | Dynamic valve list (1–5 zones), next-run countdown, zone-exclude dot, Test button | 15 valve-slot helpers; script rewritten |
| 4 | v0.4.0 | Automatic rain cancel | 1 automation (daily weather check) + 1 reset automation |
| 5 | v0.5.0 | Active per-zone countdown, progress bar, Cancel this run | Timer integration in card |
| — | v1.0.0 | Full polish + complete docs | None |

---

## How to publish a new version

1. Create `releases/vX.Y.Z/` with `card.yaml` and `RELEASE_NOTES.md`
2. Update `hacs.json` → `"filename"` to point to the new release path
3. Update `CHANGELOG.md` — move content from `[Unreleased]` to new version section
4. Update `README.md` versions table to mark the new version complete
5. Update version comment at top of `card.yaml`
6. Commit: `git commit -m "feat: vX.Y.Z — description"`
7. Tag: `git tag -a vX.Y.Z -m "vX.Y.Z — description"`
8. Push: `git push origin master --tags`
9. On GitHub: create a Release from the tag, paste `RELEASE_NOTES.md` as body

## How to modify the card via Claude

Tell Claude:

> "Update the Garden Watering Scheduler card. [describe change]. Reference https://github.com/rhamblen/garden-watering-scheduler for context."

Claude should:
1. Read `docs/ai-context.md` for repo context
2. Read `releases/vX.Y.Z/card.yaml` for current card source
3. Apply the change
4. Update `CHANGELOG.md` under `[Unreleased]`
5. Commit and push

## How to add the card to a HA dashboard via Claude

Tell Claude:

> "Add the Garden Watering Scheduler card to my [dashboard] dashboard, [view] view, [section] section."

Claude should:
1. Call `ha_config_get_dashboard(url_path='...')` to locate the target section
2. Note the `config_hash`
3. Build the card YAML from the latest `releases/vX.Y.Z/card.yaml` (currently `releases/v0.3.0/card.yaml`)
4. Call `ha_config_set_dashboard(python_transform='...', config_hash='...')` to insert the card
