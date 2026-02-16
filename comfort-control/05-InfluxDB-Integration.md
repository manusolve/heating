# InfluxDB Integration for Time-Series Data Logging

## Introduction

This document describes the integration of InfluxDB 2.x as the time-series database for logging and analyzing heating system performance. InfluxDB is specifically optimized for time-series data, making it ideal for sensor readings, control states, and performance metrics.

## Why InfluxDB?

### Database Comparison

| Feature | InfluxDB 2.x | PostgreSQL + TimescaleDB | SQLite |
|---------|-------------|--------------------------|--------|
| **Time-Series Optimization** | ✅ Native | ⚠️ Extension Required | ❌ No |
| **Write Performance** | ✅ Excellent (100k+ pts/sec) | ⚠️ Good (10k+ pts/sec) | ❌ Limited |
| **Query Language** | Flux (purpose-built) | SQL (general) | SQL (basic) |
| **Downsampling** | ✅ Automatic | ⚠️ Manual | ❌ No |
| **Retention Policies** | ✅ Built-in | ⚠️ Manual | ❌ No |
| **Memory Footprint** | ⚠️ Moderate (100-200MB) | ⚠️ Moderate (50-150MB) | ✅ Low (< 10MB) |
| **Storage Compression** | ✅ Excellent (~10:1) | ⚠️ Good (~5:1) | ⚠️ Minimal |
| **Built-in Dashboards** | ✅ Yes | ❌ No | ❌ No |
| **API** | ✅ REST + Native | ⚠️ SQL only | ⚠️ File-based |
| **Scalability** | ✅ Horizontal | ⚠️ Vertical | ❌ Limited |
| **Learning Curve** | ⚠️ Moderate | ✅ SQL familiar | ✅ Easy |
| **Raspberry Pi Support** | ✅ Official ARM binaries | ✅ Yes | ✅ Built-in |

**Recommendation**: InfluxDB 2.x is the optimal choice for this application due to:
- Superior write performance for high-frequency sensor data
- Built-in retention and downsampling policies
- Excellent storage compression
- Native time-series query capabilities
- Integration with Grafana for visualization

## InfluxDB Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenPLC (C++ Layer)                       │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Main Program (IEC 61131-3 Structured Text)           │ │
│  │  • Runs every 100ms                                    │ │
│  │  • Calls external C++ logging functions                │ │
│  └──────────────────────┬─────────────────────────────────┘ │
│                         │                                    │
│  ┌──────────────────────▼─────────────────────────────────┐ │
│  │  C++ Logging Functions (libcurl)                       │ │
│  │  • log_zone_temperature()                              │ │
│  │  • log_zone_state()                                    │ │
│  │  • log_zone_physics()                                  │ │
│  │  • log_zone_learning()                                 │ │
│  └──────────────────────┬─────────────────────────────────┘ │
└────────────────────────┬───────────────────────────────────┘
                         │ HTTP POST (Line Protocol)
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              InfluxDB 2.x (Port 8086)                        │
│                                                              │
│  Bucket: heating_system                                      │
│  ├─ zone_temperatures (raw data)                            │
│  ├─ zone_state (valve, demand)                              │
│  ├─ zone_physics (heat delivery, comfort)                   │
│  └─ zone_learning (learned parameters)                      │
│                                                              │
│  Retention: 90 days full resolution                          │
│  Downsampling: 5min averages → 1 year retention             │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                Grafana (Port 3000)                           │
│  • Real-time dashboards                                      │
│  • Historical analysis                                       │
│  • Alert notifications                                       │
└─────────────────────────────────────────────────────────────┘
```

## Schema Design

### Measurement 1: zone_temperatures

Stores all temperature sensor readings.

**Line Protocol Format**:
```
zone_temperatures,zone_id=1,sensor_type=0 temperature=21.45,is_valid=true 1644854400000000000
```

**Schema**:
```
Measurement: zone_temperatures

Tags (indexed):
  - zone_id       : INT (1-12)
  - sensor_type   : INT (0=Air, 1=Flow, 2=Return, 3=Outdoor)
  - zone_name     : STRING (e.g., "Living Room")

