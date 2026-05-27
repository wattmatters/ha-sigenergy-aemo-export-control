# Node-RED Flows

This folder contains two Node-RED flows that act as the "brain" behind the battery charging and export automations. They run calculations every minute, make decisions based on solar forecasts, battery state, and household consumption, and write their outputs to Home Assistant helper entities — which the HA automations then act on.

---

## Flows

| Flow | Description |
|---|---|
| [Battery Charging Calculator](./battery-charging-calculator/) | Calculates whether and how fast to charge the battery during the cheap/free rate period |
| [Battery Export Calculator](./battery-export-calculator/) | Calculates whether and how fast to export during the evening peak feed-in period |

---

## How they fit together

```
Node-RED flows (every 60s)
  ├── Battery Charging Calculator
  │     reads: solar forecast, battery SoC, PV output, household power,
  │            EV charger state, sunset time, battery rated capacity
  │     writes: input_boolean.battery_supplemental_charging_required
  │             input_number.battery_charge_suggested_rate
  │             input_text.battery_charge_strategy
  │             input_text.battery_charge_urgency
  │
  └── Battery Export Calculator
        reads: solar forecast, battery SoC, available discharge energy,
               household power, overnight load flag, max export rate,
               battery rated capacity
        writes: input_boolean.battery_export_enabled
                input_number.battery_export_suggested_rate
                input_text.battery_export_strategy
                input_text.battery_export_urgency

HA Automations (every 60s)
  ├── Supplemental Grid Charging  ← reads battery_supplemental_charging_required
  │                                          battery_charge_suggested_rate
  └── Peak Export Control         ← reads battery_export_enabled
                                             battery_export_suggested_rate
```

The Node-RED flows decide the *strategy*; the HA automations execute it.

---

## Requirements

### Node-RED
- **Node-RED** 4.1.10 or later (installed as a Home Assistant add-on)
- **node-red-contrib-home-assistant-websocket** 0.80.3 or later

Install Node-RED from the Home Assistant add-on store. The HA websocket node package is installed via **Manage Palette** inside Node-RED.

### Solcast Integration
Both flows use [Solcast](https://toolkit.solcast.com.au/) solar forecast data via the [Solcast PV Forecast](https://github.com/iamfotx/ha-solcast-solar) HACS integration. The following sensors are required:

| Sensor | Description |
|---|---|
| `sensor.solcast_pv_forecast_forecast_remaining_today` | kWh remaining today |
| `sensor.solcast_pv_forecast_forecast_this_hour` | kWh forecast this hour |
| `sensor.solcast_pv_forecast_forecast_next_hour` | kWh forecast next hour |

### PV Power Smoothed Sensor
The charging calculator uses a low-pass filtered version of the PV output to reduce noise. Create this as a filter helper or via `configuration.yaml`:

**Option A — configuration.yaml:**
```yaml
sensor:
  - platform: filter
    name: "PV Power Smoothed"
    entity_id: YOUR_PV_POWER_SENSOR   # ← REPLACE with your raw PV power sensor
    filters:
      - filter: lowpass
        time_constant: 20
        precision: 0
```

**Option B — Helpers UI:**
1. Go to **Settings → Devices & Services → Helpers → Create Helper**
2. Choose **Filter**
3. Set input entity to your raw PV power sensor
4. Set filter to **Low-pass** with time constant `20` and precision `0`

The sigenergy2mqtt integration exposes `sensor.sigen_0_plant_pv_power` as the raw PV power sensor. Sigenergy Local Modbus users should check Developer Tools → States for the equivalent.

### Overnight Load Prediction
The export calculator reads `input_boolean.aircon_expected_overnight` to adjust the overnight battery reserve. This boolean is set by a separate automation — see the [ha-overnight-load-prediction](https://github.com/wattmatters/ha-overnight-load-prediction) repository for an example implementation. If you don't have an equivalent, create the helper and set it manually each evening, or remove the aircon reserve logic from the export calculator function.

---

## Importing the flows

1. Open Node-RED (via the Home Assistant add-on UI)
2. Click the **hamburger menu** (top right) → **Import**
3. Paste the contents of the JSON file, or click **select a file to import** and upload it
4. Click **Import**
5. The flow will appear as a new tab — review and update all entity IDs before deploying

---

## After importing

**You must replace the HA server node** before the flow will work:
1. Double-click any poll-state or api-call-service node
2. Click the pencil icon next to the server field
3. Select your existing Home Assistant server connection (or create one)
4. Click **Update** — this updates all nodes in the flow that share the same server node

Then replace all `YOUR_` entity ID placeholders — see each flow's individual README for the full list.
