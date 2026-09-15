# SolarEdge Battery Control Roadmap

This document tracks future improvements, cleanup activities, investigations and diagnostics enhancements for the SolarEdge Battery Control package. Implemented fixes and known issues - including deployment/verification status - are tracked separately in `known_issues_and_fixes.md`, most recent first. Everything in this document is still open.

---

# 1. Product / Design Improvements

## HIGH

(No open HIGH items - "Align Safety SOC Limits And EMHASS SOC Limits" was fixed 2026-09-03, see known_issues_and_fixes.md.)

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

Note: `battery_minimum_percent` and `battery_maximum_state_of_charge` were already moved onto existing helpers as part of the Align Safety/EMHASS SOC Limits fix (2026-09-03, see known_issues_and_fixes.md) - only `maximum_power_from_grid` and `maximum_power_to_grid` remain hardcoded and in scope here.

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

### Update 2026-08-28 - modbus_busy stale-lock finding (Test 6.3 / 7.5)

While verifying test_plan.md Test 6.3 (Recovery After Failed Write) against
a real incident, found and confirmed via direct read of
`solaredge_modbusqueue.yaml`: `script.modbus_queue`'s "Release Modbus lock"
step (`input_boolean.turn_off modbus_busy`) is the last step in a plain
linear `sequence:`, not protected against an exception raised inside the
preceding `choose:` block. When a dispatched SolarEdge service call fails
(e.g. the 2026-08-27 06:45:06 "Connection failed: Modbus Error: [Connection]
Not connected" write failure), the script aborts before reaching that step
and `input_boolean.modbus_busy` is left stuck "on".

Confirmed via history this is not a permanent deadlock: `modbus_busy` sat
"on" for ~30 minutes after the 06:45 failure (only because no further
state change happened to re-trigger the apply automation in that window),
then was forced clear by `script.modbus_queue`'s own next-invocation guard
(`wait_template` with a 10s `timeout` + `continue_on_timeout: true`) at
07:15, after which normal on/off cycling resumed immediately. No command
was silently lost; the design self-heals within 10s of the next real
command, at worst.

Suggested fix: make the lock-release step exception-safe, e.g. wrap the
`choose:` action with `continue_on_error: true`, or restructure so the
`input_boolean.turn_off modbus_busy` step always runs regardless of what
happens inside `choose:` (HA's `continue_on_error` on individual actions,
or a `sequence:`/`if`-based cleanup that isn't skipped by an upstream
raise). This would release the lock immediately on failure instead of
relying on the next caller's 10-second timeout to force through it -
low-risk change, same self-healing behaviour as a fallback if the fix
itself is ever wrong.

Separately, the same 48h error_log review used for Test 7.5 (Modbus
Long-Term Stability) surfaced a baseline of Modbus connectivity noise not
yet root-caused: 27x "Cancel send"/"Repeating call"/"No response", 8x
"Cancel send" alone, 7x transaction_id mismatch, 1x coordinator fetch
failure, alongside the one real write failure above. Worth a focused
before/after comparison once the lock-release fix lands, and/or a longer
observation window to see whether these correlate with write bursts,
mode-command-guard skip patterns, or something environmental (RS485/TCP
contention, inverter response latency).

Note: the same 48h error-log review used for test_plan.md Test 7.5 found a
non-trivial baseline of Modbus connectivity noise (27x "Cancel send"/
"Repeating call"/"No response", 7x transaction_id mismatch) that made this
stale-lock fix unsafe to reintroduce naively - a bare `continue_on_error:
true` on script.modbus_queue's release step could let a queued backlog
(mode: queued, max: 10) drain rapidly with no pacing against a connection
that's still struggling, which is plausibly what caused a past attempt at
that same fix to flood the system with commands. That fix is intentionally
left alone for now (self-healing via the existing 10s wait-timeout is
considered acceptable, see test_plan.md Test 6.3) in favour of lower-risk
fixes elsewhere.

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

### SOC-sensor unavailable false-trigger - fixed 2026-09-03

A separate, higher-severity issue in the same connectivity family (a
transient `sensor.solaredge_b1_state_of_energy` dropout spuriously
commanding a real Layer 1 grid-charge) was found and fixed 2026-09-03,
and confirmed durably effective over a full week as of 2026-09-10 - see
known_issues_and_fixes.md for the full write-up. The underlying nightly
dropout itself (root cause still in this Modbus Connectivity
investigation, not the SOC-sensor fix) continues unchanged at 6/6 nights,
38-45s each, with a newly-noticed pattern: each night's episode lands
~11-13 minutes earlier than the previous night's (drifting ~22:00 ->
~21:13 over the 6 nights checked 2026-09-04 through 2026-09-09) - a
possible clue for whoever chases the ~22:00 clustering question above.

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

Fix implemented for the immediate consequence: a **Grid Charge Export
Cooldown** - see known_issues_and_fixes.md for the full write-up. This
is a circuit breaker on the *consequence* (immediate loss-making
reversal), not a fix for the *causes* above. Still open:

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

The momentary PV/load sampling issue itself (the false 11:00/14:00
triggers) was root-caused and fixed 2026-08-28, deployed and verified
live 2026-08-29 - see known_issues_and_fixes.md. Two related
branch-coverage gaps found during that verification were also fixed,
2026-09-03 (one confirmed firing live 2026-09-10, the other still
unconfirmed) - also in known_issues_and_fixes.md. A third, related
branch-coverage gap was found live 2026-09-10 - not yet fixed, see
known_issues_and_fixes.md - Known Issues - Not Yet Fixed.

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

A Mode Command Guard was implemented 2026-08-26 to reduce write frequency
for the dominant steady-state case - see known_issues_and_fixes.md.
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

---

Implemented fixes and known issues: see `known_issues_and_fixes.md`.