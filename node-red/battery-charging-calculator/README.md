# Battery Charging Calculator

A Node-RED flow **shared as an example** that calculates an optimal battery charging strategy during a cheap or free electricity rate period, using Solcast solar forecasts, current battery state, and household consumption.

This is posted as a reference to adapt to your own system — not a plug-and-play solution. Time windows, consumption estimates, and capacity values are all specific to the author's system and must be adjusted.

---

## What it does

Polls nine Home Assistant sensors every 60 seconds, joins them into a single object, and passes them to a JavaScript function that calculates a charging strategy. The result is written back to four HA helper entities consumed by the [Supplemental Grid Charging](../../supplemental-grid-charging/) automation.

### Charging modes

| Mode | Description |
|---|---|
| `BATTERY_FULL` | Battery at capacity — no action needed |
| `GRID_CHARGING_PV_BUSY` | PV fully consumed by loads or EV charger — charge from free grid |
| `GRID_SUPPLEMENTAL` | Solar insufficient for deficit — supplement with grid |
| `OPPORTUNISTIC` | Small surplus — top up opportunistically from grid |
| `SOLAR_ONLY` | Large surplus — solar will fill battery naturally |
| `EV_CHARGER_KEEPALIVE` | Battery full but EV active — keep Remote EMS on for free grid EV charging |
| `EV_CHARGER_GRID_SUPPLEMENTAL` | Battery needs charging and EV active — charge both from free grid |
| `PLANNED_*` | Planning phase (before free period) — forecasting what will happen |
| `AFTER_FREE_PERIOD` | Free period has ended |

### EV charger latch

When an EV is detected (by running state or measurable power), a latch timer holds the `evChargerActive` flag true for a configurable number of minutes after the EV stops. This prevents the system from dropping out of Remote EMS during brief pauses (e.g. cell balancing) mid-charge.

---

## Configuration

Open the **Calculate Charging Strategy** function node and edit the constants at the top:

| Constant | Default | Description |
|---|---|---|
| `DAILY_HOUSEHOLD_CONSUMPTION` | `30` | ← REPLACE: your estimated daily consumption in kWh |
| `FREE_PERIOD_START_HOUR` | `11` | ← REPLACE: cheap/free period start hour (24h) |
| `FREE_PERIOD_END_HOUR` | `14` | ← REPLACE: cheap/free period end hour (24h) |
| `MAX_CHARGING_RATE_KW` | `16.8` | ← REPLACE: your inverter's maximum charge rate in kW |
| `FREE_PERIOD_MIN_CHARGE_RATE` | `2.0` | Minimum charge rate during free period in kW |
| `CHARGE_RATE_DEADBAND_KW` | `0.5` | Prevents rate changes smaller than this value |
| `PV_SMOOTHING_FACTOR` | `0.3` | EMA smoothing on PV reading (0=none, 1=full) |
| `EV_LATCH_MINUTES` | `20` | ← REPLACE: minutes to hold EV active flag after charger stops |

---

## Input sensors

Replace all `YOUR_` entity IDs in the poll-state nodes with your own:

| Topic | Node name | Entity to use |
|---|---|---|
| `battery_soc` | Battery SoC % | Your battery SOC sensor (e.g. `sensor.sigen_0_inverter_1_battery_soc`) |
| `solar_remaining` | PV Forecast - Remaining Today | `sensor.solcast_pv_forecast_forecast_remaining_today` |
| `solar_this_hour` | PV Forecast - This Hour | `sensor.solcast_pv_forecast_forecast_this_hour` |
| `solar_next_hour` | PV Forecast - Next Hour | `sensor.solcast_pv_forecast_forecast_next_hour` |
| `household_power` | Current Household Consumption | Your total consumed power sensor (W) |
| `sunset_hour` | Sunset time | `sensor.sun_next_setting` |
| `pv_generation` | PV Output Smoothed | Your smoothed PV power sensor (W) — see [node-red README](../README.md) |
| `dc_charger_power` | DC EV Charger Output Power | Your DC charger output power sensor (W) — remove if no DC charger |
| `dc_charger_state` | DC EV Charger Running State | Your DC charger running state sensor — remove if no DC charger |
| `battery_rated_capacity` | Battery Rated Capacity | `sensor.sigen_0_inverter_1_rated_battery_capacity` or `input_number.battery_total_capacity` |

**Note on join node:** The join node is configured to wait for **10 inputs** before firing (9 sensors + battery rated capacity). If you remove the DC charger nodes, reduce this count accordingly — double-click the join node and adjust the **Count** field.

---

## Output helpers

The flow writes to these HA helpers (create them before deploying):

| Entity | Type | Consumer |
|---|---|---|
| `input_boolean.battery_supplemental_charging_required` | Toggle | Supplemental Grid Charging automation |
| `input_number.battery_charge_suggested_rate` | Number | Supplemental Grid Charging automation |
| `input_text.battery_charge_strategy` | Text | Informational / dashboard display |
| `input_text.battery_charge_urgency` | Text | Informational / dashboard display |

---

## Known limitations / notes

- The `DAILY_HOUSEHOLD_CONSUMPTION` constant is a fixed estimate. If your household consumption varies significantly by season, consider making this an `input_number` helper so it can be adjusted without editing the flow.
- The `MAX_CHARGING_RATE_KW` constant can alternatively be read from `sensor.sigen_0_inverter_1_rated_charging_power` if available — add a poll-state node for it and reference `msg.payload.max_charge_rate` in the function.
- If DC charger nodes are removed, update the join node count from 10 to 8.
- Tested on **Node-RED 4.1.10** with **node-red-contrib-home-assistant-websocket 0.80.3**, **Home Assistant OS 17.3 / Core 2026.5.2**, and a **Sigenergy 12 kW Single Phase Energy Controller** in the NSW NEM region.

---

## Contributing

Shared as a starting point. If you adapt it for a different tariff structure, solar forecast provider, or inverter, feel free to open a PR or issue.

---

## License

MIT — do whatever you like, no warranty implied.
