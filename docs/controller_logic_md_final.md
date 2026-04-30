# Controller Logic

This document explains the core logic and design rationale of the Home Assistant EcoFlow STREAM PI Controller.

## 1. Tested setup / reference hardware

This controller was developed with the following reference setup:

```text
Battery system:   EcoFlow STREAM Ultra X / STREAM Ultra class device
EcoFlow firmware: v1.0.2.1
Home Assistant:   2026.4.x
Integration:      rabits / ha-ef-ble
ha-ef-ble:        v0.8.5 during development
Grid meter:       Home Assistant grid power sensor
PV sensor:        Home Assistant PV production power sensor
EV sensor:        optional Home Assistant EV charger power sensor
```

Grid power sign convention:

```text
positive value = grid import
negative value = grid export / feed-in
```

The package is written with an adapter layer, so other users can map their own Home Assistant entity IDs to the internal controller aliases.

## 2. Why this controller exists

EcoFlow STREAM hardware is capable, but for advanced Home Assistant automation there is currently no simple, official, fully bidirectional local control interface that behaves like a classic battery inverter API.

In an ideal control system, Home Assistant could send one signed power setpoint:

```text
+500 W = discharge battery
-500 W = charge battery
0 W    = idle
```

At the moment, EcoFlow STREAM control is exposed through separate entities and operating modes, for example:

```text
charging power limit
base-load / discharge power
energy strategy / operating mode
backup reserve
```

That means a Home Assistant controller has to coordinate multiple entities to emulate one bidirectional battery setpoint.

This project adds that missing abstraction layer.

It turns the available EcoFlow STREAM controls into one unified control model:

```text
unified_power > 0  => discharge battery
unified_power < 0  => charge battery
unified_power = 0  => idle / deadband
```

On top of that unified model, the package implements a PI controller for near-zero-grid operation.

The goal is not to replace EcoFlow firmware logic. The goal is to make the STREAM device usable as a controllable energy component inside Home Assistant, while respecting the practical limitations of the available EcoFlow control entities.

## 3. Important dependency: ha-ef-ble

This controller depends on the EcoFlow BLE integration for Home Assistant:

```text
rabits / ha-ef-ble
https://github.com/rabits/ha-ef-ble
```

Without this integration and the work behind it, this controller would not be possible in this form.

The BLE integration provides practical local access to EcoFlow STREAM entities such as:

```text
charging power limit
base-load / discharge power
energy strategy
backup reserve
battery state values
```

That local access is the key enabler for a responsive PI-based control loop.

## 4. Design goal

The controller tries to keep the grid power close to a configurable target value.

Typical target values:

```text
-20 W  => slight export bias
  0 W  => theoretical zero-grid target
+20 W  => slight import bias
```

A small bias is often more stable than targeting exactly `0 W`, because real systems have:

```text
sensor delay
BLE/cloud update delay
inverter response delay
actuator step sizes
measurement noise
Home Assistant automation timing jitter
```

Practical stability is more important than mathematically perfect zero-grid tracking.

## 5. Sign convention

Grid power:

```text
grid_power > 0  => grid import
grid_power < 0  => grid export / feed-in
```

Unified battery power:

```text
unified_power > 0  => battery discharge
unified_power < 0  => battery charge
unified_power = 0  => idle / deadband
```

EcoFlow STREAM exposes charge and discharge control through different actuator entities. The controller converts them into one unified value:

```text
current_unified_power = current_discharge_power - current_charge_power
```

Examples:

```text
discharge = 300 W, charge = 0 W     => current_unified_power = +300 W
charge = 500 W, discharge = 0 W     => current_unified_power = -500 W
charge = 0 W, discharge = 0 W       => current_unified_power = 0 W
```

## 6. Control error

The control error is calculated as:

```text
error = grid_power - target_grid
```

Example 1:

```text
grid_power  = +200 W
target_grid =  -20 W
error       = +220 W
```

The house imports more than desired. The controller should increase battery discharge.

Example 2:

```text
grid_power  = -300 W
target_grid =  -20 W
error       = -280 W
```

