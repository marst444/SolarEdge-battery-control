# SolarEdge Battery Control Test Plan

This document verifies the SolarEdge Battery Control package layer by layer.

A layer is only considered verified when all critical tests have passed.

---

# Current Status

```text
Layer 1 STRONG PARTIAL PASS 🟡

    ✅ 1.1 Minimum SOC Protection
    ✅ 1.2 Maximum SOC Protection
    ⏳ 1.3 Recovery Mode

Layer 2   PARTIAL PASS 🟡 (strong)
    ✅ 2.1 EMHASS Forecast Availability
    ✅ 2.2 SOC Target Generation
    🟡 2.3 Dynamic Charge / Discharge Power

Layer 3   PARTIAL PASS 🟡
    🟡 3.1 Negative Price Curtailment
    ⏳ 3.2 Site Limit Enforcement

Layer 4A  PARTIAL PASS 🟡 (strong)
    🟡 4A.1 Charge Decision (Maintain-Zone Scenario)
    ⏳ 4A.2 Discharge Decision
    ⏳ 4A.3 Maintain Zone (Additional Scenarios)
    🟡 4A.4 Grid Charge Export Cooldown

Layer 4B  STRONG PARTIAL PASS 🟡
    ✅ 5.1 Command Generation
    ✅ 5.2 Limit Updates

Layer 4C  PARTIAL PASS 🟡
    ✅ 6.1 Modbus Queue Serialization
    🟡 6.2 Concurrent Write Protection
    🟡 6.3 Recovery After Failed Write

Layer 4D  STRONG PARTIAL PASS 🟡
    ✅ 7.1 Storage Mode Apply
    ✅ 7.2 Charge Limit Apply
    ✅ 7.3 Discharge Limit Apply
    🟡 7.4 Mode Change Verification
    🟡 7.5 Modbus Long-Term Stability
    🟡 7.6 Mode Command Guard

System    PARTIAL PASS 🟡

```
Observation:

Dynamic charge/discharge limits and requested charge/discharge limits
do not appear to have a direct 1:1 relationship.

Future documentation should clarify the intended control chain.

```

---

# Test Status Definitions

| Status | Meaning |
|----------|----------|
| PASS | Functionality verified and behaves as expected |
| PARTIAL PASS | Partially verified, additional testing required |
| FAIL | Verified and not working correctly |
| NOT VERIFIED | Test not yet performed |
| BLOCKED | Cannot currently be tested |

---

# Layer 1 - Safety and Watchdog

Layer 1 = STRONG PARTIAL PASS 🟡

Verified:

```text
1.1 Minimum SOC Protection
1.2 Maximum SOC Protection
```

Outstanding:

```text
1.3 Recovery Mode
```

---

## Test 1.1 Minimum SOC Protection

Status: PASS ✅

Date: 2026-08-06

### Purpose

Verify battery discharge is blocked when SOC falls below configured minimum SOC.

### Observed

#### Enter Recovery

```
Current SOC: 54.44%
Minimum SOC: 65.0%

SOC < Min:
True

Effective Mode:
charge_from_solar_and_grid

Effective Charge Limit:
5000W

Effective Discharge Limit:
0W

Reason:
SOC below minimum - dynamic recovery charge
```

#### Exit Recovery

```
Current SOC: 54.44%
Minimum SOC: 20.0%

SOC < Min:
False

Effective Mode:
maximize_self_consumption

Effective Charge Limit:
5000W

Effective Discharge Limit:
0W

Reason:
Normal - effective follows requested
```
### Result
```text

PASS
```

### Notes

```text
Minimum SOC protection activated correctly when SOC fell below
the configured minimum threshold.

Battery discharge was blocked and dynamic recovery charging
was enabled.

When the minimum SOC threshold was restored below the current SOC,
the system exited recovery mode and returned to normal control.

Layer 1 override behaviour was verified in both directions
(enter and exit recovery state).
```

---


## Test 1.2 Maximum SOC Protection

Status: PASS ✅

Date: 2026-08-06

### Purpose

Verify charging is blocked when SOC exceeds configured maximum SOC.

### Design

Upper SOC protection uses High SOC Hold with hysteresis.

Enter:

```text
SOC >= Maximum SOC
```

Exit:

```text
SOC <= Maximum SOC - 3%
```

### Observed

#### Enter Hold

```text
Current SOC: 54.44%
Maximum SOC: 40.00%

High SOC Hold: on

Effective Mode: maximize_self_consumption
Effective Charge Limit: 0W
Effective Discharge Limit: 53W

Reason:
High SOC hold - charge blocked with hysteresis
```

#### Exit Hold

```text
Current SOC: 54.44%
Maximum SOC: 80.00%

High SOC Hold: off

Effective Mode: maximize_self_consumption
Effective Charge Limit: 5000W
Effective Discharge Limit: 7W

Reason:
Normal - effective follows requested
```

### Result

```text
PASS
```

### Notes

```text
High SOC Hold entered correctly when SOC exceeded Maximum SOC.

Charging was blocked while preserving normal self-consumption mode.

High SOC Hold exited correctly when Maximum SOC was increased and
SOC no longer met the entry condition.

Normal control logic was restored.

The previous forced export-discharge strategy has been replaced by
a hysteresis-based charge hold state.

### Observation 2026-08-06

High SOC ping-pong was not observed after introducing High SOC Hold.

However, SOC continued rising after crossing 90% and reached 94.44%.
Charge limit was set to 0 only after SOC had already reached the upper range.

This indicates that High SOC Hold should enter earlier than the configured
maximum SOC to compensate for SOC quantization and apply latency.

Proposed adjustment:

Enter High SOC Hold at:
SOC >= Maximum SOC - 1%

Exit High SOC Hold at:
SOC <= Maximum SOC - 3%

```

---

## Test 1.3 Recovery Mode

Status: NOT VERIFIED

### Purpose

Verify critical low SOC recovery behaviour.

### Preconditions

```text
SOC < 10%
```

### Verify

```text
charge_from_solar_and_grid
effective_charge_limit > 0
effective_discharge_limit = 0
```

### Pass

```text
Battery enters forced recovery charging mode.
```

### Observed

```text
-
```

### Result

```text
-
```

### Notes

```text
-
```

---

# Layer 2 - Optimisation

Layer 2 = PARTIAL PASS 🟡

Verified:

```text
✅ 2.1 EMHASS Forecast Availability
✅ 2.2 SOC Target Generation
🟡 2.3 Dynamic Charge / Discharge Power
```

Outstanding:

```text
⏳ Verify discharge-path behaviour
⏳ Verify dynamic limits when target SOC < current SOC
⏳ Verify forecast-loss fallback behaviour
```
```

