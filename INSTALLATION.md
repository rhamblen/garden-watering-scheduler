# Installation Guide

**Current version:** v0.3.0
**Repository:** https://github.com/rhamblen/garden-watering-scheduler

---

## Choose your installation method

### Option A — Claude-assisted (recommended)

No manual YAML or helper setup required. Claude reads the repository and configures everything — including discovering your Zigbee valve switches and wiring them into the zone slots.

**What you need first:**
- HACS installed in Home Assistant
- `custom:html-template-card` installed via HACS → Frontend → search "HTML Template Card"
- Claude with Home Assistant MCP connected
- Your watering valves paired as `switch.*` entities (any number, 1–5)

**Then tell Claude:**

> "Help me install the Garden Watering Scheduler from https://github.com/rhamblen/garden-watering-scheduler — discover my Zigbee valve switches, set them up as zones, and add the card to my [dashboard name] dashboard."

Claude will:
1. Search your HA instance for valve switch entities and confirm which to use
2. Create all helpers — including the five valve slots, pre-filled with your switch entity IDs and names
3. Create the watering script (`script.garden_watering_sequence`) and the scheduling automation
4. Deploy the card to your chosen dashboard

You can start with a single valve and add more later — just populate the next slot's entity, name, and duration helpers; the card and script pick it up automatically with no further changes.

---

### Option B — Manual installation

#### Prerequisites

| Requirement | How to get it |
|-------------|--------------|
| HACS | https://hacs.xyz |
| `custom:html-template-card` | HACS → Frontend → search "HTML Template Card" |
| Zigbee valves paired via Zigbee2MQTT | Settings → Devices & Services → Zigbee2MQTT |

---

#### Step 1 — Create HA helpers

Create all helpers at **Settings → Devices & Services → Helpers → Create helper**.

> Create them in order — some reference the entities created before them.

---

##### 1a–1g. Day toggle helpers (create 7)

Helper type: **Toggle**

Create one for each day. Name and entity ID:

| Name | Entity ID created |
|------|------------------|
| `Garden water Monday` | `input_boolean.garden_water_mon` |
| `Garden water Tuesday` | `input_boolean.garden_water_tue` |
| `Garden water Wednesday` | `input_boolean.garden_water_wed` |
| `Garden water Thursday` | `input_boolean.garden_water_thu` |
| `Garden water Friday` | `input_boolean.garden_water_fri` |
| `Garden water Saturday` | `input_boolean.garden_water_sat` |
| `Garden water Sunday` | `input_boolean.garden_water_sun` |

Leave all off by default — select your schedule days on the card.

---

##### 1h. Start time selector

Helper type: **Dropdown**

| Field | Value |
|-------|-------|
| Name | `Garden water start time` |
| Options | See the full list of 48 options below |
| Initial option | `06:00` |
| Icon | `mdi:clock-start` |

> **Tip:** Creating a 48-option dropdown via the UI is tedious. The Claude-assisted method (Option A) does this in one step. Alternatively, add it via `configuration.yaml` — see the YAML block below.

**All 48 options** (paste into the UI one by one, or use the YAML approach):

```
00:00  00:30  01:00  01:30  02:00  02:30  03:00  03:30
04:00  04:30  05:00  05:30  06:00  06:30  07:00  07:30
08:00  08:30  09:00  09:30  10:00  10:30  11:00  11:30
12:00  12:30  13:00  13:30  14:00  14:30  15:00  15:30
16:00  16:30  17:00  17:30  18:00  18:30  19:00  19:30
20:00  20:30  21:00  21:30  22:00  22:30  23:00  23:30
```

**YAML alternative** — add to `configuration.yaml` and restart HA:

```yaml
input_select:
  garden_water_start_time:
    name: Garden water start time
    options:
      - "00:00"
      - "00:30"
      - "01:00"
      - "01:30"
      - "02:00"
      - "02:30"
      - "03:00"
      - "03:30"
      - "04:00"
      - "04:30"
      - "05:00"
      - "05:30"
      - "06:00"
      - "06:30"
      - "07:00"
      - "07:30"
      - "08:00"
      - "08:30"
      - "09:00"
      - "09:30"
      - "10:00"
      - "10:30"
      - "11:00"
      - "11:30"
      - "12:00"
      - "12:30"
      - "13:00"
      - "13:30"
      - "14:00"
      - "14:30"
      - "15:00"
      - "15:30"
      - "16:00"
      - "16:30"
      - "17:00"
      - "17:30"
      - "18:00"
      - "18:30"
      - "19:00"
      - "19:30"
      - "20:00"
      - "20:30"
      - "21:00"
      - "21:30"
      - "22:00"
      - "22:30"
      - "23:00"
      - "23:30"
    initial: "06:00"
    icon: mdi:clock-start
```

This creates `input_select.garden_water_start_time`.

---

##### 1i. Valve slots (create 15 helpers — five zones × three helpers each)

The card and watering script support **1–5 valve zones**. Each zone occupies a numbered slot (1–5) made up of three helpers: the valve's switch entity, a display name, and a run duration. **You only need to fill the slots you use** — leave the entity blank on unused slots and they are ignored everywhere.

> Tip: the Claude-assisted method (Option A) discovers your valve switches and fills these in for you. Doing it by hand, create the slots you need (most installs start with 1–2).

**1i-a. Valve entity (create up to 5)** — Helper type: **Text**

| Name | Entity ID created | Set value to |
|------|------------------|--------------|
| `Garden Valve 1 Entity` | `input_text.garden_valve_1_entity` | your 1st valve switch, e.g. `switch.tap_lhs_upper_lawn_blue` |
| `Garden Valve 2 Entity` | `input_text.garden_valve_2_entity` | your 2nd valve switch (or leave blank) |
| `Garden Valve 3 Entity` | `input_text.garden_valve_3_entity` | your 3rd valve switch (or leave blank) |
| `Garden Valve 4 Entity` | `input_text.garden_valve_4_entity` | your 4th valve switch (or leave blank) |
| `Garden Valve 5 Entity` | `input_text.garden_valve_5_entity` | your 5th valve switch (or leave blank) |

**1i-b. Valve name (create up to 5)** — Helper type: **Text**

| Name | Entity ID created | Set value to |
|------|------------------|--------------|
| `Garden Valve 1 Name` | `input_text.garden_valve_1_name` | label for slot 1, e.g. `Upper lawn` |
| `Garden Valve 2 Name` | `input_text.garden_valve_2_name` | label for slot 2, e.g. `Lower lawn` |
| `Garden Valve 3 Name` | `input_text.garden_valve_3_name` | label for slot 3 |
| `Garden Valve 4 Name` | `input_text.garden_valve_4_name` | label for slot 4 |
| `Garden Valve 5 Name` | `input_text.garden_valve_5_name` | label for slot 5 |

**1i-c. Valve duration (create up to 5)** — Helper type: **Number**

For each slot: Minimum `0`, Maximum `60`, Step `5`, Unit `min`, Display mode **Box**.

| Name | Entity ID created | Initial value |
|------|------------------|---------------|
| `Garden Valve 1 Duration` | `input_number.garden_valve_1_duration` | `15` |
| `Garden Valve 2 Duration` | `input_number.garden_valve_2_duration` | `10` |
| `Garden Valve 3 Duration` | `input_number.garden_valve_3_duration` | `0` |
| `Garden Valve 4 Duration` | `input_number.garden_valve_4_duration` | `0` |
| `Garden Valve 5 Duration` | `input_number.garden_valve_5_duration` | `0` |

A duration of **0** excludes that zone from the watering run (shown with a red ✗ dot on the card) while keeping the slot visible. Slots with a blank entity are dropped entirely.

---

##### 1k. Schedule armed

Helper type: **Toggle**

| Field | Value |
|-------|-------|
| Name | `Garden schedule armed` |
| Icon | `mdi:calendar-check` |

