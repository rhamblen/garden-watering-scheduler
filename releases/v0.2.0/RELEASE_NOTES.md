# v0.2.0 — Release Notes

**Released:** 2026-06-18

## What's new

Scheduling engine — the card now actually waters the garden.

- **`script.garden_watering_sequence`** — opens the upper lawn valve, waits for the configured duration, closes it, then opens the lower lawn valve, waits for its duration, closes it. Runs in `single` mode to prevent double-triggering.
- **`automation.garden_watering_schedule`** — fires at every :00 and :30 minute mark, checks the configured start time, the current day's toggle, the armed state, and the rain cancel flag, then calls the watering script if all conditions pass.

No card changes — the card UI from v0.1.0 is unchanged.

## HA additions required

Create these in HA (or Claude will create them when pointed at this repo):

### Script — `script.garden_watering_sequence`

```yaml
alias: Garden Watering Sequence
description: Upper lawn first for configured duration, then lower lawn.
icon: mdi:sprinkler
mode: single
sequence:
  - action: switch.turn_on
    target:
      entity_id: switch.tap_lhs_upper_lawn_blue
  - delay:
      minutes: "{{ states('input_number.garden_water_upper_duration') | int }}"
  - action: switch.turn_off
    target:
      entity_id: switch.tap_lhs_upper_lawn_blue
  - action: switch.turn_on
    target:
      entity_id: switch.tap_lhs_lower_lawn_green
  - delay:
      minutes: "{{ states('input_number.garden_water_lower_duration') | int }}"
  - action: switch.turn_off
    target:
      entity_id: switch.tap_lhs_lower_lawn_green
```

### Automation — `automation.garden_watering_schedule`

```yaml
alias: Garden Watering Schedule
description: Fires at every :00 and :30, checks conditions, then runs the watering sequence.
mode: single
triggers:
  - trigger: time_pattern
    minutes: "0"
  - trigger: time_pattern
    minutes: "30"
conditions:
  - condition: state
    entity_id: input_boolean.garden_schedule_armed
    state: "on"
  - condition: state
    entity_id: input_boolean.garden_rain_cancel
    state: "off"
  - condition: template
    value_template: "{{ now().strftime('%H:%M') == states('input_select.garden_water_start_time') }}"
  - condition: template
    value_template: >
      {% set dow = now().weekday() %}
      {% set days = [
        'input_boolean.garden_water_mon',
        'input_boolean.garden_water_tue',
        'input_boolean.garden_water_wed',
        'input_boolean.garden_water_thu',
        'input_boolean.garden_water_fri',
        'input_boolean.garden_water_sat',
        'input_boolean.garden_water_sun'
      ] %}
      {{ is_state(days[dow], 'on') }}
actions:
  - action: script.garden_watering_sequence
```

## What's coming next

- **v0.3.0** — active countdown per zone in the card, progress bar, Cancel this run button
- **v0.4.0** — automatic rain cancel via daily weather forecast check