---

## Test 2.1 EMHASS Forecast Availability

✅ Status: PASS

### Purpose

Verify MPC and Day-Ahead forecasts are generated and available.

### Verify

```text
sensor.mpc_pv_batt_soc available
sensor.mpc_pv_batt_power available
sensor.mpc_pv_grid_power available
```

### Pass

Forecast sensors populated with valid optimisation data.
```

Status: PASS
Date: 2026-07-30

Observed:
MPC Optim Status = Optimal
DH Optim Status = Optimal

MPC Batt SOC = 42.98
MPC Batt Power = 433.50
MPC Grid Power = 0.00

MPC Batt SOC Forecast Entries = 192
MPC Batt Power Forecast Entries = 192

Last MPC Success = 2026-07-30 01:15:11
Last DayAhead Success = 2026-07-30 00:00:23

Result:
PASS

Notes:
EMHASS optimisation functioning correctly.
Forecast data published and available.
MPC and Day-Ahead runs both completed successfully.
```

---


## Test 2.2 SOC Target Generation
```
Test 2.2 = PASS ✅
✅ SOC target updates from MPC forecast
✅ SOC target remains within configured min/max SOC
✅ MPC forecast data exists
✅ Correct smoothing entity identified: sensor.soc_batt_forecast_smooth
✅ Forecast → smoothing → target chain verified
✅ SOC target follows forecast changes over time
✅ SOC target reacts correctly to new MPC/smooth updates
⏳ SOC target clamping at min/max boundaries
⏳ Fallback behaviour when MPC forecast is missing
```

Date: 2026-07-30

### Purpose
```
Verify SOC target follows EMHASS battery forecast.
```

### Verify

```text
input_number.soc_target updated from forecast
```
Result:
PARTIAL Pass

```text
SOC target follows optimisation forecast.
```

### Observed

```text
SOC Target = 42.98

SOC Forecast Smooth = 42.98

Raw MPC SOC = 42.98

Minimum SOC = 20.0
Maximum SOC = 90.0

Forecast Entries = 192

Next Forecast Entry:
2026-07-30T01:15:00+02:00
SOC = 42.98

Current SOC = 55.56%
SOC Target = 41.71%
SOC Forecast Smooth = unknown! wrong entitiy id!
MPC SOC Forecast = 56.65%

Minimum SOC = 20.0%
Maximum SOC = 90.0%

Target >= Minimum = True
Target <= Maximum = True

### Observed

SOC Target = 60.98
SOC Batt Forecast Smooth = 59.68
MPC Batt SOC = 56.65

MPC battery_scheduled_soc attribute exists = True
MPC battery_scheduled_soc entries = 192

## Test 2.2 SOC Target Generation

Status: PASS ✅

Date: 2026-08-06

### Purpose

Verify SOC target follows the smoothed EMHASS battery SOC forecast and remains within configured SOC safety limits.

### Observed

#### Observation 1

```text
Timestamp = 2026-08-06 09:31

SOC Target = 61.07
SOC Batt Forecast Smooth = 60.69
MPC Batt SOC = 60.31

Target - Smooth = +0.38
Smooth - MPC = +0.38
Target - MPC = +0.76

Minimum SOC = 20.0
Maximum SOC = 90.0

Smooth valid = True
MPC valid = True
Target within min/max = True

MPC forecast attribute exists = True
MPC forecast entries = 192
```

#### Observation 2

```text
Timestamp = 2026-08-06 21:28

SOC Target = 84.74
SOC Batt Forecast Smooth = 85.22
MPC Batt SOC = 86.36

Target - Smooth = -0.48
Smooth - MPC = -1.14
Target - MPC = -1.62

Minimum SOC = 20.0
Maximum SOC = 90.0

Smooth valid = True
MPC valid = True
Target within min/max = True

MPC forecast attribute exists = True
MPC forecast entries = 192
```

### Result

```text
PASS
```

### Notes

```text
SOC target generation was verified against the correct smoothing entity:

sensor.soc_batt_forecast_smooth

An earlier test referenced an incorrect entity:

sensor.soc_forecast_smooth

which returned unknown.

The correct smoothing sensor was available and derived from the
battery_scheduled_soc forecast attribute exposed by the EMHASS MPC forecast.

SOC Target closely followed SOC Batt Forecast Smooth across multiple
observations separated in time.

Observed Target - Smooth differences:

+0.38 percentage points
-0.48 percentage points

SOC Target remained within configured minimum and maximum SOC limits
during all observations.

MPC forecast data was available and contained 192 forecast entries.

The verification confirms the forecast processing chain:

MPC battery SOC forecast
→ SOC Batt Forecast Smooth
→ SOC Target

SOC target generation is therefore considered verified.

### Update 2026-08-27 - Volatility observed between successive MPC runs

While investigating a separate uneconomical charge/discharge report (see
roadmap.md - Active Investigations - Midday Charge/Discharge Oscillation),
history for input_number.soc_target and sensor.soc_batt_forecast_smooth
on 2026-08-27 showed swings of 10-40 percentage points between reads only
1-2 minutes apart (e.g. soc_target: 45.02 at 11:00:01, then 24.44 at
11:02:00). This does not contradict the forecast -> smoothing -> target
chain verified above (the chain still faithfully follows whatever the
latest MPC run publishes) - it indicates the underlying MPC plan itself
is volatile run-to-run, which "SOC Target Generation" as tested here does
not cover. Flagged as a follow-up item, not a regression of this test.
```

---

## Test 2.3 Dynamic Charge / Discharge Power
✅ Forecast ger ett rimligt charge limit-värde
 ✅ Forecast ger 0 discharge när mål-SOC ligger över aktuell SOC
 ✅ Sensor → helper-synk fungerar
 ✅ Last-value-hjälparna uppdateras
 ✅ Discharge direction: SOC > Target → discharge > 0, charge = 0
 ✅ Sensor → helper sync för discharge
 ✅ Maintain-zone ger liten discharge
 ⏳ Charge direction: SOC < Target → charge > 0, discharge = 0
 ⏳ Last Charge/Discharge Limit helpers, om de fortfarande ska användas
 ⏳ Scenario där discharge limit blir positiv
 ⏳ Scenario där target SOC sjunker under aktuell SOC
 ⏳ Fallback till last_* när forecast saknas
Status: PARTIAL PASS 🟡

### Purpose

Verify dynamic charge and discharge limits are calculated from battery forecasts.

### Verify

```text
sensor.dynamic_storage_charge_limit updated
sensor.dynamic_storage_discharge_limit updated
```

### Pass

```text
Dynamic limits follow forecasted battery SOC trajectory.
```

Status: PARTIAL PASS
Date: 2026-07-30

Observed:

Current SOC = 43.33%

SOC Target = 44.4%

Dynamic Charge Limit = 49W
Dynamic Discharge Limit = 0W

Helper Dynamic Charge = 49W
Helper Dynamic Discharge = 0W

Last Charge Limit = 49W
Last Discharge Limit = 0W

Result:

PARTIAL PASS

Notes:

- Charge limit correctly became positive when target SOC exceeded current SOC.
- Discharge limit correctly remained zero.
- Sensor to helper propagation verified.
- Discharge-path behaviour still needs verification.
- Forecast-loss fallback behaviour still needs verification.

Charge-path behaviour verified:

Current SOC = 43.33%
SOC Target = 44.40%

Dynamic Charge Limit = 49W
Dynamic Discharge Limit = 0W

The calculated limits were consistent with the forecast direction:
target SOC was above current SOC, resulting in a positive charge limit
and zero discharge limit.

Verified:
- Forecast → Dynamic Charge Limit chain
- Sensor → Helper propagation
- Last-value helpers updated correctly
SOC > Target
→ positiv discharge-limit

SOC < Target
→ positiv charge-limit

Outstanding:
- Verify scenario where target SOC falls below current SOC
- Verify positive discharge-limit generation
- Verify forecast-loss fallback behaviour

Current assessment:
PARTIAL PASS is appropriate until both charge and discharge paths
have been observed and verified.
```
#### Observation - Discharge Direction / Maintain Zone

