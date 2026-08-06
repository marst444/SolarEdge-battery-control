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
    🟡 2.2 SOC Target Generation
    🟡 2.3 Dynamic Charge / Discharge Power

Layer 3   PARTIAL PASS 🟡
    🟡 3.1 Negative Price Curtailment
    ⏳ 3.2 Site Limit Enforcement

Layer 4A  PARTIAL PASS 🟡 (strong)
    🟡 4A.1 Charge Decision (Maintain-Zone Scenario)
    ⏳ 4A.2 Discharge Decision
    ⏳ 4A.3 Maintain Zone (Additional Scenarios)

Layer 4B  PARTIAL PASS 🟡
    ✅ 5.1 Command Generation
    🟡 5.2 Limit Updates

Layer 4C  PARTIAL PASS 🟡
    ✅ 6.1 Modbus Queue Serialization
    🟡 6.2 Concurrent Write Protection
    ⏳ 6.3 Recovery After Failed Write

Layer 4D  PARTIAL PASS 🟡 (strong)
    ✅ 7.1 Storage Mode Apply
    🟡 7.2 Charge Limit Apply
    ✅ 7.3 Discharge Limit Apply
    ⏳ 7.4 Mode Change Verification
    ⏳ 7.5 Modbus Long-Term Stability

System    NOT VERIFIED

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

Layer 1 = NOT VERIFIED

Verified:

```text
-
```

Outstanding:

```text
1.1 Minimum SOC Protection
1.2 Maximum SOC Protection
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
🟡 2.2 SOC Target Generation
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

Test 2.2 = PARTIAL PASS 🟡
✅ SOC target uppdateras från MPC forecast
✅ Värdet ligger inom min/max SOC
✅ Forecast-data existerar
✅ Forecast → smoothing → target-kedjan fungerar
⏳ SOC target följer förändringar över tiden
⏳ SOC target reagerar korrekt på en ny MPC-körning
⏳ SOC target klampar korrekt vid min/max-gränser
⏳ Fallback-beteende om MPC forecast saknas


Date: 2026-07-30

### Purpose

Verify SOC target follows EMHASS battery forecast.

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
```

### Result

```text
PARTIAL PASS
```

### Notes

```text
SOC target correctly follows EMHASS battery forecast.

Forecast smoothing layer and SOC target layer were consistent with
the underlying MPC forecast.

SOC value remained within configured safety limits.
Notes:

A single forecast verification point was observed.

Additional verification still required:

- SOC target follows forecast updates over time
- SOC target responds to subsequent MPC runs
- SOC clamping behaviour at min/max limits
- Forecast-loss fallback behaviour
```

---

## Test 2.3 Dynamic Charge / Discharge Power
✅ Forecast ger ett rimligt charge limit-värde
 ✅ Forecast ger 0 discharge när mål-SOC ligger över aktuell SOC
 ✅ Sensor → helper-synk fungerar
 ✅ Last-value-hjälparna uppdateras
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

# Layer 4B - Command Generation

Layer 4B = PARTIAL PASS 🟡

Verified:

✅ 5.1 Command Generation
🟡 5.2 Limit Updates
``

```text
-
```

Outstanding:

```text

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

PARTIAL PASS

Notes:

- Discharge limit apply path verified.
- Charge limit script executed, but charge limit state change was not explicitly observed in trace, likely because value was already 0W.
- Need verify a non-zero charge limit update.
``
-
```

---

# Layer 4C - Modbus Queue

Layer 4C = PARTIAL PASS 🟡

Verified:
✅ 6.1 Modbus Queue Serialization
🟡 6.2 Concurrent Write Protection


Outstanding:

```text
⏳ 6.3 Recovery After Failed Write
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

Status: NOT VERIFIED

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

# Layer 4D - SolarEdge Apply

Layer 4D = PARTIAL PASS 🟡

Verified:

✅ 7.1 Storage Mode Apply
🟡 7.2 Charge Limit Apply
✅ 7.3 Discharge Limit Apply

Outstanding:

⏳ Verify non-zero charge limit apply
⏳ 7.4 Mode Change Verification
⏳ 7.5 Modbus Long-Term Stability
`

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

Status: NOT VERIFIED

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
### Test 7.2 Charge Limit Apply 
Status: PARTIAL PASS 
Date: 2026-07-30 
### Observed 
```text 
Effective Charge Limit = 0W Storage Charge Limit = 0W
```
### Result

```
PARTIAL PASS

```

### Notes

```
Charge limit matched the effective charge limit.

Only zero-charge-limit path verified.
A non-zero charge limit still needs verification.
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

Status: NOT VERIFIED

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

## Test 7.5 Modbus Long-Term Stability

Status: NOT VERIFIED

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

# System Tests

System = NOT VERIFIED

Verified:

```text
-
```

Outstanding:

```text
S.1 End-to-End Charge
S.2 End-to-End Discharge
S.3 Negative Price Export Block
S.4 Modbus Stability
```

---

## Test S.1 End-to-End Charge

Status: NOT VERIFIED

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

## Test S.2 End-to-End Discharge

Status: NOT VERIFIED

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
