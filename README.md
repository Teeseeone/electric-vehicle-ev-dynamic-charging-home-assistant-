# Dynamic EV Charging Automation for Home Assistant — Tesla Fleet Fork

> **Tesla Fleet focused fork of the original Dynamic EV Charging Automation by [EDV11](https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-).**
>
> This fork keeps the original blueprint behavior while adding direct Tesla Fleet `switch` support and independent charging-current ramp-up / ramp-down controls.

## Latest version: v2.3

### What this fork adds

#### Direct Tesla Fleet start/stop support

The original blueprint expected the charging start/stop control to be a Home Assistant `select` entity.

This fork supports both:

- `select` entities
- `switch` entities

For a `switch` entity:

- `switch.turn_on` starts charging
- `switch.turn_off` stops charging

This makes it possible to use the Tesla Fleet charging switch directly, for example:

```text
switch.model_y_charge
```

Existing chargers using the original `select` logic remain supported.

#### Separate ramp-up and ramp-down logic

Charging current can now increase and decrease using different step sizes and delays.

New blueprint settings:

- **Ramp-up step size (A)**
- **Ramp-up delay (seconds)**
- **Ramp-down step size (A)**
- **Ramp-down delay (seconds)**

Current defaults:

| Setting | Default |
| --- | ---: |
| Ramp-up step | 1 A |
| Ramp-up delay | 10 s |
| Ramp-down step | 2 A |
| Ramp-down delay | 10 s |

Example behavior:

```text
Target increases from 10 A to 16 A:
10 → 11 → 12 → 13 → 14 → 15 → 16
     10s   10s   10s   10s   10s   10s

Target decreases from 16 A to 8 A:
16 → 14 → 12 → 10 → 8
      10s   10s   10s   10s
```

This allows faster load shedding through a larger downward step while keeping charging increases more gradual.

#### Improved minimum-current handling

When the calculated charging target falls below the configured minimum charging current, the blueprint now relies on the configured start/stop control instead of trying to force the charging-current entity below its valid minimum.

A calculated target of `0 A` is handled by stopping charging.

---

## Installation

### Stable v2.3 import

Use the released v2.3 blueprint:

<a href="https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml" target="_blank" rel="noreferrer noopener"><img src="https://my.home-assistant.io/badges/blueprint_import.svg" alt="Import Dynamic EV Charging Automation v2.3 into Home Assistant" /></a>

Or paste this URL manually into Home Assistant:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/v2.3/Dynamic-EV-Charging-Automation.yaml
```

In Home Assistant:

1. Go to **Settings → Automations & Scenes → Blueprints**.
2. Select **Import Blueprint**.
3. Paste the URL above.
4. Select **Preview Blueprint**.
5. Select **Import Blueprint**.
6. Create a new automation from **Dynamic EV Charging Automation v2.3**.

### Development / main branch

If you intentionally want the newest changes from this fork instead of the released version:

```text
https://raw.githubusercontent.com/Teeseeone/electric-vehicle-ev-dynamic-charging-home-assistant-/main/Dynamic-EV-Charging-Automation.yaml
```

For normal use, the tagged release URL is recommended because it stays pinned to the exact v2.3 code.

---

## Tesla Fleet example

A typical Tesla Fleet setup can use:

```text
EV Charging Toggle (Start/Stop):
switch.model_y_charge

Charging Current Control:
number.model_y_charge_current
```

Entity names vary between Home Assistant installations, so select the corresponding Tesla Fleet entities from your own system.

The blueprint also requires a charging-status sensor and a whole-house power sensor.

---

## Requirements

The blueprint expects the following Home Assistant entities or values:

- **Total household power sensor**
  - `sensor` domain
  - power device class
  - measured in watts
- **Charging-current control**
  - `number` entity
  - used to set EV charging amperage
- **Charging start/stop control**
  - `select` or `switch`
- **Charging-status sensor**
  - `sensor` entity
- **Optional car/person device tracker**
  - `device_tracker`
- Grid voltage
- Maximum grid power for day and night
- Minimum and maximum charging amperage

---

## Configuration

### Adjustment interval

Choose how often the automation recalculates available charging power:

- 1 minute
- 3 minutes
- 5 minutes

A shorter interval reacts more quickly to changing household consumption.

### Location tracker

The location tracker is optional.

Leave it empty if you do not want location-based checks, for example when:

- the charger is shared
- multiple EVs use the same charger
- you want charging regulation to depend only on charging state

### Charging-status sensor

The blueprint condition currently recognizes these charging states:

```text
charging
charged
plugged_in
Charging
CHARGING
```

The internal charger-consumption calculation normalizes the sensor state to lowercase and treats these states as consuming charging power:

```text
charging
plugged_in
waiting_for_authorization
waiting
```

If your Tesla Fleet charging-status sensor uses different state values, verify them in **Developer Tools → States** before relying on the automation.

### Start/stop control

The blueprint supports two control types.

For a `switch`:

```text
ON  → start charging
OFF → stop charging
```

For a `select`, the existing upstream behavior is preserved and the blueprint sends:

```text
Start charging
Stop charging
```

### Charging-current ramping

Ramp-up and ramp-down are configured independently.

For example:

```text
Ramp-up step:      1 A
Ramp-up delay:    10 s

