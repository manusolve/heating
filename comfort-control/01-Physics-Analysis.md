# Physics Analysis for Underfloor Heating Control

## Introduction

This document explores the physical principles underlying hydronic (water-based) underfloor heating systems and explains how understanding these principles leads to better control strategies and sensor placement decisions.

## 1. Fundamental Heat Transfer Principles

### 1.1 Heat Transfer Equation

The fundamental equation governing heat transfer in a hydronic heating system is:

```
Q̇ = ṁ × Cp × ΔT
```

Where:
- **Q̇** = Heat transfer rate (W or BTU/hr)
- **ṁ** = Mass flow rate of water (kg/s or lb/hr)
- **Cp** = Specific heat capacity of water (4186 J/kg·K or 1 BTU/lb·°F)
- **ΔT** = Temperature difference between flow and return (°C or °F)

**Key Insight**: For a given flow rate, the heat delivered to a zone is directly proportional to the temperature drop across that zone. This is why measuring both flow and return temperatures is crucial.

### 1.2 Heat Delivery to Room

Once heat enters the floor structure, it transfers to the room through two mechanisms:

**Radiant Heat Transfer** (60-70% of total):
```
Q̇_radiant = ε × σ × A × (T_floor⁴ - T_surfaces⁴)
```

**Convective Heat Transfer** (30-40% of total):
```
Q̇_convection = h × A × (T_floor - T_air)
```

Where:
- **ε** = Emissivity of floor surface (~0.9 for most materials)
- **σ** = Stefan-Boltzmann constant (5.67×10⁻⁸ W/m²·K⁴)
- **A** = Floor surface area (m²)
- **h** = Convective heat transfer coefficient (~10 W/m²·K)

**Key Insight**: Radiant heating dominates, which is why floor surface temperature is so important for comfort. A person can feel comfortable at lower air temperatures if the floor is warm.

## 2. Why Flow Temperature Is Common Across Zones

### 2.1 System Architecture

In a typical hydronic underfloor heating system:

```
                    [Boiler/Heat Source]
                            |
                    (Supply Manifold)
                            |
        ┌───────────────────┼───────────────────┐
        |                   |                   |
    [Zone 1]           [Zone 2]           [Zone 3]
    Valve Open         Valve Closed       Valve Open
        |                   X                   |
        └───────────────────┼───────────────────┘
                            |
                    (Return Manifold)
                            |
                    [Circulation Pump]
```

**Physical Explanation**:
- All zones are connected to the same supply manifold
- The supply manifold contains water at a uniform temperature (mixing occurs rapidly)
- Each zone draws from this common supply when its valve opens
- The boiler/mixer controls this supply temperature globally

### 2.2 Why This Matters for Measurement

**Valid Scenario**: When a zone valve is open
- Flow temperature = Supply manifold temperature (valid reading)
- Return temperature = Heat-depleted water (valid reading)
- ΔT = Flow - Return gives actual heat transfer

**Invalid Scenario**: When a zone valve is closed
- Flow temperature sensor reads ambient pipe temperature (invalid)
- Return temperature sensor reads stagnant water (invalid)
- ΔT is meaningless

**Key Insight**: Flow and return temperature measurements are only valid when water is actively circulating through the zone (valve open + pump running).

## 3. Thermal Dynamics and Time Constants

### 3.1 Thermal Mass Effects

Underfloor heating has significant thermal mass:

```
Thermal Mass = m × Cp
```

For a typical zone:
- Concrete slab: ~150 kg/m²
- Specific heat: ~880 J/kg·K
- **Total thermal mass: ~132 kJ/K per m²**

**Compare to Radiator**:
- Steel radiator: ~10 kg
- Water content: ~5 kg
- **Total thermal mass: ~10-25 kJ/K**

**Key Insight**: Underfloor heating responds 5-10x slower than radiators. This requires:
- Predictive control algorithms
- Learning of thermal time constants
- Avoidance of rapid on/off cycling

### 3.2 Time Constants

The thermal time constant τ is defined as:

```
τ = (m × Cp) / (h × A)
```

For underfloor heating:
- **τ ≈ 2-4 hours** (concrete slab)
- **τ ≈ 0.5-1 hour** (screed floor)

