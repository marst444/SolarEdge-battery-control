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

Two issues are currently open: both found 2026-10-11 while verifying a
real SOC<10% critical-recovery event against the user's request to check
it against the stats - Critical-Recovery Charge Limit Write Silently
Dropped (the physical charge-limit register never actually reached the
critical-recovery value, stuck at a stale 5000W for ~28 minutes) and SOC<10%
Critical Recovery Triggered By a Physically-Implausible Single-Sample SOC
Reading (strong evidence the underlying SOC<10% condition itself may never
have been real). See Open Issues below for both. A third issue found
2026-10-19 during the same routine behaviour review - Modbus Queue
Pending-Count Counter Leaks On Lock-Wait Timeout, a telemetry-only bug
where `binary_sensor.solaredge_modbus_queue_backlog` had been reporting a
false "backlog" for extended periods while real command throughput stayed
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

Found 2026-10-19 during a routine "how has the system behaved lately"
review - not reported by the user, surfaced by `binary_sensor.
solaredge_modbus_queue_backlog` (pending-count >= 5) having been stuck
"on" continuously since 2026-10-18 09:30, over 33 hours by the time this
was found.

Evidence: `input_number.modbus_queue_pending_count` is incremented by 1
at the very start of every `script.modbus_queue` run and decremented by
1 (floored at 0) right before the lock releases at the end - on a plain
read of the YAML this pairing looks unconditional and safe, including
across the `continue_on_error: true` dispatch branch. In practice the
counter's resting floor (what it returns to between commands, rather
than its momentary +1 while a command is in flight) does not stay at 0:
it climbed to **67** over the course of 2026-10-17, was found at 0
again the morning of 2026-10-18 at 05:55:48 (coinciding with a gap in
`modbus_queue_timeout_count_total`'s own history that also resets to 0
at that exact timestamp - most likely an HA restart recreating these
`input_number` entities, not a code fix), then climbed again from 0 to
10 by 2026-10-19 09:15 and has sat exactly at 10 ever since (confirmed
live: still 10 as of this writing, with `pending_count` cycling 10↔11
every ~15 minutes as new commands are correctly queued and processed on
top of the stuck floor).

The floor's value matches `modbus_queue_timeout_count_total` (also 10
right now) at every point checked, and both climbed in lockstep,
one-for-one, through the exact same windows (2026-10-18 07:45-09:45:
timeout count 0->6, pending floor 0->6; 2026-10-19 05:00-09:15: timeout
count 6->10, pending floor 6->10; flat on both since). This is strong
circumstantial evidence that every lock-wait timeout (the initial 10s
`wait_template` on `input_boolean.modbus_busy` running out,
`continue_on_timeout: true` letting the run through anyway,
`wait.completed == false`) leaves a permanent, uncorrected +1 on
`modbus_queue_pending_count` - on top of whatever that same run's own
normal +1/-1 pair does. The exact code path producing this has not been
pinned down by reading alone: nothing in the sequence after the timeout
branch looks conditionally skipped, so a live trace caught in the act
(a `script.modbus_queue` run where `wait.completed == false`) would be
needed to confirm mechanism precisely, not just correlation - the fix
below was made on the strength of the correlation rather than waiting
for that trace.

Practical impact so far is cosmetic/diagnostic, not operational: actual
command throughput looks normal throughout this period (writes dispatch
and land correctly - e.g. a `set_storage_discharge_limit=409` write at
17:00:07 today was logged as a transient failure at 17:00:33 but the
register read back correctly at 409 by 17:00:36, i.e. retried/settled
fine), and nothing in this project currently *acts* on
`binary_sensor.solaredge_modbus_queue_backlog` being "on" (checked
`watchdog_automations.yaml` - no automation consumes it). The sensor was
just wrong, persistently, which undermined it as a diagnostic signal: it
had already cried wolf for over a day with no real backlog behind it,
and would have masked a genuine future backlog by making "on" look like
background noise.

Fixed 2026-10-19, same day as diagnosis: `solaredge_modbusqueue.yaml`'s
lock-wait-timeout branch (the `if: wait.completed == false` block that
already bumps `modbus_queue_timeout_count_total`/`_today`) now also
decrements `input_number.modbus_queue_pending_count` by 1 (floored at 0
via the same `max(...,0)` pattern the existing end-of-run decrement
uses), exactly once per timeout event. This makes the counter
self-correcting regardless of which earlier run's own decrement went
missing, rather than depending on fully proving the internal HA
`mode: queued` concurrency mechanism that caused the leak in the first
place - a pragmatic, evidence-driven fix (the 1:1 correlation with
`modbus_queue_timeout_count_total` was strong enough to act on) rather
than a mechanism-proven one. A brief undercount from this compensating
decrement is harmless for a diagnostic counter, unlike the permanent
overcount it replaces. The pre-existing end-of-run decrement's own
comment was updated to note this new compensating path as another
legitimate reason the counter might already be below where a naive
+1/-1 pairing would expect. The header `# Fixes:` comment in the YAML
now lists this entry too.

### Update 2026-10-20 - deployed live and confirmed against a real timeout event the same night

The user copied/reloaded the corrected `solaredge_modbusqueue.yaml` into
the live config. A real lock-wait timeout occurred overnight -
`modbus_queue_timeout_count_total` ticked 10 -> 11 at 04:05:24.891 local
on 2026-10-20 - giving the first chance to observe the fix against the
exact failure mode it targets.

`input_number.modbus_queue_pending_count`'s history across that event
(local times, 2026-10-20):

```text
04:05:14.722  10 -> 11   run A starts (pending +1)
04:05:14.875  11 -> 12   run B starts, stacked behind run A (pending +1)
04:05:24.896  12 -> 11   compensating decrement fires - 5ms after
                          modbus_queue_timeout_count_total's own
                          04:05:24.891 bump, i.e. the same timeout
                          branch, same step order as written
04:05:41.564  11 -> 10   run B's own end-of-run decrement
04:05:43.607  10 -> 11   run C starts (pending +1)
04:06:01.681  11 -> 10   run C's end-of-run decrement
```

Net change across the whole burst: zero - the floor returned to 10
(its pre-fix resting value) rather than climbing to 11 the way every
prior timeout event had. The 5-millisecond gap between the timeout
counter's own increment and the compensating decrement is as close to
a direct trace as live history can give (the actual `script.
modbus_queue` trace for run A itself already rolled off the 5-trace
retention by the time this was checked, per the known limitation noted
above) - it confirms the fix's step fired exactly once, exactly when
the timeout branch fired, consistent with the code as written rather
than coincidence.

