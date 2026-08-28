# SolarEdge Battery Control Entities

This document lists the core entities used by the SolarEdge Battery Control package.

Entities are grouped by architecture layer.

---

# Layer 1 - Safety and Watchdog

Layer 1 defines hard safety limits, watchdog state and fail-safe overrides.

This layer protects the battery and provides health signals for the rest of the control system.

---

## Safety Limit Helpers

```text
input_number.minimum_state_of_charge
-> Minimum allowed battery SOC.
```

---

```text
input_number.maximum_state_of_charge
-> Maximum allowed battery SOC.
```

---

```text
input_number.ac_maxpower
-> Configured maximum AC power limit.
```

---

```text
input_number.safety_min_discharge_limit
-> Minimum discharge limit used by safety logic.
```

---

```text
input_number.safety_recovery_charge_limit
-> Charge limit used during safety recovery.
```

---

```
input_boolean.battery_high_soc_hold
-> Upper SOC protection state with hysteresis
```

## EMHASS Watchdog Helpers

```text
input_datetime.emhass_last_dayahead_run
-> Timestamp of last EMHASS day-ahead run.
```

---

```text
input_datetime.emhass_last_mpc_run
-> Timestamp of last EMHASS MPC run.
```

---

```text
input_datetime.emhass_dayahead_last_success
-> Timestamp of last successful EMHASS day-ahead optimisation.
```

---

```text
input_datetime.emhass_mpc_last_success
-> Timestamp of last successful EMHASS MPC optimisation.
```

---

```text
input_text.emhass_mpc_status
-> Human-readable EMHASS MPC status.
```

---

```text
input_text.emhass_dayahead_status
-> Human-readable EMHASS day-ahead status.
```

---

## Effective Battery Control

```text
input_text.effective_battery_reason
-> Human-readable explanation of the current effective battery control decision.
```

---

```text
input_boolean.battery_safety_override_active
-> Enables or indicates battery safety override state.
```

---

```text
input_boolean.battery_force_discharge
-> Manual battery force-discharge control.
```

---

```text
input_boolean.battery_force_charge
-> Manual battery force-charge control.
```

---

```text
input_boolean.battery_discharge_blocked
-> Indicates that battery discharge is blocked.
```

---

```text
input_boolean.battery_charge_blocked
-> Indicates that battery charging is blocked.
```

---

```text
input_boolean.effective_battery_control_enabled
-> Main kill switch for battery control.
```

---

## Effective Battery Control Outputs

```text
input_select.effective_storage_mode
-> Final effective SolarEdge storage mode after safety evaluation.
```

---

```text
input_number.effective_charge_limit
-> Final effective battery charge limit.
```

---

```text
input_number.effective_discharge_limit
-> Final effective battery discharge limit.
```

---

## Watchdog Sensors

```text
sensor.emhass_health
-> Aggregated EMHASS health status.
```

---

```text
binary_sensor.emhass_healthy
-> Binary EMHASS health indicator.
```

---

## SolarEdge Data Freshness Diagnostics

```text
sensor.solaredge_i1_ac_power_age_seconds
-> Seconds since inverter AC power last updated.
```

---

```text
sensor.solaredge_m1_ac_power_age_seconds
-> Seconds since meter AC power last updated.
```

---

```text
binary_sensor.solaredge_modbus_data_fresh
-> Indicates whether SolarEdge Modbus data is being updated.
```

---

## Automations

```text
automation.calculate_effective_battery_control
-> Calculates effective battery mode and limits after safety evaluation.
```

---

```text
automation.EMHASS_watchdog_dayahead_missing
-> Detects missing EMHASS day-ahead optimisation.
```

---

```text
automation.EMHASS_watchdog_mpc_stalled
-> Detects stalled EMHASS MPC optimisation.
```
---

# Layer 2 - Optimisation

Layer 2 prepares forecast data, runs EMHASS optimisation and publishes optimisation results back into Home Assistant.

---

## EMHASS Control Helpers

```text
input_number.mpc_window_start
-> Start timestep for MPC sliding window.
```

---

```text
input_number.mpc_window_end
-> End timestep for MPC sliding window.
```

---

```text
input_number.mpc_prediction_horizon
-> Prediction horizon used by MPC optimisation.
```

---

```text
input_number.mpc_soc_final_target
-> Final battery SOC target used by MPC.
```

---

```text
input_number.emhass_requested_charge_limit
-> Charge power limit requested by EMHASS.
```

---

```text
input_number.emhass_requested_discharge_limit
-> Discharge power limit requested by EMHASS.
```

---

```text
input_number.emhass_active_deferrable_loads
-> Number of active EMHASS deferrable loads.
```

---