```text
Timestamp = 2026-08-06 21:37

Current SOC = 86.67%
SOC Target = 84.66%
Difference Target - Current = -2.01%
SOC Deadband = 3.33%

SOC > Target = True
SOC ~= Target = True

Dynamic Charge Limit = 0W
Dynamic Discharge Limit = 92W

Helper Dynamic Charge = 0W
Helper Dynamic Discharge = 92W
```
### Notes
```
Dynamic discharge path was verified.

When current SOC was above SOC Target, dynamic charge limit was zero and
dynamic discharge limit became positive.

Because the SOC difference was still within the configured deadband,
the resulting discharge limit was small, consistent with maintain-zone
behaviour.

Sensor values and helper values matched:

Dynamic Discharge Limit = 92W
Helper Dynamic Discharge = 92W

Last-value helpers were unavailable / unknown during this observation and
should be investigated separately if they are expected to be populated.
```

---

# Layer 3 - Grid Constraints

Layer 3 = PARTIAL PASS 🟡

Verified:

```text
-
```

Outstanding:

```text
3.2 Site Limit Enforcement
```

---

## Test 3.1 Negative Price Curtailment

Status: PARTIAL PASS
Date: 2026-07-30

### Purpose

Verify export is curtailed when export price becomes negative.

### Observed

```text
Export Price = 0.15611

Negative Price Active = off
Grid Export Blocked = off
Grid Export Limited = off
Negative Site Limit = off

Site Limit = 0
```

### Result

```text
PARTIAL PASS
```

### Notes

```text
Normal-price behaviour verified.

When export price was positive:

- negative_price_active remained off
- grid_export_blocked remained off
- negative_site_limit remained off

Outstanding:

- Negative-price activation path
- Export blocking verification
- Site-limit enforcement verification
```
```

---

## Test 3.2 Site Limit Enforcement

Status: NOT VERIFIED

### Purpose

Verify SolarEdge site limit is applied during export curtailment.

### Preconditions

```text
Negative price curtailment active
```

### Verify

```text
site_limit = 0
negative_site_limit enabled
export reduced
```

### Pass

```text
SolarEdge site limit correctly restricts export.
```

### Observed

```text
-
```

### Result

```text
-
```

### Notes

```text
-
```

---

# Layer 4A - Decision Engine
```
Layer 4A = PARTIAL PASS 🟡

Verified: 
🟡 4A.1 Charge Decision (Maintain-Zone Observation) 
✅ 4A.3 Maintain Zone O
utstanding: 
⏳Charge scenario ⏳
4A.2 Discharge Decision
🟡 4A.4 Grid Charge Export Cooldown
```

---

## Test 4A.1 Charge Decision
PARTIAL PASS


### Purpose

Verify requested charging behaviour follows forecasts and SOC conditions.

### Verify

```text
emhass_requested_storage_mode correct
emhass_requested_charge_limit correct
```

### Pass

```text
Charging decision matches forecast and current state.
```
## Test 4A.1 Charge Decision

Status: PARTIAL PASS
Date: 2026-07-30

### Purpose

Verify requested charging behaviour follows forecasts and SOC conditions.

### Observed

```text
Current SOC = 41.11%
SOC Target = 41.10%

PV Production = 0 W
House Load = 383 W

MPC Batt Power = 0 W
MPC Grid Power = 470.5 W

Requested Storage Mode = maximize_self_consumption

Requested Charge Limit = 0 W
Requested Discharge Limit = 1 W
```

### Result

```text
PARTIAL PASS
```

### Notes

```text
Decision engine selected maximize_self_consumption
when SOC was effectively equal to target SOC.

No charging request was generated.

No significant discharge request was generated.

Behaviour was consistent with maintain-zone logic.

Charge-path behaviour still requires verification under
conditions where charging is clearly required.

SOC < Target
PV surplus
Charge request tillåten
Discharge blockerad
```


---


## Test 4A.2 Discharge Decision
Layer 4A = PARTIAL PASS 🟡
⚠ Investigate maintain-zone discharge behaviour
```
Status: PARTIAL PASS
Date: 2026-07-30

Observed:

Current SOC = 45.56%
Target SOC = 43.96%

Difference = +1.60%

Dynamic Discharge Limit = 73W

Requested Storage Mode = maximize_self_consumption
Requested Charge Limit = 5000W
Requested Discharge Limit = 0W

Result:

PARTIAL PASS
`
```

---

## Test 4A.3 Maintain Zone
```

Status: PASS
Date: 2026-07-30

### Purpose

Verify battery remains near SOC target when inside deadband.

### Observed

```text
Current SOC = 41.11%

SOC Target = 41.10%

SOC Deadband = 3.33%

Difference = -0.01%

Requested Storage Mode = maximize_self_consumption

Requested Charge Limit = 0W
Requested Discharge Limit = 1W

Effective Storage Mode = maximize_self_consumption

