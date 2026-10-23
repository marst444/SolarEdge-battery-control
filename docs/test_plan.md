# SolarEdge Battery Control Test Plan

This document verifies the SolarEdge Battery Control package layer by layer. A layer is only considered verified when all critical tests have passed.

---

# Current Status

```text
Layer 1   STRONG PARTIAL PASS 🟡
    ✅ 1.1 Minimum SOC Protection
    ✅ 1.2 Maximum SOC Protection
    🟡 1.3 Recovery Mode

Layer 2   PARTIAL PASS 🟡 (strong)
    ✅ 2.1 EMHASS Forecast Availability
    ✅ 2.2 SOC Target Generation
    🟡 2.3 Dynamic Charge / Discharge Power

Layer 3   PARTIAL PASS 🟡 (strong)
    ✅ 3.1 Negative Price Curtailment
    🟡 3.2 Site Limit Enforcement (strong)

Layer 4A  STRONG PARTIAL PASS 🟡
    ✅ 4A.1 Charge Decision
    ✅ 4A.2 Discharge Decision
    ⏳ 4A.3 Maintain Zone (additional scenarios)
    🟡 4A.4 Grid Charge Export Cooldown

Layer 4B  STRONG PARTIAL PASS 🟡
    ✅ 5.1 Command Generation
    ✅ 5.2 Limit Updates

Layer 4C  STRONG PARTIAL PASS 🟡
    ✅ 6.1 Modbus Queue Serialization
    🟡 6.2 Concurrent Write Protection
    🟡 6.3 Recovery After Failed Write
    ✅ 6.4 Queue Telemetry Integrity (bug found 2026-10-19, fixed same day, confirmed live 2026-10-20)

Layer 4D  STRONG PARTIAL PASS 🟡
    ✅ 7.1 Storage Mode Apply
    ✅ 7.2 Charge Limit Apply
    ✅ 7.3 Discharge Limit Apply
    🟡 7.4 Mode Change Verification
    🟡 7.5 Modbus Long-Term Stability
    ✅ 7.6 Mode Command Guard

System    PARTIAL PASS 🟡
    🟡 S.1 End-to-End Charge
    🟡 S.2 End-to-End Discharge
    🟡 S.3 Negative Price Export Block
    🟡 S.4 Modbus Stability
```

Open observation: dynamic charge/discharge limits (Layer 4A "dynamic" helpers) and requested charge/discharge limits (`emhass_requested_*`) do not have a direct 1:1 relationship - the decision engine sits between them. Worth clarifying in architecture.md's control-chain description.

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

Layer 1 = STRONG PARTIAL PASS 🟡 — Verified: 1.1, 1.2. Partial: 1.3.

## Test 1.1 Minimum SOC Protection

**Status:** PASS ✅ (2026-08-06)

**Purpose:** verify battery discharge is blocked when SOC falls below configured minimum SOC.

**Evidence:** Enter - SOC 54.44% < min 65.0% → effective mode `charge_from_solar_and_grid`, charge limit 5000W, discharge limit 0W, reason "SOC below minimum - dynamic recovery charge". Exit - min lowered to 20.0% (no longer below) → effective mode `maximize_self_consumption`, discharge limit 0W, reason "Normal - effective follows requested". Both directions (enter/exit recovery) verified.

---

## Test 1.2 Maximum SOC Protection

**Status:** PASS ✅ (2026-08-06)

**Purpose:** verify charging is blocked when SOC exceeds configured maximum SOC (High SOC Hold with hysteresis: enters at SOC ≥ Max, exits at SOC ≤ Max - 3%).

**Evidence:** Enter - SOC 54.44% ≥ max 40.00% → High SOC Hold on, charge limit 0W, discharge limit 53W, self-consumption mode preserved. Exit - max raised to 80.00% → Hold off, charge limit 5000W, discharge limit 7W, normal control restored. The old forced export-discharge strategy has been replaced by this hysteresis-based hold and both transitions were verified.

**Note:** on the same day, SOC was observed continuing to rise past 90% to 94.44% before the charge limit zeroed out (quantization/apply latency). Proposed fix logged: enter Hold at Max - 1% instead of exactly at Max, to compensate. Not yet implemented/re-tested.

---

## Test 1.3 Recovery Mode

**Status:** PARTIAL PASS 🟡 (upgraded 2026-10-11, from NOT VERIFIED)

**Purpose:** verify critical low-SOC recovery behaviour (SOC < 10% → forced `charge_from_solar_and_grid`, charge limit > 0, discharge limit = 0).

**Evidence:** a real SOC<10% event occurred 2026-10-11 (reported SOC dropped from 22.22% at 15:36:15 to 0.0% at 15:47:41.905 local, spotted by the user in the stats). Investigated in full depth, logic through hardware:

**Confirmed working:**
- `calculate_effective_battery_control` fired correctly and atomically at 15:47:41.92: `effective_storage_mode` → `charge_from_solar_and_grid`, `effective_charge_limit` → 3000, `effective_discharge_limit` → 0, reason → "SOC critical below 10 - forced recovery charge" - correctly overriding both the active EMHASS fallback and the EV-charging discharge-limiting branch that had been active moments before.
- The physical mode and discharge-limit writes both landed correctly: `select.solaredge_i1_storage_command_mode` → "Charge from Solar Power and Grid" and `number.solaredge_i1_storage_discharge_limit` → 0, both at 15:49:19.00 local. The ~97s gap between the helper change and the physical landing is fully explained by the apply-automation's own serialized write sequence through `script.modbus_queue` (each step taking ~16-18s) - not a defect.
- The clean hand-off to the next branch (SOC 10-20%, "SOC below minimum - dynamic recovery charge", charge_limit → 3300) at 16:04:25.64, exactly matching the SOC crossing 10.00% at 16:04:25.621 - no mode flicker, since both branches request the same mode.

**Confirmed NOT working - two real gaps found, both now logged in known_issues_and_fixes.md:**
- *Charge-limit application*: `number.solaredge_i1_storage_charge_limit` never reached 3000 (or 3300) during this event at all. It was already at a stale 5000W (left over from an unrelated branch) half a second before the critical branch even fired, and stayed there continuously through both the SOC<10 and SOC<MIN bands - only correcting, straight to 3300, nearly 28 minutes later. Both the automation trace and the modbus_queue telemetry reported the 3000W write as a clean success - this is a silent failure nothing currently detects. See known_issues_and_fixes.md - Critical-Recovery Charge Limit Write Silently Dropped.
- *The trigger itself*: physics strongly suggests the reported 0.0% SOC was itself a bad sensor reading, not a real battery state. The drop from 22.22% to 0.0% happened with zero intermediate samples in an 11.43-minute gap (unlike every other transition in this dataset, including the clean 20-sample recovery climb right after it) - and even under the most generous assumption (continuous discharge at the battery's absolute hardware maximum of 4096W, well above the 100W cap actually in effect via the EV-charging branch the whole time), the maximum energy that could physically have been lost in that window is ~17% of capacity, not the observed ~22%. See known_issues_and_fixes.md - SOC<10% Critical Recovery Triggered By a Physically-Implausible Single-Sample SOC Reading.

**Result:** PARTIAL PASS - this confirms the Layer 1 decision logic and the mode/discharge-limit halves of the response chain work correctly when a SOC<10% condition is reported, including correctly overriding competing lower-priority logic and handing off cleanly as SOC recovers. It does **not** confirm the charge-limit half of the response actually reaches the inverter, and it does not confirm a genuine SOC<10% battery state was ever involved in the first place - if anything, the evidence points the other way. A clean full PASS would need both gaps closed: a charge-limit write confirmed landing correctly during a critical-recovery event, and either a plausibility guard added so a future event like this can be trusted at face value, or independent confirmation (e.g. a secondary SOC-estimation signal) that a reported near-0% reading is real.

---

# Layer 2 - Optimisation

Layer 2 = PARTIAL PASS 🟡 (strong) — Verified: 2.1, 2.2. Outstanding: 2.3 (two items).

## Test 2.1 EMHASS Forecast Availability

**Status:** PASS ✅ (2026-07-30)

**Purpose:** verify MPC and Day-Ahead forecasts are generated and available (`sensor.mpc_pv_batt_soc`/`_power`/`_grid_power` populated).

**Evidence:** MPC and Day-Ahead optimisation status both "Optimal"; 192 forecast entries on both SOC and power series; last MPC success and last day-ahead success both recent and clean.

---

## Test 2.2 SOC Target Generation

**Status:** PASS ✅ (2026-08-06)

**Purpose:** verify `input_number.soc_target` follows the smoothed EMHASS battery SOC forecast (`sensor.soc_batt_forecast_smooth`) and stays within min/max SOC.

**Evidence:** Two separated live observations (09:31 and 21:28 on 2026-08-06) both showed Target tracking Smooth within ±0.5-1.6 points, Smooth tracking raw MPC SOC closely, 192 forecast entries present, and Target always within the 20-90% configured range. Chain confirmed: MPC SOC forecast → `soc_batt_forecast_smooth` → `soc_target`. (An earlier draft of this test referenced a wrong entity, `sensor.soc_forecast_smooth`, which read `unknown` - corrected to the real entity above.)

**Update 2026-08-27:** while investigating the midday charge/discharge oscillation (see roadmap.md), `soc_target`/`soc_batt_forecast_smooth` were observed swinging 10-40 points between reads only 1-2 minutes apart. This doesn't contradict the chain verified above (it still faithfully follows whatever the latest MPC run publishes) - it shows the underlying MPC plan itself is volatile run-to-run, which this test doesn't cover. Logged as a follow-up, not a regression.

---

## Test 2.3 Dynamic Charge / Discharge Power

**Status:** PARTIAL PASS 🟡 (upgraded 2026-10-08; 10 of 12 checklist items closed)

**Purpose:** verify `sensor.dynamic_storage_charge_limit`/`_discharge_limit` are calculated correctly from the battery SOC forecast.

**Confirmed:**
- Charge direction (SOC < Target → charge > 0, discharge = 0): original single point (2026-07-30, 49W) now backed by the 2026-10-06 12:00-16:00 episode - 49 updates of `dynamic_charge_limit` ranging 100-1168W, while `dynamic_discharge_limit` held unchanged at 0 for the full 4 hours.
- Positive discharge limit beyond the small ~92W maintain-zone value (2026-08-06): the 2026-10-04 17:00-19:00 episode showed `dynamic_discharge_limit` ranging 11-1690W across 35 updates, charge pinned at 0.
- Target SOC dropping below current SOC: caught live in the same 2026-10-04 episode - at 17:00 SOC=90.00/target=89.76 (maintain-zone); by 17:15 target had dropped to 84.45 while SOC was still 90, and discharge limit jumped 11W→247W at that exact tick.
- Sensor → helper sync, last-value helper updates, maintain-zone small-discharge case: all previously confirmed (2026-07-30/08-06).

**Still open:**
- *Last Charge/Discharge Limit helpers still in use?* `last_charge_limit`/`last_discharge_limit` are confirmed alive - they track `dynamic_charge_limit`/`dynamic_discharge_limit` in exact lockstep, every tick, across both episodes above. Not confirmed: whether anything downstream actually *reads* them for a fallback decision. `ha_search` found only the writer automations (`automation.update_last_charge_limit`/`_discharge_limit`), no reader - but 60 YAML-defined automations and 39 YAML-defined scripts couldn't be scanned (this HA instance's config-fetch REST API doesn't expose YAML-defined automations/scripts, confirmed by `ha_config_get_automation` 404-ing on the writer automations themselves despite them being listed as existing).
- *Fallback to last_* when forecast is missing:* only two sub-5-second sensor blips found in 30 days (2026-09-29, consistent with the known twice-daily connectivity blip) - too brief to exercise a real "forecast missing" condition. During those blips the helpers simply held their existing value (how `input_number` entities behave natively, not necessarily dedicated fallback logic). No sustained EMHASS outage observed; YAML source not inspectable via this API to confirm intent either way.

---

# Layer 3 - Grid Constraints

Layer 3 = PARTIAL PASS 🟡 (strong) — Verified: 3.1. Outstanding: 3.2.

## Test 3.1 Negative Price Curtailment

**Status:** PASS ✅ (upgraded 2026-10-06, from PARTIAL PASS 2026-07-30)

**Purpose:** verify export is curtailed when export price goes negative.

**Evidence:** 2026-07-30 steady-state check (price positive, all curtailment helpers off) was followed by a real multi-episode overnight event on 2026-10-06: `sensor.total_export_price` flipped negative/positive 7 times between 00:00-08:00, and `negative_price_active`/`grid_export_blocked`/`switch.solaredge_i1_negative_site_limit` tracked all 7 cycles within 1-2 price-sensor update intervals (e.g. 00:00 negative→on, 01:01 positive→off, ... 07:00→on, 08:00→off). Two brief "unavailable" blips on the switch (Modbus reconnects) self-healed via the automation's 30s periodic re-assertion. No queue backlog or missed toggle despite `mode: single`.

Note: none of these episodes coincided with PV production (all pre-sunrise), so the discharge-block step fired but had nothing to actually block, and the site limit's export-curtailing *effect* was not exercised - that gap now lives in Test 3.2.

---

## Test 3.2 Site Limit Enforcement

**Status:** PARTIAL PASS 🟡 (upgraded from NOT VERIFIED, 2026-10-06)

**Purpose:** verify the SolarEdge site limit is applied during curtailment and correctly released afterwards.

**Evidence (2026-10-06, old switch-toggle design):**
1. Activation path confirmed: `number.solaredge_i1_site_limit` stayed at 0 across all 7 cycles from Test 3.1 (compare-before-write guard working).
2. Real gap found: the deactivation handler only turned the switch off - it never restored the site-limit number. Confirmed live: the number was still stuck at 0 nearly 6 hours after the switch itself went off.
3. Empirically harmless that day: 08:40-11:55, PV surplus up to 1.8kW occurred while the switch was off and the number was still stuck at 0 - and export tracked the uncovered surplus correctly anyway (meter readings matched PV - load - battery charge within 15-70W each time). This shows the SolarEdge integration gates enforcement on the switch, not the stale number - on this installation, item 2's gap had no real effect.

**Redesign (2026-10-06, same day):** rather than patching the restore step, the mechanism changed: the switch now stays permanently ON, and `number.solaredge_i1_site_limit` alone carries curtailment (0 = curtailed, 1,000,000 = normal/its own max). All three YAML files updated (see known_issues_and_fixes.md).

**Evidence under the new mechanism (2026-10-06 evening):** a one-time migration write corrected the stale 0 → 1,000,000 at 15:09:13 (price ≥ 0, number ≠ 1,000,000 → normal branch fired, self-healing as designed). Over the next ~5h25m (hundreds of trigger firings, price fluctuating but always ≥ 0), zero further writes occurred - compare-before-write guard confirmed both empirically and at the trace level (run `028e729419fc0e03a368eaee6389548a`: branch matched but no service call made). A real ~21s Modbus outage at 20:33:58 took the number "unavailable"; on recovery it was rewritten to 1,000,000 (the `| float(-1)` fallback correctly parsed "unavailable" as ≠ 1,000,000, triggering a safe rewrite rather than trusting a stale state).

**Evidence under the new mechanism, activation side (2026-10-19, upgraded from PARTIAL PASS):** the switch had sat "off" for a full week with zero negative-price episodes to exercise it. A real overnight episode on 2026-10-13 (deliberately zero PV, a safe live test) exercised the activation side for the first time: price crossed negative three times in ~20 minutes, the switch's defensive re-assert correctly flipped it `off -> on` on the very first crossing (22:34:13), and `number.solaredge_i1_site_limit` correctly cycled `1000000 <-> 0` in lockstep with all three crossings (22:34:11, 22:46:25, 22:49:16, 22:51:44, 22:54:50, 23:02:40), matched by `input_boolean.negative_price_active`/`grid_export_blocked` throughout. The switch has since stayed `on` through 6 days and 12+ twice-daily Modbus blips, confirming durability. Full evidence in known_issues_and_fixes.md - Negative Price Curtailment - Site Limit Restore Gap, Update 2026-10-19.

**Result:** PARTIAL PASS (strong) - both the deactivation/normal-value path (2026-10-06) and the activation/curtailed-value path (2026-10-19) are now confirmed correct and self-healing, including through both a real Modbus outage and a full week-plus of durability. Still open, the one scenario this test has always been waiting on: the enforcement-while-ON scenario - negative price coinciding with real PV surplus, to directly observe curtailed export rather than infer it from command correctness alone. 2026-10-13's episode was deliberately zero-PV, so this remains the single outstanding gap for a full PASS.

---

# Layer 4A - Decision Engine

Layer 4A = STRONG PARTIAL PASS 🟡 (upgraded 2026-10-07) — Verified: 4A.1, 4A.2, 4A.3 (base case). Outstanding: 4A.3 (additional deadband scenarios), 4A.4 (cases b/c).

## Test 4A.1 Charge Decision

**Status:** PASS ✅ (upgraded 2026-10-07, from PARTIAL PASS 2026-07-30)

**Purpose:** verify requested charging behaviour (`emhass_requested_storage_mode`/`_charge_limit`) follows forecast and SOC conditions.

**Evidence:** the 2026-07-30 check caught the engine at SOC≈target (41.11% vs 41.10%, maintain-zone) - charge path under real charging-required conditions was explicitly left unverified. Closed 2026-10-07 using the 2026-10-06 12:15-15:45 `charge_from_solar_and_grid` episode: 13 ticks over 3.5 hours, every one with SOC < Target (gap 5-15 points), a charge request scaled to the gap (256W-1059W, shrinking as SOC approached target), and discharge blocked at 0W throughout.

---

## Test 4A.2 Discharge Decision

**Status:** PASS ✅ (upgraded 2026-10-07, from PARTIAL PASS 2026-07-30)

**Purpose:** verify requested discharge behaviour follows forecast and SOC conditions.

**Evidence:** the 2026-07-30 check caught the engine only 1.6 points above target (barely outside deadband, 0W discharge). Closed 2026-10-07 using the 2026-10-04 17:00-19:00 `discharge_to_maximize_export` episode: 9 ticks over 2 hours, SOC 5-25 points above target throughout, charge blocked at 0W, discharge scaled with the gap (11W-1627W).

---

## Test 4A.3 Maintain Zone

**Status:** base case PASS ✅ (2026-07-30); additional deadband scenarios ⏳ not yet covered.

**Purpose:** verify the battery stays near SOC target when inside the deadband.

**Evidence:** SOC 41.11% vs target 41.10% (deadband 3.33%) → requested and effective mode both `maximize_self_consumption`, charge 0W, discharge 1W, reason "Normal - effective follows requested". Requested/effective decisions matched; no meaningful charge or discharge request generated.

---

## Test 4A.4 Grid Charge Export Cooldown

**Status:** PARTIAL PASS 🟡 - case (a) confirmed live (2026-10-07); cases (b)/(c) still open.

**Purpose:** verify that after `charge_from_solar_and_grid` fires, `discharge_to_maximize_export` is held back (substituted with `maximize_self_consumption`) until `grid_charge_export_cooldown_minutes` (default 60) has elapsed - preventing a grid-assisted charge from being sold straight back out at a loss (the issue found 2026-08-27, see roadmap.md).

**Design:** every `charge_from_solar_and_grid` branch stamps `last_grid_charge_command_time`; the DISCHARGE MAX EXPORT branch checks cooldown-elapsed before choosing `discharge_to_maximize_export` vs. the held-back `maximize_self_consumption`.

**Evidence:**
- Deployed 2026-08-28; case (d) (no recent grid charge → unchanged normal behaviour) confirmed immediately via trace (soc=51.11, target=40.01, `cooldown_elapsed=true` → `discharge_to_maximize_export`, 643W - correct, unregressed).
- Case (a) (stamping) confirmed 2026-10-07 across a 10-day window: every `charge_from_solar_and_grid` tick stamps `last_grid_charge_command_time`, including on every re-selection tick of a multi-hour charge, not just the first - a rolling cooldown window, not a one-shot.
- Cases (b)/(c) (the actual hold-back substitution, and resumption after cooldown elapses): never observed. Across the full 10-day window, every grid-charge episode either kept re-selecting `charge_from_solar_and_grid` the whole time, or moved to `maximize_self_consumption` for unrelated maintain-zone reasons before the cooldown had even elapsed - the DISCHARGE MAX EXPORT conditions never happened to become true *while* a cooldown was still active. With only 5 stored traces per automation, this can only be caught live, not dug up retroactively.

---

# Layer 4B - Command Generation

Layer 4B = STRONG PARTIAL PASS 🟡 — Verified: 5.1, 5.2. Outstanding: none.

## Test 5.1 Command Generation

**Status:** PASS (2026-07-30)

**Purpose:** verify the effective battery decision is translated into the expected command scripts.

**Evidence:** triggered by `effective_discharge_limit` change; correct mode branch (`maximize_self_consumption_script`) and both limit scripts (`set_effective_storage_charge_limit`, `set_effective_storage_discharge_limit`) executed.

---

## Test 5.2 Limit Updates

**Status:** PASS (upgraded 2026-08-28, from PARTIAL PASS 2026-07-30)

**Purpose:** verify effective charge/discharge limits are translated into the correct SolarEdge commands.

**Evidence:** 2026-07-30 confirmed the discharge path (1W propagated correctly) but the charge path's one observed value happened to already be 0W, leaving non-zero charge untested. Closed 2026-08-28 using the same 21-sample non-zero `effective_charge_limit` evidence gathered for Test 7.2 - all propagated correctly via `set_effective_storage_charge_limit` → `modbus_queue` within 15-55s. Both paths now confirmed for zero and non-zero values.

---

# Layer 4C - Modbus Queue

Layer 4C = STRONG PARTIAL PASS 🟡 — Verified: 6.1, 6.4. Outstanding: 6.2, 6.3.

## Test 6.1 Modbus Queue Serialization

**Status:** PASS (2026-07-30)

**Purpose:** verify only one Modbus write executes at a time.

**Evidence:** multiple commands sent through `script.modbus_queue`; `modbus_busy` correctly toggled on/off around each; commands executed sequentially (timeout → mode → charge limit → discharge limit) with no stuck lock.

---

## Test 6.2 Concurrent Write Protection

**Status:** PARTIAL PASS 🟡 (not independently upgraded since 2026-07-30)

**Purpose:** verify truly simultaneous command requests are serialized without race conditions.

**Evidence:** several commands fired from one automation run were handled sequentially via the `modbus_busy` lock - but this only exercises one automation's own sequential dispatch, not a genuine external write collision (e.g. two different automations firing at once). True concurrent-collision testing would mean deliberately forcing a race condition on live hardware - flagged as a live-intervention item needing explicit approval, not yet given.

---

## Test 6.3 Recovery After Failed Write

**Status:** PARTIAL PASS 🟡

**Purpose:** verify the queue recovers after a failed Modbus write without manual intervention.

**Evidence:** real incident, 2026-08-27 06:45: a discharge-limit write failed ("Connection failed... Not connected"). Root cause (confirmed by reading `solaredge_modbusqueue.yaml`): the final `modbus_busy` turn-off step is a plain sequence step, not exception-safe - a service-call failure inside the `choose:` block aborts the script and skips it, leaving the lock stuck "on". It sat stuck for ~30 minutes (nothing else needed sending in that window, not a deeper hang) until the next real command at 07:15 triggered `modbus_queue`'s own 10s `wait_template`/`continue_on_timeout` guard, which force-cleared the stale lock and resumed normal operation - directly observed working, including a clean subsequent trace and correct mode transition.

**Result:** recovery is real but not prompt - the lock isn't released by the failure itself, only forced through by the *next* caller's 10s timeout (worst case ~10s extra wait). No command was silently/permanently lost. Suggested fix logged in roadmap.md: make the lock-release step exception-safe so it runs immediately on failure.

Note (2026-10-19): this test's evidence predates the 2026-09-20 exception-safe lock-release fix (known_issues_and_fixes.md - Modbus Queue Lock Release Not Exception-Safe, confirmed behaviorally live 2026-09-24) - the stuck-lock *mechanism* this test describes no longer matches the current code. Left as-is rather than rewritten, since it's still accurate history and the fix's own verification lives in that known_issues_and_fixes.md entry; a future pass could fold a current-behavior confirmation in here instead.

---

## Test 6.4 Queue Telemetry Integrity

**Status:** PASS ✅ (bug found 2026-10-19, fixed same day, deployed and confirmed live 2026-10-20)

**Purpose:** verify `script.modbus_queue`'s own diagnostic counters (`input_number.modbus_queue_pending_count` and friends) accurately reflect queue state, since downstream watchdog sensors (`binary_sensor.solaredge_modbus_queue_backlog`) and this test plan's own evidence trail depend on them being trustworthy.

**Evidence (the bug):** found during a routine behaviour review, not a targeted test - `binary_sensor.solaredge_modbus_queue_backlog` had been reporting "on" (backlog, pending_count >= 5) continuously for 33+ hours. Investigation found `modbus_queue_pending_count`'s resting floor does not return to 0 after draining: it reached 67 over 2026-10-17, was reset to 0 (most likely an HA restart, not a fix) on 2026-10-18 morning, then climbed again to 10 by 2026-10-19 and has sat there since - tracking `modbus_queue_timeout_count_total` 1:1 throughout, pointing at the lock-wait timeout branch (`wait.completed == false`) leaking an uncorrected +1. Actual command throughput was verified normal the whole time (writes dispatch and land correctly), and nothing currently consumes the backlog sensor downstream, so the blast radius so far was "this one sensor and this test's own future evidence can't be trusted at face value" rather than any control-path impact. Full write-up in known_issues_and_fixes.md - Modbus Queue Pending-Count Counter Leaks On Lock-Wait Timeout.

**Fix (2026-10-19):** `solaredge_modbusqueue.yaml`'s lock-wait-timeout branch now also decrements `modbus_queue_pending_count` by 1 (floored at 0), exactly once per timeout event, making the counter self-correcting regardless of which run's own end-of-run decrement went missing. Made on the strength of the 1:1 correlation with `modbus_queue_timeout_count_total` rather than a live trace pinning down the exact HA-concurrency mechanism first.

**Evidence (live confirmation, 2026-10-20):** the user deployed the fix; a real lock-wait timeout occurred that same night (`modbus_queue_timeout_count_total` 10 -> 11 at 04:05:24.891 local). `modbus_queue_pending_count`'s history across the event: 10->11 (run A starts) -> 11->12 (run B starts, stacked) -> 12->11 at 04:05:24.896 (the compensating decrement, 5ms after the timeout counter's own bump) -> 11->10 (run B's end-of-run decrement) -> 10->11 (run C starts) -> 11->10 (run C's end-of-run decrement). Net change across the whole burst: zero - the floor returned to 10 (its pre-fix resting value) instead of climbing to 11 the way every prior timeout had. The 5ms gap between the two counters' updates is as close to a direct trace as live history can give, since `script.modbus_queue`'s own trace for this run had already rolled off its 5-run retention by the time this was checked.