```text
input_datetime.day_ahead_forecast_end
-> End timestamp of the day-ahead forecast window.
```

---

```text
input_select.emhass_requested_storage_mode
-> Storage mode requested by EMHASS optimisation.
```

---

## EMHASS Automations

```text
automation.EMHASS_ML_forecast_fit
-> Runs EMHASS machine-learning forecast model fitting.
```

---

```text
automation.EMHASS_ML_forecast_predict
-> Runs EMHASS machine-learning load forecast prediction.
```

---

```text
automation.generate_EMHASS_energy_plan_mpc_pv
-> Runs EMHASS MPC optimisation every 15 minutes.
```

---

```text
automation.generate_EMHASS_energy_plan_dh_pv
-> Runs EMHASS day-ahead optimisation.
```

---

## EMHASS Scripts

```text
script.generate_emhass_energy_plan_mpc_pv
-> Builds MPC payload, runs EMHASS MPC optimisation and publishes MPC sensors.
```

---

```text
script.generate_emhass_energy_plan_dh_pv
-> Builds day-ahead payload, runs EMHASS day-ahead optimisation and publishes day-ahead sensors.
```

---

```text
script.run_mlforecast_fit_load
-> Calls EMHASS load forecast model fitting endpoint.
```

---

```text
script.run_mlforecast_predict_load
-> Calls EMHASS load forecast prediction endpoint.
```

---

```text
script.emhass_update_sliding_window
-> Updates MPC sliding window helper values.
```

---

```text
script.set_mpc_soc_final_target
-> Updates final MPC SOC target from the day-ahead battery SOC forecast.
```

---

## EMHASS REST Commands

```text
rest_command.emhass_dayahead_optim
-> Calls local EMHASS day-ahead optimisation endpoint.
```

---

```text
rest_command.emhass_naive_mpc_optim
-> Calls local EMHASS naive MPC optimisation endpoint.
```

---

```text
rest_command.emhass_publish_data
-> Publishes EMHASS optimisation results back to Home Assistant.
```

---

```text
rest_command.publish_data
-> Calls EMHASS publish-data endpoint on remote EMHASS host.
```

---

```text
rest_command.mlforecast_fit_load
-> Calls EMHASS load forecast model fit endpoint.
```

---

```text
rest_command.mlforecast_predict_load
-> Calls EMHASS load forecast prediction endpoint.
```

---

## EMHASS Shell Commands

```text
shell_command.trigger_nordpool_forecast
-> Sends Nordpool price forecast payload to EMHASS day-ahead optimisation endpoint.
```

---

## House Load Sensors

```text
sensor.power_myhouse_load_no_var_loads
-> House load excluding variable loads.
```

---

```text
sensor.power_myhouse_load_no_var_loads_no_ev
-> House load excluding variable loads and EV charging.
```

---

## Import Price Forecast Sensors

```text
sensor.nordpool_se3_total_costs_import
-> Nordpool import price including grid fee, tax, Tibber fee and VAT.
```

---

```text
sensor.epex_total_costs_import_forecast
-> EPEX import price forecast including grid fee, tax, Tibber fee and VAT.
```

---

```text
sensor.merged_total_costs_import_forecast
-> Combined Nordpool and EPEX import price forecast for EMHASS.
```

---

## Export Price Forecast Sensors

```text
sensor.nordpool_se3_total_revenue_export
-> Nordpool export revenue including export compensation.
```

---

```text
sensor.epex_total_revenue_export_forecast
-> EPEX export revenue forecast including export compensation.
```

---

```text
sensor.merged_total_revenue_export_forecast
-> Combined Nordpool and EPEX export revenue forecast for EMHASS.
```

---

## PV Forecast Sensors

```text
sensor.solcast_pv_forecast_15min_intervals
-> Solcast PV forecast converted to 15-minute intervals.
```

---

```text
sensor.solcast_pv_forecast_30min_intervals
-> Solcast PV forecast converted to 30-minute intervals and split into 15-minute blocks.
```

---

## Load Forecast Sensors

```text
sensor.p_load_forecast_ml_pv
-> EMHASS machine-learning load forecast used by MPC and day-ahead optimisation
```

---

# Layer 3 - Grid Constraints

Layer 3 handles grid-related restrictions and export constraints.

---

## Grid Constraint Helpers

```text
input_boolean.grid_export_blocked
-> Grid export is blocked.
```

---

```text
input_boolean.grid_export_limited
-> Grid export is limited.
```

---

```text
input_boolean.negative_price_active
-> Negative export price condition is active.
```

---

## Grid Constraint Inputs

```text
sensor.total_export_price
-> Current total export price used to determine export curtailment.
```

---

## SolarEdge Export Control

