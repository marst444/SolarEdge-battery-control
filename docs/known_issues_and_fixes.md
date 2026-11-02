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

Five issues are currently open. The newest, found 2026-11-02 in response
to the user's report that the autumn clock change caused EMHASS to fail
for a full day (and that the same thing happened at the spring
transition, where a fix was made that evidently didn't carry over) -
Winter-Time (DST Fall-Back) Transition Caused a Full-Day EMHASS Failure -
was initially mis-diagnosed against a legacy mechanism that turned out to
be dead code (confirmed with the user: superseded by a refactor, not
live), then re-diagnosed against the actually-running scripts: a
confirmed, live bug in how `emhass_sensors.yaml`/`emhass_scripts.yaml`
round-trip forecast timestamps through bare local-time strings, which is
provably lossy on the fall-back day specifically. See Open Issues below.
The next-newest, found 2026-11-02 while reviewing whether
Maximize Self Consumption is the right nighttime fallback mode - PV
Sensor Trickle Readings Bypass the "No PV" Branch Logic - is a
decision-engine robustness gap, not yet confirmed to have caused a real
misfire: several `battery_forecast_control.yaml` branches test `pv == 0`
exactly to mean "no real PV," which a small sensor offset/trickle
reading can defeat. A noise-floor fix is implemented in project docs;
see Open Issues below. The next, found 2026-11-01 from the user's report
"the SOC now recurrently goes over 90%" - High SOC Hold's Charge Limit =
0 Does Not Stop Charging While Maximize Self Consumption Stays Active -
is a safety-relevant regression against this project's own documented
design assumption: the charge-limit register is confirmed holding at 0
throughout, yet the battery keeps charging to 100% anyway. See Open
Issues below. The other two, both found 2026-10-11 while verifying a
real SOC<10% critical-recovery event against the user's request to check
it against the stats, are Critical-Recovery Charge Limit Write Silently
Dropped (the physical charge-limit register never actually reached the
critical-recovery value, stuck at a stale 5000W for ~28 minutes) and
SOC<10% Critical Recovery Triggered By a Physically-Implausible
Single-Sample SOC Reading (strong evidence the underlying SOC<10%
condition itself may never have been real). A sixth-order item found
2026-10-19 during a routine behaviour review - Modbus Queue Pending-Count
Counter Leaks On Lock-Wait Timeout, a telemetry-only bug where
`binary_sensor.solaredge_modbus_queue_backlog` had been reporting a false
"backlog" for extended periods while real command throughput stayed
normal - was fixed the same day it was found and **confirmed deployed
and working live 2026-10-20** against a real timeout event; see Fixed
Issues below.

EMHASS Rejects Optimisation
Requests When `sensor.max_active_power_1` Goes Unavailable, diagnosed and
fixed live 2026-10-11 by removing that cloud-sourced entity from EMHASS's
critical path entirely - see Fixed Issues below - is now **confirmed
against a live post-fix EMHASS run** (16:15 local MPC cycle: payload
showed `inverter_ac_output_max`/`inverter_ac_input_max: 10600`, no HTTP
400, `sensor.emhass_health` back to OK). The 45-minute Maximize-Self-
Consumption fallback remains as a second line of defense regardless. The Negative Price
Curtailment - Site Limit Restore Gap found 2026-10-06 is a confirmed
real asymmetry (the old design's deactivation path never restored
`number.solaredge_i1_site_limit`) that was empirically verified the same
day to have no live impact, and was then closed the same day by a
structural redesign rather than a patch; that redesign's remaining
live-verification gap (the switch's defensive re-assert, and the number's
curtail/restore cycling, actually firing live) was **closed 2026-10-19**
against a real overnight negative-price episode - see below. The Modbus
Queue Pending-Count Counter Leak found 2026-10-19 was fixed the same day
via a compensating decrement in `script.modbus_queue`'s lock-wait-timeout
branch, deployed live 2026-10-20, and **confirmed working against a real
timeout event that same night** - see Modbus Queue Pending-Count
Counter Leaks On Lock-Wait Timeout below. The branch-coverage gap found 2026-09-10 was fixed 2026-09-15
and confirmed firing live - see Branch-Coverage Gaps below. Its related,
unconfirmed corner - `grid_fc < 0` together with `batt_fc <
0`, a contradictory-forecast edge case - is still uncovered by any
branch, not observed live, and not proposed for a blind fix. The Layer
4A SOC-unavailable guard found 2026-09-19 was fixed the same day and
**confirmed live 2026-09-24** against a real ~2h13m outage - see Fixed
Issues below. The modbus_busy stale-lock gap found 2026-08-28 was fixed
2026-09-20 and is now **behaviorally confirmed live** as of 2026-09-24
- see Modbus Queue Lock Release Not Exception-Safe below. The
EMHASS-outage fallback gap found 2026-09-22 was fixed the same day,
deployed and structurally confirmed 2026-09-23 - see EMHASS Outage
Fallback Guard below; its behavioral verification (an actual skipped
run, or the addon auto-restart firing) still awaits a real future
EMHASS stall. The EV Charging Sensor Swap incomplete-trigger follow-up
found 2026-09-24 was fixed 2026-09-25 - see EV Charging Sensor Swap
below; not yet confirmed against the live file or a real EV session. The
Enhanced Diagnostics write-outcome telemetry added 2026-10-06 hit a real
~50% false-positive rate live the next day; a fix moving the
verification wait into a separate async script was designed 2026-10-07,
an initial redeploy left it unwired, and a corrected redeploy the same
day was then **confirmed wired in and working live** via a real trace -
see Write-Outcome Telemetry Is Readback-Based below, all three
2026-10-07 updates, for the full picture. A related documentation-
only issue - three entity_ids documented differently than what Home
Assistant actually assigned them - was found and fixed the same day,
see Entity ID Documentation Mismatches below.)

---

# Open Issues

## Winter-Time (DST Fall-Back) Transition Caused a Full-Day EMHASS Failure

Found 2026-11-02, from the user's report: "the change from summer to
wintertime caused the EMHASS to fail a whole day. It was the same in the
spring, we analyzed and made a fix, but obviously didn't work." Sweden
(`Europe/Stockholm`, confirmed live) observes the EU rule - clocks went
back 2026-10-25 03:00 CEST -> 02:00 CET.

**First root-cause pass was wrong - diagnosed dead code.** The initial
write-up here centered on `script.emhass_update_sliding_window` and
`input_datetime.day_ahead_forecast_end`. Checked live: that entity's
value is frozen at `2026-07-10 21:00:00` and hasn't moved since;
`automation.emhass_day_ahead_optimization`, `automation.emhass_update_
sliding_window`, `automation.emhass_mpc`, and `script.run_mpc_
optimization` are all in state `unavailable` right now, with no config
resolvable for the first two via `ha_config_get_automation`. Confirmed
with the user: these were intentionally replaced by `generate_emhass_
energy_plan_mpc_pv`/`dh_pv`, which are the actually-running automations
(confirmed firing every 15 minutes / once daily in the live log). The
entire sliding-window/`day_ahead_forecast_end` mechanism is orphaned
leftover from before that refactor, not live code - the search that
found no writer for `day_ahead_forecast_end` anywhere in this project
wasn't a documentation gap, it's dead because nothing needs to write it
any more.

**Confirmed, live bug in the scripts that actually run: a lossy
local-time string round-trip.** `emhass_sensors.yaml`'s `forecast_merged`
attributes (on `nordpool_se3_total_costs_import`, `..._total_revenue_
export`, `epex_total_costs_import_forecast`, `epex_total_revenue_export_
forecast`, the merged costs/revenue sensors, the Solcast interval
sensors, and `p_load_dayahead_forecast`) all build their `ts` field the
same way:

```text
ts = as_timestamp(start) | timestamp_custom('%Y-%m-%d %H:%M')   # bare, no UTC offset
```

`emhass_scripts.yaml`'s `generate_emhass_energy_plan_mpc_pv` and
`generate_emhass_energy_plan_dh_pv` - both confirmed live and running -
then re-parse that bare string back into a dict key for every one of
`load_cost_forecast`, `prod_price_forecast`, and (via `period_start`
directly) `pv_power_forecast`:

```text
key = (ts.split(' - ')[0] | as_datetime | as_local).isoformat()
```

`as_datetime` on a bare "YYYY-MM-DD HH:MM" string with no offset produces
a *naive* datetime; `as_local()` then has to guess which UTC offset it
meant. On 2026-10-25, the local hour 02:00-02:59 happened twice in
reality (once at CEST/+02:00, once at CET/+01:00) - two genuinely
different real instants, each a real Nordpool/EPEX quarter-hour with its
own price. Both format to the identical bare string "2026-10-25 02:1X".
Python's `as_local()` resolves an ambiguous naive time using `fold=0` by
default, which always picks the *first* (CEST) interpretation - so both
real instants' bare strings decode back to the same key. When the
script's `combine()` builds the output dict, the second occurrence
silently overwrites the first under an identical key: one of the two
real quarter-hours' price (or PV) data for that hour is discarded
outright, every single day this code runs on 2026-10-25's data, for as
long as that date stays inside the `num_prediction_days: 2` window feeding
both the MPC (every 15 min) and day-ahead (daily) scripts - i.e. this
wasn't a single-moment failure, it was wrong for roughly two days'
worth of runs bracketing the transition. The exact downstream failure
mode inside the EMHASS add-on's solver (length mismatch vs. its own
internally-generated, correctly-DST-aware time index; a `NaN`/missing
value the solver rejects; something else) could not be confirmed from
this project's docs alone - the add-on's Python source isn't in this
project - but a silently wrong or missing price/PV value for part of a
day feeding a profit-optimizing MPC solver is a plausible, concrete path
to the solver failing or returning a degenerate plan for that day.

**Historical confirmation was attempted and could not be completed.**
Home Assistant's automation-trace store only retains 5 runs per
automation (all from 2026-11-02 when checked), and `ha_get_history`
against `sensor.mpc_pv_optim_status`/`sensor.dh_pv_optim_status` for
2026-10-24 through 10-26 returned zero rows - outside this recorder's
retained window. The live-code bug above is confirmed by reading the
actual running templates, not by observing the failure directly on the
day it happened.

**Why a spring fix wouldn't carry over to this:** spring-forward
(2026-03-29 this year) *removes* an hour from the local day - Nordpool/
EPEX's raw data for that day genuinely has fewer quarter-hours, so
`today_raw`/`today_ts` in `emhass_sensors.yaml` are legitimately short
that day. A fix aimed at that shape of problem (a short list, a missing
local time, a resulting out-of-range index) would not address fall-back's
shape of problem at all: fall-back doesn't produce a short list, it
produces a full-length list with a silently *duplicated* key that
overwrites real data - a bare string collision, not a missing entry.
Same underlying design choice (format forecast timestamps as bare local
strings, no UTC offset) breaks in two different, unrelated-looking ways
depending on which direction the clock moves - consistent with the
user's report that the spring fix "obviously didn't work" here: it's not
that the fix regressed, it's that fall-back is a different failure
shape the spring fix never touched.