Effective Charge Limit = 0W
Effective Discharge Limit = 1W

Battery Reason =
Normal - effective follows requested

Result:
PASS

Notes
SOC was effectively equal to target SOC.

System entered maintain-zone behaviour.

No meaningful charge request was generated.
No meaningful discharge request was generated.

Requested and effective decisions matched.

Behaviour was consistent with maintain-zone logic.

SOC ≈ Target

och

SOC inom deadband men under target
```


---

## Test 4A.4 Grid Charge Export Cooldown

Status: PARTIAL PASS

### Purpose

Verify that after `automation.emhass_battery_forecast_control` requests
`charge_from_solar_and_grid`, it does not request
`discharge_to_maximize_export` again until
`input_number.grid_charge_export_cooldown_minutes` (default 60) has
elapsed - preventing a grid-assisted charge from being sold straight back
out at a loss, as observed on 2026-08-27 (see roadmap.md - Active
Investigations - Midday Charge/Discharge Oscillation, and
architecture.md - Layer 4A - Grid Charge Export Cooldown).

### Design

```text
charge_from_solar_and_grid branches stamp
input_datetime.last_grid_charge_command_time = now()

DISCHARGE MAX EXPORT branch:
  if (maintain or above) and soc > soc_min and grid_fc < 0 and batt_fc > 0:
    if cooldown_elapsed (now - last_grid_charge_command_time >=
       grid_charge_export_cooldown_minutes):
      -> discharge_to_maximize_export (unchanged behaviour)
    else:
      -> maximize_self_consumption with dynamic discharge limit
         (held back, does not force export)
```

### Preconditions

```text
input_number.grid_charge_export_cooldown_minutes = 60 (default)
```

### Verify

```text
a) charge_from_solar_and_grid fires
   -> input_datetime.last_grid_charge_command_time updates to the trigger
      time

b) Within cooldown window, conditions for DISCHARGE MAX EXPORT become true
   (maintain/above, soc > soc_min, grid_fc < 0, batt_fc > 0)
   -> emhass_requested_storage_mode = maximize_self_consumption (NOT
      discharge_to_maximize_export)
   -> emhass_requested_discharge_limit = dynamic_discharge_limit (not
      forced to export aggressively, but not blocked either)

c) After cooldown window elapses, same DISCHARGE MAX EXPORT conditions
   -> emhass_requested_storage_mode = discharge_to_maximize_export
      (normal behaviour resumes)

d) No charge_from_solar_and_grid in recent history (cooldown "cold")
   -> DISCHARGE MAX EXPORT behaves exactly as before this change
      (input_datetime defaults far in the past, so cooldown_elapsed = true)
```

### Pass

```text
discharge_to_maximize_export is never requested within
grid_charge_export_cooldown_minutes of a charge_from_solar_and_grid
request; normal max-export behaviour resumes unchanged once the cooldown
elapses; behaviour is unaffected when no recent grid charge occurred.
```

### Observed

```text
Live evaluation 2026-08-28, via HA MCP (state, history, automation
traces) after the user copied the updated battery_forecast_control.yaml
and batterycontrol_helpers.yaml to the live config and reloaded
(08:09:06 local).

Reload verification:
- System log shows a clean "Initialized trigger EMHASS Forecast Driven
  Battery Control" at 08:09:06, no validation errors.
- input_number.grid_charge_export_cooldown_minutes = 60 (default), live.
- input_datetime.last_grid_charge_command_time initialized to today
  00:00:00 (HA's default for an input_datetime with no explicit
  `initial:` - not the assumed 1970 epoch, but still far enough in the
  past that cooldown_elapsed = true from first load, as intended).
- sensor.battery_mode_command_guard (previously missing, see Test 7.6)
  now also exists and reads "guarded" - the same reload picked up the
  fix for that gap too.

Case (d) - normal behaviour when no recent grid charge - confirmed live:
trace run_id 48a52ba6f9cdb0173903d8460cb9007d (2026-08-28T07:15:01 UTC):

  soc = 51.11, target = 40.01, db = 3.33 -> above = true
  grid_fc = -2075.49, batt_fc = 1889.99
  cooldown_elapsed = true   <- new variable present and correctly
                                evaluated, confirming the updated code is
                                live (not the pre-reload version - an
                                earlier trace checked right after reload
                                turned out to predate it and had no
                                cooldown_elapsed at all)

action/1/choose/2 (DISCHARGE MAX EXPORT) matched, its nested
choose/0 (cooldown_elapsed) matched, producing:
  emhass_requested_storage_mode = discharge_to_maximize_export
  emhass_requested_discharge_limit = 643W
Exactly the unchanged/normal behaviour expected with no recent grid
charge - confirms the nested choose/default restructuring did not
regress the existing max-export path.

Cases (a), (b), (c) - stamping on a real charge_from_solar_and_grid, and
the resulting hold-back / resume - NOT yet observed: no
charge_from_solar_and_grid has fired since the reload (input_select.
emhass_requested_storage_mode history since 00:00 today shows only
maximize_self_consumption -> discharge_to_maximize_export at 08:15;
input_datetime.last_grid_charge_command_time is still at its untouched
default). Needs a real grid-charge event to exercise the stamp-and-hold
path end to end.
```

### Result

```text
PARTIAL PASS - config deployed cleanly and verified live; the
cooldown-inactive/default path (case d) is confirmed correct via a real
trace. The actual hold-back behaviour (cases a-c, the part that fixes
the reported issue) is implemented but not yet exercised by a real
charge_from_solar_and_grid event - needs a follow-up check next time one
occurs.
```

### Notes

```text
Implemented 2026-08-27 in battery_forecast_control.yaml and
batterycontrol_helpers.yaml, in response to the charge-then-immediately-
export pattern observed the same day. Deployed and reloaded 2026-08-28.
Bonus: the same reload also resolved the missing sensor.
battery_mode_command_guard gap from Test 7.6.
```

---

# Layer 4B - Command Generation

Layer 4B = STRONG PARTIAL PASS 🟡

Verified:

```text
✅ 5.1 Command Generation
✅ 5.2 Limit Updates
```

Outstanding:

```text
-
```

---

## Test 5.1 Command Generation
```
Status: PASS
Date: 2026-07-30

Observed:

- Automation triggered by input_number.effective_discharge_limit
- effective_mode = maximize_self_consumption
- effective_charge = 0W
- effective_discharge = 1W
- Correct mode branch selected:
  script.maximize_self_consumption_script
- Charge limit script executed:
  script.set_effective_storage_charge_limit
- Discharge limit script executed:
  script.set_effective_storage_discharge_limit

