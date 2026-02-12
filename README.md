# OpenPLC Multi-Zone Heating System with Extended 1-Wire Support

## Overview

This implementation extends OpenPLC's 1-wire capabilities to support **up to 32 DS18B20 temperature sensors** for monitoring multi-zone hydronic heating systems (e.g., underfloor heating). The solution enables sophisticated control logic that considers both air temperature and radiant heat from floor surfaces to optimize thermal comfort.

## Problem Statement

Standard heating systems typically measure only air temperature, but human comfort depends on both air temperature and radiant heat from surfaces. In underfloor heating systems:

- **Air temperature alone is insufficient**: A room may have adequate air temperature but feel cold due to a cold floor
- **Radiant heat matters**: Lower air temperature with warm floors can feel more comfortable than higher air with cold floors
- **Multi-zone monitoring**: Each heating zone requires multiple sensors (air, flow pipe, return pipe) to properly assess and control comfort

This implementation allows monitoring and control of multiple heating zones using simple Structured Text (ST) programming on a Raspberry Pi-based PLC.

## Features

- ✅ **Auto-discovery** of DS18B20 sensors on the 1-wire bus
- ✅ **Support for 32+ sensors** (easily expandable)
- ✅ **Direct Linux interface** via `/sys/bus/w1/devices/`
- ✅ **Background thread** for continuous sensor updates
- ✅ **IEC 61131-3 compatible** - program in Structured Text (ST)
- ✅ **Thread-safe** sensor data access
- ✅ **Simple PLC addressing** using standard %IW registers
- ✅ **Comfort index calculation** based on air + radiant temperature
- ✅ **Zone-by-zone control logic**

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Raspberry Pi PLC                          │
│  ┌────────────────────────────────────────────────────────┐ │
│  │              OpenPLC Runtime                           │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │    Structured Text Program                       │ │ │
│  │  │    (HeatingZoneMonitor.st)                       │ │ │
│  │  │                                                   │ │ │
│  │  │  • Read sensor temperatures (%IW0-%IW31)        │ │ │
│  │  │  • Calculate comfort index                       │ │ │
│  │  │  • Control zone valves (%QX outputs)            │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  │                         ↕                              │ │
│  │  ┌──────────────────────────────────────────────────┐ │ │
│  │  │    Custom Hardware Layer                         │ │ │
│  │  │    (raspberrypi_1wire.cpp)                       │ │ │
│  │  │                                                   │ │ │
│  │  │  • Sensor discovery                              │ │ │
│  │  │  • Background temperature reading                │ │ │
│  │  │  • Buffer management                             │ │ │
│  │  └──────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                         ↕                                    │
│           Linux 1-Wire Interface (w1-gpio)                   │
└─────────────────────────────────────────────────────────────┘
                         ↕
        GPIO Pin 23 (configurable) - 1-Wire Bus
                         ↕
┌─────────────────────────────────────────────────────────────┐
│     DS18B20 Sensors (up to 32 on single bus)                │
│                                                              │
│  Zone 1:  [Air] [Flow] [Return]                            │
│  Zone 2:  [Air] [Flow] [Return]                            │
│  Zone 3:  [Air] [Flow] [Return]                            │
│  ...                                                         │
│  Zone N:  [Air] [Flow] [Return]                            │
└─────────────────────────────────────────────────────────────┘
```

## Hardware Requirements

### Minimum Requirements

- **Raspberry Pi** (any model with GPIO support)
  - Tested on: Raspberry Pi 3B+, 4B, Zero W
- **DS18B20 Digital Temperature Sensors**
  - Waterproof probe version recommended for pipe mounting
  - Quantity: 3 sensors per zone (air, flow, return)
- **4.7kΩ Pull-up Resistor** (one per 1-wire bus)
- **Jumper wires** and connection terminals

### Recommended Additional Hardware

- **Zone valve actuators** (24VAC or 230VAC depending on system)
- **Relay board** for valve control (if using OpenPLC outputs)
- **DIN rail mount case** for Raspberry Pi
- **Power supply** (5V/3A for Raspberry Pi + peripherals)
- **Ethernet connection** (more reliable than WiFi for critical control)

## Wiring Diagram

### 1-Wire Bus Connection

```
Raspberry Pi GPIO              DS18B20 Sensors
┌─────────────┐               ┌──────────────┐
│             │               │  DS18B20 #1  │
│  3.3V (Pin 1)├──┬─────4.7kΩ─┤ VDD (Red)    │
│             │  │            │              │
│  GPIO 23    ├──┼────────────┤ Data (Yellow)│
│  (Pin 16)   │  │            │              │
│             │  │            │  DS18B20 #2  │
│  GND (Pin 6)├──┴────────────┤ GND (Black)  │───┐
│             │               │              │   │
└─────────────┘               └──────────────┘   │
                                                  │
                              ┌──────────────┐   │
                              │  DS18B20 #N  │   │
                              │              │   │
                              │ (Parallel)   ├───┘
                              └──────────────┘
