# Dynamic EV Charging Automation for Home Assistant — Tesla Fleet Fork

> **Tesla Fleet focused fork of the original Dynamic EV Charging Automation by [EDV11](https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-).**
>
> This fork keeps the original dynamic load-balancing concept while adding direct Tesla Fleet control and charging logic intended to make EV charging calmer, safer and easier to tune.

## Versions

- **Stable release:** v2.3
- **Development branch:** v2.4 on `main`

v2.4 is currently intended for testing before a new release is created.

---

## What this fork adds

### Direct Tesla Fleet start/stop support

The original blueprint expected the charging start/stop control to be a Home Assistant `select` entity.

This fork supports both:

- `select`
- `switch`

For a `switch`:

- `switch.turn_on` starts charging
- `switch.turn_off` stops charging

This allows direct use of Tesla Fleet entities such as:

```text
switch.model_y_charge
```

Existing select-based chargers continue to use:

```text
Start charging
Stop charging
```

### Independent ramp-up and ramp-down

Charging current can increase and decrease using different step sizes and delays.

Current defaults:

| Setting | Default |
| --- | ---: |
| Ramp-up step | 1 A |
| Ramp-up delay | 10 s |
| Ramp-down step | 2 A |
| Ramp-down delay | 10 s |

This allows the EV to shed load faster than it adds load back.

---

# v2.4 development changes

## Electrical supply presets

v2.4 adds a **Charging Supply** selector so the available-power calculation matches the installation.

Options:

- Single-phase 230 V
- Three-phase 230 V
- Three-phase 400 V

The blueprint uses:

```text
Single phase:
Power ≈ Voltage × Current

Three phase:
Power ≈ √3 × line-to-line voltage × Current
```

Examples at 16 A:

```text
1-phase 230 V  ≈ 3.68 kW
3-phase 230 V  ≈ 6.37 kW
3-phase 400 V  ≈ 11.09 kW
```


## Tesla Fleet charging states

Current Home Assistant Tesla Fleet charging states include:

```text
starting
charging
stopped
complete
disconnected
no_power
```

v2.4 understands these states directly.

Behavior:

- `starting` / `charging` → dynamic current regulation
- `stopped` / `no_power` → eligible for automatic restart after the restart lockout
- `complete` → do not restart
- `disconnected` → do nothing

Generic states retained for non-Tesla chargers include:

```text
plugged_in
waiting
waiting_for_authorization
```

## Ramp-up deadband

A configurable ramp-up deadband prevents small target changes from causing unnecessary current adjustments.

Default:

```text
1 A
```

The deadband is intentionally biased toward stability:

- small **increases** can be ignored
- required **reductions are not blocked**

Example:

```text
Current = 14 A
Calculated target = 15 A
Deadband = 1 A

Result: HOLD at 14 A
```

## Ramp-up stability check

Before increasing current, v2.4 waits and then reads the **current household power again**.

Default:

```text
30 seconds
```

The charging increase only proceeds if the fresh calculation still shows sufficient spare power.

This avoids increasing charging current based on an old power snapshot while other household loads are changing.

## Restart lockout

When the charging-state sensor reports `stopped` or `no_power`, v2.4 requires that state to remain active for a minimum time before automatic restart.

Default:

```text
300 seconds / 5 minutes
```

After the lockout:

1. The blueprint waits for the configured ramp-up stability period.
2. Household power is read again.
3. Charging only restarts if there is still enough power.
4. The charging current is first set to the configured minimum.
5. Charging is started.
6. Current ramps upward from the safe minimum.

This is intended to reduce repeated stop/start cycling.

> **Note:** the lockout is based on the charging-state sensor. A manually stopped Tesla that remains in the `stopped` state may therefore be restarted by the automation after the configured delay if sufficient power is available. Disable the automation if you want charging to remain manually stopped.

## Minimum-current handling

If the calculated target is below **Minimum amps**, the target becomes 0 A and the blueprint uses the configured charging start/stop entity instead of trying to request an invalid low charging current.

## Starting charging current

v2.4 has a separate **Starting charging amps** setting.

Default:

```text
6 A
```

Before starting or restarting charging, the blueprint sets the charging-current entity to this preferred start value. For safety, the requested start current is clamped so it can never exceed the current calculated safe target or **Maximum amps**.

After charging starts, the normal ramp-up logic takes over.

## Optional debug logging

Enable **Debug logging** to write decision details to the Home Assistant Logbook.

Examples include:

- calculated house power
- active grid limit
- available charging power
- calculated target current
- restart lockout time
- ramp-up/ramp-down decisions
- deadband holds
- stability-check results
- charging-state holds

Debug logging is **off by default**.

---

# Installation

## Stable v2.3

For normal use of the current released version:

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/blueprint_import.svg" alt="Import Dynamic EV Charging Automation v2.3 into Home Assistant" /></a>

Manual URL:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml
```

## Test v2.4 from main

To test the current v2.4 development version:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/main/Dynamic-EV-Charging-Automation.yaml
```

