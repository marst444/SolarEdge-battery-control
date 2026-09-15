# Known Issues and Fixes

Companion to `roadmap.md`. `roadmap.md` tracks what's still open (future
work, cleanup, active investigations); this document is the durable log
of what's actually been found and changed - bugs diagnosed, fixes
implemented, and their deployment/verification status. Explanatory
narrative for a fix lives here, not as comments in the YAML - the YAML
should read as production config, not a changelog.

Most recent first within each section. Check each entry's own status
note for whether a fix has been deployed to the live instance and/or
verified live yet, or is only implemented in these project docs so far.

---

# Known Issues - Not Yet Fixed

## Branch-Coverage Gap: `above` / `grid_fc >= 0` / `batt_fc > 0` (battery_forecast_control.yaml)

### Found 2026-09-10 - deeper charge/discharge review

Caught live in an automation trace of `automation.emhass_battery_forecast_control`
(2026-09-10, ~17:30 local): `soc=56.7%`, `above=true`, `grid_fc=0`,
`batt_fc=564` matched none of the top-level `choose:` branches. The
"above" zone splits into two siblings by `grid_fc` sign:

```text
DISCHARGE MAX EXPORT: (maintain or above) and soc>soc_min and
  grid_fc<0 and batt_fc>=0

DISCHARGE MIN IMPORT/MAX SELF-CONSUMPTION: above and grid_fc>=0
  and batt_fc<=0 and soc>soc_min
```