The house exports more than desired. If the charge gate allows charging, the controller should increase battery charging.

## 7. Incremental PI control

The controller uses an incremental PI approach:

```text
P = Kp * error
I = I + Ki * error * dt

target_power = current_unified_power + P + I
```

This means the controller does not calculate a completely new absolute output from scratch. Instead, it starts with the current EcoFlow output and applies a correction.

This approach is useful when the controlled device reacts with delay or when the output is updated in steps.

## 8. P term

The proportional term reacts to the current error.

```text
P = Kp * error
```

A higher `Kp` means faster reaction.

Too high `Kp` can cause oscillation.

Conservative beta start value:

```text
Kp = 0.20
```

## 9. I term

The integral term reacts to persistent error over time.

```text
I = I + Ki * error * dt
```

It helps remove permanent residual error that the P term alone may not eliminate.

Conservative beta start value:

```text
Ki = 0.002
```

The controller interval is usually:

```text
dt = 5 seconds
```

Example:

```text
error = 100 W
Ki    = 0.002
dt    = 5 s

I increase = 0.002 * 100 * 5 = 1 W
```

The I term grows slowly and is intentionally limited.

## 10. Deadband

The deadband prevents unnecessary actuator changes around the target.

Example:

```text
Target Grid = -20 W
Deadband    = 50 W
```

If the error is inside the deadband, the controller does not aggressively change the output.

Conservative beta start value:

```text
Deadband = 50 W
```

## 11. Output limits

The target power is limited by configured maximum values:

```text
Max Discharge = 800 W
Max Charge    = 1200 W
```

The controller clamps the calculated target power into this range:

```text
-max_charge <= target_power <= max_discharge
```

Example:

```text
calculated target = +950 W
max_discharge     = +800 W
final target      = +800 W
```

## 12. Step sizes

EcoFlow charge and discharge actuators may have different practical resolutions.

Default values:

```text
Discharge Step = 10 W
Charge Step    = 1 W
```

Positive target power is rounded to the discharge step.

Negative target power is rounded to the charge step.

## 13. Anti-windup

Anti-windup prevents the integral term from growing further when the actuator is already saturated.

Example:

```text
current_discharge = 800 W
max_discharge     = 800 W
error             > 0
```

The controller already discharges at maximum power, but the error would request even more discharge.

In this case, the I term is frozen.

Same logic applies to charging:

```text
current_charge = 1200 W
max_charge     = 1200 W
error          < 0
```

The controller already charges at maximum power, but the error would request even more charging.

The I term is frozen.

## 14. Integral decay and reset behavior

The I term is reduced or reset in several cases.

### Error changes direction

If the error changes direction while the integral still points into the old direction, the I term is reduced.

```text
I = I * 0.5
```

This helps the controller recover faster from overshoot.

### Error inside deadband

If the error is inside the deadband, the I term slowly decays.

```text
I = I * 0.90
```

This avoids carrying stale correction values for too long.

### EV charging mode

When EV charging mode takes control, the I term is reset.

```text
I = 0
```

### SOC protection

When SOC protection blocks discharge, the I term is reset.

```text
I = 0
```

### Charge gate blocked

If charging is blocked and the controller would otherwise request charging, the I term decays instead of increasing into a blocked direction.

## 15. Charge gate

The charge gate decides whether charging is allowed.

It is used to avoid charging the battery from the grid when PV production is insufficient.

### Enable condition

Charging is enabled when both conditions are true for 60 seconds:

```text
PV power > Charge Enable PV
and
grid_power < -Charge Enable Export
```

Default values:

```text
Charge Enable PV     = 250 W
Charge Enable Export = 80 W
```

This means:

```text
PV is above 250 W
and
there is at least about 80 W export
```

### Disable condition

Charging is disabled when this is true for 60 seconds:

```text
PV power < Charge Disable PV
```

Default value:

```text
Charge Disable PV = 150 W
```

### Important behavior

When the charge gate is blocked:

```text
charging is stopped
discharging remains allowed
```

Blocking charging must not automatically force battery discharge to zero.

## 16. Charge/discharge switching

EcoFlow STREAM uses separate control paths for charging and discharging.

