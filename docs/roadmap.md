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

Several automations rely on:

```text
sensor.mpc_pv_optim_status
sensor.dh_pv_optim_status
```

Investigate whether optimisation failures should automatically trigger fallback planning behaviour rather than only updating watchdog status.

Update 2026-09-22: Layer 4A's own fallback behaviour (what
`battery_forecast_control.yaml` does when these are missing) was fixed
the same day after a real 8+ hour EMHASS outage caused a repeated
charge/discharge oscillation - see Resolved section below and
known_issues_and_fixes.md - EMHASS Outage Fallback Guard. Still open
here: the addon-side question of whether EMHASS itself should serve a
degraded/cached plan when a fresh optimisation fails, rather than simply
not publishing.

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

Review all published:

```text
mpc_pv_* sensors
dh_pv_* sensors
```

Identify:

```text
unused sensors
duplicated sensors
legacy entities
```

Update entities.md after cleanup.

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

(Reconfirmed during the 2026-09-24 YAML header/hygiene audit:
`negative_price_curtailment.yaml` and `gridconstrain_helpers.yaml` -
the only two files that define/could plausibly use it - still show no
read or write of `grid_export_limited` anywhere in either file.)

---

### Possibly Dead: `rest_command.publish_data`

Found during the 2026-09-24 YAML header/hygiene audit:
`rest_command.publish_data` (`emhass_restcommand.yaml`) targets a
different host (`192.168.10.3`) than the other three `emhass_*`
rest_commands in the same file (all `127.0.0.1`), and no automation or
script in `emhass_scripts.yaml`/`emhass_automations.yaml` was found
calling it (they call `rest_command.emhass_publish_data` instead, a
distinct entity). It's listed in `entities.md`. Possibly legacy/dead
config from an earlier setup - confirm whether it's still needed before
removing it.

---

### Possibly Dead/Duplicate: `shell_command.trigger_nordpool_forecast`

Found during the 2026-09-24 YAML header/hygiene audit:
`shell_command.trigger_nordpool_forecast` (`emhass_shellcommand.yaml`)
posts raw 24h Nordpool-only data directly to EMHASS's `dayahead-optim`
endpoint via curl. This appears to duplicate/be superseded by
`emhass_scripts.yaml`'s `load_cost_forecast`/`prod_price_forecast`
Jinja blocks feeding `rest_command.emhass_dayahead_optim` (merged
Nordpool+EPEX, 2-day/15-min horizon) - a more complete forecast path.
No trigger or call site for the shell_command was found anywhere in
`emhass_automations.yaml` or `emhass_scripts.yaml`. Possibly legacy;
confirm before removing.

---

### Unused `ev` Variable In `battery_forecast_control.yaml`

Found during the 2026-09-24 YAML header/hygiene audit:
`emhass_battery_forecast_control` reads `binary_sensor.
ev_charging_active` into a local `ev` variable, but no branch in the
automation's current `choose:` logic references it - EV-aware discharge
limiting is actually owned by `automation.ev_charging_discharge_control`
in `batterycontrol_automations.yaml`. Confirm whether `ev` is
intentionally reserved for future use in this automation or should be
removed as dead config.

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

### Possibly Unused Safety-Override Input Booleans

Found during the 2026-09-24 YAML header/hygiene audit: `input_boolean.
battery_safety_override_active`, `battery_force_discharge`,
`battery_force_charge`, `battery_discharge_blocked`, and
`battery_charge_blocked` are defined in `safety_and_watchdog_helpers.yaml`
but were not found to be read or written by any of
`safety_limits_and_override.yaml`, `safety_and_watchdog_helpers.yaml`,
`watchdog_sensors.yaml`, `watchdog_automations.yaml`,
`apply_effective_battery_control.yaml`, or `solaredge_modbusqueue.yaml`
during the audit (`batterycontrol_automations.yaml` and the other
battery-control-layer files weren't confirmed either way for these
specific entities). Confirm whether these are wired up as manual-override
controls somewhere, or are vestigial/planned-but-unimplemented.

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