```

**Important Notes:**
- All DS18B20 sensors share the same 3 wires (VDD, Data, GND)
- **Only one 4.7kΩ pull-up resistor** needed per bus
- Maximum recommended bus length: 100m (with proper cabling)
- Use **twisted pair or shielded cable** for long runs

### Sensor Placement

**Per Heating Zone:**
```
┌─────────────────────────────────────────────────┐
│                  Room/Zone                      │
│                                                 │
│              [Air Temp Sensor]                  │
│              (wall mounted,                     │
│               1.5m height)                      │
│                                                 │
│  ╔═══════════════════════════════════════════╗ │
│  ║   Underfloor Heating Loop                 ║ │
│  ║                                           ║ │
│  ║   [Flow Sensor]────────────►[Return      ║ │
│  ║   (pipe clamp)              Sensor]      ║ │
│  ║                             (pipe clamp)  ║ │
│  ╚═══════════════════════════════════════════╝ │
│                                                 │
│  Control: [Zone Valve]                          │
└─────────────────────────────────────────────────┘
```

## Software Installation

### Step 1: Install OpenPLC

```bash
# Clone OpenPLC repository
git clone https://github.com/thiagoralves/OpenPLC_v3.git
cd OpenPLC_v3

# Install dependencies
./install.sh linux

# Note: Select "Blank" as hardware layer during installation
# We will replace it with our custom layer
```

### Step 2: Install Custom 1-Wire Hardware Layer

Create the file `webserver/core/hardware_layers/raspberrypi_1wire.cpp` with the complete C++ implementation.

### Step 3: Configure Compilation

Edit the hardware layer selection file:

```bash
sudo nano webserver/core/hardware_layer.cpp

# Replace the include statement with:
#include "hardware_layers/raspberrypi_1wire.cpp"
```

### Step 4: Enable 1-Wire in Raspberry Pi

```bash
# Edit boot configuration
sudo nano /boot/config.txt

# Add the following line at the end:
dtoverlay=w1-gpio,gpiopin=23

# Save and reboot
sudo reboot
```

**To verify 1-wire is working:**
```bash
# Check for 1-wire devices
ls /sys/bus/w1/devices/

# You should see:
# 28-000012345678  (your DS18B20 device IDs)
# 28-000012345679
# w1_bus_master1
```

### Step 5: Recompile OpenPLC

```bash
cd OpenPLC_v3/webserver/core
./compile_program.sh
```

## Example Structured Text Programs

### Basic Single-Zone Monitor

```iecst
PROGRAM SingleZoneMonitor
VAR
    (* Temperature inputs - mapped to %IW registers *)
    AirTemp AT %IW0 : INT;           (* Air temperature * 100 *)
    FlowTemp AT %IW1 : INT;          (* Flow pipe temperature * 100 *)
    ReturnTemp AT %IW2 : INT;        (* Return pipe temperature * 100 *)
    
    (* Converted to REAL for calculations *)
    AirTempReal : REAL;
    FlowTempReal : REAL;
    ReturnTempReal : REAL;
    
    (* Calculated values *)
    DeltaT : REAL;                   (* Flow - Return difference *)
    ComfortIndex : REAL;             (* Weighted comfort metric *)
    
    (* Setpoints *)
    TargetTemp : REAL := 21.0;       (* Target comfort temperature *)
    ComfortBand : REAL := 1.0;       (* Hysteresis band *)
    MinFloorTemp : REAL := 23.0;     (* Minimum floor temperature *)
    
    (* Control outputs *)
    ZoneValve AT %QX0.0 : BOOL;      (* Zone valve control *)
    ColdFloorAlert AT %QX0.1 : BOOL; (* Alert signal *)
