# SolarEdge-battery-control
Battery management for SolarEdge using EMHASS and ModbusMulti

Aim: Optimize the use of PV (photovoltaic) power generation and battery usage from a profit point. The Optimization by EMHASS based on:

a) energy price forecast (using Nordpool spotprices 15min)
b) solar PV generation forecast (using Solcast)
c) energy usage in the home (energy usage history)

EMHASS creates an optimization blueprint for battery charge/discharge and grid import/export.

The charge/discharge logic take into account actual load, PV generation and battery SOC(state of charge) and follows the blueprint.

Commands to inverter/ battery are fed through a modbus queue and are using the powercontrol options in the SolarEdge modbus multi.

Prerequisits: 

#1 SolarEdge Modbus Multi : [https://github.com/WillCodeForCats](https://github.com/WillCodeForCats/solaredge-modbus-multi)

#2 Nordpool integration: https://github.com/custom-components/nordpool/

#3 Solcast integration: https://github.com/BJReplay/ha-solcast-solar

#4 EMHASS: https://github.com/davidusb-geek/emhass

Basic installation according to each separate guide

After installation of the integrations there has to be some additional templating for the setup to work.

We need:

# Template Sensors from the Modbus Multi integration
  - Sensor for PV-production (mine is called sensor.solar_panel_production_w)
  - I have used a modification of ryanm101energy_stats (https://github.com/ryanm101/hasolarcfg/blob/master/packages/energy_utilities.yaml)
  - Sensor named sensor.power_myhouse_load_no_var_loads (a sensor with house load, without variable loads. I have included battery discharge to house in load, but not charge or discharge to grid. Others would argue to totally  leave the battery outside the "no_var_loads" sensor.

# Set up EMHASS
  - Template for modifying the nordpool price for gridcharging/ grid discharging
  - Template to modify the solcast forecast to 15min and allign with nordpool prices
  - Restcommand/shellcommand to inject the modifyed data to EMHASS during runtime
  - Automation to run the dayahead optimization and publish back to Home assistant

# Template a modbus queue
The modbus queue hinders multiple commands to be sent over the modbus simultaneously, but rather be passed one by one [(https://github.com/marst444/SolarEdge-battery-control/blob/main/4.Real_time_execution/solaredge_modbusqueue.yaml)]
# Template a charge/discharge logic
The charge/discharge logic is the core of the templating. 
- EMHASS publish sensors for forecast of battery and grid charge/discharge
- Some battery control sensors and helpers are created for the logic to control and follow the forecasted chargecurve ([https://github.com/marst444/SolarEdge-battery-control/tree/main/2.Optimization/EMHASS])
- Scripts are created to send commands to the modbus queue (inverter)
  [(https://github.com/marst444/SolarEdge-battery-control/blob/main/4.Real_time_execution/batterycontrol_scripts.yaml)]
- An automation to evaluate current PV production/ EVcharging and the EMHASS forecast is needed.
[(https://github.com/marst444/SolarEdge-battery-control/blob/main/4.Real_time_execution/batterycontrol_automations.yaml)]
  
I take no responsibility for the use of the charging/discharging logic!

# SolarEdge Battery Control — Automated Control Logic

System Architecture — 4-Layer Control Model

The control system is inspired by hierarchical control theory and organized into four distinct layers, each with a specific responsibility. Higher layers set constraints and targets; lower layers execute them in real time. The maps and files are NOT yet always moved into the right "map/file structure", and can be hidden within another file. Work in progress for easier overview and debug.

```
┌─────────────────────────────────────────┐
│         1. SAFETY & WATCHDOG            │  Hard limits — always enforced and watchdog
├─────────────────────────────────────────┤
│         2. OPTIMIZATION                 │  EMHASS forecast and SOC targets
├─────────────────────────────────────────┤
│         3. GRID CONSTRAINTS             │  Curtailment and export limits
├─────────────────────────────────────────┤
│         4. REAL-TIME EFFECTUATION       │  Modbus, scripts, SolarEdge modes
└─────────────────────────────────────────┘
```
# Layer 1 — Safety

Hard boundaries that override all other logic. These are checked first in every automation cycle and cannot be overridden by optimization or grid signals.


input_number.minimum_state_of_charge — stops battery discharging below this SOC
input_number.maximum_state_of_charge — stops battery charging above this SOC
Low-SOC recovery: if SOC < 10%, force charge from solar and grid regardless of price or forecast
Emergency discharge stop: if SOC < min, discharge limit is set to 0 immediately


# Layer 2 — Optimization

EMHASS (Energy Management Home Assistant System) runs a model predictive optimization (MPC) every 15 minutes using weather forecasts, electricity prices (5d combined Nordpool + padded EPEX), and a Machinetrained load forecast (ML-forecast). The MPC optimization produces a 48 hour long forecast with sliding window (moving forward each run):

sensor.mpc_pv_batt_soc — forecasted battery SOC schedule for the coming hours
sensor.mpc_pv_batt_power - forecast battery chargin (negative) or discharging (positive) 
sensor.mpc_pv_grid_power - forecast grid import (positve) or export (negative)

Price-based fallback: if no EMHASS signal, compare current import price to input_number.average_last_chargingprice to decide whether grid charging makes economic sense

Automations then create:
input_number.soc_target — the next SOC target derived from the forecast
sensor.dynamic_storage_charge_limit — proportional charge power based on distance to SOC target
sensor.dynamic_storage_discharge_limit — proportional discharge power based on distance to SOC target

# Layer 3 — Grid Constraints

Handles obligations and limitations imposed by the grid connection or utility tariffs, independent of optimization goals.

Negative export price curtailment: if the spot price goes negative, grid export is curtailed via: 

switch.solaredge_i1_negative_site_limit
number.solaredge_i1_site_limit

This layer can override Layer 2 decisions — even if EMHASS wants to export, negative prices mean it is not permitted or not economical

When export/site limiting is enabled and the site limit is set to 0 W, the inverter should prevent net export from the site. This applies to the whole installation at the grid connection point, not specifically to the battery.


# Layer 4 — Real-Time Effectuation

Translates the decisions from Layers 1–3 into actual inverter commands via SolarEdge Modbus. All commands pass through a Modbus queue to prevent communication errors from overlapping writes. 


script.modbus_queue — serializes all inverter commands with a busy-lock mechanism
Storage Command Mode — sets the operational mode of the inverter (see modes below)
Storage Charge Limit / Discharge Limit — fine-grained power control, can be updated at any time independent of the 15-minute mode cycle
Storage Command Timeout — set to 900 seconds (15 min) (aligned with the EMHASS optimization interval),


# Key design principle: 

Mode changes happen every 15 minutes, and has to be followed by setting a prospective duration for the command (storage command timeout in seconds).  BUT Power limits can be adjusted at any time, and does not need the timeout to be set — for example when EV charging starts or stops — allowing real-time fine-grained control within the broader mode set by the automation cycle.


# Storage Control Modes Explained

This automation uses Remote Control mode via SolarEdge Modbus Multi [(https://github.com/WillCodeForCats/solaredge-modbus-multi)], which allows setting both the Storage Command Mode and power limits dynamically.

# Available Command Modes

## Solar Power Only (Off)
No battery charging or discharging. PV production goes directly to self-consumption and grid export. Battery is idle.
```
PV → House
PV → Export

(Battery idle)
```
→ Used when: Battery use is disabled, any PV is used for self-consumption and surplus for export.

**Caveat found 2026-11-03** (see known_issues_and_fixes.md - High SOC
Hold - Charge Limit = 0 Does Not Stop Charging While Maximize Self
Consumption Stays Active, the 2026-11-03 Updates): the "No battery
charging or discharging... Battery is idle" claim above was confirmed
live to NOT hold while meaningful PV surplus is available - the battery
charged at ~350-430W (tracking PV almost exactly) with a confirmed-
correct mode command and the charge-limit register held at 0
throughout. This is the same failure shape already documented under
Maximize Self Consumption below - this mode's internal logic appears to
also divert PV surplus into the battery regardless of the charge-limit
register. A possible device/entity mixup was specifically checked for
and ruled out. High SOC Hold no longer uses this mode for that reason -
see Discharge to Maximize Export below.

## Charge from Clipped Solar Power (Charge Excess PV power only)
Only when PV production exceeds the inverter's maximum AC output (house load + grid export) (site limit? Är inte site limit export limit??), the PV surplus is redirected to charge the battery. Battery does not compete with self-consumption or grid export.
```
PV
↓
House + Export
↓
Battery
```

→ Used when: PV > inverter AC limit AND SOC < max.



## Charge from Solar Power (Charge from PV first)
PV production charges the battery first, until it is full. Only then does PV contribute to self-consumption and grid export. Grid is not used for charging.
```
PV
↓
Battery
↓
House + Export
```
→ Not used in this automation (would starve house load during charging).

## Charge from Solar Power and Grid (Charge from PV and AC)
Battery is charged from PV and grid as needed, until full. Allows aggressive charging regardless of house load.

```
If PV > charge limit
PV
↓
Battery (up to charge limit)
↓
House + Export

If PV < Charge limit
PV+ Grid
↓
Battery (up to charge limit)
```
→ Used when: SOC is below target and EMHASS forecasts charging, or SOC is critically low.

## Discharge to Maximize Export
When PV production is below the inverter's maximum AC output, the battery discharges to supplement PV so that total output (PV + battery) reaches the inverter power limit, maximizing grid export.
```
PV+ Battery
↓
Houseload + Export (as set by inverter power limit)
```

→ Used when: EMHASS forecasts export (grid_fc < 0) AND battery discharge (batt_fc > 0) AND SOC > min; also used during High SOC Hold as of 2026-11-03 (see SOC Safety Limits below).

**Confirmed 2026-11-03**: unlike Solar Power Only and Maximize Self
Consumption (see their caveats above), this mode was checked live under
real PV surplus with charge limit = 0 and correctly stayed at 0W/Standby
- it does not charge from PV surplus. See known_issues_and_fixes.md,
High SOC Hold - Charge Limit = 0 Does Not Stop Charging..., for the
evidence.

## Discharge to Minimize Import
Battery discharges only to cover house load when PV is insufficient — no grid export from battery. Requires external meter.
```
Battery
↓
House loads

NO grid export
```

→ Not currently used by this automation. Briefly used for High SOC Hold
for part of 2026-11-02 (added after Maximize Self Consumption was
confirmed to keep charging past charge limit = 0 under PV surplus), but
replaced the same day by a day/night Solar-Power-Only/Discharge-to-
Maximize-Export split before this mode's own charge-blocking behavior
under PV surplus was ever actually tested - see known_issues_and_fixes.md,
High SOC Hold - Charge Limit = 0 Does Not Stop Charging..., for the full
history. Whether it would have blocked that charging remains unconfirmed
either way.

## Maximize Self Consumption
PV covers house load first. Any surplus charges the battery. If PV is insufficient, the battery discharges to cover the gap. Net result: grid import and export are minimized. Requires external meter.

In this setup, Storage Charge Limit and Storage Discharge Limit are used as real-time power caps while Maximize Self Consumption remains active. This allows fast modulation of battery behavior without changing the storage command mode.

Both limits at dynamic values → full EMHASS-guided charge/discharge
Charge limit = 0 → battery will only discharge, not charge
Discharge limit = 0 → battery will only charge, not discharge
Both limits = 0 → battery is effectively idle (similar to Solar Power Only, but mode stays active for faster limit adjustments)
```
a) PV > house load

PV
↓
House load
↓
Battery
↓
Grid Export

b) PV < house load

PV + Battery 
↓
House load
```

→ Default mode for maintain zone, self-consumption optimization, and fine-grained real-time control.

**Caveat found 2026-11-01** (see known_issues_and_fixes.md - High SOC Hold -
Charge Limit = 0 Does Not Stop Charging While Maximize Self Consumption
Stays Active): the "Charge limit = 0 → battery will only discharge, not
charge" line above was confirmed live to NOT hold while meaningful PV
surplus is available - the battery kept charging to 100% SOC for 6h45m+
with the charge-limit register confirmed held at 0 throughout. This mode's
own internal self-consumption logic evidently overrides the charge-limit
register in that condition. As of 2026-11-03, High SOC Hold uses
Discharge to Maximize Export instead, the only mode confirmed so far to
honor charge_limit = 0 under real PV surplus (Solar Power Only was tried
in between and found to have the same defect - see the caveat under
Solar Power Only above) - see Discharge to Maximize Export below.

# SolarEdge default/backup mode

select.solaredge_i1_storage_default_mode configures the mode SolarEdge falls
back to internally once a Remote Control command's timeout lapses without a
refreshed command. In this setup it is kept as Maximize Self Consumption,
matching the mode used above for maintain-zone/self-consumption control —
this lets Layer 4B's Mode Command Guard (see architecture.md) safely skip
resending an unchanged mode command while staying in that mode.


# Storage Power Limits

Storage Discharge Limit — Maximum battery discharge rate (W). Setting to 0 stops discharging immediately without changing mode.

Storage Charge Limit — Maximum battery charge rate (W). Setting to 0 stops charging immediately without changing mode.


# Decision Logic

The main automation (emhass_battery_forecast_control) runs every 15 minutes and evaluates the following variables:

VariableDescriptionsocCurrent battery state of charge (%)targetEMHASS forecasted SOC target for next intervalpvCurrent PV production (W)house_loadHouse load excluding variable loads like EV (W)batt_fcEMHASS battery power forecast (<0 = charge, >0 = discharge)grid_fcEMHASS grid power forecast (<0 = export, >0 = import)ac_limInverter AC power limit (W)soc_min / soc_maxConfigured min/max SOC safety limits (%)


# Control Decisions (in priority order)

# 1. SOC Safety Limits (Layer 1)

SOC > max (e.g. 90%)
Battery is full enough — stop charging. High SOC Hold then splits by whether
PV is actually present:
→ Mode: Discharge to Maximize Export, charge limit = 0, always
→ Discharge limit: 0 W by day (PV present - stay as close to idle as this
mode allows), dynamic value by night (no PV, no recharge risk - discharge
toward export instead of sitting idle)
(Changed 2026-11-03, after Solar Power Only - itself a replacement for
Maximize Self Consumption, changed 2026-11-02 - was confirmed to also charge
from PV surplus despite charge limit = 0. See the caveats under Solar Power
Only and Maximize Self Consumption above, and known_issues_and_fixes.md for
the full history. Not yet verified live against a real Hold cycle under
this latest design.)

SOC < min (e.g. 20%)
Battery is too low — stop discharging.
→ If SOC < 10% (critical): Charge from Solar Power and Grid at full power — recovery mode
→ If SOC 10–20%: Charge from Solar Power and Grid with dynamic charge limit


# 2. Clipped Solar (Layers 2 + 4)

PV is producing more than the inverter can export or use for self-consumption. Redirect the clipped surplus into the battery.
→ Mode: Charge from Clipped Solar Power, dynamic charge and discharge limits


# 3. Below SOC Target (Layers 2 + 4)

SOC is below the EMHASS forecasted target. The system expects charge, but how depends on conditions:

A) 
PV > house_load AND grid_fc ≥ 0 (import/neutral forecast)
Sun covers house load. Let surplus charge the battery via self-consumption logic. Prioritize charging from Solar, and handle variations in load by using battery.
→ Mode: Maximize Self Consumption, dynamic charge and discharge limits

## Edge case, PV is not enough to catch up to SOC target, then change to Charge from solar and grid? When?

C)
PV > house_load AND grid_fc < 0 (export forecast)
SOC is below target, but EMHASS forecasts grid export. The automation allows PV surplus to be exported, but protects the battery from being discharged because SOC still needs to recover.

→ Mode: Discharge to Maximize Export
→ Discharge limit: 0 W
→ Charge limit: dynamic

This means export is allowed from PV surplus, but the battery is kept out of export because it is below target.
D)
PV ≤ house_load AND PV > 0 AND batt_fc ≤ 0 (EMHASS says charge)
Sun partially covers load. EMHASS forecasts charging — allow grid to supplement.
→ Mode: Charge from Solar Power and Grid, dynamic charge limit

No PV AND batt_fc < 0 AND grid_fc > 0
Night or cloudy. EMHASS explicitly forecasts grid charging.
→ Mode: Charge from Solar Power and Grid, dynamic charge limit

No PV AND batt_fc = 0 AND price ≤ 120% of average charge price
No EMHASS forecast signal, but current import price is acceptable. Opportunistic grid charging.
→ Mode: Charge from Solar Power and Grid, dynamic charge limit

No PV AND batt_fc = 0 AND price > 120% of average charge price (or the price sensor is unavailable)
Price has risen significantly, or price data is missing — don't charge at a premium. Hold.
→ Mode: Solar Power Only, charge limit = 0, discharge limit = 0
(Corrected 2026-11-02 - this branch does not use Maximize Self Consumption live; see known_issues_and_fixes.md, PV Sensor Trickle Readings Bypass the "No PV" Branch Logic.)


# 4. Discharge to Maximize Export (Layers 2 + 3 + 4)

SOC is at or above target, EMHASS forecasts export (grid_fc < 0) and battery discharge (batt_fc > 0). It is profitable to push battery energy to the grid — provided grid constraints (Layer 3) permit export.
→ Mode: Discharge to Maximize Export, charge limit = 0, dynamic discharge limit


# 5. Discharge for Self Consumption (Layers 2 + 4)

SOC is above target, grid forecast is import or neutral, battery forecast is not charging. Use battery to cover house load and avoid grid import.
→ Mode: Maximize Self Consumption, dynamic charge and discharge limits


# 6. Maintain Zone (Layers 2 + 4)

SOC is within the deadband around target (target ± deadband%). Hold position, let EMHASS and self-consumption logic guide fine adjustments.

PV > house_load AND SOC below upper edge of deadband
Sun is available — absorb surplus into battery at full charge rate.
→ Mode: Maximize Self Consumption, charge limit = 5000W, dynamic discharge limit

PV ≤ house_load OR batt_fc < 0
Normal maintain — balance dynamically.
→ Mode: Maximize Self Consumption, dynamic charge and discharge limits

Fallback (reached only when PV > house_load AND batt_fc >= 0 AND SOC sits
exactly at the deadband's upper edge)
No clear signal, but real PV surplus is present — stay as close to flat as
possible without trusting Solar Power Only's charge-blocking under PV
surplus (see its caveat above).
→ Mode: Discharge to Maximize Export, charge limit = 0, discharge limit = 0
(Changed 2026-11-03 from Solar Power Only - this was the one other place in
the automation besides High SOC Hold where that mode's now-confirmed defect
could actually be triggered by real PV surplus. See known_issues_and_fixes.md.)


# Dynamic Charge/Discharge Limits

Charge and discharge power limits are calculated from the EMHASS battery SOC forecast: a weighted blend of the next three forecasted SOC points (weights 0.5/0.3/0.2, nearest-weighted-heaviest - the same weighting used by `sensor.soc_batt_forecast_smooth` for the SOC target) rather than only the single next forecast point. The difference between that blended target and the current SOC, converted to watts based on battery capacity (4.6 kWh), gives a proportional power limit. This means the battery charges or discharges faster when far from target and slower as it approaches it — a simple proportional control that works alongside the mode logic. Widening the blend from one point to three (2026-11-02) lets a genuinely sustained multi-step plan pull the limit further from the current SOC than a single point would, while still damping a single noisy MPC re-plan toward the average - see known_issues_and_fixes.md (Dynamic Discharge Limit Under-Delivers vs. EMHASS Plan During Sustained Price Spikes). Not yet verified against a real live event.

These limits update independently of the 15-minute mode cycle (see Key design principle above) - e.g. responding immediately to EV charging starting or stopping, without waiting for the next automation trigger.




# Big thanks to:
@ryanm101 (https://github.com/ryanm101)
@wilcodeforcats (https://github.com/WillCodeForCats)
@kipe (https://github.com/kipe)
@BJReplay( https://github.com/BJReplay)
@davidusbgeek (https://github.com/davidusb-geek)
@BrendanAnnable — EMHASS energy-plan scripts (emhass_scripts.yaml) are a
modification of his SiegenEnergy script (https://sigenergy.annable.me/)