Observed errors:

```text
Cancel send, because not connected
No response received after 3 retries
Request cancelled outside library
transaction_id mismatch
```

Need to determine whether these correlate with:

```text
port probe failures
data freshness failures
write bursts
battery control mode changes
negative price curtailment
```

The modbus_busy stale-lock gap found 2026-08-28 (Test 6.3 / 7.5) was fixed
2026-09-20 - see Resolved section below and known_issues_and_fixes.md for
current deployment status. The 48h error-log baseline of connectivity
noise (27x "Cancel send"/"Repeating call"/"No response", 7x transaction_id
mismatch) found during that same review is still open and not yet
root-caused - worth a focused before/after comparison now that the
lock-release fix has landed, and/or a longer observation window to see
whether it correlates with write bursts, mode-command-guard skip
patterns, or something environmental (RS485/TCP contention, inverter
response latency).

### Update 2026-09-10 - Two new real Modbus write failures found

The deeper charge/discharge review's Modbus sampling (5 windows across
2026-09-04 through 2026-09-10, explicitly not exhaustive 6-day coverage)
turned up two actual command **write** failures - not the usual poll/read
noise - on 2026-09-08 22:45:02 and 2026-09-10 13:30:03: "Modbus
Queue...Modbus Error: [Input/Output] Request cancelled outside library".
Not previously documented. Same sampling also re-confirmed the baseline
connectivity noise is essentially unchanged from the 2026-08-28 finding
above (still present, similar magnitude). Not yet root-caused or
correlated with anything specific; noted here for whoever picks up this
investigation next.

### Update 2026-09-19 - Extended SOC-sensor outage (~31 hours), root cause unknown

A routine multi-day log review found `sensor.solaredge_b1_state_of_energy`
stuck `unavailable` for roughly 31 hours (2026-09-16 14:29 to 2026-09-17
21:37 local) - far beyond the usual 6.5-45s nightly blips already tracked
below. It was isolated to this one entity: `sensor.solaredge_i1_ac_power`,
`select.solaredge_i1_storage_control_mode`, and unrelated automations
(e.g. `ev_charging_discharge_control`) kept running normally throughout,
with only the usual brief reconnect blips. The outage only cleared when
Home Assistant restarted that evening (an unrelated Supervisor
auto-update, 2026.09.0 -> 2026.09.2) - no error in the available logs
explains why the sensor was stuck for so long, or why nothing short of a
full restart recovered it. Not yet correlated with the connectivity
patterns above. The immediate consequence (Layer 4A silently running on a
fallback `soc=0` for the full outage) was fixed the same day - see
known_issues_and_fixes.md - but the cause of the outage itself is a new,
open item here.

Separately, the well-characterized short nightly dropout (6/6 nights,
38-45s each as of the 2026-09-10 check) continues unchanged, with a
noticed pattern: each night's episode lands ~11-13 minutes earlier than
the previous night's (drifting ~22:00 -> ~21:13 over the 6 nights checked
2026-09-04 through 2026-09-09) - a possible clue for the ~22:00 clustering
question above, not yet chased further.

### Update 2026-09-24 - Noise-rate re-check attempted; library appears to have changed underneath the investigation

Attempted the "focused before/after comparison now that the lock-release
fix has landed" suggested above. Two findings, both worth flagging for
whoever continues this:

1. **The Modbus client library itself appears to have changed** since the
   original 2026-08-28 baseline. Every connectivity warning in current
   logs comes from a `tmodbus.transport.async_tcp` logger (e.g. "Received
   unexpected response with Transaction ID: N. Discarding bytes: ..." and
   `custom_components.solaredge_modbus_multi.hub`'s "Coordinator has timed
   out 3 times in a row"). A direct text search for the original baseline's
   exact phrases - `"Cancel send"`, `"Repeating call"`, `"pymodbus"` - found
   **zero** matches anywhere in the currently-retained log window. This
   reads as the `solaredge_modbus_multi` custom integration having switched
   its underlying Modbus client library (pymodbus-family -> a `tmodbus`-
   named one) at some point between late August and now, likely via a
   routine integration update - not something this project changed. The
   new messages appear to be the semantic equivalents of the old ones
   (transaction ID mismatch, retry-exhausted), so a *like-for-like* noise
   comparison is still possible, just not a literal grep-for-the-same-string
   one.

   **Confirmed 2026-09-24 via the integration's GitHub repo**
   (`WillCodeForCats/solaredge-modbus-multi`): this is real and dated.
   Release **v4.0.0** (published 2026-09-14, 10 days before this check)
   adopted Home Assistant's own `modbus-connection` library - "a
   backend-neutral Modbus library with device-modelling" - replacing the
   integration's previous direct `pymodbus` usage (issue #1024 "Adopt the
   modbus-connection library", PR #1077). The `tmodbus.transport.async_tcp`
   logger seen in our logs is that new library's transport layer. Also
   bundled in v4.0.0: a repairable-issue path for Power Control detection
   failures (previously silent), a user-configurable Modbus Request Timeout
   (default 3s, confirmed matching our config's `request_timeout: 3`), and
   "Coordinator timeout scales with configured request timeout" - i.e. the
   "Coordinator has timed out 3 times in a row" message is itself new
   surfacing behavior from this release, not evidence of a new underlying
   failure mode. One correction to a same-day speculation: our config
   (`ha_get_integration`) shows `close_after_polling: false` - this
   installation keeps the Modbus connection open persistently, the
   opposite of the library's now-documented recommended default (closed).
   The frequent "Async TCP connection established/closed" cycles seen in
   logs (9-13 per ~1.5h window) track 1:1 with `script.modbus_queue`
   "Running script sequence" counts in the same windows, not with polling
   - i.e. those are this project's own command-write connections opening
   and closing per queued command, unrelated to the persistent polling
   connection or to the library migration. No config change is indicated
   here.
2. **Rate comparison is inconclusive, and even less meaningful than first
   thought.** HA's retained raw log window only reaches back a few hours
   per query (deep pagination is expensive), and the only stretch of
   2026-09-24 without a restart or the unrelated internet/router outage
   that evening (see below) was 06:45-11:10 local (~4.7h): 2 transaction-
   ID-mismatch events + 5 "Coordinator has timed out" events = 7 real
   connectivity issues, extrapolating to roughly 70/48h - notably higher
   than the original ~34/48h baseline (27+7). Given point 1 above (the
   "Coordinator has timed out" message is new *surfacing* behavior in
   v4.0.0, scaled to the configured request timeout, not necessarily a new
   underlying failure), this comparison is now doubly unreliable: different
   message vocabulary AND a changed detection/reporting mechanism, on top
   of the small-sample extrapolation. It should not be read as a confirmed
   regression - but it does not show
   the improvement one might hope the lock-release fix (known_issues_and_
   fixes.md - Modbus Queue Lock Release Not Exception-Safe) would produce
   on the underlying connectivity noise itself (that fix only addressed the
   stuck-lock *consequence* of a failure, never claimed to reduce the
   failure rate - see known_issues_and_fixes.md, now confirmed deployed
   and working as of 2026-09-24). A cleaner comparison needs a genuine
   quiet 48h window sampled in a few passes going forward, now that a
   post-library-change baseline exists to compare against.

Both the evening's automations reload (~21:24) and a full HA restart
(~22:16, from the 2026-09-23 deployment) reset HA's log buffers, and an
unrelated `router_watchdog` entry logged "internet down since 22:16:53"
recovering around 02:30 the next morning - a WAN/router event, not
Modbus/LAN-specific, and outside this project's scope, but it (along with
the two restarts) makes the first several hours of 2026-09-24's log
window unrepresentative of steady-state noise; only the 06:45-11:10
stretch above was used for the rate estimate.

### Update 2026-09-24 (later same day) - Two integration options changed live: `close_after_polling` and `scan_interval`

Following from point 1 above (SolarEdge inverters accept only one
Modbus/TCP connection at a time, confirmed via the integration's own
wiki), and the observation that `script.modbus_queue`'s own write
connections were competing with the polling coordinator for that single
slot whenever the coordinator held it open persistently:

1. **`close_after_polling`** changed `false` -> `true` at 2026-09-24
   11:52 local (user, via the integration's options UI) - the polling
   coordinator now releases the connection after each poll instead of
   holding it open, matching the integration's own recommended default
   and leaving the slot free for `script.modbus_queue` writes almost all
   the time instead of contending for it.
2. **`scan_interval`** changed `2` -> `5` (seconds) at 2026-09-24 ~12:33
   local (confirmed via `ha_get_integration`) - `close_after_polling:
   true` means every poll now opens and closes a fresh TCP connection
   (the integration's own documented default scan interval is 300s, for
   context), so a 2s interval would have meant ~1800 connect/disconnect
   cycles/hour on top of the single-connection constraint. 5s cuts that
   to ~720/hour while staying well within the reaction-time margin
   needed for the user's real reason for fast polling: per-phase
   measurement and EV-charger load balancing (a separate `EV_charge`
   package, out of scope here) - standard circuit breakers tolerate a
   moderate overload for several seconds to tens of seconds before
   tripping, so a 5s-old reading still leaves real margin.

Both changes are live as of 2026-09-24 and were made specifically as an
intervention in this investigation, not just incidentally observed -
**this is the point to measure a real before/after from**, once enough
quiet time has passed. Everything sampled earlier in this Update
2026-09-24 section (the ~70/48h extrapolation) predates both changes and
should not be blended with data collected after them.

Also found while reviewing SOC-sensor history for the Layer 4A guard
verification (see known_issues_and_fixes.md): two more extended
`sensor.solaredge_b1_state_of_energy` outages beyond the well-known short
nightly blip - 2026-09-20 21:26-21:38 local (~12 min) and 2026-09-22
14:22-16:34 local (~2h13m, daytime, unrelated to the EMHASS addon outage
that started later the same evening). Same open question as the
2026-09-19 finding above (root cause unknown, no explanatory log entry
found) - now three data points of "much longer than the usual 40s blip"
instead of one, which may be a useful pattern once more accumulate. The
Layer 4A guard correctly froze all outputs through the 2h13m one (see
known_issues_and_fixes.md), so the operational risk from these is
covered even though the cause isn't understood.

### Update 2026-10-02 - First genuine 48h sample since the settings change; modest improvement, not a clean win, one new real episode found

A proper 48h window (2026-09-30 21:29 through 2026-10-02 20:45 local, no
restarts/reloads inside it) is now available entirely **after** both the
`close_after_polling` and `scan_interval` changes above - the real
before/after this investigation has been waiting for since 2026-09-24.

1. **Transaction-ID-mismatch rate: ~45.7/48h, down from the ~70/48h
   post-library/pre-settings-change extrapolation, but still above the
   original ~34/48h pre-library baseline.** `tmodbus.transport.async_tcp`
   "Received unexpected response with Transaction ID..." warnings:
   count=45 over a first_occurred-to-latest span of 47.27h, i.e.
   ~45.7/48h extrapolated. Read as: the settings change looks like it
   helped versus the small-sample, inconclusive 70/48h figure taken right
   after the library migration and before the settings change - but the
   comparison back to the *original* 34/48h baseline (27 "Cancel
   send"/"Repeating call"/"No response" + 7 transaction_id mismatch) is
   still not apples-to-apples, per the 2026-09-24 caveats above (different
   library, different message vocabulary, different detection mechanism).
   Net read: probably somewhat better than the immediate post-migration
   state, not clearly better than the original pre-migration baseline -
   this metric alone can't settle whether the underlying connectivity
   noise itself improved, only that it didn't get worse from the settings
   change.
2. **Connection-cycle rate roughly matches the 5s-interval expectation,
   confirming the settings took effect.** A structured error_log search
   for "Async TCP connection established" found 265 occurrences inside a
   ~32-minute raw-log window (2026-10-02 22:02:42-22:34:44), i.e.
   ~8.3/min -> ~496/hour. The 2026-09-24 estimate for the new
   `close_after_polling: true` + `scan_interval: 5s` combination was
   ~720/hour; actual is about 69% of that naive figure (closer to a ~7.3s
   effective cycle than a flat 5s), likely just real-world overhead
   (connect + read + close taking a bit longer than the bare interval)
   rather than anything wrong - but it does confirm the coordinator really
   is opening and closing a connection every poll now, not holding one
   open, so both settings are active and behaving as designed.
3. **Three real connectivity episodes in the 48h window, not two.** The
   previous 24h inverter-behavior review (2026-10-01/02) had found two
   "known, well-handled Modbus blips" (2026-10-01 06:51:58 and 20:43:54).
   This 48h pull found those same two PLUS a third, new one right at the
   edge of the window: a hub-level "Connection failed: could not connect
   to 192.168.10.4:1502" at 2026-10-02 20:45:03 (count=2), alongside the
   coordinator-level "could not connect" message (count=4 total across
   all three episodes). Three episodes across 48h is *not* a worse rate
   than before (2 were already known inside a ~24h sub-window of this same
   48h span) - it's consistent with the existing ~1/24h occasional-blip
   pattern continuing, not escalating. A tight history check around the
   third episode's near-neighbour (18:40-18:55, the closest window with
   good sensor coverage) showed two brief `modbus_busy` on/off cycles
   (~4s, ~18s), no stuck lock, and normal SOC/`effective_storage_mode`
   behaviour throughout - same self-recovering character as the other two
   known episodes, no sign of escalation or a stuck state.
4. **Zero "Coordinator has timed out" messages in the 48h system log.**
   This new-in-v4.0.0 message type (flagged 2026-09-24 as scaled to
   `request_timeout`) did not appear at all in this window - a point in
   favour of the settings change, since this was the message type that
   drove most of the inconclusive 70/48h figure on 2026-09-24.
5. **One new, unrelated one-off**: "Inverter accumulator went backwards;
   this is a SolarEdge bug: AC_Energy_WH_Exported 5491356.0 < 5491357.0"
   (count=1), explicitly flagged by the integration itself as a SolarEdge
   firmware bug. Not part of the connectivity-noise picture and not
   consumed by this project's SOC-based control logic; noted for
   completeness only.
6. Three slow-entity-update warnings (~3s each, on
   `switch.solaredge_i1_negative_site_limit`, `sensor.solaredge_m1_imported_a`,
   `sensor.solaredge_b1_average_temperature`) - minor, consistent with
   normal polling jitter, not flagged as a concern.

**Working verdict**: the 2026-09-24 `close_after_polling`/`scan_interval`
change looks like a modest net positive - connection cycling behaves as
designed, the new timeout-message type has gone quiet, and the
transaction-ID-mismatch rate is down from the immediate post-migration
reading - but it is not a clean "noise solved" result, since that same
metric is still above the original pre-library baseline and the two
baselines aren't strictly comparable. The connectivity-episode rate
itself (the thing that actually matters operationally - brief,
self-recovering "could not connect" events) has not gotten worse; it is
behaving the same as before, including through the new third episode.
Given the library/message-vocabulary problem is permanent (there's no
going back to a like-for-like comparison with the original 34/48h
number), further chasing a cleaner before/after on this specific metric
has diminishing returns. Recommend treating this as adequately resolved
for now - stable, non-escalating, self-recovering - and re-opening only
if episode frequency or severity visibly increases, rather than
continuing to sample 48h windows against an unrecoverable baseline.

### Update 2026-10-06 - Further confirming data point, no re-open needed

While investigating an unrelated question, a structured error_log pull
covering 2026-10-06 20:27-21:07 local showed the same pattern as the
2026-10-02 sample: ~309 connect/close cycles in ~40 minutes (~463/hour,
in the same range as the ~496/hour figure already confirmed as expected
for `close_after_polling: true` + `scan_interval: 5s`), plus exactly one
real connectivity episode ("could not connect to 192.168.10.4:1502" at
20:33:58, self-recovered 21s later at 20:34:19, no stuck lock, no
cascading failure). Consistent with the existing "adequately resolved,
stable, self-recovering" verdict above - not a new finding, just another
data point confirming it has held. No change to the working verdict.

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

### Update 2026-10-06 - Verified against a real overnight event; one real gap found, confirmed currently harmless

A real negative-price event (7 activate/deactivate cycles, 2026-10-06
00:00-08:00 local) gave the first live evidence for this item. See
test_plan.md Test 3.1/3.2/S.3 and known_issues_and_fixes.md - Negative
Price Curtailment - Site Limit Restore Gap for the full write-up.
Summary:

```text
negative export price        -> CONFIRMED: price sign flips matched
                                 by negative_price_active/grid_export_
                                 blocked/switch toggles within one
                                 price-sensor update, 7/7 times.
site limit enabled            -> CONFIRMED: number.solaredge_i1_site_
                                 limit correctly written to 0 on first
                                 activation each time it wasn't already.
site limit value applied      -> STILL NOT DIRECTLY OBSERVED: every
                                 episode so far was overnight with zero
                                 PV, so there was no export attempt for
                                 the limit to actually curtail. Needs a
                                 negative-price episode that coincides
                                 with PV surplus.
export reduced                -> same gap as above - not yet observed.
normal operation restored     -> PARTIALLY CONFIRMED, with a real find:
                                 the switch correctly toggles off, but
                                 negative_site_limit_off
                                 (solaredge_modbusqueue.yaml) never
                                 writes number.solaredge_i1_site_limit
                                 back to anything else - it stays at the
                                 stale "0" indefinitely (confirmed: still
                                 0 nearly 6 hours after the switch went
                                 off and price went positive). Empirically
                                 confirmed harmless today: real grid
                                 export (positive meter readings up to
                                 ~1.1kW, matching PV-surplus-minus-
                                 battery-charging arithmetic) happened
                                 normally during this stale-0 window,
                                 proving the switch - not the number - is
                                 what actually gates enforcement on this
                                 installation.
```

### Update 2026-10-06 (later same day) - Restore-gap item closed via redesign, not the proposed patch

The low-priority cleanup proposed just above (have
`negative_site_limit_off` restore the number to a chosen "normal" value)
was explicitly declined by the user in favor of their own, cleaner
architecture: stop toggling `switch.solaredge_i1_negative_site_limit` per
price change - leave it permanently ON - and vary
`number.solaredge_i1_site_limit` itself between 0 (curtailed) and
1,000,000 W, the number's own max (normal/indifferent export). This
eliminates the restore-asymmetry structurally rather than patching it: the
number is now written on every transition, lifetime, so there is nothing
to forget to restore.

Implemented and deployed to project docs the same day - see
known_issues_and_fixes.md, Negative Price Curtailment - Site Limit Restore
Gap, for the full implementation write-up (changes to
`solaredge_modbusqueue.yaml`, `batterycontrol_scripts.yaml`,
`negative_price_curtailment.yaml`), and architecture.md / entities.md for
the updated mechanism description.

Remaining open items, split out:

1. The one still-unobserved scenario: a negative-price window that
   coincides with real PV surplus, to directly see export actually get
   suppressed (and then resume) rather than inferring it from command
   correctness alone. Still applies unchanged under the new mechanism.
2. The new mechanism itself is not yet observed live through a real
   negative-price cycle - deployed to project docs only as of 2026-10-06.
   Needs a live activate/deactivate cycle to confirm the number toggles
   0 ↔ 1,000,000 correctly and the compare-before-write guard actually
   suppresses redundant writes on the 30-second periodic trigger.
3. Operational: the switch was left "off" by the last deactivation of the
   old design (08:00:16, 2026-10-06) and will only self-heal to "on" the
   next time a negative-price episode fires the new automation's
   defensive re-assert, or when turned on manually in the meantime. Not a
   functional risk on its own (per the empirical finding above), but
   worth confirming once, deliberately, rather than leaving it to chance.

### Update 2026-10-06 (evening) - Item 2 confirmed live

Price stayed non-negative all day (0.03-1.83 SEK/kWh), so this confirms
the normal/restore-path half of item 2, not the negative/curtail half:
the one-time migration write (stale "0" -> 1,000,000 at 15:09:13, the
moment the new YAML first saw price >= 0 and a non-1,000,000 number),
then zero further writes over the next ~5h25m of continuous 30s-cycle
and price-state triggering while the number stayed correctly at
1,000,000 - directly confirmed at the trace level (a real trace showing
the `number != 1000000` condition evaluating false and the branch being
skipped). A ~21s real Modbus outage at 20:33:58 even gave an unplanned
test of the guard's recovery behaviour: the number was correctly
rewritten to 1,000,000 once it came back (having read "unavailable" in
between, which the `| float(-1)` fallback treats as not-yet-1,000,000).
Full detail in test_plan.md Test 3.2's evening update.

Item 2 is now closed for the restore/guard half. Items 1 and 3 remain
open - no negative-price window has occurred yet since deployment, and
the switch is still sitting "off" (confirmed again this evening).

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

Status as of 2026-10-06: all built. See entities.md - Layer 1 - Modbus
Port/Write Health Diagnostics and Modbus Write/Queue Diagnostic Helpers.

```text
binary_sensor.solaredge_modbus_port_raw      - DONE 2026-10-06
binary_sensor.solaredge_modbus_port_stable   - DONE 2026-10-06

input_text.modbus_last_successful_write      - DONE 2026-10-06
input_text.modbus_last_failed_write          - DONE 2026-10-06
input_text.modbus_last_failed_reason         - DONE 2026-10-06
  (close cousin of "last command reason" above, named for what it is)

sensor.solaredge_i1_ac_power_age_seconds     - already existed (2026-09-25,
sensor.solaredge_m1_ac_power_age_seconds       predates this list's last
binary_sensor.solaredge_modbus_data_fresh      edit - correcting a stale
                                                claim here that these were
                                                unbuilt)
```

Also added, beyond this original list: `binary_sensor.
solaredge_modbus_write_health` (consecutive-write-failure based) and
`binary_sensor.solaredge_modbus_queue_backlog` (pending-count based).

None of the port/write-health sensors are a real TCP probe or new
integration - all are derived from existing entity availability, the
pre-existing data-freshness sensor, and the new write/queue counters
above (user's explicit choice: "derive from existing signals" over
building a real port probe). See known_issues_and_fixes.md -
Write-Outcome Telemetry Is Readback-Based for the specific, documented
limitation this implies (a silently-optimistic entity write could read
back as "success" even if the underlying Modbus write failed on the
wire) - these sensors help distinguish the cases below, but with that
caveat in mind, not as ground truth:

```text
TCP port unavailable
existing Modbus session stale
SolarEdge data not updating
write queue overload
pymodbus transaction recovery issue
```

Note 2026-09-19: a `binary_sensor.solaredge_modbus_data_fresh`-style
"age since last update" sensor on `sensor.solaredge_b1_state_of_energy`
specifically would have made the 31-hour outage above visible immediately
instead of only being caught by a manual multi-day history review - this
sensor exists for the inverter/meter AC-power entities (2026-09-25) but
not yet for the battery state-of-energy sensor specifically; still worth
doing if another extended SOC-sensor outage like the 2026-09-16/17 or
2026-09-22 ones recurs.

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
how often it ran, how long commands waited for the busy-lock, or what it
last did. Added: `input_number.modbus_queue_command_count_total`/`_today`
and `modbus_queue_timeout_count_total`/`_today` (today's counters reset
at local midnight via `automation.reset_modbus_queue_daily_counters`,
totals are lifetime); `input_number.modbus_queue_last_wait_seconds`
(time from a command being received to the busy-lock actually clearing,
measured via a `variables:` timestamp against `now()` before and after
the existing `wait_template`) and `modbus_queue_last_execution_seconds`
(lock-acquired to lock-released, i.e. the `choose:` dispatch plus the
2s pacing delay); and `input_text.modbus_queue_last_queued_command`/
`last_executed_command` (`"<queue_service>=<queue_value> @ <timestamp>"`,
stamped on receipt and again just before the lock releases). The timeout
counters increment only when the 10s `wait_template` actually runs out
(`wait.completed == false`) rather than resolving normally - i.e. when
`continue_on_timeout` is what let the run through, not the lock clearing
on its own. See `entities.md` - Layer 4C - Queue Telemetry / Queue
Automations for the full entity list, and `solaredge_modbusqueue.yaml`
for the implementation. Not yet correlated against the Modbus
Connectivity investigation above - that's a natural next step once a
real connectivity episode happens with this telemetry live (e.g. does
`modbus_queue_last_wait_seconds` or the timeout counters spike during a
"could not connect" episode).

**Enhanced Diagnostics - Full Build-Out** (implemented 2026-10-06) - the
remaining open items from section 4 above (Required Telemetry,
Recommended Diagnostic Sensors) were built in one pass, per the user's
explicit request to put the new entities in Layer 1 ("safety and
watchdog") and to derive port/write health from existing signals rather
than a new TCP probe or integration. Added to
`safety_and_watchdog_helpers.yaml` (Layer 1): `input_text.
modbus_last_successful_write`/`_failed_write`/`_failed_reason`;
`input_number.modbus_write_success_count_total`/`_today`,
`_failure_count_total`/`_today`, `_consecutive_failures`,
`modbus_queue_pending_count`, `modbus_queue_last_command_delta`,
`mode_command_skipped_count_total`/`_today`. Added to
`watchdog_sensors.yaml` (Layer 1): `binary_sensor.
solaredge_modbus_port_raw`/`_port_stable` (entity-availability +
data-freshness derived, debounced on the "_stable" variant),
`solaredge_modbus_write_health` (consecutive-failure based),
`solaredge_modbus_queue_backlog` (pending-count based). Added to
`watchdog_automations.yaml`: `automation.reset_watchdog_daily_counters`
(midnight reset for the new "_today" counters). `solaredge_modbusqueue.yaml`
(Layer 4C) now classifies every write as success/failure/unmatched_service
via before/after entity readback and writes into the Layer 1 helpers
above; `apply_effective_battery_control.yaml` (Layer 4B) now increments
the mode-skip counters from the Mode Command Guard's existing `else:`
branch. Two things deliberately NOT built: a real TCP port probe (per
the user's explicit choice of the derive-from-existing-signals option),
and any skip/dedup counter for charge/discharge *limit* writes (no such
skip logic exists yet - see Write Pressure above). Also corrected a
stale claim in section 4 above: the data-freshness sensors
(`sensor.solaredge_i1_ac_power_age_seconds` etc.) already existed since
2026-09-25, predating this build-out - they were mistakenly described as
unbuilt in an earlier pass over this document.

See known_issues_and_fixes.md - Write-Outcome Telemetry Is
Readback-Based for the full design and its documented limitations (the
write-outcome classification is not exception-based and can't fully
distinguish a real wire-level failure from an optimistic local entity
update), and entities.md - Layer 1 for the complete entity list.

Status: implemented in project docs 2026-10-06. Not yet confirmed
against the live file or any real write/outage - no trace or history
evidence collected yet for any of the new counters or derived binary
sensors.

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
helper file placement, the possibly-unused safety-override booleans,
and the leftover debug logging in `solaredge_modbusqueue.yaml`) plus one
known_issues_and_fixes.md update (EV Charging Sensor Swap - stale
trigger entity_id follow-up). No new known_issues_and_fixes.md,
README.md, or architecture.md *entries* were needed beyond that one
update - existing entries already covered essentially everything found
inline.

---

Implemented fixes and known issues: see `known_issues_and_fixes.md`.