Result:

PASS

Notes:

- Effective battery decision was translated into the expected command scripts.
```

---

## Test 5.2 Limit Updates

Status: PARTIAL PASS

### Purpose

Verify effective charge/discharge limits are translated into command requests.

### Verify

```text
Correct charge limit command
Correct discharge limit command
```

### Pass

```text
Requested limits correctly translated into SolarEdge commands.
```
Status: PARTIAL PASS
Date: 2026-07-30

Observed:

- effective_charge = 0W
- effective_discharge = 1W
- script.set_effective_storage_charge_limit executed
- script.set_effective_storage_discharge_limit executed
- number.solaredge_i1_storage_discharge_limit updated to 1W

Result:

PASS

Notes:

- Discharge limit apply path verified.
- Charge limit script executed; at the time of this original run the value
  was already 0W so no state change was directly observed in-trace.

### Update 2026-08-28 - Non-zero charge limit update closed out

Using the same live history evidence gathered for Test 7.2 (21 sampled
non-zero input_number.effective_charge_limit changes, all correctly
propagated to number.solaredge_i1_storage_charge_limit via
script.set_effective_storage_charge_limit -> script.modbus_queue within
~15-55s, 2026-08-26 through 2026-08-28), the previously outstanding
"need verify a non-zero charge limit update" note is closed. Both the
charge and discharge limit command-generation paths are now confirmed
working for zero and non-zero values. Upgraded PARTIAL PASS -> PASS.
```

---

# Layer 4C - Modbus Queue

Layer 4C = PARTIAL PASS 🟡

Verified:
✅ 6.1 Modbus Queue Serialization
🟡 6.2 Concurrent Write Protection
🟡 6.3 Recovery After Failed Write


Outstanding:

```text
-
```

---

## Test 6.1 Modbus Queue Serialization

Status: PASS

### Purpose

Verify only one Modbus write executes at a time.

### Verify

```text
modbus_busy works correctly
No overlapping writes
```

### Pass

```text
Commands are serialized correctly.
```
Status: PASS
Date: 2026-07-30

Observed:

- Multiple commands were sent through script.modbus_queue.
- input_boolean.modbus_busy turned on during queue execution.
- input_boolean.modbus_busy returned to off after each queued command.
- Commands executed sequentially:
  - set_storage_command_timeout
  - maximize_self_consumption
  - set_storage_charge_limit
  - set_storage_discharge_limit

Result:

PASS

Notes:

- Queue serialization worked for this command sequence.
- No stuck modbus_busy observed.
```

---

## Test 6.2 Concurrent Write Protection

Status: NOT VERIFIED

### Purpose

Verify simultaneous command requests are serialized.

### Verify

```text
Commands execute sequentially
No race conditions
```

### Pass

```text
Concurrent command requests are safely serialized.
```
Status: PARTIAL PASS
Date: 2026-07-30

Observed:

- Several commands were fired from one automation run.
- Commands were handled sequentially through script.modbus_queue.
- modbus_busy was used as lock.

Result:

PARTIAL PASS

Notes:

- Sequential command handling verified.
- True simultaneous external write collision not yet tested.
```

---

## Test 6.3 Recovery After Failed Write

Status: PARTIAL PASS

### Purpose

Verify queue recovers after failed Modbus operations.

### Verify

```text
modbus_busy released
Subsequent commands continue working
```

### Pass

```text
Queue recovers without manual intervention.
```

### Observed

```text
Live evaluation 2026-08-28, via HA MCP (logs + history), using a real
failure found in the 48h error log.

Incident: 2026-08-27 06:45:00.708 - input_boolean.modbus_busy turned on
by script.modbus_queue (dispatching a discharge-limit write, effective_
discharge_limit had just changed to 63W). At 06:45:06.02 the write
failed:

  custom_components.solaredge_modbus_multi.hub: "Connection failed:
  Modbus Error: [Connection] Not connected"
  automation.apply_effective_battery_control_to_solaredge_inverter:
  "Connection to inverter ID 1 failed."

Root cause of the "stuck" symptom, confirmed by reading solaredge_
modbusqueue.yaml directly: script.modbus_queue's final step (input_
boolean.turn_off modbus_busy) is a plain last step in a linear sequence,
not wrapped in a try/finally-equivalent. When a service call inside the
choose: block raises (as it did here), the script aborts and that final
turn_off never runs - modbus_busy is left "on".

input_boolean.modbus_busy history confirms it then sat "on" for ~30
minutes (06:45:00.708 -> 07:15:24.68), because no further state change
on effective_storage_mode/effective_charge_limit/effective_discharge_
limit occurred in that window to re-trigger the apply automation - there
was simply nothing else to send, not a deeper hang.

Recovery mechanism: script.modbus_queue's own guard -
  wait_template: "{{ not is_state('input_boolean.modbus_busy','on') }}"
  timeout: "00:00:10"
  continue_on_timeout: true
means the NEXT call to modbus_queue only waits up to 10s for a stale
lock before proceeding anyway (rather than deadlocking forever). This is
exactly what happened at 07:15: effective_storage_mode and effective_
discharge_limit both changed at 07:15:00.6-0.7, the apply automation
fired, and modbus_busy history shows it forced through and then resumed
completely normal on/off cycling (07:15:24 onward) for the rest of the
observed period. The 07:15 automation.emhass_battery_forecast_control
trace also completed cleanly, and select.solaredge_i1_storage_command_
mode correctly transitioned to Discharge to Maximize Export at 07:15:27.

So: the lock is not released promptly by the failure itself (a real gap
- a raised exception skips the cleanup step), but the system is
self-healing on the next command via the 10-second timeout fallback, and
that recovery was directly observed working. No manual intervention was
needed; no command appears to have been silently and permanently lost.
```

### Result

```text
PARTIAL PASS - subsequent commands did continue working without manual
intervention (confirmed), but "modbus_busy released" is not immediate on
failure - it stayed stuck for ~30 minutes until the next state change
happened to trigger another script.modbus_queue call, which force-cleared
it via its own 10s wait-timeout. The design tolerates this (worst case
~10s extra wait on the next real command), but a write failure does
directly cause a stale lock, which is worth fixing at the source.
```

### Notes

```text
Suggested improvement for roadmap.md: wrap script.modbus_queue's choose:
block so the "Release Modbus lock" step always runs (e.g. continue_on_
error: true on the choose action, or restructure the final turn_off as a
sequence step that isn't skipped by an upstream exception), so a failed
write releases the lock immediately rather than relying on the next
caller's 10-second timeout to force through. Logged in roadmap.md -
Active Investigations - Modbus Connectivity.
```

