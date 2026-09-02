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

### Update 2026-08-28 - Momentary PV/house-load sampling: root-cause fix implemented

Added `sensor.pv_production_smoothed` and `sensor.house_load_smoothed`
(`platform: statistics`, 4-minute trailing mean, `batterycontrol_sensors.yaml`)
and switched `automation.emhass_battery_forecast_control`'s `pv` and
`house_load` decision variables to read from them instead of the raw
instantaneous sensors. Full design, and the window-size validation against
the two known 2026-08-27 false triggers (only a 4-minute window correctly
resolves both; 3 minutes alone misses the 14:00 event, 5 minutes alone
misses the 11:00 event): architecture.md - Layer 4A - PV/Load Sampling
Smoothing.

Not yet deployed/verified live - needs copying to the live config and a
reload, then a follow-up check via HA MCP once the smoothed sensors have
been running long enough to observe a real would-be-false-trigger instant
and confirm the branch it actually takes. Logged as a pending test_plan.md
item once live.

### Update 2026-08-28 - Split smoothed vs instantaneous by decision type

Refined per feedback: smoothing should only apply to the storage MODE
decision (an infrequent, "heavy" change worth protecting from noise), not
to the charge-limit ceiling chosen inside Maintain Zone, which should
still react to a real short PV/load change at each 15-min tick rather
than lag behind a 4-minute average. battery_forecast_control.yaml now
defines both `pv`/`house_load` (smoothed, used by every mode-selecting
branch) and `pv_now`/`house_load_now` (raw, used only for Maintain
Zone's 5000W-vs-dynamic charge ceiling choice, where both outcomes stay
in maximize_self_consumption so there's no mode-flapping risk). Full
detail: architecture.md - Layer 4A - PV/Load Sampling Smoothing. Note the
automation only evaluates on a 15-minute time_pattern trigger either way,
so "instantaneous" means "this tick's snapshot," not "reacts within
seconds."

### Update 2026-08-29 - Deployed and confirmed live

Copied to the live config and reloaded. First reload attempt did NOT pick
up the two new `platform: statistics` sensors (`sensor.
pv_production_smoothed`, `sensor.house_load_smoothed`) - confirmed via HA
MCP that `automation.emhass_battery_forecast_control` itself reloaded
cleanly (fresh traces showed `cooldown_elapsed`, `pv_now`, `house_load_now`
all computing correctly) but the two smoothed sensors were absent from the
entity registry with zero matching log lines anywhere (ruling out a config/
schema error). A full Home Assistant restart (not just a YAML/config
reload) was needed, since `platform: statistics` is a legacy YAML sensor
platform outside HA's hot-reloadable domains - confirmed via `current_
recorder_run` resetting and the core version bumping 2026.8.2 -> 2026.8.3.
Even after that first restart the sensors still weren't there for several
minutes (this instance's ~7GB recorder database makes the statistics
platform's own startup slow - "Setup of sensor platform statistics is
taking over 10 seconds" is a known, non-fatal, pre-existing warning on
this instance, seen on every restart since at least 2026-08-26). A second
restart-and-wait cycle brought both sensors up successfully.

Verified via a live trace (2026-08-29T01:00:00 local,
`automation.emhass_battery_forecast_control`): `pv=0, house_load=153`
(smoothed, live, driving mode selection) and `pv_now=0, house_load_now=137`
(raw, live, driving the Maintain Zone charge-ceiling choice) - landed in
Maintain Zone as expected for nighttime with no PV. This confirms the
smoothed sensors are populating correctly and the mode-vs-limit variable
split is wired up as designed. Not yet confirmed: real daylight exercise
of the mode-selecting branches (Clipped Solar, Below-SOC) that actually
motivated this fix - follow-up check planned after sunrise (05:55 local)
on 2026-08-29.

### Update 2026-08-29 (morning) - Daylight review

Reviewed all 5 retained traces of `automation.emhass_battery_forecast_control`
from 2026-08-29 09:45-10:45 local (real PV: 700-1800W, real load:
130-2040W) plus the full day's `input_select.emhass_requested_storage_mode`
/ `effective_storage_mode` history since 05:00 local.

Mode stayed at `maximize_self_consumption` for the entire morning with
zero transitions - no oscillation, no repeat of the charge-then-export
pattern this fix targeted. The Below-SOC and Clipped-Solar branches (the
ones that actually caused the original bug) weren't exercised today, since
SOC tracked at or above target all morning - so the smoothing's effect on
*those* specific branches is still unconfirmed with real data; this was a
quiet, uneventful morning by coincidence, not a stress test.

What the split DID demonstrably do, twice (07:45 and 08:45 local): in
Maintain Zone, the 4-min smoothed `pv`/`house_load` said PV was tracking
above load on average (e.g. 07:45: pv=712 vs house_load=529), which would
have opened the 5000W charge ceiling if that average were used for the
ceiling choice too. But the raw `pv_now`/`house_load_now` at that exact
tick showed load currently ahead of PV (761 vs 825 at 07:45) - so the
system correctly stayed on the normal dynamic ceiling instead of opening
to 5000W on stale averaged headroom. This is direct evidence the mode/
limit split is doing real work, not just passing through unchanged values.

Unrelated finding surfaced while reviewing (not caused by this fix):
twice this morning (08:15 and 08:30 local, `above` true, `grid_fc` around
-500 to -700 i.e. export favourable, `batt_fc` exactly 0), the automation
matched NO branch at all and left the previous requested mode/limits
unchanged for that cycle. Root cause: the DISCHARGE MAX EXPORT branch
requires `batt_fc > 0` and DISCHARGE MIN IMPORT/MAX SELF-CONSUMPTION
requires `grid_fc >= 0`; a state where SOC is above target, export is
forecast favourable, but EMHASS's near-term battery-power forecast is
exactly flat falls through both. Logged as a new gap to look at
separately - not urgent (the fallback is simply "keep the previous
requested values," not an unsafe state), but worth an explicit branch or
a `<=`/`>=` boundary fix so every reachable (above, grid_fc, batt_fc)
combination is covered.

Note: the same 48h error-log review used for test_plan.md Test 7.5 found a
non-trivial baseline of Modbus connectivity noise (27x "Cancel send"/
"Repeating call"/"No response", 7x transaction_id mismatch) that made the
modbus_busy stale-lock fix (see Modbus Connectivity, above) unsafe to
reintroduce naively - a bare `continue_on_error: true` on script.
modbus_queue's release step could let a queued backlog (mode: queued,
max: 10) drain rapidly with no pacing against a connection that's still
struggling, which is plausibly what caused a past attempt at that same fix
to flood the system with commands. That fix is intentionally left alone
for now (self-healing via the existing 10s wait-timeout is considered
acceptable, see test_plan.md Test 6.3) in favour of this lower-risk
smoothing fix.

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