Residual, not a bug: the floor itself is still sitting at 10, not 0 -
this fix stops *future* leaks, it does not retroactively repay the
debt accumulated before 2026-10-19. `binary_sensor.
solaredge_modbus_queue_backlog` will keep reading a false "on" until
that resting value is manually cleared (e.g. `input_number.set_value`
to 0, or it will reset naturally on the next HA restart) - a one-time
cleanup action for the user, not something this fix needed to handle
itself.

Status: **fixed, deployed live, and confirmed against a real timeout
event, all as of 2026-10-20.** `test_plan.md` Test 6.4 upgraded to PASS
accordingly (see below).

---

## EMHASS Rejects Optimisation Requests When `sensor.max_active_power_1` Goes Unavailable

Found live 2026-10-11, during the first real MPC stall since the new
45-minute Maximize-Self-Consumption fallback was deployed (see
roadmap.md - Improve EMHASS Optimisation Failure Handling). Root cause
chased down precisely, not inferred:

`sensor.max_active_power_1` ("Inverter 3622516.1 Max. aktiv effekt," kW)
went `unavailable` at **11:46:12 local** and stayed unavailable for the
rest of the day - last good value 10.6 (kW) at 10:00:00, no recovery
observed as of this writing. `emhass_scripts.yaml` feeds this single
entity into BOTH `inverter_ac_output_max` and `inverter_ac_input_max` in
every MPC/day-ahead payload (`max_inverter_power:
sensor.max_active_power_1`, `| float(0) * 1000`), so once the entity went
unavailable, both fields silently became `0` instead of erroring or
falling back to something sane.

Confirmed directly from `homeassistant.components.rest_command`'s own
log (not inferred): both `http://127.0.0.1:5000/action/naive-mpc-optim`
and `.../action/publish-data` have returned **HTTP 400 Bad Request**
every 15-minute cycle since ~11:51 local, logged with the full request
payload showing `inverter_ac_output_max: 0, inverter_ac_input_max: 0`
alongside otherwise-normal values (battery limits, prices, forecasts all
populated correctly). The EMHASS addon itself is not down (structured
400 response, not a connection error; `ha_get_app` shows `state:
started`, on `v0.18.5`).

**Corrected root cause (superseding the initial "isolated Modbus
register" read above)**: `sensor.max_active_power_1` is NOT part of the
`solaredge_modbus_multi` local Modbus integration that drives the rest of
this project (that integration - entry "SolarEdge", v4.0.6, HACS shows no
pending update - stayed `state: loaded` throughout, with
`sensor.solaredge_i1_ac_power`, `sensor.solaredge_m1_ac_power`,
`select.solaredge_i1_storage_command_mode`, and
`sensor.solaredge_b1_state_of_energy` all updating normally the whole
time, so this is **not** a Modbus Connectivity episode and the
2026-10-11 entry added to that roadmap.md section under that assumption
has been retracted). `sensor.max_active_power_1` instead belongs to a
completely separate, cloud-API-based custom integration -
`custom_components.solaredgeoptimizers` (HACS: "SolarEdge Optimizers
Data," AndrewTapp/solaredgeoptimizers, installed v2.5.0, which is also
HACS's latest - no pending update fixes this), config entry "SolarEdge
Site 3622516," which pulls per-optimizer/inverter data from SolarEdge's
monitoring portal rather than reading local Modbus registers. All 8
entities on that integration's one device (`current_average_1`,
`power_1`, `voltage_average_1`, `lifetime_energy_1`,
`last_measurement_1`, `child_count_1`, `status_1`, and
`max_active_power_1`) went `unavailable` within the same millisecond
(11:46:12.41xxxx local) - a whole-config-entry failure, not a
single-register glitch. HA's system log pins the cause precisely: this
config entry has been in `state: setup_retry` since **09:45:13 UTC
(11:45:13 local) - 59 seconds before the entities dropped** - with the
integration logging `SolarEdge Optimizers: Login check failed (status:
410)` on every retry, repeating every 10 minutes continuously through at
least 15:37 local with no recovery. HTTP 410 Gone (not a timeout or a
5xx) means the login endpoint itself is gone, not transiently
unreachable - the component's own log names it a "Dual API" that
maintains both a "SolarEdge One" and a "legacy" portal client
("Closed One and legacy API clients" on every failed retry), so this
looks like SolarEdge retiring the legacy portal login this custom
component depends on, not an update on the Home Assistant/Modbus side.
The user's initial suspicion that a SolarEdge Modbus integration update
caused this was checked directly and ruled out - that integration is
healthy, unchanged, and unaffected.

The 90-minute addon-auto-restart escalation (`EMHASS_addon_watchdog_auto_restart`)
fired right on schedule at 13:15:01 - and, as expected once the real
cause was understood, did not help: the 13:30 run failed with the
identical HTTP 400, since restarting the addon does nothing for a
different integration's cloud-login failure.

**Fixed 2026-10-11, same day as diagnosis**, by removing
`sensor.max_active_power_1` from EMHASS's critical path entirely rather
than hardening its fallback in place - the user's call once the root
cause (a separate, currently-broken cloud integration with no fix ETA)
was clear: this is a hardware nameplate rating, not a live measurement,
so it essentially never changes, and a static value loses nothing
meaningful in exchange for removing a fragile cloud dependency from
every MPC/day-ahead run.

Implementation: a new helper, `input_number.emhass_max_inverter_power`
(`emhass_helpers.yaml`, Layer 2, unit W, initial `10600` - the entity's
own last-reported value before it broke, min 5000/max 15000/step 100),
replaces `sensor.max_active_power_1` in both
`generate_emhass_energy_plan_mpc_pv` and `generate_emhass_energy_plan_dh_pv`
(`emhass_scripts.yaml`)'s `sensors.max_inverter_power` mapping. The
calculation line dropped its `* 1000` conversion (the old sensor read in
kW; the new helper is defined directly in W, consistent with every other
power helper in this project) and its fallback default changed from
`| float(0)` to `| float(10600)` as defense in depth.

Deliberately **not** reused: `input_number.ac_maxpower`
(`safety_and_watchdog_helpers.yaml`, currently 5000W) - checked directly
before implementing and ruled out. That helper is a distinct, pre-existing
Layer 4A decision threshold (`battery_forecast_control.yaml`'s `ac_lim`,
the Clipped Solar branch's `pv > ac_lim` comparison), deliberately set
well below the inverter's real 10.6kW AC capacity for its own purpose
(README.md's Clipped Solar section itself notes some ambiguity over
whether that threshold represents the inverter's AC limit or the site's
grid-export limit). Reusing it for EMHASS's `inverter_ac_output_max`/
`inverter_ac_input_max` would have silently capped the optimizer's
planning at under half the inverter's true throughput - a materially
wrong constraint, not just an imperfect fallback - so a new, dedicated
helper was created instead.

The `solaredgeoptimizers` integration itself is left unfixed - its
cloud-portal login still returns HTTP 410 as of this writing - but
nothing in this project's control path depends on it any longer, so
that's now optional cleanup (useful only if those 7 other entities on
the same device are wanted for something else) rather than a blocker.

Status: **fixed and confirmed live 2026-10-11** - the first scheduled
MPC cycle after the restart (16:15 local) succeeded:
`input_datetime.emhass_mpc_last_success` advanced from the stalled
11:30:16 to 16:15:24, the actual request payload (pulled from the
automation trace) shows `inverter_ac_output_max`/`inverter_ac_input_max:
10600` (not 0), no new HTTP 400 in the error log after the restart, and
`sensor.emhass_health`/`input_text.emhass_mpc_status` both read `OK`.
The 45-minute Maximize-Self-Consumption fallback (see roadmap.md)
remains in place as a second line of defense regardless.

---

## Write-Outcome Telemetry Is Readback-Based

Added 2026-10-06 as part of the Enhanced Diagnostics build-out (user
request: build the still-open items from roadmap.md - Enhanced
Diagnostics, full set in one pass, deriving port/write health from
existing signals rather than a new probe). `script.modbus_queue`
(`solaredge_modbusqueue.yaml`) now classifies every command as
`success`/`failure`/`unmatched_service` and feeds
`input_text.modbus_last_successful_write`/`_failed_write`/
`_failed_reason` and `input_number.modbus_write_success_count_total`/
`_today`, `_failure_count_total`/`_today`, `_consecutive_failures` (all
new Layer 1 helpers in `safety_and_watchdog_helpers.yaml`), plus
`modbus_queue_last_command_delta` for the six numeric write branches and
`modbus_queue_pending_count` for in-flight queue depth.

Important limitation, stated plainly rather than left implicit: this is
NOT based on catching an exception from the dispatched SolarEdge service
call. Home Assistant's `continue_on_error: true` (already on the
`choose:` block for the lock-release-safety reason documented below)
stops the script from aborting on a failed write, but does not expose a
success/fail signal to later template steps - there is no documented way
to read "did the last action error" back into a template after
`continue_on_error` catches it. Instead, `write_outcome` is derived by
snapshotting the target entity's value immediately before the `choose:`
dispatch and reading the same entity back ~2s after (right after the
existing pacing delay), comparing it to the value/option the command
should have produced. This is the same "derive from existing signals, no
new probe" approach used for the new port-health binary sensors in
`watchdog_sensors.yaml` (see entities.md - Layer 1).

