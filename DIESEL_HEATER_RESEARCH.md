# Diesel Heater Research Document
## Controller Replacement Project - Technical Reference

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Major Components](#2-major-components)
3. [Operating Principles](#3-operating-principles)
4. [Electrical Specifications](#4-electrical-specifications)
5. [Communication Protocols](#5-communication-protocols)
6. [Sensors and Feedback](#6-sensors-and-feedback)
7. [Control Parameters](#7-control-parameters)
8. [Common Failure Modes](#8-common-failure-modes)
9. [Design Constraints for Controller Replacement](#9-design-constraints-for-controller-replacement)
10. [Design Limitations of Stock Controllers](#10-design-limitations-of-stock-controllers)
11. [Existing DIY Projects](#11-existing-diy-projects)
12. [Sources](#12-sources)

---

## 1. System Overview

### What is a Diesel Parking Heater?
A diesel parking heater (also called a diesel air heater) is a standalone combustion device that provides cabin heat without running the vehicle's engine. It draws diesel fuel from the vehicle's tank and operates independently off the 12V (or 24V) electrical system.

### Basic Operating Cycle
1. **Fuel Supply**: Diesel is drawn via a metering pump that delivers precise micro-doses
2. **Air Intake**: Fresh combustion air is drawn into the combustion chamber
3. **Ignition**: A glow plug ignites the fuel/air mixture
4. **Heat Transfer**: Hot combustion gases pass over a heat exchanger
5. **Distribution**: A blower fan circulates heated air into the cabin
6. **Exhaust**: Combustion byproducts are expelled via external exhaust pipe

---

## 2. Major Components

### 2.1 Electronic Control Unit (ECU)
The "brain" of the heater that controls all operations.

**Typical Specifications (Higher-end units like VVKB):**
- Microcontroller: STM32F103C8T6 (32-bit ARM Cortex-M3)
- Operating temperature range: -40°C to +85°C
- MOSFET driver: IRF3205S (55V, 110A capability)
- Communication: CAN bus protocol (on premium units)
- PCB: FR-4 board with self-extinguishing properties

**Chinese Clone Controllers:**
- Two categories: "Universal" and "Dumb" (proprietary pairing)
- Universal controllers/motherboards can be swapped
- Dumb controllers are paired to specific motherboards (Error E07 if mismatched)
- Common unlock code for advanced settings: **1688**

### 2.2 Glow Plug (Igniter)
Initiates combustion by heating the fuel/air mixture.

**Specifications:**
- Voltage: 12V or 24V versions available
- Material: Platinum/iridium filaments (oxidation resistant)
- Temperature capability:
  - Standard metal glow plugs: ~1000°C+
  - Ceramic glow plugs (silicon nitride): Up to 1300°C
- Preheat time:
  - Metal: 2-5 seconds to operating temperature
  - Ceramic: 800°C in ~2 seconds
- Cold resistance: Typically 0.6 to 2.0 ohms
- Used during both startup AND shutdown (for cleaning/burn-off)

### 2.3 Blower Motor Assembly (Dual-Function)

**IMPORTANT: Single Motor, Dual Shaft Architecture**

In most Chinese diesel heaters (and many other brands), a **single motor drives both the combustion blower and cabin air fan**. The motor has a dual-ended shaft with impellers on each end:

```
[Cabin Air Fan] ←──── [MOTOR] ────→ [Combustion Blower]
   (cold side)                         (hot side)
```

**Motor Specifications:**
- Type: Brushed DC motor with carbon brushes
- RPM range: ~1450 to 5000+ RPM (depending on model)
- Control method: PWM (Pulse Width Modulation)
- PWM frequency: Typically 16-20 kHz (above audible range)
- Bearings: Quality units use ball bearings (e.g., NMB/MinebeaMitsumi)
- Features: Should include EMI suppression filter

**Design Implications:**
| Aspect | Constraint |
|--------|------------|
| Speed coupling | Fan RPM and combustion air are mechanically locked together |
| No independent control | Cannot adjust cabin airflow without affecting combustion |
| Tuning tradeoff | Changing fan speed directly affects air/fuel ratio |
| Noise vs. heat | Running quieter (slower) reduces combustion air |

**Critical Function:**
Precise airflow is essential for complete combustion:
- Too little air → carbon deposits, smoke, incomplete burn
- Too much air → reduced efficiency, heat trapped in chamber

**Premium Alternative:**
Some manufacturers (e.g., Wallas marine heaters) use **separate motors** for each function, allowing independent control - but this adds cost and complexity

### 2.4 Fuel Metering Pump
Delivers precise micro-doses of fuel to the combustion chamber.

**Specifications:**
- Type: Solenoid-driven pulse pump
- Voltage: 12V or 24V
- Dosing volume per pulse:
  - Webasto pumps: ~63.3 µL/pulse
  - Chinese "22mL" pumps: 22 mL per 1000 pulses (0.022 mL/pulse)
  - Chinese "16mL" pumps: 16 mL per 1000 pulses (0.016 mL/pulse)
  - Measured variance: Some as low as 0.0125 mL/pulse
- Frequency range: 1-12 Hz (pulses per second)
- Typical settings:
  - Low power: ~1.3-1.5 Hz
  - High power: ~5.0-5.5 Hz
- Flow rate: 0.18-0.25 L/hour
- **CRITICAL**: System is open-loop; wrong pump = wrong fuel delivery

### 2.5 Heat Exchanger
Transfers combustion heat to cabin air without mixing exhaust gases.

**Construction:**
- Typically stainless steel or aluminum alloy
- Separates combustion chamber from air passage
- Must withstand temperatures >450°F (232°C) minimum for ignition
- Combustion can reach ~1600°F internally

### 2.6 Overheat Sensor
Critical safety component monitoring combustion chamber temperature.

**Requirements:**
- Must accurately detect flame temperature in real-time
- Triggers shutdown if temperature exceeds safe limits
- Often integrated into the combustion chamber assembly

### 2.7 Exhaust System
- Exhaust pipe with silencer
- Spiral designs counteract sound waves for noise reduction
- Must be properly sealed to prevent CO intrusion

---

## 3. Operating Principles

### 3.1 Startup Sequence

**Total startup duration: ~2-5 minutes**

| Phase | Duration | Current Draw | Description |
|-------|----------|--------------|-------------|
| 1. Pre-check | ~1-2 sec | <1A | Verify voltage, test overheat sensor, check glow plug continuity |
| 2. Glow plug preheat | 30-90 sec | 8-12A | Heat glow plug to ignition temperature (~1000°C) |
| 3. Fan ramp-up | 10-30 sec | 2-4A | Blower starts slow, gradually increases speed |
| 4. Fuel delivery | Overlaps | +0.5A | Pump begins clicking, delivering fuel pulses |
| 5. Ignition attempt | 30-60 sec | 10-14A | Fuel ignites; ECU monitors for temperature rise |
| 6. Flame stabilization | 30-60 sec | 6-10A | Glow plug may stay on briefly to ensure stable combustion |
| 7. Run mode transition | - | 1-4A | Glow plug off, normal modulated operation |

**Ignition Verification:**
- ECU monitors overheat sensor for temperature rise
- If no temperature increase detected within timeout → E08 error (flame failure)
- Multiple failed attempts trigger E10 lockout (typically after 6 retries)

**Common Startup Issues:**
- Air in fuel lines: Pump clicks but no ignition (fuel hasn't reached chamber)
- Low voltage: Glow plug doesn't reach ignition temperature
- Undersized wiring: Voltage drop prevents proper glow plug heating

### 3.2 Running Operation

**Modulated Control:**
- Fuel pump frequency modulated based on heat demand (1-6 Hz typical)
- Fan RPM adjusted proportionally via PWM
- Linear interpolation between min/max settings based on thermostat demand

**Control Loop (Thermostat Mode):**
```
IF cabin_temp < (setpoint - hysteresis):
    Increase fuel/fan toward maximum
ELIF cabin_temp > (setpoint + hysteresis):
    Decrease fuel/fan toward minimum OR cycle off
```

**Power Consumption During Run:**

| Output Level | Fuel Pump (Hz) | Fan RPM | Current Draw |
|--------------|----------------|---------|--------------|
| Minimum | 1.0-1.5 | 1500-2000 | 0.5-1.0A |
| Medium | 2.5-3.5 | 2500-3500 | 1.5-2.5A |
| Maximum | 5.0-6.0 | 4000-5000 | 3.0-4.0A |

### 3.3 Shutdown Sequence

**Total shutdown duration: ~3-5 minutes**

| Phase | Duration | Current Draw | Description |
|-------|----------|--------------|-------------|
| 1. Fuel pump stop | Immediate | - | No more fuel delivered to combustion chamber |
| 2. Glow plug reactivation | 60-90 sec | 8-12A | Burns off residual fuel and carbon deposits |
| 3. Burn-off complete | - | - | Glow plug deactivates |
| 4. Cool-down | 2-4 min | 1-3A | Fan continues at high speed to dissipate heat |
| 5. Fan ramp-down | 30-60 sec | <1A | Fan gradually slows as temperature drops |
| 6. Standby | - | ~0A | System enters low-power standby mode |

**CRITICAL WARNINGS:**
- **Never interrupt power during shutdown** - incomplete burn-off causes carbon buildup
- **Never restart immediately** - allow full cool-down cycle to complete
- Interrupting shutdown can lead to "coking" (carbon deposits blocking combustion chamber)

### 3.4 Air-to-Fuel Ratio
- Precise ratio essential for complete combustion
- Too little air → carbon deposits, smoke, incomplete burn
- Too much air → reduced efficiency, potential heat issues
- Open-loop system relies on factory calibration

---

## 4. Electrical Specifications

### 4.1 Power Consumption

| Phase | Current Draw (12V) | Duration |
|-------|-------------------|----------|
| **Startup** | 6-18 amps | 2-5 minutes |
| **Running (Low)** | 0.5-1.0 amps | Continuous |
| **Running (Medium)** | 1.5-2.0 amps | Continuous |
| **Running (High)** | 3.0-4.0 amps | Continuous |
| **Shutdown** | 6-10 amps | 2-3 minutes |

**Notes:**
- Startup current primarily from glow plug (~10A) and blower
- Premium brands (Webasto) may draw 17-18A on startup
- Overnight use: ~12-24 Ah typical for 8-hour operation
- Many portable power stations limited to 10A on 12V socket

### 4.2 Voltage Requirements

| System | Normal Range | Undervoltage Cutoff | Overvoltage Limit |
|--------|-------------|--------------------|--------------------|
| 12V | 12.0-14.5V | ~11.4V (E01 error) | 15V (E02 error) |
| 24V | 24.0-29.0V | ~22.8V | 30V |

### 4.3 Connector Pinout (Typical Chinese Heater)

| Pin | Function |
|-----|----------|
| 1 | +5V (logic power) |
| 2 | Blue wire (data/control bus) |
| 3 | Down button (+5V when pressed) |
| 4 | Power button (+5V when pressed) |
| 5 | Up button (+5V when pressed) |
| 6 | Button ground (0V) |
| 8 | Ground |

---

## 5. Communication Protocols

### 5.1 "Blue Wire" Protocol
The most common protocol used by Chinese diesel heaters (originally reverse-engineered by Ray Jones for the Afterburner project).

**Specifications:**
- Baud rate: **25,000 bps** (non-standard)
- Signal type: Single-wire half-duplex serial
- Logic levels: 5V signaling
- Interface: TX and RX multiplexed on single wire (time-division)
- Recommended: 470Ω series resistor for signal integrity

**Hardware Interface (3.3V MCU to 5V bus):**
- Requires level shifting (e.g., single gates in SOT-23 packages)
- ESP32 (3.3V) cannot directly interface with 5V bus
- Typical interface: Two SOT-23 gates for TX/RX multiplexing

### 5.2 Frame Structure (Blue Wire Protocol)

**Request Frame (Controller → Heater):**
```
Byte 0:    0x76  - Start of Frame
Byte 1:    0x16  - Data size (22 bytes follow)
Bytes 2-23: Command data + checksum
```

**Response Frame (Heater → Controller):**
```
Byte 0:    0x76  - Start of Frame
Byte 1:    0x16  - Data size (22 bytes follow)
Bytes 2-23: Status data + checksum
```

**Key Data Fields (approximate byte positions):**

| Field | Description |
|-------|-------------|
| Heater command | On/Off/Standby state request |
| Desired temperature | Target thermostat setpoint |
| Actual temperature | Current cabin temperature reading |
| Min/Max pump frequency | Fuel pump Hz range settings |
| Min/Max fan RPM | Blower speed range settings |
| Operating voltage | Current supply voltage |
| Run state | Current heater state (startup/run/shutdown/error) |
| Error state | Active error code (if any) |
| Glow plug state | On/off status |
| Runtime counter | Increments while heater is running |

**Checksum Calculation:**
- Simple sum of all bytes after byte 1 (not CRC)
- `checksum = sum(bytes[2:]) & 0xFF`

**Command Values:**
| Value | Command |
|-------|---------|
| 0x06 | Turn heater ON |
| 0x02 | Turn heater OFF |

### 5.3 Alternative Protocols

**WARNING:** Not all Chinese diesel heaters use the Blue Wire protocol!

| Brand/Model | Baud Rate | Frame Start | Notes |
|-------------|-----------|-------------|-------|
| Afterburner-compatible | 25,000 | 0x76 0x16 | Most common |
| Some older units | 25,000 | 0x78 0x16 0x00 | Slight variation |
| VEVOR (some models) | 4,800 | 0xAA 0x66 0x02 | Completely different protocol |

**Protocol Detection:**
Before designing a controller, verify protocol compatibility:
1. Use logic analyzer or oscilloscope on blue wire
2. Measure baud rate (bit width)
3. Capture and analyze frame headers

### 5.4 Connector Standards
- Mk1 Afterburner: 4-way JST-XH (2.54mm pitch)
- Mk2 Afterburner: 3-way JST-PH (2.0mm pitch)
- Output termination: 3-way JST-XH (2.54mm pitch)

### 5.5 Protocol Documentation
For complete byte-by-byte protocol specification, refer to:
- Ray Jones: "Hacking the Chinese Diesel Heater Communications Protocol V9.pdf"
- GitHub: `GeneralUltra758/generic-diesel-heater-controller`
- GitLab: `mrjones.id.au/bluetoothheater`

---

## 6. Sensors and Feedback

### 6.1 Cabin Temperature Sensor (External NTC Thermistor)

Located on the controller or externally in the cabin space.

**Typical Specifications:**
- Type: NTC (Negative Temperature Coefficient)
- Common values: 5kΩ or 10kΩ at 25°C (R25)
- B-value: 3435K to 3950K (typically 3470K)
- Temperature range: -40°C to +125°C
- Tolerance: ±1% to ±3%
- Sensitivity: -3% to -6% resistance change per °C

**Resistance-Temperature Relationship:**
- Highly non-linear (exponential)
- Requires linearization in software (Steinhart-Hart equation) or lookup table

**Example Values (10K NTC, B=3950):**

| Temperature | Resistance |
|-------------|------------|
| -20°C | ~67 kΩ |
| 0°C | ~32 kΩ |
| 20°C | ~12.5 kΩ |
| 25°C | 10.0 kΩ |
| 40°C | ~5.3 kΩ |

**Steinhart-Hart Equation:**
```
1/T = A + B*ln(R) + C*(ln(R))³
```
Where T is temperature in Kelvin, R is resistance.

### 6.2 Body/Overheat Sensor (Internal NTC or Thermal Switch)

Located inside the heater body, monitoring combustion chamber temperature.

**Specifications:**
- Type: High-temperature NTC thermistor OR thermal cutoff switch
- Temperature range: Up to 200-300°C
- Location: Mounted in ceramic enclosure with aluminum heat spreader
- Response time: Fast enough to detect flame presence/absence

**Functions:**
1. **Flame detection** - monitors temperature rise during startup
2. **Overheat protection** - triggers E05 error if temperature exceeds limit
3. **Shutdown verification** - confirms combustion chamber has cooled

**Premium Units (e.g., VVKB):**
- Use German Heraeus temperature sensors
- STM32 microcontroller monitors for unusual temperature changes
- Digital signal sent to MCU for precise control

### 6.3 Flame Detection Methods

Chinese diesel heaters typically have **no dedicated flame sensor** (unlike gas appliances with ionization probes). Instead, flame presence is inferred:

**Primary Method - Temperature Monitoring:**
```
During startup:
  IF body_temp rises > threshold within timeout:
    Flame detected → continue to run mode
  ELSE:
    Flame failure → E08 error, retry or lockout
```

**Secondary Methods (some controllers):**
| Method | How It Works |
|--------|--------------|
| Temperature rise rate | Rapid rise indicates successful ignition |
| Glow plug current | Changes when flame heats the plug |
| Exhaust temperature | Separate sensor in some premium units |

**Glow Plug as Temperature Sensor:**
Some advanced systems use glow plug resistance as a temperature indicator:
- Resistance changes with temperature
- Can detect flame presence by monitoring current/resistance
- Requires time-sharing between heating and sensing modes

### 6.4 Voltage Sensing

Built into the ECU to monitor supply voltage:

| Condition | Action |
|-----------|--------|
| < 11.4V (12V system) | E01 undervoltage error, shutdown |
| 11.4V - 15V | Normal operation |
| > 15V (12V system) | E02 overvoltage error, shutdown |

### 6.5 Sensor Summary

| Sensor | Location | Purpose | Typical Type |
|--------|----------|---------|--------------|
| Cabin temp | External/controller | Thermostat feedback | 10K NTC |
| Body temp | Combustion chamber | Flame detect, overheat | High-temp NTC |
| Voltage | ECU internal | Power monitoring | Resistor divider |
| (Optional) Altitude | Controller | Auto fuel adjustment | Barometric pressure |

---

## 7. Control Parameters

### 7.1 Adjustable Settings (Advanced Menu)

| Parameter | Typical Range | Notes |
|-----------|--------------|-------|
| Min pump frequency | 0.8-2.0 Hz | Fuel rate at low setting |
| Max pump frequency | 4.0-6.0 Hz | Fuel rate at high setting |
| Min fan RPM | 1400-2000 RPM | Blower speed at low |
| Max fan RPM | 3500-5000 RPM | Blower speed at high |
| Target temperature | 8-36°C | Thermostat setpoint |
| Glow plug timeout | Varies | Preheat duration |

### 7.2 Altitude Compensation

**The Problem:**
- Thin air at altitude = less oxygen = incomplete combustion
- Results in smoke, carbon buildup, reduced efficiency

**Manual Adjustment Rule:**
- Reduce fuel pump rate by **4% per 1,000 ft** elevation gain
- Fan RPM typically not adjusted

**Example Settings:**

| Elevation | Low (Hz/RPM) | High (Hz/RPM) |
|-----------|--------------|---------------|
| Sea level | 1.4 / 2000 | 3.2 / 3500 |
| 7,000 ft | 0.9 / 1750 | 2.2 / 4500 |

**Automatic Compensation:**
- Some premium units have barometric sensors
- Eberspacher High Altitude kit available
- Some VEVOR models claim auto-adjustment up to 18,045 ft (5,500m)

### 7.3 Fuel/Air Ratio Tuning
- Running lean (high altitude settings at low altitude): Less heat output but safe
- Running rich: Smoke, carbon deposits, potential overheating
- No closed-loop feedback in most units

---

## 8. Common Failure Modes

### 8.1 Error Codes Reference

| Code | Issue | Common Causes |
|------|-------|---------------|
| **E01** | Undervoltage | Battery below 11.4V |
| **E02** | Overvoltage | Voltage exceeds 15V (12V system) |
| **E03** | Glow plug fault | Failed plug, wiring issue, low voltage |
| **E05** | Overheat | Blocked airflow, crushed duct, fan failure |
| **E06** | Fan motor fault | Low voltage, motor failure |
| **E07** | Communication error | Controller/motherboard mismatch, wiring |
| **E08** | Flame out / No fuel | Empty tank, air in lines, pump failure |
| **E10** | Lockout | Multiple failed start attempts |

**Lockout Recovery:** Remove fuse, wait, reinstall, retry

### 8.2 Primary Failure Categories

**90% of failures come from:**
1. Battery voltage issues
2. Poor wiring connections
3. Blocked airflow
4. Fuel system problems

### 8.3 Common Component Failures

| Component | Failure Mode | Symptoms |
|-----------|--------------|----------|
| Glow plug | Burnout, carbon fouling | E03, no ignition |
| Fuel pump | Wear, contamination | E08, erratic operation |
| Combustion chamber | Carbon buildup | Smoke, poor combustion |
| Gaskets | Heat degradation | Exhaust leaks, smoke smell |
| Temp sensor | Drift, failure | Wrong readings, no shutdown |

### 8.4 Environmental Limitations

- **Cold weather**: Diesel gels below -12°C (10°F); use winter diesel or additives
- **Hot weather**: Reduced efficiency, potential overheating
- **Altitude**: See Section 7.2
- **Fuel quality**: Contamination causes rapid wear

---

## 9. Design Constraints for Controller Replacement

Before listing stock controller limitations, it's important to understand the fundamental constraints any replacement controller must work within.

### 9.0 Architectural Constraints

| Constraint | Implication for Design |
|------------|----------------------|
| **Single motor, dual shaft** | Cannot independently control cabin airflow and combustion air |
| **Open-loop fuel system** | Must trust calibrated pump rate; no fuel flow sensor |
| **No dedicated flame sensor** | Must infer flame from temperature rise rate |
| **Protocol compatibility** | Must support 25kbps Blue Wire OR replace entire ECU |
| **Safety-critical operation** | Must handle all error conditions and safe shutdown |
| **Power interruption risk** | Carbon buildup if shutdown interrupted |

### 9.0.1 Controller Replacement Options

There are two fundamental approaches:

**Option A: Replace Display/Interface Only (Afterburner approach)**
- Keep stock ECU/motherboard
- Communicate via Blue Wire protocol
- Limited to parameters exposed by protocol
- Safer: stock ECU handles combustion control

**Option B: Full ECU Replacement**
- Replace entire control system
- Direct control of glow plug, fuel pump, fan motor
- Maximum flexibility but maximum responsibility
- Must implement all safety logic from scratch

### 9.0.2 Key Parameters to Control/Monitor

| Parameter | Control | Monitor | Notes |
|-----------|---------|---------|-------|
| On/Off state | ✓ | ✓ | Basic operation |
| Target temperature | ✓ | ✓ | Thermostat setpoint |
| Pump frequency (Hz) | ✓ | ✓ | Fuel delivery rate |
| Fan RPM | ✓ | ✓ | Via PWM duty cycle |
| Glow plug | ✓ | ✓ | On/off + current sensing |
| Body temperature | - | ✓ | Flame detection, safety |
| Cabin temperature | - | ✓ | Thermostat feedback |
| Supply voltage | - | ✓ | Under/overvoltage protection |
| Error state | - | ✓ | Diagnostics |
| Runtime | - | ✓ | Maintenance tracking |

---

## 10. Design Limitations of Stock Controllers

### 10.1 User Interface Issues
- Small, low-resolution displays
- Limited menu navigation
- No remote access (basic models)
- Poor button tactile feedback

### 10.2 Thermostat Limitations
- Binary on/off cycling (no modulation on basic units)
- Large temperature swing before cycling
- No programmable schedules
- No adaptive learning

### 10.3 Safety Concerns
- Limited diagnostic information
- No data logging
- No trend monitoring for predictive maintenance
- Fixed shutdown timers

### 10.4 Connectivity
- No smartphone integration (basic models)
- No home automation integration
- No OTA firmware updates
- Bluetooth-only (iOS incompatible on many)

### 10.5 Altitude Handling
- Manual adjustment required
- No automatic compensation (most units)
- Settings must be changed when elevation changes

### 10.6 Fuel Monitoring
- No fuel consumption tracking
- No fuel level awareness
- Cannot estimate remaining runtime

### 10.7 Operational Modes
- Limited to single heat output curve
- No quiet/eco modes
- No boost/rapid heat modes
- Fixed startup/shutdown sequences

---

## 11. Existing DIY Projects

### 11.1 Afterburner (Ray Jones)
**The most comprehensive open-source replacement controller.**

- **Website**: http://www.mrjones.id.au/afterburner/
- **Repository**: https://gitlab.com/mrjones.id.au/bluetoothheater
- **Hardware**: ESP32 + HC-05 Bluetooth
- **Display**: 1.3" 128x64 OLED
- **Interface**: 5-button keypad
- **Features**:
  - WiFi web interface
  - MQTT IoT integration
  - Bluetooth control
  - Comprehensive monitoring
  - Detailed run-time data

**Note**: Requires "Blue Wire" protocol compatible ECU

### 11.2 ESPHome Integrations
- **cdh-esphome**: https://github.com/daoudeddy/cdh-esphome
- **Chinese-Diesel-Heater---ESPHome**: https://github.com/timmchugh11/Chinese-Diesel-Heater---ESPHome
- Enables Home Assistant integration
- UART configuration: `baud_rate: 25000`

### 11.3 Arduino Thermostat Controller
- **Repository**: https://github.com/wshelley/Chinese-Diesel-Heater-Advanaced-Temperature-Controller
- Simple inline thermostat add-on
- Keeps existing controller
- Arduino Nano based
- Monitors and overrides power based on temperature

### 11.4 RF Remote Library (ESP32)
- **Repository**: https://github.com/jakkik/DieselHeaterRF
- Replicates 433MHz remote protocol
- Uses CC1101 transceiver
- Parts cost: <$10 USD

### 11.5 I2C Interface Board
- **Repository**: https://github.com/TMakins/CDH_I2C_Interface
- Exposes heater parameters as I2C registers
- Enables integration with other microcontrollers

---

## 12. Sources

### General Information
- [VVKB Parking Heaters Buying Guide](https://www.vvkb.com/parking-heaters-buying-guide/)
- [FIRSTRATE: What Is a Diesel Parking Heater](https://firstratetools.com/what-is-a-diesel-parking-heater-and-how-does-it-work/)
- [VVKB Diesel Air Heater Guide](https://www.vvkb.com/diesel-air-heater/)
- [Wallas Marine Heaters](https://wallas.fi/marine-heaters/) - Comparison of single vs. dual motor designs
- [VVKB Combustion Blower Motor](https://www.vvkb.com/combustion-blower-motor/) - Motor specifications

### Controller and Protocol Information
- [Afterburner Official Site](https://www.mrjones.id.au/afterburner/)
- [Hackaday: Intelligent Control for Diesel Heaters](https://hackaday.com/2019/09/21/intelligent-control-for-that-cheap-diesel-heater/)
- [Experimental Engineering: Afterburner Build](https://www.experimental-engineering.co.uk/2019/07/20/afterburner-aftermarket-diesel-heater-controller-build/)
- [Van Life UK: Controller and Motherboard Guide](https://www.vanlifeuksurvivorsguide.co.uk/post/chinese-diesel-heater-controller-motherboard-remote-control-guide)
- [Hackaday.io: VEVOR Protocol](https://hackaday.io/project/195170-vevor-diesel-heater-protocol)
- [GitHub: Generic Diesel Heater Controller](https://github.com/GeneralUltra758/generic-diesel-heater-controller) - Protocol implementation

### Technical Specifications
- [Electronics Weekly: Fuel Metering Pump Operation](https://www.electronicsweekly.com/blogs/engineer-in-wonderland/learning-hard-way-diesel-water-heaters-regulate-fuel-2022-01/)
- [VVKB Electronic Control Unit](https://www.vvkb.com/electronic-control-unit/)
- [VVKB Overheat Sensor](https://www.vvkb.com/overheat-sensor/) - Temperature sensor specifications
- [Dieselheat: Power Consumption](https://www.dieselheat.com.au/faq/air-heating/what-is-the-electrical-power-consumption/)
- [The Camping Advisor: Amp Usage](https://thecampingadvisor.com/how-many-amps-does-a-diesel-heater-use/)
- [Portescap: Controlling Brushed DC Motors Using PWM](https://www.portescap.com/en/newsroom/whitepapers/2022/03/controlling-brushed-dc-motors-using-pwm)

### Troubleshooting and Error Codes
- [My Rig Adventures: 15 Problems + Error Codes](https://myrigadventures.com/chinese-diesel-heater-problems/)
- [ACLS Retail: Error Codes](https://www.aclsretail.com/support-request/heater-error-codes)
- [Hcalory: Fault Solutions](https://hcalory.com/blogs/news/diesel-heater-panel-display-types-of-faults-and-solutions)
- [VVKB Troubleshooting Guide](https://www.vvkb.com/vvkb-heater-fault-solution/)

### Altitude and Tuning
- [Stoke Loaf Van: Elevation Adjustment](https://www.stokeloafvan.com/blog-1/winter-van-life-chinese-diesel-heater-cdh-elevation-adjustment)
- [Hcalory: High Altitude Usage](https://hcalory.com/blogs/news/usage-guidelines-of-diesel-heaters-at-high-altitudes)
- [Dieselheat: High Altitude Information](https://www.dieselheat.com.au/information-and-buyers-guides/useful-dieselheat-downloads/dieselheat-high-altitude-information-for-dieselheat-products/)
- [Planar Heaters: High Altitude Use](https://planarheaters.com/use-portable-diesel-heaters-high-altitudes-extreme-cold/)

### Forum Discussions
- [Arduino Forum: Control Interface](https://forum.arduino.cc/t/12v-chinese-diesel-parking-heater-control-interface/571219)
- [Promaster Forum: Installation Tips](https://www.promasterforum.com/threads/diesel-heater-parking-heater-installation-collected-tips-from-years-of-posts.101306/)
- [Sportsmobile Forum: Altitude Settings](https://www.sportsmobileforum.com/forums/f22/chinese-diesel-heater-altitude-settings-26724.html)

---

## Document Information

- **Created**: January 2026
- **Last Updated**: January 2026
- **Purpose**: Technical reference for diesel heater controller replacement design
- **Status**: Research phase complete; ready for feature planning phase

### Key Findings Summary

| Topic | Key Insight |
|-------|-------------|
| Motor architecture | Single motor drives both combustion and cabin fans (coupled) |
| Control method | Fan speed via PWM; fuel via pulse frequency |
| Communication | Blue Wire protocol at 25kbps (non-standard) |
| Flame detection | Inferred from temperature rise, no dedicated sensor |
| Controller options | Interface-only (safer) vs. full ECU replacement (flexible) |

---
