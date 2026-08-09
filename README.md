# 🚗 Vehicle Telemetry & Predictive Maintenance System

> An IoT-enabled vehicle telemetry system for real-time monitoring, anomaly detection, and predictive maintenance using embedded hardware, CAN communication, cloud telemetry, and an interactive analytics dashboard.

---

## 📌 Overview

The **Vehicle Telemetry & Predictive Maintenance System** is an embedded IoT project designed to monitor important vehicle parameters and provide a centralized view of vehicle health.

The system collects telemetry such as:

- Engine RPM
- Vehicle speed
- Coolant temperature
- Throttle position
- 3-axis acceleration
- MPU6050 temperature
- Vehicle tilt status
- GPS location

The collected data is transmitted through an ESP32-based controller and uploaded to **ThingSpeak** for cloud-based telemetry storage and monitoring.

An interactive web dashboard is provided for analyzing telemetry trends, identifying critical events, and evaluating overall vehicle health.

---

## 🎯 Objectives

- Monitor vehicle parameters in real time.
- Collect engine data through CAN communication.
- Monitor acceleration and tilt using MPU6050.
- Transmit telemetry data through Wi-Fi.
- Store telemetry data using ThingSpeak.
- Provide an interactive analytics dashboard.
- Detect abnormal operating conditions.
- Generate alerts for critical tilt events.
- Support predictive maintenance by analyzing vehicle behavior.

---

## 🏗️ System Architecture

```text
                   ┌─────────────────────┐
                   │   Simulated ECU      │
                   │                     │
                   │ RPM / Speed /       │
                   │ Coolant / Throttle  │
                   └──────────┬──────────┘
                              │
                         CAN Bus
                              │
                              ▼
                  ┌──────────────────────┐
                  │     ESP32 Controller │
                  │                      │
                  │ CAN + MPU6050        │
                  │ GPS + GSM + LCD      │
                  │ Wi-Fi                │
                  └──────────┬───────────┘
                             │
                       Wi-Fi / Internet
                             │
                             ▼
                  ┌──────────────────────┐
                  │      ThingSpeak      │
                  │   Cloud Telemetry    │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Analytics Dashboard  │
                  │                      │
                  │ Charts               │
                  │ Health Metrics       │
                  │ Critical Events      │
                  │ Correlation Analysis │
                  │ Data Explorer        │
                  └──────────────────────┘

                       ┌──────────────┐
                       │ GSM / GPS    │
                       │ Alert &      │
                       │ Location     │
                       └──────────────┘
```

---

# 🔧 Hardware Components

### Main Controller

- ESP32
- MCP2515 CAN Controller
- MPU6050 Accelerometer/Gyroscope
- GPS Module
- GSM Module
- 16×2 I2C LCD
- Vehicle/ECU CAN interface
- Wi-Fi connectivity

### Communication

- CAN Bus
- Wi-Fi
- GPS
- GSM
- ThingSpeak Cloud

---

# 📡 Telemetry Parameters

The system transmits multiple parameters to ThingSpeak.

| ThingSpeak Field | Parameter |
|---|---|
| Field 1 | Engine Speed |
| Field 2 | Engine RPM |
| Field 3 | Engine Coolant Temperature |
| Field 4 | Engine Throttle |
| Field 5 | Acceleration X |
| Field 6 | Acceleration Y |
| Field 7 | Tilt Alert |
| Field 8 | MPU Temperature |

The controller uploads the telemetry fields periodically through the ThingSpeak API.

---

# ⚙️ How the System Works

## 1. ECU Data Generation

The ECU transmitter simulates realistic vehicle operating conditions.

It generates:

- RPM
- Speed
- Coolant temperature
- Throttle position

The generated values are converted into a CAN frame and transmitted through the MCP2515 CAN controller.

The transmitter also implements an acknowledgement mechanism with retries to improve communication reliability.

---

## 2. CAN Data Acquisition

The ESP32 controller receives CAN frames and extracts:

```text
RPM
Speed
Coolant
Throttle
```

The telemetry controller decodes the CAN frame and stores the values for further processing.

---

## 3. Sensor Monitoring

The MPU6050 continuously provides:

```text
Acceleration X
Acceleration Y
Acceleration Z
Temperature
```

The system evaluates acceleration values to identify abnormal tilt conditions.

---

## 4. GPS Tracking

GPS data is processed using the TinyGPSPlus library.