**Result:** PASS - the bug is confirmed, the fix is deployed live, and a real timeout event afterward confirms the counter no longer leaks. One residual item, not a defect: the pre-existing floor of 10 itself wasn't retroactively repaid (the fix only stops future leaks), so `binary_sensor.solaredge_modbus_queue_backlog` will keep reading a false "on" until that resting value is manually cleared or an HA restart resets it - a one-time cleanup action, not a re-test condition.

---

# Layer 4D - SolarEdge Apply

Layer 4D = STRONG PARTIAL PASS 🟡 — Verified: 7.1, 7.2, 7.3, 7.6. Outstanding: 7.4, 7.5.

## Test 7.1 Storage Mode Apply

**Status:** PASS (2026-07-30)

**Purpose:** verify storage mode commands are applied to SolarEdge.

**Evidence:** effective mode `maximize_self_consumption` matched live `select.solaredge_i1_storage_command_mode` = "Maximize Self Consumption"; Remote Control authority active; command timeout 900s as expected.

---

## Test 7.2 Charge Limit Apply

**Status:** PASS ✅ (upgraded from PARTIAL PASS, 2026-08-28)

**Purpose:** verify charge limit commands are applied to SolarEdge.

**Evidence:** zero-limit path confirmed 2026-07-30. Non-zero path closed 2026-08-28: 21 sampled non-zero `effective_charge_limit` values (20W to 5000W, spanning small/mid-range/ceiling values) over a 2-day window, every one propagated to `number.solaredge_i1_storage_charge_limit` within ~15-55s with no mismatches or stale values.