Creates `input_boolean.garden_schedule_armed`. Leave off by default.

---

##### 1l. Winter shutdown

Helper type: **Toggle**

| Field | Value |
|-------|-------|
| Name | `Garden winter shutdown` |
| Icon | `mdi:snowflake` |

Creates `input_boolean.garden_winter_shutdown`. Leave off by default.

---

##### 1m. Rain cancel

Helper type: **Toggle**

| Field | Value |
|-------|-------|
| Name | `Garden rain cancel` |
| Icon | `mdi:weather-rainy` |

Creates `input_boolean.garden_rain_cancel`. Leave off by default. Set automatically by the rain check automation (v0.4.0) or manually via the card header button.

---

##### 1n. Rain threshold

Helper type: **Number**

| Field | Value |
|-------|-------|
| Name | `Garden rain threshold` |
| Minimum | `0` |
| Maximum | `100` |
| Step size | `5` |
| Unit of measurement | `%` |
| Display mode | Slider |
| Icon | `mdi:water-percent` |

Creates `input_number.garden_rain_threshold`. Set initial value to `60` — watering skips when forecast precipitation probability is 60% or above.

> Per-zone live countdown timers (`timer.*`) are planned for v0.5.0 and are **not** required for the current release — the card computes the next-run countdown directly in Jinja2.

---

#### Step 2 — Add the card to your dashboard

1. Download [`releases/v0.3.0/card.yaml`](releases/v0.3.0/card.yaml)
2. Open your Lovelace dashboard → click the pencil (edit) icon
3. Click **+ Add card** in the target section
4. Scroll to the bottom and choose **Manual card**
5. Delete the placeholder YAML and paste the full contents of `card.yaml`
6. Click **Save**

---

#### Step 3 — Verify

The card should show:

- **GARDEN WATERING** title with ❄ / ■ / 🌧 buttons and Disarmed badge in the header
- **Day buttons** M T W T F S S — tap to toggle green
- **Start time** stepper showing the current `input_select` value
- **Total run** showing the summed duration of all active zones
- **Zone duration** rows — one row per configured valve slot, each with a red ✗ (off) dot plus 5/10/15/20-minute dots
- **Go** and **Test** buttons
- **"Press Go to arm the schedule"** in the status area

Tap **Go** — badge changes to Armed and the status shows the next run date with a live countdown (`in 6d 22h 10m`).

> Only slots whose **entity** helper is filled appear as zone rows. If you see fewer rows than expected, check the `input_text.garden_valve_N_entity` helpers.

---

#### Step 4 — Scheduling automations (v0.2.0)

The watering script (`script.garden_watering_sequence`) and the time-pattern automation that triggers it are created automatically by the Claude-assisted install (Option A). If you installed manually, create them from the definitions in the repo, or ask Claude to add them. Use the **Test** button on the card to run the full sequence on demand once they exist.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Card shows "Custom element doesn't exist" | `html-template-card` not installed | Install via HACS and reload browser |
| All values show `unknown` | Helpers not created or wrong entity IDs | Check Settings → Helpers that every helper from Step 1 exists |
| Day buttons do nothing | Helper entity IDs don't match | Verify `input_boolean.garden_water_mon` etc. exist exactly |
| Start time arrows do nothing | `input_select` not created or wrong entity ID | Verify `input_select.garden_water_start_time` exists with 48 options |
| A valve zone is missing from the card | That slot's entity helper is blank | Set `input_text.garden_valve_N_entity` to the valve's switch entity ID |
| A zone shows but never waters | Its duration is 0 (red ✗ dot) | Tap a minute dot to give the zone a non-zero duration |
| Watering runs but valve never opens | Wrong entity ID in the slot | Confirm `input_text.garden_valve_N_entity` matches a real `switch.*` entity exactly |
| Next run shows "No days selected" | No day toggles are on | Tap at least one day button |
| Next run date or countdown looks wrong | Time zone mismatch | Check HA time zone setting matches your local time |