---

# Layer 4D - SolarEdge Apply

Layer 4D = STRONG PARTIAL PASS 🟡

Verified:

✅ 7.1 Storage Mode Apply
✅ 7.2 Charge Limit Apply
✅ 7.3 Discharge Limit Apply

Outstanding:

```text
🟡 7.4 Mode Change Verification (3 of 5 modes confirmed; 2 not exercised in 7d window)
🟡 7.5 Modbus Long-Term Stability (no lockup, but unexplained baseline Modbus noise)
🟡 7.6 Mode Command Guard (case c not yet tested; anomalous 12s flip unexplained)
```

---

## Test 7.1 Storage Mode Apply

Status: PASS

### Purpose

Verify storage mode commands are applied to SolarEdge.

### Verify

```text
select.solaredge_i1_storage_command_mode updated
```

### Pass

```text
SolarEdge storage mode matches requested mode.
```
```


Status: PASS
Date: 2026-07-30

### Observed

```text
Effective Storage Mode = maximize_self_consumption
Storage Command Mode = Maximize Self Consumption
Storage Control Mode = Remote Control
Storage Command Timeout = 900
```
```
Notes
SolarEdge storage command mode matched the effective storage mode.
Remote Control mode was active.
```


---

## Test 7.2 Charge Limit Apply

Status: PASS ✅

### Purpose

Verify charge limit commands are applied to SolarEdge.

### Verify

```text
number.solaredge_i1_storage_charge_limit updated
```

### Pass

```text
Charge limit correctly applied.
```

### Observed (2026-07-30, zero-limit path)

```text
Effective Charge Limit = 0W
Storage Charge Limit = 0W
```

### Observed (2026-08-28, non-zero path, live via HA MCP history)

```text
input_number.effective_charge_limit vs number.solaredge_i1_storage_charge_limit,
2026-08-26 through 2026-08-28, significant_changes_only history, cross-matched
by timestamp. Every non-zero effective_charge_limit change was followed by a
matching number.solaredge_i1_storage_charge_limit update within ~15-55 seconds
(the delay is the modbus_queue serialization + scan interval, consistent with
Test 6.1/7.3 timing):

  20W, 24W, 50W, 53W, 95W, 102W, 114W, 127W, 195W, 292W, 576W, 597W, 681W,
  703W, 813W, 985W, 1050W, 1081W, 1084W, 3000W, 5000W

All 21 sampled non-zero values propagated correctly with matching final
values (no mismatches, no missed updates, no stale values left behind after
a change).
```

### Result

```text
PASS
```

### Notes

```text
Charge limit apply path fully verified across both the zero-limit case
(2026-07-30) and a wide range of non-zero values (2026-08-28), including
small values (20-127W), mid-range (195-1084W), and the two ceiling values
used by Layer 1 recovery/High SOC Hold logic (3000W, 5000W). Upgraded from
PARTIAL PASS to PASS; the previously outstanding "non-zero charge limit"
gap is closed.
```

---

## Test 7.3 Discharge Limit Apply

Status: PASS

### Purpose

Verify discharge limit commands are applied to SolarEdge.

### Verify

```text
number.solaredge_i1_storage_discharge_limit updated
```

### Pass

```text
Discharge limit correctly applied.
```
Status: PASS
Date: 2026-07-30

Observed:

Run 1:
- Effective Discharge Limit = 1W
- Storage Discharge Limit = 1W

Run 2:
- Effective Discharge Limit = 4W
- Storage Discharge Limit = 4W

- script.set_effective_storage_discharge_limit executed
- script.modbus_queue executed set_storage_discharge_limit

Result:

PASS

Notes:

- Discharge limit command successfully reached SolarEdge Modbus Multi entity.
- Discharge limit command successfully reached SolarEdge and matched the effective discharge limit.
```

---

## Test 7.4 Mode Change Verification

Status: PARTIAL PASS 🟡

### Purpose

Verify that each storage mode is correctly applied.

### Modes

```text
Solar Power Only
Charge From Clipped Solar
Charge From Solar And Grid
Discharge To Maximize Export
Maximize Self Consumption
```

### Pass

```text
Each requested mode results in the correct SolarEdge command mode.
```

### Observed

```text
Live evaluation 2026-08-28, via HA MCP 7-day history (104 entries) on
select.solaredge_i1_storage_command_mode, cross-referenced against
input_select.effective_storage_mode over the same window.

Confirmed correctly applied (input_select value -> matching SolarEdge
select value, each transition observed multiple times over the 7 days):

  maximize_self_consumption   -> "Maximize Self Consumption"
  discharge_to_maximize_export -> "Discharge to Maximize Export"
  charge_from_solar_and_grid  -> "Charge from Solar Power and Grid"

Never observed in the 7-day window (neither as an effective_storage_mode
request nor as a SolarEdge command mode):

  solar_power_only            -> "Solar Power Only"
  charge_from_clipped_solar   -> "Charge From Clipped Solar"

This is a coverage gap in the observation period, not evidence of a
defect - the decision engine (Layer 4A) simply never selected these two
modes under the conditions encountered over the last 7 days. Both modes
are legitimate options in effective_storage_mode's input_select and are
handled by named branches in battery_forecast_control.yaml and the
apply/command-generation scripts; nothing in the reviewed logic suggests
they would fail to apply if selected.
```

### Result

```text
PARTIAL PASS - 3 of 5 modes confirmed correctly applied over a 7-day
live window (Maximize Self Consumption, Discharge to Maximize Export,
Charge from Solar Power and Grid). The remaining 2 modes (Solar Power
Only, Charge From Clipped Solar) were not exercised by real conditions
in this window and remain unverified - not because of any known issue,
but purely a testing-coverage gap.
```

### Notes

```text
To fully close this test, either wait for natural conditions that would
trigger Solar Power Only / Charge From Clipped Solar (see
battery_forecast_control.yaml for their trigger conditions), or manually
force effective_storage_mode through each of the two untested options
and confirm the corresponding SolarEdge command mode updates.
```

---

## Test 7.5 Modbus Long-Term Stability

Status: PARTIAL PASS 🟡

### Purpose

Verify stable queue operation during extended runtime.

### Observation Window

```text
24h+
```

### Verify

```text
No queue lockups
No repeated transaction_id mismatch storms
No runaway retries
No stuck modbus_busy
```

### Pass

```text
System remains operational throughout the observation period.
```

### Observed

