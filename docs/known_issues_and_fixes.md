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

(None currently open requiring code changes. The Negative Price
Curtailment - Site Limit Restore Gap found 2026-10-06 is a confirmed
real asymmetry (the old design's deactivation path never restored
`number.solaredge_i1_site_limit`) that was empirically verified the same
day to have no live impact, and was then closed the same day by a
structural redesign rather than a patch - see below. The branch-coverage gap found 2026-09-10 was fixed 2026-09-15
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

# Fixed Issues

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

Status: closed via redesign 2026-10-06. Deployed to project docs; not yet
observed live through a real negative-price cycle under the new
mechanism - see test_plan.md Test 3.2/S.3.

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