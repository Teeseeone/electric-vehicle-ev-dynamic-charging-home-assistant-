# Dynamic EV Charging Automation v3.0 - Tesla Fleet for Home Assistant

> **Tesla Fleet-focused fork of the original Dynamic EV Charging Automation by [EDV11](https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-).**

This blueprint dynamically adjusts EV charging current so the **total household power draw** stays below a configured limit.

It is designed to work especially well with **Tesla Fleet + Tesla Wall Connector + a fast whole-house power sensor such as Tibber Pulse**, while retaining compatibility with the original select-based charger control.

## Version status

- **Stable release:** v3.0 Tesla Fleet
- **Release tag:** `v3.0-Tesla-Fleet`
- **Previous stable release:** v2.3

v3.0 Tesla Fleet is the current stable release.

---

## Highlights in v3.0

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

v3.0 re-evaluates charging once per minute while idle. It runs in **single** mode so a scheduled one-minute trigger cannot cancel an active ramp. While ramping up, every increase step waits for the configured delay and then re-reads whole-house power plus charger power/current before deciding whether another increase is still safe.

The **Ramp-Up Deadband** is only used to decide whether a *new* upward ramp is worth starting. The configured value is the minimum upward difference required to start a ramp, so with a 1 A deadband an increase from 18 A to 19 A is allowed. Once a ramp has started, it continues toward the latest safe target and may use a smaller final step to land exactly on that target.

Default values:

| Setting | Default |
| --- | ---: |
| Ramp-up step | 1 A |
| Ramp-up step delay | 10 s |
| Ramp-down step | 2 A |
| Ramp-down step delay | 10 s |
| Ramp-up deadband | 1 A (used only to start a new ramp) |
| Ramp-up stability time | 30 s |
| Restart lockout | 300 s / 5 min |

### One Maximum Grid Draw limit

v3.0 removes the old separate day/night limits and schedules.

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

v3.0 can use local charger measurements for more accurate control.

When both are available:

```text
watts per amp = actual charger power / actual charger current
```

The measured W/A value is preferred over the theoretical supply formula.

Measured charger watts are only counted as active EV load while the charger state is
`charging`, `charging_reduced`, or `starting`. This prevents a stale charger-power
reading immediately after charging stops from artificially inflating the EV power budget budget.

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

Before restarting, the blueprint waits for the configured stability period and checks household power again. After charging starts, every ramp-up step also uses fresh house/charger readings before increasing current.

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

The blueprint labels entity inputs by source:

- **[CAR]** = entity belonging to the EV, normally from Tesla Fleet
- **[CHARGER]** = entity belonging to the local wall charger, such as Tesla Wall Connector
- **[HOUSE]** = whole-house power meter, such as Tibber Pulse
- **[CAR/PERSON]** = optional location tracker

For a Tesla Fleet + Tesla Wall Connector installation, use:

| Blueprint input | Use this source |
| --- | --- |
| **[CAR] EV Charging State Sensor** | Tesla Fleet charging-state sensor |
| **[CAR] Charging Start/Stop Control** | Tesla Fleet charge switch |
| **[HOUSE] Whole-House Power Sensor** | Tibber Pulse or another fast total-power sensor |
| **[CHARGER] Actual Power Sensor** | Tesla Wall Connector total power / `Billader` |
| **[CHARGER] Actual Current Sensor** | Tesla Wall Connector `Vehicle current` |
| **[CHARGER] Local Status Sensor** | `sensor.tesla_wall_connector_status` |
| **[CHARGER] Vehicle Connected Sensor** | `binary_sensor.tesla_wall_connector_vehicle_connected` |
| **[CAR] Target EV Charge Cable Sensor** | Tesla Fleet Model Y charge-cable binary sensor |
| **[CAR] Charging Current Control** | Tesla Fleet charging-current number |
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

Two useful values are exposed in debug logs:

```text
grid_headroom = Maximum Grid Draw - current whole-house power
EV_budget     = Maximum Grid Draw - current non-EV household load
```

`grid_headroom` shows how far the house is currently below or above the configured limit.
`EV_budget` shows how many watts the charger is allowed to use while respecting that same limit.
The EV budget is clamped so it can never exceed **Maximum Grid Draw**, even if house and charger sensors update at slightly different moments.

When an actual charger-power sensor is available:

```text
non-EV load = whole-house power - actual charger power
```

Then:

```text
EV power budget = Maximum Grid Draw - non-EV load
```

