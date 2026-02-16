# Physics-Based Comfort Control System Documentation

## Overview

Welcome to the comprehensive documentation for the **Physics-Based Comfort Control System** - an advanced, intelligent heating control solution designed for multi-zone underfloor heating systems using OpenPLC.

This system goes beyond traditional thermostat control by incorporating:
- **Multiple sensor types per zone** (air temperature, flow temperature, return temperature)
- **Physics-based calculations** for heat transfer and thermal dynamics
- **Adaptive learning algorithms** that optimize heating patterns over time
- **Manifold-level coordination** to ensure efficient operation
- **Time-series data logging** for analysis and continuous improvement

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      OpenPLC Runtime                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         PRG_HeatingSystem (Main Cyclic Program)            │ │
│  │                                                            │ │
│  │  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │ │
│  │  │ Zone Sensors │→│ Zone Physics│→│ Zone Control   │  │ │
│  │  │ (FC_Read)    │  │ (FC_Calc)   │  │ (FC_Control)   │  │ │
│  │  └──────────────┘  └─────────────┘  └────────────────┘  │ │
│  │          ↓                ↓                  ↓            │ │
│  │  ┌──────────────┐  ┌─────────────┐  ┌────────────────┐  │ │
│  │  │   Learning   │  │  Manifold   │  │  Data Logging  │  │ │
│  │  │(FC_Learning) │  │  Control    │  │  (InfluxDB)    │  │ │
│  │  └──────────────┘  └─────────────┘  └────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Key Features

✅ **Multi-Sensor Intelligence**
- Air temperature for ambient comfort
- Flow temperature to measure heat input
- Return temperature to calculate heat transfer
- Automatic sensor validation and error detection

✅ **Physics-Based Control**
- Heat transfer calculations (Q = ṁ × Cp × ΔT)
- Thermal mass modeling
- Predictive time-to-target calculations
- Comfort index based on radiant + convective heat

✅ **Adaptive Learning**
- Self-calibrating heat loss rates
- Time constant identification
- Optimal supply temperature learning
- Seasonal adaptation

✅ **Manifold Coordination**
- Multiple manifolds (up to 4)
- Pump control with minimum flow protection
- Bypass valve management
- Zone priority scheduling

✅ **Data-Driven Optimization**
- InfluxDB time-series storage
- Historical pattern analysis
- Performance metrics tracking
- Predictive maintenance alerts

## Documentation Structure

This documentation is organized into focused, topic-specific files:

### 📖 [01-Physics-Analysis.md](./01-Physics-Analysis.md)
**Physical Principles and Sensor Strategy**
- Underfloor heating physics fundamentals
- Heat transfer equations and dynamics
- Why flow temperature is common across zones
- Sensor placement rationale (2 vs 3 sensors per zone)
- Valid vs invalid measurement conditions
- What physics reveals about system performance

### 📖 [02-UDT-Definitions.md](./02-UDT-Definitions.md)
**User Defined Types (UDTs)**
- Complete IEC 61131-3 Structured Text definitions
- T_ZoneSensor - Sensor data structure
- T_ZoneConfig - Configuration parameters
- T_ZoneState - Runtime state tracking
- T_ZonePhysics - Calculated physics values
- T_ZoneLearning - Learning algorithm data
- T_Zone - Complete zone structure
- T_Manifold - Manifold control structure

### 📖 [03-Function-Blocks.md](./03-Function-Blocks.md)
**Function Blocks and Functions**
- FC_ReadZoneSensors - Sensor reading with validation
- FC_CalculateZonePhysics - Physics calculations
- FC_ZoneControl - Hysteresis control with flow validation
- FC_ZoneLearning - Adaptive learning algorithms
- FB_ManifoldControl - Manifold pump and bypass control
- Complete implementation code with detailed comments

### 📖 [04-Main-Program.md](./04-Main-Program.md)
**Main Cyclic Program**
- PRG_HeatingSystem structure
- Scan cycle organization
- Step-by-step program flow
- Integration of all function blocks
- Error handling and safety logic
- Complete working example

### 📖 [05-InfluxDB-Integration.md](./05-InfluxDB-Integration.md)
**Time-Series Database Strategy**
- Why InfluxDB for heating control
- Comparison with PostgreSQL/TimescaleDB
- Schema design for all measurements
- Flux query examples for analysis
- C++ integration code using libcurl
- Grafana dashboard setup

### 📖 [06-Installation-Guide.md](./06-Installation-Guide.md)
**Setup and Deployment**
- Hardware requirements and wiring
- InfluxDB installation on Raspberry Pi
- Grafana installation and configuration
- OpenPLC integration steps
- System service configuration
- Testing and commissioning procedures
- Troubleshooting guide

## Quick Start

1. **Understand the Physics** → Start with [01-Physics-Analysis.md](./01-Physics-Analysis.md)
2. **Learn the Data Structures** → Review [02-UDT-Definitions.md](./02-UDT-Definitions.md)
3. **Study the Control Logic** → Read [03-Function-Blocks.md](./03-Function-Blocks.md)
4. **See It All Together** → Examine [04-Main-Program.md](./04-Main-Program.md)
5. **Set Up Data Logging** → Follow [05-InfluxDB-Integration.md](./05-InfluxDB-Integration.md)
6. **Deploy the System** → Use [06-Installation-Guide.md](./06-Installation-Guide.md)

## System Requirements

### Hardware
- Raspberry Pi 3B+ or newer
- DS18B20 temperature sensors (2-3 per zone)
- Zone valve actuators (24VAC or 230VAC)
- 1-Wire interface (GPIO with 4.7kΩ pull-up)
- Relay board for valve control

### Software
- OpenPLC Runtime v3+
- Linux kernel with 1-Wire support (w1-gpio, w1-therm)
- InfluxDB 2.x (optional, for data logging)
- Grafana (optional, for visualization)

### Network
- Ethernet connection recommended
- Static IP address for PLC
- Network access to InfluxDB server

## Support and Resources

- **OpenPLC Official**: https://www.openplcproject.com
- **OpenPLC Forum**: https://openplc.discussion.community
- **InfluxDB Documentation**: https://docs.influxdata.com
- **IEC 61131-3 Reference**: Standard for PLC programming languages

## License

This documentation and associated code are released under the GPL v3 license, compatible with OpenPLC.

## Contributing

Improvements and contributions are welcome! Areas of particular interest:
- Enhanced control algorithms (MPC, neural networks)
- Web-based configuration interface
- Mobile app integration
- Additional sensor types (humidity, occupancy)

---

**Version**: 1.0  
**Last Updated**: February 2026  
**Status**: Production Ready

*Built with precision engineering and attention to comfort* 🌡️