```text
switch.solaredge_i1_negative_site_limit
-> SolarEdge export/site limit control used during negative prices.
```

---

## Automations

```text
automation.negative_price_curtailment
-> Enables or removes export curtailment based on export price.
```

---

## Scripts

```text
script.turn_negative_site_limit_on_script
-> Activates SolarEdge export blocking.
```

---

```text
script.turn_negative_site_limit_off_script
-> Removes SolarEdge export blocking.
```

---

```text
script.maximize_self_consumption_script
-> Switches inverter operation to self-consumption mode.
```

---

```text
script.set_storage_discharge_limit0
-> Blocks battery discharge by setting discharge limit to zero.
```
---

# Layer 4A - Decision Engine

Layer 4A translates EMHASS forecasts into requested battery behaviour.

---

## Decision Helpers

```text
input_number.dynamic_charge_limit
-> Dynamic battery charge power target.
```

---

```text
input_number.dynamic_discharge_limit
-> Dynamic battery discharge power target.
```

---

```text
input_number.last_charge_limit
-> Last valid dynamic charge limit.
```

---

```text
input_number.last_discharge_limit
-> Last valid dynamic discharge limit.
```

---

```text
input_number.soc_target
-> Current battery SOC target.
```

---

```text
input_number.soc_deadband
-> SOC deadband around target SOC.
```

---

```text
input_number.grid_charge_export_cooldown_minutes
-> Minutes that must elapse after a charge_from_solar_and_grid command
   before discharge_to_maximize_export is allowed to fire again. Default
   60. See architecture.md - Layer 4A - Grid Charge Export Cooldown.
```

---

```text
input_datetime.last_grid_charge_command_time
-> Timestamp stamped whenever the decision engine requests
   charge_from_solar_and_grid. Used to gate discharge_to_maximize_export
   during the cooldown window above.
```

---

## Requested Battery Control

```text
input_select.emhass_requested_storage_mode
-> Storage mode requested by decision logic.
```

---

```text
input_number.emhass_requested_charge_limit
-> Charge limit requested by decision logic.
```

---

```text
input_number.emhass_requested_discharge_limit
-> Discharge limit requested by decision logic.
```

---

## Battery Value Sensors

```text
sensor.battery_value
-> Estimated value of currently stored battery energy.
```

---

```text
sensor.use_battery_for_house_load_worth_it
-> Indicates whether battery use is economically preferable to grid import.
```

---

```text
sensor.battery_worth_exporting
-> Indicates whether exporting battery energy is economically beneficial.
```

---

## Dynamic Control Sensors

```text
sensor.dynamic_storage_charge_limit
-> Dynamic charge power limit derived from battery SOC forecast.
```

---

```text
sensor.dynamic_storage_discharge_limit
-> Dynamic discharge power limit derived from battery SOC forecast.
```

---

```text
sensor.soc_batt_forecast_smooth
-> Smoothed battery SOC forecast.
```

---

## Decision Automations

```text
automation.emhass_battery_forecast_control
-> Main EMHASS-driven battery decision logic. Includes the Grid Charge
   Export Cooldown: discharge_to_maximize_export is held back in favour of
   maximize_self_consumption until grid_charge_export_cooldown_minutes has
   elapsed since the last charge_from_solar_and_grid command, to prevent
   selling recently grid-charged energy straight back out at a loss.
```

---

```text
automation.ev_charging_discharge_control
-> Reduces battery discharge while EV charging.
```

---

```text
automation.update_dynamic_charge_helper_from_sensor
-> Updates dynamic charge helper from forecast sensor.
```

---

```text
automation.update_dynamic_discharge_helper_from_sensor
-> Updates dynamic discharge helper from forecast sensor.
```

---

```text
automation.update_last_charge_limit
-> Stores last valid charge limit.
```

---

```text
automation.update_last_discharge_limit
-> Stores last valid discharge limit.
```

---

```text
automation.update_soc_target_from_next_forecast
-> Updates SOC target from next EMHASS forecast interval.
```

---

```text
automation.update_grid_charge_average
-> Tracks average battery grid charging cost.
```

---

```text
automation.reset_grid_charge_average
-> Resets charging price statistics when charging session ends.
```

---

# Layer 4B - Command Generation

Layer 4B converts requested battery behaviour into SolarEdge commands.

---

## Effective Battery Helpers

```text
input_select.effective_storage_mode
-> Effective storage mode after safety evaluation.
```

---

```text
input_number.effective_charge_limit
-> Effective charge limit after safety evaluation.
```

---

```text
input_number.effective_discharge_limit
-> Effective discharge limit after safety evaluation.
```

---

## Command Generation Automation