**Fix candidates, not yet implemented:** the root problem is that
`timestamp_custom('%Y-%m-%d %H:%M')` throws away the UTC offset that
`as_timestamp()` already had. Two options: (a) in `emhass_sensors.yaml`,
format `ts` to include the offset (e.g. an ISO 8601 string with
`%z`/`+00:00`, or keep it as a UTC epoch/ISO string instead of a local
string) so `emhass_scripts.yaml` can parse an already-unambiguous,
offset-aware value and never needs `as_local()`'s fold guess at all; or
(b) narrower, leave the producer as-is and have the consumer deduplicate
by real timestamp before building the dict (e.g. carry the epoch
alongside the display string and key on that instead). Option (a) is the
more complete fix - it removes the ambiguity at the source instead of
working around it on the read side - but touches every `forecast_merged`
producer in `emhass_sensors.yaml` plus both consumer scripts in
`emhass_scripts.yaml`, so it needs a deliberate pass across all of them
together, not a one-line patch. Not done yet this session.

Status: **root cause confirmed in currently-live code (the bare
local-time round-trip in `forecast_merged`/`load_cost_forecast`/
`prod_price_forecast`/`pv_power_forecast`); not yet fixed.** The original
theory centered on `emhass_update_sliding_window` is retired - confirmed
dead code, superseded by a refactor the user confirmed was intentional.
Direct historical confirmation of the exact 2026-10-25 EMHASS failure
(e.g. a non-"Optimal" solver status recorded that day) was not possible
from this recorder's ~10-day retention; the fix above should be treated
as confirmed-by-mechanism, not confirmed-by-observed-incident.

---

## PV Sensor Trickle Readings Bypass the "No PV" Branch Logic

Found 2026-11-02, prompted by the user's question of whether Maximize
Self Consumption is really the right fallback mode at night, given the
PV sensor sometimes shows a small trickle reading instead of a clean 0
when it's dark. Investigating that question (by reading the live
`battery_forecast_control.yaml` directly rather than relying on
README.md's description, which turned out to be stale - see the note
under the Fix below) surfaced a more concrete issue than the one
originally asked about.

Several of the "Below SOC Target" branches test `pv == 0` *exactly* to
mean "no real PV, it's night" - e.g. the branches that route to
`solar_power_only` or the price-based grid-charge fallback. The 4-minute
trailing-mean smoothing already in place (see Midday Charge/Discharge
Oscillation - PV/Load Sampling Smoothing, and architecture.md - Layer 4A)
damps momentary spikes/dips, but does nothing for a steady sensor offset
or trickle reading - averaging a small constant noise floor still
produces a small constant nonzero result, not 0. A trickle reading of
even a few watts breaks the exact-equality test and falls through into
the `pv > 0` branches instead, which assume real daytime partial PV and
include real grid-charging logic (`charge_from_solar_and_grid`). This
means a sensor artifact at night could in principle trigger an actual
unwanted grid-charge decision, not merely a cosmetic mode mismatch - a
more significant risk than the Maximize-Self-Consumption-specific
concern that prompted the investigation, though not yet confirmed to
have actually caused a live misfire (no specific incident traced to this
mechanism so far - found by code review, not by observed symptom).

**Fix implemented 2026-11-02**: a new helper, `input_number.
pv_noise_floor_w` (`batterycontrol_helpers.yaml`, default 20W), is now
applied to both `pv` and `pv_now` in `battery_forecast_control.yaml` -
any reading below the floor is treated as exactly 0 before any branch
condition sees it, restoring the `pv == 0` comparisons' intended meaning
regardless of smoothing. YAML-validated (`yaml.safe_load`) before push.

**Separately, while investigating this: Maximize Self Consumption is
*not* actually used for the "no PV, price too high, hold" case** -
README.md's Decision Logic section describes that case as landing on
Maximize Self Consumption, but the live YAML's actual final branch for
`pv == 0 and batt_fc == 0` sets `solar_power_only` instead (charge and
discharge both pinned to 0), which is the *stronger* of the two
guarantees - the documentation was stale, not the code. None of the
"Below SOC Target" no-PV branches use Maximize Self Consumption at all;
the one place that still pairs Maximize Self Consumption with a
hardcoded `charge_limit: 0` (the same pattern that broke High SOC Hold -
see below) is the Discharge Max Export section's cooldown fallback,
which is not night-specific and was left as-is per the user's decision,
pending the High SOC Hold mode-switch fix's own live verification first
rather than spreading the same unverified substitution further before
that's confirmed. README.md's stale line was corrected to match the live
code.

Status: **implemented in project docs, not yet deployed to the live HA
instance or verified against a real low-PV/trickle-reading night.**

---

## High SOC Hold - Charge Limit = 0 Does Not Stop Charging While Maximize Self Consumption Stays Active

Found 2026-11-01, in response to the user's report "the SOC now
recurrently goes over 90%." This is a direct contradiction of this
project's own documented design assumption (README.md - Maximize Self
Consumption: "Storage Charge Limit and Storage Discharge Limit are used
as real-time power caps while Maximize Self Consumption remains active...
Charge limit = 0 → battery will only discharge, not charge") and of
`architecture.md`'s High SOC Hold description ("SOC >= Maximum SOC → High
SOC Hold enabled → Battery charging blocked").

**History first - confirmed recurring, not a one-off:** `input_boolean.
battery_high_soc_hold` / `sensor.solaredge_b1_state_of_energy` history for
2026-10-22 through 2026-11-01 (10 days) shows 11 clean hold-engage events,
one per day, each a regular morning engage (~09:12-11:40 local) and early-
morning disengage (~02:09-05:24 local) - exactly matching "recurrently."
In every single one of the 11 cycles, hold engaged at exactly SOC = 90.00%
(the configured max, not ~89% as the `soc_max - 1` entry margin coded in
`battery_high_soc_hold` would suggest - see the margin note below). In 10
of the 11 cycles, SOC continued climbing all the way to 100.00% after hold
engaged (not a few points of overshoot - the full remaining range to the
sensor's ceiling) before eventually coming back down; the most recent,
still-in-progress cycle (2026-11-01) had reached 96.67% by the time this
was investigated. Time from hold-engage to peak ranged 64.6-395.7 minutes
across the 10 completed cycles.

**Root cause, pinned to the register level, not inferred:** pulled
`number.solaredge_i1_storage_charge_limit` history directly for two of
these windows.

- 2026-10-23 (the longest cycle, 10:10-16:45 local, 6h45m): the entity has
  exactly **one state for the entire window: `0`.** It was never
  rewritten, never reverted, never dropped back to a stale nonzero value
  - it simply held at 0, confirmed, the whole time. SOC still climbed from
  87.78% at 10:00 to 100.00% at 16:45:55, oscillating down a point here
  and there (consistent with the small dynamic discharge limit kicking in
  periodically) but net climbing throughout.
- 2026-11-01 (today, still in progress at the time of writing): the
  register dropped to exactly `0` at 10:54:48.438931 local, 42 seconds
  after `input_number.effective_charge_limit` itself hit 0 at 10:54:06.272
  (the gap is the normal `script.modbus_queue` write latency) - the same
  instant hold engaged (SOC crossed 90.00% at 10:54:06.187). It has not
  moved since. SOC nonetheless climbed from 90.0% to 96.67% over the
  following 74+ minutes. `input_select.effective_storage_mode` and the
  live `select.solaredge_i1_storage_command_mode` both stayed on
  `maximize_self_consumption`/"Maximize Self Consumption" throughout - no
  mode flapping, no fallback-timeout lapse - and `input_number.
  effective_discharge_limit` only held a small dynamic value (16-204W),
  nowhere near large enough to explain the net climb as discharge simply
  losing a tug-of-war.

This rules out the two most obvious alternative explanations: it is not
the already-documented silent-write-drop pattern (see Critical-Recovery
Charge Limit Write Silently Dropped below) - the register genuinely holds
at the commanded 0, no reversion - and it is not a mode-guard/fallback-
timeout issue - the mode never left Maximize Self Consumption. The
inverter's own AC-side power (`sensor.solaredge_i1_ac_power`) stayed
near-idle (roughly -10W to 0W, one brief 2526W spike) throughout the
2026-10-23 window while the battery kept charging - consistent with the
battery being charged from PV on the DC side, a path that this installation's
Remote Control charge-limit register may simply not gate once Maximize
Self Consumption's own internal self-consumption logic is active, contrary
to what README.md documents. Not yet confirmed against the SolarEdge
Modbus Multi integration's own source or documentation - this is the
leading hypothesis from the live evidence, not a proven mechanism.

**The `soc_max - 1` entry margin is a red herring for this issue, not
the cause and not (by itself) a fix.** `safety_limits_and_override.yaml`'s
`battery_high_soc_hold` automation already implements `soc >= (soc_max -
1)` for entry and `soc <= (soc_max - 3)` for exit - this already matches
what `test_plan.md` Test 1.2's long-standing note describes as merely
"proposed... not yet implemented," so that note is now confirmed stale
(see test_plan.md update below) independent of this issue. But the margin
doesn't matter here: every observed entry happened at exactly 90.00% (the
SOC sensor's own quantization appears to skip straight from below 89% to
90% in one sample on this battery, consistent with the ~1.11-point steps
seen elsewhere in this project), and even if entry happened at a true 89%,
the same mechanism would still let charging continue past it - the margin
only buys an earlier trigger, not an actual charge-stop.

Not yet proposed as a fix here: forcing `effective_storage_mode` away
from `maximize_self_consumption` during High SOC Hold (e.g. to
`solar_power_only`, which README.md documents as fully idling the
battery with no charge or discharge path) rather than relying on the
charge-limit register alone while remaining in MSC. This would be a
real behavior change to a safety-critical path and needs a design
decision (and likely a live test with the user's explicit approval,
same as the Test 7.6(c) live intervention), not a blind fix - flagged in
roadmap.md as the leading candidate.

Status: **open, not yet fixed.** `test_plan.md` Test 1.2 downgraded from
PASS to FAIL - the test's own stated purpose ("verify charging is blocked
when SOC exceeds configured maximum SOC") is now confirmed false under
live PV conditions, in 10 of 11 recently observed cycles.

### Update 2026-11-01 (same day) - second, distinct mechanism caught live: a routine Modbus reconnect resets both limit registers AND the command timeout to non-project values, and nothing re-asserts them promptly

Found live, mid-investigation, after the user reported "it is also
currently flip-flopping between charging and discharging" while Hold was
active. This is a second, independently-confirmed failure mode layered on
top of the one above - not a replacement for it, since the original
blip-free 2026-10-23 case (charge limit held at a confirmed, unchanging 0
for 6h45m while SOC still climbed to 100%) is not explained by this
mechanism and remains unexplained on its own.

