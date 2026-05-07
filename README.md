# CG2271 Final Project - IoT RTOS Shipment Monitor

## Overview
This project is an IoT-based smart shipment monitoring system built using FreeRTOS. It utilizes a dual-microcontroller architecture with an NXP MCXC444 MCU handling real-time hardware interrupts and physical indicators, and an ESP32 handling environmental sensors, geolocation, and cloud synchronization via Firebase Realtime Database. A Django web backend acts as a dashboard for tracking shipment metrics and configuring run thresholds.

## Hardware Components

### NXP MCXC444 (FRDM Board)
- **Sensors:** Hall Effect Sensor (detects box open/close), Shock Sensor.
- **Actuators & Indicators:** 
  - RGB LED (State indication: Blue = Idle, Green = Active Run, Red = Box Opened).
  - Active Buzzer (Sounds an alarm when the box is opened during a run).
  - 4-Digit SLCD (Cycles through Shocks, Opens, and Environmental Threshold Exceeded counts).
- **Inputs:** SW2 Button (Manual counter reset).

### ESP32
- **Sensors:** DHT11 (Temperature & Humidity), LDR / Photoresistor (Light Level).
- **Connectivity:** Wi-Fi module connecting to Firebase RTDB and an external IP Geolocation API (`ip-api.com`).

## Software Architecture
The system uses **FreeRTOS** on both microcontrollers to handle concurrent tasks safely and efficiently.

### NXP MCXC444 RTOS Tasks
- **`shockTask` / `hallTask`:** Responds to hardware interrupts (via binary semaphores) to increment shock counters and detect box state changes.
- **`indicatorTask`:** Controls the RGB LED based on the current system and shipment state.
- **`buzzerTask`:** Manages the active buzzer alert when a box is opened during an active shipment run.
- **`lcdTask`:** Periodically cycles the SLCD display to show runtime statistics (Shocks, Opens, Temp/Humidity/Light exceeded counts).
- **`recvTask`:** Parses UART messages from the ESP32 (Run status, environmental alerts).
- **`resetTask`:** Monitors the SW2 button for manual resetting of the local system state.

### ESP32 RTOS Tasks
- **`TaskTempHumid` & `TaskPhotoresistor`:** Periodically samples DHT11 and LDR sensors, comparing them against cloud-configured thresholds. Sends environmental alerts over UART to the MCXC444 if thresholds are exceeded.
- **`TaskGeoLocation`:** Fetches the shipment's latitude and longitude via IP-based geolocation every 5 minutes.
- **`TaskReceiveFromMCXC444`:** Listens for UART payloads from the MCXC444 regarding shock events and box state changes.
- **`TaskFirebase`:** Handles two-way synchronization with Firebase Realtime Database:
  - **Polling:** Checks `/Active_Run` status and `/run_config/` (thresholds) periodically.
  - **Pushing:** Uploads telemetry (temperature, humidity, light, geolocation, shocks, opens, status) to `/shipment_logs` every 30 seconds when a run is active.
- **`TaskWiFiManager`:** Ensures persistent Wi-Fi connectivity and handles auto-reconnection or rebooting if the connection is lost.

## Communication Protocol
The microcontrollers communicate over a UART interface (9600 Baud).
- **MCXC444 -> ESP32:** 
  - `SHOCK:<count>`, `BOX_OPEN:<count>`, `BOX_CLOSED`
- **ESP32 -> MCXC444:** 
  - `RUN:<1/0>` (Run state synced from cloud)
  - `TEXC:<count>`, `HEXC:<count>`, `LEXC:<count>` (Threshold exceeded alerts)
  - `TEMP:<f>,HUMI:<f>`, `LIGHT:<val>` (Informational environmental metrics)

## Cloud Integration
- **Firebase Realtime Database:** Acts as the central data broker, maintaining run configurations (temperature, humidity, and light thresholds), the active run toggle, and a historical log of telemetry data.
- **Django Backend:** Interfaces with Firebase to present a real-time tracking dashboard, allowing users to start/stop shipment runs, visualize environmental data, and receive real-time alerts.