Two concrete gaps this leaves, both known trade-offs rather than bugs:

1. If the `number`/`select` entities in `solaredge_modbus_multi` update
   optimistically on write (set their own state immediately, regardless
   of whether the underlying Modbus write actually landed on the wire),
   a real silent wire-level failure will read back as `success` here -
   this readback can only catch a failure that either (a) leaves the
   entity's state not matching what was commanded, or (b) is a flat
   dispatch error (`unmatched_service`, when `queue_service` matches no
   branch at all - this one is unambiguous, no readback involved).
   Whether these specific entities are optimistic or poll-confirmed has
   not been verified against the integration's source. Until it is,
   treat `modbus_write_failure_count_*` as a lower bound on real write
   failures, not a ground-truth count.
2. The readback happens only ~2s after dispatch (the script's existing
   pacing delay). If an entity's true state takes longer than that to
   reflect a real write (e.g. it needs the next poll cycle), this could
   register a transient `failure` that would have self-corrected - a
   false positive in the opposite direction from (1). Not observed so
   far; flagged as a design caveat, not an observed defect.

The command-delta sensor (`modbus_queue_last_command_delta`) only updates
on a `success` outcome for the numeric branches (storage command timeout,
charge/discharge limit, dynamic charge/discharge limit, site limit) -
mode-select and the negative-site-limit switch have no meaningful numeric
delta and don't touch it.

Deliberately NOT implemented: per-write skip tracking for charge/discharge
*limit* writes (as opposed to the mode-select command, which already has
a skip counter via `apply_effective_battery_control.yaml`'s Mode Command
Guard). No logic currently skips/deduplicates a limit write before it
reaches `script.modbus_queue` - building a counter for that would be
telemetry for behavior that doesn't exist yet. See roadmap.md - Write
Pressure (minimum write interval / delta threshold for limit writes is
still an undecided, separate item) and Enhanced Diagnostics.

Status: implemented in this project's docs 2026-10-06. Not yet confirmed
against the live file or any real write (success or failure) - no trace
or history evidence yet collected for `modbus_write_success_count_total`,
`modbus_write_failure_count_total`, or the derived port-health/write-
health/queue-backlog binary sensors in `watchdog_sensors.yaml`.

### Update 2026-10-07 - Gap 2 confirmed live, far worse than expected; fixed via async verification

The user deployed the 2026-10-06 design live. Verification the same day
found gap 2 above - the 2s readback happening before the entity has
actually settled - firing on roughly half of numeric writes, not a rare
edge case: in under 5 minutes of live operation, 3 of ~6 writes were
misclassified as `failure`. Two concrete examples, both real:
`number.solaredge_i1_storage_discharge_limit` commanded to `0`, read
back as `2` (its old value) at the 2s mark, classified `failure` -
confirmed via history it actually settled to `0` about 10 seconds later
(not a real failure, just slow). Same pattern on
`number.solaredge_i1_storage_charge_limit` (commanded `5000`, read `0`
at the 2s mark). `modbus_write_consecutive_failures` reached 2 (of the
3-failure threshold for `binary_sensor.solaredge_modbus_write_unhealthy`
- see entities.md for the naming note: the `unique_id` is
`solaredge_modbus_write_health`, but HA derives the real entity_id from
the friendly name, not the unique_id, so the two disagree)
within minutes, which would have been a false alarm if a third
misclassification had landed before a real success reset it. Gap 1
(optimistic-vs-poll-confirmed entities) was not what caused this - the
live evidence shows these entities genuinely wait for poll confirmation,
they just take longer than the 2s pacing delay to do it, consistent with
the 5s Modbus scan interval set during the Modbus Connectivity
investigation (see roadmap.md).