The controller therefore switches deterministically.

### Switching from charge to discharge

```text
1. set charge power to 0 W
2. switch EcoFlow strategy to self_powered
3. set backup reserve if needed
4. set discharge/base-load power
```

### Switching from discharge to charge

```text
1. set discharge/base-load power to 0 W
2. switch EcoFlow strategy to scheduled
3. set charge power
```

The automation intentionally stops after important intermediate steps. This gives EcoFlow time to process the state change and avoids fighting both actuators at once.

## 17. SOC protection

Discharge is blocked when the battery SOC is at or below the configured minimum SOC.

Default value:

```text
Min Discharge SOC = 21 %
```

When SOC protection is active:

```text
discharge power = 0 W
charge power = 0 W if needed
integral value = 0 W
```

## 18. EV support mode

The controller can optionally support EV charging.

EV charging is detected by EV charger power:

```text
EV Power > EV Detection Threshold
```

Default threshold:

```text
1000 W
```

### EV support disabled

When EV charging is detected and EV support is disabled:

```text
EcoFlow output = 0 W
integral value = 0 W
```

This prevents the battery from unintentionally supporting EV charging.

### EV support enabled

When EV charging is detected and EV support is enabled:

```text
EcoFlow strategy = self_powered
Backup reserve   = 20 %
Discharge power  = configured EV support power
```

Default EV support discharge power:

```text
800 W
```

SOC protection still applies.

## 19. Recommended beta values

Start with conservative values:

```text
Kp               = 0.20
Ki               = 0.002
Target Grid      = -20 W
Deadband         = 50 W
Integral Limit   = 120 W
Control interval = 5 s
Max Discharge    = 800 W
Max Charge       = 1200 W
Discharge Step   = 10 W
Charge Step      = 1 W
```

Only tune more aggressively after the system is stable.

## 20. Tuning hints

### If the system reacts too slowly

Try:

```text
increase Kp slightly
increase Ki slightly
reduce deadband carefully
```

Example:

```text
Kp = 0.25
Ki = 0.003
Deadband = 40 W
```

### If the system oscillates

Try:

```text
lower Kp
lower Ki
increase deadband
increase smoothing window
lower integral limit
```

Example:

```text
Kp = 0.15
Ki = 0.001
Deadband = 70 W
Integral Limit = 80 W
```

### If the controller keeps overshooting after large load changes

Try:

```text
lower Ki
lower integral limit
increase deadband
reset integral memory
```

## 21. Practical target choice

Exact `0 W` grid exchange is often not the best practical target.

A small offset can be more stable:

```text
Target Grid = -20 W  => prefer tiny export
Target Grid = +20 W  => prefer tiny import
```

Which one is better depends on the user's preference, meter behavior and tariff situation.

## 22. Important limitations

The controller is not a native EcoFlow inverter API.

It emulates a unified bidirectional battery setpoint by coordinating the available EcoFlow STREAM control entities inside Home Assistant.

It cannot remove physical or software delays.

Typical limitations:

```text
grid meter update rate
BLE or cloud update delay
EcoFlow actuator response delay
inverter ramp behavior
Home Assistant automation timing
sensor noise
quantized actuator steps
competing automations
```

Because of this, the controller should be tuned for stable behavior, not for mathematically perfect zero-grid tracking.

## 23. Inspiration and references

This controller is built on top of and inspired by:

```text
rabits / ha-ef-ble
https://github.com/rabits/ha-ef-ble
```

```text
Proportional-integral-derivative controller
https://en.wikipedia.org/wiki/Proportional%E2%80%93integral%E2%80%93derivative_controller
```

```text
SunEnergyXT / Zero-Feed-in-Controller.yaml
https://gist.github.com/SunEnergyXT/e5487ff1669ed17c7a14e00ca2390f75
```

The EcoFlow STREAM implementation is not a direct copy of the SunEnergyXT controller. It adapts the general incremental PI idea to EcoFlow STREAM devices, including EcoFlow-specific mode switching, separated charge/discharge controls, PV charge gating, SOC protection, EV support and anti-windup handling.

