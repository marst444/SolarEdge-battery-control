# Standard test blueprint:

## Test Timestamp

Now:
{{ now() }}

## Forecast / EMHASS

MPC Optim Status:
{{ states('sensor.mpc_pv_optim_status') }}

MPC Batt SOC:
{{ states('sensor.mpc_pv_batt_soc') }}

MPC Batt Power:
{{ states('sensor.mpc_pv_batt_power') }}

MPC Grid Power:
{{ states('sensor.mpc_pv_grid_power') }}

SOC Target:
{{ states('input_number.soc_target') }}

SOC Forecast Smooth:
{{ states('sensor.soc_batt_forecast_smooth') }}

## PV Surplus

PV Surplus:
{{ (
  states('sensor.solar_panel_production_w')|float(0)
  -
  states('sensor.power_myhouse_load_no_var_loads')|float(0)
) | round(0) }}


## Battery State

Current SOC:
{{ states('sensor.solaredge_b1_state_of_energy') }}

SOC Difference Current - Target:
{{ (
  states('sensor.solaredge_b1_state_of_energy') | float(0)
  -
  states('input_number.soc_target') | float(0)
) | round(2) }}

SOC Deadband:
{{ states('input_number.soc_deadband') }}

Dynamic Charge Limit:
{{ states('sensor.dynamic_storage_charge_limit') }}

Dynamic Discharge Limit:
{{ states('sensor.dynamic_storage_discharge_limit') }}

Helper Dynamic Charge:
{{ states('input_number.dynamic_charge_limit') }}

Helper Dynamic Discharge:
{{ states('input_number.dynamic_discharge_limit') }}


## Physical Power / Loads

PV Production:
{{ states('sensor.solar_panel_production_w') }}

House Load No Var Loads:
{{ states('sensor.power_myhouse_load_no_var_loads') }}

House Load No Var Loads No EV:
{{ states('sensor.power_myhouse_load_no_var_loads_no_ev') }}

EV Charging Power:
{{ states('sensor.ev_charging_power_w') }}

EV Charging On:
{{ states('binary_sensor.ev_charging_on') }}


## SolarEdge / Grid Meter

Inverter AC Power I1:
{{ states('sensor.solaredge_i1_ac_power') }}

Meter AC Power M1:
{{ states('sensor.solaredge_m1_ac_power') }}

SolarEdge Data Fresh:
{{ states('binary_sensor.solaredge_modbus_data_fresh') }}

I1 AC Power Age:
{{ states('sensor.solaredge_i1_ac_power_age_seconds') }}

M1 AC Power Age:
{{ states('sensor.solaredge_m1_ac_power_age_seconds') }}

M1 Interpretation:
{% set m1 = states('sensor.solaredge_m1_ac_power') | float(0) %}
{% if m1 > 0 %}
Import / grid to site: {{ m1 }} W
{% elif m1 < 0 %}
Export / site to grid: {{ m1 }} W
{% else %}
Neutral / zero flow
{% endif %}

## Prices / Grid Constraints

Import Price:
{{ states('sensor.total_import_price') }}

Export Price:
{{ states('sensor.total_export_price') }}

Negative Price Active:
{{ states('input_boolean.negative_price_active') }}

Grid Export Blocked:
{{ states('input_boolean.grid_export_blocked') }}

Grid Export Limited:
{{ states('input_boolean.grid_export_limited') }}

Negative Site Limit:
{{ states('switch.solaredge_i1_negative_site_limit') }}

Site Limit:
{{ states('number.solaredge_i1_site_limit') }}


## Requested Decision / Layer 4A

Requested Storage Mode:
{{ states('input_select.emhass_requested_storage_mode') }}

Requested Charge Limit:
{{ states('input_number.emhass_requested_charge_limit') }}

Requested Discharge Limit:
{{ states('input_number.emhass_requested_discharge_limit') }}


## Effective Decision / Safety Layer

Effective Storage Mode:
{{ states('input_select.effective_storage_mode') }}

Effective Charge Limit:
{{ states('input_number.effective_charge_limit') }}

Effective Discharge Limit:
{{ states('input_number.effective_discharge_limit') }}

Effective Reason:
{{ states('input_text.effective_battery_reason') }}

Battery Control Enabled:
{{ states('input_boolean.effective_battery_control_enabled') }}


## SolarEdge Apply / Layer 4D

Storage Control Mode:
{{ states('select.solaredge_i1_storage_control_mode') }}

Storage Command Mode:
{{ states('select.solaredge_i1_storage_command_mode') }}

Storage Command Timeout:
{{ states('number.solaredge_i1_storage_command_timeout') }}

Storage Charge Limit:
{{ states('number.solaredge_i1_storage_charge_limit') }}

Storage Discharge Limit:
{{ states('number.solaredge_i1_storage_discharge_limit') }}

Modbus Busy:
{{ states('input_boolean.modbus_busy') }}