The `batt_fc <= 0` clause on the second branch is the gap: with
`grid_fc >= 0` and `batt_fc > 0` (discharge forecast, any grid sign
that isn't export), neither branch matches, there's no top-level
`default:`, and the automation leaves `emhass_requested_storage_mode`/
limits unchanged for that cycle. Same family as the two gaps fixed
2026-09-03 below (DISCHARGE MAX EXPORT `batt_fc==0`, BELOW-SOC catch-all
`grid_fc<=0`).

Proposed fix (not yet implemented, pending approval): drop the
`batt_fc <= 0` clause from the DISCHARGE MIN IMPORT/MAX SELF-CONSUMPTION
branch, widening it to `above and grid_fc >= 0 and soc > soc_min`. Its
action (maximize_self_consumption with dynamic charge/discharge limits)
is already sign-agnostic, and this mirrors how DISCHARGE MAX EXPORT
already accepts any `batt_fc >= 0`.

Related, unconfirmed corner noted while diagnosing this: `grid_fc < 0`
together with `batt_fc < 0` (export forecast while the battery is also
forecast to charge - a contradictory-forecast edge case) is still
uncovered by either branch. Not observed live; flagged for awareness
only, not proposed for a blind fix.

---

# Fixed Issues

## EV Charging Sensor Swap

### Found + fixed 2026-09-15 - `binary_sensor.ev_charging_on` was dead; switched to `binary_sensor.ev_charging_active`

Found while chasing why the EV-charging branches never fired despite the
EV genuinely charging (raised during the 2026-09-10 deeper
charge/discharge review). `binary_sensor.ev_charging_on` was confirmed
stuck in state `unavailable` continuously since at least 2026-09-03,
with `restored: true` in its attributes - the signature of an entity
whose backing integration/template never actually sets up; HA just
re-stamps a placeholder "unavailable" across restarts (confirmed
re-stamped again at a 2026-09-10 20:10:47 restart, still unavailable).
Its entity-registry `platform` is `"template"` with no `config_entry_id`,
and its definition isn't in any of this project's tracked docs, so it's
either a UI-created Template Helper or a YAML package outside this
project's scope - not chased further since the fix is to stop depending
on it.

This fully explained the missing EV branches: `is_state(...,'on')` can
never be true against a permanently-unavailable sensor, so (a) the
"EV CHARGING" branch in `calculate_effective_battery_control`
(`safety_limits_and_override.yaml`) could never fire, and (b) the
standalone `automation.ev_charging_discharge_control`
(`batterycontrol_automations.yaml`) - which pulls
`emhass_requested_discharge_limit` down to 100 while the EV charges -
was both triggered by state changes on this dead sensor and conditioned
on its literal state, so it had been silently doing nothing for at
least the full week under review, even during real EV charging
sessions.

User identified the correct live replacement: `binary_sensor.
ev_charging_active`, from a separate package called `EV_charge`, which
only goes "on" while current is actually flowing to the EV. Confirmed
live and healthy (state `off`, no `unavailable`/`restored` markers) via
`ha_search` before swapping.

Swapped `binary_sensor.ev_charging_on` -> `binary_sensor.ev_charging_active`
in:

```text
safety_limits_and_override.yaml   - calculate_effective_battery_control's
                                     ev_charging variable
battery_forecast_control.yaml     - the ev variable (currently unused by
                                     any condition in this automation;
                                     kept for parity/possible future use)
batterycontrol_automations.yaml   - ev_charging_discharge_control's
                                     trigger and both branch conditions
test_templates.md                 - the EV Charging Active diagnostic line
```

Status: implemented in these project docs 2026-09-15. Not yet deployed
to the live instance. Once deployed, confirm during the next real EV
charging session that `input_text.effective_battery_reason` shows "EV
charging - battery discharge limited" and
`input_number.emhass_requested_discharge_limit` drops to 100 for the
duration. Logged as a pending test_plan.md item once live.

---

## Align Safety SOC Limits And EMHASS SOC Limits

Safety layer and EMHASS used to maintain separate minimum (and maximum)
SOC settings - examples: `input_number.minimum_state_of_charge` /
`battery_minimum_percent`. Fixed by using a single source of truth for
both.

### Update 2026-09-03 - Fixed: EMHASS now reads the same minimum-SOC helper

In `emhass_scripts.yaml`, both `generate_emhass_energy_plan_mpc_pv` and
`generate_emhass_energy_plan_dh_pv` had `battery_minimum_percent: 20`
hardcoded in their "Configure critical settings" step - a separate,
independently-maintained copy of the same 20% floor the safety layer
already reads from `input_number.minimum_state_of_charge` (`soc_min` in
`safety_limits_and_override.yaml` and `battery_forecast_control.yaml`).
The two values only agreed by coincidence: changing the helper via the UI
changed what the safety layer treated as the floor, but left EMHASS
silently still planning down to a stale hardcoded 20%, so EMHASS's own
optimisation horizon (`soc_final`) and its `battery_minimum_state_of_
charge` payload field could diverge from the real safety floor.

Fixed by changing `battery_minimum_percent: 20` in both scripts to
`battery_minimum_percent: "{{ states('input_number.minimum_state_of_
charge') | float(20) }}"` - same `| float(20)` fallback and default value
as before if the helper is ever unavailable, so behaviour is unchanged
at the current default (20) and only diverges going forward if the
helper is actually changed, which is exactly the point.

### Update 2026-09-03 - Fixed: EMHASS now reads the same maximum-SOC helper too

Same fix applied, per follow-up request, to the other half of the pair:
`battery_maximum_state_of_charge: 0.9` was also hardcoded in both
scripts' `payload`, duplicating `input_number.maximum_state_of_charge`
(`soc_max` in `safety_limits_and_override.yaml` and `battery_forecast_
control.yaml`). Changed to `"{{ states('input_number.maximum_state_of_
charge') | float(90) / 100 }}"` in both scripts - same `| float(90)`
fallback/default as the helper's own `initial: 90`, so behaviour is
unchanged unless the helper is actually changed. `maximum_power_from_
grid` and `maximum_power_to_grid` remain hardcoded (15000/5000) - those
have no equivalent safety-layer helper to align with, so they're pure
"Centralise EMHASS Configuration" scope (see roadmap.md), not this one.

Status: deployed - confirmed 2026-09-04 when the user pasted the live
`emhass_scripts.yaml` content and both changes were present verbatim.
Runtime confirmation that a live MPC/day-ahead payload actually reflects
the helper values (rather than just the deployed source matching) is
still outstanding.

---

## Modbus Connectivity - SOC-Sensor Unavailable False-Trigger

### Update 2026-09-03 - NEW, higher-severity finding: nightly SOC-sensor dropout causes Layer 1 to spuriously command a real grid-charge

Reviewing 2026-08-29 through 2026-09-03 (5 days) surfaced a recurring
pattern distinct from anything logged under Modbus Connectivity in
roadmap.md: `sensor.solaredge_b1_state_of_energy` briefly goes
`unavailable` -> (usually) `unknown` -> recovers, for 6.5-43.8 seconds, on
**5 of the last 6 nights**, always in a tight ~22:00-22:10 local window:

```text
2026-08-29 22:02:31  39.0s  recovered to 76.67%
2026-08-30 22:07:41  41.5s  recovered to 87.78%
2026-08-31 22:00:15  43.8s  recovered to 72.22%
2026-09-01 22:09:05  40.9s  recovered to 78.89%
2026-09-01 22:48:07   6.5s  recovered to 76.67%  (2nd episode same night, no "unknown" step)
2026-09-02 22:06:58  36.8s  recovered to 68.89%
```

Cross-referencing the 2026-09-02 22:06:58 episode against the error log
confirms the trigger: `Error fetching SolarEdge Coordinator data: Modbus/
TCP connect to 192.168.10.4:1502 failed` at 22:06:58.554, recovering at
22:07:17.914 - a genuine, brief TCP-level connect failure to the inverter,
same family as the connectivity noise already tracked under Modbus
Connectivity in roadmap.md, not a new communication fault. No HA-side
automation/script is scheduled in the 22:00-22:10 window, so the ~22:00
clustering itself is still unexplained - possibly network-level (DHCP
lease renewal, router/switch housekeeping, Wi-Fi channel change) rather
than anything in this package.

The severity is what's new here, not the connectivity blip itself. Read
directly from `safety_limits_and_override.yaml`:
`automation.calculate_effective_battery_control` is **state-triggered**
(not time-based) on a list including `sensor.solaredge_b1_state_of_energy`,
and its first action step sets `soc: "{{ states('sensor.solaredge_b1_
state_of_energy') | float(0) }}"`. When the sensor is `unavailable`, that
Jinja fallback silently makes `soc = 0`, which satisfies the very first
`choose:` branch - `soc < 10` "CRITICAL RECOVERY" - and commands
`effective_storage_mode = charge_from_solar_and_grid` with a hardcoded
`effective_charge_limit = 3000` (W) for the full duration of the glitch.
This was confirmed against live history: `input_boolean.modbus_busy`
toggles exactly at each episode's recovery moment (a real command was
dispatched to the inverter), and `effective_storage_mode` shows a matching
brief `charge_from_solar_and_grid` blip at each of the six timestamps
above, reverting via the `default:` branch as soon as the automation
re-fires on the sensor's real value returning.

