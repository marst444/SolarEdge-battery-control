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
The modbus queue hinders multiple commands to be sent over the modbus simultaneously, but rather be passed one by one (https://github.com/marst444/SolarEdge-battery-control/blob/main/solaredge_modbusqueue.yaml)
# Template a charge/discharge logic
The charge/discharge logic is the core of the templating. 
- EMHASS publish sensors for forecast of battery and grid charge/discharge
- Some battery control sensors and helpers are created for the logic to control and follow the forecasted chargecurve (https://github.com/marst444/SolarEdge-battery-control/blob/main/batterycontrol_sensors.yaml)
- Scripts are created to send commands to the modbus queue (inverter)
(https://github.com/marst444/SolarEdge-battery-control/blob/main/batterycontrol_scripts.yaml)
- An automation to evaluate current PV production/ EVcharging and the EMHASS forecast is needed.
(https://github.com/marst444/SolarEdge-battery-control/blob/main/batterycontrol_automation.yaml)
  

I take no responsibility for the use of the charging/discharging logic!


# Big thanks to:
@ryanm101 (https://github.com/ryanm101)
@wilcodeforcats (https://github.com/WillCodeForCats)
@kipe (https://github.com/kipe)
@BJReplay( https://github.com/BJReplay)
@davidusbgeek (https://github.com/davidusb-geek)
