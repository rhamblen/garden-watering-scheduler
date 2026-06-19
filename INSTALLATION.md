# Installation Guide

**Repository:** https://github.com/rhamblen/garden-watering-scheduler

---

## Choose your installation method

### Option A — Claude-assisted (recommended)

Claude reads the repository and sets everything up for you — no manual YAML or helper creation. It discovers your valve switches, creates the helpers, scripts, and automations, wires up rain cancel against your own weather forecast, and deploys the card.

**What you need first:**
- HACS installed in Home Assistant
- `custom:html-template-card` installed via HACS → Frontend → search "HTML Template Card"
- Claude with the Home Assistant MCP connected
- Your watering valves paired as `switch.*` entities (any number, 1–5)
- A weather integration installed if you want automatic rain cancel (Met Office, Met.no, OpenWeatherMap, AccuWeather, …) — Claude picks up the entity for you

**Tell Claude:**

> "Help me install the Garden Watering Scheduler from https://github.com/rhamblen/garden-watering-scheduler — discover my Zigbee valve switches, set them up as zones, add automatic rain cancel using my weather forecast, and put the card on my [dashboard name] dashboard."

Claude will:
1. Find your valve switch entities and confirm which to use
2. Create every helper, pre-filling the valve slots with your switch entity IDs and names
3. Create the watering script and the scheduling automation
4. Discover your weather entity and wire up the rain-cancel automations
5. Add the card to your chosen dashboard

Already running the scheduler and just want to add rain cancel? Ask:

> "Add automatic rain cancel to my Garden Watering Scheduler — discover my weather entity and set up the rain automations."

You can start with a single valve and add more later — populate the next slot's entity, name, and duration helpers and the card and script pick them up automatically, no further changes needed.

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

Creates `input_boolean.garden_rain_cancel`. Leave off by default. Set automatically by the rain-cancel automation, or manually via the card's 🌧 header button.

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

Creates `input_number.garden_rain_threshold`. Set initial value to `60` — the rain check skips watering when the **forecast** precipitation probability for any hour in the next 24 h is at or above this value. Lower it (e.g. `40`) to skip on lighter chances of rain; raise it to only skip when rain is very likely.

---

##### 1o. Last rain timestamp (needed for automatic rain cancel)

Helper type: **Date and/or time** — enable **both** date and time.

| Field | Value |
|-------|-------|
| Name | `Garden Last Rain` |
| Date | ✅ enabled |
| Time | ✅ enabled |
| Icon | `mdi:weather-rainy` |

Creates `input_datetime.garden_last_rain`. After creating it, set it once to a clearly-old value (e.g. `2020-01-01 00:00`) so it never reads as "rained recently" before the recorder automation first fires. The `Garden Rain Recorder` automation (Step 5) keeps it updated from then on. Skip this helper if you don't want automatic rain cancel.

> You do not need to create any `timer.*` helpers — the card computes the next-run countdown directly in Jinja2.

---

#### Step 2 — Add the card to your dashboard

1. Download [`releases/v0.4.0/card.yaml`](releases/v0.4.0/card.yaml)
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

#### Step 4 — Watering script and schedule automation

The watering script (`script.garden_watering_sequence`) and the time-pattern automation that triggers it are created automatically by the Claude-assisted install (Option A). If you installed manually, create them from the definitions in the repo, or ask Claude to add them. Use the **Test** button on the card to run the full sequence on demand once they exist.

---

#### Step 5 — Automatic rain cancel (optional)

This step is **optional** — skip it if you only want manual rain skipping via the card's 🌧 button. It adds three automations that automatically skip a scheduled run when it has rained recently or rain is forecast, and post a Home Assistant notification when they do. The card does not change — the automations simply set `input_boolean.garden_rain_cancel`, which the scheduler already honours.

**Prerequisites:** the `input_datetime.garden_last_rain` helper from Step 1o, and a working weather integration (see below).

---

##### 5a. Pick your weather provider

The rain check reads a Home Assistant **weather entity** — it is *not* hard-coded, so you choose which provider to trust.

1. Make sure you have at least one weather integration installed (**Settings → Devices & Services → Add Integration** — e.g. Met Office, Met.no, OpenWeatherMap, AccuWeather).
2. Find its entity ID: **Developer Tools → States**, filter on `weather.` — e.g. `weather.met_office_fleet`, `weather.forecast_home`, `weather.openweathermap`.
3. **Requirement:** the entity must support **hourly** forecasts. Most do. You can confirm in **Developer Tools → Actions**, call `weather.get_forecasts` with `type: hourly` against your entity and check it returns a list of hours, each with a `precipitation_probability` field.

