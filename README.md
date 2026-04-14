# Phoenix Robotics — MATE ROV Project

**Brooks DeBartolo Collegiate High School**
Competition: [MATE ROV](https://materovcompetition.org/) | Seasons: 2023, 2024

---

## Overview

This repository contains all firmware, sensor code, and competition documentation for the Phoenix Robotics team's underwater ROV ("Ranger") and supporting subsystems. The project spans three independent hardware systems:

| System | Purpose |
|---|---|
| [Propulsion System](#propulsion-system) | Thruster control via ESCs, joystick and Bluetooth gamepad input |
| [Underwater Monitoring System](#underwater-monitoring-system) | Real-time depth/pressure sensing over Wi-Fi |
| [Vertical Profiling Float](#vertical-profiling-float) | Autonomous buoyancy engine (non-ROV device) |

---

## Repository Structure

```
Robotics/
├── Propulsion-System-main/
│   ├── arduinoapr25          # Arduino Uno — ESC control via UART from ESP32
│   ├── espapr25              # ESP32 — Bluetooth gamepad receiver (Bluepad32)
│   ├── temp2                 # Arduino Mega variant (uses hardware Serial1)
│   ├── Motor Code            # Standalone joystick → 4-thruster control
│   ├── AlternativeCode       # Alternate pin-mapping variant of Motor Code
│   ├── PASTE                 # Early joystick prototype (MAX=150)
│   ├── Motor Test Code       # Single ESC arming/signal test
│   ├── Joystick Test Code    # Raw analog read diagnostic sketch
│   └── Main SID.pdf          # System Interface Diagram
│
├── Underwater-Monitoring-System-main/
│   ├── Sensor Code           # ESP8266 — live depth/pressure → WebSerial over Wi-Fi
│   ├── MS5837.cpp            # Custom MS5837 driver (Blue Robotics Bar02/Bar30)
│   └── Librarys              # Library installation notes and URLs
│
├── Vertical-Profiling-Float-main/
│   ├── Code with L293D       # Buoyancy engine using Adafruit Motor Shield (L293D)
│   ├── Code with MDD10A      # Buoyancy engine using relay-controlled pumps
│   ├── may2nd20251           # Enhanced VP float: sensor + SD logging + WebSerial
│   └── VP Diagram.pdf        # Hardware/pneumatic diagram of the float
│
└── Documentation/
    ├── 2023/                 # RN31 competition season documents
    └── 2024/                 # RN29 competition season documents
```

---

## Propulsion System

### Architecture

The ROV uses **4 Blue Robotics T-series thrusters** arranged in a vectored configuration:

- **Horizontal plane:** TH_L (left), TH_R (right) — forward/backward/yaw
- **Vertical plane:** TH_U (up), TH_D (down) — ascent/descent

All thrusters are driven by **ESCs using PWM servo signals** (1100–1900 µs, neutral at 1500 µs).

### Bluetooth Gamepad System (Current — `espapr25` + `arduinoapr25`)

This is the primary control architecture using a two-board design:

```
[Bluetooth Gamepad]
       |  Bluetooth
  [ESP32 (espapr25)]
       |  UART (115200 baud, TX:17, RX:16)
  [Arduino Uno (arduinoapr25)]
       |  PWM (Servo library)
  [4x ESC + Thruster]
```

**ESP32 (`espapr25`):**
- Uses [Bluepad32](https://github.com/ricardoquesada/bluepad32) to receive input from any Bluetooth gamepad (PS4, Xbox, etc.)
- Maps joystick axes (`axisY`, `axisRY`, `axisX`, `axisRX`) from raw range ±511 to ±150
- Transmits motor commands as a formatted string over UART:
  ```
  M1:<val>,M2:<val>,M3:<val>,M4:<val>\n
  ```

**Arduino Uno (`arduinoapr25`):**
- Receives command strings via SoftwareSerial (RX:11, TX:10)
- Parses the `M1:M2:M3:M4:` packet using `sscanf`
- Maps values from ±400 → ESC pulse width 1100–1900 µs
- Sends 1500 µs neutral on startup for 7 seconds to arm ESCs

**Pin mapping (Uno):**

| Motor | ESC Pin |
|---|---|
| M1 (Left) | D8 |
| M2 (Right) | D9 |
| M3 (Up) | D12 |
| M4 (Down) | D7 |

> Note: M4 is driven as `-M3` in the current firmware to maintain symmetry for vertical control.

### Arduino Mega Variant (`temp2`)

Functionally identical to `arduinoapr25` but targets an **Arduino Mega**, using the hardware `Serial1` port (no SoftwareSerial needed) and PWM pins 3, 5, 6, 9.

### Standalone Joystick Control (`Motor Code`, `AlternativeCode`, `PASTE`)

Earlier/alternative firmware for direct wired joystick control without the ESP32. Reads analog joystick axes and buttons directly from the Arduino's analog pins:

| Input | Pin | Motor |
|---|---|---|
| Joystick Y-axis | A12 | TH_L (port thruster) |
| Joystick Z-axis | A8 / A0 | TH_R (starboard thruster) |
| Button 1 | A4 | TH_U / TH_D (vertical) |
| Button 2 | A4 | TH_U / TH_D (vertical) |

**Thruster output pins:** 24 (TH_L), 25 (TH_D), 26 (TH_R), 27 (TH_U)

### Test Sketches

- **`Motor Test Code`** — Arms a single ESC on pin 9, then sends a fixed 1700 µs signal. Use this to verify an ESC is responding before full integration.
- **`Joystick Test Code`** — Prints raw analog values from A12, A8, A4 at 500 ms intervals. Use this to calibrate joystick center/endpoints.

---

## Underwater Monitoring System

### Hardware

- **Microcontroller:** NodeMCU (ESP8266)
- **Sensor:** Blue Robotics MS5837 Bar02 (2 bar) or Bar30 (30 bar) — depth + temperature
- **Communication:** ESP8266 Wi-Fi Access Point → browser-based WebSerial console

### Operation (`Sensor Code`)

On boot, the ESP8266 creates a Wi-Fi access point:

```
SSID:     Ranger 33
Password: 1234567890
```

Connect any device to this network and open the WebSerial interface at the ESP's IP address. Every **5 seconds**, the system broadcasts a log line:

```
RN33   HH:MM:SS UTC   XX.XX kPa   X.XX meters
```

The time counter starts from a hardcoded offset (`current_Time = "1417"`) corresponding to a competition window start time.

Fluid density is configurable in firmware:
- Freshwater: `997 kg/m³`
- Seawater: `1029 kg/m³`

### Advanced Variant (`PASTE` / `may2nd20251`)

An enhanced version targeting the **Vertical Profiling Float** (ESP8266 + SD card) with:

- **Depth zeroing** — sets a depth offset on first reading to compensate for surface atmospheric pressure
- **SD card logging** — continuously appends readings to `/sensor_data.txt`
- **Batch output** — buffers 30 readings, then dumps them to WebSerial and writes `/sensor_data_batch.txt`
- **Elapsed time tracking** — increments a session timer by 5 s each loop

Log format:
```
RN01 00:00:05 UTC 1013.5 mbar 0.02 m
```

Wi-Fi AP for this variant: `Ranger 7 / 1234567890`

### MS5837 Driver (`MS5837.cpp`)

A local copy of the [Blue Robotics MS5837 library](https://github.com/bluerobotics/BlueRobotics_MS5837_Library) with second-order temperature compensation. Key methods:

| Method | Description |
|---|---|
| `init()` | Resets sensor, reads PROM calibration, verifies CRC4 |
| `read()` | Triggers D1 (pressure) and D2 (temperature) ADC conversions |
| `pressure(conversion)` | Returns pressure in mbar, Pa, or bar |
| `temperature()` | Returns temperature in °C |
| `depth()` | Calculates depth in meters: `(P_abs - 101300) / (ρ × g)` |

### Library Dependencies

| Library | Source |
|---|---|
| ESP8266WiFi | Arduino ESP8266 core |
| ESPAsyncTCP | [dvarrel/ESPAsyncTCP](https://github.com/dvarrel/ESPAsyncTCP) |
| ESPAsyncWebServer | [me-no-dev/ESPAsyncWebServer](https://github.com/me-no-dev/ESPAsyncWebServer) |
| WebSerial / WebSerialLite | Ayush Sharma |
| MS5837 | [bluerobotics/BlueRobotics_MS5837_Library](https://github.com/bluerobotics/BlueRobotics_MS5837_Library) |
| RTClib | [adafruit/RTClib](https://github.com/adafruit/RTClib) |
| ArduinoJson | [bblanchon/ArduinoJson](https://arduinojson.org/) |

> See `Librarys` for the full installation guide including the ESP8266 board manager URL.

---

## Vertical Profiling Float

A non-ROV autonomous device that cycles between positive and negative buoyancy by inflating/deflating internal pneumatic bladders (balloons). This device is submitted as the team's non-ROV engineering challenge entry.

See `VP Diagram.pdf` for the full pneumatic and mechanical layout.

### Buoyancy Engine — L293D Version (`Code with L293D`)

Uses the **Adafruit Motor Shield (L293D)**:

| Channel | Device | Action |
|---|---|---|
| M3 | Water pump | Draws water in (sinks float) |
| M4 | Water pump | Expels water (rises float) |
| M1 | Air valve | Controls air bladder |

The loop alternates: fill with water (5 s) → stop (2 s) → expel water (5 s) → stop (2 s).

### Buoyancy Engine — MDD10A/Relay Version (`Code with MDD10A`)

Uses **6 relay outputs** (pins 8–13) to control pumps and solenoid valves in a timed sequence:

| Step | Action | Duration |
|---|---|---|
| 1 | Idle / init | 1 s |
| 2 | Compress orange balloon (RELAY_PIN2 + RELAY_PIN3 HIGH) | 8 s |
| 3 | All off (float at depth) | 20 s |
| 4 | Expand blue balloon (RELAY_PIN + RELAY_PIN5 HIGH) | 20 s |
| 5 | All off | 20 s |
| 6 | Compress orange balloon again | 16 s |
| 7 | All off, restart cycle | 16 s |

Pin 4 is held LOW as a common ground reference for the relay board.

### VP Float Sensor (`may2nd20251`)

Combined sensor + logging firmware for the float itself (described in detail under [Underwater Monitoring System — Advanced Variant](#advanced-variant-paste--may2nd20251) above).

---

## Competition Documentation

Located in `Documentation/`:

### 2023 Season (RN31)
| Document | Description |
|---|---|
| `2023_RANGER_Manual_Final_12_20_2022_cover.pdf` | Official MATE RANGER class competition manual |
| `RN31 Phoenix_Robotics_Technical_Documentation_2023.pdf` | Team's full technical writeup |
| `RN31 Phoenix_Robotics_Company_Specification_Sheet_2023.pdf` | ROV specifications / component list |
| `RN31 NON_ROV_Device_Design_Buoyancy_Engine_2023.pdf` | Buoyancy engine design document |
| `Engineering Presentation draft.pdf` | Engineering presentation slides |

### 2024 Season (RN29)
| Document | Description |
|---|---|
| `2024 RANGER Manual FINAL_1_16_2024_with_Cover-1.pdf` | Official MATE RANGER class competition manual |
| `RN29 Brooks DeBartolo Collegiate High Schoolchool_Phoenix Robotics_Technical Documentation_2024.pdf` | Team technical documentation |
| `RN29 Brooks DeBartolo Collegiate High School_Phoenix Robotics_Spec Sheet_2024.pdf` | Specification sheet |
| `RN29 Brooks DeBartolo Collegiate High School_Phoenix Robotics_Non-ROV Device Design_2024.pdf` | Non-ROV device design |
| `RN29 Brooks DeBartolo Collegiate High School_Phoenix Robotics_JSA_2024.pdf` | Job Safety Analysis |

---

## Hardware Summary

| Component | Part | Used In |
|---|---|---|
| Microcontroller | Arduino Uno | Propulsion (ESC driver) |
| Microcontroller | Arduino Mega 2560 | Propulsion (Mega variant) |
| Microcontroller | ESP32 | Propulsion (gamepad receiver) |
| Microcontroller | NodeMCU (ESP8266) | Sensor monitoring |
| Thruster ESC | BlueRobotics Basic ESC | All 4 thrusters |
| Pressure sensor | MS5837 Bar02 / Bar30 | Depth monitoring, VP float |
| Motor driver | Adafruit Motor Shield (L293D) | VP float (pump control) |
| Motor driver | Relay board (6-channel) | VP float (MDD10A version) |
| Storage | MicroSD via SPI | VP float data logging |
| Gamepad | Any Bluepad32-compatible controller | ROV pilot input |

---

## Getting Started

### Prerequisites

- [Arduino IDE](https://www.arduino.cc/en/software) 1.8+ or 2.x
- ESP32 board support via Arduino Board Manager
- ESP8266 board support: `http://arduino.esp8266.com/stable/package_esp8266com_index.json`
- All libraries listed in the [Library Dependencies](#library-dependencies) section

### Uploading Firmware

**Propulsion (Bluetooth gamepad setup):**
1. Flash `espapr25` to the ESP32 (select board: `ESP32 Dev Module`)
2. Flash `arduinoapr25` to the Arduino Uno
3. Wire ESP32 TX (pin 17) → Uno RX (pin 11), ESP32 RX (pin 16) → Uno TX (pin 10), share GND
4. Power on — Uno will arm ESCs over 7 seconds, then await gamepad data

**Sensor monitoring:**
1. Flash `Sensor Code` to NodeMCU ESP8266
2. Connect the MS5837 sensor via I2C (SDA/SCL + 3.3V + GND)
3. Power on — join Wi-Fi AP and open WebSerial in a browser

**Vertical Profiling Float:**
1. Flash `Code with MDD10A` to Arduino for relay-based buoyancy cycling
2. Flash `may2nd20251` to NodeMCU for sensor logging
3. Ensure SD card is formatted FAT32 and inserted before power-on

---

## Team

**Phoenix Robotics**
Brooks DeBartolo Collegiate High School
MATE ROV Competition — RANGER Class