Root cause of why a longer delay couldn't just be added inline: `script.
modbus_queue` runs `mode: queued`, which serializes the script's own
execution - HA won't start the next queued run until the current one's
entire sequence finishes, regardless of the separate `modbus_busy`
lock. Any wait added directly in the main sequence (even after releasing
the lock) would still have delayed the start of the next queued command
by that same amount, reintroducing exactly the kind of write-pressure
problem roadmap.md's Write Pressure item is about.

Fixed 2026-10-07 by splitting the wait out of `script.modbus_queue`
entirely: a new script, `script.modbus_write_verify` (mode: queued, its
own independent queue), is started fire-and-forget
(`service: script.turn_on`, not a direct service call - the latter
blocks until the called script finishes, confirmed from a live trace of
`apply_effective_battery_control_to_solaredge_inverter` showing a ~4.7s
gap waiting on a child script run) with the dispatch's variables passed
via `script.turn_on`'s `variables:` data key. It then
`wait_template`s (reusing the same success-comparison logic, moved
out of a plain variable into the wait condition itself) for up to 8
more seconds (continue_on_timeout: true) - resolving as soon as the
real value lands rather than always waiting a fixed window, and only
calling it `failure` if the entity genuinely never reaches the expected
value within ~10s total from dispatch (2s pacing + up to 8s more).
`script.modbus_queue` itself no longer waits for or computes
write-outcome at all (except the one synchronous, no-wait-needed case:
an unmatched `queue_service`, classified immediately since there's
nothing to wait for) - its own runtime is back to what it was before
Enhanced Diagnostics, so queue throughput is unaffected.

One implementation detail flagged rather than silently assumed:
`script.turn_on`'s `variables:` data key passing initial values into a
script started this way is standard, documented Home Assistant
behavior, but hadn't been exercised anywhere else in this project before
now - worth confirming via `script.modbus_write_verify`'s own trace
after deployment (fields should show the real `queue_service`/
`write_target_entity`/etc., not blank) rather than assuming it worked
silently.

Status: implemented in project docs 2026-10-07. Superseded by the Update
below - the live redeploy did not actually wire this fix in.

### Update 2026-10-07 (second check) - redeploy left the fix unwired; old 2s classification is still what's running live

The user redeployed and reported it live. Re-verification the same day,
within minutes, found the exact same false-positive pattern as the
original 2026-10-06 incident: `input_text.modbus_last_failed_reason`
recorded "Entity number.solaredge_i1_storage_discharge_limit expected 0
but reads 289" immediately after a `set_storage_discharge_limit=0`
command - the entity's *old* value (289), read back too soon, same as
before.

Confirmed via a full detailed trace (`script.modbus_queue`, run_id
`ef46f005c6f908cf8f688293f04429f0`, 2026-10-07 11:00:07-11:00:25 UTC)
that the live sequence is still the *old*, pre-fix structure: `delay:
"00:00:02"` (sequence/16) immediately followed by a `variables:` step
computing `write_success`/`write_outcome` inline (sequence/17), then an
`if`/`else` on that outcome writing straight into the success/failure
telemetry helpers (sequence/18) - word-for-word the design this fix was
supposed to replace. There is no `service: script.turn_on` dispatch to
`script.modbus_write_verify` anywhere in the trace.

Separately confirmed `script.modbus_write_verify` itself: the entity
exists live (`state: off`, `mode: queued`, `max: 10` - matching what was
specified), but `ha_get_automation_traces` shows `trace_count: 0`,
`last_triggered: null` - it has never once been invoked - and
`ha_search` across all automations/scripts for `modbus_write_verify`
found exactly one match (the script's own name) with
`match_in_config: false` and `match_in_references: false` - nothing in
the live config calls it. It was added as a standalone, fully dead
script.

Net effect: the redeploy added the new `script.modbus_write_verify`
script as a live entity, but `script.modbus_queue`'s own sequence was
not updated to replace its old inline classification block with the
dispatch to it. Likely cause: the new script was pasted into the live
YAML without replacing the corresponding block in `script.modbus_queue`
- a partial copy of this fix's diff rather than the full one. This is a
deployment gap, not a design flaw - the design itself hasn't actually
been exercised live yet.

Status: still only genuinely implemented in project docs. The live
instance has the new `script.modbus_write_verify` script present but
unused, and `script.modbus_queue` is unchanged from before this fix -
needs a corrected redeploy of `solaredge_modbusqueue.yaml` (specifically
the end of `script.modbus_queue`'s sequence, replacing the inline
`write_success`/`write_outcome`/`if`/`else` block with the
`if target==none / else: script.turn_on script.modbus_write_verify`
dispatch) before this can be verified again.

### Update 2026-10-07 (third check) - corrected redeploy confirmed wired in correctly, via a real live write

The user ran a Home Assistant update/restart around 12:00 UTC, which
reloaded `solaredge_modbusqueue.yaml` again (`script.modbus_queue` and
`script.modbus_write_verify` both show a config reload at 12:12:51 UTC).
The next real write after that reload - `set_storage_discharge_limit`
dispatched 12:30:07 UTC - gives the first clean evidence of the fix
actually running as designed:

`script.modbus_queue` (run_id `355c5403720034a5f7c95416902a976c`)
dispatches `number.set_value` (`number.solaredge_i1_storage_discharge_
limit` -> 3300), waits the normal 2s pacing delay, then - unlike every
prior trace - its step right after the delay is `service: script.
turn_on` targeting `script.modbus_write_verify`, with a `child_id`
linking to that script's own run
(`774a4e2a80f6edea2955a009420ebaa0`), and its `service_data.variables`
show `queue_service`, `queue_value`, `write_target_entity`,
`write_expected_value`, and `write_before_value` all correctly
populated - confirming the previously-flagged-as-untested
`script.turn_on` variables-passing mechanism does work as expected.
`script.modbus_queue` itself then finished its own remaining steps
(lock release, counters) within ~13ms of that dispatch - it did not
block waiting on the child script, confirming the fire-and-forget
design is genuinely non-blocking live, not just in theory. A second,
simpler write in the same burst (`set_storage_charge_limit`, 0 -> 0, a
no-op) shows the same dispatch pattern and its own
`script.modbus_write_verify` run (`4bbf462a0118a298e914b8cb35433a36`)
resolved instantly (`wait.completed: true` immediately, since
before/expected were already equal) - classified `success`, counters
incremented correctly.

One residual wrinkle, not the systemic bug this fix targeted but worth
tracking: the discharge-limit write's own verify run
(`774a4e2a80f6edea2955a009420ebaa0`) timed out at the full 8s
(`wait.completed: false`) and classified `failure` - but
`number.solaredge_i1_storage_discharge_limit` actually did land on
3300, just ~2 more seconds after that timeout (history shows the real
state change at 12:30:35.903 UTC, vs. the verify script's timeout
firing at 12:30:33.695 UTC - about 28s after the original dispatch,
well past the design's ~10s total budget). This happened on the first
real write after an HA restart, immediately after the Modbus
integration itself was reconnecting, which is a plausible one-off
explanation (not yet confirmed) rather than evidence the 8s window is
generally too short - worth re-checking whether this recurs during
normal (non-restart) operation before concluding anything about the
window size itself.

Status: the core fix - splitting the wait out of `script.modbus_queue`
into `script.modbus_write_verify`, dispatched via `script.turn_on` with
variables - is now **confirmed wired in and working live** 2026-10-07,
with trace evidence of both the dispatch and the non-blocking behavior.
The classification logic itself produced one false `failure` in this
same verification window (discharge-limit write, likely restart-related
slow settle) - not yet enough data to say whether the 8s wait needs
widening; watch for a repeat under normal operating conditions.

---

## Entity ID Documentation Mismatches (Derived From Name/Alias, Not unique_id/id)

Found 2026-10-07 while verifying the Enhanced Diagnostics deployment
live. Three entity_ids documented in this project's docs turned out not
to exist - the entities are live and working, just under a different
entity_id than documented:

```text
documented                                    actual (live)
binary_sensor.solaredge_modbus_write_health   binary_sensor.solaredge_modbus_write_unhealthy
automation.reset_watchdog_daily_counters      automation.watchdog_reset_daily_diagnostic_counters
automation.reset_modbus_queue_daily_counters  automation.reset_modbus_queue_daily_telemetry_counters
```

Root cause, same in all three cases: Home Assistant derives a template
entity's entity_id by slugifying its `name:` field, and an automation's
entity_id by slugifying its `alias:` field - in both cases, NOT from the
`unique_id:` (template entities) or `id:` (automations) field, which
only identify the entity/automation for editing/registry purposes and
have no effect on the entity_id once assigned. The first case
(`write_health`/`write_unhealthy`) was introduced 2026-10-06: the
friendly name was deliberately written as "SolarEdge Modbus Write
Unhealthy" (to avoid the same on/off-naming ambiguity already flagged on
`binary_sensor.emhass_healthy` - see Cleanup section of roadmap.md),
without accounting for that name also becoming the entity_id. The second
case (`reset_watchdog_daily_counters`) was introduced the same day for
the same structural reason (the `id:` field was chosen to read
cleanly, independent of the alias). The third case
(`reset_modbus_queue_daily_counters`) predates this session entirely
(from the 2026-10-02 Add Queue Telemetry work) and was only found
incidentally while double-checking the other two - it was wrong in
entities.md and roadmap.md all along, just never actually looked up
live until now.

Fixed 2026-10-07: corrected all three references across entities.md,
roadmap.md, and this file to the real, live entity_ids. No YAML changed
- the entities themselves were never broken, only the documentation
pointing at them. General note for future Enhanced-Diagnostics-style
work: when a template entity's chosen friendly name needs to differ from
what the entity_id "should" read as (to dodge a naming-ambiguity trap
like the emhass_healthy one), the actual entity_id Home Assistant
assigns needs to be looked up live (e.g. via ha_search) rather than
assumed from the name used in the YAML - `unique_id`/`id:` do not
control it.

Status: docs corrected 2026-10-07. No live/functional impact - this was
purely an accuracy problem in the project's own documentation.

---

## Negative Price Curtailment - Site Limit Restore Gap

Found 2026-10-06, while checking how the negative-price curtailment
limit behaves during a real overnight event (7 negative/positive
price-sign flips, 2026-10-06 00:00-08:00 local). Reading
`solaredge_modbusqueue.yaml` directly: the `negative_site_limit_on`
handler writes `number.solaredge_i1_site_limit = 0` (only if it isn't
already 0) before turning the switch on, but the mirror-image
`negative_site_limit_off` handler only turns `switch.
solaredge_i1_negative_site_limit` off - it never writes the number back
to any other value. Confirmed live: the number was still reading "0" at
13:45 local, nearly 6 hours after the switch itself last went off
(08:00:16) and the price returned positive (08:00:02).

This exact gap is what `test_plan.md` Test 3.2 (Site Limit Enforcement)
had been waiting on since 2026-07-30 ("NOT VERIFIED") - previously an
untested assumption, now a confirmed asymmetry in the code.

**Empirically confirmed harmless the same day**, via real power-flow
arithmetic rather than guesswork: during the 2026-10-06 08:40-11:55
window, PV production exceeded house load by up to ~1.8kW while the
switch was "off" and the site limit number was still stuck at "0" the
whole time (statistics pulled for `sensor.solar_panel_production_w`,
`sensor.power_myhouse_load_no_var_loads`, `sensor.solaredge_b1_dc_power`
and `sensor.solaredge_m1_ac_power`). Whenever PV surplus exceeded what
the battery was absorbing by charging, the uncovered remainder reliably
showed up as positive (exporting) meter readings, matching the
PV-minus-load-minus-battery-charge arithmetic within ~15-70W each time:

```text
11:30  PV 1419W  load 253W  batt charging 515W  -> meter +645W
11:35  PV 1804W  load 266W  batt charging 578W  -> meter +941W
11:40  PV 1967W  load 270W  batt charging 585W  -> meter +1091W
```

This directly demonstrates that on this installation,
`number.solaredge_i1_site_limit`'s value is not enforced independently
of the switch - `switch.solaredge_i1_negative_site_limit` is the real
enable/disable gate for the site-limit feature, and a stale "0"
underneath an "off" switch has no effect on actual export. The
enforcement-while-ON path (does `site_limit=0` actually suppress export
that would otherwise happen) remains unobserved, since every
negative-price episode so far has occurred overnight with zero PV - see
roadmap.md - Negative Price Curtailment for that still-open item.

**Closed 2026-10-06 via redesign, not the originally-proposed patch.**
Rather than teaching `negative_site_limit_off` to restore the number to a
chosen "normal" value (the cleanup first proposed here, which the user
declined - see roadmap.md), the user proposed a cleaner architecture that
eliminates the asymmetry structurally: stop toggling
`switch.solaredge_i1_negative_site_limit` per price change and leave it
permanently ON, and instead vary `number.solaredge_i1_site_limit` itself
between 0 (curtailed) and 1,000,000 W - the number's own max (normal/
indifferent export, chosen to reproduce the exact unconstrained-export
behaviour already confirmed live above). Since the number is now written
on every transition, lifetime, there is nothing left to "forget" to
restore.

Implemented 2026-10-06:
- `solaredge_modbusqueue.yaml`: replaced the combined
  `negative_site_limit_on` handler (which wrote the number AND the switch)
  with a new standalone `set_site_limit` handler (`number.set_value` only).
  The switch on/off handlers remain, now defensive/manual-only.
- `batterycontrol_scripts.yaml`: added `set_site_limit_curtailed_script`
  (value 0) and `set_site_limit_normal_script` (value 1000000), each
  calling `script.modbus_queue` with `queue_service: set_site_limit`.
- `negative_price_curtailment.yaml`: both `choose:` branch conditions now
  key off `number.solaredge_i1_site_limit`'s current value (`!= 0` /
  `!= 1000000`) instead of the switch's state - this is the
  compare-before-write guard that avoids redundant Modbus writes on the
  30-second periodic trigger. The negative branch additionally asserts the
  switch ON defensively (`if` the switch is "off", turn it on) before
  writing the curtailed value, so the mechanism self-heals if the switch
  is ever found off - including the one-time transition away from the old
  design, where the switch was last left "off" at 08:00:16 on 2026-10-06.

Operational note: because the switch was off when this redesign deployed,
it will not flip back ON until either (a) the next negative-price episode
triggers the defensive re-assert, or (b) someone turns it on manually in
the interim. Until then the switch being "off" is harmless on its own
(per the empirical finding above) - the number alone now carries the
curtailment logic once the switch is confirmed on.

### Update 2026-10-19 - New mechanism confirmed live against a real negative-price episode; switch self-heal and curtail/restore cycling both verified

The switch had sat "off" for a full week with no corrective mechanism
ever firing (confirmed via history: zero negative-price episodes
2026-10-14 through 2026-10-19, so the automation's defensive re-assert -
the only thing that ever touches the switch - had no occasion to run).
On the night of 2026-10-13, a user-anticipated negative-price window
arrived with zero PV surplus (safe test conditions: if the mechanism
failed, there was no export to curtail anyway). It did not fail.

Evidence, all local time 2026-10-13: `sensor.total_export_price` went
negative three separate times as it oscillated in a narrow band around
zero (-0.00014 to -0.00028 SEK/kWh, 22:22-22:52) before settling
positive. On the first crossing, `switch.solaredge_i1_negative_site_limit`
flipped `off -> on` at 22:34:13.07 - the defensive re-assert firing
exactly as designed, on the very first real negative-price tick since
the redesign. Across the three oscillations, `number.
solaredge_i1_site_limit` correctly tracked each crossing: `1000000 -> 0`
at 22:34:11.90, `0 -> 1000000` at 22:46:25.96, `0` again at 22:49:16.64,
`1000000` again at 22:51:44.42, `0` again at 22:54:50.22, and finally
back to `1000000` at 23:02:40.65 once price stayed positive -
`input_boolean.negative_price_active`/`grid_export_blocked` tracked the
same three cycles in lockstep. Three full curtail/restore cycles in
about 20 minutes, driven by price noise within a ~0.0003 SEK/kWh band
around exactly zero rather than a real trend - not a malfunction, but
worth noting as a minor write-pressure footnote given this project's
general SD-card-wear sensitivity: each cycle is 1-2 Modbus writes, so a
flickering-around-zero night costs a handful of extra writes, not a
meaningful number.

Durability: the switch has stayed `on` through every one of the
twice-daily brief Modbus "unavailable" blips since (2026-10-14 through
2026-10-19, 6 days, 12+ blips observed), confirming the permanently-on
design holds up over time, not just at the moment of the triggering
event.

This closes roadmap.md's items 2 and 3 for this issue (the new mechanism
observed live through a real cycle; the switch's one-time self-heal from
its post-redesign "off" state). Item 1 - a negative-price window
coinciding with real PV surplus, to directly observe export actually get
suppressed rather than inferring it from command correctness alone -
remains open; 2026-10-13's episode was deliberately chosen for zero PV
and still has not been observed. See test_plan.md Test 3.2/S.3 for the
test-plan-side update.

Status: **closed and confirmed live 2026-10-19** for the switch
self-heal and the curtail/restore mechanism itself. The one remaining
unobserved scenario (negative price + real PV surplus) is tracked
separately in roadmap.md, not blocking this issue's closure.

---

## EMHASS Outage Fallback Guard

Found 2026-09-22: a power outage left the EMHASS addon stuck rebuilding
itself on every start (see Root Cause below) for the rest of the day -
`input_datetime.emhass_mpc_last_success` was frozen at 14:16 while
`emhass_battery_forecast_control` (Layer 4A) kept running every 15
minutes regardless, since its only prior guard was the SOC sensor
(`battery_forecast_control.yaml`'s `soc`/`batt_fc`/`grid_fc`/
`price_import` variables all read *other* EMHASS-dependent entities that
were themselves unaffected by that guard).

Live history for the outage window (2026-09-22 16:00-22:50) confirmed
real, repeated harm, not just a theoretical gap:
`input_select.emhass_requested_storage_mode` alternated
`charge_from_solar_and_grid` <-> `maximize_self_consumption` on almost
every single 15-minute tick for over 6 hours (`input_number.
emhass_requested_discharge_limit` swinging 0 <-> 3300 in lockstep), and
`sensor.solaredge_b1_state_of_energy` sawtoothed between roughly 19% and
34% the entire time - a real charge/discharge cycle every ~15-20 minutes,
not a one-off. `sensor.total_import_price` was *also* unavailable for
the whole window (only recovering post-restart at 3.4-3.76 SEK/kWh,
confirming prices genuinely were high, matching the user's report),
which is what made this a real-money problem and not just wasted battery
cycles.

Root cause, two independent gaps in `battery_forecast_control.yaml`:

1. `batt_fc`/`grid_fc` are read from `sensor.mpc_pv_batt_power`/
   `sensor.mpc_pv_grid_power` - EMHASS-published entities that don't
   exist at all while EMHASS is down (confirmed via entity search: not
   merely `unavailable`, genuinely unregistered). Their `| float(0)`
   fallback reads as "battery forecast flat, grid forecast flat" -
   satisfying several branches as if EMHASS had actually forecast a
   neutral plan.
2. The BELOW-SOC "PRICE-BASED FALLBACK" branch gates a real grid-charge
   command on `price_import <= last_average_chargingprice * 1.2`, where
   `price_import` has the same `| float(0)` fallback on
   `sensor.total_import_price`. A missing price sensor therefore read as
   "price is free" (0 satisfies almost any positive threshold), the
   opposite of the safe assumption - and with `batt_fc` pinned to 0 by
   gap 1, this was exactly the branch reached at night, on every cycle
   where SOC read as `below` its frozen target. That combination -
   `pv==0`, `batt_fc==0` (fake), `price_import==0` (fake, always <=
   threshold) - fired `charge_from_solar_and_grid` repeatedly; each
   charge then pushed SOC out of `below` for a tick or two, landing on
   `maximize_self_consumption` with the discharge limit reopened (frozen
   at its last real value, 3300W), which drained it back down - the
   observed oscillation.

Fixed in `battery_forecast_control.yaml`, two changes:

1. Added a third automation-level `condition:` (alongside "Remote
   Control" and the SOC-unavailable guard) requiring `sensor.
   emhass_health` to not be `MPC stalled` or `MPC problem`. This is the
   primary fix: it skips the entire run whenever EMHASS's own published
   forecast is stale/failed, leaving the requested mode and charge/
   discharge limits frozen at their last good value instead of
   recomputing a new decision from fabricated zeros every 15 minutes.
   `RUNNING` is deliberately still allowed through, since the previous
   successful forecast is still valid during an in-flight EMHASS run.
2. Independently hardened the PRICE-BASED FALLBACK branch itself: added
   `states('sensor.total_import_price') not in ['unavailable',
   'unknown']` to its condition, so a missing price sensor now falls
   through to the existing safe catch-all (`solar_power_only`, no grid
   charge) instead of being misread as "free". This covers the narrower
   case where the price feed alone fails (e.g. a Nordpool/EPEX outage)
   while EMHASS itself stays healthy - fix 1 above wouldn't catch that
   case, since `sensor.emhass_health` doesn't track the price sensor at
   all.

Root cause (why EMHASS was down for 8+ hours, separate from the guard
gap above): the EMHASS addon rebuilds its Python environment on every
start (`Building emhass @ file:///app`) rather than running a pre-built
image, and that build needs live DNS/network access to pypi.org /
files.pythonhosted.org. The outage's DNS wasn't back up yet by the time
HA (and the addon's container) restarted, so every build attempt failed
identically; once failed, the addon sat in Supervisor state `error` and
did not retry itself. A plain restart once DNS had recovered fixed it
immediately (v0.18.3, no code change involved).

Supervisor's own addon watchdog (`watchdog: true`) and boot-on-start
(`boot: auto`) were already both enabled on the EMHASS addon - that
watchdog restarts an addon that crashes *after* having started, but
doesn't retry one that keeps failing its own build step and never
reaches "started". To close that gap, added a new automation, `EMHASS
Watchdog - Addon Auto-Restart` (`watchdog_automations.yaml`): if MPC
has been stalled over 90 minutes (double the existing 45-minute
notify-only threshold, since a restart is more disruptive than a
notification), calls `hassio.addon_restart` on the EMHASS addon
(slug `5b918bf2_emhass`) and notifies either way, at most once per hour
(new `input_datetime.emhass_addon_last_restart_attempt` helper in
`safety_and_watchdog_helpers.yaml`) to avoid restart-looping if the
underlying cause (e.g. no network) hasn't actually cleared. This would
have retried the addon automatically as soon as DNS came back tonight,
rather than requiring a manual restart the next time someone noticed.

Status: deployed live 2026-09-23 (all three changed files -
`battery_forecast_control.yaml`, `watchdog_automations.yaml`,
`safety_and_watchdog_helpers.yaml` - copied to the live config and
reloaded). Structurally **confirmed live** the same day via
`ha_get_automation_traces` on `automation.
emhass_forecast_driven_battery_control` (run_id
`ff5412076e1ede4c9737d80949f37fef`, 2026-09-23T19:45:00 UTC): the
`condition_results` show exactly the 3 expected top-level conditions -
Remote Control, the pre-existing SOC-unavailable guard, and the new
`sensor.emhass_health` guard - all evaluating, `script_execution:
"finished"`. Also confirmed directly: `automation.
emhass_watchdog_addon_auto_restart` exists and is `state: "on"`, and
`input_datetime.emhass_addon_last_restart_attempt` exists (still its
default never-triggered value). `sensor.emhass_health` currently reads
`OK`.

This confirms the guard is wired in correctly and evaluating on every
cycle, but not yet exercised in anger - no EMHASS stall or addon
`error` state has occurred since deployment to confirm a run is
actually *skipped* under the new health condition, or that the addon
auto-restart automation actually fires and succeeds. Both still await
a real future occurrence, same as noted below for the SOC-unavailable
guard.

Update 2026-09-24 (near-miss, inconclusive): `sensor.emhass_health`
briefly went `unavailable` (21:15:48, 21:24:50 local) then `MPC problem`
(21:27:42-21:32:51) the evening of deployment - but this lines up with
the deployment's own automations reload, not a real EMHASS failure:
`automation.emhass_forecast_driven_battery_control`'s own entity
flickered `unavailable` -> `on` within the same second (21:24:46-47
local), and its usually-clockwork 15-minute logbook trigger history has
no "triggered by time pattern" entry at all for the 21:30 tick that
fell inside the bad-health window (present at every other tick before
and after) - consistent with the reload's trigger re-registration
missing that one boundary, not with the new guard evaluating and
skipping. No clean before/after limit-write pair exists to confirm the
guard actually blocked that tick. Still an open verification, not a
false alarm - just not usable evidence either way.

---

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

Status: implemented in this project's docs 2026-09-20. Deployment
wasn't directly confirmed by a file diff (`ha_config_get_script` 404s
on this package-defined script, same known limitation as YAML
automations), but is now **behaviorally confirmed live** as of
2026-09-24: `input_boolean.modbus_busy` history was checked across
~4.5 days since the fix (2026-09-20 02:00 - 2026-09-22 08:15, then
2026-09-23 15:30 - 2026-09-24 11:00; ~750 on/off cycles sampled, real
Modbus write/queue traffic throughout including the 2026-09-22 power-
outage day) and found **zero** episodes over 60s - none of the
~30-minute stuck-lock pattern from the original incident. This window
also contains genuine, ongoing connectivity noise to correlate against
(a "Received unexpected response with Transaction ID" mismatch at
2026-09-24 10:30:35 and two "Coordinator has timed out 3 times in a
row" warnings at 09:46:38/10:16:34) - `modbus_busy` toggled cleanly
(released in 18-33s) around all three, exactly the behavior the fix is
supposed to produce, where the old code would have left it stuck for
~30 minutes. Traces for `script.modbus_queue` only retain the last 5
runs, so no historical trace directly shows `continue_on_error`
catching a real failure - this is circumstantial (absence of the old
symptom under continued real error conditions), not a smoking-gun
trace, but it's a much stronger signal than "not yet deployed."

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

Status: deployed and **confirmed live** 2026-09-24, via a real ~2h13m
SOC-unavailable window the guard actually caught: `sensor.
solaredge_b1_state_of_energy` was `unavailable` 2026-09-22 14:22:15-
16:34:52 local (found while reviewing history for other unavailable
episodes this fix era - separate from, and 2 days before, the EMHASS-
addon outage documented above). `input_select.emhass_requested_
storage_mode` and both `input_number.emhass_requested_*_limit` helpers
were frozen solid at their 14:15:00 values (`charge_from_solar_and_
grid`, discharge=0, charge=250.0) across all 9 fifteen-minute ticks
inside the outage (14:30 through 16:30) - no trace history survives
from that far back, but the entity history is unambiguous: the very
first change of any kind happened at 16:45:01.785, the first tick after
the sensor recovered at 16:34:52. Same evidence class as the original
2026-09-16/17 31-hour outage this fix was written for, just shorter and
with a cleaner before/after (a single frozen tuple of values for 2+
hours, vs. one-off values before and after). No further verification
needed for the core skip behavior; still open is whether a *different*
fallback branch (one that isn't conveniently safe) would also be
skipped correctly - not exercised by either observed outage so far.

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

Update 2026-09-24 (incomplete-swap follow-up, found during the YAML
header/hygiene audit): `calculate_effective_battery_control`'s
**trigger** `entity_id` list in `safety_limits_and_override.yaml` still
named the dead `binary_sensor.ev_charging_on` - only the action logic's
`ev_charging` variable was swapped to `binary_sensor.ev_charging_active`
back on 2026-09-15, not the trigger list itself. Practical effect: the
automation wouldn't state-trigger promptly on a real EV-charging state
change (it also has a `time_pattern` trigger as a backstop, so a full
miss was unlikely, but a real-time reaction to `ev_charging_active`
flipping could have been delayed to the next scheduled tick instead of
firing immediately).

Fixed 2026-09-25: swapped the trigger list's `binary_sensor.
ev_charging_on` to `binary_sensor.ev_charging_active` in
`safety_limits_and_override.yaml`, matching the action-logic variable
fixed 2026-09-15. No other change to this automation.

Status: implemented in this project's docs 2026-09-25. Not yet
confirmed against the live file or a real EV session.

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

Status: deployed, confirmed 2026-09-04 (live file matched verbatim),
and now **runtime-confirmed** 2026-09-24: the EMHASS addon's own log
(`Passed runtime parameters` lines from live MPC runs, 2026-09-24
10:30-11:00) shows `'battery_minimum_state_of_charge': 0.2` and
`'battery_maximum_state_of_charge': 0.9` - an exact match to the live
`input_number.minimum_state_of_charge` (20.0) / `maximum_state_of_charge`
(90.0) helper values read at the same time. The template hookup is
confirmed working end-to-end, not just present in the script file.

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
it's currently active. A `mode_command_skipped_count_total`/`_today`
counter (Enhanced Diagnostics, added 2026-10-06 in
`safety_and_watchdog_helpers.yaml`) now also counts every tick the guard
actually skips, incremented from the `else:` branch of the same `if:
not mode_guard_active` in `apply_effective_battery_control.yaml`.

Status: deployed and confirmed working (test_plan.md Test 7.6)
2026-08-27. Modbus error counts look pre-existing rather than
guard-caused, but before/after impact on the overall error rate hasn't
been quantified. The new skip counter (2026-10-06) is implemented in
project docs only, not yet confirmed against live history.

### Update 2026-10-22 - third guard precondition (configured default mode) confirmed live, with user's explicit approval for the live intervention

The one precondition never yet exercised live - the guard staying
inactive when the inverter's configured default mode disagrees with
Maximize Self Consumption, even though effective mode and live command
mode both still say MSC - needed someone to deliberately change the
default mode on the live inverter, since 30 days of history (checked
2026-10-08) had shown it never once deviating from MSC on its own. The
user did so with explicit approval, changing `select.solaredge_i1_storage_
default_mode` to "Solar Power Only (Off)" via the entity's real control.

Unexpected but useful: the live `select.solaredge_i1_storage_command_mode`
changed to the same value at the same instant (10:08:51.52 local, 2ms
apart) - on this installation, writing the configured default mode
evidently also lands on the live command register, not just the
fallback. `sensor.battery_mode_command_guard` flipped `guarded -> active`
immediately. The next natural automation trigger (`effective_discharge_
limit` ticking at 10:15:01, trace `9d6c21d44d77ee8e0aeb55ccecfb67da`)
confirmed `mode_guard_active: false` and ran the full command sequence -
`set_value_storage_command_timeout15m`, `maximize_self_consumption_
script`, then both limit scripts - correctly resending the mode command
that a MSC-only check on effective mode and live command mode alone
would have skipped. `select.solaredge_i1_storage_command_mode` landed
back on "Maximize Self Consumption" ~69s later, and `mode_command_
skipped_count_total`/`_today` stayed flat throughout, confirming the
skip counter correctly did not fire while the guard was inactive.

Status: **all three guard preconditions now confirmed live** -
test_plan.md Test 7.6 upgraded to PASS.