Evidence, all local time 2026-11-01, pulled directly from entity history
across today's ~6-hour Hold window (12:00-18:04): `number.solaredge_i1_
storage_charge_limit` held at a confirmed, unchanging `0` continuously
from 12:00 until 18:03:57, when it went `unavailable` for 23 seconds (an
ordinary brief Modbus reconnect blip - the same kind this project has
documented happening multiple times a day in normal operation) and came
back at 18:04:20 reading **5000** - not 0. In the same instant, `number.
solaredge_i1_storage_discharge_limit` also came back at 5000 (its
pre-blip value, from a fresh dynamic-driven write at 17:45:55, had been
307). `number.solaredge_i1_storage_command_timeout` - steady at the
project's own configured `900` for the same 6 hours - came back reading
**3600**, a value nothing in this project's YAML requests anywhere
(confirmed: no script, automation, or hardcoded value anywhere in this
project writes 3600 to that entity). `input_number.effective_charge_
limit`/`effective_discharge_limit` (0.0 and 307.0 respectively) did not
change at all across this entire window, before or after the blip -
ruling out this project's own automations as the source of the 5000/5000/
3600 values. This points at the inverter or the SolarEdge Modbus Multi
integration itself reinitializing storage-control registers to its own
internal defaults on reconnect, independent of anything this project
commands.

`sensor.battery_mode_command_guard` flipped `guarded -> active` at
18:03:57.62 (the instant of the blip - `select.solaredge_i1_storage_
command_mode` presumably read something other than "Maximize Self
Consumption" for that moment) and back to `guarded` 23ms after recovery,
once the live command mode confirmed MSC again.

**Root cause, tying this to the Mode Command Guard's own documented
design (architecture.md - Layer 4B - Mode Command Guard):** the guard
intentionally skips `script.set_value_storage_command_timeout15m` (the
900s command-timeout refresh) whenever effective mode, live command mode,
and configured default mode all agree on Maximize Self Consumption -
exactly the condition that holds continuously throughout a sustained High
SOC Hold, since Hold deliberately keeps `effective_storage_mode` on
`maximize_self_consumption` rather than switching away. Confirmed
directly: `number.solaredge_i1_storage_command_timeout` shows zero writes
of any kind for the full 6-hour Hold window today, consistent with that
refresh step never running once. The guard's own design rationale was
that skipping the *mode* refresh is safe because MSC is also the
inverter's configured safe fallback - that rationale was never extended
to cover what happens to the *limit registers* once the underlying
Remote Control session itself is disturbed (by a routine reconnect, or
possibly just by its own internal timeout elapsing with nothing to
measure that against from the HA side).

Separately, nothing in this project proactively corrects a reset like
this afterward either: `apply_effective_battery_control_to_solaredge_
inverter` only re-fires on a state *change* of `effective_storage_mode`/
`effective_charge_limit`/`effective_discharge_limit` - a reconnect-driven
register reset touches none of those, so the registers can sit at
5000/5000 (effectively uncapped in both directions) until the next time
EMHASS's dynamic recalculation happens to tick `effective_discharge_
limit` over naturally. In between, with no charge or discharge cap in
effect, the inverter's native self-consumption logic is free to swing at
near-full rate as PV and load fluctuate - the most direct explanation
available so far for the user's live "flip-flopping between charging and
discharging" report.

**The user's own follow-up observation, confirmed correct and worth
stating plainly:** because `number.solaredge_i1_storage_command_timeout`
reads 3600 now (not the project's own 900), and nothing resends that
register except the same guarded "then" branch that only runs on a real
mode transition away from MSC, this corrupted value is **not
self-healing** - it will sit at 3600 indefinitely through any number of
future limit-only writes (which don't touch the timeout register at all)
until an actual mode change occurs. Manually re-pushing the correct
charge/discharge limit values (e.g. via `script.set_effective_storage_
charge_limit`/`_discharge_limit`) re-asserts the numbers but does nothing
to fix the underlying timeout value or confirm what Remote Control
session state the inverter now considers itself in - so the same
5000/5000 reset could recur at any time, not only after a blip, until
this is understood and fixed properly.

Not yet implemented, needs a design decision rather than a blind fix -
see the Update directly below, which narrows and partly revises these
candidates based on a direct test.

### Update 2026-11-01 (same day, continued) - user-proposed natural experiment disconfirms the "stale timeout" mechanism; narrows this to a reconnect-specific reset

The user proposed a direct test of the hypothesis above: find a period
where `input_number.effective_discharge_limit` was deliberately forced
low (the EV Charging Discharge Control override, which pins the
"requested" discharge limit to 100W whenever `binary_sensor.ev_charging_
active` is "on") and check whether real discharge happened anyway during
a stretch where the Mode Command Guard had been "guarded" (skipping the
timeout refresh) for a long time - directly testing whether an unrefreshed
command-timeout register is what lets the limit lapse, independent of any
reconnect blip.

Found a clean case: a 22-minute EV-charging session on 2026-10-31,
20:46:15-21:08:16 local, night-time (zero PV, so any discharge need is
real, not masked by PV surplus). Through this entire window:
`sensor.battery_mode_command_guard` read a flat, unchanging "guarded"
(mode_guard_active true throughout - no mode transition, no reconnect).
`number.solaredge_i1_storage_command_timeout` read a flat, unchanging
"3600" throughout (already the non-project, presumably-stale value *before*
this window even started, and never refreshed back to 900 during it or
since - confirming the earlier suspicion that this value, once knocked
off 900, really does sit there indefinitely with nothing correcting it).
Crucially, `number.solaredge_i1_storage_discharge_limit` still tracked
`effective_discharge_limit` correctly through three separate changes in
this one window (100 -> 178 -> 157 W, each landing within the normal
15-55s write lag) - and `sensor.solaredge_i1_ac_power` (the inverter's own
AC-side power sensor) stayed near-idle throughout (a few W either side of
zero, one single noisy +2225.7W sample), showing no sign of sustained
discharge above the commanded cap. No `unavailable` transition occurred
anywhere in this window on any of the entities checked.

**This disconfirms the mechanism proposed in the Update above as the
actual gate.** If an unrefreshed command-timeout (or the guard simply
having been "guarded" for a long stretch) were what let the limit
registers drift from `effective_*`, this window - guard engaged the
whole time, timeout already sitting at the "wrong" 3600 throughout -
should have shown the same kind of drift. It didn't: the discharge limit
held and updated correctly on every change, with no reconnect in sight.
`number.solaredge_i1_storage_command_timeout`'s value does not appear to
gate whether charge/discharge limit writes are honored day to day - it
looks like a separate register that itself also happens to get knocked
off its intended value by the same event, not the cause of the drift.

**Revised, narrower conclusion:** the actual trigger for registers
resetting to non-project defaults (5000W/5000W/3600s, as caught live
earlier today) appears to be the Modbus *reconnect event itself*
(`unavailable -> available`), not elapsed time since the last refresh and
not the guard's "guarded" state by itself. Every case examined so far is
consistent with this narrower rule: 2026-10-31's 22-minute guarded,
stale-timeout, blip-free window held the limit correctly throughout;
today's 18:03:57 reset happened in the same instant as (and only as) an
`unavailable` transition.

This also suggests a cleaner explanation for how this coexists with the
original, still-separately-open 2026-10-23 finding (charge limit
confirmed held at a steady, blip-free 0 for 6h45m, yet SOC still climbed
to 100%): discharge limit, confirmed again here, governs AC-side power
the inverter delivers outward and is reliably enforced absent a
reconnect. Charge limit, when the charging source is PV surplus routed
internally to the battery under Maximize Self Consumption rather than an
AC-sourced charge command, may not be gated by this register at all, on
this hardware - independent of reconnects entirely. That would mean two
genuinely separate problems share the same symptom: discharge-side limits
break specifically on a reconnect event; charge-side limits (at least
under MSC with PV present) may not be enforced by this register in the
first place. Neither half of this unifying read is proven yet - it's the
best fit to the evidence gathered so far, not a confirmed mechanism.

Fix candidates revised accordingly: (a) resending `script.set_value_
storage_command_timeout15m` more often is now a lower-confidence fix,
since the timeout value itself doesn't appear to be what's gating
anything; (b) detecting the specific `unavailable -> available` transition
on the SolarEdge mode/limit entities and immediately re-pushing
`effective_charge_limit`/`_discharge_limit` (and the mode) right at that
moment, rather than waiting for the next unrelated `effective_*` change,
targets the mechanism actually observed; (c) for the charge-side-under-MSC
question specifically, nothing short of checking the SolarEdge Modbus
Multi integration's own source, or deliberately forcing a non-MSC mode
during a controlled PV-surplus test, would confirm or rule it out further.
None of this is implemented - still needs a design decision.

Status: **open, not yet fixed.** Two distinct mechanisms are now
suspected behind the same user-reported symptom ("SOC recurrently over
90%", now also "flip-flopping between charging and discharging"): a
reconnect-triggered reset of the discharge (and possibly charge) limit
registers to non-project defaults, now narrowed away from the
"unrefreshed timeout" theory to the reconnect event itself; and the
original 2026-10-23 finding (charge limit confirmed held at a steady,
unreset, blip-free 0 for 6h45m, yet SOC still climbed to 100%), which
remains unexplained by either mechanism above and stays open on its own.

### Update 2026-11-02 - full historical survey finds a sharp 2026-10-14 inflection point; what might have caused it checked and ruled out, cause itself still open

Per the user's request to look back further than the ~10-day live
history window allows, pulled permanent daily statistics (`source:
statistics`, `period: day`, `max`/`min`/`mean`) for `sensor.
solaredge_b1_state_of_energy` across the project's full history to date,
2026-07-01 through 2026-11-01 (121 days).

79 of 121 days reached a daily max SOC >= 90%, but they split into three
clearly distinct phases, not a steady trickle:

```text
2026-07-01 to 2026-08-09   frequent 90-98% days, zero 100% days
2026-08-10, 2026-08-11     two isolated 100.00% days
2026-08-12 to 2026-10-13   ceiling settles back to ~90.00% almost every
                           time (one 94.44% outlier, 2026-09-22) -
                           zero 100% days for this entire 9-week stretch
2026-10-14 onward          dense, essentially unbroken run of 100.00%
                           days - every day from 10-14 through 11-01
                           except 10-21 (98.89%)