```text
Live evaluation 2026-08-28, via HA MCP structured error_log (48h window,
deduplicated/counted by issue).

  06:45:06 write-failure cluster ("Connection failed: Modbus Error:
    [Connection] Not connected" / "Connection to inverter ID 1 failed."):
    1 occurrence - see Test 6.3 for full incident analysis.
  "Cancel send": 8 occurrences
  "Cancel send" / "Repeating call" / "No response": 27 occurrences
    (transient, self-recovering Modbus retry chatter)
  transaction_id mismatch: 7 occurrences
  coordinator update/fetch failure: 1 occurrence
  (plus unrelated third-party API errors not related to Modbus/SolarEdge)

No indefinite queue lockup was observed - script.modbus_queue's own
10-second wait_template/continue_on_timeout guard (see Test 6.3) forces
through any stale input_boolean.modbus_busy on the next invocation, and
this was directly confirmed working during the 06:45-07:15 incident.
input_boolean.modbus_busy history over the 48h window otherwise shows
normal, brief on/off cycling consistent with individual queued commands,
not a runaway or storm pattern. Error counts (27 "Cancel send/No
response", 7 transaction_id mismatches) are non-zero but modest for a
48h window and did not visibly disrupt control-loop behaviour - Test 6.3
and Test 7.4's mode-transition history both show clean end-to-end
command delivery across the same period.
```

### Result

```text
PARTIAL PASS - no permanent lockup, no runaway retry storm, and the
self-healing stale-lock recovery was directly confirmed (see Test 6.3).
However, the error log does show a non-trivial baseline of Modbus
connectivity noise (27 "Cancel send/No response", 7 transaction_id
mismatches, 8 "Cancel send" over 48h) that has not been root-caused, and
only one write actually failed outright in this window. Not a clean
PASS until this baseline noise is either explained (e.g. RS485/TCP
contention, inverter response latency) or shown to have zero impact
over a longer, ideally full-week, observation window.
```

### Notes

```text
Cross-references Test 6.3 (Recovery After Failed Write) for the one
confirmed write failure and the stale-lock finding in this same window.
See roadmap.md - Active Investigations - Modbus Connectivity for the
consolidated write-up and suggested fix.
```

---

## Test 7.6 Mode Command Guard

Status: NOT VERIFIED

### Purpose

Verify that `automation.apply_effective_battery_control_to_solaredge_inverter`
correctly skips the mode-select command and 15-minute command timeout reset
when continuing in `maximize_self_consumption`, and correctly still sends
the full command sequence for any real mode transition or when the guard's
preconditions are not met.

### Preconditions

```text
input_select.effective_storage_mode = maximize_self_consumption
select.solaredge_i1_storage_command_mode = "Maximize Self Consumption"
select.solaredge_i1_storage_default_mode = "Maximize Self Consumption"
```

### Verify

Guard active (mode command skipped):

```text
sensor.battery_mode_command_guard = "guarded"
Only script.set_effective_storage_charge_limit and
script.set_effective_storage_discharge_limit run when
input_number.effective_charge_limit / effective_discharge_limit change.
select.solaredge_i1_storage_command_mode is NOT rewritten.
number.solaredge_i1_storage_command_timeout is NOT rewritten.
```

Guard inactive (full command sent) - test each of these separately:

```text
a) effective_storage_mode transitions away from maximize_self_consumption
   -> full command sequence sent (timeout reset + mode select + limits),
      sensor.battery_mode_command_guard = "active"

b) effective_storage_mode transitions back into maximize_self_consumption
   from another mode
   -> full command sequence sent once, sensor.battery_mode_command_guard
      transitions active -> guarded only after the live command mode
      confirms Maximize Self Consumption

c) select.solaredge_i1_storage_default_mode is set to something other than
   "Maximize Self Consumption" while effective mode stays
   maximize_self_consumption
   -> guard stays inactive; mode command keeps being resent every trigger;
      sensor.battery_mode_command_guard = "active"
```

### Pass

```text
Mode command is skipped only when effective mode, live command mode, and
configured default mode all agree on Maximize Self Consumption. Any
mismatch - including a non-msc configured default mode - results in the
full command sequence being sent, so no unintended control authority is
ceded to a misconfigured fallback.
```

### Observed

```text
Live evaluation 2026-08-27, via HA MCP (state, automation trace, logbook, logs).

Trace run_id 83276d1d92e8bb49a69e2f61c0082c9f (2026-08-27T08:30:00 UTC),
triggered by state of input_number.effective_discharge_limit:

  effective_mode        = maximize_self_consumption
  inverter_command_mode = "Maximize Self Consumption"
  inverter_default_mode = "Maximize Self Consumption"
  mode_guard_active      = true

action/1/if evaluated to false (guard branch taken) - the mode-select +
timeout-reset block was skipped entirely. Only
script.set_effective_storage_charge_limit and
script.set_effective_storage_discharge_limit ran. Confirms the guard
condition, the skip behaviour, and that mode_guard_active is computed
correctly from live entity state (not a cached "last sent" value).

select.solaredge_i1_storage_command_mode history over the prior 24h shows
7 real mode transitions (maximize_self_consumption <-> discharge_to_maximize_export,
one brief charge_from_solar_and_grid during a HA restart), each one
correctly triggering the full command sequence per the logbook
(context_name: "Apply Effective Battery Control to Solaredge Inverter").

One anomaly: command_mode flipped Maximize Self Consumption -> Discharge to
Maximize Export at 09:30:19 -> 09:30:31 (12s apart), while
input_select.effective_storage_mode shows no corresponding change in its
own history at that time. Not yet explained - possibly a manual/live test
during this same evaluation session rather than a control-loop issue,
since it doesn't correlate with any automation-driven effective_storage_mode
transition. Flagged for follow-up, not treated as a guard defect given the
directly-traced run above is unambiguous.

ISSUE FOUND: sensor.battery_mode_command_guard (the standalone diagnostic
added alongside the guard) does not exist on the live system -
ha_get_state and the unique_id resolver both return ENTITY_NOT_FOUND, and
it does not appear anywhere in the logs (not even a "Setting up" or
validation-error line), despite ~24 other template sensors being freshly
set up around the same reload. This points to the live
batterycontrol_sensors.yaml on the HA instance not yet containing the new
sensor block from the project doc - the automation-side guard logic is
live and correct, but the diagnostic sensor's file wasn't copied/reloaded.
```

### Result

```text
PARTIAL PASS - core guard behaviour (skip on confirmed continuation,
full sequence on real transitions) verified live. Diagnostic sensor
missing on the live instance; anomalous 12s mode flip unexplained.
```

### Notes