If measured charger power and measured charger current are both valid:

```text
real W/A = measured charger power / measured charger current
target current = EV power budget / real W/A
```

Otherwise, the selected electrical supply preset is used as the W/A fallback.

The resulting target current is:

- capped at **Maximum Charging Current**
- changed to 0 A when below **Minimum Charging Current**
- otherwise rounded down to a whole amp

During ramp-up, this calculation is repeated before **every** increase step. If the fresh target falls, the blueprint stops increasing and can reverse into the configured ramp-down behavior.

After the initial ramp-up stability delay, a fresh target below the current charging current now triggers an immediate ramp-down instead of waiting for the next one-minute evaluation. If the fresh target falls below the configured minimum, charging is stopped.

---

## Suggested starting settings

```text
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

### Stable v3.0 Tesla Fleet

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v3.0-Tesla-Fleet/Dynamic-EV-Charging-Automation.yaml" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/blueprint_import.svg" alt="Import Dynamic EV Charging Automation v3.0 Tesla Fleet into Home Assistant" /></a>

Manual URL:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v3.0-Tesla-Fleet/Dynamic-EV-Charging-Automation.yaml
```

GitHub release:

```text
https://github.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/releases/tag/v3.0-Tesla-Fleet
```

### Development build

Use `main` only when testing unreleased changes:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/main/Dynamic-EV-Charging-Automation.yaml
```

In Home Assistant:

1. Go to **Settings → Automations & Scenes → Blueprints**.
2. Select **Import Blueprint**.
3. Paste the URL.
4. Preview the blueprint.
5. Import or override the existing blueprint.
6. Open the automation and verify all v3.0 inputs.

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
- **CAR/Tesla Fleet entities are used for vehicle-specific commands. CHARGER/Wall Connector entities are used for fast local status and measured power/current.**
- Very low requested currents may not correspond to a physically usable charging current on every EV/charger.
- The blueprint does not change the vehicle charge limit.
- For shared chargers, configure both **Charger Vehicle Connected Sensor** and **Target EV Charge Cable Sensor**. If either configured gate is not ON, the blueprint sends no EV charging commands.
- The target-EV cable sensor comes from vehicle telemetry and may update more slowly than local Wall Connector data, so this reduces shared-charger risk but cannot cryptographically identify the vehicle connected to the Wall Connector.
- Reading these Home Assistant sensor states does not itself send extra Tesla commands; Tesla Fleet commands are only sent when the blueprint changes charging current or start/stop state.
- Keep Debug Logging enabled during initial testing.

---

## v3.0 validation checklist

The v3.0 Tesla Fleet release was validated against:

- charging starts successfully
- starting current is applied correctly
- measured charger W/A is shown in debug logs when both sensors are configured
- whole-house load stays near/below Maximum Grid Draw
- the fixed one-minute idle re-evaluation works as expected
- ramp-up behaves as configured without being interrupted by the one-minute trigger
- each ramp-up step re-checks fresh Tibber/house and charger measurements
- ramp-down reacts correctly to a sudden household load
- charging stops below the configured minimum
- restart lockout lasts the configured time
- completed/disconnected charging is not restarted
- optional location tracker works both filled and empty

---

## Version history

### v3.0 Tesla Fleet — stable

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
- Changed deadband behavior so it is the minimum difference required to start a new ramp; a 1 A difference starts when deadband is 1 A, and active ramps finish at the latest safe target
- Added a five-minute default restart lockout
- Fixed the optional location tracker when left empty
- Added detailed optional Logbook debugging, including both `grid_headroom` and `EV_budget`
- Added explicit start-command logging and documented Tesla Fleet automatic wake behavior
- Added shared-charger safety gates using local vehicle-connected and target-EV charge-cable sensors
- Labelled blueprint entity inputs as **[CAR]**, **[CHARGER]**, **[HOUSE]**, or **[CAR/PERSON]** so the intended source is obvious
- Removed the 1/3/5-minute interval selector and standardized idle re-evaluation to once per minute
- Changed automation execution from `restart` to `single` so scheduled triggers cannot cancel an active ramp
- Added fresh whole-house and charger re-checks before every ramp-up step; ramp-up halts or reverses if the safe target drops
- Fixed stale charger-power handling so measured charger watts are ignored when the local charger is no longer actively charging
- Clamped EV budget so sensor timing mismatches cannot make it exceed Maximum Grid Draw
- Added immediate post-stability ramp-down/stop when the fresh target falls below the current charging current

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