```

So the behaviour now confirmed broken (2026-10-22 through 2026-11-01,
the 11-cycle history documented above) did not start on 2026-10-22 - it
started eight days earlier, 2026-10-14, and the 10-22 to 11-01 window is
simply the most recent, continuous stretch of it. The two 2026-08-10/11
spikes show the same end-state (100%) occurred once before, briefly,
well before this investigation began, then the system ran correctly
(ceiling held at ~90%) for nine weeks before failing again on 10-14 - a
real regression at that date, not a condition present since Hold was
first implemented.

**What might explain 2026-10-14, checked and ruled out:**

- *SolarEdge Modbus Multi integration update* - the user's specific
  question. HACS/the `update.solaredge_modbus_multi_update` entity's
  logbook (checked back 900 hours, to 2026-09-25) shows exactly **one**
  install event in that whole span: 2026-10-31 20:41:09-20:44:28 local,
  user-initiated (`context_service: install`, `auto_update: false`),
  v4.0.6 -> v4.0.11 (current, matches latest, no pending update). It sat
  on v4.0.6 the entire time from 2026-10-11 (confirmed healthy that day -
  see EMHASS Rejects Optimisation Requests above) through 2026-10-30.
  **This update happened 17 days after the 2026-10-14 onset - it cannot
  be the cause.** (It is, however, the version every piece of register-
  level evidence gathered on 2026-10-31/11-01 above was collected
  against, not v4.0.6 - worth keeping in mind as a variable for any
  future test, separately from this question.)
- *The manual default-mode test* (test_plan.md Test 7.6 case (c), logged
  above) - happened 2026-10-22 at 10:08:51 local, briefly setting
  `select.solaredge_i1_storage_default_mode` to "Solar Power Only (Off)"
  before self-correcting back to Maximize Self Consumption about 8
  minutes later. **Eight days after the 10-14 onset, and a single,
  fully-reverted flip - cannot explain 19+ days of near-daily 100% SOC.**
- *SolarEdge inverter/battery firmware* - the user raised this
  separately: no manual firmware action was taken, and there is no way
  to see from Home Assistant whether SolarEdge pushed an automatic
  update, since firmware updates happen over the inverter's own
  connection to SolarEdge's cloud, entirely outside this project's local
  Modbus integration and outside HA's visibility - HA only ever reads
  whatever version is currently active, never the update process itself.
  `sensor.solaredge_i1_version` (`0004.0025.0013`) and `sensor.
  solaredge_b1_version` (`48V DCDC 3.5.13`) were checked and have held
  those exact values across the entire retained recorder history window
  (2026-10-23 through 2026-11-01, ~10 days - as far back as this entity's
  history reaches). **No firmware change in the last 10 days, but 10-14
  itself is outside that window and cannot be checked this way.** If a
  definitive answer is needed, it would have to come from the SolarEdge
  monitoring portal directly (mySolarEdge / monitoring.solaredge.com),
  which logs the site's own firmware-update history independently of
  Home Assistant. Baseline recorded here for future comparison: as of
  2026-11-02, inverter `0004.0025.0013`, battery `48V DCDC 3.5.13`.

None of the three candidates checked explain the 2026-10-14 inflection
point. **The cause of the onset itself remains open** - logged here so
it isn't lost, not as a finding with a resolution.

Status: **open, not yet fixed.** Historical scope of the symptom is now
fully characterized (two isolated 2026-08-10/11 spikes, nine clean weeks
2026-08-12 to 10-13, then chronic from 2026-10-14); what caused the
2026-10-14 shift specifically is a new, separate open question, not
answered by either of the two register-level mechanisms above.

### Update 2026-11-02 (later same day) - a fourth candidate, found via EMHASS's own GitHub/PyPI release history: a new, unset terminal-SOC reward parameter, bracketing the onset almost exactly - strong lead, not confirmed

Per the user's request to check GitHub. The EMHASS *add-on* itself (not
the SolarEdge integration already ruled out above) was confirmed v0.18.4
and "up to date" as of 2026-10-08 (per the EMHASS Outage Fallback Guard
review that day). It is now v0.18.8 (`ha_get_app`, 2026-11-02). Something
between those updated it at least once - how many times, and exactly
when, could not be pinned down: `update.emhass_update`'s own logbook
only retains the same ~10 days as every other entity history checked in
this project, so it only shows two install cycles (2026-10-23 13:23:50 ->
2026-10-25 02:40:14, and 2026-11-01 18:00:49 -> 18:12:14) and nothing
from the 10-08 to 10-23 gap that actually covers the 10-14 onset - the
same retention wall hit on the SolarEdge firmware check above, now hit a
second time on a different entity.

PyPI's own release history for the `emhass` package (checked directly,
not through GitHub's release page, which this session could not reach
live) gives exact dates for the versions that gap could span:

```text
v0.18.4   2026-09-27   (confirmed installed + up to date 10-08)
v0.18.5   2026-10-09   feat: add battery_soc_final_reward_factor
                       (also fixes the web UI multi-battery SOC chart)
v0.18.6   2026-10-16   fix: reject a state of charge above 1 before it
                       reaches the solver
                       fix: scale the soc_final penalty by the
                       self-consumption big-M