**Practical Implications**:
- It takes 3-5 time constants to reach 95% of target temperature
- For concrete: 6-20 hours from cold start
- For maintenance: 1.5-5 hours from small setback

**Key Insight**: Control algorithms must look ahead and start heating well before the desired time. Simple on/off control based on current temperature will always lag behind demand.

## 4. Sensor Placement Strategy

### 4.1 Three-Sensor Configuration (Recommended)

**Per Zone**:
1. **Air Temperature Sensor**
   - Location: Wall-mounted, 1.5m height
   - Measures: Ambient air temperature
   - Purpose: Primary comfort feedback

2. **Flow Temperature Sensor**
   - Location: Supply pipe entering zone
   - Measures: Hot water entering floor
   - Purpose: Heat input monitoring

3. **Return Temperature Sensor**
   - Location: Return pipe leaving zone
   - Measures: Cooled water leaving floor
   - Purpose: Heat transfer calculation

**Advantages**:
```
✓ Calculate actual heat delivery: Q̇ = ṁ × Cp × (T_flow - T_return)
✓ Detect flow problems (ΔT too small → no flow or valve issue)
✓ Detect heat transfer problems (ΔT too large → poor thermal coupling)
✓ Validate sensor readings (flow should exceed return when flowing)
✓ Learning: measure system response to control actions
```

### 4.2 Two-Sensor Configuration (Minimum)

**Per Zone**:
1. **Air Temperature Sensor**
2. **Flow OR Return Temperature Sensor**

**Advantages**:
```
✓ Lower cost
✓ Simpler installation
✓ Still provides basic comfort control
```

**Disadvantages**:
```
✗ Cannot calculate actual heat delivery
✗ Cannot detect flow problems automatically
✗ Cannot validate sensor readings
✗ Limited learning capability
```

### 4.3 Physics-Based Recommendation

For a **physics-based control system with learning**, the three-sensor configuration is strongly recommended because:

1. **Heat Delivery Measurement**
   - Without ΔT measurement, heat delivery is unknown
   - Learning algorithms cannot correlate heat input with temperature response
   - Optimal supply temperature cannot be determined

2. **Fault Detection**
   - Stuck valves undetectable without flow confirmation
   - Air locks in pipes undetectable
   - Pump failures may go unnoticed

3. **Efficiency Optimization**
   - Cannot determine if zones are over-supplied (low ΔT)
   - Cannot balance flow rates across zones
   - Cannot optimize pump speed

## 5. Valid vs Invalid Measurement Conditions

### 5.1 Measurement Validity Table

| Condition | Flow Temp | Return Temp | ΔT Calculation | Air Temp |
|-----------|-----------|-------------|----------------|----------|
| **Valve Open + Pump Running** | ✅ Valid | ✅ Valid | ✅ Valid | ✅ Valid |
| **Valve Open + Pump Off** | ❌ Invalid | ❌ Invalid | ❌ Invalid | ✅ Valid |
| **Valve Closed + Pump Running** | ⚠️ Reads manifold | ❌ Stagnant | ❌ Invalid | ✅ Valid |
| **Valve Closed + Pump Off** | ❌ Ambient | ❌ Ambient | ❌ Invalid | ✅ Valid |
| **System Cold Start** | ⏳ Stabilizing | ⏳ Stabilizing | ⏳ Wait 2-5 min | ✅ Valid |

**Legend**:
- ✅ Valid: Reading accurately represents current condition
- ❌ Invalid: Reading does not represent current condition
- ⚠️ Caution: Reading valid but not representative of zone
- ⏳ Wait: Need time for sensors to stabilize

### 5.2 Flow Establishment Time

After opening a valve, there is a delay before valid readings:

```
Time to Stable Reading:
├─ Valve actuation: 30-90 seconds
├─ Water reaches zone: 10-30 seconds  
├─ Sensor thermal response: 30-60 seconds
└─ Total: 70-180 seconds (1-3 minutes)
```

**Implementation**:
```iecst
(* Track time since valve opened *)
IF ZoneValveOpen AND NOT FlowEstablished THEN
    IF (CurrentTime - ValveOpenTime) > T#120s THEN
        FlowEstablished := TRUE;
    END_IF;
END_IF;

(* Only use ΔT when flow is established *)
IF FlowEstablished THEN
    DeltaT := FlowTemp - ReturnTemp;
    HeatDeliveryRate := FlowRate * 4186.0 * DeltaT;  (* Watts *)
END_IF;
```