```text
Action needed: re-copy the current batterycontrol_sensors.yaml (with the
Battery Mode Command Guard template sensor) to the live config and reload
template entities, then re-check sensor.battery_mode_command_guard exists
and tracks "guarded"/"active" correctly.

Still outstanding: explicit test of the guard staying inactive when
select.solaredge_i1_storage_default_mode is not Maximize Self Consumption
(case c in Verify above) - not observed in this session since the live
default mode has stayed at Maximize Self Consumption throughout.

### Update 2026-08-28 - Missing sensor resolved

sensor.battery_mode_command_guard now exists on the live instance and
reads "guarded" - confirmed via ha_get_state after the user's 08:09:06
reload (the same reload that deployed the Grid Charge Export Cooldown,
Test 4A.4). The missing-sensor issue above is resolved; case (c) is
still outstanding.
```

---

# System Tests

System = PARTIAL PASS 🟡

Verified:

```text
🟡 S.1 End-to-End Charge (one real event fully traced across all 5 layers)
🟡 S.2 End-to-End Discharge (one real event fully traced across all 5 layers)
```

Outstanding:

```text
S.3 Negative Price Export Block
S.4 Modbus Stability
```

---

## Test S.1 End-to-End Charge

Status: PARTIAL PASS 🟡

### Purpose

Verify complete charging flow from EMHASS forecast to SolarEdge command.

### Flow

```text
Layer 2
→ Layer 4A
→ Layer 4B
→ Layer 4C
→ Layer 4D
```

### Pass

```text
Final inverter behaviour matches charging request.
```

### Observed

```text
Live evaluation, cross-layer trace of the 2026-08-27 ~11:00 charge event
(the same event analyzed in roadmap.md - Midday Charge/Discharge
Oscillation and used to design the Grid Charge Export Cooldown, Test
4A.4):

  Layer 2 (EMHASS): MPC batt_fc < 0 (charge favoured), grid_fc supporting
    grid-assisted charge at that timestep.
  Layer 4A (battery_forecast_control.yaml): decision engine selected
    charge_from_solar_and_grid, emhass_requested_charge_limit = 597W.
  Layer 4B (apply automation): effective_storage_mode /
    effective_charge_limit updated to match the request, command scripts
    (script.set_effective_storage_charge_limit + mode-select) triggered.
  Layer 4C (script.modbus_queue): commands serialized through the queue,
    modbus_busy cycled on/off cleanly for this event (no failure in this
    particular window).
  Layer 4D (SolarEdge): select.solaredge_i1_storage_command_mode ->
    "Charge from Solar Power and Grid", number.solaredge_i1_storage_
    charge_limit -> 597W, matching the Layer 4A request end to end.

Full chain confirmed consistent for this one real event: EMHASS forecast
-> decision -> command generation -> queue -> inverter, with the 597W
value matching at every layer.
```

### Result

```text
PARTIAL PASS - one complete real end-to-end charge event fully traced
and confirmed consistent across all 5 layers. Only a single event was
traced in this depth; broader statistical confirmation (many events,
different charge-limit magnitudes, across different times of day) would
be needed for a full PASS.
```

### Notes

```text
This same 2026-08-27 11:00 event is the one that originally motivated
the Grid Charge Export Cooldown fix (Test 4A.4) - the charge itself
applied correctly end-to-end, it was the subsequent immediate
maximize-export reversal that was the reported problem, not this
charging flow.
```

---

## Test S.2 End-to-End Discharge

Status: PARTIAL PASS 🟡

### Purpose

Verify complete discharge flow from EMHASS forecast to SolarEdge command.

### Flow

```text
Layer 2
→ Layer 4A
→ Layer 4B
→ Layer 4C
→ Layer 4D
```

### Pass

```text
Final inverter behaviour matches discharge request.
```

### Observed

```text
Live evaluation, cross-layer trace of the 2026-08-28 07:15 discharge
event (the same trace used to confirm Test 4A.4 case (d)):

  Layer 2 (EMHASS): soc = 51.11%, target = 40.01%, deadband = 3.33% ->
    above-target; grid_fc = -2075.49 (export favourable), batt_fc =
    1889.99.
  Layer 4A (battery_forecast_control.yaml, trace run_id
    48a52ba6f9cdb0173903d8460cb9007d): DISCHARGE MAX EXPORT branch
    matched; cooldown_elapsed = true (no recent grid charge) so the
    nested choose selected discharge_to_maximize_export,
    emhass_requested_discharge_limit = 643W.
  Layer 4B (apply automation): effective_storage_mode and effective_
    discharge_limit both changed at 07:15:00.6-0.7 to match the request.
  Layer 4C (script.modbus_queue): commands serialized cleanly through
    the queue for this event.
  Layer 4D (SolarEdge): select.solaredge_i1_storage_command_mode
    correctly transitioned to "Discharge to Maximize Export" at
    07:15:27; discharge limit propagated to match 643W.

Full chain confirmed consistent for this one real event: EMHASS forecast
-> decision (including the new cooldown gate correctly evaluating to
"pass through") -> command generation -> queue -> inverter, with the
643W value matching at every layer.
```

### Result

```text
PARTIAL PASS - one complete real end-to-end discharge event fully traced
and confirmed consistent across all 5 layers, including the new Grid
Charge Export Cooldown gate at Layer 4A. Only a single event was traced
in this depth; broader statistical confirmation across more events and
magnitudes would be needed for a full PASS.
```

### Notes

```text
This trace is also the direct evidence for Test 4A.4 case (d) - the
cooldown-inactive/default path was exercised and behaved identically to
the pre-fix logic, confirming no regression to normal discharge
behaviour.
```

---

## Test S.3 Negative Price Export Block

Status: NOT VERIFIED

### Purpose

Verify complete export curtailment path.

### Flow

```text
Layer 3
→ Layer 4B
→ Layer 4C
→ Layer 4D
```

### Pass

```text
Export curtailment correctly reaches the inverter.
```

### Observed

```text
-
```

### Result

```text
-
```

### Notes

```text
-
```

---

## Test S.4 Modbus Stability

Status: NOT VERIFIED

### Purpose

Verify stable operation during extended runtime.

### Observation Period

```text
24h+
```

### Verify

```text
No disconnect loops
No repeated queue lockups
No uncontrolled retries
No transaction_id mismatch bursts
```

### Pass

```text
System remains stable throughout the observation period.
```

### Observed

```text
-
```

### Result

```text
-
```

### Notes

```text
-
```
### Mixed final notes:
Maintain-zone scenario observed with low positive discharge request.
Charge request remained high at 5000 W despite SOC above target.