v0.18.7   2026-10-23
v0.18.8   current (date not confirmed)
```

Why v0.18.5/0.18.6 specifically stand out: this project's own EMHASS
payload (`emhass_scripts.yaml`, both the MPC and day-ahead scripts)
explicitly sets `soc_final: '{{ battery_minimum_percent / 100 }}'` -
i.e. it deliberately targets ending every optimization horizon back down
near the battery's configured *minimum* SOC (20%), the opposite of the
100% ceiling now seen daily. It does **not** set the new
`battery_soc_final_reward_factor` key v0.18.5 introduced - that knob has
been running on whatever the `emhass` library's own internal default is,
silently, since whenever this install first crossed from v0.18.4 to
v0.18.5+. `v0.18.6`'s own changelog line - "scale the soc_final penalty
by the self-consumption big-M" - was confirmed against the live
`emhass` source (`src/emhass/optimization.py`, `SELF_CONSUMPTION_BIGM =
1e3`, with a comment stating the terminal-SOC deviation penalty is
scaled by that same factor) - i.e. a real, upstream change to exactly
the constraint this project depends on to pull SOC back down to
`soc_final` at the end of each run, landing 5-9 days after the 2026-10-14
onset in terms of raw release date, with no way from this project's own
data to confirm whether the installed add-on had already picked it up
by then.

**This is not confirmed - flagged as a strong lead, not a finding,**
for two honest reasons: (1) the exact install timing is unrecoverable
from HA's own ~10-day-retained data, same limitation as the firmware
check above, and (2) if the add-on's actual update to something >=
v0.18.6 happened anywhere close to its *release* date (10-16) rather
than lagging to the first retained logbook entry (10-23), the fix it
shipped should have started pulling the SOC ceiling back down again soon
after - but the 79/121-day survey above shows the 100% pattern
continuing unbroken all the way through 2026-11-01, past both the 0.18.6
and 0.18.7 release dates. That tension is real and unresolved: either
the fix didn't fully correct this project's specific case, or the
install lagged its release by more than a week either way, or this
mechanism isn't actually the right one. Not ruled in or out.

Recommended next steps, independent of whether this exact mechanism
turns out to be the cause:

1. Check Home Assistant's own Add-on Store page for EMHASS directly
   (Settings > Add-ons > EMHASS > the changelog/info tab) - Supervisor's
   own UI may retain a longer version-history record than the recorder's
   ~10-day window this project keeps hitting.
2. Regardless of root cause, explicitly set
   `battery_soc_final_reward_factor` in the `payload:` block of both
   `generate_emhass_energy_plan_mpc_pv` and
   `generate_emhass_energy_plan_dh_pv` (`emhass_scripts.yaml`) to a
   chosen, known value instead of leaving it on the library's internal
   default - this is a defensive fix either way, since the add-on will
   keep auto-updating and this key's default is not something this
   project controls or has verified. Not yet implemented - needs a
   decision on what value to set (and ideally upstream confirmation of
   what the current default actually is, which could not be confirmed
   from the source excerpts fetched this session).

Status: **strong candidate, not confirmed; EMHASS's internal
`battery_soc_final_reward_factor` default (introduced v0.18.5, scaling
fixed v0.18.6) is currently unset and unverified in this project's own
payload - defensive fix recommended above, not yet applied.**

### Update 2026-11-02 (later same day) - mode-switch fix implemented for the underlying Hold bug itself: High SOC Hold now uses Discharge to Minimize Import instead of Maximize Self Consumption

Separately from the search for what caused the 2026-10-14 onset (still
open, see above), the user redirected toward fixing the underlying
mechanism directly: MSC's own self-consumption logic evidently overrides
`charge_limit=0` whenever PV surplus is available (confirmed live, 10 of
11 Hold cycles 2026-10-22 to 11-01 - see this issue's main entry above),
so no value of `charge_limit` alone can make MSC actually stop charging.
The user's proposal: stop relying on MSC during Hold at all, and switch
`effective_storage_mode` itself to a different native SolarEdge command
mode instead - weighing two options, "Discharge to Minimize Import"
(battery can still cover house load, no export, discharge pinned to 0 by
default) versus "Solar Power Only" (battery fully idle - no charge, no
discharge, the stronger documented guarantee per README.md but at the
cost of losing any discharge during Hold). User chose **Discharge to
Minimize Import**, discharge limit pinned to 0.

Implemented across three files (all previously only read via
`project_search` snippets, now fully read and edited):

1. **`batterycontrol_helpers.yaml`** - added `discharge_to_minimize_import`
   to `input_select.effective_storage_mode`'s `options:` list. It was not
   previously selectable there (`solar_power_only`,
   `charge_from_clipped_solar`, `charge_from_solar_and_grid`,
   `discharge_to_maximize_export`, `maximize_self_consumption` only) -
   writing an option outside this list would have failed silently
   (HA input_select rejects it). `input_select.emhass_requested_storage_
   mode` (`emhass_helpers.yaml`) was deliberately left unchanged - the
   High SOC Hold branch writes `effective_storage_mode` directly, never
   through the "requested" side, so only one of the two input_selects
   needed the new option for this fix.
2. **`apply_effective_battery_control.yaml`** - added a new `choose:`
   branch, `effective_mode == 'discharge_to_minimize_import'` ->
   `script.discharge_to_minimize_import_script`. This script and its
   `script.modbus_queue` wiring (`queue_service: discharge_to_minimize_
   import`, which already maps to the real SolarEdge "Discharge to
   Minimize Import" command mode in `solaredge_modbusqueue.yaml`) already
   existed in `batterycontrol_scripts.yaml` - built but never called by
   anything, since README.md had always documented this mode as "Not
   used in this automation - Maximize Self Consumption covers this use
   case." That README line is now stale; not yet updated there.
3. **`safety_limits_and_override.yaml`** - the HIGH SOC HOLD branch
   (`calculate_effective_battery_control`) changed from setting
   `effective_storage_mode` to `maximize_self_consumption` (charge=0,
   discharge=dynamic `requested_discharge`) to setting it to
   `discharge_to_minimize_import` (charge=0, discharge=**0**, not the
   dynamic value - per the user's explicit "keep discharge limit to 0",
   so the battery stays passive by default during Hold rather than
   actively discharging to cover load). Reason string updated to "High
   SOC hold - discharge to minimize import, charge blocked".

Side effect, expected and not a bug: Layer 4B's Mode Command Guard
(`apply_effective_battery_control.yaml`'s `mode_guard_active`) only
skips the mode-select command when `effective_mode ==
'maximize_self_consumption'` - since Hold no longer sets that mode, the
guard will correctly stay inactive (`active`, not `guarded`) for the
entire duration of every future Hold period, resending the full command
sequence (timeout reset + mode-select) every 15-minute tick instead of
being skipped as it was before. This increases Modbus write volume
during Hold (which has run 6h45m+ at a stretch) - a real, acceptable
trade-off of this fix, not investigated further here. No guard logic was
changed; this is just a consequence of Hold now setting a different
mode.

YAML-validated (`yaml.safe_load`) on all three files before push.

**Not yet verified live - this is the key open risk.** Whether SolarEdge's
"Discharge to Minimize Import" firmware mode actually blocks charging
from PV surplus the way "Solar Power Only" is explicitly documented to
("No battery charging or discharging... Battery is idle") was not
confirmed before this change - README.md only describes this mode's
*discharge* behavior ("Battery discharges only to cover house load...
no grid export from battery"), not whether it shares MSC's internal
self-consumption logic that was just confirmed to override charge-limit
registers. If the next live Hold cycle still shows SOC climbing past the
ceiling under this mode, the documented, one-line fallback is to change
`safety_limits_and_override.yaml`'s HIGH SOC HOLD branch option from
`"discharge_to_minimize_import"` to `"solar_power_only"` (no other file
needs to change - that option and its apply-script wiring already
exist).

Also not yet done: `test_plan.md` Test 1.2 (currently FAIL, downgraded
2026-11-01) should be re-run against this change once deployed live, and
README.md's "Discharge to Minimize Import... Not used in this
automation" line and its SOC Safety Limits table ("SOC > max -> Mode:
Maximize Self Consumption, charge limit = 0, dynamic discharge limit")
both need updating to match - not done in this update.

Status: **implemented in project docs, not yet deployed to the live HA
instance or verified against a real Hold cycle.** This session has no
live write access - the user needs to push these three files and watch
the next time SOC reaches the ceiling.

### Update 2026-11-02 (next day) - fix WAS deployed and the decision layer worked correctly; the inverter's mode silently reverted 15 minutes later, letting the already-documented MSC bug take back over

The user reported "today we again ran above the SOC limit and ran at
100% despite our last fix." Checked live, read-only. An earlier draft of
this update wrongly concluded `apply_effective_battery_control_to_
solaredge_inverter` never ran after 10:43 - that was a timestamp-reading
error on this session's part (Home Assistant's trace API returns UTC;
`10:42:03 UTC` is `11:42:03` local, the same instant as everything else
below, not three hours earlier). Corrected using a full trace the user
pasted directly from the live UI.

**The mode-switch fix is confirmed deployed and the decision layer is
confirmed correct.** `input_select.effective_storage_mode` changed
`maximize_self_consumption -> discharge_to_minimize_import` at exactly
11:42:03.337 local, the same instant Hold engaged (SOC crossed 90.0%).
`effective_charge_limit`/`effective_discharge_limit` both went to `0.0`
in the same instant, and the reason text updated correctly. Not in
question.

**The apply automation also ran correctly and did get the inverter into
the new mode - briefly.** `automation.apply_effective_battery_control_
to_solaredge_inverter` triggered at 11:42:03, queued all three writes
(command-timeout refresh, mode -> Discharge to Minimize Import, then
charge/discharge limit -> 0), and its own write-verify step logged all
three as **failed** 10 seconds after each queue (`input_text.modbus_
last_failed_reason`, e.g. "Entity select.solaredge_i1_storage_command_
mode expected Discharge to Minimize Import but reads Maximize Self
Consumption (after ~10s)"). Checking the actual entity history shows
this was a **false failure**: `select.solaredge_i1_storage_command_mode`
shows "Discharge to Minimize Import" at **11:43:35** - about 10 seconds
*after* the verify check already gave up. The Modbus queue was running
slow that minute (`modbus_queue_last_execution_seconds` logged 21.95s,
18.02s, 8.05s, 18.01s for the four queued commands in this one
automation run - each command taking far longer than the verify step's
fixed ~10s wait), so the fixed-delay verify is checking before a
backlogged queue has actually gotten to the command. This tripped
`input_number.modbus_write_consecutive_failures` 1 -> 2 -> 3 and set
`binary_sensor.solaredge_modbus_write_unhealthy` to `on` - both still
`on`/`4` as of this writing, since nothing has run a fresh write to
clear them. This false-failure/premature-verify issue is real but
separate from what actually caused the SOC excursion, see below.

**What actually caused SOC to reach 100%: the inverter's live command
mode reverted to Maximize Self Consumption on its own, 15 minutes after
the fix applied, with no project-side trigger.** `select.solaredge_i1_
storage_command_mode` history: `Maximize Self Consumption` (pre-Hold) ->
`Discharge to Minimize Import` at 11:43:35 (the fix landing, confirmed
above) -> back to **`Maximize Self Consumption` at 11:58:00.197** -
fifteen minutes later, with `input_select.effective_storage_mode`
(this project's own intended-mode helper) confirmed unchanged at
`discharge_to_minimize_import` throughout, and no automation of this
project's own touching `select.solaredge_i1_storage_command_mode`
directly (it's only ever set by the apply scripts, and nothing re-ran
until much later). Once back on Maximize Self Consumption, the
already-documented root mechanism of this same issue - MSC's own
self-consumption logic overriding `charge_limit=0` (see this issue's
main entry above) - took back over uncontested for the next ~2 hours,
and SOC climbed from there to 100.0% at 14:02:23.

This 11:58:00 mode-revert is the same shape of event this issue's
2026-11-01 Update already documented and attributed to a Modbus
disturbance resetting inverter registers to non-project defaults
(`inverter_default_mode`, confirmed elsewhere as Maximize Self
Consumption) - not a new mechanism, but a second live occurrence of it,
this time undoing the High SOC Hold fix specifically rather than just
the limit registers. (Separately, another clear instance of the same
family happened later today: `select.solaredge_i1_storage_command_mode`
and both limit registers went `unavailable` at 18:07:55 and came back at
18:08:18 reading Maximize Self Consumption / 5000 / 5000 - the same
5000/5000 non-project-default pattern as 2026-11-01's documented case.)

**The real gap, confirmed now by a second occurrence: nothing re-asserts
or monitors the live inverter mode after the one-shot apply.**
`apply_effective_battery_control_to_solaredge_inverter` only fires on a
state *change* of `effective_storage_mode`/`effective_charge_limit`/
`effective_discharge_limit` - once those are set and applied, nothing
checks that `select.solaredge_i1_storage_command_mode` still agrees with
`effective_storage_mode` afterward. A silent revert like the one at
11:58 - from whatever causes it - goes completely undetected until the
next time an `effective_*` helper happens to change for an unrelated
reason (here, that didn't happen again until well after SOC had already
hit 100%).

Separately, a confirmed, reproducible crash bug was found in this same
automation from the night before (2026-11-01 22:01:11 local): a Modbus
transaction logged taking 69.7 seconds, and `input_number.modbus_queue_
last_execution_seconds`'s own 0-60 range rejected the write, raising an
uncaught exception that killed that run outright. Not tied to today's
SOC excursion (a separate successful run happened this morning after
that crash), but a real vulnerability given this system's Modbus
reliability today (`modbus_write_failure_count_today` 43+,
`modbus_queue_timeout_count_today` 7+).

Not yet proposed as a fix: nothing changed in any YAML file for this.
Three independent things worth doing, none implemented: (1) a watchdog
that compares `effective_storage_mode` against `select.solaredge_i1_
storage_command_mode` (and the limit registers against `effective_
charge_limit`/`effective_discharge_limit`) on a short interval, and
re-applies if they've drifted - this project already has the watchdog
pattern (`emhass_watchdog_day_ahead_missing_or_failed`, `emhass_
watchdog_mpc_stalled`); this is the same idea applied to the
decision-to-inverter path, and it's the fix that would have caught the
11:58 revert within minutes instead of hours; (2) make the write-verify
step's wait adaptive to queue backlog (or just re-check once more before
declaring failure) instead of a fixed ~10s, to stop generating false
failure/unhealthy signals when the queue is simply slow; (3) loosen or
clamp `input_number.modbus_queue_last_execution_seconds`'s 0-60 range so
a slow transaction logs a fact rather than crashing the logging
automation.

Status: **open, not fixed.** The mode-switch fix (this issue's main
entry) is deployed and confirmed correct at both the decision layer and
the initial apply; the newly-confirmed gap is that nothing defends
against - or even notices - the live inverter silently reverting away
from the commanded mode afterward, which is what actually let SOC reach
100% today.

### Update 2026-11-02 (later same day) - day/night mode split replaces the single-mode Hold fix, plus a new Battery Control Drift Watchdog to catch future silent reverts

Per the user's explicit direction ("lets create the watchdog, and lets
change the mode of working to 'solar power only' when SOC >90% and PV
>0 (above floor), and keep discharge to maximize export during night
when PV=0 (minus floor)"), two further changes, both addressing gaps
confirmed by the two Updates above:

**1. High SOC Hold no longer uses one mode for the whole Hold period -
it now splits by whether PV is actually present.** The previous fix (the
first Update above) moved Hold from Maximize Self Consumption to
Discharge to Minimize Import, but whether that mode's *charge* behavior
under PV surplus is actually any safer than MSC's was explicitly flagged
there as unconfirmed - and the very next Hold cycle's SOC excursion (the
second Update above) was caused by a different mechanism entirely (a
silent mode revert), so that question was never actually tested either
way. Rather than deploy and wait again, `safety_limits_and_override.yaml`
now decides between two already-existing, already-wired modes based on
live PV:

- **Daytime (PV > the noise floor):** `solar_power_only` - charge AND
  discharge both pinned to 0. This is README.md's strongest documented
  guarantee ("No battery charging or discharging... Battery is idle"),
  closing off the MSC-style charge-limit-override vulnerability
  entirely rather than trusting any mode's charge behavior under PV
  surplus to be safely assumable.
- **Nighttime (PV at/below the noise floor):** `discharge_to_maximize_
  export` - charge pinned to 0, discharge set to `input_number.
  dynamic_discharge_limit` (not pinned to 0, unlike the daytime branch).
  With no PV, there's no recharge risk, so instead of sitting idle the
  battery actively discharges toward export to bring SOC down ahead of
  the next day's solar - a deliberate behavior change from the previous
  single-mode fix, which left the battery passive (discharge pinned to
  0) during Hold regardless of time of day.

The PV reading used is `sensor.solar_panel_production_w` floored to 0
below `input_number.pv_noise_floor_w` (default 20W) - the same noise-
floor pattern already proven in `battery_forecast_control.yaml`'s own
`pv`/`pv_now` variables (see PV Sensor Trickle Readings Bypass the "No
PV" Branch Logic, above), replicated here since Layer 1 didn't
previously look at PV at all. Both `solar_power_only` and `discharge_
to_maximize_export` were already selectable in `input_select.
effective_storage_mode` and already had working `apply_effective_
battery_control.yaml` → script wiring before today (unlike `discharge_
to_minimize_import`, which needed new plumbing in the first Update
above) - no changes needed to `batterycontrol_helpers.yaml`,
`apply_effective_battery_control.yaml`, or `batterycontrol_scripts.
yaml` for this part.

**2. A new watchdog closes the "nothing notices a silent revert" gap
identified in the Update directly above.** `watchdog_automations.yaml`
gains `battery_control_drift_watchdog`: every 5 minutes, while the
killswitch (`input_boolean.effective_battery_control_enabled`) is on and
the inverter is in Remote Control (the same precondition
`apply_effective_battery_control_to_solaredge_inverter` itself requires
- no point flagging or trying to fix drift the project has no authority
to correct anyway), it compares:

- `select.solaredge_i1_storage_command_mode` (live) against `input_
  select.effective_storage_mode` (intended), translated through the same
  mode-string mapping `script.modbus_queue` itself uses in `solaredge_
  modbusqueue.yaml` (kept as a literal copy here, not a shared helper -
  the two need to be kept in sync by hand if either changes);
- `number.solaredge_i1_storage_charge_limit`/`_discharge_limit` (live)
  against `input_number.effective_charge_limit`/`_discharge_limit`
  (intended), with a 50W tolerance to avoid flagging register-rounding
  noise.

On any mismatch, it increments two new diagnostic counters (`input_
number.battery_control_drift_count_total`/`_today`, added to `safety_
and_watchdog_helpers.yaml`, the `_today` one reset at midnight by
`reset_watchdog_daily_counters` alongside the existing Modbus counters),
records what disagreed in a new `input_text.battery_control_last_drift_
reason`, sends a persistent notification plus a mobile push, and then
force-re-applies by calling `automation.trigger` on `apply_effective_
battery_control_to_solaredge_inverter` directly (`skip_condition:
false`, so that automation's own guard conditions still apply) rather
than duplicating its script-dispatch logic here. This specifically would
have caught the 11:58:00 silent revert documented in the Update above
within 5 minutes, instead of the ~2 hours it actually took to reach
100% SOC undetected.

Not addressed by this watchdog, deliberately: it treats a mismatch as
something to re-apply, not diagnose - it doesn't distinguish a genuine
external revert (the mechanism seen today) from, say, a write genuinely
still in flight, though the 5-minute interval (versus the ~15-55s normal
write lag seen elsewhere in this project) makes a false-positive from
ordinary lag unlikely. It also doesn't address the two separately-
flagged, still-open telemetry bugs from the Update above (the fixed-
~10s write-verify window producing false failures under a backlogged
queue; the 0-60s range crash in `modbus_queue_last_execution_seconds`) -
neither is fixed here.

All three files (`safety_limits_and_override.yaml`, `watchdog_
automations.yaml`, `safety_and_watchdog_helpers.yaml`) YAML-validated
(`yaml.safe_load`) before push.

Status: **implemented in project docs, not yet deployed to the live HA
instance or verified against a real Hold cycle (either the daytime or
nighttime branch) or a real drift event.** This session has no live
write access - the user needs to push these three files and watch the
next time SOC reaches the ceiling, ideally once during the day (PV
present) and once at night, plus watch for the drift watchdog firing
`persistent_notification`/mobile push if the inverter ever reverts again.

---

## Critical-Recovery Charge Limit Write Silently Dropped

Found 2026-10-11, while verifying a real SOC<10% critical-recovery event
against the user's request to check it against the stats (see the
companion issue below for the trigger itself). The Layer 1 logic and the
mode/discharge-limit writes all checked out - but the charge-limit write
did not.

Timeline (local time, 2026-10-11): `calculate_effective_battery_control`
fired correctly at 15:47:41.92, setting `input_select.effective_storage_
mode` -> `charge_from_solar_and_grid`, `input_number.effective_charge_
limit` -> 3000, `input_number.effective_discharge_limit` -> 0. The apply
automation (`apply_effective_battery_control_to_solaredge_inverter`,
`mode: restart`) then dispatched all three physical writes in series
through `script.modbus_queue`. Confirmed via automation trace + the
`modbus_queue` telemetry helpers:

```text
storage command timeout   dispatched 15:47:42.09, ~26.5s (incl. a 10.03s
                           internal queue wait, logged as timeout #2 of
                           the day - recovered, not a hard failure)