Fields (values):
  - temperature   : FLOAT (°C)
  - is_valid      : BOOLEAN
  - error_count   : INTEGER

Timestamp: Nanosecond precision
```

**Flux Query Example** - Average temperature per zone:
```flux
from(bucket: "heating_system")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "zone_temperatures")
  |> filter(fn: (r) => r.sensor_type == "0")  // Air sensors only
  |> group(columns: ["zone_id", "zone_name"])
  |> mean()
  |> yield(name: "avg_temp")
```

### Measurement 2: zone_state

Stores zone control state and valve actuation.

**Line Protocol Format**:
```
zone_state,zone_id=1,zone_name="Living Room" valve_open=true,heating_demand=true,comfort_achieved=false,flow_established=true 1644854400000000000
```

**Schema**:
```
Measurement: zone_state

Tags:
  - zone_id       : INT
  - zone_name     : STRING

Fields:
  - valve_open          : BOOLEAN
  - heating_demand      : BOOLEAN
  - comfort_achieved    : BOOLEAN
  - flow_established    : BOOLEAN
  - overheat_warning    : BOOLEAN
  - sensor_error        : BOOLEAN

Timestamp: Nanosecond precision
```

**Flux Query Example** - Valve duty cycle:
```flux
from(bucket: "heating_system")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "zone_state")
  |> filter(fn: (r) => r._field == "valve_open")
  |> group(columns: ["zone_id"])
  |> mean()
  |> map(fn: (r) => ({ r with duty_cycle: r._value * 100.0 }))
  |> yield(name: "valve_duty_cycle")
```

### Measurement 3: zone_physics

Stores calculated physics values.

**Line Protocol Format**:
```
zone_physics,zone_id=1 delta_t=8.5,heat_delivery_rate=2150.0,comfort_index=21.2,comfort_deviation=-0.2 1644854400000000000
```

**Schema**:
```
Measurement: zone_physics

Tags:
  - zone_id       : INT
  - zone_name     : STRING

Fields:
  - delta_t               : FLOAT (°C)
  - heat_delivery_rate    : FLOAT (Watts)
  - cumulative_heat       : FLOAT (kWh)
  - comfort_index         : FLOAT (°C)
  - comfort_deviation     : FLOAT (°C)
  - predicted_time_target : FLOAT (seconds)
  - est_floor_temp        : FLOAT (°C)

Timestamp: Nanosecond precision
```

**Flux Query Example** - Total energy by zone:
```flux
from(bucket: "heating_system")
  |> range(start: -30d)
  |> filter(fn: (r) => r._measurement == "zone_physics")
  |> filter(fn: (r) => r._field == "cumulative_heat")
  |> group(columns: ["zone_id", "zone_name"])
  |> last()
  |> yield(name: "total_energy")
```

### Measurement 4: zone_learning

Stores learned parameters from adaptive algorithms.

**Line Protocol Format**:
```
zone_learning,zone_id=1 heat_loss_rate=450.0,thermal_time_constant=7200,optimal_supply_temp=42.5,is_calibrated=true 1644854400000000000
```

**Schema**:
```
Measurement: zone_learning

Tags:
  - zone_id       : INT
  - zone_name     : STRING

Fields:
  - heat_loss_rate        : FLOAT (W/K)
  - thermal_time_constant : FLOAT (seconds)
  - optimal_supply_temp   : FLOAT (°C)
  - is_calibrated         : BOOLEAN
  - sample_count          : INTEGER
  - calibration_quality   : FLOAT (0.0-1.0)
  - energy_per_degree     : FLOAT (kWh/°C·day)

Timestamp: Nanosecond precision
```

**Flux Query Example** - Compare learned heat loss rates:
```flux
from(bucket: "heating_system")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "zone_learning")
  |> filter(fn: (r) => r._field == "heat_loss_rate")
  |> group(columns: ["zone_id", "zone_name"])
  |> last()
  |> sort(columns: ["_value"], desc: true)
  |> yield(name: "heat_loss_comparison")
```

## C++ Integration Code

### Header File: influxdb_logger.h

```cpp
#ifndef INFLUXDB_LOGGER_H
#define INFLUXDB_LOGGER_H

