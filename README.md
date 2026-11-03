# INTENT — Human–Machine Interface for Mobility
### Turning Robotics Concepts into Practical Assistive Technology

[![ESP32](https://img.shields.io/badge/Platform-ESP32%20WROOM--32-blue.svg?logo=espressif&logoColor=white)](https://www.espressif.com/)
[![Arduino-ESP32](https://img.shields.io/badge/Arduino--ESP32-Core%20v3.x-E67E22.svg?logo=arduino&logoColor=white)](https://github.com/espressif/arduino-esp32)
[![ESP-NOW](https://img.shields.io/badge/Protocol-ESP--NOW%20(2.4GHz)-9B59B6.svg)](https://www.espressif.com/en/solutions/low-power-solutions/esp-now)
[![Build](https://github.com/sam-black007/INTENT/actions/workflows/build.yml/badge.svg)](https://github.com/sam-black007/INTENT/actions/workflows/build.yml)
[![Release](https://img.shields.io/github/v/release/sam-black007/INTENT?color=blue)](https://github.com/sam-black007/INTENT/releases/latest)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

**INTENT** is an open-source, gesture-controlled smart wheelchair prototype designed to translate micro-wrist and hand movements into intuitive power mobility. Engineered as an affordable, accessible Human–Machine Interface (HMI), the system replaces cumbersome traditional joysticks with an ergonomic wearable sensor, a low-latency peer-to-peer wireless telemetry link, and an intelligent fail-safe motor drive unit with a real-time web attendant console.

Built around dual **ESP32** microcontrollers communicating over connectionless **ESP-NOW** (~20 ms packet latency), INTENT works entirely off-grid without requiring external Wi-Fi routers or pairing procedures.

---

## 📑 Table of Contents

- [Vision & Purpose](#-vision--purpose)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Telemetry & Packet Protocol](#-telemetry--packet-protocol)
- [Attendant Web Console & Telemetry](#-attendant-web-console--telemetry)
- [Hardware & Bill of Materials](#-hardware--bill-of-materials)
- [Schematics & Wiring](#-schematics--wiring)
- [Control Engineering & Firmware Dynamics](#-control-engineering--firmware-dynamics)
- [Wearable Enclosure & 3D Printing](#-wearable-enclosure--3d-printing)
- [Quickstart & Flashing Guide](#-quickstart--flashing-guide)
- [Operational Reference](#-operational-reference)
- [Hardware Gallery & Working Demo](#-hardware-gallery--working-demo)
- [Safety & Assistive Tech Disclaimer](#-safety--assistive-tech-disclaimer)
- [Troubleshooting & Diagnostics](#-troubleshooting--diagnostics)
- [Repository Layout](#-repository-layout)
- [Roadmap](#-roadmap)
- [Contributing & License](#-contributing--license)

---

## 💡 Vision & Purpose

For individuals living with neuromuscular conditions, spinal cord injuries, or limited fine-motor dexterity, standard mechanical joysticks can induce rapid fatigue, pain, or total operational exclusion. 

**INTENT** demonstrates that advanced robotics principles can be packaged into an accessible assistive mobility stack costing under **$100**:
1. **Low Physical Effort:** Natural wrist tilting provides proportional, intuitive steering without grip strength.
2. **Deterministic Responsiveness:** Off-grid ESP-NOW broadcasting eliminates router handshakes and connection lag.
3. **Multi-Layered Safety:** Physical, kinematic, and watchdog fail-safes safeguard the user against signal drops, reverse-EMF spikes, and mechanical tipping.
4. **Attendant Dual-Control:** A zero-install local web console lets caregivers or clinicians monitor telemetry, adjust deadband sensitivity on-the-fly, or take manual override control.

---

## ✨ Key Features

- **Intuitive Gesture Driving:** 6-DOF inertial measurement via the MPU-6050 accelerometer (Pitch = Forward/Reverse, Roll = Turn Left/Right) with configurable deadband filtering (6°–25°).
- **Zero-Router Wireless Link:** Direct 2.4 GHz ESP-NOW broadcast delivering 16-byte telemetry frames at ~20 Hz (50 ms interval) with ~20 ms round-trip execution.
- **Dynamic Slew-Rate Ramping:** Smooth PWM ramping (8/255 counts every 10 ms = ~318 ms 0-to-100% transition) eliminates neck strain and jerky starts.
- **Anti-Stiction Momentum Kick:** A calibrated one-shot 95% PWM burst for 300 ms overcomes static gearbox friction from standstill without surging runaway speed.
- **Zero-Crossing H-Bridge Safeguard:** Direction reversals automatically ramp motor voltage down to 0 PWM before flipping polarity, protecting driver FETs and preventing back-EMF spikes.
- **Embedded Web Console & AP:** Receiver broadcasts its own SoftAP (`ESP32_Rover`) hosting an ultra-fast, responsive dark-mode web console with an artificial horizon IMU dot, link telemetry, and attendant touch D-pad.
- **Bidirectional Over-The-Air Tuning:** Attendant can dynamically slide sensitivity thresholds on the web UI, which are broadcast back to update the transmitter without reflashing firmware.
- **Tri-Fold Safety Interlock:**
  - **Mechanical Tier:** High-amperage normally-closed emergency-stop switch cut into the primary battery rail.
  - **Watchdog Tier:** 500 ms gesture timeout and 700 ms web timeout immediately engage an instantaneous 0-PWM brake.
  - **Firmware Governor Tier:** Gesture velocity is firmware-capped at 80% maximum duty cycle; attendant manual mode unlocks 100%.

---

## 🏗 System Architecture

The INTENT ecosystem is decoupled into two autonomous ESP32 nodes plus an attendant client device:

```mermaid
flowchart TD
    subgraph Wearable["Wearable Transmitter (Wrist HMI)"]
        IMU["MPU-6050 6-Axis IMU<br/>(Direct I2C Register Reads @ 0x68)"]
        ESP_TX["ESP32 DevKit V1 (Transmitter)<br/>- Tilt Estimation & Normalization<br/>- Deadzone & Sensitivity Gate<br/>- Packet Sequencer"]
        IMU -->|Raw Accel X, Y, Z| ESP_TX
    end

    subgraph Robot["Wheelchair Mobility Platform (Receiver & Drive)"]
        ESP_RX["ESP32 DevKit V1 (Receiver)<br/>- ESP-NOW Telemetry Parser<br/>- Dead-man Watchdog (500ms)<br/>- PWM Ramping & Momentum Kick<br/>- Local SoftAP & Web Server"]
        L298N["L298N Dual H-Bridge Driver<br/>(15 kHz High-Frequency PWM)"]
        MOTORS["2x DC Gearmotors<br/>(Differential / Skid Steering)"]
        ESTOP["Mechanical E-Stop Switch<br/>(In-Series Primary Power Cut)"]
        POWER[("7.4V - 12V Battery Pack")]

        ESP_RX -->|ENA, ENB (PWM 0-255)| L298N
        ESP_RX -->|IN1..IN4 (Direction Control)| L298N
        POWER --> ESTOP
        ESTOP --> L298N
        L298N --> MOTORS
    end

    subgraph Attendant["Attendant / Clinician Interface"]
        DASH["Responsive Web Console<br/>(http://192.168.4.1)<br/>- Live IMU Horizon<br/>- RSSI / Packet Health<br/>- Manual Override D-Pad<br/>- Sensitivity Slider"]
    end

    ESP_TX -- "ESP-NOW 2.4 GHz Broadcast<br/>16-byte Packet @ 20 Hz (50 ms)" --> ESP_RX
    ESP_RX -. "ESP-NOW Parameter Push<br/>(Sensitivity Settings)" .-> ESP_TX
    ESP_RX <== "HTTP / REST API (2 Hz Auto-Poll)<br/>SoftAP: ESP32_Rover" ==> DASH
```

---

## 📡 Telemetry & Packet Protocol

Communication over ESP-NOW uses a lightweight, packed 16-byte binary struct for minimal transmission overhead and sub-millisecond serialization:

```cpp
typedef struct __attribute__((packed)) {
  uint8_t  type;            // 1 = Gesture Telemetry, 2 = Attendant Settings
  uint8_t  command;         // 0: STOP, 1: FORWARD, 2: BACKWARD, 3: LEFT, 4: RIGHT
  uint8_t  speedPercent;    // Target velocity 0–100% (firmware-capped at 80% for gestures)
  uint8_t  sensitivityDeg;  // Tilt angle activation threshold (6°–25°)
  float    pitch;           // Live pitch angle in degrees (-90.0° to +90.0°)
  float    roll;            // Live roll angle in degrees (-90.0° to +90.0°)
  uint32_t counter;         // Monotonic packet sequence counter for gap/loss detection
} RoverPacket;
```

### Packet Field Breakdown

| Field | Data Type | Bytes | Range / Units | Description |
|:---|:---:|:---:|:---:|:---|
| `type` | `uint8_t` | 1 | `1` or `2` | Dispatches packet as motion command (`1`) or remote parameter update (`2`). |
| `command` | `uint8_t` | 1 | `0 – 4` | Motion vector (`0`=STOP, `1`=FWD, `2`=BWD, `3`=LEFT, `4`=RIGHT). |
| `speedPercent`| `uint8_t` | 1 | `0 – 100 %` | Proportional requested speed; scaled by current tilt angle above deadzone. |
| `sensitivityDeg`| `uint8_t`| 1 | `6° – 25°` | Configurable tilt threshold synchronized across transmitter and receiver. |
| `pitch` | `float` | 4 | `±90.0°` | Filtered forward/backward angular tilt for live telemetry plotting. |
| `roll` | `float` | 4 | `±90.0°` | Filtered lateral tilt angle for turning verification. |
| `counter` | `uint32_t`| 4 | `0 – 2^32-1` | Monotonic frame counter used by the receiver to calculate dropped frames and jitter. |

---

## 💻 Attendant Web Console & Telemetry

The receiver boots a dedicated Wi-Fi SoftAP on **Channel 1** with zero reliance on cloud servers or external routers:
- **SSID:** `ESP32_Rover`
- **Password:** `rover123`
- **Dashboard Gateway:** [`http://192.168.4.1`](http://192.168.4.1)

![Dashboard Preview](docs/media/dashboard.png)

### Dashboard Feature Matrix

| Module | Title | Primary Functionality |
|:---:|:---|:---|
| **01** | **Control Mode Selector** | Seamlessly toggle between **Gesture Mode** (speed capped at safe 80%) and **Web Mode** (attendant manual control up to 100%). |
| **02** | **Manual Override D-Pad** | Touch-friendly continuous drive controls with speed slider. Incorporates hold-to-drive repeat pulses (~200 ms) and immediate cut on release. |
| **03** | **Gesture & IMU Horizon** | Dynamic 2D tilt indicator dot displaying live orientation, combined with an interactive slider to calibrate wrist deadband (6°–25°). |
| **04** | **System Diagnostics** | Real-time readout of PWM output %, active movement state, link health, uptime, packet counters, and last recorded stop trigger. |
| **05** | **RF Connection Health** | ESP-NOW channel lock, packet sequence gap counter, live reception rate (~20 Hz meter), and physical receiver RSSI (dBm). |

> [!TIP]
> The **Advanced Connection Diagnostics** card tracks RF packet gaps. If packet loss exceeds 5% in high-interference environments, the caregiver can immediately detect signal degradation before a watchdog trip occurs.

---

## 🧰 Hardware & Bill of Materials

INTENT is engineered for extreme affordability and ease of sourcing. All components are standard off-the-shelf maker modules.

| Component | Quantity | Specification / Role | Est. Cost (USD) |
|:---|:---:|:---|:---:|
| **ESP32 DevKit V1** | 2 | Dual-core Tensilica Xtensa LX6, 240 MHz (1x Wearable TX, 1x Base RX) | $8 – $12 each |
| **MPU-6050 / GY-521** | 1 | 6-DOF IMU accelerometer/gyroscope on I2C address `0x68` | $2 – $4 |
| **L298N Dual H-Bridge** | 1 | Dual DC motor driver module (rated up to 2A per channel; PWM capable) | $3 – $6 |
| **2WD Chassis + Motors** | 1 | Robot chassis platform with 2x DC gearmotors + front caster | $15 – $30 |
| **Rechargeable Battery** | 1 | 7.4 V – 12 V Li-ion / LiPo or 8x AA battery pack | $10 – $20 |
| **Emergency Stop Button**| 1 | Heavy-duty panel-mount Normally-Closed (NC) latching push switch | $3 – $8 |
| **Cables & Prototyping**| 1 set | Breadboard, jumpers, USB micro cables for flashing | $5 – $8 |
| **Total Estimated Build Cost** | | | **~$50 – $90** |

👉 Detailed breakdown, shopping notes, and part alternatives: **[docs/BOM.md](docs/BOM.md)**.

---

## 🔌 Schematics & Wiring

![Wiring Schematic](docs/media/wiring.svg)

### 1. Receiver Node (ESP32 DevKit → L298N Motor Driver)

> [!IMPORTANT]
> **Remove both the `ENA` and `ENB` jumper caps** from the L298N driver board before wiring. The ESP32 drives these pins directly with 15 kHz high-frequency PWM for smooth speed control.
> 
> **Common Ground:** The ESP32 `GND` pin and the L298N power `GND` **must be securely tied together**.

| L298N Driver Pin | ESP32 GPIO | ESP32 Pin Type | Function |
|:---|:---:|:---|:---|
| **ENA** | **GPIO 32** | Output (LEDC PWM) | Left Motor Speed (0–255 duty) |
| **IN1** | **GPIO 25** | Digital Output | Left Motor Direction Bit 1 |
| **IN2** | **GPIO 26** | Digital Output | Left Motor Direction Bit 2 |
| **ENB** | **GPIO 33** | Output (LEDC PWM) | Right Motor Speed (0–255 duty) |
| **IN3** | **GPIO 27** | Digital Output | Right Motor Direction Bit 1 |
| **IN4** | **GPIO 14** | Digital Output | Right Motor Direction Bit 2 |
| **GND** | **GND** | Power Ground | **Mandatory Common Reference Ground** |

### 2. Mechanical Emergency-Stop Circuit

The mechanical E-stop switch is a physical normally-closed (NC) circuit wired directly in series with the positive battery rail feeding the L298N motor driver:

```
                  +---------------------------+
                  |  Normally-Closed E-STOP   |
Battery (+) ----->|  [ Latching Red Button ]  |-----> L298N +12V Motor Input
                  +---------------------------+

Battery (-) ----------------------------------------> L298N GND & ESP32 GND
```

Pressing the switch physically severs motor current at the power rail, halting locomotion instantaneously even during total microprocessor lockup.

### 3. Transmitter Node (Wearable ESP32 → MPU-6050 IMU)

| MPU-6050 Pin | ESP32 GPIO | Electrical Function |
|:---|:---:|:---|
| **VCC** | **3V3** | 3.3V Regulated Power |
| **GND** | **GND** | System Ground |
| **SDA** | **GPIO 21** | I2C Serial Data line |
| **SCL** | **GPIO 22** | I2C Serial Clock line |

---

## ⚙️ Control Engineering & Firmware Dynamics

INTENT avoids simplistic on/off motor bang-bang switching by incorporating real-time motion shaping algorithms directly within the firmware loop:

```
[Standstill] 
     │
     ├── New Motion Command Triggered
     ▼
[Momentum Kick]  ──> 95% PWM (300 ms one-shot burst) to overcome gearbox stiction
     │
     ├── Elapsed 300 ms
     ▼
[PWM Slew-Rate Ramping]  ──> ±8 counts / 10 ms steps toward Target Speed
     │
     ├── Reversal Requested? ──> Ramp down to 0 PWM FIRST, then invert H-Bridge
     ▼
[Active Locomotion] 
     │
     ├── Signal Lost (> 500 ms) OR E-Stop Triggered
     ▼
[Hard Instant Stop]  ──> 0% PWM bypasses ramp immediately for safety
```

### 1. Dynamic Momentum Kick (Stiction Compensation)
Small DC gearboxes and loaded casters suffer from high static friction (*stiction*). Starting from a dead stop at 40% PWM often causes motors to buzz without turning. INTENT introduces an intelligent momentum kick:
- Applies a **95% PWM burst for exactly 300 ms** upon departure from `CMD_STOP`.
- Once motion begins, the duty cycle seamlessly drops to the user's requested velocity.
- The kick is state-locked; continuous stream packets during motion do not re-trigger the pulse.

### 2. Slew-Rate Acceleration Limiter
To prevent abrupt acceleration that could startle an assistive device user:
- PWM is clamped to step by a maximum of **8/255 counts every 10 ms**.
- Reaching full speed (255) from zero requires approximately **~318 ms**, providing gentle, predictable transitions.

### 3. Direction Zero-Crossing Interlock
Flipping motor direction at high speed causes massive inductive back-EMF spikes and mechanical gearbox wear. INTENT enforces a software interlock: before direction GPIOs (`IN1`–`IN4`) can toggle, the PWM output is linearly stepped down to 0, ensuring smooth transitions.

### 4. Dual Watchdog Dead-Man Timers
- **Gesture Telemetry Watchdog:** If no valid ESP-NOW packet is received within **500 ms**, the receiver cuts PWM instantly (`0%`) and sets `lastStopReason = "Gesture Timeout"`.
- **Web Console Watchdog:** In web override mode, commands require a periodic pulse (~200 ms). If disconnected for more than **700 ms**, the system stops immediately.

---

## 🖨 Wearable Enclosure & 3D Printing

A parametric 3D enclosure for the wrist-worn transmitter is provided in the repository, designed in [OpenSCAD](https://openscad.org):
- **Source File:** [`docs/hardware/enclosure-remote.scad`](docs/hardware/enclosure-remote.scad)
- **Documentation:** [`docs/hardware/README.md`](docs/hardware/README.md)

### Print Recommendations

| Parameter | Recommended Setting |
|:---|:---|
| **Filament** | PETG or PLA+ (PETG preferred for drop resistance) |
| **Layer Height** | 0.20 mm |
| **Infill** | 20% – 25% (Gyroid or Grid) |
| **Perimeters** | 3 walls |
| **Hardware** | 4x M2.5 socket cap screws + wrist strap slots |

---

## 🚀 Quickstart & Flashing Guide

### 1. Prerequisites
- [Arduino IDE 2.x](https://www.arduino.cc/en/software)
- [Espressif Arduino-ESP32 Core v3.x](https://github.com/espressif/arduino-esp32) installed via Boards Manager:
  ```
  https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
  ```
- USB drivers: CP2102 or CH340 depending on your ESP32 board revision.
- *Zero external Arduino libraries required!* The transmitter talks directly to MPU-6050 hardware registers via core `Wire.h`.

### 2. Flash the Receiver Node
1. Connect the receiver ESP32 to your computer via USB.
2. In Arduino IDE, open:
   [`firmware/Rover_Receiver_Web_Gesture/Rover_Receiver_Web_Gesture.ino`](firmware/Rover_Receiver_Web_Gesture/Rover_Receiver_Web_Gesture.ino)
3. Select **Board: ESP32 Dev Module** and choose your COM port.
4. Click **Upload**.
5. Open Serial Monitor at **115200 baud** to confirm initialization:
   ```
   [WiFi] AP Started: ESP32_Rover (IP: 192.168.4.1)
   [ESP-NOW] Initialized on Channel 1
   [Motor] LEDC Attached (15 kHz, 8-bit)
   ```

### 3. Flash the Transmitter Node
1. Connect the transmitter ESP32 to your computer via USB.
2. Open:
   [`firmware/Gesture_Transmitter/Gesture_Transmitter.ino`](firmware/Gesture_Transmitter/Gesture_Transmitter.ino)
3. Select **Board: ESP32 Dev Module** and choose your COM port.
4. Click **Upload**.

> [!IMPORTANT]
> **Stationary IMU Calibration:** Keep the transmitter completely flat and stationary during the first **2 seconds** following power-up. The firmware samples the accelerometer offsets to calibrate zero pitch and roll.

### 4. Bench Testing Protocol
1. Prop the wheelchair chassis on a block so the drive wheels spin freely off the ground.
2. Power both boards.
3. Tilt the transmitter forward, backward, left, and right to verify motor direction.
4. On your phone or laptop, join Wi-Fi `ESP32_Rover` (password: `rover123`) and browse to `http://192.168.4.1`.
5. Test the **Emergency Stop** software button on the web console and press the **Physical E-Stop** switch to confirm instantaneous power cutoff.

---

## 🎮 Operational Reference

### Gesture Mapping

```
            [ TILT FORWARD ]
                   ▲
                   │  Move Forward
 [ TILT LEFT ] ◄───┼───► [ TILT RIGHT ]
   Turn Left       │       Turn Right
                   ▼
           [ TILT BACKWARD ]
             Move Backward
```

| Controller Orientation | Commanded Action | Drive Behavior | Speed Regulation |
|:---|:---:|:---|:---:|
| **Flat / Neutral** (< deadzone) | `CMD_STOP` | Both motors stationary; 0 PWM | Neutral deadband |
| **Tilt Wrist Forward** (Pitch > threshold) | `CMD_FORWARD` | Both motors drive forward | Proportional (35%–80%) |
| **Tilt Wrist Backward** (Pitch < -threshold)| `CMD_BACKWARD` | Both motors drive backward | Proportional (35%–80%) |
| **Tilt Wrist Left** (Roll < -threshold) | `CMD_LEFT` | Differential pivot left (Left BWD, Right FWD) | Proportional (35%–80%) |
| **Tilt Wrist Right** (Roll > threshold) | `CMD_RIGHT` | Differential pivot right (Left FWD, Right BWD) | Proportional (35%–80%) |

- **Sensitivity Adjustment:** Controlled on the web slider between **6°** (very sensitive, minimal wrist effort) to **25°** (broad movement required, tremor-resistant).
- **Deadband Protection:** Micro-movements within the deadband do not trigger motion, preventing jitter from resting hand positions.

---

## 📸 Hardware Gallery & Working Demo

### Prototype Hardware Assembly

| Receiver & Drive Assembly | Wearable Controller Prototype |
|:---:|:---:|
| ![Receiver Hardware 1](docs/media/hardware/hardware-1.jpeg) | ![Transmitter Hardware 2](docs/media/hardware/hardware-2.jpeg) |

### Demonstration Video

Watch INTENT in action demonstrating proportional gesture driving, live web telemetry updates, and instant emergency stop response:

[![Watch Working Demo](docs/media/demo-poster.jpg)](https://github.com/sam-black007/INTENT/blob/main/docs/media/demo.mp4)

> 🎬 *Click the preview image above to view the demo clip, or [download the raw MP4 video](https://raw.githubusercontent.com/sam-black007/INTENT/main/docs/media/demo.mp4).*

---

## ⚠️ Safety & Assistive Tech Disclaimer

> [!CAUTION]
> **RESEARCH & PROTOTYPE NOTICE:** INTENT is an exploratory, open-source engineering prototype and assistive technology proof of concept. **It is NOT a certified medical device and has NOT been approved by the FDA, CE-MDR, or any medical regulatory authority.**

Prior to any testing or demonstration:
- **Always perform initial trials with wheels suspended off the floor.**
- Never operate the platform without the mechanical, hardwired emergency-stop button within immediate reach of an attendant.
- Be aware that ESP-NOW broadcast frames in this prototype are unencrypted and unauthenticated. For clinical or real-world deployment, authenticated pairing, cryptographic MAC verification, and redundant obstacle detection sensors must be implemented.
- The author and contributors are not liable for any property damage, physical injury, or operational failure resulting from the assembly or usage of this codebase or hardware designs.

---

## 🔍 Troubleshooting & Diagnostics

| Symptom | Probable Cause | Corrective Action |
|:---|:---|:---|
| **Motors do not spin; driver hums** | Inadequate motor voltage or stiction. | Ensure battery voltage is ≥ 7.4V. Verify L298N jumpers are removed and `ENA`/`ENB` are receiving PWM from GPIO 32 and 33. |
| **Motors spin backward or turn opposite** | Inverted motor polarity wiring. | Swap the two output wire leads on the respective L298N terminal block (`OUT1/OUT2` or `OUT3/OUT4`). |
| **Transmitter fails to calibrate** | Sensor moved during power-on boot. | Hold the transmitter completely flat on a table for 3 seconds after resetting the board. |
| **Web dashboard unreachable at `192.168.4.1`** | Wi-Fi client disconnected or IP conflict. | Confirm connection to `ESP32_Rover`. Disable cellular data on mobile devices to prevent automatic route bypass. |
| **Frequent "Gesture Timeout" stops** | RF channel collision or distance limit. | Verify both sketches are set to `WIFI_CHANNEL = 1`. Ensure clear line-of-sight between wrist controller and receiver. |
| **No telemetry readings displayed** | I2C bus wiring fault on MPU-6050. | Check SDA (GPIO 21) and SCL (GPIO 22). Use an I2C scanner sketch to verify MPU responds at `0x68`. |

---

## 📂 Repository Layout

```
INTENT/
├── .github/
│   ├── workflows/
│   │   └── build.yml               # Automated CI: compiles both ESP32 sketches on push/PR
│   ├── ISSUE_TEMPLATE/             # Bug report and feature request templates
│   └── PULL_REQUEST_TEMPLATE.md    # Pull request checklist
├── docs/
│   ├── BOM.md                      # Detailed Bill of Materials with pricing & sourcing
│   ├── hardware/
│   │   ├── enclosure-remote.scad   # Parametric OpenSCAD wrist controller enclosure
│   │   └── README.md               # 3D printing & enclosure documentation
│   └── media/
│       ├── diagram.svg             # System architecture vector diagram
│       ├── wiring.svg              # Complete wiring schematic
│       ├── dashboard.png           # Attendant web console screenshot
│       ├── demo.mp4                # Working demonstration video
│       ├── demo-poster.jpg         # Video preview poster
│       └── hardware/
│           ├── hardware-1.jpeg     # Base chassis and receiver hardware photo
│           └── hardware-2.jpeg     # Wearable wrist controller photo
├── firmware/
│   ├── Gesture_Transmitter/
│   │   └── Gesture_Transmitter.ino # MPU-6050 tilt sensing & ESP-NOW broadcast TX
│   └── Rover_Receiver_Web_Gesture/
│       └── Rover_Receiver_Web_Gesture.ino # Motor driver, AP dashboard, ESP-NOW RX
├── CHANGELOG.md                    # Keep a Changelog version history
├── CODE_OF_CONDUCT.md              # Contributor Covenant Code of Conduct
├── CONTRIBUTING.md                 # Contribution guidelines & coding conventions
├── LICENSE                         # MIT Open Source License
└── README.md                       # Main documentation (this file)
```

---

## 🗺 Roadmap

- [ ] **Sensor Fusion Upgrade:** Incorporate gyro integration with a complementary / Madgwick filter for stable tilt tracking during rapid arm movements.
- [ ] **Closed-Loop Speed Control:** Wheel encoder feedback with PID velocity regulation on inclined ramps.
- [ ] **Collision Avoidance:** Front and rear ultrasonic / Time-of-Flight (ToF) distance sensors with automated emergency braking.
- [ ] **Battery Telemetry:** Voltage divider ADC circuit with low-voltage audible warning and dashboard state-of-charge percentage.
- [ ] **Cryptographic Pairing:** Encrypted ESP-NOW pairing with rolling nonce to prevent replay and broadcast spoofing.
- [ ] **Web Over-The-Air (OTA):** Firmware updates directly via the local web dashboard.

---

## 🤝 Contributing & License

Contributions, feature requests, and field-test feedback are warmly welcomed! Please read [CONTRIBUTING.md](CONTRIBUTING.md) and review our [Code of Conduct](CODE_OF_CONDUCT.md) before opening pull requests.

Distributed under the **MIT License**. See [`LICENSE`](LICENSE) for complete terms.

```
Copyright (c) 2026 Sam Joseph Thimothy
```

---

<div align="center">
  <sub>Built with ❤️ to bridge robotics engineering and accessible human mobility.</sub>
</div>
