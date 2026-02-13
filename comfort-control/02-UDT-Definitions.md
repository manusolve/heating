# User Defined Types (UDTs) for Physics-Based Comfort Control

## Introduction

This document provides complete IEC 61131-3 Structured Text definitions for all User Defined Types (UDTs) used in the physics-based comfort control system. These structures organize data logically and enable modular, maintainable PLC code.

## UDT Design Philosophy

The UDT architecture follows these principles:

1. **Separation of Concerns**: Configuration, state, physics, and learning data are separated
2. **Scalability**: Structures support multiple sensors, zones, and manifolds
3. **Maintainability**: Clear naming and comprehensive documentation
4. **Performance**: Efficient memory layout for PLC execution
5. **Extensibility**: Easy to add new fields without breaking existing code

## 1. T_ZoneSensor

Represents a single temperature sensor with validation and error tracking.

```iecst
TYPE T_ZoneSensor : STRUCT
    (* Sensor Identification *)
    SensorAddress : INT;           (* Hardware address: %IW0, %IW1, etc. *)
    SensorID : STRING[20];          (* DS18B20 ID: "28-000012345678" *)
    
    (* Temperature Readings *)
    CurrentTemp : REAL;             (* Current temperature in °C *)
    LastValidTemp : REAL;           (* Last known good temperature in °C *)
    
    (* Sensor Status *)
    IsValid : BOOL;                 (* TRUE if reading is valid *)
    ErrorCount : INT;               (* Consecutive read errors *)
    LastReadTime : TIME;            (* Time of last successful read *)
    
    (* Sensor Type Classification *)
    SensorType : INT;               (* 0=Air, 1=Flow, 2=Return, 3=Outdoor *)
    
    (* Validation Thresholds *)
    MinValidTemp : REAL := -20.0;  (* Minimum plausible temperature *)
    MaxValidTemp : REAL := 80.0;   (* Maximum plausible temperature *)
    MaxErrorCount : INT := 5;       (* Errors before marking invalid *)
    
    (* Rate of Change Validation *)
    MaxTempChangePerScan : REAL := 2.0;  (* Max °C change per scan cycle *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- `SensorAddress` maps to PLC input registers (%IW0-%IW31)
- `SensorID` is the unique DS18B20 hardware ID for tracking
- `ErrorCount` increments on each failed read; resets to 0 on success
- `LastValidTemp` provides fallback when current reading is invalid
- `SensorType` helps with type-specific validation rules

## 2. T_ZoneConfig

Configuration parameters for a heating zone (typically loaded at startup).

```iecst
TYPE T_ZoneConfig : STRUCT
    (* Zone Identification *)
    ZoneName : STRING[32];          (* Human-readable name: "Living Room" *)
    ZoneID : INT;                   (* Unique zone number: 1-12 *)
    ManifoldID : INT;               (* Which manifold serves this zone: 1-4 *)
    
    (* Temperature Control Parameters *)
    TargetTemp : REAL := 21.0;      (* Desired comfort temperature in °C *)
    ComfortBand : REAL := 1.0;      (* Hysteresis band in °C (±0.5°C) *)
    MaxFloorTemp : REAL := 29.0;    (* Maximum floor temperature limit *)
    
    (* Comfort Index Weighting *)
    AirTempWeight : REAL := 0.35;   (* Weight for air temperature (0.0-1.0) *)
    FloorTempWeight : REAL := 0.65; (* Weight for floor temperature (0.0-1.0) *)
    (* Note: AirTempWeight + FloorTempWeight should equal 1.0 *)
    
    (* Hardware Addresses *)
    ValveAddress : INT;             (* Output address for zone valve: %QX0.0, etc. *)
    AirSensorAddr : INT;            (* Input register for air sensor *)
    FlowSensorAddr : INT;           (* Input register for flow sensor *)
    ReturnSensorAddr : INT;         (* Input register for return sensor *)
    
    (* Zone Physical Properties *)
    FloorArea : REAL := 20.0;       (* Floor area in m² *)
    RoomVolume : REAL := 50.0;      (* Room volume in m³ *)
    ExternalWalls : INT := 0;       (* Number of external walls (0-4) *)
    
    (* Control Enable *)
    IsEnabled : BOOL := TRUE;       (* Enable/disable zone control *)
    ScheduleEnabled : BOOL := FALSE; (* Use time-based scheduling *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- Configuration should be stored in persistent memory
- `TargetTemp` can be adjusted via HMI or external system
- `ComfortBand` of 1.0°C provides ±0.5°C hysteresis
- Weight sum validation should be checked at startup
- Physical properties used by learning algorithms

## 3. T_ZoneState

Runtime state information for a heating zone (changes dynamically).

```iecst
TYPE T_ZoneState : STRUCT
    (* Valve Control State *)
    ValveOpen : BOOL;               (* TRUE = valve open, FALSE = valve closed *)
    ValveOnTime : TIME;             (* Time when valve was last opened *)
    ValveOffTime : TIME;            (* Time when valve was last closed *)
    ValveOnDuration : TIME;         (* Cumulative time valve open this cycle *)
    ValveCycleCount : DINT;         (* Number of valve actuations *)
    
    (* Flow Status *)
    FlowEstablished : BOOL;         (* TRUE after flow stabilization delay *)
    FlowEstablishedTime : TIME;     (* When flow was established *)
    LastValveChange : TIME;         (* Time of last valve state change *)
    MinFlowDelay : TIME := T#120s;  (* Delay before flow considered stable *)
    
    (* Heating Demand *)
    HeatingDemand : BOOL;           (* TRUE if zone needs heating *)
    DemandStartTime : TIME;         (* When heating demand began *)
    DemandDuration : TIME;          (* How long demand has been active *)
    
    (* Comfort Status *)
    ComfortAchieved : BOOL;         (* TRUE if within comfort band *)
    OverheatWarning : BOOL;         (* TRUE if floor temp exceeds limit *)
    UnderheatWarning : BOOL;        (* TRUE if unable to maintain temp *)
    
    (* Error Flags *)
    SensorError : BOOL;             (* Sensor fault detected *)
    ValveError : BOOL;              (* Valve stuck or not responding *)
    FlowError : BOOL;               (* No flow when expected *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- `FlowEstablished` only TRUE after `MinFlowDelay` since valve opened
- `ValveCycleCount` helps detect valve wear
- Error flags trigger alarms and logging
- State persists between scan cycles but not across power cycles

## 4. T_ZonePhysics

Calculated physics values for real-time performance monitoring.

```iecst
TYPE T_ZonePhysics : STRUCT
    (* Temperature Measurements *)
    AirTemp : REAL;                 (* Current air temperature in °C *)
    FlowTemp : REAL;                (* Current flow temperature in °C *)
    ReturnTemp : REAL;              (* Current return temperature in °C *)
    EstFloorTemp : REAL;            (* Estimated floor surface temp in °C *)
    
    (* Temperature Differences *)
    DeltaT : REAL;                  (* Flow - Return temperature in °C *)
    DeltaToTarget : REAL;           (* Target - Current comfort index in °C *)
    
    (* Heat Transfer Calculations *)
    HeatDeliveryRate : REAL;        (* Instantaneous heat delivery in Watts *)
    CumulativeHeatDelivered : REAL; (* Total heat this session in kWh *)
    HeatDeliveryTime : TIME;        (* Duration of active heating *)
    
    (* Comfort Metrics *)
    ComfortIndex : REAL;            (* Weighted comfort metric in °C *)
    ComfortDeviation : REAL;        (* Difference from target comfort *)
    ComfortAchievedPercent : REAL;  (* Percentage of time at target *)
    
    (* Predictive Calculations *)
    PredictedTimeToTarget : TIME;   (* Estimated time to reach target *)
    PredictedMaxTemp : REAL;        (* Predicted peak temperature *)
    
    (* Heat Transfer Coefficients *)
    OverallHeatTransferCoeff : REAL; (* U*A value in W/K *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- `DeltaT` only valid when `FlowEstablished` is TRUE
- `HeatDeliveryRate` calculated using Q̇ = ṁ × Cp × ΔT
- `EstFloorTemp` typically ReturnTemp + 2-4°C
- `ComfortIndex` = AirTempWeight × AirTemp + FloorTempWeight × EstFloorTemp
- Cumulative values reset daily or per heating session

## 5. T_ZoneLearning

Learning algorithm data for adaptive optimization.

```iecst
TYPE T_ZoneLearning : STRUCT
    (* Heat Loss/Gain Rates *)
    HeatLossRate : REAL;            (* Heat loss to environment in W/K *)
    HeatGainRate : REAL;            (* Internal heat gains in W *)
    SolarGainFactor : REAL;         (* Solar contribution factor *)
    
    (* Thermal Time Constants *)
    ThermalTimeConstant : TIME;     (* Zone thermal lag (τ) in seconds *)
    FloorResponseTime : TIME;       (* Floor heating response time *)
    AirResponseTime : TIME;         (* Air temperature response time *)
    
    (* Optimal Parameters *)
    OptimalSupplyTemp : REAL;       (* Learned optimal supply temp in °C *)
    OptimalFlowRate : REAL;         (* Optimal flow rate in L/min *)
    OptimalStartTime : TIME;        (* How early to start before schedule *)
    
    (* Calibration Status *)
    IsCalibrated : BOOL := FALSE;   (* TRUE after sufficient learning *)
    LastCalibration : DATE_AND_TIME; (* Timestamp of last calibration *)
    CalibrationQuality : REAL;      (* Quality metric 0.0-1.0 *)
    
    (* Learning Statistics *)
    SampleCount : DINT := 0;        (* Number of learning samples *)
    MinSamplesRequired : DINT := 100; (* Samples needed for calibration *)
    
    (* Seasonal Adaptation *)
    WinterHeatLoss : REAL;          (* Winter heat loss coefficient *)
    SummerHeatLoss : REAL;          (* Summer heat loss coefficient *)
    CurrentSeason : INT;            (* 0=Winter, 1=Spring, 2=Summer, 3=Fall *)
    
    (* Prediction Errors *)
    MeanPredictionError : REAL;     (* Average temperature prediction error *)
    MaxPredictionError : REAL;      (* Worst-case prediction error *)
    
    (* Energy Efficiency *)
    EnergyPerDegree : REAL;         (* kWh per degree-hour maintained *)
    AverageEfficiency : REAL;       (* Overall system efficiency 0.0-1.0 *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- Learning data persists across power cycles (stored in retentive memory)
- `IsCalibrated` becomes TRUE after `SampleCount` ≥ `MinSamplesRequired`
- Seasonal parameters auto-update based on outdoor temperature patterns
- Learning continues indefinitely to adapt to building changes

## 6. T_Zone

Complete zone structure combining all zone-related data.

```iecst
TYPE T_Zone : STRUCT
    (* Zone Components *)
    Config : T_ZoneConfig;          (* Configuration parameters *)
    Sensors : ARRAY[1..4] OF T_ZoneSensor;  (* Array of sensors *)
        (* Index 1: Air sensor *)
        (* Index 2: Flow sensor *)
        (* Index 3: Return sensor *)
        (* Index 4: Outdoor sensor (optional, shared) *)
    
    State : T_ZoneState;            (* Runtime state *)
    Physics : T_ZonePhysics;        (* Physics calculations *)
    Learning : T_ZoneLearning;      (* Learning data *)
    
    (* Quick Access Aliases (computed from Sensors array) *)
    AirSensor : T_ZoneSensor;       (* Points to Sensors[1] *)
    FlowSensor : T_ZoneSensor;      (* Points to Sensors[2] *)
    ReturnSensor : T_ZoneSensor;    (* Points to Sensors[3] *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- Single structure contains all zone information
- Sensors array allows iteration for validation
- Quick access aliases improve code readability
- Typical memory size: ~400 bytes per zone

## 7. T_Manifold

Manifold control structure for coordinating multiple zones.

```iecst
TYPE T_Manifold : STRUCT
    (* Manifold Identification *)
    ManifoldID : INT;               (* Manifold number: 1-4 *)
    ManifoldName : STRING[32];      (* Name: "Ground Floor Manifold" *)
    
    (* Supply Temperature Monitoring *)
    SupplyTempSensor : T_ZoneSensor; (* Common supply temperature sensor *)
    SupplyTempTarget : REAL := 45.0; (* Target supply temperature in °C *)
    SupplyTempMin : REAL := 35.0;    (* Minimum supply temperature *)
    SupplyTempMax : REAL := 55.0;    (* Maximum supply temperature *)
    
    (* Pump Control *)
    PumpRunning : BOOL;             (* TRUE = pump active *)
    PumpAddress : INT;              (* Output address: %QX1.0, etc. *)
    PumpStartTime : TIME;           (* When pump started *)
    PumpRunTime : TIME;             (* Cumulative pump runtime *)
    MinPumpRunTime : TIME := T#180s; (* Minimum run time to prevent short cycling *)
    
    (* Flow Measurement *)
    FlowRate : REAL;                (* Current flow rate in L/min *)
    FlowMeterAddress : INT;         (* Flow meter input if available *)
    TotalFlowVolume : REAL;         (* Cumulative volume in liters *)
    
    (* Zone Management *)
    NumberOfZones : INT := 0;       (* Number of zones on this manifold *)
    ZoneIDs : ARRAY[1..12] OF INT;  (* Zone IDs served by this manifold *)
    ActiveZones : INT := 0;         (* Number of zones currently demanding heat *)
    
    (* Minimum Flow Protection *)
    MinimumFlowRequired : BOOL;     (* TRUE if at least one zone must be open *)
    BypassValveOpen : BOOL;         (* Bypass valve state *)
    BypassValveAddress : INT;       (* Bypass valve output address *)
    
    (* Manifold Status *)
    ManifoldEnabled : BOOL := TRUE; (* Enable/disable entire manifold *)
    PumpError : BOOL;               (* Pump fault detected *)
    LowFlowError : BOOL;            (* Flow rate too low *)
    HighTempError : BOOL;           (* Supply temperature too high *)
    
    (* Energy Tracking *)
    TotalEnergyDelivered : REAL;    (* Total energy in kWh *)
    InstantaneousPower : REAL;      (* Current power delivery in kW *)
    
END_STRUCT;
END_TYPE
```

**Usage Notes**:
- One manifold can serve multiple zones (typically 2-12)
- `SupplyTempSensor` measures common supply temperature for all zones
- Pump must run if any zone on manifold is active
- Bypass valve opens if all zones close (boiler protection)
- `ZoneIDs` array maps zones to manifolds

## 8. T_SystemConfig

Global system configuration (optional, for completeness).

```iecst
TYPE T_SystemConfig : STRUCT
    (* System Information *)
    SystemName : STRING[40] := "Physics-Based Heating Control";
    SoftwareVersion : STRING[20] := "1.0.0";
    InstallationDate : DATE;
    
    (* Timing *)
    ScanCycleTime : TIME := T#100ms;  (* PLC scan cycle *)
    SensorUpdateRate : TIME := T#1s;  (* Sensor reading frequency *)
    LoggingInterval : TIME := T#60s;  (* Data logging frequency *)
    
    (* Global Limits *)
    GlobalMaxFloorTemp : REAL := 29.0;
    GlobalMinSupplyTemp : REAL := 30.0;
    GlobalMaxSupplyTemp : REAL := 55.0;
    
    (* Safety *)
    EmergencyStopActive : BOOL := FALSE;
    FreezeProtectionTemp : REAL := 5.0;
    OverheatProtectionTemp : REAL := 32.0;
    
    (* Communication *)
    InfluxDBEnabled : BOOL := FALSE;
    InfluxDBAddress : STRING[100];
    LoggingEnabled : BOOL := TRUE;
    
END_STRUCT;
END_TYPE
```

## 9. Data Type Summary

| Type | Purpose | Size (approx) | Quantity |
|------|---------|---------------|----------|
| T_ZoneSensor | Individual sensor | ~80 bytes | 4 per zone |
| T_ZoneConfig | Zone configuration | ~120 bytes | 1 per zone |
| T_ZoneState | Zone runtime state | ~60 bytes | 1 per zone |
| T_ZonePhysics | Physics calculations | ~100 bytes | 1 per zone |
| T_ZoneLearning | Learning data | ~120 bytes | 1 per zone |
| T_Zone | Complete zone | ~480 bytes | 12 zones |
| T_Manifold | Manifold control | ~200 bytes | 4 manifolds |
| **Total Memory** | **All structures** | **~7 kB** | **Typical system** |

## 10. Variable Declarations

Typical global variable declarations in OpenPLC:

```iecst
VAR_GLOBAL
    (* System Configuration *)
    SystemConfig : T_SystemConfig;
    
    (* Manifolds *)
    Manifolds : ARRAY[1..4] OF T_Manifold;
    
    (* Zones *)
    Zones : ARRAY[1..12] OF T_Zone;
    
    (* Outdoor Temperature *)
    OutdoorTemp : REAL;
    OutdoorSensor : T_ZoneSensor;
    
    (* System Time *)
    SystemTime : DATE_AND_TIME;
    
END_VAR

VAR_GLOBAL RETAIN
    (* Learning data persists across power cycles *)
    ZoneLearningData : ARRAY[1..12] OF T_ZoneLearning;
    
END_VAR
```

## 11. Initialization Example

How to initialize a zone at system startup:

```iecst
(* Initialize Zone 1 - Living Room *)
Zones[1].Config.ZoneName := 'Living Room';
Zones[1].Config.ZoneID := 1;
Zones[1].Config.ManifoldID := 1;
Zones[1].Config.TargetTemp := 21.0;
Zones[1].Config.ComfortBand := 1.0;
Zones[1].Config.AirTempWeight := 0.35;
Zones[1].Config.FloorTempWeight := 0.65;
Zones[1].Config.ValveAddress := 0;  (* %QX0.0 *)
Zones[1].Config.AirSensorAddr := 0;  (* %IW0 *)
Zones[1].Config.FlowSensorAddr := 1; (* %IW1 *)
Zones[1].Config.ReturnSensorAddr := 2; (* %IW2 *)
Zones[1].Config.FloorArea := 25.0;
Zones[1].Config.IsEnabled := TRUE;

(* Initialize sensors *)
Zones[1].Sensors[1].SensorAddress := 0;
Zones[1].Sensors[1].SensorID := '28-000012345678';
Zones[1].Sensors[1].SensorType := 0;  (* Air *)
Zones[1].Sensors[2].SensorAddress := 1;
Zones[1].Sensors[2].SensorID := '28-000012345679';
Zones[1].Sensors[2].SensorType := 1;  (* Flow *)
Zones[1].Sensors[3].SensorAddress := 2;
Zones[1].Sensors[3].SensorID := '28-000012345680';
Zones[1].Sensors[3].SensorType := 2;  (* Return *)
```

## Conclusion

These UDT definitions provide a solid foundation for implementing the physics-based comfort control system. The structures are:

- ✅ **Type-safe**: IEC 61131-3 compliant
- ✅ **Well-documented**: Every field has clear purpose
- ✅ **Scalable**: Support for 12 zones, 4 manifolds
- ✅ **Maintainable**: Logical organization
- ✅ **Efficient**: Optimized memory layout

---

**Next**: [03-Function-Blocks.md](./03-Function-Blocks.md) - Implementation of functions and function blocks that operate on these data structures.