#include <string>
#include <curl/curl.h>

class InfluxDBLogger {
private:
    std::string url;
    std::string token;
    std::string org;
    std::string bucket;
    
    CURL* curl;
    struct curl_slist* headers;
    
    bool post_line_protocol(const std::string& data);
    
public:
    InfluxDBLogger(
        const std::string& host,
        int port,
        const std::string& token,
        const std::string& org,
        const std::string& bucket
    );
    
    ~InfluxDBLogger();
    
    bool log_zone_temperature(
        int zone_id,
        const std::string& zone_name,
        int sensor_type,
        float temperature,
        bool is_valid,
        int error_count
    );
    
    bool log_zone_state(
        int zone_id,
        const std::string& zone_name,
        bool valve_open,
        bool heating_demand,
        bool comfort_achieved,
        bool flow_established,
        bool overheat_warning,
        bool sensor_error
    );
    
    bool log_zone_physics(
        int zone_id,
        const std::string& zone_name,
        float delta_t,
        float heat_delivery_rate,
        float cumulative_heat,
        float comfort_index,
        float comfort_deviation,
        float est_floor_temp
    );
    
    bool log_zone_learning(
        int zone_id,
        const std::string& zone_name,
        float heat_loss_rate,
        float thermal_time_constant,
        float optimal_supply_temp,
        bool is_calibrated,
        int sample_count,
        float calibration_quality
    );
};

#endif // INFLUXDB_LOGGER_H
```

### Implementation: influxdb_logger.cpp

```cpp
#include "influxdb_logger.h"
#include <sstream>
#include <chrono>
#include <iostream>

InfluxDBLogger::InfluxDBLogger(
    const std::string& host,
    int port,
    const std::string& token,
    const std::string& org,
    const std::string& bucket
) : token(token), org(org), bucket(bucket) {
    
    // Build URL
    std::stringstream ss;
    ss << "http://" << host << ":" << port 
       << "/api/v2/write?org=" << org 
       << "&bucket=" << bucket
       << "&precision=ns";
    url = ss.str();
    
    // Initialize curl
    curl = curl_easy_init();
    if (!curl) {
        std::cerr << "Failed to initialize curl" << std::endl;
        return;
    }
    
    // Setup headers
    headers = NULL;
    std::string auth_header = "Authorization: Token " + token;
    headers = curl_slist_append(headers, auth_header.c_str());
    headers = curl_slist_append(headers, "Content-Type: text/plain");
    
    curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);
    curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
}

InfluxDBLogger::~InfluxDBLogger() {
    if (headers) {
        curl_slist_free_all(headers);
    }
    if (curl) {
        curl_easy_cleanup(curl);
    }
}

bool InfluxDBLogger::post_line_protocol(const std::string& data) {
    if (!curl) {
        return false;
    }
    
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, data.c_str());
    
    CURLcode res = curl_easy_perform(curl);
    
    if (res != CURLE_OK) {
        std::cerr << "curl_easy_perform() failed: " 
                  << curl_easy_strerror(res) << std::endl;
        return false;
    }
    
    return true;
}

bool InfluxDBLogger::log_zone_temperature(
    int zone_id,
    const std::string& zone_name,
    int sensor_type,
    float temperature,
    bool is_valid,
    int error_count
) {
    // Get current timestamp in nanoseconds
    auto now = std::chrono::system_clock::now();
    auto nanos = std::chrono::duration_cast<std::chrono::nanoseconds>(
        now.time_since_epoch()
    ).count();
    
    // Build line protocol
    std::stringstream ss;
    ss << "zone_temperatures,"
       << "zone_id=" << zone_id << ","
       << "sensor_type=" << sensor_type << ","
       << "zone_name=" << zone_name
       << " "
       << "temperature=" << temperature << ","
       << "is_valid=" << (is_valid ? "true" : "false") << ","
       << "error_count=" << error_count << "i"
       << " " << nanos;
    
    return post_line_protocol(ss.str());
}

