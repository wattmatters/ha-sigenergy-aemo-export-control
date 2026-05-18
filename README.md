# SIGEN – AEMO Critical Event Export Control (Native Sensors)

A Home Assistant automation for **SIGEN inverter owners in the Australian NEM** that automatically maximises battery export to the grid during AEMO critical price events, using native HACS integrations — no paid services or external APIs required.

---

## What it does

- **Detects AEMO critical price events** in real time using the AEMO NEM Web HACS integration
- **Pre-alerts** based on the 5-minute forecast price before an event officially starts
- **Automatically switches your SIGEN battery to maximum export mode** when a critical event is detected, subject to configurable time windows and a minimum battery reserve
- **Monitors every minute** during an active event to handle edge cases (battery depletion, Remote EMS turning off, export rate dropping)
- **Restores your normal operating mode** when the event ends, with context-aware handling for solar charging windows and peak export periods
- **Sends persistent notifications** at every state change so you always know what's happening

### Logic overview

```
Critical event detected?
  └─ Sufficient battery (>5 kWh)?
       ├─ Already exporting at max during peak window? → No action needed
       ├─ Solar/charging window (e.g. 11am–2pm) AND EMS on? → Override to export
       ├─ Remote EMS off? → Enable EMS, set export mode
       ├─ EMS on but export rate low? → Increase to maximum
       └─ Battery too low? → Notify only, don't export

Event ends?
  ├─ During solar window → Restore charging mode
  ├─ During peak window → Stay in export (no change needed)
  └─ Otherwise → Return to Maximum Self-consumption
```

---

## Requirements

### Hardware
- **SIGEN inverter** with a Home Assistant integration (HACS or custom) that exposes:
  - `switch.sigen_0_plant_remote_ems` — Remote EMS toggle
  - `number.sigen_0_plant_grid_max_export_limit` — Grid export limit (kW)
  - `number.sigen_0_plant_max_charging_limit` — Max charge rate (kW)
  - An operating mode `select` entity (e.g. Self-consumption / Command Charging / Command Discharging)
  - `sensor.sigen_0_inverter_1_available_battery_discharge_energy` — Available discharge energy (kWh)

### HACS Integrations
- [**AEMO NEM Web**](https://github.com/pvandenh/aemo_nemweb) — install via HACS, providing:
  - `sensor.aemo_nemweb_<REGION>_realtime_price` ($/MWh)
  - `sensor.aemo_nemweb_<REGION>_5min_forecast` ($/MWh)
  - Replace `nsw1` in the YAML with your region: `vic1`, `sa1`, `qld1`, etc.

### Helper Entities
Create these in **Settings → Devices & Services → Helpers** before importing the automation:

| Entity | Type | Suggested default |
|---|---|---|
| `input_number.aemo_critical_threshold` | Number | 3.0 (= $3,000/MWh) |
| `input_number.aemo_pre_alert_threshold` | Number | 2.0 (= $2,000/MWh) |
| `input_number.aemo_nsw_spot_price` | Number | 0 (updated automatically) |
| `input_boolean.aemo_critical_event` | Toggle | — |
| `input_boolean.aemo_critical_override_active` | Toggle | — |
| `input_boolean.aemo_pre_alert` | Toggle | — |
| `input_boolean.battery_export_enabled` | Toggle | — |
| `input_number.battery_charge_suggested_rate` | Number | your normal charge rate (kW) |

---

## Installation

1. **Create all helper entities** listed above.

2. **Copy `sigen_aemo_critical_event_export_control.yaml`** to your Home Assistant `config/automations/` directory, or paste its contents into the **Automation editor → Edit in YAML** view.

3. **Find and replace all `← REPLACE` placeholders** in the YAML:

   - **AEMO region**: change `nsw1` to your NEM region (`vic1`, `sa1`, `qld1`, `tas1`)
   - **SIGEN device_id**: open the SIGEN device page in HA, copy the device ID from the URL (`/config/devices/device/<id>`)
   - **SIGEN entity IDs**: use Developer Tools → States to find the exact IDs for your Remote EMS switch and operating mode select entity
   - **Time windows**: adjust the `after`/`before` times in the `critical_event_start` and `critical_event_end` branches to match your tariff schedule and solar generation window
   - **Export/charge limits**: change the `value: 5` entries to match your inverter's actual maximum export capacity and your preferred settings

4. **Reload automations** (Developer Tools → YAML → Reload Automations), or restart Home Assistant.

---

## Configuration reference

| YAML placeholder | What to replace it with |
|---|---|
| `sensor.aemo_nemweb_nsw1_*` | Your region's AEMO sensor prefix |
| `YOUR_SIGEN_DEVICE_ID` | Device ID from your HA SIGEN device page |
| `YOUR_SIGEN_OPERATING_MODE_SELECT_ENTITY_ID` | Entity ID of the SIGEN operating mode select |
| `YOUR_SIGEN_REMOTE_EMS_SWITCH_ENTITY_ID` | Entity ID of `switch.sigen_0_plant_remote_ems` |
| `value: 5` (export limit) | Your inverter's maximum grid export in kW |
| Time windows (`17:59:50`–`20:00:00`) | Your peak export / VPP window |
| Time windows (`10:59:50`–`14:00:00`) | Your solar/cheap-rate charging window |
| `above: 5` / `below: 5` (battery reserve) | Your desired minimum battery reserve in kWh |

---

## How thresholds work

Prices in the AEMO NEM Web integration are in **$/MWh**. The automation multiplies by 1,000 to convert to **$/MWh display values** (so `3.0` in `input_number.aemo_critical_threshold` = $3,000/MWh). Adjust these to suit your VPP agreement or personal threshold for when exporting makes financial sense.

The **pre-alert** triggers on the 5-minute forecast, giving you advance warning before the realtime price crosses the critical threshold. It doesn't change inverter behaviour — it just sets a flag and sends a notification so you can keep an eye on battery SoC.

---

## Notifications

The automation sends persistent notifications to your HA dashboard for every state change:

| Icon | Meaning |
|---|---|
| ✅ | Event ended or no action needed |
| ⚡ | Override active, exporting to grid |
| ⚠️ | Pre-alert or insufficient battery |
| 🛑 | Export stopped due to low battery |

---

## Known limitations / notes

- The automation is written for **SIGEN inverters** and uses device-action calls (by `device_id`) for the operating mode select. These cannot be replaced with simple entity service calls — you must supply your own `device_id`.
- Time windows are hardcoded. If your tariff windows change seasonally, update them accordingly or consider converting to `input_datetime` helpers.
- The automation assumes a **5 kW maximum export limit**. Adjust if your inverter or DNSP export limit differs.
- Tested on **Home Assistant 2024.x** with a SIGEN hybrid inverter in the NSW NEM region.

---

## Contributing

Issues and PRs welcome. If you adapt this for a different inverter brand or NEM region, please open a PR or issue — it would be great to document other configurations here.

---

## License

MIT — do whatever you like, no warranty implied.