mode write (charge_from_solar_and_grid)
                           dispatched 15:48:10.63, 18.04s execution,
                           reported success
charge limit write (3000) dispatched 15:48:30.75, 18.14s execution,
                           reported success - no error, no timeout
discharge limit write (0) dispatched 15:48:50.96, 18.17s execution,
                           reported success
```

Physical register history confirms mode and discharge limit both landed
correctly - `select.solaredge_i1_storage_command_mode` ->
"Charge from Solar Power and Grid" and `number.solaredge_i1_storage_
discharge_limit` -> 0, both at 15:49:19.00 (within ~1.2ms of each other,
consistent with both riding the same coordinator poll cycle that first
reflected the already-completed writes).

`number.solaredge_i1_storage_charge_limit`, however, never shows 3000 at
all. Its full history across this window: 3300 (baseline) -> **5000** at
15:47:41.436 (half a second *before* the critical branch even fired - a
leftover from a different, unrelated write, most likely
`battery_forecast_control.yaml`'s Clipped Solar branch, which hardcodes
`value: 5000`) -> stays at 5000 continuously through both the SOC<10
critical band and the following SOC 10-20% band -> finally changes,
**straight to 3300** (skipping 3000 again), at 16:15:37 - nearly 28
minutes later, coincident with a later, unrelated burst of modbus_queue
activity. Both the automation's own trace and the modbus_queue telemetry
reported the 3000W write as a clean, timely success - there is nothing
in Home Assistant's own records that flags this write as having failed.

This means the charge-rate ceiling actually enforced at the inverter
(5000W, a stale leftover from a different branch) was looser than the
3000W the critical-recovery logic intended, for the entire ~28-minute
critical-recovery window. Not a direct safety hazard - the two controls
that actually prevent over-discharge (mode and discharge limit) both
landed correctly and promptly - but it is a real gap in the control
chain that nothing currently detects: the existing write-outcome
telemetry (see Write-Outcome Telemetry Is Readback-Based below) reads
back via `script.modbus_write_verify`, dispatched fire-and-forget from
inside `script.modbus_queue`'s own sequence - but traces for
`script.modbus_queue` and `script.modbus_write_verify` only retain their
last 5 runs each, and by the time this was investigated (after the
scheduled post-restart EMHASS check-in that prompted a return to this
session), those traces from 15:48 had long since rolled past that limit.
So it isn't known what, if anything, `modbus_write_verify` itself
reported for this specific write - only that the entity's real-world
value never reflects the intended one.

Not yet root-caused at the Modbus/inverter level. Two hypotheses, neither
confirmed: (a) the SolarEdge inverter's firmware may gate charge-limit
register writes on the command mode already being in a compatible state,
and the mode write for this same cycle hadn't yet taken hold on the wire
when the charge-limit write arrived 20 seconds later; (b) a genuine
dropped/overwritten write at the Modbus transport layer, independent of
mode timing. Needs a fresh `script.modbus_queue`/`script.modbus_write_
verify` trace captured *during* a future occurrence (caught promptly,
before the 5-trace retention rolls it off) to distinguish between these -
not proposing a blind fix without that.

Status: **open, not yet fixed**. `test_plan.md` Test 1.3 downgraded
from the "NOT VERIFIED" placeholder to PARTIAL PASS on this basis - mode
and discharge-limit application confirmed live, charge-limit application
confirmed *not* reaching the inverter.

---

## SOC<10% Critical Recovery Triggered By a Physically-Implausible Single-Sample SOC Reading

Found 2026-10-11, investigating the same event as above, in response to
the user's question: why was the battery allowed to discharge to 0% at
all, when `input_number.minimum_state_of_charge` (20%) and the Layer 1
"SOC < MIN" branch should stop discharge well before that point?

Short answer: the live evidence says the battery most likely never
actually reached 0% - the SOC reading itself looks like a bad single
sample, not a real measurement, and the existing SOC-sensor guard
(`calculate_effective_battery_control`'s unavailable/unknown condition)
doesn't catch it because a bad `0.0` is a syntactically valid state, not
`unavailable`/`unknown`.

Evidence: `sensor.solaredge_b1_state_of_energy` history for 2026-10-11
shows a slow, steady decline through the afternoon - 27.78% (14:00) ->
26.67% (14:14) -> 25.56% (14:35) -> 24.44% (14:56) -> 23.33% (15:16) ->
22.22% (15:36:15) - roughly 1.1 points per ~20 minutes. The very next
recorded sample is **0.0% at 15:47:41.905** - an 11.43-minute gap with
*zero* intermediate samples, unlike every other transition in this
dataset. For contrast, the recovery climb immediately afterward (0.0% ->
20.0%, 15:47:41 to 16:15:37) shows 20 clean, evenly-spaced samples at
~1.11-point steps roughly every 70-90 seconds - a textbook real charge
curve, consistent with the ~3000W critical-recovery charge rate on this
4600Wh battery. The drop has none of that texture.

Physics check, using this project's own numbers: `input_number.
effective_discharge_limit` (and the physical discharge-limit register)
had been pinned at **100W** continuously since well before 14:00 (the EV
Charging branch - `binary_sensor.ev_charging_active` was "on" the whole
window). At 100W for 11.43 minutes, maximum possible energy drawn is
~19Wh - about 0.4% of the 4600Wh battery, nowhere near the observed
22.2-point (~1021Wh) drop. Even granting the maximally generous (and
unproven) assumption that the 100W cap itself was silently unenforced the
whole time (the same class of gap as the issue above) and the battery
discharged continuously at its own absolute hardware ceiling
(`battery_discharge_power_max: 4096W`, confirmed live from EMHASS's own
MPC payload), the most energy that could physically have been drawn in
11.43 minutes is ~779Wh - about 16.9% of capacity. That is still short of
the observed 22.2-point drop. No combination of assumptions this
project's own numbers support explains a real, continuous physical
discharge from 22.2% to 0.0% in this window.

Most likely explanation: a bad single-sample read from `sensor.
solaredge_b1_state_of_energy` - the same Modbus-sourced entity already
documented as prone to flaky reads (see Modbus Connectivity - SOC-Sensor
Unavailable False-Trigger below, plus the recurring "Coordinator has
timed out 3 times in a row" warnings and TCP connect/close cycling
visible in the error log around this timeframe) - except this time it
returned a plausible-looking but wrong numeric value (`0.0`) instead of
`unavailable`/`unknown`. Nothing in the current guard logic screens for
that: `calculate_effective_battery_control` and `battery_high_soc_hold`
both only exclude the literal states `unavailable`/`unknown`, with no
rate-of-change or plausibility check against what the currently-enforced
limits could actually produce.

Consequence assessment: the system's *response* to this likely-false
reading was itself safe. Forcing `charge_from_solar_and_grid` and zeroing
discharge can't harm a battery that was actually healthy around 22% - it
just means an unnecessary (but harmless) ~28+ minute grid-assisted charge
cycle, plus the stale-charge-limit gap documented above compounding on
top of it. This is the same "safe side-effect of a bad trigger" pattern
already proven for the original SOC-Sensor Unavailable False-Trigger
fix - just a new variant (a bad *value* instead of a bad *state*) that
guard doesn't cover.

Proposed hardening (not yet implemented - needs a design decision, not
blind-fixed here): a plausibility check alongside the existing
unavailable/unknown guard - e.g. hold the last good SOC value (same
freeze behaviour the unavailable/unknown guard already uses) whenever a
new reading implies a single-step change bigger than what the
currently-enforced discharge/charge limit, or the battery's absolute
max charge/discharge power, could physically produce in the elapsed time
since the previous reading.

Status: **open, not yet fixed**. Reclassifies what the live stats
showed the user as probably a sensor glitch responded to safely, not a
genuine near-empty-battery near-miss - `test_plan.md` Test 1.3 updated
accordingly (see below).

---

# Fixed Issues

## Modbus Queue Pending-Count Counter Leaks On Lock-Wait Timeout

Found 2026-10-19 during a routine review: `binary_sensor.
solaredge_modbus_queue_backlog` had been stuck "on" for over 33 hours,
though real command throughput was normal throughout. Root cause:
`input_number.modbus_queue_pending_count`'s +1/-1 pairing leaked on
every lock-wait timeout (the 10s `wait_template` on `modbus_busy` timing
out via `continue_on_timeout`), permanently incrementing the resting
floor instead of returning it to 0 - confirmed by a 1:1 lockstep
correlation with `modbus_queue_timeout_count_total` across multiple
days, strong enough to act on without a direct trace of the leak itself.
Fixed the same day in `solaredge_modbusqueue.yaml`: the lock-wait-timeout
branch now also decrements `modbus_queue_pending_count` by 1 (floored at
0), making the counter self-correcting regardless of the exact internal
`mode: queued` mechanism that caused the leak. Deployed live 2026-10-20
and confirmed against a real overnight timeout event the same night: the
pending-count floor returned to its pre-fix resting value (10) instead
of climbing to 11, with the compensating decrement firing within 5ms of
the timeout counter's own increment. The pre-fix resting value itself is
a one-time debt this fix doesn't retroactively repay (clears on manual
reset or the next HA restart). Status: fixed, deployed live, and
confirmed against a real timeout event as of 2026-10-20; `test_plan.md`
Test 6.4 upgraded to PASS.

---

## EMHASS Rejects Optimisation Requests When `sensor.max_active_power_1` Goes Unavailable

Found live 2026-10-11: `sensor.max_active_power_1` went unavailable at
11:46:12 local, causing `emhass_scripts.yaml` to silently feed `0` into
both `inverter_ac_output_max` and `inverter_ac_input_max`, which made
every MPC/day-ahead REST call fail with HTTP 400 for the rest of the
day. Root cause, traced precisely rather than inferred: the entity
belongs not to the local `solaredge_modbus_multi` Modbus integration
(which stayed healthy throughout) but to a separate cloud-API
integration (`solaredgeoptimizers`), whose login endpoint was returning
HTTP 410 Gone - SolarEdge retiring the legacy portal login, unrelated to
the Modbus/HA side the user had initially suspected. Fixed the same day
by removing the entity from EMHASS's critical path entirely: a new
static helper, `input_number.emhass_max_inverter_power` (initial 10600W,
the hardware's nameplate AC rating, which essentially never changes),
replaces it in both EMHASS payload scripts, with the fallback hardened
from `0` to `10600`. An existing but unrelated helper
(`input_number.ac_maxpower`, 5000W, a different Layer 4A threshold) was
deliberately not reused, since it represents a much lower value for a
different purpose. Confirmed live 2026-10-11: the next MPC cycle (16:15
local) succeeded, with the request payload showing the correct `10600`
values, no further HTTP 400s, and `sensor.emhass_health` back to OK. The
45-minute Maximize-Self-Consumption fallback remains in place as a
second line of defense regardless. Status: fixed and confirmed live
2026-10-11.

---

## Write-Outcome Telemetry Is Readback-Based

Added 2026-10-06 as part of Enhanced Diagnostics: `script.modbus_queue`
began classifying every write as success/failure/unmatched_service by
snapshotting the target entity before dispatch and reading it back ~2
seconds later, since Home Assistant's `continue_on_error` gives no other
way to detect a caught write failure. Live verification the next day
found this 2-second window far too short - roughly half of real writes
in a short test window were misclassified as `failure` because the
entity simply hadn't settled yet (confirmed via entities that read back
their old value at 2s but the correct new value ~10 seconds later).
Root cause of why the delay couldn't just be lengthened inline:
`script.modbus_queue` runs `mode: queued`, so any added wait would have
serialized and delayed the next queued command, reintroducing the same
write-pressure problem tracked separately in roadmap.md. Fixed
2026-10-07 by splitting verification out into a new, independently
queued script (`script.modbus_write_verify`), dispatched fire-and-forget
via `script.turn_on` with variables, which waits up to 8 more seconds for
the real value before classifying the outcome - leaving
`script.modbus_queue` itself unblocked. The first live redeploy attempt
only added the new script without rewiring `script.modbus_queue` to call
it (a partial-copy deployment gap, confirmed via trace showing the old
inline classification still running and a zero-trigger
`script.modbus_write_verify`), so the old, too-fast classification kept
misfiring live for about a day. A corrected redeploy was confirmed wired
in and working correctly via a real write on 2026-10-07: the trace
showed the fire-and-forget dispatch with variables correctly passed and
non-blocking execution, with the classification logic itself producing
just one false failure (likely a restart-related slow settle) in that
verification window. Status: confirmed wired in and working live as of
2026-10-07; the 8-second window's adequacy under normal (non-restart)
conditions was still being watched as of that date.

---

## Entity ID Documentation Mismatches (Derived From Name/Alias, Not unique_id/id)

Found 2026-10-07 while verifying the Enhanced Diagnostics deployment:
three entity_ids documented in this project's docs did not match what
Home Assistant actually assigned live (`binary_sensor.
solaredge_modbus_write_health` vs. actual `_write_unhealthy`;
`automation.reset_watchdog_daily_counters` vs. actual
`watchdog_reset_daily_diagnostic_counters`; `automation.
reset_modbus_queue_daily_counters` vs. actual
`reset_modbus_queue_daily_telemetry_counters`). Root cause, the same in
all three cases: Home Assistant derives a template entity's entity_id by
slugifying its `name:` field (and an automation's by slugifying
`alias:`), never from `unique_id`/`id:` - so a friendly name chosen for
an unrelated reason (avoiding a naming ambiguity elsewhere) produced a
different entity_id than assumed. Fixed 2026-10-07 by correcting all
three references across entities.md, roadmap.md, and this file - no
YAML was changed, since the entities themselves were never broken, only
the documentation pointing at them. Status: docs corrected 2026-10-07,
no live/functional impact.

---

## Negative Price Curtailment - Site Limit Restore Gap

Found 2026-10-06: the `negative_site_limit_off` handler in
`solaredge_modbusqueue.yaml` turned the curtailment switch off but never
restored `number.solaredge_i1_site_limit` from 0 back to a normal value
- confirmed live, the number sat at 0 for nearly 6 hours after the switch
and the price both returned to normal, the exact asymmetry
`test_plan.md` Test 3.2 had been waiting to verify since 2026-07-30.
Empirically confirmed harmless the same day via real power-flow
arithmetic: `switch.solaredge_i1_negative_site_limit`, not the number, is
the actual enable/disable gate on this installation, so a stale 0 under
an "off" switch had no effect on real export. Closed the same day via a
structural redesign (the user's proposal) rather than a patch: the
switch now stays permanently ON, and `number.solaredge_i1_site_limit`
itself is varied between 0 (curtailed) and 1,000,000W (normal) on every
price transition, eliminating anything to "forget" to restore;
`negative_price_curtailment.yaml`'s branch conditions now key off the
number's value instead of the switch's state, with a defensive
re-assert of the switch if ever found off. Confirmed live 2026-10-19
against a real negative-price episode: the defensive re-assert fired
correctly on the very first negative-price tick since the redesign, and
the number correctly tracked three full curtail/restore cycles in about
20 minutes of price noise around zero; the switch then stayed on through
6+ days of routine Modbus reconnect blips. The one still-unobserved
scenario - a negative-price window coinciding with real PV surplus, to
directly confirm export actually gets suppressed - remains open and is
tracked separately in roadmap.md. Status: closed and confirmed live
2026-10-19 for the switch self-heal and curtail/restore mechanism.

---

## EMHASS Outage Fallback Guard

Found 2026-09-22: a power outage during which the EMHASS addon got stuck
rebuilding itself on every restart (it rebuilds its Python environment
live against pypi.org on every start, and DNS hadn't recovered yet) went
undetected by Layer 4A (`emhass_battery_forecast_control`), which had no
guard against EMHASS being down - its `batt_fc`/`grid_fc`/`price_import`
variables all fell back to `0` via `| float(0)`, satisfying several
branches as if EMHASS had forecast a neutral/free-price plan. Live
history confirmed real harm, not a theoretical gap: the requested mode
oscillated between grid-charge and MSC on almost every 15-minute tick
for 6+ hours, and SOC sawtoothed between ~19% and 34% repeatedly while
import prices were genuinely high. Fixed in `battery_forecast_control.
yaml` with two changes: a new automation-level condition requiring
`sensor.emhass_health` not be `MPC stalled`/`MPC problem` (skipping the
whole run and freezing the last good values instead of recomputing from
fabricated zeros), plus hardening the price-based fallback branch to
require the import-price sensor not be unavailable/unknown (closing a
narrower case where only the price feed fails). Separately, a new
watchdog automation now calls `hassio.addon_restart` if MPC has been
stalled over 90 minutes, rate-limited to once per hour, to recover
automatically from a build-failure loop like this one. Deployed live
2026-09-23 and structurally confirmed via automation trace the same day
(all 3 expected conditions evaluating correctly), though a near-miss
health flicker the following evening was inconclusive as live behavioral
evidence (consistent with the deployment's own automation reload, not a
real EMHASS failure). Status: deployed and structurally confirmed live
2026-09-23; full behavioral confirmation (a real skipped run or a real
addon auto-restart) still awaited a future occurrence as of that date.

---

## Modbus Queue Lock Release Not Exception-Safe

Found 2026-08-28: `script.modbus_queue`'s lock-release step sat at the
end of a plain linear sequence, unprotected against an exception raised
inside the preceding `choose:` block - a failed SolarEdge write (e.g. a
real "Not connected" error) aborted the script before releasing
`input_boolean.modbus_busy`, leaving it stuck "on" for up to ~30 minutes
until the next run's own timeout self-healed it. A naive fix (bare
`continue_on_error` on the release step alone) was deliberately deferred
for weeks, since a 48-hour error-log review found a non-trivial baseline
of connectivity noise, and releasing the lock instantly on every failure
risked flooding a struggling connection with the queued backlog. Fixed
2026-09-20 by adding `continue_on_error: true` to the whole `choose:`
block itself (so an exception falls through rather than aborting) and
consolidating ~13 per-branch trailing delays into one uniform 2-second
delay after the `choose:` block, so the same pacing applies on the
failure path as the success path. Status: implemented 2026-09-20 and
behaviorally confirmed live as of 2026-09-24 - zero stuck-lock episodes
over 60 seconds across ~4.5 days and ~750 on/off cycles sampled,
including real connectivity noise events during that window that
released cleanly in 18-33 seconds (versus the old ~30-minute stuck
pattern).

---

## Layer 4A SOC-Unavailable Guard

Found 2026-09-19 during a routine log review: `sensor.
solaredge_b1_state_of_energy` had a real ~31-hour unavailable gap
(2026-09-16 to 09-17, ending with an HA restart). The existing
unavailable/unknown guard on `calculate_effective_battery_control` held
perfectly throughout, but `emhass_battery_forecast_control` (Layer 4A)
had no equivalent guard - its `soc` variable's `| float(0)` fallback
meant it ran every 15 minutes on a fake `soc=0` for the full 31 hours,
landing (by luck) on the conservative `solar_power_only` branch rather
than a dangerous one. Fixed by adding the same automation-level
unavailable/unknown condition already proven elsewhere as a second
condition on `emhass_battery_forecast_control`, skipping the whole run
while the sensor reads bad instead of computing a fake value. Status:
deployed and confirmed live 2026-09-24 against a real ~2h13m
SOC-unavailable window the guard actually caught - the requested mode
and limits stayed frozen solid across all 9 ticks inside the outage,
with the first real change happening only on the first tick after
recovery; whether a different, non-conveniently-safe fallback branch
would also be caught correctly remained unexercised by either observed
outage.

---

## EV Charging Sensor Swap

`binary_sensor.ev_charging_on` was found stuck `unavailable` since at
least 2026-09-03 (a dead, disconnected template entity), silently
preventing the EV-charging branches in `calculate_effective_battery_
control` and `ev_charging_discharge_control` from ever firing during
real EV charging sessions. Fixed by swapping to `binary_sensor.
ev_charging_active` (a separate, live sensor that's only "on" while
current is actually flowing) across all four referencing files; verified
live 2026-09-15 against a real EV session, with the sensor, the reason
text, and the discharge-limit trace all changing in lockstep. A
follow-up gap was found 2026-09-24: the swap had missed `calculate_
effective_battery_control`'s own trigger entity_id list, which still
named the dead sensor, so a real EV-charging state change wouldn't
state-trigger promptly (a time_pattern backstop trigger made a full miss
unlikely, but reaction could be delayed to the next scheduled tick).
Fixed 2026-09-25 by swapping the trigger list to the same live sensor,
matching the action-logic variable fixed on 2026-09-15. Status:
implemented 2026-09-25, not yet confirmed against the live file or a
real EV session.

---

## Align Safety SOC Limits And EMHASS SOC Limits

The safety layer and EMHASS each kept independent, hardcoded SOC min/max
values that only agreed by coincidence. Fixed 2026-09-03: both the MPC
and day-ahead EMHASS scripts now read `input_number.minimum_state_of_
charge`/`maximum_state_of_charge` via template (same fallback defaults
as before), so the two layers only diverge if the helper is deliberately
changed; `maximum_power_from_grid`/`maximum_power_to_grid` remain
hardcoded, with no safety-layer equivalent to align with. Status:
deployed and confirmed 2026-09-04 (live file matched verbatim), and
runtime-confirmed 2026-09-24 via the EMHASS addon's own log of passed
runtime parameters, which exactly matched the live helper values at the
same timestamp - the template hookup works end-to-end, not just in the
script file.

---

## Modbus Connectivity - SOC-Sensor Unavailable False-Trigger

Found 2026-09-03: `sensor.solaredge_b1_state_of_energy` briefly drops to
unavailable/unknown most nights (~22:00-22:10 local, a genuine Modbus
TCP connect failure, clustering cause still unexplained), and its
`| float(0)` fallback turned each dropout into a fake `soc=0`,
triggering a real hardcoded 3000W grid-charge command for the glitch's
duration - confirmed across all 6 dropout nights via `modbus_busy`/
`effective_storage_mode` history. Fixed by adding an automation-level
unavailable/unknown guard on `calculate_effective_battery_control` (skip
the run instead of computing a fake soc=0) and the same guard on
`battery_high_soc_hold` (same fallback pattern would have falsely exited
an active high-SOC hold). Status: deployed and verified - zero spurious
triggers across the following week's 6 dropouts (2026-09-04 to 09-09);
the underlying nightly Modbus dropout itself remains open in roadmap.md.

---

## Midday Charge/Discharge Oscillation - Grid Charge Export Cooldown

Implemented 2026-08-27 to address the consequence (not the underlying
cause) of a midday charge/discharge oscillation: a hysteresis cooldown
(`input_number.grid_charge_export_cooldown_minutes`, default 60) is
stamped whenever `charge_from_solar_and_grid` fires, and the DISCHARGE
MAX EXPORT branch only sells back to the grid once that cooldown has
elapsed, otherwise falling back to `maximize_self_consumption`. Full
design documented in architecture.md - Layer 4A - Grid Charge Export
Cooldown. The underlying causes (EMHASS MPC plan chatter, cooldown-
duration tuning) remain open in roadmap.md. Status: implemented
2026-08-27.

---

## Midday Charge/Discharge Oscillation - PV/Load Sampling Smoothing

Added 2026-08-28: `sensor.pv_production_smoothed`/`house_load_smoothed`
(4-minute trailing mean), used for every mode-selecting branch except
Maintain Zone's charge-ceiling choice (which keeps raw instantaneous
readings since it can't itself cause mode-flapping). Deployment required
a full HA restart rather than a reload, since `platform: statistics` is
a legacy, non-hot-reloadable sensor platform. Status: deployed and
verified live 2026-08-29 - zero oscillation across a full morning,
including two ticks where the raw/smoothed split demonstrably prevented
a stale-average charge-ceiling open; below-SOC/Clipped-Solar behaviour
under the smoothed variables still awaited a real exercising event at
the time.

---

## Midday Charge/Discharge Oscillation - Branch-Coverage Gaps

Three gaps found in `battery_forecast_control.yaml`'s top-level
`choose:` block (which has no `default:`), each a case matching no
branch and silently leaving the requested mode/limits unchanged for that
cycle: (1) DISCHARGE MAX EXPORT with `batt_fc==0` (found 2026-08-29,
fixed 2026-09-03 by widening `batt_fc > 0` to `>= 0`); (2) a BELOW-SOC
no-PV/batt_fc==0 catch-all (found and fixed 2026-09-03 by dropping an
unnecessary `grid_fc > 0` requirement); (3) DISCHARGE MIN IMPORT/MAX
SELF-CONSUMPTION with `batt_fc>0` (found 2026-09-10, fixed 2026-09-15 by
dropping the `batt_fc <= 0` clause, since the action was already
sign-agnostic). Gap 3's rollout hit a stale-config snag: traces right
after the first "deployed" still showed the old pre-fix behavior, which
`ha_eval_template` proved was a not-yet-reloaded live config rather than
a bad fix; after the user reloaded automations, the next cycle fired
correctly end-to-end. The mirror-image case - `(maintain or above)` with
`grid_fc < 0` and `batt_fc < 0` together - remains open, not observed
live and not blind-fixed. Status: all three gaps deployed; gaps 1 and 3
verified live (3 as of 2026-09-15 ~19:30 UTC), gap 2 still unexercised.

---

## Write Pressure - Mode Command Guard

Implemented 2026-08-26: `apply_effective_battery_control_to_solaredge_
inverter` skips the mode-select command and the 15-minute
command-timeout reset whenever the effective mode, the inverter's live
command mode, and its configured default mode all already agree on
Maximize Self Consumption - only the charge/discharge limit writes still
go out in that case. `sensor.battery_mode_command_guard` exposes whether
the guard is active, and a skip counter (added 2026-10-06) tracks every
tick it actually skips. Deployed and confirmed working 2026-08-27
(test_plan.md Test 7.6), though the before/after impact on the overall
Modbus error rate was never quantified. The one precondition never yet
exercised live - the guard staying inactive when the configured default
mode disagrees with MSC - was tested directly 2026-10-22 with the user's
explicit approval, deliberately changing the live default mode: the
guard correctly flipped to inactive immediately, and the next automation
tick correctly resent the full command sequence rather than skipping it,
with the skip counter correctly staying flat throughout. Status: all
three guard preconditions now confirmed live; test_plan.md Test 7.6
upgraded to PASS.

---

## Dynamic Discharge Limit Under-Delivers vs. EMHASS Plan During Sustained Price Spikes

Found 2026-11-02, investigating a report of "a charge around 06:30" that
coincided with the morning price peak - live data showed no actual
charging (what the user had seen spike was the SOC forecast target, not
real SOC, confirmed and accepted), but did reveal a genuine
under-delivery: the real Discharge-to-Maximize-Export response at the
price peak only reached 932W against a load spike of up to 1965W,
pulling power from the grid almost 1:1 at the literal price peak. Root
cause: `sensor.dynamic_storage_discharge_limit`/`_charge_limit` were
pure proportional controllers driven by only the single next EMHASS
forecast interval (~15 minutes ahead), so their output naturally decayed
toward zero as SOC approached that one checkpoint regardless of whether
EMHASS's actual multi-step plan (`batt_fc`, which peaked at 2136W at the
same time) wanted sustained high discharge for much longer. This
single-point design was deliberate (documented in README.md, chosen to
react fast to events like EV charging and to avoid reintroducing the
flip-flop that an earlier smoothed-target fix was built to damp - see
Midday Charge/Discharge Oscillation above), so the user chose to widen
the lookahead rather than scrap it: both sensors now target a weighted
blend of the next three future forecast points (the same proven
0.5/0.3/0.2 weighting already used for `soc_target`), with the same
fallback chain, so an isolated noisy re-plan is still damped while a
genuinely sustained trend now pulls the computed power further from
zero. Deployed to the project doc (`batterycontrol_sensors.yaml`) and
YAML-validated on 2026-11-02. Status: fix designed and written to the
project doc; not yet deployed to the live HA instance or verified
against a real event.