When valid coordinates are available, the system can generate a Google Maps location URL for the vehicle.

---

## 5. Cloud Telemetry

The ESP32 connects to Wi-Fi and uploads vehicle parameters to ThingSpeak.

The telemetry data can then be used for:

- Monitoring
- Trend analysis
- Historical analysis
- Anomaly investigation
- Predictive maintenance

---

## 6. Alert System

When a significant tilt condition is detected, the controller can trigger a GSM SMS alert.

The alert can contain:

- Vehicle speed
- Engine RPM
- Coolant temperature
- MPU temperature
- Vehicle location

This provides an additional safety and monitoring mechanism.

---

# 📊 Analytics Dashboard

The project includes an interactive telemetry analytics dashboard.

### Dashboard Features

- 📈 Live Trend Explorer
- 🚗 Engine Load Analysis
- 📊 System Health Metrics
- ⚠️ Critical Event Detection
- 📉 Acceleration Analysis
- 🌡️ Thermal Behavior Analysis
- 🔵 Operating State Distribution
- 📊 Distribution Analytics
- 🔗 Correlation Heatmap
- 📋 Telemetry Data Explorer
- 🧠 Vehicle Health Indicators

The dashboard supports filtering based on:

- Start time
- End time
- Selected metric
- Operating state

Operating-state filters include:

```text
All Records
Moving Only
Idle Only
Tilt Alerts Only
High Coolant
```

---

# 📈 Dashboard Analytics

The dashboard analyzes relationships between different telemetry parameters.

### Engine Load

Analyzes:

```text
Speed × RPM × Throttle
```

### Acceleration

Analyzes:

```text
AX
AY
Acceleration Magnitude
Tilt Events
```

### Thermal Behavior

Analyzes:

```text
Coolant Temperature
MPU Temperature
```

### Correlation Analysis

The dashboard calculates Pearson correlation values between:

```text
Speed
RPM
Coolant
Throttle
Acceleration
```

---

# 🤖 Anomaly Detection & Predictive Maintenance

The project is designed around the concept of predictive maintenance by identifying abnormal vehicle behavior from telemetry data.

Potential indicators include:

- Unusual acceleration
- Sudden tilt events
- High coolant temperature
- High engine RPM
- Abnormal relationships between vehicle parameters
- Changes in normal operating patterns

These indicators can help identify conditions that may require further inspection or maintenance.

> **Note:** The current repository implementation includes telemetry collection and dashboard-based diagnostic indicators. If an actual machine-learning Isolation Forest implementation is added, it should be included as a separate `ml/` module with its training and inference code.

---

# 🔄 Project Workflow

```text
Vehicle / ECU
     ↓
CAN Data
     ↓
ESP32 Controller
     ↓
Sensor Acquisition
     ↓
Data Processing
     ↓
Wi-Fi
     ↓
ThingSpeak
     ↓
Telemetry Analytics
     ↓
Anomaly / Health Indicators
     ↓
Maintenance Decision
```

---

### ⚠️ Security

Never commit real Wi-Fi passwords, ThingSpeak API keys, phone numbers, or other private credentials to GitHub.

Use placeholders such as:

```cpp
const char* WIFI_SSID = "YOUR_WIFI_SSID";
const char* WIFI_PASS = "YOUR_WIFI_PASSWORD";

unsigned long CHANNEL_ID = YOUR_CHANNEL_ID;
const char* WRITE_API_KEY = "YOUR_THINGSPEAK_API_KEY";

const char* ALERT_PHONE = "YOUR_PHONE_NUMBER";
```

---

# 🔌 Required Arduino Libraries

Install the following libraries through the Arduino IDE Library Manager:

```text
WiFi
ThingSpeak
TinyGPSPlus
Adafruit MPU6050
Adafruit Unified Sensor
MCP2515
LiquidCrystal I2C
```

---

# 🧪 Running the ECU Simulator

Open:

```text
ecu_can_transmitter.txt
```

Upload it to the CAN transmitter/controller setup.

The simulator generates realistic:

```text
RPM
Speed
Coolant
Throttle
```

values and transmits them through CAN.

---

# 📊 Running the Dashboard

Open:

```text
telemetry_dashboard.html
```

The dashboard can be opened directly in a modern web browser.

It provides interactive visualization of telemetry records and calculated vehicle-health indicators.

---

⭐ If you find this project useful, consider giving the repository a star!