## 6. What Physics Tells Us About Performance

### 6.1 Expected ΔT Values

For a properly operating zone:

| Condition | Expected ΔT | Interpretation |
|-----------|-------------|----------------|
| High heating demand | 8-12°C | Good heat transfer to room |
| Moderate demand | 5-8°C | Normal operation |
| Low demand | 2-5°C | Room nearing target |
| Maintenance mode | 0-2°C | Minimal heat loss |

**Problem Detection**:
```
ΔT < 2°C when heating: 
  → Insufficient flow rate OR valve partially open OR air lock

ΔT > 15°C when heating:
  → Excessive flow resistance OR zone valve stuck OR pump issue

ΔT negative (return > flow):
  → Sensor error OR reverse flow OR sensor swap
```

### 6.2 Heat Loss Coefficient

From measurements, we can calculate the zone's heat loss coefficient:

```
Heat Loss = U × A × (T_indoor - T_outdoor)

Where:
U = Overall heat transfer coefficient (W/m²·K)
A = Zone surface area (m²)
```

**Learning Algorithm**:
```
1. Measure: Indoor temp, Outdoor temp, Heat delivery rate
2. During steady state (temp stable):
   Heat Loss = Heat Delivery
3. Calculate: U × A = Heat_Delivery / (T_indoor - T_outdoor)
4. Store for predictive control
```

### 6.3 Comfort Index

Instead of controlling to air temperature alone, use a comfort index:

```
Comfort_Index = w_air × T_air + w_floor × T_floor

Typical weights:
w_air = 0.3-0.4
w_floor = 0.6-0.7
```

**Physical Basis**:
- Radiant heat from floor affects perceived comfort more than air temperature
- PMV (Predicted Mean Vote) comfort models include radiant temperature
- Floor temperature can be estimated from return temperature + offset

**Implementation**:
```iecst
(* Estimate floor surface temperature *)
FloorTemp := ReturnTemp + 3.0;  (* Typical offset *)

(* Calculate comfort index *)
ComfortIndex := 0.35 * AirTemp + 0.65 * FloorTemp;

(* Control to comfort target, not just air temp *)
IF ComfortIndex < (TargetComfort - Hysteresis/2) THEN
    ZoneValveOpen := TRUE;
END_IF;
```

## 7. Advanced Physics Considerations

### 7.1 Weather Compensation

Supply temperature should adapt to outdoor conditions:

```
T_supply = T_base + K × (T_indoor_target - T_outdoor)

Where:
K = Weather compensation curve (typically 0.3-0.6)
```

**Example**:
- Indoor target: 21°C
- Outdoor: 10°C
- K = 0.5
- T_supply = 35°C + 0.5 × (21-10) = 40.5°C

### 7.2 Inter-Zone Heat Transfer

Adjacent zones affect each other:

```
Q̇_zone_i = Σ U_wall_j × A_j × (T_zone_i - T_zone_j)
```

**Practical Impact**:
- Interior zones require less heating than perimeter zones
- Corner rooms lose more heat (two external walls)
- Heat from active zones migrates to inactive zones

## 8. Summary: Physics-Based Control Advantages

| Aspect | Traditional Thermostat | Physics-Based Control |
|--------|----------------------|----------------------|
| **Response Time** | Reactive (hours late) | Predictive (on time) |
| **Energy Efficiency** | Frequent cycling | Optimized supply temp |
| **Comfort** | Air temp only | Radiant + air |
| **Fault Detection** | User notices problem | Automatic detection |
| **Learning** | None | Adaptive to building |
| **Maintenance** | Reactive | Predictive |

## Conclusion

Physics-based control leverages:
1. ✅ **Multiple temperature measurements** to calculate actual heat transfer
2. ✅ **Thermal dynamics modeling** for predictive control
3. ✅ **Adaptive learning** of building characteristics
4. ✅ **Comfort indices** that include radiant heat
5. ✅ **Fault detection** through physics validation

The result is a system that is more comfortable, more efficient, and more reliable than traditional on/off thermostatic control.

---

**Next**: [02-UDT-Definitions.md](./02-UDT-Definitions.md) - User Defined Type structures for implementing this physics-based system.
