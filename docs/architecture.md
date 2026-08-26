# SolarEdge Battery Control Architecture

## Purpose

This package manages SolarEdge battery and inverter behaviour using:

- SolarEdge Modbus Multi
- EMHASS optimisation
- Nordpool/EPEX electricity prices
- Solcast PV forecast
- Home Assistant sensors, helpers, scripts and automations

The goal is to optimise battery charging, discharging, grid import and grid export based on:

- electricity price
- PV production
- house load
- battery SOC
- EMHASS forecast
- grid/export constraints
- SolarEdge inverter capabilities

The system is intentionally layered. Higher layers decide what should happen. Lower layers translate those decisions into safe SolarEdge Modbus commands.

---

## High-Level Control Flow

```
Prices / PV forecast / Load forecast
        ↓
      EMHASS
        ↓
Forecasted battery SOC / battery power / grid power
        ↓
Battery planning and target generation
        ↓
Safety and grid constraints
        ↓
Charge/discharge decision
        ↓
Modbus command generation
        ↓
Modbus Queue
        ↓
SolarEdge Modbus Multi
        ↓
SolarEdge inverter / battery
```

## Layer overview
```
Layer 1  Safety and Watchdog
Layer 2  Optimisation
Layer 3  Grid Constraints
Layer 4  Real-Time Execution
    Layer 4A  Decision Engine
    Layer 4B  Command Generation
    Layer 4C  Modbus Queue
    Layer 4D  SolarEdge Apply
```
## Layer Dependencies
```
Layer 1 may override:
- Layer 2
- Layer 3
- Layer 4

Layer 3 may override:
- Layer 2

Layer 2 may not bypass:
- Layer 1
- Layer 3

Layer 4 executes:
- The effective decision produced by Layers 1–3
```

---
# Layer 1 - Safety and Watchdog
## Purpose

Layer 1 defines hard safety boundaries for battery operation.

This layer must always take priority over optimisation, price signals, export logic and other control decisions.

### Responsibilities
- Protect battery minimum SOC.
- Protect battery maximum SOC.
- Detect unsafe or invalid states.
- Trigger recovery behaviour when SOC is critically low.
- Prevent discharging below configured limits.
- Prevent charging above configured limits.
- Provide watchdog signals for downstream logic.
### Main Files:
```
1.Safety_and_watchdog/safety_and_watchdog_helpers.yaml
1.Safety_and_watchdog/safety_limits_and_override.yaml
1.Safety_and_watchdog/watchdog_automations.yaml
1.Safety_and_watchdog/watchdog_sensors.yaml
```
## Main Concepts
### Minimum SOC

If battery SOC falls below the configured minimum SOC, discharge must stop.

### Critical SOC recovery

If SOC falls below a critical threshold, the system may force charging from PV and/or grid regardless of price or forecast.

### Maximum SOC

If battery SOC is above the configured maximum SOC, charging should be limited or stopped. The upper SOC safety limit is implemented using a dedicated hysteresis state:


### Inputs

```
Battery SOC
Minimum SOC helper
Maximum SOC helper
Critical SOC threshold
Watchdog enable/disable helpers
input_boolean.battery_high_soc_hold

```

### Outputs

```
Battery charging allowed
Battery discharging allowed
Safety reason
Recovery required
Watchdog status
High SOC Hold state
input_boolean.battery_high_soc_hold


```
### High SOC Hold

High SOC Hold provides hysteresis around the upper battery SOC limit.

The controller does not force battery export when the maximum SOC is reached.

Instead:

```text
SOC >= Maximum SOC
→ High SOC Hold enabled
→ Battery charging blocked

SOC <= Maximum SOC - 3%
→ High SOC Hold disabled
→ Normal control restored
```

This avoids repeated charge/discharge oscillation caused by SOC
quantisation near the upper safety limit.

# Layer 2 - Optimisation
## Purpose

Layer 2 determines the economically optimal battery behaviour using forecast data and EMHASS.

EMHASS provides a rolling optimisation plan based on:

- electricity price forecast
- PV forecast
- house load forecast
- current and forecast battery SOC
- grid import/export opportunities

### Responsibilities
- Run day-ahead and MPC optimisation.
- Publish EMHASS forecast data back into Home Assistant.
- Provide battery SOC target.
- Provide forecast battery charge/discharge power.
- Provide forecast grid import/export power.
- Generate dynamic charge/discharge targets.

