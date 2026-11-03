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

Investigated whether EMHASS optimisation failures should automatically trigger fallback planning rather than only updating watchdog status (the automations in question rely on `sensor.mpc_pv_optim_status`/`sensor.dh_pv_optim_status`). No real optimisation failure occurred to observe directly, but tracing confirmed: the watchdogs themselves (`EMHASS_watchdog_dayahead_missing`/`_mpc_stalled`) are notification-only; a separate automation (`EMHASS_addon_watchdog_auto_restart`) already auto-restarts the EMHASS addon after 90 minutes of MPC stall; and Layer 4A (`battery_forecast_control.yaml`) already freezes the last-computed plan rather than recalculating from stale/zeroed data when EMHASS health is bad - functionally equivalent to serving a cached plan. The one remaining gap (EMHASS itself serving a degraded/cached plan internally on a failed optimisation) was confirmed genuinely out of scope - it would require a change inside EMHASS's own code, not this HA package.

Per the user's decision, an escalation was added 2026-10-10: if MPC stays stalled for 45 minutes (the same threshold as the existing stall notification), the system now actively forces `maximize_self_consumption` with 3300W charge/discharge limits, folded into the existing `EMHASS_watchdog_mpc_stalled` automation, so the frozen pre-failure plan doesn't go stale indefinitely. The frozen/fallback plan was also confirmed to survive both an EMHASS addon restart and a full HA core restart (via HA's normal `input_select`/`input_number` restore-state behaviour). The escalation was confirmed live end-to-end on 2026-10-11 against a real extended MPC stall - verified at the HA-entity, Layer 1 passthrough, and physical-inverter Modbus-register levels, with the redundant-write guard and other automations (e.g. EV discharge limiting) behaving correctly alongside it.

Status: resolved - no outstanding verification gap remains for this item.

---

### EMHASS Addon Build-On-Start Fragility

Discovered 2026-09-22: the EMHASS addon (v0.18.3, installed via the
manual/local install method) rebuilds its Python environment on every
container start (`Building emhass @ file:///app`, resolving and
downloading `aiohttp`/`hatchling` from PyPI) rather than running a
pre-built image. This makes addon startup itself dependent on live
DNS/internet access - after the 2026-09-22 power outage, DNS wasn't back
up yet when the addon's container restarted, so the build failed
identically on every attempt and the addon was stuck in Supervisor state
`error` for the rest of the day, needing a manual restart once DNS had
recovered.

A same-day mitigation (`EMHASS Watchdog - Addon Auto-Restart`, see
known_issues_and_fixes.md - EMHASS Outage Fallback Guard) now retries
`hassio.addon_restart` automatically after 90+ minutes stalled, at most
once/hour - this would have recovered tonight's incident unattended once
DNS came back, but it's a retry loop around the same fragile build, not
a fix for it. Worth investigating whether a pinned/pre-built version of
the addon (Installation Method 2 in the addon's own docs, or an updated
addon release that ships a built image) avoids the network dependency at
startup entirely.

Update 2026-09-24: user confirmed this isn't actionable right now -
runs on Home Assistant Green, which the user doesn't believe can handle
a local/pre-built EMHASS install, and there's no NUC or other spare
hardware available to run EMHASS on instead. Parking this item as-is
(the auto-restart watchdog mitigation stands); don't re-propose a
reinstall/hardware-swap path without checking back on this constraint
first. If revisited, worth asking specifically about the addon's own
Installation Method 2 (pinned/pre-built image) rather than a hardware
change - that may not carry the same resource concern, but hasn't been
evaluated against Green's constraints yet.

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

(Reconfirmed during the 2026-09-24 YAML header/hygiene audit - same
naming/semantics mismatch, unique_id typo noted separately below.)

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

Found during the 2026-11-02 HA cleanup audit (confirmed via targeted
dashboard `query` text searches, which catch references inside
`custom:apexcharts-card` that a plain `entity_id` wildcard search
misses): 7 of the 22 published `mpc_pv_*`/`dh_pv_*` forecast sensors are
not referenced on either dashboard (`url_path: lovelace`/`default`) -
`sensor.mpc_pv_inverter_power`, `mpc_pv_cost_fun`, `mpc_pv_export_price`,
`dh_pv_inverter_power`, `dh_pv_cost_fun`, `dh_pv_import_price`,
`dh_pv_export_price`. The other 15 (including `batt_power`, `batt_soc`,
`pv_power`, `load_power`, `grid_power`, `ev_power`, `import_price` for
MPC, and the full DH_PV set used by the "DH PV Energy plan" chart) are
confirmed in active use. Not removed in the 2026-11-02 cleanup pass -
the user scoped that pass to the dead sliding-window subsystem and the
5 override booleans only (see Resolved below); this 7-sensor list is
carried forward here for a future round, pending the user's decision on
whether "unused on the dashboard" should mean "safe to remove" for
these (they may still be useful for ad-hoc inspection even if no card
currently plots them).

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

(Reconfirmed during the 2026-09-24 YAML header/hygiene audit and again
during the 2026-11-02 HA cleanup audit: `negative_price_curtailment.yaml`
and `gridconstrain_helpers.yaml` - the only two files that define/could
plausibly use it - still show no read or write of `grid_export_limited`
anywhere in either file, and it appears on no dashboard card either.
Not removed in the 2026-11-02 pass - the user did not select this item
for that round.)

---

### Possibly Dead: `rest_command.publish_data`

Found during the 2026-09-24 YAML header/hygiene audit, reconfirmed
2026-11-02: `rest_command.publish_data` (`emhass_restcommand.yaml`)
targets a different host (`192.168.10.3`) than the other three
`emhass_*` rest_commands in the same file (all `127.0.0.1`), and no
automation or script in `emhass_scripts.yaml`/`emhass_automations.yaml`
was found calling it (they call `rest_command.emhass_publish_data`
instead, a distinct entity). It's listed in `entities.md`. Possibly
legacy/dead config from an earlier setup - confirm whether it's still
needed before removing it. Not removed in the 2026-11-02 pass - the
user did not select this item for that round.

---

### Possibly Dead/Duplicate: `shell_command.trigger_nordpool_forecast`

Found during the 2026-09-24 YAML header/hygiene audit, reconfirmed
2026-11-02: `shell_command.trigger_nordpool_forecast`
(`emhass_shellcommand.yaml`) posts raw 24h Nordpool-only data directly
to EMHASS's `dayahead-optim` endpoint via curl. This appears to
duplicate/be superseded by `emhass_scripts.yaml`'s
`load_cost_forecast`/`prod_price_forecast` Jinja blocks feeding
`rest_command.emhass_dayahead_optim` (merged Nordpool+EPEX, 2-day/15-min
horizon) - a more complete forecast path. No trigger or call site for
the shell_command was found anywhere in `emhass_automations.yaml` or
`emhass_scripts.yaml`. Possibly legacy; confirm before removing. Not
removed in the 2026-11-02 pass - the user did not select this item for
that round.

---

### Unused `ev` Variable In `battery_forecast_control.yaml`

Found during the 2026-09-24 YAML header/hygiene audit:
`emhass_battery_forecast_control` reads `binary_sensor.
ev_charging_active` into a local `ev` variable, but no branch in the
automation's current `choose:` logic references it - EV-aware discharge
limiting is actually owned by `automation.ev_charging_discharge_control`
in `batterycontrol_automations.yaml`. Confirm whether `ev` is
intentionally reserved for future use in this automation or should be
removed as dead config. Not removed in the 2026-11-02 pass - the user
did not select this item for that round.

---

### `effective_*` Helpers Defined In A Different Layer's File Than They're Written

Found during the 2026-09-24 YAML header/hygiene audit: `input_number.
effective_charge_limit`, `input_number.effective_discharge_limit`, and
`input_select.effective_storage_mode` are Layer 1 output entities -
written by `safety_limits_and_override.yaml`'s
`calculate_effective_battery_control` - but their *definitions* live in
`batterycontrol_helpers.yaml` (a Layer 4A file) rather than
`safety_and_watchdog_helpers.yaml` (the Layer 1 helpers file). Purely
organisational, no functional impact. Consider moving the definitions
to match ownership, or documenting the split as intentional (e.g. if
Layer 4A files were meant to own the "effective" output helpers as
their primary consumer).

---

### Leftover Debug Logging In `solaredge_modbusqueue.yaml`

Found during the 2026-09-24 YAML header/hygiene audit: `script.
modbus_queue` still contains several `system_log.write ... level: debug`
calls (DEBUG START / LOCK ACQUIRED / BEFORE CHOOSE / END markers) left
over from earlier debugging. Not narrative, just verbose instrumentation
- worth trimming in a future pass if the debug log gets noisy, otherwise
harmless.

---

# 3. Active Investigations

## Modbus Connectivity

Observed Modbus errors (`Cancel send, because not connected`, `No response received after 3 retries`, `Request cancelled outside library`, `transaction_id mismatch`) prompted investigation into whether they correlate with port probe failures, data freshness failures, write bursts, battery control mode changes, or negative price curtailment. The `modbus_busy` stale-lock gap (found 2026-08-28) was fixed 2026-09-20 - see Resolved section below and known_issues_and_fixes.md for current deployment status. The original 48h baseline connectivity-noise rate (27x "Cancel send"/"Repeating call"/"No response", 7x transaction_id mismatch) was investigated separately, below.

Investigation (2026-09-10 through 2026-10-07) found: two real Modbus command-write failures (2026-09-08, 2026-09-10), not just the usual poll/read noise; and that the `solaredge_modbus_multi` integration had migrated its underlying Modbus client library (pymodbus-family -> a `tmodbus`/`modbus-connection`-based one, release v4.0.0, 2026-09-14) at some point before the 2026-09-24 re-check, which changed the error message vocabulary and detection mechanism entirely - making a literal before/after comparison against the original baseline permanently unrecoverable (only a looser like-for-like reading remained possible). Two integration settings were changed live 2026-09-24 as a deliberate intervention - `close_after_polling` (`false` -> `true`) and `scan_interval` (`2s` -> `5s`) - to stop the polling coordinator from contending with `script.modbus_queue` writes for the inverter's single Modbus/TCP connection slot. Multiple follow-up samples (2026-10-02, 2026-10-06, 2026-10-07) found connection-cycle rates behaving as the new settings predict, connectivity episodes (brief "could not connect" drops) brief, self-recovering and non-escalating (not worse than before the settings change), and a 6.7-day `modbus_busy` lock-cycle sample (2026-10-01 to 2026-10-07, spanning a live HA restart) found zero lockups (max 50.4s, see test_plan.md Test S.4).

**Working verdict: adequately resolved** - the transaction-ID-mismatch rate never cleanly recovered to below the original (now unrecoverable, non-comparable) baseline, but the connectivity-episode rate itself - the thing that matters operationally - has stayed stable and self-recovering since the 2026-09-24 settings change. Recommend not continuing to chase a before/after against an unrecoverable baseline, and re-opening only if episode frequency or severity visibly increases.

A briefly-added fourth "unexplained connectivity" entry (triggered by `sensor.max_active_power_1` stalling MPC) was retracted 2026-10-11: that sensor belongs to a separate, cloud-API-based custom integration (`solaredgeoptimizers`), not the local `solaredge_modbus_multi` integration this section tracks (which stayed healthy throughout, confirmed via its own entities updating normally). The real root cause (a SolarEdge monitoring-portal login failure, status 410) is documented in known_issues_and_fixes.md - EMHASS Rejects Optimisation Requests When `sensor.max_active_power_1` Goes Unavailable - instead.

**Still open:**

- `sensor.solaredge_b1_state_of_energy` has had three extended, unexplained outages beyond the normal short nightly blip: ~31 hours (2026-09-16 14:29 - 2026-09-17 21:37, only cleared by an unrelated HA restart), ~12 minutes (2026-09-20 21:26-21:38), and ~2h13m (2026-09-22 14:22-16:34, daytime). Root cause unknown in all three cases and not yet correlated with the connectivity findings above. Operational risk is covered by the Layer 4A guard, which correctly freezes outputs during these outages, but the underlying cause itself is still an open item.
- The short nightly "unavailable" blip (originally seen only around ~22:00, now observed clustering around two times a day - roughly 06:3x-06:5x and 20:1x-20:5x local, 21-33s, affecting multiple entities simultaneously - consistent with a brief whole-integration reconnect) is still not root-caused, though harmless and caught by existing unavailable/unknown guards.

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

### Update 2026-10-06 - Possible cooldown gap flagged, not confirmed

While confirming other test_plan.md items live, a 10-day history pull on
`input_select.emhass_requested_storage_mode` / `input_datetime.
last_grid_charge_command_time` surfaced one sequence worth flagging:

```text
2026-10-04 11:00:01  charge_from_solar_and_grid requested
                      (last_grid_charge_command_time stamped 11:00:01)
2026-10-04 11:15:01  maximize_self_consumption
2026-10-04 11:30:01  discharge_to_maximize_export requested
```

`input_number.grid_charge_export_cooldown_minutes` was confirmed still
at its default 60 at the time of this check, and the stamp was not
updated again between 11:00:01 and 11:30:01 - so on the stamp/timestamp
evidence alone, the DISCHARGE MAX EXPORT branch's cooldown condition
should have been active (cooldown NOT elapsed, only 30 of 60 minutes
passed) and should have held the request back to
`maximize_self_consumption`, per the design in the "Diagnosed root
cause" section above. Instead `discharge_to_maximize_export` was
requested directly.

**Not confirmed as a real defect** - `automation.
emhass_battery_forecast_control`'s stored traces only retain the last 5
runs, so the actual trace for this specific 11:30 run (which would show
the live `cooldown_elapsed` variable and which `choose:` branch
actually matched) is long gone and could not be inspected. A look at
`sensor.mpc_pv_grid_power`/`sensor.mpc_pv_batt_power` around the same
window shows forecast values consistent with a genuine DISCHARGE MAX
EXPORT match (grid_fc very negative, batt_fc positive) appearing within
the same quarter-hour, but not provably at the exact 11:30:01 decision
tick - EMHASS's forecast sensors and the decision engine's 15-minute
trigger don't necessarily update in lockstep, so this could equally be
a timing/sampling artifact rather than the cooldown logic itself
failing. Needs a live trace caught in the act (ideally right after a
charge_from_solar_and_grid stamp, within the following 60 minutes) to
confirm either way - logged here so a future session watching for this
knows what to look for and doesn't have to rediscover it from scratch.

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

A real overnight negative-price event (2026-10-06, 7 activate/deactivate cycles) gave the first live evidence for this item (see test_plan.md Test 3.1/3.2/S.3 and known_issues_and_fixes.md - Negative Price Curtailment - Site Limit Restore Gap). It confirmed price-sign flips correctly toggle `negative_price_active`/`grid_export_blocked`/the switch (7/7), and that `number.solaredge_i1_site_limit` is correctly written to 0 on first activation each time. It also found a real gap: `negative_site_limit_off` never wrote the site limit back off 0 after deactivation, leaving it stale at 0 for hours - empirically confirmed harmless on the day it was found, since real grid export happened normally during the stale-0 window, proving the switch (not the number) is what actually gates export enforcement on this installation.

Per the user's preferred design, the restore-gap was closed by redesign rather than patch (implemented 2026-10-06): the switch (`switch.solaredge_i1_negative_site_limit`) now stays permanently ON, and `number.solaredge_i1_site_limit` itself is varied between 0 (curtailed) and 1,000,000 W (its own max, normal/indifferent export), written on every transition - so there is nothing to forget to restore. Full implementation write-up: known_issues_and_fixes.md - Negative Price Curtailment - Site Limit Restore Gap (changes to `solaredge_modbusqueue.yaml`, `batterycontrol_scripts.yaml`, `negative_price_curtailment.yaml`).

The new mechanism was confirmed live through a real negative-price episode on 2026-10-13 (price oscillated across zero three times within a ~20-minute window; the switch flipped off -> on on the very first crossing, and the site-limit number tracked all three curtail/restore cycles correctly in lockstep, remaining stable through 6+ days and 12+ subsequent Modbus blips afterward) - this closes both the live-mechanism verification and the switch self-heal question. Minor footnote: rapid price-noise cycling right around exactly zero costs a handful of extra Modbus writes per cycle (1-2); not frequent or large enough to warrant a deadband/hysteresis fix on current evidence, but worth keeping in mind given this project's SD-card-wear sensitivity.

**Still open**: no negative-price episode has yet coincided with real PV surplus, so actual export suppression (and resumption) during curtailment has still never been directly observed - only inferred from command correctness. The 2026-10-13 live test was deliberately run during zero PV specifically to make it a safe test, so this one scenario still has not occurred.

---

# 4. Enhanced Diagnostics

## Required Telemetry

Status as of 2026-10-06 (full build-out - see Resolved section below and
known_issues_and_fixes.md - Write-Outcome Telemetry Is Readback-Based):

```text
Modbus write frequency      -> DONE (modbus_write_success/failure_count_*)
Last successful write       -> DONE (input_text.modbus_last_successful_write)
Last failed write            -> DONE (input_text.modbus_last_failed_write)
Last command type            -> already covered pre-existing
                                 (modbus_queue_last_queued/executed_command)
Last command reason          -> PARTIAL: input_text.effective_battery_reason
                                 covers the decision-engine's reason;
                                 modbus_last_failed_reason covers why a
                                 write specifically failed. No single
                                 "why was THIS command issued" field below
                                 the decision-engine level.
Queue length / pending count -> DONE (input_number.modbus_queue_pending_count)
Command delta                 -> DONE (modbus_queue_last_command_delta,
                                 numeric write branches only)
Skipped writes                -> PARTIAL: mode-select skips now counted
                                 (mode_command_skipped_count_total/_today).
                                 Charge/discharge *limit* writes still have
                                 no skip/dedup logic at all - a counter for
                                 that would be telemetry for behavior that
                                 doesn't exist yet. See Write Pressure below.
```

`sensor.battery_mode_command_guard` (added 2026-08-26) still only exposes
the mode-select guard's current state; the new counters above now also
track *how often* it has fired, lifetime and daily.

---

## Recommended Diagnostic Sensors

Status as of 2026-10-06: all built. See entities.md - Layer 1 - Modbus Port/Write Health Diagnostics and Modbus Write/Queue Diagnostic Helpers.

```text
binary_sensor.solaredge_modbus_port_raw      - DONE 2026-10-06
binary_sensor.solaredge_modbus_port_stable   - DONE 2026-10-06

input_text.modbus_last_successful_write      - DONE 2026-10-06
input_text.modbus_last_failed_write          - DONE 2026-10-06
input_text.modbus_last_failed_reason         - DONE 2026-10-06

sensor.solaredge_i1_ac_power_age_seconds     - already existed (2026-09-25,
sensor.solaredge_m1_ac_power_age_seconds       predates this build-out)
binary_sensor.solaredge_modbus_data_fresh
```

Also added, beyond this original list: `binary_sensor.solaredge_modbus_write_unhealthy` (consecutive-write-failure based; its entity_id doesn't match its unique_id `solaredge_modbus_write_health` - see known_issues_and_fixes.md, Entity ID Documentation Mismatches) and `binary_sensor.solaredge_modbus_queue_backlog` (pending-count based).

None of the port/write-health sensors are a real TCP probe or new integration - all are derived from existing entity availability, the pre-existing data-freshness sensor, and the write/queue counters above, per the user's explicit choice to derive from existing signals rather than build a real port probe. See known_issues_and_fixes.md - Write-Outcome Telemetry Is Readback-Based for the documented limitation this implies: a silently-optimistic entity write could read back as "success" even if the underlying Modbus write failed on the wire, so these sensors help distinguish TCP-port-unavailable / stale-session / data-not-updating / queue-overload / transaction-recovery cases, but aren't ground truth.

Note 2026-09-19: a `binary_sensor.solaredge_modbus_data_fresh`-style "age since last update" sensor exists for the inverter/meter AC-power entities (since 2026-09-25) but not yet for the battery state-of-energy sensor specifically - still worth adding if another extended SOC-sensor outage like the 2026-09-16/17 or 2026-09-22 ones (see Modbus Connectivity above) recurs.

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

**HA Cleanup - Dead Sliding-Window Subsystem Removed** (found and removed
2026-11-02) - a pre-refactor EMHASS day-ahead/MPC mechanism was confirmed
fully dead during a project-wide unused-entity audit: `script.
emhass_update_sliding_window` and `script.set_mpc_soc_final_target`
(`emhass_scripts.yaml`, both last_triggered 2026-07-09, no call site
anywhere), and the 5 helpers only they ever wrote - `input_number.
mpc_window_start`/`mpc_window_end`/`mpc_prediction_horizon`/
`mpc_soc_final_target` and `input_datetime.day_ahead_forecast_end`
(`emhass_helpers.yaml`), all frozen at their initial values. One of the
dead scripts also referenced `sensor.soc_batt_forecast`, which returns
`ENTITY_NOT_FOUND` - further confirming staleness. Removed from both
files; see known_issues_and_fixes.md for the full write-up. Separately,
4 purely-registry "ghost" automations/script that used to call this
subsystem (`automation.emhass_day_ahead_optimization`, `automation.
emhass_update_sliding_window`, `automation.emhass_mpc`, `script.
run_mpc_optimization` - all `unavailable`, `restored: true`, no YAML
backing anywhere in this project) cannot be removed by a doc edit and
need manual deletion via the HA UI (Settings -> Devices & Services ->
Entities, filter "unavailable"). A dead dashboard row referencing the
removed `input_number.mpc_prediction_horizon` helper (the "EMHASS -
Status" entities card) also needs manual removal in the HA UI/dashboard
editor - this session has no live write access to Home Assistant,
dashboards included, so neither the ghost-entity deletion nor the
dashboard-row removal could be done directly.

**HA Cleanup - Unused Manual-Override Booleans Removed** (found and
removed 2026-11-02) - `input_boolean.battery_safety_override_active`,
`battery_force_discharge`, `battery_force_charge`,
`battery_discharge_blocked`, and `battery_charge_blocked`
(`safety_and_watchdog_helpers.yaml`), flagged as possibly unused during
the 2026-09-24 YAML header/hygiene audit (see the former "Possibly
Unused Safety-Override Input Booleans" item, previously here in section
2), were confirmed during the 2026-11-02 audit to have no read or write
anywhere in the project (including `batterycontrol_automations.yaml`,
not checked in the original 2026-09-24 pass) and no dashboard card
reference either. Removed per the user's explicit choice.

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

**EMHASS Outage Fallback Guard** (found and fixed 2026-09-22) - an 8+
hour EMHASS addon outage (stuck rebuilding after a power-outage DNS
failure) left Layer 4A (`battery_forecast_control.yaml`) computing
decisions from EMHASS-dependent entities that were silently reading as
fake zeros, causing a real charge/discharge oscillation for 6+ hours
against genuinely high prices (confirmed via history: requested mode and
discharge limit alternating almost every 15-minute tick, SOC sawtoothing
19-34%). Fixed with an `sensor.emhass_health`-based guard skipping the
whole run while EMHASS is stalled/failed, plus an availability check on
the price sensor specifically so a missing price feed can't be misread
as "free". Also added an addon-level auto-restart watchdog. See
known_issues_and_fixes.md and EMHASS Addon Build-On-Start Fragility
above (still open) for the underlying addon fragility this doesn't fix.

**Add Queue Telemetry** (implemented 2026-10-02) - `script.modbus_queue`
(`solaredge_modbusqueue.yaml`) was previously opaque: no visibility into
run frequency, busy-lock wait time, or last action. Added lifetime/daily
command and timeout counters, last-wait/last-execution-duration sensors,
and last-queued/last-executed-command text helpers - see entities.md -
Layer 4C - Queue Telemetry / Queue Automations for the full entity list,
and `solaredge_modbusqueue.yaml` for the implementation. The timeout
counters increment only when the 10s `wait_template` genuinely runs out,
not when the lock clears normally. Not yet correlated against the Modbus
Connectivity investigation above - a natural next step once a real
connectivity episode happens with this telemetry live.

**Enhanced Diagnostics - Full Build-Out** (implemented 2026-10-06,
deployed live and confirmed working 2026-10-07) - the remaining open
items from section 4 above (Required Telemetry, Recommended Diagnostic
Sensors) were built in one pass, per the user's explicit request to put
the new entities in Layer 1 ("safety and watchdog") and to derive
port/write health from existing signals rather than a new TCP probe or
integration. Added write-outcome/queue counters and text helpers to
`safety_and_watchdog_helpers.yaml` (Layer 1); new port/write-health/
backlog binary sensors to `watchdog_sensors.yaml` (Layer 1); and a daily
counter-reset automation to `watchdog_automations.yaml`.
`solaredge_modbusqueue.yaml` (Layer 4C) now classifies every write as
success/failure/unmatched_service via before/after entity readback;
`apply_effective_battery_control.yaml` (Layer 4B) increments the
mode-skip counters from the Mode Command Guard's existing `else:`
branch. Deliberately not built: a real TCP port probe, and any
skip/dedup counter for charge/discharge *limit* writes (no such skip
logic exists - see Write Pressure above). See known_issues_and_fixes.md
- Write-Outcome Telemetry Is Readback-Based for the full design and its
documented limitations, and entities.md - Layer 1 for the complete
entity list.

Live verification (2026-10-07) found the write-outcome classification
initially misfiring on roughly half of numeric writes (a fixed 2s
readback check firing before some entities, which were genuinely
poll-confirmed rather than optimistic, had settled) - fixed the same day
by moving the wait-and-classify step out of `script.modbus_queue` (which
runs `mode: queued`, so an inline wait there would delay the next queued
command) into a new, independently-queued `script.modbus_write_verify`
using `wait_template` instead of a fixed delay. Also found and fixed the
same day, unrelated: three documented entity_ids didn't match what Home
Assistant actually assigned, because entity_id is derived from a
template entity's `name:`/an automation's `alias:`, not from
`unique_id`/`id:` - see known_issues_and_fixes.md, Entity ID
Documentation Mismatches. A first redeploy left the fix unwired; a
subsequent HA update/restart reloaded the package correctly, and the
next real write confirmed the new dispatch path working end-to-end.
Status: confirmed wired in and working live 2026-10-07.

**Modbus Queue Pending-Count Leak** (found and fixed 2026-10-19,
confirmed live 2026-10-20) - `input_number.modbus_queue_pending_count`'s
resting floor never returned to 0 (reached 67 on 2026-10-17, reset by
what was most likely an HA restart on 2026-10-18, then climbed again to
10 by 2026-10-19), tracking `modbus_queue_timeout_count_total` 1:1 -
strong evidence every lock-wait timeout left a permanent, uncorrected +1
that the normal end-of-run decrement never claws back. Diagnostic-only
impact (real command throughput stayed normal throughout), but
`binary_sensor.solaredge_modbus_queue_backlog` had been reporting a
false "on" (backlog) for 33+ hours. Fixed the same day with a
compensating decrement inside the lock-wait-timeout branch itself
(floored at 0, same pattern as the existing end-of-run decrement).
Deployed live 2026-10-20; a real lock-wait timeout occurred that same
night and the floor returned cleanly to its prior resting value instead
of climbing further, confirming the fix. The pre-existing resting floor
of 10 itself is unchanged (the fix stops future leaks, it doesn't
retroactively repay the old debt), so the backlog sensor will keep
reading a false "on" until that resting value is manually cleared or an
HA restart resets it - a one-time cleanup, not a further code fix. See
known_issues_and_fixes.md for full detail. `test_plan.md` Test 6.4
upgraded to PASS.

---

# 6. Documentation Housekeeping

### YAML File Header Standardisation (2026-09-24)

All 19 package YAML files were audited, given a standard header (see
`claude/yaml_file_header_template.md` in the project docs for the
reusable template and field definitions), and had any inline fix
narrative / design rationale trimmed to short pointer comments -
`# See known_issues_and_fixes.md - <Entry Name>` for fixes,
`# See architecture.md - <Section>` for design/architecture reasoning.
Most files already followed this pattern from prior cleanup passes;
genuinely new content extracted by the audit is a handful of Cleanup /
Technical Debt items above (rest_command.publish_data,
trigger_nordpool_forecast, the unused `ev` variable, the `effective_*`
helper file placement, the possibly-unused safety-override booleans -
removed 2026-11-02, see Resolved above, and the leftover debug logging
in `solaredge_modbusqueue.yaml`) plus one known_issues_and_fixes.md
update (EV Charging Sensor Swap - stale trigger entity_id follow-up).
No new known_issues_and_fixes.md, README.md, or architecture.md
*entries* were needed beyond that one update - existing entries already
covered essentially everything found inline.

### HA Cleanup Audit (2026-11-02)

A project-wide audit for dead/unused entities, automations, scripts, and
helpers, cross-checked against live HA state (read-only access) and both
dashboards. Found: 4 pure registry "ghost" entities with no YAML backing
(need manual HA UI deletion, not a doc edit); a dead sliding-window
subsystem and 5 unused override booleans (both removed this pass - see
Resolved above); a dead dashboard row; and several already-known
possibly-dead items reconfirmed but not touched (`rest_command.
publish_data`, `shell_command.trigger_nordpool_forecast`,
`input_boolean.grid_export_limited`, the unused `ev` variable, and 7
unused EMHASS forecast sensors - see their respective items above). The
user scoped this round to the sliding-window subsystem and the override
booleans only.

---

Implemented fixes and known issues: see `known_issues_and_fixes.md`.