---

## Test 7.3 Discharge Limit Apply

**Status:** PASS (2026-07-30)

**Purpose:** verify discharge limit commands are applied to SolarEdge.

**Evidence:** two runs (1W, 4W) both propagated correctly via `set_effective_storage_discharge_limit` → `modbus_queue` to the live discharge-limit entity.

---

## Test 7.4 Mode Change Verification

**Status:** PARTIAL PASS 🟡 (4 of 5 modes confirmed)

**Purpose:** verify each of the 5 storage modes is correctly applied to SolarEdge.

**Evidence:** 2026-08-28, 7-day history cross-reference confirmed `maximize_self_consumption`, `discharge_to_maximize_export`, and `charge_from_solar_and_grid` all apply correctly (each observed multiple times). `solar_power_only` and `charge_from_clipped_solar` hadn't occurred naturally in that window. **Update 2026-10-06:** `solar_power_only` occurred twice more in a fresh 10-day check (2026-09-26, 2026-09-30), both times applying correctly within 1-2 minutes - upgraded to 4 of 5. `charge_from_clipped_solar` still has not occurred naturally in any window checked - pure coverage gap, not a known defect.

---

## Test 7.5 Modbus Long-Term Stability

**Status:** PARTIAL PASS 🟡

**Purpose:** verify stable queue operation over 24h+ (no lockups, no mismatch storms, no runaway retries).