In Home Assistant:

1. Go to **Settings → Automations & Scenes → Blueprints**.
2. Select **Import Blueprint**.
3. Paste the chosen URL.
4. Preview and import the blueprint.
5. Create an automation from the imported blueprint.

---

# Tesla Fleet example

Typical Tesla Fleet entities may look like:

```text
EV Charging Toggle (Start/Stop):
switch.model_y_charge

Charging Current Control:
number.model_y_charge_current
```

For **EV Charging Sensor**, select the Tesla Fleet charging-state sensor that reports values such as:

```text
starting
charging
stopped
complete
disconnected
no_power
```

Entity names vary between Home Assistant installations.

For faster and more reliable control, v2.4 can also use a **Local Charger Status Sensor**. With the Tesla Wall Connector integration, use the local status entity, for example:

```text
sensor.tesla_wall_connector_status
```

When configured and available, the local charger status is preferred over the slower Tesla Fleet charging-state sensor for control decisions.

The blueprint does **not** modify `number.model_y_charge_limit`. Tesla's configured battery charge limit remains under Tesla/Home Assistant control.

---

# Requirements

The blueprint expects:

- whole-house power sensor in watts
- optional actual charger-power sensor in watts
- optional local charger-status sensor
- charging-current `number` entity
- charging start/stop `switch` or compatible `select`
- charging-state `sensor`
- optional `device_tracker`
- correct electrical supply choice
- maximum grid draw
- minimum and maximum charging current

---

# How the power calculation works

The blueprint uses one **Maximum Grid Draw (W)** limit at all times. There are no separate day/night limits or time schedules in v2.4.

You can optionally select an **Actual Charger Power Sensor**. When configured, the blueprint uses that measured charger wattage when separating EV load from the total house load. For a Tesla Wall Connector, select the local power sensor that reports the charger's current watt draw.

If no charger-power sensor is selected, the blueprint falls back to estimating charger power from the charging-current setting and selected electrical supply.

The electrical supply presets are still used to convert available watts into a charging-current target.

The resulting target current is:

- capped at **Maximum amps**
- changed to 0 A if below **Minimum amps**
- otherwise rounded down to a whole amp

Ramp-down can happen immediately.

Ramp-up must pass the configured deadband and the fresh-power stability check.

---

# Recommended starting settings

A sensible starting point for Tesla Fleet:

```text
Adjustment interval:       1 minute

Ramp-up step:              1 A
Ramp-up delay:             10 s

Ramp-down step:            2 A
Ramp-down delay:           10 s

Ramp-up deadband:          1 A
Ramp-up stability time:    30 s

Minimum charging current:  1 A (requested minimum; EV/charger may enforce a higher physical minimum)
Starting charging current:  6 A
Maximum grid draw:          9000 W

Restart delay:             300 s / 5 min
Debug logging:             Off
```

For a Norwegian 230 V IT installation with three-phase EV charging, select:

```text
Three-phase 230 V
```

For a 400 V TN installation with three-phase charging, select:

```text
Three-phase 400 V
```

---

# Version history

## v2.4 — development

- Added 1-phase 230 V, 3-phase 230 V and 3-phase 400 V supply choices
- Removed Custom supply settings
- Replaced separate day/night grid limits and schedules with one Maximum Grid Draw setting
- Set the minimum charging-current default/range floor to 5 A
- Corrected three-phase power/current calculations
- Added Tesla Fleet charging-state handling
- Added ramp-up deadband
- Added fresh-power ramp-up stability check
- Added 5-minute default restart lockout
- Added safe minimum-current restart behavior
- Added configurable starting charging current (default 6 A), clamped to the current safe target
- Added optional measured charger-power input with electrical-supply fallback
- Added optional local charger-status input, preferred over slow vehicle telemetry when available
- Lowered the configurable requested minimum charging current to 1 A
- Added optional Home Assistant Logbook debug output
- Added a clear `START COMMAND SENT` debug entry; Tesla Fleet commands already wake the vehicle automatically when required
- Preserved v2.3 switch/select control and independent ramp settings

## v2.3

- Added direct `switch` support for charging start/stop
- Added Tesla Fleet start/stop compatibility
- Added separate ramp-up and ramp-down step sizes
- Added separate ramp-up and ramp-down delays
- Improved minimum-current handling

## v2.2 — upstream

- Made the location tracker optional

## v2.1 — upstream

- Added charging-state capitalization support, including ESPHome Tesla BLE `Charging`

## v2.0 — upstream

- Added start/stop charger control
- Updated available-power calculation
- Added gradual charging-current adjustment

---

# Credits

This repository is a fork of:

**Dynamic EV Charging Automation** by **EDV11**

https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-

The original project provides the core dynamic charging concept, day/night grid limits, optional location tracking and the original start/stop logic. This fork's v2.4 development branch simplifies the grid limit to one Maximum Grid Draw value.

This fork adds direct Tesla Fleet control and the additional charging stability logic documented above.

---

# License

Please refer to the repository license and the upstream project's licensing terms.