bool InfluxDBLogger::log_zone_state(
    int zone_id,
    const std::string& zone_name,
    bool valve_open,
    bool heating_demand,
    bool comfort_achieved,
    bool flow_established,
    bool overheat_warning,
    bool sensor_error
) {
    auto now = std::chrono::system_clock::now();
    auto nanos = std::chrono::duration_cast<std::chrono::nanoseconds>(
        now.time_since_epoch()
    ).count();
    
    std::stringstream ss;
    ss << "zone_state,"
       << "zone_id=" << zone_id << ","
       << "zone_name=" << zone_name
       << " "
       << "valve_open=" << (valve_open ? "true" : "false") << ","
       << "heating_demand=" << (heating_demand ? "true" : "false") << ","
       << "comfort_achieved=" << (comfort_achieved ? "true" : "false") << ","
       << "flow_established=" << (flow_established ? "true" : "false") << ","
       << "overheat_warning=" << (overheat_warning ? "true" : "false") << ","
       << "sensor_error=" << (sensor_error ? "true" : "false")
       << " " << nanos;
    
    return post_line_protocol(ss.str());
}

bool InfluxDBLogger::log_zone_physics(
    int zone_id,
    const std::string& zone_name,
    float delta_t,
    float heat_delivery_rate,
    float cumulative_heat,
    float comfort_index,
    float comfort_deviation,
    float est_floor_temp
) {
    auto now = std::chrono::system_clock::now();
    auto nanos = std::chrono::duration_cast<std::chrono::nanoseconds>(
        now.time_since_epoch()
    ).count();
    
    std::stringstream ss;
    ss << "zone_physics,"
       << "zone_id=" << zone_id << ","
       << "zone_name=" << zone_name
       << " "
       << "delta_t=" << delta_t << ","
       << "heat_delivery_rate=" << heat_delivery_rate << ","
       << "cumulative_heat=" << cumulative_heat << ","
       << "comfort_index=" << comfort_index << ","
       << "comfort_deviation=" << comfort_deviation << ","
       << "est_floor_temp=" << est_floor_temp
       << " " << nanos;
    
    return post_line_protocol(ss.str());
}

bool InfluxDBLogger::log_zone_learning(
    int zone_id,
    const std::string& zone_name,
    float heat_loss_rate,
    float thermal_time_constant,
    float optimal_supply_temp,
    bool is_calibrated,
    int sample_count,
    float calibration_quality
) {
    auto now = std::chrono::system_clock::now();
    auto nanos = std::chrono::duration_cast<std::chrono::nanoseconds>(
        now.time_since_epoch()
    ).count();
    
    std::stringstream ss;
    ss << "zone_learning,"
       << "zone_id=" << zone_id << ","
       << "zone_name=" << zone_name
       << " "
       << "heat_loss_rate=" << heat_loss_rate << ","
       << "thermal_time_constant=" << thermal_time_constant << ","
       << "optimal_supply_temp=" << optimal_supply_temp << ","
       << "is_calibrated=" << (is_calibrated ? "true" : "false") << ","
       << "sample_count=" << sample_count << "i,"
       << "calibration_quality=" << calibration_quality
       << " " << nanos;
    
    return post_line_protocol(ss.str());
}
```

### Usage in OpenPLC Hardware Layer

```cpp
// In raspberrypi_1wire.cpp or custom hardware layer

#include "influxdb_logger.h"

// Global logger instance
InfluxDBLogger* logger = nullptr;

// Initialize in hardware initialization function
void initializeIO() {
    // ... existing initialization code ...
    
    // Initialize InfluxDB logger
    logger = new InfluxDBLogger(
        "localhost",           // InfluxDB host
        8086,                  // InfluxDB port
        "YOUR_TOKEN_HERE",     // API token
        "home",                // Organization
        "heating_system"       // Bucket name
    );
}