**Evidence (2026-08-28, 48h error log):** one real write-failure cluster (see Test 6.3), 8 "Cancel send", 27 "Cancel send/Repeating call/No response" (transient self-recovering chatter), 7 transaction_id mismatches, 1 coordinator fetch failure. No indefinite lockup; `modbus_busy` cycling normal throughout.

**Result:** no lockup or storm, and the self-healing stale-lock recovery (Test 6.3) was directly confirmed - but the 27/7/8 baseline noise over 48h hasn't been root-caused (RS485/TCP contention? inverter latency?) or shown to have zero impact over a longer window. See roadmap.md for the 2026-10-02 close_after_polling/scan_interval change and its before/after numbers.

---

## Test 7.6 Mode Command Guard

**Status:** PASS ✅ (core behaviour verified 2026-08-27; diagnostic-sensor gap resolved 2026-08-28; case (c) confirmed live 2026-10-22)

**Purpose:** verify the mode-select command and 15-minute timeout reset are skipped only when effective mode, live command mode, and configured default mode all agree on Maximize Self Consumption - and that the full command sequence is sent for any real transition or guard-precondition mismatch.

**Evidence:** trace `83276d1d92e8bb49a69e2f61c0082c9f` (2026-08-27) confirmed the skip branch firing correctly (only limit scripts ran, mode-select/timeout skipped) with `mode_guard_active` computed from live state, not a cache. 24h of command-mode history showed 7 real transitions, each correctly triggering the full sequence. One anomaly noted: a 12s mode flip with no corresponding `effective_storage_mode` change - likely a manual test during the evaluation session, not a control-loop issue; left unexplained but not treated as a guard defect. The standalone diagnostic `sensor.battery_mode_command_guard` was initially missing from the live config (file not yet copied) - resolved 2026-08-28, now reads "guarded"/"active" correctly.