```text
automation.apply_effective_battery_control_to_solaredge_inverter
-> Applies effective battery control to SolarEdge. Includes a mode command
   guard: when the effective mode is maximize_self_consumption, the inverter
   already confirms that mode (select.solaredge_i1_storage_command_mode), AND
   the inverter's configured default/backup mode is confirmed to also be
   Maximize Self Consumption (select.solaredge_i1_storage_default_mode), the
   mode-select command and the 15-minute command timeout reset are skipped,
   and only the charge/discharge limits are sent. Any other mode, an
   unconfirmed live mode, or a default mode other than Maximize Self
   Consumption, still sends the full command sequence.
```

---

## Write Guard Diagnostics

```text
sensor.battery_mode_command_guard
-> Pure-template diagnostic mirroring the guard condition above (effective
   mode, live command mode, and configured default mode all confirmed as
   Maximize Self Consumption). State is "guarded" when the mode-select
   command + timeout reset would currently be skipped (only limits are
   being pushed), or "active" when the full command sequence is/would be
   sent on the next trigger.
```

---

## Command Scripts

```text
script.maximize_self_consumption_script
-> Request Maximize Self Consumption mode.
```

---

```text
script.charge_from_solar_power_and_grid_script
-> Request Charge From Solar Power And Grid mode.
```

---

```text
script.charge_from_solar_power_script
-> Request Charge From Solar Power mode.
```

---

```text
script.charge_from_clipped_solar_script
-> Request Charge From Clipped Solar mode.
```

---

```text
script.discharge_to_maximize_export_script
-> Request Discharge To Maximize Export mode.
```

---

```text
script.discharge_to_minimize_import_script
-> Request Discharge To Minimize Import mode.
```

---

```text
script.solar_power_only_script
-> Request Solar Power Only mode.
```

---

```text
script.set_effective_storage_charge_limit
-> Apply effective charge limit.
```

---

```text
script.set_effective_storage_discharge_limit
-> Apply effective discharge limit.
```

---

```text
script.set_value_storage_command_timeout15m
-> Set storage command timeout to 900 seconds.
```

---

# Layer 4C - Modbus Queue

Layer 4C serialises all SolarEdge Modbus writes.

---

## Queue Helpers

```text
input_boolean.modbus_busy
-> Modbus queue lock state.
```

---

## Queue Scripts

```text
script.modbus_queue
-> Serialises all SolarEdge Modbus commands.
```

---

## Queue Command Types

```text
set_storage_command_timeout
-> Set storage timeout.
```

---

```text
set_storage_charge_limit
-> Set storage charge limit.
```

---

```text
set_storage_discharge_limit
-> Set storage discharge limit.
```

---

```text
negative_site_limit_on
-> Enable export curtailment.
```

---

```text
negative_site_limit_off
-> Disable export curtailment.
```

---

```text
charge_from_solar_power_and_grid
-> Set SolarEdge charge mode.
```

---

```text
maximize_self_consumption
-> Set SolarEdge self-consumption mode.
```

---

```text
charge_from_clipped_solar_power
-> Set clipped solar charging mode.
```

---

```text
discharge_to_maximize_export
-> Set export discharge mode.
```

---

```text
discharge_to_minimize_import
-> Set import minimisation discharge mode.
```

---

```text
solar_power_only
-> Set Solar Power Only mode.
```

---

# Layer 4D - SolarEdge Apply

Layer 4D applies commands to SolarEdge Modbus Multi.

---

## SolarEdge Control Entities

```text
select.solaredge_i1_storage_control_mode
-> SolarEdge storage control authority mode.
```

---

```text
select.solaredge_i1_storage_command_mode
-> Active SolarEdge storage command mode. Also read live by the Layer 4B
   mode command guard to confirm the inverter is already in Maximize Self
   Consumption before deciding whether the mode-select command can be
   skipped.
```

---

```text
select.solaredge_i1_storage_default_mode
-> SolarEdge's configured default/backup storage mode - the mode the
   inverter falls back to internally once its Remote Control command
   timeout lapses without a refreshed command. Read live by the Layer 4B
   mode command guard: the mode-select command is only skipped when this is
   confirmed to be Maximize Self Consumption, so letting the timeout lapse
   is verified safe rather than assumed.
```

---

```text
number.solaredge_i1_storage_charge_limit
-> SolarEdge battery charge power limit.
```

---

```text
number.solaredge_i1_storage_discharge_limit
-> SolarEdge battery discharge power limit.
```

---

```text
number.solaredge_i1_storage_command_timeout
-> SolarEdge storage command timeout.
```

---

```text
switch.solaredge_i1_negative_site_limit
-> SolarEdge export curtailment control.
```

---

```text
number.solaredge_i1_site_limit
-> SolarEdge site export limit.
```