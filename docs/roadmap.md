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

## Midday Charge/Discharge Oscillation (Uneconomical)

### Update 2026-08-27 - Diagnosed and mitigated with Grid Charge Export Cooldown

Reported: on 2026-08-27, during a low-export-price midday window
(`sensor.total_export_price` ~0.95-1.05 SEK/kWh vs.
`sensor.total_import_price` ~2.1-2.33 SEK/kWh), the system executed
`charge_from_solar_and_grid` (real grid import) at 11:00 and 14:00, each
followed within 15-75 minutes by `discharge_to_maximize_export` - selling
the same energy back out at the same depressed export price. Guaranteed
loss on the round trip regardless of interpretation.

Confirmed via HA history/logs this was NOT a Layer 1 safety override and
NOT Layer 3 negative-price curtailment (`negative_price_active` has been
off since 2026-08-20) - `effective_storage_mode` tracked
`emhass_requested_storage_mode` 1:1 throughout ("Normal - effective
follows requested"), so the decision originated entirely in Layer 4A
(`battery_forecast_control.yaml`).

Root cause, confirmed with data pulled at the exact trigger instants:

```text
automation.emhass_battery_forecast_control fires on a rigid time_pattern
every 15 minutes and reads sensor.solar_panel_production_w and
sensor.power_myhouse_load_no_var_loads as raw, ~3-second-resolution
instantaneous values, with no averaging.

At 11:00:01, PV had momentarily dipped to ~430-480W (from a baseline of
~2150-2250W moments earlier) while house load had momentarily spiked to
~2800W (from a baseline of ~1100W) - both transient, lasting a few
seconds, coincidentally overlapping the exact trigger timestamp.

At 14:00:01, the same pattern repeated: PV momentarily ~800-1000W (own
baseline nearby was similar - lower sun angle) against a load spike to
~2500-2600W.

Both coincidences flipped the "pv <= house_load" branch split for the
full 15-minute interval, authorizing charge_from_solar_and_grid even
though PV covered load for nearly all of each interval. None of the
branches that fired (charge_from_solar_and_grid, discharge_to_maximize_
export) check price - the one existing price mechanism
(input_number.average_last_chargingprice) is a running average of price
already paid during an active charging session (used only to decide
whether to keep charging, in the deeply-nested no-PV fallback branch),
not an upfront gate on starting a grid charge - it cannot prevent this.

A secondary, related factor observed in the same data: input_number.
soc_target (recomputed every minute in batterycontrol_automations.yaml
from sensor.mpc_pv_batt_soc's battery_scheduled_soc attribute, i.e. the
latest EMHASS MPC plan) swung by 10-40 percentage points between
successive 1-2 minute reads (e.g. 45.02 at 11:00:01, then 24.44 at
11:02:00). sensor.soc_batt_forecast_smooth, despite its name, shows the
same volatility. This points to EMHASS's own MPC re-optimisation
producing substantially different near-term battery plans run to run
(rolling-horizon "plan chatter"), likely enabled by the low
weight_battery_charge/weight_battery_discharge (0.02) giving the solver
little disincentive against a chattering trajectory. This was NOT fixed
in this pass - see "Still open" below.
```

Fix implemented (`battery_forecast_control.yaml`, `batterycontrol_helpers.yaml`):
a **Grid Charge Export Cooldown**, following the same hysteresis pattern
already used by `battery_high_soc_hold`. Whenever
`charge_from_solar_and_grid` fires, `input_datetime.
last_grid_charge_command_time` is stamped. The `DISCHARGE MAX EXPORT`
branch only executes `discharge_to_maximize_export` if `input_number.
grid_charge_export_cooldown_minutes` (default 60) has elapsed since that
stamp; otherwise it falls back to `maximize_self_consumption` with the
normal dynamic discharge limit, so the battery serves house load instead
of being sold straight back to the grid. Full design and rationale:
architecture.md - Layer 4A - Grid Charge Export Cooldown.

This is a circuit breaker on the *consequence* (immediate loss-making
reversal), not a fix for the *causes* above. Still open:

```text
Momentary PV/house-load sampling: the pv > house_load / pv <= house_load
comparison in battery_forecast_control.yaml still reads raw instantaneous
sensor state at the exact 15-min tick. A short rolling average (e.g.
2-5 min) for sensor.solar_panel_production_w and sensor.
power_myhouse_load_no_var_loads, used only for this comparison, would
reduce false triggers directly rather than just containing their
consequence. Needs a new template/statistics sensor - not implemented
here to keep this fix minimal and low-risk.

EMHASS MPC plan chatter: input_number.soc_target and sensor.
soc_batt_forecast_smooth swing tens of percentage points between
consecutive MPC runs. Investigate emhass_scripts.yaml cost function
weights (weight_battery_charge/discharge currently 0.02) and whether the
MPC optimisation needs a "stay near previous plan" penalty or coarser
re-plan cadence. This is an EMHASS-side change, out of scope for a single
HA package edit - needs its own investigation.

Whether 60 minutes is the right default cooldown - only bounded by one
day of observation (first discharge event trailed its charge event by 75
minutes; a second charge/discharge pair happened same day). Needs a
longer observation window and possibly a shorter/longer default.
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

### Update 2026-08-27 - Live log check after guard deployment

Checked HA logs/traces the same day the guard went live. The guard itself
is confirmed working (see test_plan.md Test 7.6). Modbus errors are still
present but appear pre-existing rather than guard-caused:

```text
"Cancel send, because not connected" / "No response received after 3
retries" - ~15 occurrences, spread across several hours before and
after the reload
"transaction_id mismatch" - 3 occurrences, same period
"Connection failed: Modbus Error: [Connection] Not connected" plus a
cascade of modbus_queue / apply_effective_battery_control script errors
- one cluster of ~7-8 entries, timestamps line up with the
select.solaredge_i1_storage_command_mode "unavailable" blip during a HA
restart the previous evening, not with the guard's reload
```

Counts are modest (single digits to low teens over many hours), consistent
with the pre-existing intermittent connectivity picture already described
above rather than a new problem. No direct evidence yet either way that
the guard has measurably reduced or worsened error frequency - the
"Skipped writes" / "Modbus write frequency" telemetry from the Enhanced
Diagnostics section below would be needed to say that with confidence.
Worth a longer before/after observation window once
sensor.battery_mode_command_guard is actually live (see test_plan.md - it
was missing on the instance checked).

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