**Case (c), confirmed live 2026-10-22:** guard should stay inactive (resending the mode command) if the inverter's configured default mode isn't Maximize Self Consumption. 30 days of history (checked 2026-10-08) had shown `select.solaredge_i1_storage_default_mode` never once deviating from MSC, so this needed a deliberate live change - done with the user's explicit approval. The user set `select.solaredge_i1_storage_default_mode` to "Solar Power Only (Off)" via the entity's real control (not a Developer Tools state override) at 10:08:51 local; the SolarEdge integration's live `select.solaredge_i1_storage_command_mode` changed to the same value at the same instant (10:08:51.52, 2ms apart - the write evidently lands on both registers together on this installation, not just the configured fallback). `sensor.battery_mode_command_guard` flipped from "guarded" to "active" immediately (pure template, reacted within 2ms).

The next natural trigger (`input_number.effective_discharge_limit` ticking 1→52 at 10:15:01 local, run `9d6c21d44d77ee8e0aeb55ccecfb67da`) confirmed the guard logic directly from its own trace: `mode_guard_active: false` (both `inverter_command_mode` and `inverter_default_mode` read "Solar Power Only (Off)", not MSC), `if: not mode_guard_active` resolved `true`, and the full "then" sequence ran - `script.set_value_storage_command_timeout15m`, then (via the `choose:` block matching `effective_mode == maximize_self_consumption`) `script.maximize_self_consumption_script`, then both limit scripts. Nothing was skipped. `select.solaredge_i1_storage_command_mode` landed back on "Maximize Self Consumption" at 10:16:19 (normal ~69s queue lag), and `input_number.mode_command_skipped_count_total`/`_today` stayed at 139/26 throughout - unchanged from immediately before the test, confirming the skip counter correctly did not increment while the guard was inactive.