### Update 2026-09-03 - Fix implemented: automation-level unavailable/unknown guard

Fixed in `safety_limits_and_override.yaml`. Added an automation-level
`condition:` block on `calculate_effective_battery_control`, right after
`trigger:` and before `action:`, requiring
`sensor.solaredge_b1_state_of_energy` to NOT be `unavailable` or
`unknown` (via `condition: not` wrapping a `condition: state` check).
When the SOC sensor is in either bad state, the entire run is skipped -
none of the `choose:` branches evaluate, `soc` is never computed as a
fallback 0, and the previous `effective_*` values are left untouched
until the automation re-fires on the sensor's next real state change
(`mode: restart` on this automation makes that immediate). This guards
both the SOC<10 and SOC<soc_min branches with a single condition, since
neither can be reached at all while the sensor reads bad.

`battery_high_soc_hold` has the identical `| float(0)` fallback pattern
on the same sensor - a false `soc=0` there satisfies the "Exit hold"
branch (`soc <= soc_max-3`) and would release an already-active high-SOC
hold based on a fake reading. Same fix applied here too.

### Update 2026-09-10 - Confirmed durably effective across a full week

The deeper charge/discharge review re-checked all 6 nightly SOC-dropout
episodes from 2026-09-04 through 2026-09-09 against `effective_storage_
mode` history. Result: **0/6 spurious `charge_from_solar_and_grid`
triggers** - the guard held cleanly every single night. The underlying
Modbus connectivity dropout itself is unchanged (still 6/6 nights,
38-45s each - this fix was never meant to address that, only the false
command it was causing), and a new pattern was noticed in the dropout
timing itself: each night's episode lands roughly 11-13 minutes earlier
than the previous night's (drifting ~22:00 -> ~21:13 over the 6 nights) -
a fresh clue for the still-open, un-root-caused Modbus Connectivity
investigation in roadmap.md, not yet chased further.

Status: deployed and verified live, durably effective over a full week
of real nightly occurrences.

---

## Midday Charge/Discharge Oscillation - Grid Charge Export Cooldown

### Update 2026-08-27 - Fix implemented

