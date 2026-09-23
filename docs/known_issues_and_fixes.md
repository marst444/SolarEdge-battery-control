# Known Issues and Fixes

Companion to `roadmap.md` (tracks what's still open). This is the durable
log of what's actually been found and fixed - bugs diagnosed, fixes
implemented, deployment/verification status. Fix narrative lives here,
not as comments in the YAML - the YAML should read as production config,
not a changelog.

Most recent first within each section. Each entry's Status line says
whether a fix is deployed and/or verified live, or only implemented in
these project docs so far.

---

(None currently open. The branch-coverage gap found 2026-09-10 was fixed
2026-09-15 and confirmed firing live - see Branch-Coverage Gaps below.
Its related, unconfirmed corner - `grid_fc < 0` together with `batt_fc <
0`, a contradictory-forecast edge case - is still uncovered by any
branch, not observed live, and not proposed for a blind fix. The Layer
4A SOC-unavailable guard found 2026-09-19 was fixed the same day - see
Fixed Issues below. The modbus_busy stale-lock gap found 2026-08-28 was
fixed 2026-09-20 - see Modbus Queue Lock Release Not Exception-Safe
below.)

---

# Fixed Issues

## Modbus Queue Lock Release Not Exception-Safe

Found 2026-08-28 (test_plan.md Test 6.3 / 7.5 review): `script.
modbus_queue`'s "Release Modbus lock" step (`input_boolean.turn_off
modbus_busy`) sat at the end of a plain linear `sequence:`, unprotected
against an exception raised inside the preceding `choose:` block. When a
dispatched SolarEdge service call failed (e.g. the 2026-08-27 06:45:06
"Connection failed: Modbus Error: [Connection] Not connected" write
failure), the script aborted before reaching the release step and
`input_boolean.modbus_busy` was left stuck "on" - confirmed via history
not a permanent deadlock (self-healed after ~30 min via the next
invocation's 10s `wait_template`/`continue_on_timeout`), but every
command in between waited out that timeout instead of running
immediately.

A naive fix (bare `continue_on_error: true` on the release step alone)
was identified early on as unsafe: a 48h error-log review for the same
period found a non-trivial baseline of Modbus connectivity noise (27x
"Cancel send"/"Repeating call"/"No response", 7x transaction_id
mismatch), and releasing the lock instantly on every failure with no
pacing could let a queued backlog (`mode: queued, max: 10`) drain
back-to-back against a connection that's still struggling - plausibly
what caused a past, unsuccessful attempt at this same fix to flood the
system with commands. The fix was deliberately deferred for several
weeks on that basis (see roadmap.md history) in favour of lower-risk
fixes elsewhere.

Fixed 2026-09-20 in `solaredge_modbusqueue.yaml`: added `continue_on_error:
true` to the `choose:` step itself, so an exception inside any branch no
longer aborts the script - it logs (HA's default behaviour for a caught
error) and falls through to the steps after the choose block. The
previous ~13 per-branch trailing `delay: "00:00:02"` steps (one at the
end of nearly every mode/limit branch) were removed and replaced with a
single uniform `delay: "00:00:02"` placed once, after the whole `choose:`
block and before the lock-release step - so it runs identically whether
the dispatched command succeeded or was caught by `continue_on_error`.
This keeps the same backlog-draining pace on the failure path as on the
success path, addressing the flood-risk concern that blocked the earlier
attempt, while making the lock release itself unconditional. The one
internal delay inside the `negative_site_limit_on` branch (sequencing two
writes within that single command) was left as-is - it serves a different
purpose than the removed trailing delays.

Status: implemented in this project's docs 2026-09-20. Not yet deployed
or verified live. Deployment needs copying to the live config and a
"Reload Automations/Scripts" (or full restart if that doesn't take,
per the precedent below). Verification is harder than the automation
fixes in this file: this is a `script:`, not an `automation:`, so there's
no trace history to read, and the failure condition (a real Modbus write
error) is intermittent and can't be triggered on demand - the practical
check is watching `input_boolean.modbus_busy`'s on/off history after
deployment and confirming it never again sits "on" for the ~30-minute
stuck-lock pattern seen in the original 2026-08-27 incident, ideally
around a future night with the usual dropout.

---

## Layer 4A SOC-Unavailable Guard

Found 2026-09-19 during a routine multi-day log/Modbus review. History
for `sensor.solaredge_b1_state_of_energy` shows a real ~31-hour gap:
last `unavailable` at 2026-09-16 14:29:09 local, no further state at all
(not even a repeated `unavailable`) until it recovered to 28.9% at
2026-09-17 21:36:51 local - right after a Home Assistant restart
(supervisor auto-updated 2026.09.0 -> 2026.09.2 that evening;
`current_recorder_run` shows the restart at 19:35:42 UTC). This was
isolated to the battery state-of-energy entity specifically, not a whole
integration or HA outage: `sensor.solaredge_i1_ac_power`,
`select.solaredge_i1_storage_control_mode`, and unrelated automations
like `ev_charging_discharge_control` kept running normally throughout
(only two ~25-30s reconnect blips, the known nightly pattern).

`calculate_effective_battery_control` (`safety_limits_and_override.yaml`)
has an unavailable/unknown guard on this exact sensor, added 2026-09-03 -
it held perfectly: `input_select.effective_storage_mode` and both
`effective_*_limit` helpers stayed frozen at their last good values
(`maximize_self_consumption`) for the entire 31 hours, confirming that
fix generalizes well beyond the short nightly blips it was built for.

`emhass_battery_forecast_control` (`battery_forecast_control.yaml`,
Layer 4A) had **no equivalent guard**. Its `soc` variable has the same
`| float(0)` fallback pattern the 2026-09-03 fix addressed elsewhere.
History is consistent with it running every 15 minutes throughout the
outage on a fallback `soc=0` (which always satisfies `below`):
`emhass_requested_storage_mode` flipped to `solar_power_only` within a
minute of the sensor going unavailable and then never changed again for
31 hours, and `input_number.emhass_requested_charge_limit` was frozen at
exactly `0.0` for the same span (traces from that period are long gone,
so this is inferred from the entity history, not directly confirmed via
a trace). The consequence was benign here purely by luck of which branch
that fallback happened to land in - `solar_power_only` (charge=0,
discharge=0) is conservative, not dangerous - but it's the same class of
bug as the original SOC-Sensor Unavailable False-Trigger issue, and a
different fallback path (e.g. one landing in the price-based grid-charge
branch) could have produced a real spurious command over 31 hours instead
of one that happened to be safe.

Fixed in `battery_forecast_control.yaml`: added the same
automation-level unavailable/unknown guard already proven on
`calculate_effective_battery_control`, as a second `condition:` entry on
`emhass_battery_forecast_control` (alongside the existing
"Remote Control" gate) requiring `sensor.solaredge_b1_state_of_energy`
to not be `unavailable` or `unknown`. The whole run is now skipped while
the sensor reads bad, instead of computing a fake `soc=0` - matching the
established pattern exactly, no other logic changed.

Status: implemented in this project's docs 2026-09-19. Not yet deployed
or verified live - next step is copying to the live config, reloading
automations (see Branch-Coverage Gaps below for why a reload, not just
a doc match, needs to be confirmed), and then confirming via a live
trace that a run during any future SOC-unavailable window is actually
skipped rather than executed with `soc=0`.

---

## EV Charging Sensor Swap

`binary_sensor.ev_charging_on` was found stuck `unavailable` since at
least 2026-09-03 (dead template entity, `restored: true`, no
`config_entry_id`) - so `is_state(...,'on')` could never fire the
EV-charging branches in `calculate_effective_battery_control`
(safety_limits_and_override.yaml) or `ev_charging_discharge_control`
(batterycontrol_automations.yaml), even during real EV charging sessions.
Swapped to `binary_sensor.ev_charging_active` (a separate `EV_charge`
package's sensor, only "on" while current is actually flowing) in
safety_limits_and_override.yaml, battery_forecast_control.yaml,
batterycontrol_automations.yaml, and test_templates.md.

Status: deployed and **verified live** 2026-09-15 - a real EV session
(12:30:46-13:31:26 local) showed the sensor, `effective_battery_reason`,
and the discharge-limit automation trace all changing in lockstep
(discharge limit set to 100 on, restored to dynamic value off).

---

## Align Safety SOC Limits And EMHASS SOC Limits

Safety layer and EMHASS each kept their own SOC min/max, only agreeing by
coincidence (`battery_minimum_percent`/`battery_maximum_state_of_charge`
hardcoded in emhass_scripts.yaml). Fixed 2026-09-03: both MPC and
day-ahead scripts now read `input_number.minimum_state_of_charge` /
`maximum_state_of_charge` via template, same fallback defaults as before,
so behaviour only diverges if the helper is actually changed.
`maximum_power_from_grid`/`maximum_power_to_grid` remain hardcoded - no
safety-layer equivalent to align with (see roadmap.md).

Status: deployed, confirmed 2026-09-04 (live file matched verbatim).
Runtime confirmation that a live MPC/day-ahead payload reflects the
helper values is still outstanding.

---

## Modbus Connectivity - SOC-Sensor Unavailable False-Trigger

Found 2026-09-03: `sensor.solaredge_b1_state_of_energy` briefly drops to
`unavailable`/`unknown` most nights (~22:00-22:10 local, 6.5-43.8s - a
genuine Modbus TCP connect failure to the inverter; the ~22:00 clustering
itself still unexplained). Severity: `calculate_effective_battery_control`
is state-triggered on this sensor, and its `| float(0)` fallback turned
each dropout into `soc=0`, satisfying the SOC<10 "CRITICAL RECOVERY"
branch and commanding a real hardcoded 3000W grid-charge for the
glitch's duration - confirmed via `input_boolean.modbus_busy` and
`effective_storage_mode` history matching all 6 dropout nights.

Fixed by adding an automation-level unavailable/unknown guard on
`calculate_effective_battery_control` (skips the whole run instead of
computing a fake `soc=0`) and the same guard on `battery_high_soc_hold`
(same fallback pattern, would have falsely exited an active high-SOC
hold).

Status: deployed and verified - 0/6 spurious triggers across the
following week's dropouts (2026-09-04 to 09-09). The underlying Modbus
dropout itself remains open in roadmap.md (new clue: each night's
episode lands ~11-13 min earlier than the last).

---

## Midday Charge/Discharge Oscillation - Grid Charge Export Cooldown

Implemented 2026-08-27: a hysteresis cooldown
(`input_number.grid_charge_export_cooldown_minutes`, default 60) stamped
whenever `charge_from_solar_and_grid` fires; DISCHARGE MAX EXPORT only
sells back to the grid once the cooldown has elapsed, otherwise falls
back to `maximize_self_consumption`. Full design: architecture.md -
Layer 4A - Grid Charge Export Cooldown. Addresses the *consequence* of
the 2026-08-27 oscillation, not its underlying causes (EMHASS MPC plan
chatter, cooldown-duration tuning) - those remain open in roadmap.md.

---

## Midday Charge/Discharge Oscillation - PV/Load Sampling Smoothing

Added `sensor.pv_production_smoothed`/`house_load_smoothed` (4-minute
trailing mean) 2026-08-28, used for every mode-selecting branch, while
Maintain Zone's charge-ceiling choice kept raw instantaneous readings
(`pv_now`/`house_load_now`) since that choice can't cause mode-flapping.
Deploying required a full HA restart, not just a reload -
`platform: statistics` is a legacy, non-hot-reloadable sensor platform.

Status: deployed and **verified live** 2026-08-29 - zero oscillation
across a full morning, plus two ticks (07:45, 08:45) where the
raw/smoothed split demonstrably prevented a stale-average charge-ceiling
open. Below-SOC/Clipped-Solar behaviour under the smoothed variables
still awaits a real exercising event. (A branch-coverage gap was also
spotted during this review - see below.)

---

## Midday Charge/Discharge Oscillation - Branch-Coverage Gaps

Three gaps found and fixed in `battery_forecast_control.yaml`'s
top-level `choose:` (which has no `default:`), each a case that matched
no branch and silently left the requested mode/limits unchanged for that
cycle:

1. **DISCHARGE MAX EXPORT, `batt_fc==0`** (found 2026-08-29, fixed
   2026-09-03): widened `batt_fc > 0` to `>= 0`.
2. **BELOW-SOC no-PV/batt_fc==0 catch-all** (found + fixed 2026-09-03):
   dropped an unnecessary `grid_fc > 0` requirement on the final
   BELOW-SOC branch.
3. **DISCHARGE MIN IMPORT/MAX SELF-CONSUMPTION, `batt_fc>0`** (found
   2026-09-10, fixed 2026-09-15): dropped the `batt_fc <= 0` clause,
   widening to `above and grid_fc>=0 and soc>soc_min` (the action was
   already sign-agnostic).

2026-09-10 spot-check: gap 1 confirmed firing live; gap 2 not yet
exercised (no `below` trace in the retained window).

Gap 3 had an eventful rollout on 2026-09-15: traces right after
"deployed" (5 consecutive ticks, SOC genuinely above target) showed the
branch still evaluating false, matching the *old* pre-fix condition
exactly. The user then pasted the live file - byte-identical to this
project's copy, fix included - so `ha_eval_template` was used to test
the fixed condition directly against live states (result: True) versus
what the trace showed (false), proving the deployed automation was
running stale, not-yet-reloaded config (same failure mode as the
PV/Load Smoothing restart issue above, just without needing a full
restart this time). After the user reloaded automations, the next cycle
(19:30 UTC) fired correctly: branch evaluated true, mode set to
`maximize_self_consumption`, limits updated, and it correctly cascaded
into the inverter-apply automation.

Still open, not fixed (not observed live, no blind fix proposed): the
mirror-image case, `(maintain or above)` with `grid_fc < 0` and
`batt_fc < 0` together.

Status: all three gaps deployed; 1 and 3 **verified live** (3 as of
2026-09-15 ~19:30 UTC), 2 still unexercised.

---

## Write Pressure - Mode Command Guard

Implemented 2026-08-26:
`apply_effective_battery_control_to_solaredge_inverter` skips the
mode-select command and the 15-minute command-timeout reset whenever the
effective mode is already `maximize_self_consumption` and both the
inverter's live command mode and its configured default mode confirm
that's safe to skip - only the charge/discharge limit writes still go
out in that case. `sensor.battery_mode_command_guard` exposes whether
it's currently active.

Status: deployed and confirmed working (test_plan.md Test 7.6)
2026-08-27. Modbus error counts look pre-existing rather than
guard-caused, but before/after impact on the overall error rate hasn't
been quantified.