**Result:** PASS - all three guard preconditions (effective mode, live command mode, configured default mode) are now independently confirmed live: the skip path (2026-08-27), real mode transitions (24h history, same date), and now the guard correctly staying inactive and self-correcting the live mode when the configured default mode alone disagrees (2026-10-22), even though both the live and default mode happened to change together here. The guard never silently ceded control to an unexpected fallback - it caught the mismatch on its very next cycle and resent the full command.

---

# System Tests

System = PARTIAL PASS 🟡 — all four (S.1-S.4) verified-but-partial. Outstanding: none distinct from what's listed under each test.

## Test S.1 End-to-End Charge

**Status:** PARTIAL PASS 🟡

**Purpose:** verify the complete charge flow, Layer 2 → 4A → 4B → 4C → 4D.

**Evidence:** one full-depth trace of the 2026-08-27 ~11:00 charge event (the same event that motivated Test 4A.4): Layer 2 MPC favoured charging; Layer 4A selected `charge_from_solar_and_grid` at 597W; Layer 4B/4C applied and queued cleanly; Layer 4D's command mode and charge limit both landed on 597W. **Update 2026-10-07:** the 2026-10-06 13-tick charge episode (used for Test 4A.1) was cross-checked against Layer 4D and matched throughout (mode + limit, 15-85s lag) - a second, longer episode, but it didn't re-pull Layer 2's raw forecast variables, so it strengthens rather than replaces the single-event caveat.