### Main Files
```
2.Optimization/EMHASS/emhass_automations.yaml
2.Optimization/EMHASS/emhass_helpers.yaml
2.Optimization/EMHASS/emhass_restcommand.yaml
2.Optimization/EMHASS/emhass_scripts.yaml
2.Optimization/EMHASS/emhass_sensors.yaml
2.Optimization/EMHASS/emhass_shellcommand.yaml
```
### Main Inputs
```
Nordpool/EPEX price forecast
Solcast PV forecast
House load forecast
Battery SOC
Battery capacity
Current import/export prices
```
### Main Outputs


```
sensor.mpc_pv_batt_soc
sensor.mpc_pv_batt_power
sensor.mpc_pv_grid_power
input_number.soc_target
sensor.dynamic_storage_charge_limit
sensor.dynamic_storage_discharge_limit
Forecast SOC target
```
## Design Notes

EMHASS is the planning layer. It should not directly write to the inverter.

EMHASS output is interpreted by lower layers before any SolarEdge command is created.

This allows safety, grid constraints and real-time conditions to override or adjust the optimisation result.
---
# Layer 3 - Grid Constraints
## Purpose

Layer 3 handles grid-related restrictions and economic export constraints.

This layer can override optimisation if exporting or importing is not allowed or not economically desirable.

### Responsibilities
- Curtail export during negative export prices.
- Apply site/export limits.
- Prevent uneconomic export behaviour.
- Expose grid constraint status to downstream logic.

### Main Files
```
3.Grid_constraints/gridconstrain_helpers.yaml
3.Grid_constraints/negative_price_curtailment.yaml
```
## Main Concepts
### Negative price curtailment

When spot/export price is negative, exporting energy to the grid may be economically harmful.

In that case, the system can enable SolarEdge site limiting.

```
switch.solaredge_i1_negative_site_limit
number.solaredge_i1_site_limit
```
### Site limit

Site limit applies at the grid connection level. It limits net export from the whole installation, not only battery export.

### Inputs


```
Current electricity price
Forecast electricity price
Export price
Grid/export state
PV production
House load
```
### Outputs


```
Export allowed
Site limit enabled
Site limit value
Curtailment reason
```


---

# Layer 4 - Real-Time Execution
## Purpose

Layer 4 turns planning and constraints into real inverter commands.

This layer is responsible for:

- deciding the current battery mode
- calculating effective charge/discharge limits
- generating SolarEdge commands
- serialising writes through the Modbus queue
- applying commands to SolarEdge

Layer 4 must be robust because it interacts directly with the inverter.

```
Layer 4A Decision Engine
        ↓
Layer 4B Command Generation
        ↓
Layer 4C Modbus Queue
        ↓
Layer 4D SolarEdge Apply
```

---

# Layer 4A - Decision Engine
## Purpose

Layer 4A evaluates current state and decides what the battery should do now.

### Responsibilities
- Interpret EMHASS forecast.
- Compare current SOC to target SOC.
- Apply safety limits.
- Apply grid constraints.
- Select effective battery behaviour.
- Decide whether to charge, discharge, hold, curtail or recover.

### Main Files
```
4.Real_time_execution/battery_forecast_control.yaml
4.Real_time_execution/batterycontrol_automations.yaml
4.Real_time_execution/batterycontrol_sensors.yaml
```
### Decision Inputs


```
Battery SOC
SOC target
PV production
House load
EMHASS battery forecast
EMHASS grid forecast
Dynamic charge limit
Dynamic discharge limit
Safety state
Grid constraint state
```

### Sign Conventions

```
p_pv            > 0 when PV energy is available
p_batt_forecast < 0 charging, = 0 stable SOC, > 0 discharging
p_grid_forecast > 0 import,   = 0 self consumption, < 0 export to grid
```
### Decision Outputs


```
Requested storage command mode
Requested charge limit
Requested discharge limit
Battery control reason
Effective battery control state
```
---

# Layer 4B - Command Generation
## Purpose

Layer 4B translates decisions into SolarEdge command requests.

This layer should not directly bypass the Modbus Queue.

### Responsibilities
- Create command payloads.
- Select SolarEdge storage mode.
- Set storage charge limit.
- Set storage discharge limit.
- Set command timeout when required.
- Call Modbus Queue scripts.

### Main Files
```
4.Real_time_execution/batterycontrol_scripts.yaml
4.Real_time_execution/apply_effective_battery_control.yaml
```
### Outputs


