# 🔌 Smart charging for Hyundai Ioniq Electric (28 kWh) — a beginner's guide

This Home Assistant package turns an ordinary smart plug into a **smart charger**:

- calculates how much energy and time you need for a top-up,
- **"Quick Charge"** = charge now to 80 %,
- **"Charge to 100 % by departure"** = 80 % now, the rest to 100 % timed exactly to your departure (battery-friendly),
- **"Manual mode"** = dumb on/off; HA doesn't interfere, it only shows the smart numbers (SOC, ETA to 100 %, consumption),
- switches the plug off itself once the target is reached (in Quick / Scheduled modes),
- on a **manual interrupt** it saves the real reached SOC (it won't fall back to the start value),
- shows cost, monthly/yearly consumption and estimated range in km,
- notifies your phone and (optionally) speaks out loud through a speaker.

> ⏱️ Time needed: **15–20 minutes.** No programming required, just copy & replace text.

---

## What you need

1. **Home Assistant** (any common install – HAOS, Supervised…).
2. A **smart plug** you plug the car's charging cable into. It must be able to:
   - **turn on/off** from HA (a `switch.…` entity),
   - **measure instantaneous power in watts** (a `sensor.… power` entity).
   - Cheap proven options: Shelly Plug S, Tapo P110, Athom/Tasmota plug, etc.
3. The **"Studio Code Server"** or **"File editor"** add-on (to edit files). Find it in *Settings → Add-ons → Add-on Store*.

> ⚠️ **Mind the current!** From a normal socket the Ioniq draws ~9–10 A (about 2.1 kW) continuously for many hours. Use a good-quality plug and circuit. Cheap plug-ins can overheat.

---

## Step 1 – Find your entity names

In HA go to **Settings → Devices & Services → Entities** and find your plug. You need two:

| What to look for | Looks like | Write it down |
|---|---|---|
| Plug switch | `switch.charger` | _____________ |
| Power in W | `sensor.charger_power` | _____________ |

And your phone for notifications (you need the **Home Assistant Companion** app signed in):

| What to look for | Example |
|---|---|
| Phone notify service | `notify.mobile_app_YOURPHONE` |

> 💡 How to find the notify service: *Developer Tools → Actions* → type `notify.` in the search and browse the list.

(Optional, if you have a smart speaker for voice: `media_player.…` and `tts.…`.)

---

## Step 2 – Enable "packages"

1. Open **`configuration.yaml`** (via Studio Code Server / File editor).
2. Check for these two lines near the top. If missing, **add them at the very top**:

   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```

3. Create a **`packages`** folder next to `configuration.yaml`.

---

## Step 3 – Add the template and replace the names

1. Copy **`IONIQ_EV_charging_EN.yaml`** into the `packages/` folder.
2. Open it and do **"Find & Replace"** (in Studio Code Server: `Ctrl+H`). Replace these tokens with your entities from Step 1:

   | Find (token) | Replace with |
   |---|---|
   | `switch.PLUG` | your switch, e.g. `switch.charger` |
   | `sensor.PLUG_power` | your power sensor, e.g. `sensor.charger_power` |
   | `notify.MY_PHONE` | your notify service, e.g. `notify.mobile_app_pixel` |
   | `media_player.MY_SPEAKER` | (optional) your speaker |
   | `tts.MY_TTS` | (optional) your TTS service |

   > **No speaker/TTS?** Find every block starting with `- action: tts.speak` and delete it together with its indented lines. Phone notifications work without it.

3. **You don't need to change the car constants** – they're set for the Ioniq 28 kWh. If you charge at a different power (e.g. a wallbox), edit the `sensor: "EV config"` block:
   - `charger_kw` – charging power in kW (a normal socket ≈ `2.12`),
   - leave the rest (`battery_kwh`, range, losses).

4. **Save** and restart: *Settings → System → (top right) → Restart*.

> ✅ After the restart, check *Settings → System → Logs* for any red error about this file. If it's clean, you're good.

---

## Step 4 – Set your electricity price

In *Settings → Devices & Services → Helpers* find **"EV price per kWh"** and set your price. It's used for cost calculations.

---

## Step 5 – Add the dashboard

1. Open any dashboard → top right **pencil (Edit)** → **"+ Add view"**.
2. On the new view click **three dots → Edit in YAML** (Raw editor).
3. Paste this:

```yaml
title: EV charging
path: ev
icon: mdi:ev-station
type: sections
max_columns: 3
sections:
  - type: grid
    cards:
      - type: heading
        heading: Battery status
        icon: mdi:battery-charging-high
      - type: gauge
        entity: sensor.ev_estimated_soc
        name: Estimated SOC
        min: 0
        max: 100
        needle: true
        segments:
          - { from: 0, color: "#db4437" }
          - { from: 20, color: "#0f9d58" }
          - { from: 80, color: "#f4b400" }
          - { from: 100, color: "#0f9d58" }
      # current mode – shown only while charging
      - type: tile
        entity: input_select.ev_mode
        name: Current mode
        icon: mdi:ev-station
        visibility:
          - condition: state
            entity: binary_sensor.ev_charging
            state: "on"
      - type: entities
        entities:
          - entity: sensor.ev_range_km
            name: Range
            icon: mdi:map-marker-distance
          - entity: input_number.ev_soc_current
            name: Current SOC (enter)
      - type: tile
        entity: script.ev_quick_charge
        name: ⚡ Charge now to 80 %
        icon: mdi:flash
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: script.ev_quick_charge
      - type: entities
        entities:
          - entity: input_datetime.ev_ready_time
            name: Departure time
      - type: tile
        entity: script.ev_charge_100_by_departure
        name: 🕐 To 100 % by departure
        icon: mdi:calendar-clock
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: script.ev_charge_100_by_departure
      - type: tile
        entity: input_select.ev_mode
        name: 🔧 Manual charging
        icon: mdi:hand-back-right
        hide_state: true
        vertical: true
        tap_action:
          action: perform-action
          perform_action: input_select.select_option
          target:
            entity_id: input_select.ev_mode
          data:
            option: Manual
      - type: entities
        title: Detail
        entities:
          - entity: sensor.PLUG_power
            name: Current power
            icon: mdi:flash-outline
          - entity: sensor.ev_session_delivered_kwh
            name: Delivered
          - entity: sensor.ev_kwh_needed
            name: Target (needed)
          - entity: sensor.ev_session_cost
            name: Current cost
          - entity: sensor.ev_remaining_text
            name: Remaining
          - entity: sensor.ev_eta
            name: ETA (to target)
          - entity: sensor.ev_eta_100
            name: Done at 100 %
          - entity: binary_sensor.ev_charging
            name: Charging
  - type: grid
    cards:
      - type: heading
        heading: Plan & history
        icon: mdi:calendar-clock
      - type: entities
        title: Inputs
        entities:
          - entity: input_number.ev_soc_target
            name: Target
          - entity: input_datetime.ev_ready_time
            name: Ready by (departure)
          - entity: input_select.ev_mode
            name: Mode
          - entity: input_number.ev_price
            name: Price per kWh
      - type: entities
        title: Calculation
        entities:
          - entity: sensor.ev_planned_start
            name: Starts charging at
          - entity: sensor.ev_duration_text
            name: Will take
      - type: entities
        title: Summary
        entities:
          - entity: sensor.ev_monthly_kwh_display
            name: This month kWh
          - entity: sensor.ev_monthly_cost
            name: This month cost
          - entity: sensor.ev_yearly_cost
            name: This year cost
      - type: history-graph
        title: Charging power (24 h)
        hours_to_show: 24
        entities:
          - entity: sensor.PLUG_power
            name: Power W
```

> ❗ Here too, replace `sensor.PLUG_power` with your power sensor (in two places: Current power + the last graph).

4. Save. Done.

---

## How to use it

### A) Quick charge to 80 %
1. Move the **"Current SOC"** slider to what the car shows (e.g. 45 %).
2. Tap **"⚡ Charge now to 80 %"**.
3. Charging starts and stops itself at 80 %.

### B) Full charge by departure (battery-friendly)
1. Move the **"Current SOC"** slider to the real value.
2. Set the **departure time** next to it.
3. Tap **"🕐 To 100 % by departure"**.
   - The car charges to 80 % **immediately** (in case you have to leave early).
   - The rest to 100 % starts so it finishes **exactly at departure**.

> 🔋 **Why two phases?** A battery doesn't like sitting fully charged. This way it hits 100 % just before you drive.

### C) Manual charging (dumb on/off)
1. (Optional) move the **"Current SOC"** slider to the real value so the numbers match.
2. Tap **"Manual charging"** → the plug turns on **dumbly** and HA **does not interfere**.
3. While charging you see live SOC, range, **ETA to 100 %**, delivered kWh and cost.
4. **Turn it off manually** (plug tile / switch) → the **real reached SOC is saved** and the mode returns to "Off".

> 🔧 Manual is for full control (e.g. charge to exactly whatever you want, with no target) – the smart numbers keep running as a readout.

---

## Good to know

- **Always set the real "Current SOC"** before starting. The stop point is computed from it. Wrong value → inaccurate charge.
- The system **doesn't know** the real SOC in the car (the Ioniq doesn't send it over the cable). It works from delivered energy. Fine-tune accuracy after a few sessions (below).
- After reaching the target the plug **switches off itself** and the mode returns to "Off" (Quick / Scheduled modes).
- If you **interrupt charging manually** (e.g. a storm rolls in), the real reached SOC is saved – it won't drop back to the start value. Applies to all modes.

---

## Fine-tuning accuracy (after the first charge)

Charge e.g. from 50 to 80 and check what % the car actually reached:

- Reached **more** than 80 (overshoot) → slightly **raise** `eff_below_80` in `sensor: EV config` (e.g. 0.95 → 0.96).
- Reached **less** than 80 (undershoot) → slightly **lower** `eff_below_80` (e.g. 0.95 → 0.93).

Same for `eff_above_80` for the 80–100 % band. After editing, restart or *Developer Tools → YAML → Reload template entities*.

> The included values (`eff_below_80: 0.95`, `charger_kw: 2.12`) were measured on a real Ioniq via schuko charging – a good starting point.

---

## When something doesn't work

| Problem | Fix |
|---|---|
| Red error about the file in the logs after restart | Usually wrong indentation after deleting TTS lines, or an unreplaced token. Check it. |
| `sensor.ev_estimated_soc` is "unknown" | Templates not loaded yet – restart, or *Developer Tools → YAML → Reload template entities*. |
| A button does nothing | Check that `script.ev_quick_charge` and `script.ev_charge_100_by_departure` exist (*Settings → Automations & scenes → Scripts*). |
| Charges past the target | Your "Current SOC" was equal to or above the target → nothing to charge. Set the real (lower) value. |
| Power / current-power is empty | You didn't replace `sensor.PLUG_power` with your sensor in the dashboard. |
| "Manual" mode missing from the dropdown | You didn't reload after editing. Do *Developer Tools → YAML → Input select*, or restart. |
| "Current mode" not showing on the dashboard | By design – it only appears **while charging**. |

---

## Entities the package creates

**Controls:** `input_number.ev_soc_current`, `input_number.ev_soc_target`, `input_number.ev_price`, `input_datetime.ev_ready_time`, `input_select.ev_mode`, `input_button.ev_reset_session`, `input_boolean.ev_finish_to_100`

**Scripts:** `script.ev_quick_charge`, `script.ev_charge_100_by_departure`

**Sensors:** `sensor.ev_estimated_soc`, `sensor.ev_range_km`, `sensor.ev_kwh_needed`, `sensor.ev_duration_text`, `sensor.ev_planned_start`, `sensor.ev_session_delivered_kwh`, `sensor.ev_session_cost`, `sensor.ev_remaining_text`, `sensor.ev_eta`, `sensor.ev_eta_100`, `sensor.ev_monthly_kwh_display`, `sensor.ev_monthly_cost`, `sensor.ev_yearly_cost`, `binary_sensor.ev_charging`

**Modes** (`input_select.ev_mode`): Off, Charge now, Scheduled, **Manual**.

**Automations:** start now, start scheduled, stop on target (ignores Manual), 80/100 % alerts, failsafe, manual start, save SOC on interrupt.

---

## Changelog

- **v1.2 (2026-07-05)** – Fix: session baseline captured on plug-ON (not power > 50 W); a short mid-charge pause no longer resets the counter (previously caused overcharge). Tuned by measurement: efficiency below 80 % = 0.95, charger power = 2.12 kW.
- **v1.1 (2026-06)** – Added Manual mode + ETA-to-100 % sensor; save real SOC on manual interrupt; stop-on-target ignores Manual.
- **v1.0 (2026-05)** – Base: Quick Charge, two-phase charge to 100 % by departure, SOC/cost/range, monthly/yearly consumption, failsafe.

---

*Made for the Hyundai Ioniq Electric 28 kWh. Free to use and modify.*
