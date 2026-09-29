# Dynamic EV Charging Automation for Home Assistant

> **Tesla Fleet-focused fork of the original Dynamic EV Charging Automation by [EDV11](https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-).**

This blueprint dynamically adjusts EV charging current so the **total household power draw** stays below a configured limit.

It is designed to work especially well with **Tesla Fleet + Tesla Wall Connector + a fast whole-house power sensor such as Tibber Pulse**, while retaining compatibility with the original select-based charger control.

## Version status

- **Stable release:** v2.3
- **Release candidate:** v2.4 on `main`

v2.4 should be tested in Home Assistant before the release tag is created.

---

## Highlights in v2.4

### Tesla Fleet start/stop support

The charging start/stop input supports both:

- `switch` entities
- `select` entities

For a Tesla Fleet switch:

```text
switch.turn_on  → start charging
switch.turn_off → stop charging
```

Tesla Fleet commands already wake the vehicle automatically when required.

### Independent ramp-up and ramp-down

Charging current can increase and decrease using different step sizes and delays.

Default values:

| Setting | Default |
| --- | ---: |
| Ramp-up step | 1 A |
| Ramp-up step delay | 10 s |
| Ramp-down step | 2 A |
| Ramp-down step delay | 10 s |
| Ramp-up deadband | 1 A |
| Ramp-up stability time | 30 s |
| Restart lockout | 300 s / 5 min |

### One Maximum Grid Draw limit

v2.4 removes the old separate day/night limits and schedules.

You now configure one value:

```text
Maximum Grid Draw (W)
```

This represents the **maximum total household power draw**, including EV charging.

### Supply presets

Available fallback supply types:

- Single-phase 230 V
- Three-phase 230 V
- Three-phase 400 V

These are used when measured charger power/current are unavailable.

### Measured charger power and current

v2.4 can use local charger measurements for more accurate control.

When both are available:

```text
watts per amp = actual charger power / actual charger current
```

The measured W/A value is preferred over the theoretical supply formula.

If either sensor is unavailable or zero, the blueprint automatically falls back to the selected supply type.

### Local charger status

An optional **Local Charger Status Sensor** can be used instead of slower vehicle telemetry for charging-state decisions.

For Tesla Wall Connector:

```text
sensor.tesla_wall_connector_status
```

Typical local states include:

```text
charging
charging_reduced
ready
connected
waiting_car
negotiating
charging_finished
not_connected
error
```

### Configurable starting current

**Starting Charging Current** controls the current requested immediately before a start/restart command.

Default:

```text
6 A
```

The value is automatically clamped so it cannot exceed the calculated safe target or the configured maximum charging current.

### Minimum charging current

The requested minimum can be configured down to:

```text
1 A
```

If the calculated safe target falls below the configured minimum, charging is stopped instead.

> The EV or charger may enforce a higher physical minimum even if Home Assistant accepts a lower requested value.

### Restart protection

After charging has been stopped, the blueprint applies a restart lockout.

Default:

```text
300 seconds / 5 minutes
```

Before restarting, the blueprint waits for the configured stability period and checks household power again.

### Debug logging

Enable **Debug Logging** while testing.

Messages are written to the Home Assistant Logbook and can include:

- current whole-house power
- charger power source: measured or estimated
- W/A source: measured or theoretical
- available charging power
- calculated target current
- ramp-up / ramp-down decisions
- deadband holds
- restart lockout status
- start command status

Example:

```text
START COMMAND SENT – Tesla Fleet will wake vehicle automatically if required.
Requested start current=6 A (configured=6 A, safe target=14 A).
```

---

## Recommended Tesla setup

For a Tesla Fleet + Tesla Wall Connector installation, the recommended inputs are:

| Blueprint input | Recommended entity/source |
| --- | --- |
| EV Charging State Sensor | Tesla Fleet charging-state sensor |
| Charging Start/Stop Control | Tesla Fleet charge switch |
| Whole-House Power Sensor | Tibber Pulse or another fast total-power sensor |
| Actual Charger Power Sensor | Tesla Wall Connector total power / `Billader` |
| Actual Charger Current Sensor | Tesla Wall Connector `Vehicle current` |
| Local Charger Status Sensor | `sensor.tesla_wall_connector_status` |
| Charger Vehicle Connected Sensor | `binary_sensor.tesla_wall_connector_vehicle_connected` |
| Target EV Charge Cable Sensor | Tesla Fleet Model Y charge-cable binary sensor |
| Charging Current Control | Tesla Fleet charging-current number |
| Charging Supply | Match the electrical installation |

For a Norwegian 230 V IT installation with three-phase EV charging, select:

```text
Three-phase 230 V
```

For a 400 V TN installation with three-phase charging, select:

```text
Three-phase 400 V
```

---

## How the calculation works

The blueprint starts with the configured:

```text
Maximum Grid Draw
```

It reads the current whole-house load and determines how much of that load belongs to the EV.

When an actual charger-power sensor is available:

```text
non-EV load = whole-house power - actual charger power
```

Then:

```text
available EV power = Maximum Grid Draw - non-EV load
```

