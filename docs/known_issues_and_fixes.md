# Known Issues and Fixes

Durable log of bugs found and fixed - root cause, fix, deployment/
verification status. Companion to `roadmap.md` (open design/cleanup
items, not bugs). Most recent first within each entry. This session has
**read-only** access to the live Home Assistant instance - every fix
below is implemented in these project docs only until the user deploys
it to their live config; "confirmed live" means the user has deployed
it and live data has verified it working, not just that the doc change
is correct.

This file was compacted 2026-11-20 to cut narrative/forensic detail
(exact timestamps, discarded wrong hypotheses, raw log excerpts) while
keeping every root cause, fix, and current status. If a past
conversation referenced detail that's no longer here, it was the
process of getting to the conclusions below, not the conclusions
themselves.

Open issues, in brief (full detail below): the ML predict/MPC collision
fix (2026-11-20, forecast had gone stale) is confirmed working, but the
ML load forecast's 6-9 kW blocks were a stale model trained on the
with-EV sensor (refit 2026-11-21, forecast now plausible; model quality
weak, R2 0.12; weekly fit has no success tracking yet); training sensor
also gated against Easee stalls/meter freezes (2026-11-21);
the MPC/DH status-stuck-at-
RUNNING and MPC/DH collision fix (2026-11-20) is deployed live but not
yet verified against a real event;
the compound EMHASS-outage
incident from 2026-11-19/20 is closed (moved to Fixed Issues) except
its `batterycontrol_sensors.yaml` date-filter fix, still unconfirmed
deployed;
the DST
fall-back timestamp collision is root-caused but not fixed; PV trickle
readings bypassing no-PV branch logic is fixed and deployed (confirmed
2026-11-20), not yet verified against a real trickle night;
High SOC Hold's core charge-limit-bypass mechanism is still unsolved
(its Modbus-reconnect register-reset sub-issue now has a 5000/5000
re-apply trigger, 2026-11-20, deployment unconfirmed); the Remote Control
command timeout fallback (inverter silently reverting a non-MSC mode to
MSC after 900s) is fixed and deployed (3600s timeout + 1800s keepalive,
confirmed 2026-11-20), not yet verified against a non-MSC period;
two SOC/critical-recovery register-write issues remain open.

---

# Open Issues

## ML Load Forecast Predict Fails When It Collides With MPC - Forecast Goes Stale

Found 2026-11-20 23:35 while starting the ML-forecast accuracy review.
`sensor.p_load_forecast_ml_pv` had not been written since 06:00 (first
forecast slot 06:15), so MPC/DH were planning on a 16h-old load
forecast. `script.run_mlforecast_predict_load` succeeded at 00:00 and
06:00 but failed at 12:00 and 18:00 within ~1s with an aiohttp
`Client error occurred when calling resource
"http://192.168.10.3:5000/action/forecast-model-predict"` - a
connection-level failure (an HTTP error would log a status code).

**Likely cause (strongly indicated, not confirmed from the EMHASS side -
the add-on log only retains ~2h):** predict fired at hh:00:02, the same
second as the 15-min MPC run. Until ~06:15 2026-11-20 MPC was failing
fast with HTTP 400, so it never loaded EMHASS at 00:00/06:00; from 12:00
on, every predict landed on a real MPC run and the connection was
dropped. A REST sensor polling the same 192.168.10.3:5000 logged
"Server disconnected"/"Cannot connect" 21 times 02:00-22:00, consistent
with the same contention. The weekly fit (Sun 19:00) had the same
exposure, and the predict's 5-min post-restart delay collided with
MPC/DH's identical 5-min delay.

Also noticed, not yet analysed: the stale 06:00 forecast averaged
~1350W (peaks 9.3kW, first slot 4.9kW) vs. ~465W mean actual load in the
2026-11-03 comparison - possibly an inflated forecast; re-check once a
fresh one exists (see ML Load Forecast entry below).

**Fix (2026-11-20):** `emhass_automations.yaml` - predict trigger now
`hours: "/6", minutes: "7"`, post-restart delay 5 -> 7 min; fit moved
19:00 -> 19:07. `emhass_restcommand.yaml` - fit/predict/legacy
publish_data URLs switched 192.168.10.3:5000 -> 127.0.0.1:5000 for
consistency with the Siegen commands (loopback avoids dependence on the
host's LAN IP; InfluxDB host in the payloads stays 192.168.10.3, since
EMHASS resolves it from inside its own container).

**Status: confirmed fixed live 2026-11-21** - the 00:07 and 06:07
predicts ran clean and `sensor.p_load_forecast_ml_pv` refreshed each
time.

**But the fresh forecast is badly wrong** - tracked in the ML Load
Forecast entry below (Update 2026-11-21): 7-9 kW forecast for
00:15-06:15 on 11-21 vs. a typical ~200-300 W night, traced to EV
charging leaking into the training sensor.

---

## EMHASS Status Stuck at RUNNING After a Failed Run; MPC and Day-Ahead Can Collide

