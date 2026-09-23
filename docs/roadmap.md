# SolarEdge Battery Control Roadmap

This document tracks future improvements, cleanup activities, investigations and diagnostics enhancements for the SolarEdge Battery Control package. Implemented fixes and known issues - including deployment/verification status - are tracked separately in `known_issues_and_fixes.md`, most recent first. Everything in this document is still open; resolved narrative that used to be interspersed below has been moved to the Resolved section at the end, kept only for context.

---

# 1. Product / Design Improvements

## HIGH

(No open HIGH items.)

---

## MEDIUM

### Centralise EMHASS Configuration

Two EMHASS parameters are still hardcoded inside scripts:

```text
maximum_power_from_grid
maximum_power_to_grid
```

Evaluate moving these values into dedicated helpers so that they can be adjusted without modifying scripts. (`battery_minimum_percent` and `battery_maximum_state_of_charge` were already moved onto existing helpers - see Resolved section below.)

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

The modbus_busy stale-lock gap found 2026-08-28 (Test 6.3 / 7.5) was fixed
2026-09-20 - see Resolved section below and known_issues_and_fixes.md for
current deployment status. The 48h error-log baseline of connectivity
noise (27x "Cancel send"/"Repeating call"/"No response", 7x transaction_id
mismatch) found during that same review is still open and not yet
root-caused - worth a focused before/after comparison now that the
lock-release fix has landed, and/or a longer observation window to see
whether it correlates with write bursts, mode-command-guard skip
patterns, or something environmental (RS485/TCP contention, inverter
response latency).

### Update 2026-09-10 - Two new real Modbus write failures found

The deeper charge/discharge review's Modbus sampling (5 windows across
2026-09-04 through 2026-09-10, explicitly not exhaustive 6-day coverage)
turned up two actual command **write** failures - not the usual poll/read
noise - on 2026-09-08 22:45:02 and 2026-09-10 13:30:03: "Modbus
Queue...Modbus Error: [Input/Output] Request cancelled outside library".
Not previously documented. Same sampling also re-confirmed the baseline
connectivity noise is essentially unchanged from the 2026-08-28 finding
above (still present, similar magnitude). Not yet root-caused or
correlated with anything specific; noted here for whoever picks up this
investigation next.

### Update 2026-09-19 - Extended SOC-sensor outage (~31 hours), root cause unknown

A routine multi-day log review found `sensor.solaredge_b1_state_of_energy`
stuck `unavailable` for roughly 31 hours (2026-09-16 14:29 to 2026-09-17
21:37 local) - far beyond the usual 6.5-45s nightly blips already tracked
below. It was isolated to this one entity: `sensor.solaredge_i1_ac_power`,
`select.solaredge_i1_storage_control_mode`, and unrelated automations
(e.g. `ev_charging_discharge_control`) kept running normally throughout,
with only the usual brief reconnect blips. The outage only cleared when
Home Assistant restarted that evening (an unrelated Supervisor
auto-update, 2026.09.0 -> 2026.09.2) - no error in the available logs
explains why the sensor was stuck for so long, or why nothing short of a
full restart recovered it. Not yet correlated with the connectivity
patterns above. The immediate consequence (Layer 4A silently running on a
fallback `soc=0` for the full outage) was fixed the same day - see
known_issues_and_fixes.md - but the cause of the outage itself is a new,
open item here.

Separately, the well-characterized short nightly dropout (6/6 nights,
38-45s each as of the 2026-09-10 check) continues unchanged, with a
noticed pattern: each night's episode lands ~11-13 minutes earlier than
the previous night's (drifting ~22:00 -> ~21:13 over the 6 nights checked
2026-09-04 through 2026-09-09) - a possible clue for the ~22:00 clustering
question above, not yet chased further.

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

### Update 2026-08-27 - Diagnosed root cause

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

A circuit breaker on the immediate consequence (Grid Charge Export
Cooldown) was implemented 2026-08-26 - see Resolved section below. Still
open, the underlying causes:

```text
EMHASS MPC plan chatter: input_number.soc_target and sensor.
soc_batt_forecast_smooth swing tens of percentage points between
consecutive MPC runs. Investigate emhass_scripts.yaml cost function
weights (weight_battery_charge/discharge currently 0.02) and whether the
MPC optimisation needs a "stay near previous plan" penalty or coarser
re-plan cadence. This is an EMHASS-side change, out of scope for a single
HA package edit - needs its own investigation.

Related: the EMHASS solver itself hit its internal `user_limit` (time/
iteration limit) once, on 2026-09-04 06:45-06:49, causing a 490.6s
optimisation request that exceeded the `timeout: 240` on
`rest_command.emhass_naive_mpc_optim`/`emhass_dayahead_optim`
(`emhass_restcommand.yaml`); the system self-recovered by the 07:00 run.
Reassessed 2026-09-10: no evidence of recurrence found in the available
(partial, sampled) log coverage across the following 6 days - both log
sources have limited retention and can't see the original incident
window itself, so this is "no evidence of recurrence," not certainty. No
change to weight_battery_charge/discharge was made on this evidence; a
modest rest_command timeout increase (from 240s) was floated as cheap
insurance but is still the user's call, not yet decided. Not yet logged
as its own known_issues_and_fixes.md entry - captured here for now.

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

(A Mode Command Guard already reduces write frequency for the dominant
steady-state case - see Resolved section below.) Still outstanding from
this investigation:

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

Note 2026-09-19: a `binary_sensor.solaredge_modbus_data_fresh`-style
"age since last update" sensor on `sensor.solaredge_b1_state_of_energy`
specifically would have made the 31-hour outage above visible immediately
instead of only being caught by a manual multi-day history review -
worth prioritising this one sensor even before the rest of this list.

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

---

# 5. Resolved (Context Only)

Everything below is already fixed - kept here only as background for the
open items above that reference it. Deployment/verification status for
each lives in `known_issues_and_fixes.md`; this is not a duplicate log.

**Align Safety SOC Limits And EMHASS SOC Limits** (fixed 2026-09-03) -
`battery_minimum_percent` and `battery_maximum_state_of_charge` were
hardcoded separately from the safety layer's SOC helpers; both now read
the same `input_number.minimum_state_of_charge` /
`maximum_state_of_charge` helpers EMHASS reads too.

**SOC-Sensor Unavailable False-Trigger** (fixed 2026-09-03, confirmed
durable over a full week as of 2026-09-10) - a transient
`sensor.solaredge_b1_state_of_energy` dropout was spuriously commanding a
real Layer 1 grid-charge (`soc` falling back to 0, satisfying the
"critical recovery" branch). Fixed with an automation-level
unavailable/unknown guard on `calculate_effective_battery_control` and
`battery_high_soc_hold`. This same guard was extended to
`emhass_battery_forecast_control` (Layer 4A) on 2026-09-19 after the
extended outage noted above - see known_issues_and_fixes.md for current
deployment status of that extension.

**Midday Oscillation - PV/Load Sampling Smoothing** (root-caused and
fixed 2026-08-28, deployed and verified live 2026-08-29) - the false
11:00/14:00 triggers were momentary PV/load sampling artifacts; fixed
with 4-minute trailing-mean sensors for mode-selecting decisions.

**Midday Oscillation - Branch-Coverage Gaps** (three found and fixed,
2026-09-03 through 2026-09-15) - two related gaps found during the
PV/Load Sampling Smoothing verification (one confirmed firing live
2026-09-10, the other still unconfirmed), plus a third found live
2026-09-10 and fixed 2026-09-15 (confirmed firing live the same day,
after an initial stale-reload false alarm).

**Midday Oscillation - Grid Charge Export Cooldown** (implemented
2026-08-26) - a hysteresis circuit breaker: `discharge_to_maximize_export`
is withheld for a configurable cooldown after any grid-assisted charge,
so a charge isn't immediately sold back out at a loss. Addresses the
consequence of the midday oscillation, not its causes (see Active
Investigations above).

**Write Pressure - Mode Command Guard** (implemented 2026-08-26) - skips
the mode-select command and 15-minute command-timeout reset when the
effective mode is already `maximize_self_consumption` and the inverter
confirms that's both its current and default mode, cutting write
frequency for the dominant steady-state case.

**Modbus Connectivity - modbus_busy Stale Lock** (found 2026-08-28, fixed
2026-09-20) - a Modbus write failure inside `script.modbus_queue`'s
`choose:` block aborted the script before its lock-release step, leaving
`input_boolean.modbus_busy` stuck "on" (self-healed after ~30 min via the
next invocation's 10s timeout, but delayed every command in between). A
naive fix was deferred for weeks over flood risk (releasing instantly on
every failure could drain a queued backlog with no pacing against a
still-struggling connection). Fixed by adding `continue_on_error: true`
to the `choose:` step and consolidating ~13 scattered per-branch trailing
delays into one uniform post-choose delay that runs on both the success
and the caught-failure path, keeping the same pacing either way. See
known_issues_and_fixes.md for full detail and current deployment status.

---

Implemented fixes and known issues: see `known_issues_and_fixes.md`.