**Result:** still PARTIAL - one event traced to full depth (all 5 layers including Layer 2 internals); broader confirmation at that same depth, across more events and magnitudes, would be needed for PASS.

---

## Test S.2 End-to-End Discharge

**Status:** PARTIAL PASS 🟡

**Purpose:** verify the complete discharge flow, Layer 2 → 4A → 4B → 4C → 4D.

**Evidence:** one full-depth trace of the 2026-08-28 07:15 discharge event (also the evidence for Test 4A.4 case (d)): Layer 2 showed SOC above target with export-favourable forecast; Layer 4A selected `discharge_to_maximize_export` at 643W; Layer 4B/4C applied cleanly; Layer 4D's command mode and discharge limit both landed on 643W. **Update 2026-10-07:** the 2026-10-04 9-tick discharge episode (used for Test 4A.2) was cross-checked against Layer 4D and matched throughout (15-35s lag) - same caveat as S.1: strengthens, doesn't replace, the single full-depth event.

**Result:** still PARTIAL, for the same reason as S.1.

---

## Test S.3 Negative Price Export Block

**Status:** PARTIAL PASS 🟡 (upgraded from NOT VERIFIED, 2026-10-06)

**Purpose:** verify the complete export-curtailment flow, Layer 3 → 4B → 4C → 4D.

**Evidence:** command path confirmed via Test 3.1/3.2's 7-cycle overnight event (Layer 3 → 4D correct and prompt). Site-limit mechanism was redesigned the same day (see Test 3.2); the new `set_site_limit` path's *normal* (1,000,000) side was exercised live that evening via the migration write and subsequent guarded no-op behaviour. The *curtailed* (0) side was confirmed live 2026-10-19 against a real overnight negative-price episode (see Test 3.2's 2026-10-19 update) - three full activate/deactivate cycles, command path correct and prompt each time. Still missing: the one scenario that would show curtailment as an actual export reduction rather than inferring it from command correctness - negative price coinciding with real PV surplus. Every episode so far (2026-10-06 and 2026-10-13) has been overnight with zero PV, 2026-10-13 deliberately so.

---

## Test S.4 Modbus Stability

**Status:** PARTIAL PASS 🟡 (upgraded from NOT VERIFIED, 2026-10-07)

**Purpose:** verify stable operation over 24h+ (no disconnect loops, no repeated lockups, no uncontrolled retries, no mismatch bursts).

**Evidence (6.7-day window, 2026-10-01 to 10-07, spanning a live HA restart):**
- 1500 `modbus_busy` lock cycles analyzed: zero over 60s, max 50.4s, median 18.1s, only 2.9% over 30s - extends the earlier 48h and 4.5-day no-lockup windows into a longer, restart-inclusive one.
- A 44-minute slice showed TCP reconnecting every ~7.2s - expected shape of `close_after_polling: true` + 5s `scan_interval`, not instability; no disconnect *loop* present.
- transaction_id-mismatch/Cancel-send baseline couldn't be freshly compared like-for-like: the restart reset HA's log-dedup counters, leaving only ~9.5h of post-restart history (2 mismatches, 1 connect failure in that window - consistent with, not contradicting, the earlier 48h baseline).
- The known nightly "~22:00 unavailable" blip is now twice-daily (~06:3x-06:5x and ~20:1x-20:5x local), ~15 occurrences of 21-33s each - a timing update to the existing documented item, not a new defect.

**Result:** no lockup or disconnect-loop pattern across 6.7 days including a restart. Not a full PASS because the mismatch/Cancel-send baseline couldn't be freshly re-measured over a comparable multi-day window this time - re-check once enough post-restart history accumulates.