A **Grid Charge Export Cooldown**, following the same hysteresis pattern
already used by `battery_high_soc_hold` (`battery_forecast_control.yaml`,
`batterycontrol_helpers.yaml`). Whenever `charge_from_solar_and_grid`
fires, `input_datetime.last_grid_charge_command_time` is stamped. The
`DISCHARGE MAX EXPORT` branch only executes `discharge_to_maximize_export`
if `input_number.grid_charge_export_cooldown_minutes` (default 60) has
elapsed since that stamp; otherwise it falls back to
`maximize_self_consumption` with the normal dynamic discharge limit, so
the battery serves house load instead of being sold straight back to the
grid. Full design and rationale: architecture.md - Layer 4A - Grid
Charge Export Cooldown.

This is a circuit breaker on the *consequence* (immediate loss-making
reversal) of the 2026-08-27 midday oscillation, not a fix for its
underlying causes (EMHASS MPC plan chatter, cooldown-duration tuning) -
those remain open, see roadmap.md - Active Investigations - Midday
Charge/Discharge Oscillation.

---

## Midday Charge/Discharge Oscillation - PV/Load Sampling Smoothing

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

### Update 2026-08-28 - Split smoothed vs instantaneous by decision type

Refined per feedback: smoothing should only apply to the storage MODE
decision (an infrequent, "heavy" change worth protecting from noise), not
to the charge-limit ceiling chosen inside Maintain Zone, which should
still react to a real short PV/load change at each 15-min tick rather
than lag behind a 4-minute average. `battery_forecast_control.yaml` now
defines both `pv`/`house_load` (smoothed, used by every mode-selecting
branch) and `pv_now`/`house_load_now` (raw, used only for Maintain
Zone's 5000W-vs-dynamic charge ceiling choice, where both outcomes stay
in maximize_self_consumption so there's no mode-flapping risk). Full
detail: architecture.md - Layer 4A - PV/Load Sampling Smoothing.

### Update 2026-08-29 - Deployed and confirmed live

Copied to the live config and reloaded. First reload attempt did NOT pick
up the two new `platform: statistics` sensors - a full Home Assistant
restart (not just a YAML/config reload) was needed, since `platform:
statistics` is a legacy YAML sensor platform outside HA's hot-reloadable
domains. A second restart-and-wait cycle brought both sensors up
successfully.

Verified via a live trace (2026-08-29T01:00:00 local,
`automation.emhass_battery_forecast_control`): `pv=0, house_load=153`
(smoothed, live, driving mode selection) and `pv_now=0, house_load_now=137`
(raw, live, driving the Maintain Zone charge-ceiling choice) - landed in
Maintain Zone as expected for nighttime with no PV.

### Update 2026-08-29 (morning) - Daylight review

Reviewed all 5 retained traces of `automation.emhass_battery_forecast_control`
from 2026-08-29 09:45-10:45 local plus the full day's mode history since
05:00 local. Mode stayed at `maximize_self_consumption` for the entire
morning with zero transitions - no oscillation, no repeat of the
charge-then-export pattern this fix targeted. What the split DID
demonstrably do, twice (07:45 and 08:45 local): the 4-min smoothed
`pv`/`house_load` would have opened the 5000W charge ceiling on stale
averaged headroom, but the raw `pv_now`/`house_load_now` at the exact
tick showed load currently ahead of PV, so the system correctly stayed on
the normal dynamic ceiling instead - direct evidence the mode/limit split
is doing real work.

Unrelated finding surfaced while reviewing: twice this morning (08:15 and
08:30 local, `above` true, `grid_fc` export-favourable, `batt_fc` exactly
0), the automation matched no branch at all. Root cause: DISCHARGE MAX
EXPORT required `batt_fc > 0` and DISCHARGE MIN IMPORT/MAX SELF-
CONSUMPTION required `grid_fc >= 0`; `batt_fc` exactly flat fell through
both. Logged as a gap, fixed 2026-09-03 - see Branch-Coverage Gaps below.

Status: deployed and verified live for the mode-selecting branches
exercised so far (Maintain Zone ceiling choice, steady-state morning).
Below-SOC and Clipped-Solar branch behaviour under the smoothed variables
still awaits a real exercising event.

---

## Midday Charge/Discharge Oscillation - Branch-Coverage Gaps

### Update 2026-09-03 - Branch-coverage gap fixed: DISCHARGE MAX EXPORT now includes batt_fc == 0

Fixed the gap found in the 2026-08-29 (morning) Daylight review above. In
`battery_forecast_control.yaml`, the top-level DISCHARGE MAX EXPORT
condition changed from `(maintain or above) and (soc > soc_min) and
(grid_fc < 0) and (batt_fc > 0)` to `... and (batt_fc >= 0)`. Previously,
`above` true with `grid_fc < 0` (export favourable) and `batt_fc` exactly
`0` matched neither DISCHARGE MAX EXPORT (`batt_fc > 0`) nor DISCHARGE MIN
IMPORT/MAX SELF-CONSUMPTION (`grid_fc >= 0`), nor MAINTAIN ZONE (`above`
isn't `maintain`) - the whole top-level `choose:` fell through with no
`default:`, leaving the previous requested mode/limits unchanged for that
cycle. Widening the boundary to `>=` closes exactly that gap without
overlapping the sibling branch (still mutually exclusive on `grid_fc`
sign), and the newly-included `batt_fc == 0` case still passes through
the existing Grid Charge Export Cooldown check before actually forcing
`discharge_to_maximize_export`.

### Update 2026-09-03 - Branch-coverage gap fixed, part 2: BELOW-SOC no-PV / batt_fc==0 catch-all

Also fixed, per follow-up request: the second gap spotted while
implementing part 1 above. In `battery_forecast_control.yaml`'s
BELOW-SOC nested `choose:`, the last branch's condition changed from
`(pv == 0) and (batt_fc == 0) and (grid_fc > 0)` to `(pv == 0) and
(batt_fc == 0)` - dropping the `grid_fc > 0` requirement. This branch is
only reached after the price-based fallback branch immediately above it
has already failed to match, i.e. price is already known to be elevated -
so requiring `grid_fc > 0` on top of that was an unnecessary extra
condition that left `(pv==0, batt_fc==0, grid_fc<=0, price high)` matching
no branch anywhere in the automation. Same `solar_power_only` outcome as
before for the previously-covered `grid_fc > 0` case; now also covers
`grid_fc <= 0` the same way.

### Update 2026-09-10 - Partial live confirmation

The deeper charge/discharge review spot-checked both branches against
live traces (only ~1 hour of trace history was retained for this
automation - 5 runs, 2026-09-10 17:30-18:30 local, not a multi-day
check). DISCHARGE MAX EXPORT (`batt_fc >= 0`) **confirmed firing**: a
trace with `batt_fc=1546.04`, `grid_fc=-1242.04`, `above=true`,
`soc(51.1%) > soc_min` correctly took this branch (though that instance
had `batt_fc` well above zero, not testing the exact boundary). The
BELOW-SOC catch-all (`pv==0 and batt_fc==0`) could **not** be confirmed
firing in the available window - none of the 5 traces had `below` true.
A third, related branch-coverage gap was found live during this same
check - see Known Issues - Not Yet Fixed above.

Status: DISCHARGE MAX EXPORT deployed and confirmed firing live.
BELOW-SOC catch-all deployed but not yet confirmed firing live (trace
retention too short to catch a qualifying event so far).

---

## Write Pressure - Mode Command Guard

### Update 2026-08-26 - Mode Command Guard implemented

`automation.apply_effective_battery_control_to_solaredge_inverter` now skips
the mode-select command and the 15-minute command timeout reset whenever the
effective mode is `maximize_self_consumption` (the most-used fallback mode),
`select.solaredge_i1_storage_command_mode` already confirms the inverter is
in that mode, AND `select.solaredge_i1_storage_default_mode` confirms the
inverter's own configured fallback is also Maximize Self Consumption (so
letting the command timeout lapse is verified safe, not assumed). Only the
charge/discharge limit writes still go out in that case, instead of the
full 4-write sequence on every trigger. `sensor.battery_mode_command_guard`
exposes whether the guard is currently active for observability.

### Update 2026-08-27 - Live log check after guard deployment

Checked HA logs/traces the same day the guard went live. The guard itself
is confirmed working (see test_plan.md Test 7.6). Modbus errors are still
present but appear pre-existing rather than guard-caused - counts are
modest (single digits to low teens over many hours), consistent with the
pre-existing intermittent connectivity picture rather than a new problem.
No direct evidence yet either way that the guard has measurably reduced
or worsened error frequency.

Status: deployed and confirmed working (see test_plan.md Test 7.6).
Before/after impact on overall Modbus error rate not yet quantified.