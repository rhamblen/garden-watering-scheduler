# AI Context — Garden Watering Scheduler Card

This file helps Claude (or any AI assistant) understand the repository structure so it can assist with modifications, updates, and publishing new versions.

---

## Repository purpose

This repository contains a Home Assistant Lovelace dashboard card that schedules sequential two-zone garden watering. A single `custom:html-template-card` provides the full UI — day selection, time picking, duration selection, armed/disarmed/winterised state management, and next-run display. The actual watering sequence is controlled by a separate HA script and automation (added in v0.2.0).

## Companion project

**Zigbee Smart Water Valve Card** — https://github.com/rhamblen/Zigbee-smart-water-valve
Same tech stack. Defines the valve entities this scheduler uses. Reference for CSS style consistency.

---

## File structure

```
/
├── README.md                        — Project description, features, entity table, sequence diagram
├── CHANGELOG.md                     — Version history (Keep a Changelog format)
├── INSTALLATION.md                  — Step-by-step setup: 16 helpers, card deployment, troubleshooting
├── hacs.json                        — HACS registry metadata
├── .gitignore
├── docs/
│   └── ai-context.md                — This file. AI/Claude reference for repo navigation
└── releases/
    └── v0.1.0/
        ├── card.yaml                — Full Lovelace card YAML (copy-paste ready)
        └── RELEASE_NOTES.md         — Release-specific notes for v0.1.0
```

---

## Card architecture

Single `custom:html-template-card` with `ignore_line_breaks: true`. Uses:
- **Jinja2** for all live state (helper entity reads, next-run date arithmetic, badge/class logic)
- **Inline CSS** scoped under `.gws-` prefix (Garden Watering Scheduler)
- **JavaScript IIFE onclick handlers** for all interactivity — `hass.callService()` calls, no window functions needed (fixed entity names, no multi-instance collision risk)
- No external dependencies beyond `custom:html-template-card`

---

## Entity reference

### Helpers (16 total)

| Entity | Type | Purpose |
|--------|------|---------|
| `input_boolean.garden_water_mon` … `_sun` | Toggle × 7 | Day-of-week schedule |
| `input_select.garden_water_start_time` | Dropdown | Start time (48 options: 00:00–23:30) |
| `input_number.garden_water_upper_duration` | Number (5–20, step 5) | Upper zone minutes |
| `input_number.garden_water_lower_duration` | Number (5–20, step 5) | Lower zone minutes |
| `input_boolean.garden_schedule_armed` | Toggle | Schedule active/inactive |
| `input_boolean.garden_winter_shutdown` | Toggle | Seasonal suspension |
| `input_boolean.garden_rain_cancel` | Toggle | Skip today's run (manual or auto) |
| `input_number.garden_rain_threshold` | Number (0–100, step 5) | Auto rain cancel % threshold |
| `timer.garden_upper_lawn` | Timer | Active countdown — upper zone (v0.3.0) |
| `timer.garden_lower_lawn` | Timer | Active countdown — lower zone (v0.3.0) |

### Valve entities (from companion project)

| Entity | Role |
|--------|------|
| `switch.tap_lhs_upper_lawn_blue` | Upper lawn valve (runs first) |
| `switch.tap_lhs_lower_lawn_green` | Lower lawn valve (runs second) |

---

## Card state logic

| Condition | Badge | Go button | Status area |
|-----------|-------|-----------|-------------|
| `winter == on` | Winterised (blue) | Dim / no-op | Suspension message |
| `winter == off, armed == off` | Disarmed (grey) | Active | "Press Go to arm" |
| `armed == on, rain == on` | Rain skip (blue) | Active (overrides rain) | Struck-through next run + rescheduled date |
| `armed == on, rain == off, any_day` | Armed (green) | Active | Next run date/time |
| `armed == on, no days selected` | Armed (green) | Active | "No days selected" |
| upper timer active (v0.3.0) | Running (green) | Dim | Zone countdown + progress |

---

## Button behaviour

| Button | Action when clicked |
|--------|-------------------|
| 🌧 (header) | Toggles `input_boolean.garden_rain_cancel` |
| ▶ Go | `turn_on(armed)` + `turn_off(rain_cancel)`. No-op if winterised. |
| ❄ Winterise (off→on) | `turn_on(winter_shutdown)` + `turn_off(armed)` |
| ❄ Winterise (on→off) | `turn_off(winter_shutdown)` |
| ■ Disarm | `turn_off(armed)` + `turn_off(winter_shutdown)` |
| Cancel this run (v0.3.0) | Stop script + cancel both timers + close both valves |

---

## Next-run date calculation (Jinja2)

Uses `namespace` to walk 0–7 days ahead from today's weekday (0=Mon, 6=Sun):
1. If today's day is enabled AND `now().strftime('%H:%M') < start_time` → offset = 0 (today)
2. Otherwise scan days 1–7 forward and take the first enabled one
3. `now() + timedelta(days=offset)` gives the date; formatted as `"Fri 20 Jun — 06:30"`

A second namespace (`ns2`) calculates the rain-skip "next after" date by always starting from offset 1 (never today).

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
| `.gws-dot` / `.gws-don2` | Duration dot — default / selected (green) |
| `.gws-bgo` / `.gws-bgo-dim` | Go button — active / disabled |
| `.gws-bw` / `.gws-bw-on` | Winterise button — default / on (blue) |
| `.gws-bdis` | Disarm button |
| `.gws-stat` | Status area background box |
| `.gws-skip` / `.gws-skipn` | Rain-skip struck-through date / replacement date |
| `.gws-ntcr` / `.gws-ntcw` | Rain / winter notice box |

---

## Build phases

| Phase | Version | Feature | HA additions required |
|-------|---------|---------|----------------------|
| 1 | v0.1.0 ✅ | Card UI, all helpers, next-run display | 16 helpers |
| 2 | v0.2.0 | Scheduling engine — script + automation | 1 script, 1 automation |
| 3 | v0.3.0 | Active countdown, progress bar, Cancel | Timer integration in card |
| 4 | v0.4.0 | Automatic rain cancel | 1 automation (daily weather check) + 1 reset automation |
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
3. Build the card YAML from `releases/v0.1.0/card.yaml`
4. Call `ha_config_set_dashboard(python_transform='...', config_hash='...')` to insert the card