END_VAR

(* Convert integer sensor values to real temperatures *)
AirTempReal := INT_TO_REAL(AirTemp) / 100.0;
FlowTempReal := INT_TO_REAL(FlowTemp) / 100.0;
ReturnTempReal := INT_TO_REAL(ReturnTemp) / 100.0;

(* Calculate temperature delta across zone *)
DeltaT := FlowTempReal - ReturnTempReal;

(* Calculate comfort index: weighted average of air and floor temp *)
ComfortIndex := (AirTempReal * 0.4) + (ReturnTempReal * 0.6);

(* Main control logic with hysteresis *)
IF ComfortIndex < (TargetTemp - ComfortBand/2.0) THEN
    ZoneValve := TRUE;
ELSIF ComfortIndex > (TargetTemp + ComfortBand/2.0) THEN
    ZoneValve := FALSE;
END_IF;

(* Detect cold floor condition *)
IF AirTempReal >= TargetTemp AND ReturnTempReal < MinFloorTemp THEN
    ColdFloorAlert := TRUE;
ELSE
    ColdFloorAlert := FALSE;
END_IF;

END_PROGRAM
```

## Sensor Addressing Map

| Register | Description | Formula | Example |
|----------|-------------|---------|---------|
| %IW0 | Zone 1 Air Temperature | Raw / 100.0 | 2145 = 21.45°C |
| %IW1 | Zone 1 Flow Temperature | Raw / 100.0 | 4520 = 45.20°C |
| %IW2 | Zone 1 Return Temperature | Raw / 100.0 | 3580 = 35.80°C |
| %IW3 | Zone 2 Air Temperature | Raw / 100.0 | 1998 = 19.98°C |
| ... | ... | ... | ... |
| %IW31 | Sensor 31 | Raw / 100.0 | Up to 32 sensors |

## Testing and Commissioning

### Step 1: Verify Sensor Detection

```bash
# Check 1-wire devices
ls -la /sys/bus/w1/devices/

# Read a sensor manually
cat /sys/bus/w1/devices/28-*/w1_slave
```

Expected output:
```
5e 01 55 05 7f a5 81 66 8a : crc=8a YES
5e 01 55 05 7f a5 81 66 8a t=21875
```
Temperature is `t=21875` → 21.875°C

### Step 2: Monitor Register Values

1. Open OpenPLC web interface: `http://<raspberry_pi_ip>:8080`
2. Login (default: `openplc` / `openplc`)
3. Navigate to **Monitoring** page
4. Observe %IW registers updating with temperature values

## Troubleshooting

### Problem: No Sensors Detected

**Solutions:**

1. Check kernel modules:
```bash
lsmod | grep w1
```

2. Load modules manually:
```bash
sudo modprobe w1-gpio
sudo modprobe w1-therm
```

3. Check wiring and pull-up resistor

### Problem: Intermittent Sensor Readings

**Solutions:**

1. Cable length too long - use shielded cable
2. Power supply issues - use external 5V power
3. Add ferrite beads for EMI protection

## Safety Considerations

⚠️ **IMPORTANT SAFETY WARNINGS** ⚠️

1. **Electrical Safety**
   - Always include manual override for zone valves
   - Install separate high-temperature safety cutoff
   - Do not rely solely on PLC for safety functions

2. **Boiler Protection**
   - Never close all zone valves simultaneously
   - Always leave one zone or bypass open

3. **Backup Heating**
   - Install battery backup (UPS)
   - Provide manual control capability

## Support Resources

- **OpenPLC Official:** https://www.openplcproject.com
- **Forum:** https://openplc.discussion.community
- **DS18B20 Datasheet:** https://datasheets.maximintegrated.com/en/ds/DS18B20.pdf

## License

GPL v3 (compatible with OpenPLC)

## Contributing

Improvements welcome! Areas for contribution:
- Enhanced auto-configuration tools
- Web-based sensor mapping interface
- Advanced control algorithms (PID, MPC)

---

**Built with ❤️ for the open-source PLC community**