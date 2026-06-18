# Installation Guide

**Current version:** v0.1.0
**Repository:** https://github.com/rhamblen/garden-watering-scheduler

---

## Choose your installation method

### Option A — Claude-assisted (recommended)

No manual YAML or helper setup required. Claude reads the repository and configures everything.

**What you need first:**
- HACS installed in Home Assistant
- `custom:html-template-card` installed via HACS → Frontend → search "HTML Template Card"
- Claude with Home Assistant MCP connected

**Then tell Claude:**

> "Help me install the Garden Watering Scheduler from https://github.com/rhamblen/garden-watering-scheduler — add it to my [dashboard name] dashboard."

Claude will create all 16 helpers, the watering script, the scheduling automation, and deploy the card.

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

##### 1i. Upper zone duration

Helper type: **Number**

| Field | Value |
|-------|-------|
| Name | `Garden water upper duration` |
| Minimum | `5` |
| Maximum | `20` |
| Step size | `5` |
| Unit of measurement | `min` |
| Display mode | Box |
| Icon | `mdi:timer` |

Creates `input_number.garden_water_upper_duration`. Set initial value to `15`.

---

##### 1j. Lower zone duration

Helper type: **Number**

| Field | Value |
|-------|-------|
| Name | `Garden water lower duration` |
| Minimum | `5` |
| Maximum | `20` |
| Step size | `5` |
| Unit of measurement | `min` |
| Display mode | Box |
| Icon | `mdi:timer` |

Creates `input_number.garden_water_lower_duration`. Set initial value to `10`.

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

---

##### 1o. Upper lawn timer

Helper type: **Timer**

| Field | Value |
|-------|-------|
| Name | `Garden upper lawn` |
| Duration | `00:20:00` |
| Restore | enabled |
| Icon | `mdi:timer` |

Creates `timer.garden_upper_lawn`. Used from v0.3.0 for the active countdown display.

---

##### 1p. Lower lawn timer

Helper type: **Timer**

| Field | Value |
|-------|-------|
| Name | `Garden lower lawn` |
| Duration | `00:20:00` |
| Restore | enabled |
| Icon | `mdi:timer` |

Creates `timer.garden_lower_lawn`. Used from v0.3.0 for the active countdown display.

---

#### Step 2 — Add the card to your dashboard

1. Download [`releases/v0.1.0/card.yaml`](releases/v0.1.0/card.yaml)
2. Open your Lovelace dashboard → click the pencil (edit) icon
3. Click **+ Add card** in the target section
4. Scroll to the bottom and choose **Manual card**
5. Delete the placeholder YAML and paste the full contents of `card.yaml`
6. Click **Save**

---

#### Step 3 — Verify

The card should show:

- **GARDEN WATERING** title with 🌧 button and Disarmed badge in the header
- **Day buttons** M T W T F S S — tap to toggle green
- **Start time** stepper showing the current `input_select` value
- **Zone duration** dots — one dot selected per zone (filled green)
- **Go / Winterise / Disarm** buttons
- **"Press Go to arm the schedule"** in the status area

Tap **Go** — badge changes to Armed and the status shows the next run date.

---

#### Step 4 — Scheduling automations (v0.2.0)

The card UI is fully functional from v0.1.0. The automations that actually trigger watering are added in v0.2.0. Until then, use the individual valve cards from the [Zigbee Smart Water Valve](https://github.com/rhamblen/Zigbee-smart-water-valve) project to open valves manually.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Card shows "Custom element doesn't exist" | `html-template-card` not installed | Install via HACS and reload browser |
| All values show `unknown` | Helpers not created or wrong entity IDs | Check Settings → Helpers for all 16 entities |
| Day buttons do nothing | Helper entity IDs don't match | Verify `input_boolean.garden_water_mon` etc. exist exactly |
| Start time arrows do nothing | `input_select` not created or wrong entity ID | Verify `input_select.garden_water_start_time` exists with 48 options |
| Next run shows "No days selected" | No day toggles are on | Tap at least one day button |
| Next run date looks wrong | Time zone mismatch | Check HA time zone setting matches your local time |