Found 2026-11-20 21:12 ("ran DH, new plan published and Optimal, but
the traffic light still says running"). The dashboard traffic light is
`sensor.emhass_health`, which reflects **MPC only** - day-ahead itself
was fine (`emhass_dayahead_status` = OK, last success 21:10:40).

**Root cause:** the 21:00 MPC run set `input_text.emhass_mpc_status` =
RUNNING, then `script.generate_emhass_energy_plan_mpc_pv` raised
`Client error ... http://127.0.0.1:5000/action/publish-data`. The
unhandled error aborted the automation before its OK/FAILED `choose`,
so the status stayed RUNNING until the next successful run. Same
exposure in the day-ahead automation. Cause of the publish-data client
error itself not investigated (EMHASS add-on log around 21:00:15).

**Related gap:** both automations/scripts are `mode: single` (no
self-overlap), but nothing prevented MPC and DH running concurrently
against the shared EMHASS instance - they collide every night (both
trigger at 00:00) and after every HA restart (both delay 5 min).

**Fix (2026-11-20, `emhass_automations.yaml`):**
- `continue_on_error: true` on both script calls, so a failure falls
  through to the FAILED branch.
- MPC: after the restart delay, a template condition skips the run if
  `state_attr('script.generate_emhass_energy_plan_dh_pv','current') > 0`.
- DH: after the restart delay, `wait_template` until the MPC script's
  `current` == 0, timeout 2 min, `continue_on_timeout: true` (DH runs
  once a day, so it waits rather than skips; a hung MPC can't block it).
- Lock is the script `current` attribute, deliberately not the
  `*_status` input_texts, which a failed run can leave stuck at RUNNING.

**Status: deployed live 2026-11-20 21:20. (b) verified 2026-11-21:
00:00 DH trace waited 7.5s on the in-flight MPC run, then ran and
wrote OK.** Still to verify: (a) and (c). Verify:
(a) next publish-data failure -> trace continues past the script step
and writes FAILED (if it still aborts there, `continue_on_error` doesn't
cover this error type - fall back to writing FAILED before the script
call); (b) 00:00 DH trace shows the `wait_template` step; (c) a manual
DH run just before a 15-min mark -> MPC trace stops at the new
condition.

---

## Remote Control Command Timeout Fallback - Non-MSC Mode Silently Reverts to Maximize Self Consumption

Found 2026-11-20 analysing 10 days of drift-watchdog alerts (298 ticks).
The single largest drift source, separate from the reconnect resets in
the High SOC Hold entry (mechanism #3): the intended mode was e.g.
Charge from Solar Power and Grid, the inverter ran Maximize Self
Consumption. Episodes: 2026-11-12 03:50-04:00, 11-13 01:05-03:50 (3
cycles), 11-18 01:15-03:15, and **11-19 06:32-21:55** (~180 ticks; the
EMHASS 401 outage froze the `effective_*` values all day, so ~15h of MSC
instead of the intended grid charging).

**Root cause:** the inverter runs a Remote Control command mode only
for the Remote Control Command Timeout (register E00B, was 900s via
`script.set_value_storage_command_timeout15m`); when it expires without
a new command it reverts to Storage Charge/Discharge Default Mode (MSC
on this install). `apply_effective_battery_control` only writes when an
`effective_*` helper changes, so any non-MSC mode held unchanged >15 min
reverted. Confirmed: every revert landed exactly 15:00 after the last
apply run (e.g. 06:45:48 -> 07:00:48, 16:45:48 -> 17:00:48, 21:30:53 ->
21:45:53 on 11-19; 01:15:54 -> 01:30:54, 02:15:59 -> 02:30:59 on 11-13;
apply times back-calculated for the integration's ~70s poll lag). MSC
itself never shows this drift (fallback == intended).

**Verified behaviour (live data + SolarEdge Storage Control technical
note):** (a) a new command takes effect immediately and replaces the
running one - commands do not stack (11-19 22:02:58 and 11-13 04:02:14:
battery DC power switched in the same poll, 8 and 5 min before the
previous command's timeout would have expired); (b) re-sending restarts
the timeout (SolarEdge: "renew the timeout with each write cycle");
(c) charge/discharge limits do NOT revert on timeout (173W held 9.5h
after the 07:00:48 mode revert on 11-19) - only Modbus reconnects reset
them; (d) timeout range is 0-86400s, 900s was only the vendor example.
Not separable from the data: whether the E00B (timeout) or E00D (mode)
write restarts the timer - both are always sent together.

**Fix (2026-11-20, docs only):** timeout raised 900s -> **3600s**
(`batterycontrol_scripts.yaml`, script id kept as `..._timeout15m` so
references don't break); new `solaredge_command_mode_keepalive`
automation (`apply_effective_battery_control.yaml`) re-sends timeout +
mode every **1800s** while a non-MSC mode is intended - limits are not
re-sent. 3600s is the deliberate fail-safe window: if HA/Modbus dies the
inverter stays in the last commanded mode at most 1h before falling back
to MSC.

**Status: deployed, confirmed 2026-11-20 21:50** (`number.solaredge_i1_
storage_command_timeout` = 3600; `automation.solaredge_command_mode_
keepalive` exists, not yet triggered - inverter has stayed in MSC).
**Not yet verified:** no mode-drift ticks during a non-MSC period longer
than 1h, and keepalive `last_triggered` advancing every 30 min in one.

---

## Date-String Filter No-Op in Dynamic Storage Limit / SOC Forecast Templates

Split out 2026-11-20 from the compound EMHASS-outage incident of
2026-11-19/20 (bug #3 there); the other four bugs from that incident
are closed and now live under Fixed Issues.

**Bug:** date-string-comparison no-op in `batterycontrol_sensors.yaml`.
Three templates (Dynamic Storage Charge Limit, Dynamic Storage Discharge
Limit, SOC Batt Forecast Smooth) filtered forecasts with `selectattr
('date', '>=', now() | string)`. `now() | string` renders a
space-separated format that never lexicographically matches the
forecast's ISO `T`-separated dates, so the "future only" filter was a
permanent no-op for all three - they always blended the array's literal
first 3 elements regardless of staleness, which (combined with bug #1's
outage) pinned the discharge-limit formula to a stale, wrong blend all
day. Fixed by changing to `now().isoformat()`, matching the already-
correct pattern in `update_soc_target_from_next_forecast`. **Status:
implemented in project docs, not yet confirmed deployed/verified live.**

---

## ML Load Forecast Diverges Sharply From Actual Load - No Better Than a Flat-Average Guess

Found 2026-11-03, checking whether EMHASS's ML load forecast is
reasonable against actual load. Compared the last 12 forecast snapshots
(00:00/06:00/12:00/18:00 cadence, `sensor.p_load_forecast_ml_pv`) against
actual `sensor.power_myhouse_load_no_var_loads_no_ev` over 2026-11-01
through 11-03 (973 matched forecast/actual pairs, 15min-23.75h lead
times).

**Result: the forecast is no better than guessing the period average.**
MAE 324W (~70% of mean actual 465W), Pearson correlation with actual
0.054 (essentially none), negligible net bias (+23W). A flat-average
baseline (MAE 256W) and "same time yesterday" (MAE 282W) both beat the
model; only naive persistence (MAE 342W) is worse. Error is roughly flat
across lead times (0-60min MAE 263W vs. 720-1440min MAE 348W) - not a
staleness artifact, consistent with a weekly-fit KNN pattern-matcher
(`KNeighborsRegressor`, 30-day history, weekly refit) that doesn't
incorporate recent behavior. One concrete example: the model forecast a
sustained 18:00-20:00 evening bump but missed the real timing by 1-2.5
hours.

Open, unanswered questions: whether this 3-day window was unusually hard
(daily mean load nearly doubled day to day, which no backward-looking
model could anticipate); whether `num_lags`/`historic_days_to_retrieve`/
`KNeighborsRegressor` are well-tuned for this household; what EMHASS's
own `perform_backtest` reported (not visible from Home Assistant).
Matters because `load_power_forecast` feeds directly into EMHASS's
MPC/day-ahead solver - a weak forecast here plausibly degrades every
optimization plan, though the impact isn't quantified.

**Update 2026-11-21 - likely root cause found: EV charging leaks into
the training/target sensor.** The first fresh forecast after the
predict-collision fix (00:07 11-21) predicts 7-9 kW hourly means for
00:15-06:15, against an actual overnight load of ~150-300 W on normal
nights (10-day hourly stats). The model's target,
`sensor.power_myhouse_load_no_var_loads_no_ev`, is computed in
`emhass_sensors.yaml` as house load minus `sensor.ev_charging_power_w`,
clamped at 0. On the EV-charging night of 2026-11-16 the two inputs were
clearly out of phase (hourly means, house load / EV power / resulting
"no EV"): 01:00 4967 / 0 / **4967**; 02:00 7873 / 2620 / **5253**;
04:00 132 / 2137 / **0**; 05:00 0 / 2947 / **0**; 06:00 4601 / 1806 /
**3566**. I.e. EV draw is sometimes not subtracted at all (EV sensor
reading 0 while the car charges) and sometimes subtracted when the
house load doesn't contain it, driving the result to the 0 clamp.
Similar 0-hour blocks on 11-19 04:00-06:00. The KNN model (30-day,
num_lags 96) learns these multi-kW overnight blocks as "house load"
and reproduces them. This also likely explains the 2026-11-03 finding
(MAE worse than a flat average). Not yet established: why the EV power
sensor and house load are misaligned (EV sensor update lag/cloud
polling vs. local, or the EV sensor dropping to 0 while charging).

Impact: EMHASS MPC/DH plan against phantom multi-kW night load whenever
the model reproduces an EV night.

**Update 2026-11-21 08:20 - ACTUAL root cause of the 6-9 kW forecast:
stale model trained on the wrong sensor.** EMHASS logged on every
predict: "The saved model was trained on
sensor.power_myhouse_load_no_var_loads but
sensor.power_myhouse_load_no_var_loads_no_ev was retrieved". The saved
`/data/load_forecast_mlf.pkl` was trained on the WITH-EV sensor (age
unknown - possibly months); the weekly Sunday 19:00 fit has no HA-side
success tracking, and the 2026-11-15 fit fell in the 401 outage. The
forecast turned bad from 2026-11-20 00:00 (also changed from 96 to 288
points) - coincides with the 18 hourly add-on restarts/rebuilds on
11-19, so an EMHASS-side behaviour change is suspected but unproven.
The training-data contamination below was real but NOT the driver
(11-18/19 forecasts were normal despite it). **Fixed 2026-11-21 08:18:**
manual forecast-model-fit (no_ev sensor, 30 days, 2910 points; backtest
MAE 266 W, RMSE 549 W, R2 0.12; test R2 -0.30 - weak) then predict:
96 points, mean 428 W, max 1.19 kW - plausible. Follow-up done 2026-11-21 08:35: `emhass_ml_monitoring.yaml` (Layer 1,
safety & watchdog folder) - fit/predict OK/FAILED + last success (set by
run_mlforecast_* scripts in emhass_scripts.yaml), measured operational
forecast accuracy per 15-min slot -> daily R2/MAE/RMSE (long-term stats),
and stoplight `sensor.ml_load_forecast_quality` on 7-day R2 (good >= 0.5,
modest >= 0.2, bad < 0.2; purple for fit/predict failed or fit > 8 days
old). Deployed live. Note: the new last-success input_datetimes default to
"today 00:00" until the first real run overwrites them. Still open: tune
the model once clean history accumulates.

**Update 2026-11-21 01:00 - refined root cause (two independent data
faults, not a systematic EV misalignment).** 10-day hour-by-hour check
of grid import + inverter output vs. Easee-reported EV power across all
8 EV sessions:
1. **Easee cloud data stall (rare)** - only one real stall: 11-16
   00:12-02:39, Easee reported 0 W / `awaiting_start` while the M1 meter
   showed a flat 7.80-7.90 kW for 85 min (EV signature; not the washing
   machine or pellet boiler, which are short ~2 kW bursts). Session
   energy then batch-jumped (4.4 -> 16.7 kWh in <1s at 03:13). All other
   sessions matched within a few hundred W, with ~1 kW mismatches only
   at session start/end hours.
2. **SolarEdge M1 meter freeze (recurring)** - after a brief
   `unavailable` the meter sat at exactly `-0.0` for 2-3h: 11-16
   03:54-06:10 and 11-19 04:00-06:59, both inside the nightly Modbus
   reconnect window. This is the source of the 0 W training blocks.
`sensor.ev_charging_power_w` itself is a faithful x1000 copy of the
Easee power source - not a template bug.

**Fix (2026-11-21, `emhass_sensors.yaml`):** `availability:` added to
`sensor.power_myhouse_load_no_var_loads_no_ev` - unavailable (and so not
written to InfluxDB / not used for training) when (1)
`binary_sensor.solaredge_modbus_data_fresh` is off (~5% of time over 10
days, incl. the two freezes), (2) car connected
(`sensor.ev_charger_status_normalized_source` != disconnected) AND result
> 4000 W (Easee stall signature), or (3) raw result < -200 W (delayed EV
value subtracted from an interval without it; previously clamped to a
fake 0). Household loads (washing machine, pellet boiler) deliberately
stay in. Chosen over a blanket "car plugged in" exclusion, which would
discard all hours the car sits connected (sometimes days).

**Status: deployed live 2026-11-21 01:13, not yet verified.** Past
contaminated history remains in InfluxDB until it ages out of the
30-day fit window (~mid-December) unless 11-16 00:00-07:00 and 11-19
04:00-07:00 are deleted manually. Verify: sensor goes unavailable during
the next nightly meter freeze / Easee stall; re-run the accuracy
comparison once clean history dominates.

---

## Winter-Time (DST Fall-Back) Transition Caused a Full-Day EMHASS Failure

Found 2026-11-02 ("the autumn clock change caused EMHASS to fail a whole
day - same as spring, where we made a fix that apparently didn't work").
First theory (centered on `script.emhass_update_sliding_window`) was
wrong - confirmed dead code by the user, superseded by a refactor;
retired.

**Real, confirmed-live bug**: `emhass_sensors.yaml`'s `forecast_merged`
producers format timestamps as bare local-time strings
(`as_timestamp | timestamp_custom('%Y-%m-%d %H:%M')`, no UTC offset).
`emhass_scripts.yaml`'s MPC/day-ahead scripts re-parse that bare string
with `as_datetime | as_local`, which is ambiguous on the one local hour
(02:00-02:59, CEST then CET) that occurs twice during fall-back -
Python's `fold=0` default always resolves to the first (CEST) instant,
so both real quarter-hours decode to the same dict key and the second
silently overwrites the first in `load_cost_forecast`/
`prod_price_forecast`/`pv_power_forecast`, for roughly the 2-day
`num_prediction_days` window bracketing the transition.

Why the earlier spring fix didn't help: spring-forward *removes* an hour
(a short list); fall-back *duplicates* a key (same-length list, silent
overwrite) - different failure shapes from the same root design choice
(bare local-time strings), so a fix aimed at one doesn't touch the
other.

Fix candidates, not implemented: (a) format `ts` with a UTC offset/epoch
at the source (`emhass_sensors.yaml`) so the consumer scripts never need
`as_local()`'s fold guess - the complete fix, touching every producer
plus both consumer scripts; (b) a narrower consumer-side dedup by epoch
instead. **Status: root cause confirmed in live code, not yet fixed.**
Direct historical confirmation of the actual 2026-10-25 failure wasn't
possible (recorder only retains ~10 days).

---

## PV Sensor Trickle Readings Bypass the "No PV" Branch Logic

Found 2026-11-02 while reviewing whether Maximize Self Consumption is
the right nighttime fallback. Several `battery_forecast_control.yaml`
branches test `pv == 0` exactly to mean night/no-PV; the existing
4-minute trailing-mean smoothing doesn't help a steady sensor offset or
trickle reading, which can fall through into the `pv > 0` branches
(including real grid-charging logic) at night. Not confirmed to have
caused a live misfire - found by code review, not an observed incident.

**Fix implemented 2026-11-02**: new `input_number.pv_noise_floor_w`
(default 20W, `batterycontrol_helpers.yaml`) applied to `pv`/`pv_now` in
`battery_forecast_control.yaml` before any branch condition sees them.

Also found and fixed the same day: README.md's Decision Logic section
was stale - the no-PV/price-too-high/hold case actually lands on
`solar_power_only` (the stronger guarantee), not Maximize Self
Consumption as previously documented; corrected to match the live code.

**Status: deployed, confirmed 2026-11-20** (`input_number.pv_noise_
floor_w` = 20 live). Not yet verified against a real low-PV/trickle-
reading night.

---

## High SOC Hold - Charge Limit = 0 Does Not Stop Charging While Maximize Self Consumption Stays Active

Found 2026-11-01 ("SOC recurrently goes over 90%", later also "flip-
flopping between charging and discharging"). Confirmed recurring (11
clean Hold cycles over 10 days, SOC reaching 100% in 10 of 11) and
confirmed at the register level: `number.solaredge_i1_storage_
charge_limit` genuinely held at a steady, unreset 0 for 6h45m straight
on one occasion, yet SOC still climbed from 87.78% to 100%. A historical
survey (121 days of daily stats) found a sharp regression onset at
2026-10-14 (chronic 100%-SOC days from then on, vs. 9 clean weeks
before); candidates checked for that onset - a SolarEdge integration
update (17 days too late), a manual mode test (too small/reverted),
firmware (unknown, not visible from HA), and an EMHASS
`battery_soc_final_reward_factor` parameter introduced in a nearby PyPI
release (plausible, unconfirmed) - none were conclusively confirmed or
ruled out.

**Three distinct, independently-confirmed mechanisms, found over
several days of live testing:**

1. **The inverter's own self-consumption logic appears to override
`charge_limit=0` and route real PV surplus straight to the battery on
the DC side**, confirmed under Maximize Self Consumption, Solar Power
Only, *and* Discharge to Maximize Export alike, and independent of the
discharge-limit register's value (reproduced identically at a nonzero
460W discharge limit). This is the core, still-unsolved defect - no
command mode tested on this hardware actually blocks DC-coupled
PV-surplus charging.

2. **The same PV-surplus charging also suppresses the inverter's own
measured AC-side output (`sensor.solaredge_i1_ac_power`) to ~0**,
confirmed live and directly observed by the user (apparent "production
stopped" while in Discharge to Maximize Export with discharge=0;
switching live to Maximize Self Consumption immediately restored
measured production). Reproduced under Solar Power Only too. Mechanism
not isolated (the discharge-limit register capping *total* AC output in
these modes vs. something SOC-ceiling-proximity-specific are both
plausible, neither tested in isolation).

3. **A Modbus reconnect event (confirmed via a disconfirming natural
experiment to be the trigger, not elapsed time) resets the live mode
AND both charge/discharge-limit registers to the inverter's own
non-project defaults** (Maximize Self Consumption / 5000W / 5000W) and
does not self-heal - nothing re-asserts until an unrelated `effective_*`
helper next changes state. The same reconnects also knock `number.
solaredge_i1_storage_command_timeout` from this project's 900s to 3600s,
which likewise never self-heals, though it was specifically disconfirmed
as a gating factor for the limit-register drift itself.

**Fix history for the Hold mode itself** (mechanism #3's reconnect-reset
risk remains regardless of which mode Hold uses): single-mode
`discharge_to_minimize_import` (2026-11-02) -> day/night split by mode,
`solar_power_only` (day) / `discharge_to_maximize_export` (night)
(2026-11-02) -> that split mode-flapped because its PV test used raw
instantaneous PV instead of the already-proven smoothed sensor, fixed by
switching to `sensor.pv_production_smoothed` (2026-11-03) -> **current
design (2026-11-03): the mode distinction was removed entirely - Hold
now always uses `discharge_to_maximize_export`, day and night; only the
discharge limit differs (0 by day, the existing dynamic value by
night)** - adopted once mechanisms #1/#2 showed Solar Power Only wasn't
actually any safer. The same fix was applied to one other
`solar_power_only`-under-PV-surplus corner in `battery_forecast_
control.yaml`'s Maintain Zone; three no-PV `solar_power_only` branches
were deliberately left alone (mechanism #1 requires real PV surplus to
manifest at all).

A separate, inconclusive side investigation (2026-11-03): the SolarEdge
integration's `select.solaredge_i1_ac_charge_policy` ("AC Charge
Policy", found set to "Always Allowed") likely governs AC/grid-side
charging specifically - probably unrelated to the DC-coupled PV-surplus
defect above, but not disproven (the one authoritative source, a
SolarEdge PDF, couldn't be fetched). Left unchanged - don't disable it
experimentally, since this project's own `charge_from_solar_and_grid`
mode (including SOC<10% critical recovery) depends on AC-side charging
working.

**Battery Control Drift Watchdog** (`watchdog_automations.yaml`, added
2026-11-02 to catch mechanism #3's silent reverts): originally
force-re-applied on every detected mismatch; changed to **record-only**
2026-11-03 after the user identified it as a likely contributor to
Modbus/mode thrashing. A real 35h13m SOC-sensor outage (2026-11-06) and
a calmer 24h check (2026-11-09) both found the record-only watchdog's
own log substantially understating real drift, from two bugs: its
condition required the SOC sensor available (blinding it for nearly the
whole 2026-11-06 outage), and `drift_reason` only ever reported the
highest-priority mismatched category (hiding simultaneous charge+
discharge mismatches under a logged "mode" entry) - both checks found
real episodes up to 16+ minutes with real power excursions up to
~3000W. **Both watchdog bugs fixed 2026-11-09**: the SOC-availability
guard was removed; `drift_reason` now joins every mismatched category
instead of just one; the mobile push was throttled to one per episode
(new `input_boolean.battery_control_drift_active`) so the richer
per-tick logging didn't become per-tick phone spam.

**Update 2026-11-20 - mechanism #3 caught twice with the watchdog live;
re-apply-on-reconnect fix proposed.** Two Modbus reconnects
(`unavailable` 02:23:34-02:23:57 and 16:45:28-16:45:51) each brought the
inverter back at Maximize Self Consumption / 5000W / 5000W against an
intended MSC / 2W / 100W (night) and MSC / 2W / 0W (evening). The
watchdog logged both correctly - 10 ticks 02:25-03:10 and 19 ticks
16:50-18:20, both categories joined in one `drift_reason`, one push per
episode - which **confirms the 2026-11-09 watchdog fixes working live**,
including the multi-category join that had not been exercised before.
Real register drift lasted ~51 min and ~1h39m (registers restored
03:14:38 and 18:24:23-31), ending only because an unrelated `effective_*`
change happened to re-trigger the apply automation. A third, single-tick
alert (08:10, discharge expected 0 / live 100) was a benign race: the
5-min tick landed seconds before the apply wrote the new limit
(08:10:18).

Root cause of the persistence: `apply_effective_battery_control.yaml`
triggers only on `effective_*` helper changes, so a reconnect that leaves
those helpers unchanged is never re-applied. 10-day history shows these
resets happen **twice a day** (~03:20-04:05 and ~17:15-18:05, drifting
earlier: 02:23/16:45 on 11-20); usually the next MPC-driven `effective_*`
change re-applies within 2-16 min, they only persist when the intended
values are flat. Cause of the schedule unknown (inverter/network/DHCP
side - needs router or inverter logs). **Fix (2026-11-20, docs only,
supersedes the earlier `from: unavailable` proposal)**: a template
trigger (`id: limit_reset`) re-runs the full apply when a live limit
reads 5000 while its intended value is not 5000, held 2 min (exceeds the
~70s poll lag). Value-based rather than `from: unavailable` because some
resets (11-19 16:22) jump straight to 5000 with no recorded unavailable
state, and because 5000 is sometimes a genuine intended value. The full
apply also restores the mode and the timeout (reconnects reset the
timeout to 3600s). Optional follow-up: a ~30s
grace period in the watchdog to skip ticks that land during an in-flight
apply (would suppress 08:10-type alerts).

Also noted, not investigated: 2026-11-19 ~07:15-21:55 sustained mode
drift (expected Charge from Solar Power and Grid, live Maximize Self
Consumption) with no reconnect visible in the alert log - a different
mechanism that the proposed reconnect trigger will not catch.

**Status: open. Mechanisms #1 and #2 (PV-surplus DC-side charging
bypassing charge_limit; the same charging suppressing measured AC
output) are the core unsolved problem, and no command mode tested is
free of them - the current Hold design is the best available
mitigation given that, not a real fix. Mechanism #3 (reconnect resetting
registers) now has a 5000/5000 re-apply trigger - **confirmed working
live 2026-11-21**: reconnect 03:22:55-03:23:18 reset limits to
5000/5000, re-applied to the intended 147/0 at 03:26:17 (~3 min, vs.
51 min-1h39m drift before).

**Update 2026-11-21 - reconnect cause narrowed to the inverter side.**
With solaredge_modbus_multi at debug: the integration opens and closes
a TCP connection on every poll (~7s cycle, normal). At 03:22:55 a
connect attempt failed ("could not connect to 192.168.10.4:1502"),
entities went unavailable ~22s until the next successful connect. The
inverter's storage-control registers came back at its own defaults
(MSC/5000/5000) - an HA-side or network-side connection failure alone
would not change inverter registers, so the inverter's Modbus/control
subsystem itself restarted. Schedule: twice daily, ~03:20-04:05 and
~17:15-18:05 (10-day list 11-11..11-20, plus odd extras 11-15 22:01,
11-17 01:12), ~22s each. Web research 2026-11-21: matches widely reported
SolarEdge behaviour - other HA users see the same twice-daily inverter
reset (e.g. ~05:00/~19:00, <1 min, register settings lost), and SolarEdge
support reportedly confirmed it as expected behaviour; community
workaround is a re-apply trigger on recovery from unavailable, same as
limit_reset. Not fixable from HA - limit_reset is the mitigation.
Sources: community.home-assistant.io thread 464084 (pages 27, 37,
post 545). The 2026-11-09 watchdog fixes are confirmed working
live 2026-11-20.**

---

## Critical-Recovery Charge Limit Write Silently Dropped

Found 2026-10-11 verifying a real SOC<10% critical-recovery event. Mode
and discharge-limit writes landed correctly and promptly, but the
charge-limit write (3000W) never reached the inverter - the register was
already stuck at a stale 5000 (a leftover from an unrelated branch)
immediately before the critical branch fired, and stayed at 5000 through
the entire critical-recovery window. Both the automation trace and the
write-verify telemetry reported this write as a clean success - nothing
flagged it, and by the time it was noticed the relevant traces had
already rolled past their 5-run retention. Not a direct safety hazard
(mode and discharge-limit, which actually prevent over-discharge, both
worked), but a real detection gap.

Two hypotheses, neither confirmed: mode-dependent register gating on the
inverter's firmware, or a genuine dropped/overwritten Modbus write.
Needs a fresh trace caught during a future occurrence to distinguish.

**Status: open, not fixed.** `test_plan.md` Test 1.3 downgraded to
PARTIAL PASS.

---

## SOC<10% Critical Recovery Triggered By a Physically-Implausible Single-Sample SOC Reading

Found 2026-10-11, same event as above. A physics check (even under
maximally generous assumptions about the enforced discharge cap) rules
out a real continuous discharge producing the observed 22.2% -> 0.0%
drop in the 11.43 minutes between samples - the most energy that could
plausibly have been drawn in that time is well short of what the drop
implies. The 0.0% reading also has none of the texture of a real
discharge curve (an 11+ minute gap with no intermediate samples, versus
the immediately-following real recovery climb's clean ~70-90s steps) -
most likely a bad single-sample Modbus read (a known-flaky entity on
this install) that happened to return a plausible-looking wrong value
instead of unavailable/unknown, which the existing guards don't screen
for.

The system's response to the bad trigger was itself safe (forcing
charge + zero discharge can't harm an actually-healthy ~22% battery).
Proposed hardening, not implemented: a plausibility check capping how
much a single SOC reading can move, based on the currently-enforced
limits and elapsed time since the last reading.

**Status: open, not fixed.** Reclassifies the original stats-based
near-miss concern as a probable sensor glitch handled safely, not a
real near-empty-battery event.

---

# Fixed Issues

## Compound EMHASS-Outage Incident 2026-11-19/20 (401 auth, 415 header, 400 price_prep, health-check false-positive, hourly addon-restart)

Found 2026-11-19 from the user's report that the dynamic discharge limit
had been stuck at 0 all day despite target SOC at 20%, with no
discharge happening. Investigation (continuing into 2026-11-20 via a
fresh "failing MPC runs from 04 this morning" report) uncovered five
separate, independently-confirmed bugs stacked together. This session
has no live write access - every fix below is a project-doc change the
user must deploy.

**1. EMHASS REST API 401 Unauthorized** (~10 days intermittent, ~17h
continuous outage on 2026-11-19). Root cause: a recent EMHASS add-on
update requires bearer-token auth
(https://emhass.readthedocs.io/en/latest/api_authentication.html);
`emhass_restcommand.yaml` wasn't sending it. Fixed by adding `headers:
authorization: !secret emhass_bearer` to all 6 rest_commands (the
secret's value must be the literal `"Bearer <token>"` string, per
EMHASS's own `secrets.yaml` example - not just the bare token).

**2. Follow-on regression: HTTP 415 Unsupported Media Type** on the 3
live Siegen-energy rest_commands, introduced by the first version of
fix #1. Root cause (confirmed from the user's actual live file, not
guessed): every `headers:` block spelled the key `content_type`
(underscore) instead of the real HTTP header `Content-Type` (hyphen) -
not recognized by EMHASS as a Content-Type header at all. Fixed by
using hyphenated `Content-Type` inside `headers:` throughout (no
top-level `content_type:` field anywhere in the file). **Confirmed
fully fixed live 2026-11-19** end-to-end: a real MPC run (`price_prep`,
`optim_solve`=Optimal, all `publish-data` POSTs HTTP 201) succeeded with
no errors. A separate, earlier HTTP 400 ("Passed data from passed list
is not long enough" at `price_prep`) never recurred once Content-Type
was fixed - treated as a casualty of the same broken-header window, not
a separate still-open bug.

Minor unresolved loose end from the same log: `publish-data`'s
`custom_unit_production_price_id` posts to `sensor.unit_prod_price`
instead of the requested `sensor.mpc_pv_export_price` - looks like an
EMHASS-side quirk on this one field only. Worth the user spot-checking
that sensor updates correctly; not investigated further.

(Bug #3 of this incident, the date-filter no-op, is still open - moved
to its own entry under Open Issues on 2026-11-20.)

**4. Tautological health check in `emhass_automations.yaml`.** Both
plan-generation automations judged success via `condition: state` on
`sensor.mpc_pv_optim_status`/`dh_pv_optim_status` == "Optimal" - but
those sensors are *only* ever written by the same (sometimes-failing)
publish step, so once frozen on "Optimal" the check could never
register a failure again. This masked the entire bug-#1 outage from the
existing 45-min/90-min watchdog chain. First fix (2026-11-19) added a
recency requirement on the sensor's own `last_updated` (<120s).

**This recency fix introduced a new, near-100%-false-positive bug**,
found 2026-11-20: EMHASS posts the same literal, unchanging "Optimal"
string on every success, and Home Assistant does not advance
`last_updated`/`last_changed` on a repeat write of an identical
value+attributes - only `last_reported` does (confirmed live via direct
`ha_eval_template` reads against the state object; also confirmed the
`ha_get_state` tool does not reliably surface `last_reported`'s true
live value, so re-check via `ha_eval_template` if this comes up again).
So the recency check failed on every genuine success after the first,
reading perpetual "FAILED" even while EMHASS's own log showed clean
200-OK posts. Fixed 2026-11-20 by swapping `.last_updated` ->
`.last_reported` in both automations. **Confirmed deployed and working
live 2026-11-20**: status correctly read OK across 3+ consecutive ticks
post-deploy, with no FAILED since.

**5. Side effect of bug #4: genuine hourly EMHASS add-on restarts,
previously mistaken for independent EMHASS-side behavior.**
`input_datetime.emhass_mpc_last_success` (only ever updated inside the
broken "OK" branch) froze for ~19 hours; the pre-existing
`EMHASS_addon_watchdog_auto_restart` automation (built 2026-09-22, fires
`hassio.addon_restart` after 90 min of apparent MPC stall, 1-hour
cooldown) saw a permanently-stale timestamp and restarted the add-on
every single hour, on the hour, for 18 consecutive hours. Each restart
triggered this install's known "rebuilds its Python environment on
every container start" fragility (see `roadmap.md` - EMHASS Addon
Build-On-Start Fragility), producing ~60s of downtime each hour (the
`Quart shutdown complete` / `Building emhass` / relaunch sequence seen
in the logs) - this had looked like an unexplained, independent
EMHASS-side hourly restart, but was actually a direct consequence of
bug #4. **Confirmed fully stopped 2026-11-20** once the `last_reported`
fix deployed: the 19:00 restart slot (and the two preceding 15-minute
ticks) passed clean, with `emhass_mpc_last_success` advancing normally
every 15 minutes since. No separate fix was needed or applied here
beyond #4's own fix.

**Status: closed. Bugs #1/#2 (auth+415+400) confirmed fixed live
2026-11-19; #4/#5 (health check + hourly-restart side effect) confirmed
fixed live 2026-11-20. Moved to Fixed Issues 2026-11-20.**

---

## HA Cleanup - Dead Sliding-Window Subsystem and Unused Override Booleans Removed

Removed 2026-11-02 per the user's request to clean up unused HA
entities (doc-only, since this session has no live write access).
Confirmed dead: `script.emhass_update_sliding_window`/`set_mpc_soc_
final_target` and the 5 helpers only they ever wrote (all frozen since
2026-07-09, no live call site) - removed from `emhass_scripts.yaml`/
`emhass_helpers.yaml`. Also removed 5 unused manual-override
`input_boolean`s from `safety_and_watchdog_helpers.yaml` (confirmed no
read/write anywhere, across a wider search than the original 2026-09-24
flag). Left for manual HA-UI cleanup, since a doc edit can't reach them:
4 ghost registry entities with no YAML backing at all, and one dead
dashboard row referencing the removed `mpc_prediction_horizon` helper.
Several other reconfirmed-but-out-of-scope items are carried forward in
`roadmap.md`.

**Status: doc-side removal complete 2026-11-02, not yet confirmed
reloaded live; 3 manual follow-ups remain for the user.**

---

## Modbus Queue Pending-Count Counter Leaks On Lock-Wait Timeout

Found 2026-10-19: `binary_sensor.solaredge_modbus_queue_backlog` stuck
"on" for 33+ hours despite normal real throughput. Root cause: the
pending-count's +1/-1 pairing leaked +1 on every lock-wait timeout
(confirmed via a 1:1 correlation with the timeout counter). Fixed the
same day: the lock-wait-timeout branch now also decrements (floored at
0).

**Status: deployed and confirmed live 2026-10-20** against a real
timeout event - the floor returned to its correct resting value instead
of climbing further. `test_plan.md` Test 6.4 -> PASS.

---

## EMHASS Rejects Optimisation Requests When `sensor.max_active_power_1` Goes Unavailable

Found live 2026-10-11: that sensor (a separate cloud-API integration,
not the local Modbus one) went unavailable, feeding a silent `0` into
EMHASS's `inverter_ac_output_max`/`_input_max` and causing HTTP 400 on
every optimisation request for the rest of the day. Fixed by replacing
it with a static helper, `input_number.emhass_max_inverter_power`
(10600W nameplate), hardening the fallback from 0 to 10600.

**Status: confirmed live 2026-10-11** - the next MPC cycle succeeded
with correct payload values, no further 400s.

---

## Write-Outcome Telemetry Is Readback-Based

Added 2026-10-06: `script.modbus_queue` classifies writes success/
failure by snapshotting and reading back ~2s later (no other way to
detect a caught failure under `continue_on_error`). Found too-fast the
next day - roughly half of real writes were misclassified as failure
before the entity had settled. Fixed 2026-10-07 by splitting
verification into an independently-queued `script.modbus_write_verify`
(up to 8s wait, fire-and-forget) so `script.modbus_queue` itself stays
unblocked. A first redeploy left the new script unwired (a partial-copy
gap); a corrected redeploy was confirmed working.

**Status: confirmed wired in and working live 2026-10-07** via a real
write trace.

---

## Entity ID Documentation Mismatches (Derived From Name/Alias, Not unique_id/id)

Found 2026-10-07: 3 entity_ids documented in this project didn't match
what Home Assistant actually assigned, because HA derives a template
entity's id from `name:` (an automation's from `alias:`), never from
`unique_id`/`id:`. Docs-only fix across `entities.md`/`roadmap.md`/this
file - no YAML changed, nothing was actually broken live.

**Status: fixed 2026-10-07, no live/functional impact.**

---

## Negative Price Curtailment - Site Limit Restore Gap

Found 2026-10-06: the curtailment-off handler never restored `number.
solaredge_i1_site_limit` from 0, confirmed stale for ~6 hours -
empirically harmless the same day (the switch, not the number, is the
real export gate on this install). Closed by redesign rather than
patch: the switch now stays permanently ON, and the number itself is
varied 0<->1,000,000W on every price transition, so there's nothing to
forget to restore.

**Status: confirmed live 2026-10-19** against a real negative-price
episode (the defensive switch re-assert fired correctly; the number
tracked 3 curtail/restore cycles cleanly). Still open in `roadmap.md`:
no negative-price window has yet coincided with real PV surplus, so
actual export suppression during curtailment has never been directly
observed.

---

## EMHASS Outage Fallback Guard

Found 2026-09-22: an 8+ hour EMHASS outage (stuck rebuilding after a
power-outage DNS failure) went undetected by Layer 4A, whose
EMHASS-dependent variables silently fell back to fake zeros, causing a
real 6+ hour charge/discharge oscillation against genuinely high
prices. Fixed with a `sensor.emhass_health`-based guard that skips the
run (freezing last-good values) instead of computing from fabricated
data, plus an addon-level auto-restart watchdog (`hassio.addon_restart`
after 90 min of stalled MPC, max once/hour - this is the same automation
later implicated in the 2026-11-20 hourly-restart finding, see the
Compound EMHASS-Outage Incident entry above).

**Status: deployed and structurally confirmed live 2026-09-23**; full
behavioral confirmation (an actual skipped run or addon auto-restart)
came later, in the 2026-11-20 incident above - which turned out to be a
symptom of a different, since-fixed bug rather than evidence about this
guard's own correctness.

---

## Modbus Queue Lock Release Not Exception-Safe

Found 2026-08-28: a failed SolarEdge write could abort `script.
modbus_queue` before its lock-release step, leaving `modbus_busy` stuck
for up to ~30 minutes. A naive fix was deliberately deferred over flood
risk. Fixed 2026-09-20: `continue_on_error: true` on the whole `choose:`
block, with ~13 scattered per-branch delays consolidated into one
uniform post-choose delay on both the success and failure path.

**Status: confirmed live 2026-09-24** - zero stuck-lock episodes over
60s across ~4.5 days/~750 cycles, including real connectivity noise
releasing cleanly in 18-33s.

---

## Layer 4A SOC-Unavailable Guard

Found 2026-09-19: a real ~31h SOC-sensor outage ran Layer 4A on a fake
`soc=0` fallback the whole time (landed, by luck, on a conservative
branch). Fixed by adding the same unavailable/unknown guard already
proven elsewhere in the project.

**Status: confirmed live 2026-09-24** against a real ~2h13m outage -
requested mode/limits stayed frozen throughout, with the first change
only on the first tick after recovery.

---

## EV Charging Sensor Swap

`binary_sensor.ev_charging_on` was found dead/unavailable since at
least 2026-09-03, silently preventing EV-charging branches from ever
firing. Swapped to the live `binary_sensor.ev_charging_active` across
all 4 referencing files. A follow-up gap (the trigger entity_id list in
`calculate_effective_battery_control` still named the dead sensor) was
found and fixed 2026-09-25.

**Status: verified live 2026-09-15** against a real EV session for the
main swap; the 2026-09-25 trigger-list follow-up was not separately
re-verified live.

---

## Align Safety SOC Limits And EMHASS SOC Limits

SOC min/max were hardcoded independently in the safety layer and in
EMHASS. Fixed 2026-09-03: both now read the same `input_number.
minimum_state_of_charge`/`maximum_state_of_charge` helpers.

**Status: confirmed 2026-09-04** (file match) **and runtime-confirmed
2026-09-24** via EMHASS's own logged runtime parameters matching the
live helper values.

---

## Modbus Connectivity - SOC-Sensor Unavailable False-Trigger

Found 2026-09-03: a nightly ~22:00 SOC-sensor dropout's `float(0)`
fallback was triggering a real hardcoded 3000W grid-charge for the
glitch's duration. Fixed with an unavailable/unknown guard on
`calculate_effective_battery_control` and `battery_high_soc_hold`.

**Status: verified** - zero spurious triggers across the following
week's 6 dropouts. The underlying nightly dropout cause itself remains
open in `roadmap.md`.

---

## Midday Charge/Discharge Oscillation - Grid Charge Export Cooldown

Implemented 2026-08-27 to address the symptom (a hysteresis cooldown
before selling back a just-grid-charged battery at a loss) rather than
the underlying EMHASS MPC plan-chatter cause, which remains open in
`roadmap.md`. Full design in `architecture.md`.

**Status: implemented 2026-08-27.**

---

## Midday Charge/Discharge Oscillation - PV/Load Sampling Smoothing

Added 2026-08-28: 4-minute trailing-mean PV/load sensors for every
mode-selecting branch (momentary spikes/dips were flipping mode).

**Status: verified live 2026-08-29** - zero oscillation across a full
morning.

---

## Midday Charge/Discharge Oscillation - Branch-Coverage Gaps

Three no-`default:` branch gaps found in `battery_forecast_
control.yaml`'s top-level `choose:` (each silently leaving mode/limits
unchanged for that cycle), fixed 2026-09-03 (one gap) and 2026-09-15
(two gaps). Two of the three are verified live; the third unexercised.
The mirror-image case (`grid_fc<0` and `batt_fc<0` together) remains
open, not blind-fixed.

**Status: fixed; two of three gaps verified live.**

---

## Write Pressure - Mode Command Guard

Implemented 2026-08-26: skips the mode-select command and the
command-timeout reset (15 min until 2026-11-20, now 3600s) whenever effective/live/default mode all already
agree on Maximize Self Consumption.

**Status: confirmed live 2026-08-27**; the one untested precondition
(guard inactive when default mode disagrees) was tested live 2026-10-22
and passed. `test_plan.md` Test 7.6 -> PASS.

---

## Dynamic Discharge Limit Under-Delivers vs. EMHASS Plan During Sustained Price Spikes

Found 2026-11-02: the dynamic discharge/charge-limit sensors were pure
single-point (next 15-min forecast) proportional controllers, decaying
toward 0 near that one checkpoint even when EMHASS's actual multi-step
plan wanted sustained discharge much longer (observed: 932W real
discharge against a 1965W load spike at the literal price peak). Fixed
by widening both sensors to the same proven 3-point (0.5/0.3/0.2)
weighted-blend lookahead already used for `soc_target`.

**Update 2026-11-19**: the lookahead itself was correct, but the
"future points only" filter was a silent no-op due to the same
date-string bug tracked in the Date-String Filter No-Op entry under
Open Issues - fixed there too.

**Status: implemented, not yet confirmed deployed/verified live.**