```
Storage command mode request
Storage charge limit request
Storage discharge limit request
Storage command timeout request
```
### Behaviour

Mode changes are relatively heavy and should be infrequent.

Power limit updates can be more frequent, but should still be throttled to avoid excessive Modbus write pressure.

### Mode Command Guard

`automation.apply_effective_battery_control_to_solaredge_inverter` is triggered
whenever the effective mode OR either effective limit changes state - and the
effective limits can change every cycle simply because they track the
EMHASS-derived dynamic charge/discharge limits. Without a guard this would
resend the mode-select command (plus a 15-minute command timeout reset) on
every such trigger, even when the mode itself hasn't changed.

`maximize_self_consumption` is the most frequently requested effective mode.
It is also, when the inverter is configured that way, the mode SolarEdge
falls back to internally once its own Remote Control command timeout lapses
without a refreshed command - that configured fallback is exposed as
`select.solaredge_i1_storage_default_mode`. The automation skips the
mode-select command and timeout reset only when ALL of the following hold:

```
effective mode == maximize_self_consumption
AND
select.solaredge_i1_storage_command_mode == "Maximize Self Consumption"
AND
select.solaredge_i1_storage_default_mode == "Maximize Self Consumption"
```

The first two conditions confirm we want to stay in Maximize Self
Consumption and the inverter is already there (so there's no pending
transition to apply). The third confirms letting the command timeout lapse
is actually safe - it checks the inverter's real configured fallback rather
than assuming it. If the default/backup mode is ever reconfigured to
something else, the guard simply stays inactive and the mode command keeps
being refreshed every cycle, so control authority is never silently ceded
to an unexpected fallback mode.

In the guarded case only the charge/discharge limit scripts run - which,
per the Layer 4D command types below, apply independently of the storage
command mode/timeout cycle. Any other effective mode, or either SolarEdge
mode entity not (yet/still) confirming Maximize Self Consumption, still
sends the full command sequence, so real mode transitions remain immediate.
The guard is self-healing: it reads the inverter's live reported state
rather than trusting a locally cached "last sent" value, so a manual mode
change, restart, reconfiguration of the default mode, or Modbus hiccup
simply causes the guard to re-arm rather than silently drift.

`sensor.battery_mode_command_guard` mirrors this three-part condition as a
standalone diagnostic ("guarded" vs "active") for observability, independent
of the automation actually firing.

---

# Layer 4C – Modbus Queue

## Purpose

Serialize all Modbus writes to SolarEdge.

### Responsibilities

- Ensure only one write is active at a time
- Maintain busy state
- Queue command execution
- Protect against concurrent writes

### Main Files

```
4.Real_time_execution/solaredge_modbusqueue.yaml
```
### Main Entities


```
script.modbus_queue
input_boolean.modbus_busy
```


## Design Principle

All writes to SolarEdge should go through the Modbus Queue.

Direct writes from other automations should be avoided.

---

# Layer 4D - SolarEdge Apply
## Purpose

Layer 4D is the actual application of commands to SolarEdge through SolarEdge Modbus Multi.

### Responsibilities
- Execute SolarEdge Modbus Multi number/switch service calls.
- Apply storage command mode.
- Apply charge/discharge limits.
- Apply site/export limits.
- Handle command timeout where required.

### Main Integration
```
SolarEdge Modbus Multi
```
### Main Command Types


```
Storage Command Mode
Storage Charge Limit
Storage Discharge Limit
Storage Command Timeout
Site Limit
Negative Site Limit
```

---

Layer 4D uses SolarEdge Storage Command Modes.

See README.md for detailed mode descriptions.

# Key Design Principles
## 1. Safety wins

Safety limits must override optimisation and export logic.

## 2. Optimisation plans, lower layers execute

EMHASS should provide targets and forecasts, not directly control the inverter.

## 3. Grid constraints override profit

Negative price curtailment and export limits can override EMHASS export decisions.

## 4. Writes must be serialised

All SolarEdge writes should pass through the Modbus Queue.

## 5. Reduce unnecessary writes

SolarEdge Modbus appears sensitive to frequent or overlapping writes.

Write logic should avoid repeated commands when the requested value has not changed significantly.

The Layer 4B Mode Command Guard (see above) is a concrete application of this
principle: it stops the automation from resending an unchanged
`maximize_self_consumption` mode command every 15 minutes purely because a
dynamic power limit changed, while still updating the limits themselves.