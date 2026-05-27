# SIGEN – Control Peak Export to Grid (with Debug & Zero Hero)

A Home Assistant automation **shared as an example** for Sigenergy energy controller owners who want to automatically manage battery export to the grid during an evening peak feed-in bonus period.

This is posted as a reference to adapt to your own system — not as a plug-and-play solution. It was developed and tested on a specific system (see [Tested on](#known-limitations--notes)) and makes assumptions about time windows, battery capacity, helper entities, and operating modes that reflect that setup.

---

## What it does

Runs every minute between a configurable monitoring window (e.g. 2:30–10 PM) and manages four scenarios:

- **Debug snapshot** — at the start of the peak window, posts a persistent notification with a full state summary to help diagnose issues
- **Start export** — if the peak window is active, export is enabled, and Remote EMS is currently off, enables Remote EMS, sets Command Discharging mode, and calculates an appropriate export rate
- **Adjust rate** — if already exporting during the peak window, recalculates and updates the export rate every minute based on current SOC, time remaining, and the Zero Hero minimum
- **Stop export** — if export is disabled mid-window, turns off Remote EMS and resets the export limit
- **Cleanup** — outside the peak window, ensures Remote EMS is off (unless an AEMO critical override or grid bias offset automation owns it)

---

## Rate calculation logic

The export rate is recalculated every minute using:

```
SOC headroom = current SOC% − minimum target SOC%
SOC-limited rate = (SOC headroom / 100 × battery capacity kWh) / hours remaining
rate = max(Zero Hero minimum, suggested rate)
rate = min(rate, SOC-limited rate)
rate = min(rate, peak export maximum)
```

This ensures the battery reaches the minimum target SOC by the end of the export window, exports at the suggested rate where possible, never falls below the Zero Hero minimum, and never exceeds the configured maximum.

### Zero Hero protection

The "Zero Hero" concept applies to tariff plans that pay a daily bonus for keeping net grid import below a threshold during the peak period — rather than penalising zero export directly. The distinction is subtle but important:

- The retailer pays a bonus (e.g. $1/day) if your net import stays below a small allowance over the full peak period
- Because of metering bias (see the [Grid Bias Offset Export](../grid-bias-offset-export/) automation), the inverter may believe it is at net zero while the utility meter records a small import
- To account for this, a minimum export floor (e.g. 30 W average, expressed as a kW rate) is applied to ensure the utility meter sees a net export, keeping you safely below the import threshold and preserving the bonus

The `input_number.battery_minimum_export_rate_zero_hero` helper sets this floor rate. The value should be calibrated to your own metering bias — in this system 0.5 kW was used as a conservative floor to reliably stay below the retailer's import allowance.

Set this to `0` if your tariff has no such bonus or import threshold.

---

## Requirements

### Hardware — SIGEN inverter integration

See the [AEMO Critical Event automation README](../sigen-aemo-export-control/README.md) for full details on the two supported integrations (sigenergy2mqtt and Sigenergy Local Modbus). The same entities are required here:

| Function | Example entity (sigenergy2mqtt) | What to look for |
|---|---|---|
| Remote EMS toggle | `switch.sigen_0_plant_remote_ems` | Switch controlling remote EMS / grid export mode |
| Grid export limit | `number.sigen_0_plant_grid_max_export_limit` | Max grid export in kW |
| Operating mode | *(select entity)* | Select with Self-consumption / Command Charging / Command Discharging options |
| Remote EMS control mode | `select.sigen_0_plant_remote_ems_control_mode` | Used to detect if another automation owns the EMS |
| Battery SOC | `sensor.sigen_0_inverter_1_battery_soc` | State of charge in % |

### Helper Entities

Create these in **Settings → Devices & Services → Helpers**:

| Entity | Type | Notes |
|---|---|---|
| `input_boolean.battery_export_enabled` | Toggle | Master switch — set by another automation or manually |
| `input_boolean.aemo_critical_override_active` | Toggle | Shared with the AEMO Critical Event automation |
| `input_number.battery_peak_export_maximum_rate` | Number | Your inverter/DNSP maximum export limit in kW |
| `input_number.battery_minimum_export_rate_zero_hero` | Number | Minimum export rate during peak window (set 0 if not needed) |
| `input_number.battery_minimum_soc_end_of_export` | Number | Target minimum SOC% to reach by end of peak window |
| `input_number.battery_export_suggested_rate` | Number | *(Optional)* Target rate from another automation; see below |
| `input_text.battery_export_strategy` | Text | *(Optional)* Human-readable label for current export strategy |

### Optional integration points

**`input_number.battery_export_suggested_rate`** — if you have a separate automation that calculates an optimal export rate (e.g. based on forecast SOC, pricing, or VPP signals), write its output to this helper and this automation will use it as the target rate, subject to the Zero Hero minimum and SOC-limited ceiling. If you don't have such an automation, set this helper to your preferred fixed rate.

**`input_text.battery_export_strategy`** — used in notifications to show a human-readable reason when export is enabled or disabled (e.g. "Peak tariff export", "Low SOC"). Purely informational — the automation will work without it.

**`input_boolean.battery_export_enabled`** — this automation does not decide *whether* to export; it only manages *how*. A separate automation (or manual toggle) is expected to set this boolean based on your pricing, VPP schedule, or other logic.

---

## Installation

1. Create all helper entities listed above.

2. Copy `sigen_control_peak_export_to_grid.yaml` to your `config/automations/` directory, or paste into the Automation editor → Edit in YAML.

3. Find and replace all `← REPLACE` and `← ADJUST` markers:

| Placeholder | What to replace with |
|---|---|
| `YOUR_SIGEN_DEVICE_ID` | Device ID from your HA SIGEN device page |
| `YOUR_SIGEN_REMOTE_EMS_SWITCH_ENTITY_ID` | Entity ID of your Remote EMS switch |
| `YOUR_SIGEN_OPERATING_MODE_SELECT_ENTITY_ID` | Entity ID of your operating mode select |
| `sensor.sigen_0_plant_remote_ems_control_mode` | Your Remote EMS control mode select entity |
| Peak window times (`17:59:50`–`20:00:00`) | Your tariff's peak feed-in window |
| Monitoring window (`14:30:00`–`22:00:00`) | Adjust to suit your schedule |
| `20` in minutes_remaining calculation | Your peak window end hour (24h) |

4. Reload automations or restart Home Assistant.

---

## Known limitations / notes

- The rate calculation template is duplicated across all four branches. This is intentional (Home Assistant YAML automations don't support reusable macros) but means you must update it in every branch if you change any values.
- Battery capacity is read from `sensor.sigen_0_inverter_1_rated_battery_capacity` rather than being hardcoded, so it adapts automatically to your system. Replace the sensor name in the template if yours differs.
- The automation deliberately does not act during the cleanup phase if Remote EMS is in `Command Discharging` mode — this prevents it from interfering with the Grid Bias Offset automation if that is also in use.
- Tested on **Home Assistant OS 17.3 / Core 2026.5.2 / Supervisor 2026.05.0** with a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region, using the **sigenergy2mqtt** add-on integration.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, inverter, or NEM region, feel free to open a PR or issue to share what you changed.

---

## License

MIT — do whatever you like, no warranty implied.