> If you have more than one weather integration, pick the one most accurate for your location. In the UK the Met Office entity is usually the best local choice.

---

##### 5b. Which parameter is used for what

The feature combines **two independent signals**. Understanding which field each uses makes the provider choice clear:

| Check | Window | Data source | Field used | Cancels when |
|-------|--------|-------------|-----------|--------------|
| **Actual rain** | previous **12 h** | the weather entity's **`state`** (its current *condition*) | condition is one of `rainy`, `pouring`, `lightning-rainy`, `snowy-rainy`, `hail` | it entered a rain condition in the last 12 h |
| **Forecast rain** | next **24 h** | `weather.get_forecasts` (**`type: hourly`**) | each hour's **`precipitation_probability`** (%) | any hour ≥ `input_number.garden_rain_threshold` |

- **Actual** is recorded continuously: whenever the condition turns to rain, the `Garden Rain Recorder` automation stamps `input_datetime.garden_last_rain`. "Rained recently" is then just *now − last_rain < 12 h*.
- **Forecast** is fetched on demand 30 minutes before each run, and the highest probability across the next 24 hourly entries is compared to your threshold %.

If **either** signal says rain, the run is cancelled.

> **Note on `precipitation` (mm) vs `precipitation_probability` (%):** the automations decide on **probability**, because many providers (Met Office included) often report `0 mm` even when the chance of rain is moderate. If you would rather gate on forecast rainfall *amount*, see the comment block in `automations.yaml` — but probability is recommended.

---

##### 5c. Create the three automations

Open [`releases/v0.4.0/automations.yaml`](releases/v0.4.0/automations.yaml). It contains three automations:

| Automation | Fires | Does |
|-----------|-------|------|
| `Garden Rain Recorder` | weather condition → rain | stamps `input_datetime.garden_last_rain` (the **actual**-rain source) |
| `Garden Rain Auto-Cancel Check` | 30 min **before** the start time, only when a run is due | evaluates actual + forecast; if either says rain, sets `garden_rain_cancel` and notifies |
| `Garden Rain Cancel Daily Reset` | 30 min **after** the start time | clears `garden_rain_cancel` and dismisses the notification for a clean next day |

**To add them via the UI:** Settings → Automations & Scenes → Automations → **Create automation** → ⋮ menu → **Edit in YAML**, then paste one automation block at a time (each block starts at a `- alias:` line). Repeat for all three.

**⚠️ Set your weather entity — TWO places.** In `automations.yaml`, replace `weather.YOUR_FORECAST_ENTITY` with the entity from Step 5a in the two spots marked `# << SET WEATHER ENTITY`:

1. In **`Garden Rain Recorder`** → `triggers:` → `entity_id:` (this is the **actual**-rain watcher)
2. In **`Garden Rain Auto-Cancel Check`** → the `weather.get_forecasts` action → `target: entity_id:` (this is the **forecast** source)

That's it — the probability scan inside the check is provider-agnostic and needs no editing. Nothing else references the weather entity.

> **Claude-assisted shortcut:** instead of editing YAML, say *"Add automatic rain cancel to my Garden Watering Scheduler — discover my weather entity and wire up the rain automations."* Claude finds your `weather.*` entity, confirms it has hourly forecasts, and creates all three automations for you.

---

##### 5d. Verify

1. With a watering day and start time set and the schedule armed, call `Garden Rain Auto-Cancel Check` from Settings → Automations (⋮ → **Run**) — running it bypasses the time conditions so you can test the logic immediately.
2. If rain is currently recorded or forecast above your threshold, `input_boolean.garden_rain_cancel` flips **on**, the card shows the **Rain skip** badge, and a 🌧 notification appears in the HA sidebar bell explaining the reason.
3. Press **Go** on the card (or toggle the 🌧 button) to clear it and water anyway.

> Tip: lower `input_number.garden_rain_threshold` temporarily to force a cancel and confirm the wiring, then set it back.

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
| Rain check never cancels | Weather entity not set, or has no hourly forecast | Confirm both `# << SET WEATHER ENTITY` spots use a real `weather.*` entity, and that `weather.get_forecasts` with `type: hourly` returns data for it |
| Rain check cancels every day | `garden_last_rain` left at a recent value, or threshold too low | Set `input_datetime.garden_last_rain` to an old date (e.g. `2020-01-01`); raise `input_number.garden_rain_threshold` |
| Rain skip never clears | Reset automation missing or start time changed mid-day | Confirm `Garden Rain Cancel Daily Reset` exists; it clears 30 min after the start time |
| "Rained recently" never triggers | Recorder automation not firing | Check `Garden Rain Recorder`'s trigger entity matches your weather entity; it only stamps when the condition actually turns to rain |
