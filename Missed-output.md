# Physics-Based Comfort Control System

## UDT Definitions
### T_ZoneSensor
```structured-text
TYPE T_ZoneSensor : STRUCT
    temperature : REAL;
    humidity : REAL;
END_STRUCT;
END_TYPE
```

### T_ZoneConfig
```structured-text
TYPE T_ZoneConfig : STRUCT
    setpoint : REAL;
    control_mode : INT;
END_STRUCT;
END_TYPE
```

### T_ZoneState
```structured-text
TYPE T_ZoneState : STRUCT
    current_temperature : REAL;
    target_temperature : REAL;
    heating_status : BOOL;
END_STRUCT;
END_TYPE
```

### T_ZonePhysics
```structured-text
TYPE T_ZonePhysics : STRUCT
    thermal_mass : REAL;
    insulation : REAL;
END_STRUCT;
END_TYPE
```

### T_ZoneLearning
```structured-text
TYPE T_ZoneLearning : STRUCT
    comfort_index : REAL;
    learning_rate : REAL;
END_STRUCT;
END_TYPE
```

### T_Zone
```structured-text
TYPE T_Zone : STRUCT
    sensor : T_ZoneSensor;
    config : T_ZoneConfig;
    state : T_ZoneState;
    physics : T_ZonePhysics;
    learning : T_ZoneLearning;
END_STRUCT;
END_TYPE
```

### T_Manifold
```structured-text
TYPE T_Manifold : STRUCT
    zones : ARRAY[1..10] OF T_Zone;
END_STRUCT;
END_TYPE
```

## Function Blocks
### FC_ReadZoneSensors
```structured-text
FUNCTION_BLOCK FC_ReadZoneSensors
VAR_INPUT
    zone : T_Zone;
END_VAR
VAR_OUTPUT
    sensor_data : T_ZoneSensor;
END_VAR

// Reading from sensors logic
END_FUNCTION_BLOCK
```

### FC_CalculateZonePhysics
```structured-text
FUNCTION_BLOCK FC_CalculateZonePhysics
VAR_INPUT
    zone : T_Zone;
END_VAR
// Calculation logic for physics parameters
END_FUNCTION_BLOCK
```

### FC_ZoneControl
```structured-text
FUNCTION_BLOCK FC_ZoneControl
VAR_INPUT
    zone : T_Zone;
END_VAR
// Control logic for the zone
END_FUNCTION_BLOCK
```

### FC_ZoneLearning
```structured-text
FUNCTION_BLOCK FC_ZoneLearning
VAR_INPUT
    zone : T_Zone;
END_VAR
// Learning logic for zone temperature
END_FUNCTION_BLOCK
```

### FB_ManifoldControl
```structured-text
FUNCTION_BLOCK FB_ManifoldControl
VAR_INPUT
    manifold : T_Manifold;
END_VAR
// Control logic for manifold of zones
END_FUNCTION_BLOCK
```

## Main Cyclic Program
### PRG_HeatingSystem
```structured-text
PROGRAM PRG_HeatingSystem
VAR
    manifold : T_Manifold;
END_VAR
// Main cyclic logic
END_PROGRAM
```

## Time-Series Database Comparison
| Feature | InfluxDB | PostgreSQL/TimescaleDB |
|---------|----------|------------------------|
| Performance | High     | Moderate               |
| Scalability | Excellent | Good                  |
| Query Language | Flux    | SQL                    |

## InfluxDB Schema Design
### Measurements
- **zone_temperatures**: Stores temperature readings for each zone.
- **zone_state**: Stores the current state data for zoning.
- **zone_physics**: Stores physical parameters associated with the zones.
- **zone_learning**: Stores learning data regarding comfort indices.

## Example Flux Queries
### Average Comfort Index
```flux
from(bucket: "comfort")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "zone_learning")
  |> mean()
```

### Detecting Poor Heat Transfer
```flux
from(bucket: "zone_temperatures")
  |> range(start: -1h)
  |> filter(fn: (r) => r._field == "temperature")
  |> yield(name: "Poor Heat Transfer")
```

### Seasonal Heat Loss Analysis
```flux
from(bucket: "energy")
  |> range(start: 2025-09-01T00:00:00Z, stop: 2026-02-01T00:00:00Z)
  |> sum()
```

## C++ Integration Code for Logging to InfluxDB
```cpp
#include <curl/curl.h>

void logToInfluxDB(const char* data) {
    CURL* curl;
    CURLcode res;

    curl = curl_easy_init();
    if(curl) {
        curl_easy_setopt(curl, CURLOPT_URL, "http://localhost:8086/write?db=mydb");
        curl_easy_setopt(curl, CURLOPT_POSTFIELDS, data);

        res = curl_easy_perform(curl);
        curl_easy_cleanup(curl);
    }
}
```