Ramp-down step:    2 A
Ramp-down delay:  10 s
```

A larger ramp-down step can reduce EV load more quickly when household consumption rises.

### Grid limits

The blueprint supports separate maximum grid-power limits for:

- daytime
- nighttime

The active limit is selected using the configured days and start/end hours.

### Minimum charging current

If available power would require charging below the configured minimum current, the target becomes `0 A` and the blueprint stops charging through the start/stop control.

When enough power becomes available again, charging is started and the current ramps toward the newly calculated target.

---

## How the calculation works

The blueprint reads:

- total household consumption
- current charging-current setting
- configured grid-power limit
- configured voltage

When the charging-status sensor indicates an active charging state, estimated charger consumption is calculated as:

```text
charging current × voltage
```

Available charging power is then calculated from the active day/night grid limit while accounting for the charging load already included in the household power reading.

The resulting target current is:

- capped at **Maximum amps**
- changed to `0 A` if below **Minimum amps**
- otherwise rounded down to a whole amp

The current then ramps toward that target using the independently configured up/down step sizes and delays.

---

## Compatibility

This fork is designed for Tesla Fleet but remains compatible with the original blueprint control model.

Required capabilities:

- ✅ Dynamic amperage control through a Home Assistant `number` entity
- ✅ Start/stop through either a `switch` or compatible `select`
- ✅ Charging status through a `sensor`
- ✅ Whole-house power measurement

### Tesla Fleet

The main fork-specific addition is support for Tesla Fleet charging control exposed as a `switch`, such as:

```text
switch.model_y_charge
```

This removes the need for a template-select or input-select helper just to translate start/stop commands.

---

## Version history

### v2.3 — Tesla Fleet switch support + asymmetric ramping

- Added `switch` support to **EV Charging Toggle (Start/Stop)**
- Added direct Tesla Fleet charging start/stop support
- Preserved existing `select` behavior
- Added separate ramp-up step size
- Added separate ramp-up delay
- Added separate ramp-down step size
- Added separate ramp-down delay
- Improved handling when the calculated target is below minimum charging current

### v2.2 — Optional location tracker

Inherited from the upstream project:

- Car/person location tracker became optional
- Leaving it empty disables location-based checks
- Better suited to shared chargers and multi-EV households

### v2.1 — Charging-status capitalization support

Inherited from the upstream project:

- Added support for `Charging` from ESPHome Tesla BLE
- Added support for multiple capitalization variants
- Uses lowercase normalization in the charger-consumption calculation

### v2.0 — Smart start/stop control

Inherited from the upstream project:

- Added start/stop charger control
- Avoided relying on unsupported `0 A` current settings
- Updated available-power calculation
- Added gradual amperage adjustment

---

## Notes and limitations

- This automation controls real electrical load. Start with conservative limits and verify its behavior in Home Assistant before relying on it unattended.
- The blueprint estimates EV charging power from configured current × voltage. It does not directly measure charger power.
- Charging-state values vary between integrations. Confirm that your selected sensor reports a state recognized by the blueprint.
- The blueprint uses a single configured voltage value for its power/current calculation.

---

## Credits

This repository is a fork of:

**Dynamic EV Charging Automation** by **EDV11**

https://github.com/EDV11/electric-vehicle-ev-dynamic-charging-home-assistant-

The original project introduced the core dynamic charging, day/night limits, location tracking, charging-state handling, and start/stop logic.

This fork adds Tesla Fleet `switch` support, independent ramp-up/ramp-down controls, and the minimum-current handling changes described above.

---

## License

Please refer to the repository license and the upstream project's licensing terms.
