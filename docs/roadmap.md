# SolarEdge Battery Control Roadmap

This document tracks future improvements, cleanup activities, investigations and diagnostics enhancements for the SolarEdge Battery Control package.

---

# 1. Product / Design Improvements

## HIGH

### Align Safety SOC Limits And EMHASS SOC Limits

Safety layer and EMHASS currently maintain separate minimum SOC settings.

Examples:

```text
input_number.minimum_state_of_charge
battery_minimum_percent
```

Evaluate using a single source of truth for battery minimum SOC.

---

## MEDIUM

### Centralise EMHASS Configuration

Several EMHASS parameters are currently hardcoded inside scripts.

Examples:

```text
battery_minimum_percent
battery_maximum_state_of_charge
maximum_power_from_grid
maximum_power_to_grid
```

Evaluate moving these values into dedicated helpers so that they can be adjusted without modifying scripts.

---

### Improve EMHASS Optimisation Failure Handling

Several automations rely on:

```text
sensor.mpc_pv_optim_status
sensor.dh_pv_optim_status
```

Investigate whether optimisation failures should automatically trigger fallback planning behaviour rather than only updating watchdog status.

---

### Review Dynamic Charge/Discharge Limit Duplication

Current control chain:

```text
sensor.dynamic_storage_charge_limit
→ input_number.dynamic_charge_limit
→ input_number.last_charge_limit

sensor.dynamic_storage_discharge_limit
→ input_number.dynamic_discharge_limit
→ input_number.last_discharge_limit
```

Review whether all intermediate helpers are still required or if the control chain can be simplified.

---

### Add Queue Telemetry

Current queue operation is largely opaque.

Add telemetry for:

```text
queue command count
queue timeout count
queue wait time
queue execution time
last queued command
last executed command
```

---

# 2. Cleanup / Technical Debt

### EMHASS Healthy Naming

Current entity:

```text
binary_sensor.emhass_healthy
```

uses:

```text
device_class: problem
```

meaning:

```text
ON  = problem detected
OFF = healthy
```

Review whether the entity name should be changed to better reflect its behaviour.

Possible alternatives:

```text
binary_sensor.emhass_problem
binary_sensor.emhass_unhealthy
```

---

### EMHASS Healthy Unique ID Typo

Current unique_id:

```text
emahss_healthy
```

Expected spelling:

```text
emhass_healthy
```

Review impact on Home Assistant entity registry before making any change.

---

### Audit Published EMHASS Forecast Sensors

Review all published:

```text
mpc_pv_* sensors
dh_pv_* sensors
```

Identify:

```text
unused sensors
duplicated sensors
legacy entities
```

Update entities.md after cleanup.

---

### Clarify Grid Export States

Current implementation contains:

```text
input_boolean.grid_export_blocked
input_boolean.grid_export_limited
```

Only `grid_export_blocked` is currently used by the negative price curtailment logic.

Review whether:

```text
grid_export_limited is used elsewhere
grid_export_limited is redundant
grid_export_limited should be removed
grid_export_limited should be expanded
```

---

# 3. Active Investigations

## Modbus Connectivity

Observed errors:

```text
Cancel send, because not connected
No response received after 3 retries
Request cancelled outside library
transaction_id mismatch
```

Need to determine whether these correlate with:

```text
port probe failures
data freshness failures
write bursts
battery control mode changes
negative price curtailment
```

---

## Dynamic Discharge Oscillation

Observed graphs suggest that dynamic discharge power may drive visible SolarEdge power oscillations.

Need to determine whether this is:

```text
expected control behaviour
or
control loop instability
```

---

## Write Pressure

Frequent Modbus Queue activity appears to coincide with some observed errors.

Need to test whether reducing write frequency improves Modbus stability.

Potential future mitigation:

```text
minimum write interval
minimum delta threshold
limited retry
backoff after failure
```

### Update 2026-08-26 - Mode Command Guard implemented

`automation.apply_effective_battery_control_to_solaredge_inverter` now skips
the mode-select command and the 15-minute command timeout reset whenever the
effective mode is `maximize_self_consumption` (the most-used fallback mode),
`select.solaredge_i1_storage_command_mode` already confirms the inverter is
in that mode, AND `select.solaredge_i1_storage_default_mode` confirms the
inverter's own configured fallback is also Maximize Self Consumption (so
letting the command timeout lapse is verified safe, not assumed). Only the
charge/discharge limit writes still go out in that case, instead of the
full 4-write sequence (timeout reset, mode select, charge limit, discharge
limit) on every trigger.

This directly reduces write frequency for the dominant steady-state case
(dynamic limit changes while already in Maximize Self Consumption), without
introducing a minimum write interval or delta threshold. `sensor.
battery_mode_command_guard` exposes whether the guard is currently active for
observability.

Still outstanding from this investigation:

```text
Whether reduced mode-command write frequency measurably improves Modbus
stability (needs a before/after observation window)
Minimum write interval / delta threshold for the charge/discharge limit
writes themselves, which the guard does not throttle
limited retry / backoff after failure
```

---

## Negative Price Curtailment

Needs explicit verification of:

```text
negative export price
site limit enabled
site limit value applied
export reduced
normal operation restored
```

---

# 4. Enhanced Diagnostics

## Required Telemetry

Add visibility for:

```text
Modbus write frequency
Last successful write
Last failed write
Last command type
Last command reason
Queue length
Pending command count
Command delta
Skipped writes
```

`sensor.battery_mode_command_guard` (added 2026-08-26) partially covers
"Skipped writes" for the mode-select command specifically - it does not yet
cover skipped/deduplicated charge or discharge limit writes.

---

## Recommended Diagnostic Sensors

```text
binary_sensor.solaredge_modbus_port_raw
binary_sensor.solaredge_modbus_port_stable

sensor.modbus_last_successful_write
sensor.modbus_last_failed_write
sensor.modbus_last_command_reason

sensor.solaredge_i1_ac_power_age_seconds
sensor.solaredge_m1_ac_power_age_seconds

binary_sensor.solaredge_modbus_data_fresh
```

These should help distinguish between:

```text
TCP port unavailable
existing Modbus session stale
SolarEdge data not updating
write queue overload
pymodbus transaction recovery issue
```

---

## Observability Goals

The system should make it possible to answer:

```text
What did the optimiser want?
What did the decision engine decide?
What command was generated?
Was the command queued?
Was the command applied?
Did SolarEdge respond?
```

---

# Current Technical Risk

Recent observations show that Modbus errors may occur during periods with frequent inverter writes.

Observed symptoms:

```text
Cancel send, because not connected
No response received after 3 retries
Request cancelled outside library
transaction_id mismatch
```

The current hypothesis is that one or more of the following may contribute:

```text
Modbus write pressure
Reconnect handling
SolarEdge Modbus TCP behaviour
Mode switching frequency
Export curtailment transitions
```

A structured investigation is required before introducing mitigation logic.

## Add Parallel Shadow Apply Layer

Create a parallel apply path for diagnostics and future algorithm validation.

Goals:

- Compare effective vs applied values
- Test new control logic safely
- Validate write-throttling strategies
- Avoid impacting production operation

Priority: Low

Implementation:

effective_* entities
→ shadow_* entities

No SolarEdge writes.