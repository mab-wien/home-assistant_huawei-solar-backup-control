# Dynamic PV Backup Reserve Control for Huawei + Home Assistant

## Overview

This project implements a fully dynamic backup reserve control for a Huawei ESS system using Home Assistant and Forecast.Solar.

Instead of keeping a fixed backup SOC (e.g. 30%), the system continuously calculates the minimum required battery reserve based on:

- Current average house consumption (30-minute mean)
- Forecasted PV production (Forecast.Solar API)
- Effective usable battery capacity
- System efficiency
- Daily peak cutoff logic

The goal is simple:

Keep as little backup reserve as necessary — but never too little.

The system automatically adjusts the inverter’s backup reserve (`number.wechselrichter_backup_power_ladestand`) every few minutes.

---

## Why This Is Different

Typical setups use a fixed backup reserve:

- 20%
- 30%
- 40%

This is inefficient because:

- On sunny days it wastes usable capacity.
- On cloudy days it may not be sufficient.
- It does not adapt to load or forecast conditions.

This project replaces fixed values with a predictive, load-aware, forecast-aware control logic.

---

## Core Idea

The system calculates:

How much battery capacity is required to survive until PV production can take over the load again.

It then sets the backup reserve accordingly — in 10% steps — with a maximum cap of 80%.

---

## Logic Flow

### 1. Determine Relevant Load

- Uses 30-minute mean house load (`sensor.emma_ladestrom`)
- Minimum enforced load = 1000 W  
  (because in blackout mode heavy consumers are shed)

Effective Load = max(30min_avg_load, 1000W)

---

### 2. Peak Cutoff Rule

After the daily peak production time has passed (`sensor.power_highest_peak_time_today_2`):

The system ignores the remaining hours of today  
and only evaluates starting from tomorrow 00:00.

This prevents late-afternoon false discharge decisions.

---

### 3. Find PV Takeover Time

Using Forecast.Solar hourly production data:

1. Try to find first future time where:

   PV >= Average Load

2. If not possible, fallback to:

   PV >= 1000W

3. If neither is found → FailSafe mode.

---

### 4. Calculate Required Energy

Energy required until takeover:

Energy = Load × Time_until_takeover / Efficiency

Battery percentage required:

Required_SOC = Energy / Battery_Capacity × 100

Efficiency factor default = 90%

---

### 5. Reserve Ramp Behavior

The system does NOT jump directly to 10%.

Instead:

- Reserve is rounded up in 10% steps
- Maximum reserve = 80%
- When PV takeover is reached → reserve becomes 0%

This creates a natural ramp:

80% → 70% → 60% → 50% → ... → 0%

Discharge begins as late as possible, but still guarantees survival until PV resumes.

---

## FailSafe Mode

If no future PV production can cover even 1000W:

Backup Reserve = 80%

This protects against multi-day cloudy periods.

---

## Ampel Status (Traffic Light Indicator)

The system exposes a status sensor:

| Status              | Meaning                                  |
|---------------------|------------------------------------------|
| ok                  | Average load can be covered              |
| fallback_1000w      | Only 1000W backup load possible          |
| fail_safe_80        | No sufficient PV forecast available      |

This can be visualized using Mushroom Cards.

---

## Features

- Dynamic reserve calculation
- Load-aware logic
- Forecast-based prediction
- Peak cutoff protection
- Blackout-mode optimization
- 10% step smoothing
- 80% maximum cap
- Automatic inverter update
- Dashboard-ready status sensors

---

## File Structure

Main configuration file:

configuration.yaml

All logic is implemented directly inside the Home Assistant configuration.

---

## Configuration File

Full configuration example:

See: configuration.yaml

---

## Requirements

- Home Assistant
- Forecast.Solar API
- Huawei inverter with modbus integration
- EMMA load sensor
- Battery capacity sensor
- Optional: Mushroom Cards

---

## Disclaimer

This project directly controls inverter backup reserve settings.

Use at your own risk.

Ensure:

- Correct battery capacity configuration
- Correct efficiency assumptions
- Reliable load shedding in blackout mode

---

## Philosophy

This is not just a script.

It is a small energy management system that tries to:

- Maximize self-consumption
- Minimize unnecessary reserve
- Maintain blackout readiness
- Adapt daily to weather conditions

---

## Author

Created as a personal energy optimization experiment.  
Shared for educational and experimental purposes.