// Call from PLC program (exposed as external C function)
extern "C" void log_zone_data(
    int zone_id,
    const char* zone_name,
    float air_temp,
    float flow_temp,
    float return_temp,
    bool valve_open,
    float heat_rate,
    float comfort_index
) {
    if (logger) {
        // Log temperatures
        logger->log_zone_temperature(zone_id, zone_name, 0, air_temp, true, 0);
        logger->log_zone_temperature(zone_id, zone_name, 1, flow_temp, true, 0);
        logger->log_zone_temperature(zone_id, zone_name, 2, return_temp, true, 0);
        
        // Log state
        logger->log_zone_state(zone_id, zone_name, valve_open, 
                               valve_open, false, valve_open, false, false);
        
        // Log physics
        float delta_t = flow_temp - return_temp;
        logger->log_zone_physics(zone_id, zone_name, delta_t, 
                                 heat_rate, 0.0, comfort_index, 0.0, return_temp + 3.0);
    }
}
```

## Advanced Flux Query Examples

### 1. Detect Zones with Poor Heat Transfer

Identifies zones where ΔT is consistently low, indicating flow problems.

```flux
from(bucket: "heating_system")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "zone_physics")
  |> filter(fn: (r) => r._field == "delta_t")
  |> group(columns: ["zone_id", "zone_name"])
  |> mean()
  |> filter(fn: (r) => r._value < 3.0)  // Less than 3°C average
  |> yield(name: "poor_heat_transfer")
```

### 2. Seasonal Heat Loss Analysis

Compares heat loss between summer and winter.

```flux
import "timezone"

winter_data = from(bucket: "heating_system")
  |> range(start: 2025-12-01T00:00:00Z, stop: 2026-02-28T23:59:59Z)
  |> filter(fn: (r) => r._measurement == "zone_learning")
  |> filter(fn: (r) => r._field == "heat_loss_rate")
  |> group(columns: ["zone_id"])
  |> mean()
  |> map(fn: (r) => ({ r with season: "winter" }))

summer_data = from(bucket: "heating_system")
  |> range(start: 2025-06-01T00:00:00Z, stop: 2025-08-31T23:59:59Z)
  |> filter(fn: (r) => r._measurement == "zone_learning")
  |> filter(fn: (r) => r._field == "heat_loss_rate")
  |> group(columns: ["zone_id"])
  |> mean()
  |> map(fn: (r) => ({ r with season: "summer" }))

union(tables: [winter_data, summer_data])
  |> yield(name: "seasonal_comparison")
```

### 3. Calculate Average Comfort Achievement

Percentage of time each zone maintains target comfort.

```flux
from(bucket: "heating_system")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "zone_state")
  |> filter(fn: (r) => r._field == "comfort_achieved")
  |> group(columns: ["zone_id", "zone_name"])
  |> mean()
  |> map(fn: (r) => ({ r with percent: r._value * 100.0 }))
  |> yield(name: "comfort_percentage")
```

### 4. Energy Cost Analysis

Calculate daily energy consumption per zone.

```flux
from(bucket: "heating_system")
  |> range(start: -30d)
  |> filter(fn: (r) => r._measurement == "zone_physics")
  |> filter(fn: (r) => r._field == "heat_delivery_rate")
  |> aggregateWindow(every: 1d, fn: mean)
  |> map(fn: (r) => ({ 
      r with 
      daily_kwh: r._value * 24.0 / 1000.0,
      daily_cost: r._value * 24.0 / 1000.0 * 0.15  // £0.15/kWh
  }))
  |> yield(name: "daily_energy_cost")
```

## Retention Policies and Downsampling

### Configure Retention Policy

```bash
# Keep full resolution data for 90 days
influx bucket update \
  --id YOUR_BUCKET_ID \
  --retention 90d

# Create downsampled bucket for long-term storage
influx bucket create \
  --name heating_system_downsampled \
  --retention 365d
```

### Create Downsampling Task

```flux
option task = {
  name: "downsample_heating_data",
  every: 1h,
}

from(bucket: "heating_system")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "zone_temperatures")
  |> aggregateWindow(every: 5m, fn: mean)
  |> to(bucket: "heating_system_downsampled")
```

## Conclusion

InfluxDB integration provides:

✅ **High-performance logging** for real-time sensor data  
✅ **Flexible schema** for all measurement types  
✅ **Powerful queries** using Flux language  
✅ **Automatic retention** and downsampling  
✅ **Seamless Grafana integration** for visualization  

---

**Next**: [06-Installation-Guide.md](./06-Installation-Guide.md) - Complete setup and deployment procedures.