If measured charger power and measured charger current are both valid:

```text
real W/A = measured charger power / measured charger current
target current = available EV power / real W/A
```

Otherwise, the selected electrical supply preset is used as the W/A fallback.

The resulting target current is:

- capped at **Maximum Charging Current**
- changed to 0 A when below **Minimum Charging Current**
- otherwise rounded down to a whole amp

---

## Suggested starting settings

```text
Adjustment interval:        1 minute

Maximum grid draw:          9000 W

Minimum charging current:   1 A
Starting charging current:  6 A
Maximum charging current:   Set for your installation/car

Ramp-up step:               1 A
Ramp-up step delay:         10 s

Ramp-down step:             2 A
Ramp-down step delay:       10 s

Ramp-up deadband:           1 A
Ramp-up stability time:     30 s

Restart lockout:            300 s / 5 min
Debug logging:              On while testing
```

---

## Installation

### Stable v2.3

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/blueprint_import.svg" alt="Import Dynamic EV Charging Automation v2.3 into Home Assistant" /></a>

Manual URL:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml
```

### Test v2.4 release candidate

Use the current `main` branch:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/main/Dynamic-EV-Charging-Automation.yaml
```

In Home Assistant:

1. Go to **Settings → Automations & Scenes → Blueprints**.
2. Select **Import Blueprint**.
3. Paste the URL.
4. Preview the blueprint.
5. Import or override the existing blueprint.
6. Open the automation and verify all v2.4 inputs.

---

## Charging-state behavior

When a valid Local Charger Status Sensor is configured, it is preferred over the vehicle charging-state sensor.

Active/startable states currently handled by the blueprint include:

```text
charging
charging_reduced
starting
stopped
no_power
ready
connected
waiting_car
negotiating
plugged_in
waiting
waiting_for_authorization
```

States outside the supported list are left alone.

This means states such as a completed or disconnected charging session are not automatically restarted.

---

## Location tracker

The location tracker is optional.

- Empty → no location restriction
- Selected and `home` → charging control allowed
- Selected and not `home` → automation blocked

---

## Important notes

- **Maximum Grid Draw refers to total household load, not charger power.**
- Tesla Fleet is used for commands; local Wall Connector sensors are preferable for fast charger status and measured power/current.
- Very low requested currents may not correspond to a physically usable charging current on every EV/charger.
- The blueprint does not change the vehicle charge limit.
- For shared chargers, configure both **Charger Vehicle Connected Sensor** and **Target EV Charge Cable Sensor**. If either configured gate is not ON, the blueprint sends no EV charging commands.
- The target-EV cable sensor comes from vehicle telemetry and may update more slowly than local Wall Connector data, so this reduces shared-charger risk but cannot cryptographically identify the vehicle connected to the Wall Connector.
- Reading these Home Assistant sensor states does not itself send extra Tesla commands; Tesla Fleet commands are only sent when the blueprint changes charging current or start/stop state.
- Keep Debug Logging enabled during initial testing.

---

## v2.4 release checklist

Before tagging v2.4, verify:

- charging starts successfully
- starting current is applied correctly
- measured charger W/A is shown in debug logs when both sensors are configured
- whole-house load stays near/below Maximum Grid Draw
- ramp-up behaves as configured
- ramp-down reacts correctly to a sudden household load
- charging stops below the configured minimum
- restart lockout lasts the configured time
- completed/disconnected charging is not restarted
- optional location tracker works both filled and empty

---

## Version history

### v2.4 — release candidate

- Added 230 V single-phase, 230 V three-phase, and 400 V three-phase supply presets
- Simplified power limiting to one **Maximum Grid Draw** value
- Added direct Tesla Fleet switch start/stop support
- Added local charger-status support
- Added measured charger-power support
- Added measured charger-current support
- Added real measured W/A calibration with automatic supply-formula fallback
- Added configurable minimum, starting, and maximum charging currents
- Added independent ramp-up and ramp-down settings
- Added ramp-up deadband and stability checking
- Added a five-minute default restart lockout
- Fixed the optional location tracker when left empty
- Added detailed optional Logbook debugging
- Added explicit start-command logging and documented Tesla Fleet automatic wake behavior
- Added shared-charger safety gates using local vehicle-connected and target-EV charge-cable sensors

### v2.3

- Added direct `switch` support for charging start/stop
- Added Tesla Fleet start/stop compatibility
- Added separate ramp-up and ramp-down step sizes and delays
- Improved minimum-current handling

### v2.2 — upstream

- Made the location tracker optional

### v2.1 — upstream

- Added case-insensitive charging-state handling, including ESPHome Tesla BLE `Charging`

### v2.0 — upstream

- Added charger start/stop control
- Updated power-calculation logic
- Added gradual charging-current adjustment

---

## Credits

Based on the original **Dynamic EV Charging Automation** by **EDV11**:

https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-

The original project introduced the core dynamic-charging concept and the earlier start/stop, location, and power-limit logic.

This fork adds Tesla Fleet support, local charger measurements, measured W/A calibration, and the additional charging-stability controls described above.

## License

Please refer to this repository's license and the upstream project's licensing terms.
