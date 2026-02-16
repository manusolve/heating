# Installation and Deployment Guide

## Introduction

This comprehensive guide walks you through the complete setup of the Physics-Based Comfort Control System on a Raspberry Pi, from initial hardware configuration to production deployment.

## Prerequisites

### Hardware Requirements

- **Raspberry Pi** 3B+ or newer (4B recommended)
- **MicroSD Card** 32GB minimum, Class 10 or better
- **Power Supply** 5V/3A USB-C (Pi 4) or Micro USB (Pi 3)
- **DS18B20 Temperature Sensors** (2-3 per zone)
- **Zone Valve Actuators** (230VAC or 24VAC motorized)
- **Relay Board** for valve control (8-16 channels)
- **4.7kΩ Resistor** for 1-Wire pull-up
- **Ethernet Cable** (recommended over WiFi for reliability)
- **DIN Rail Enclosure** (optional, for professional installation)

### Software Requirements

- **Raspberry Pi OS** (64-bit recommended)
- **OpenPLC Runtime** v3+
- **InfluxDB** 2.x
- **Grafana** 9.x or newer
- **Git** for repository management

## Installation Steps

### Step 1: Raspberry Pi Setup

#### 1.1 Install Raspberry Pi OS

```bash
# Download Raspberry Pi Imager
# https://www.raspberrypi.com/software/

# Flash Raspberry Pi OS (64-bit) to SD card
# Enable SSH in imager settings
# Configure WiFi/Ethernet credentials
```

#### 1.2 Initial Configuration

```bash
# SSH into Raspberry Pi
ssh pi@raspberrypi.local
# Default password: raspberry

# Update system
sudo apt update
sudo apt upgrade -y

# Configure system
sudo raspi-config
# - Change password
# - Set hostname (e.g., "heating-plc")
# - Set timezone
# - Expand filesystem

# Reboot
sudo reboot
```

#### 1.3 Enable 1-Wire Interface

```bash
# Edit boot configuration
sudo nano /boot/config.txt

# Add at the end:
dtoverlay=w1-gpio,gpiopin=23

# Save (Ctrl+X, Y, Enter) and reboot
sudo reboot

# Verify 1-wire is enabled
lsmod | grep w1
# Should show: w1_gpio, w1_therm

# Check for sensors
ls /sys/bus/w1/devices/
# Should show: 28-XXXXXXXXXXXX (DS18B20 IDs)
```

### Step 2: OpenPLC Installation

#### 2.1 Install Dependencies

```bash
# Install required packages
sudo apt install -y git sqlite3 cmake \
    g++ libmodbus-dev libssl-dev \
    python3-pip

# Install Node.js (for web interface)
curl -fsSL https://deb.nodesource.com/setup_16.x | sudo -E bash -
sudo apt install -y nodejs
```

#### 2.2 Clone and Install OpenPLC

```bash
# Clone repository
cd ~
git clone https://github.com/thiagoralves/OpenPLC_v3.git
cd OpenPLC_v3

# Run installation script
./install.sh linux

# When prompted:
# - Select "Raspberry Pi" as hardware platform
# - Enable compilation of examples: No
# - Web server port: 8080 (default)
```

#### 2.3 Install Custom 1-Wire Hardware Layer

```bash
# Navigate to hardware layers directory
cd ~/OpenPLC_v3/webserver/core/hardware_layers

# Create custom 1-wire layer
sudo nano raspberrypi_1wire.cpp

# Paste the complete C++ code from the main README.md
# (See repository documentation for full code)

# Edit hardware layer selection
cd ..
sudo nano hardware_layer.cpp

# Change the include line to:
#include "hardware_layers/raspberrypi_1wire.cpp"

# Save and exit
```

#### 2.4 Compile OpenPLC

```bash
cd ~/OpenPLC_v3/webserver/core
./compile_program.sh

# Check for successful compilation
# Should see: "Compilation finished successfully!"
```

#### 2.5 Start OpenPLC Service

```bash
# Start OpenPLC
cd ~/OpenPLC_v3/webserver
sudo ./start_openplc.sh

# Verify service is running
sudo systemctl status openplc

# Enable auto-start on boot
sudo systemctl enable openplc
```

#### 2.6 Access OpenPLC Web Interface

```bash
# Open browser and navigate to:
http://raspberrypi.local:8080

# Default credentials:
# Username: openplc
# Password: openplc

# Change password immediately!
# Hardware → Settings → Change Password
```

### Step 3: Deploy Heating Control Program

#### 3.1 Create Structured Text Program

Create a new file `HeatingControl.st` with the complete program from [04-Main-Program.md](./04-Main-Program.md).

```bash
# Create program file
nano ~/HeatingControl.st

# Paste complete program from documentation
# Save and exit
```

#### 3.2 Upload to OpenPLC

1. Open OpenPLC web interface
2. Navigate to **Programs** → **Upload Program**
3. Click **Choose File** and select `HeatingControl.st`
4. Click **Upload**
5. Wait for compilation (may take 1-2 minutes)
6. Click **Start PLC** when compilation completes

#### 3.3 Verify Program Execution

1. Navigate to **Monitoring** page
2. Observe %IW registers updating with sensor values
   - %IW0: Zone 1 Air Temp (e.g., 2145 = 21.45°C)
   - %IW1: Zone 1 Flow Temp
   - %IW2: Zone 1 Return Temp
   - etc.
3. Check %QX outputs showing valve states
   - %QX0.0: Zone 1 Valve (TRUE/FALSE)
   - %QX0.1: Zone 2 Valve
   - etc.

### Step 4: InfluxDB Installation

#### 4.1 Install InfluxDB 2.x

```bash
# Add InfluxDB repository
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list

# Update and install
sudo apt update
sudo apt install -y influxdb2

# Start InfluxDB service
sudo systemctl start influxdb
sudo systemctl enable influxdb

# Verify service
sudo systemctl status influxdb
```

#### 4.2 Configure InfluxDB

```bash
# Open web interface
# Navigate to: http://raspberrypi.local:8086

# Initial setup wizard:
# 1. Username: admin
# 2. Password: (choose strong password)
# 3. Organization: home
# 4. Bucket: heating_system
# 5. Click "Configure Later"

# Copy API token (save securely!)
# Settings → API Tokens → admin's Token
```

#### 4.3 Create Retention Policies

```bash
# Install InfluxDB CLI
sudo apt install -y influxdb2-cli

# Configure CLI
influx config create \
  --config-name default \
  --host-url http://localhost:8086 \
  --org home \
  --token YOUR_API_TOKEN_HERE \
  --active

# Update bucket retention
influx bucket update \
  --name heating_system \
  --retention 90d

# Create downsampled bucket
influx bucket create \
  --name heating_system_downsampled \
  --retention 365d
```

#### 4.4 Create Downsampling Task

```bash
# Create task for automatic downsampling
influx task create \
  --org home \
  - <<EOF
option task = {name: "downsample_heating", every: 1h}

from(bucket: "heating_system")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "zone_temperatures")
  |> aggregateWindow(every: 5m, fn: mean)
  |> to(bucket: "heating_system_downsampled")
EOF
```

### Step 5: Grafana Installation

#### 5.1 Install Grafana

```bash
# Add Grafana repository
sudo apt install -y software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

# Install
sudo apt update
sudo apt install -y grafana

# Start service
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# Verify
sudo systemctl status grafana-server
```

#### 5.2 Configure Grafana

```bash
# Access Grafana web interface
# Navigate to: http://raspberrypi.local:3000

# Default credentials:
# Username: admin
# Password: admin
# (Change password when prompted)
```

#### 5.3 Add InfluxDB Data Source

1. Click **Configuration** (gear icon) → **Data Sources**
2. Click **Add data source**
3. Select **InfluxDB**
4. Configure:
   - **Name**: InfluxDB Heating
   - **Query Language**: Flux
   - **URL**: http://localhost:8086
   - **Organization**: home
   - **Token**: (paste API token from Step 4.2)
   - **Default Bucket**: heating_system
5. Click **Save & Test** (should show green success)

#### 5.4 Import Dashboard

Create a basic dashboard:

1. Click **Dashboards** → **Import**
2. Click **Upload JSON file**
3. Create `heating_dashboard.json`:

```json
{
  "dashboard": {
    "title": "Heating System Overview",
    "panels": [
      {
        "title": "Zone Temperatures",
        "type": "timeseries",
        "targets": [
          {
            "query": "from(bucket: \"heating_system\") |> range(start: -1h) |> filter(fn: (r) => r._measurement == \"zone_temperatures\") |> filter(fn: (r) => r.sensor_type == \"0\")"
          }
        ]
      },
      {
        "title": "Valve States",
        "type": "state-timeline",
        "targets": [
          {
            "query": "from(bucket: \"heating_system\") |> range(start: -1h) |> filter(fn: (r) => r._measurement == \"zone_state\") |> filter(fn: (r) => r._field == \"valve_open\")"
          }
        ]
      },
      {
        "title": "Heat Delivery",
        "type": "timeseries",
        "targets": [
          {
            "query": "from(bucket: \"heating_system\") |> range(start: -1h) |> filter(fn: (r) => r._measurement == \"zone_physics\") |> filter(fn: (r) => r._field == \"heat_delivery_rate\")"
          }
        ]
      }
    ]
  }
}
```

### Step 6: Integrate InfluxDB Logging

#### 6.1 Install libcurl

```bash
sudo apt install -y libcurl4-openssl-dev
```

#### 6.2 Add Logging Code to Hardware Layer

Edit `~/OpenPLC_v3/webserver/core/hardware_layers/raspberrypi_1wire.cpp`:

```cpp
// Add at the top (after includes)
#include "influxdb_logger.h"

// Add global logger
InfluxDBLogger* g_logger = nullptr;

// In initializeIO() function:
g_logger = new InfluxDBLogger(
    "localhost",
    8086,
    "YOUR_API_TOKEN_HERE",  // Replace with actual token
    "home",
    "heating_system"
);

// Add logging function (called from PLC program)
extern "C" void log_zone_temperatures() {
    if (g_logger && pthread_mutex_trylock(&bufferLock) == 0) {
        for (int i = 0; i < MAX_SENSORS; i++) {
            if (sensorBuffers[i].valid) {
                g_logger->log_zone_temperature(
                    i / 3,  // Zone ID (3 sensors per zone)
                    "Zone",
                    i % 3,  // Sensor type
                    sensorBuffers[i].temperature,
                    true,
                    0
                );
            }
        }
        pthread_mutex_unlock(&bufferLock);
    }
}
```

#### 6.3 Compile with Logging Support

```bash
cd ~/OpenPLC_v3/webserver/core
./compile_program.sh
```

### Step 7: Testing and Commissioning

#### 7.1 Sensor Testing

```bash
# Verify all sensors detected
ls /sys/bus/w1/devices/ | grep 28-

# Read each sensor manually
for sensor in /sys/bus/w1/devices/28-*; do
    echo "Reading $sensor:"
    cat $sensor/w1_slave
done

# Expected output:
# ... YES
# ... t=21875  (21.875°C)
```

#### 7.2 Single Zone Test

1. In OpenPLC web interface, navigate to **Monitoring**
2. Enable only Zone 1
3. Set target temperature to 2°C **above** current temperature
4. Observe:
   - Zone 1 valve opens (%QX0.0 = TRUE)
   - Manifold pump starts (%QX100 = TRUE)
   - Flow temp rises (check %IW1)
   - After 2 minutes, flow established
   - Heat delivery rate increases (check database)

#### 7.3 Multi-Zone Test

1. Enable multiple zones (2-3)
2. Set different target temperatures
3. Observe:
   - Valves open/close independently
   - Pump runs when any zone demands
   - Bypass closes when zones active
   - No short-cycling (minimum run times respected)

#### 7.4 Learning Verification

1. Let system run for 24 hours
2. Check learning data in InfluxDB:

```flux
from(bucket: "heating_system")
  |> range(start: -24h)
  |> filter(fn: (r) => r._measurement == "zone_learning")
  |> filter(fn: (r) => r._field == "sample_count")
  |> last()
```

3. Verify sample counts increasing
4. Check calibration status after 100+ samples

### Step 8: Production Deployment

#### 8.1 System Service Configuration

Create systemd service for monitoring:

```bash
sudo nano /etc/systemd/system/heating-monitor.service

# Add:
[Unit]
Description=Heating System Monitor
After=network.target openplc.service influxdb.service

[Service]
Type=simple
User=pi
ExecStart=/home/pi/heating_monitor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

# Enable service
sudo systemctl enable heating-monitor
sudo systemctl start heating-monitor
```

#### 8.2 Backup Configuration

```bash
# Backup script
cat > ~/backup_heating.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/home/pi/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup OpenPLC program
cp ~/HeatingControl.st $BACKUP_DIR/HeatingControl_$DATE.st

# Backup InfluxDB
influx backup $BACKUP_DIR/influxdb_$DATE

# Backup learning data (from OpenPLC persistent memory)
# This is stored in OpenPLC's internal database

echo "Backup completed: $DATE"
EOF

chmod +x ~/backup_heating.sh

# Add to cron (daily at 2 AM)
crontab -e
# Add: 0 2 * * * /home/pi/backup_heating.sh
```

#### 8.3 Configure Alerts

Create alert rules in Grafana:

1. **Sensor Failure Alert**
   - Condition: No data for sensor > 5 minutes
   - Action: Send notification

2. **Temperature Extreme Alert**
   - Condition: Any zone < 5°C or > 30°C
   - Action: Send notification + SMS

3. **Pump Failure Alert**
   - Condition: Pump running but no flow for > 10 minutes
   - Action: Send notification

### Step 9: Maintenance Procedures

#### 9.1 Weekly Checks

```bash
# Check system status
sudo systemctl status openplc influxdb grafana-server

# Check disk space
df -h

# Check log files
tail -f /var/log/openplc.log
```

#### 9.2 Monthly Maintenance

1. Review learning data quality
2. Check valve cycle counts (maintenance indicator)
3. Verify sensor calibration
4. Review energy consumption trends
5. Update system software:

```bash
sudo apt update
sudo apt upgrade -y
```

#### 9.3 Seasonal Tasks

**Summer**:
- Reduce target temperatures
- Check for unexpected heating demand
- Clean filters

**Winter**:
- Verify freeze protection working
- Check outdoor sensor accuracy
- Review heat loss rates vs. last year

## Troubleshooting

### Problem: No Sensor Readings

**Diagnosis**:
```bash
# Check 1-wire interface
lsmod | grep w1

# Check sensor directory
ls /sys/bus/w1/devices/

# Read sensor directly
cat /sys/bus/w1/devices/28-*/w1_slave
```

**Solutions**:
- Check wiring (VDD, Data, GND)
- Verify 4.7kΩ pull-up resistor installed
- Check GPIO pin configured correctly (gpiopin=23)
- Try different sensor or cable

### Problem: OpenPLC Won't Start

**Diagnosis**:
```bash
sudo systemctl status openplc
journalctl -u openplc -n 50
```

**Solutions**:
- Check compilation errors in log
- Verify hardware layer code compiled correctly
- Check port 8080 not already in use: `sudo netstat -tulpn | grep 8080`
- Restart service: `sudo systemctl restart openplc`

### Problem: Valves Not Actuating

**Diagnosis**:
1. Check %QX outputs in OpenPLC monitoring page
2. Check relay board wiring
3. Measure voltage at relay outputs

**Solutions**:
- Verify valve addresses match %QX assignments
- Check relay board power supply
- Test relay board manually (close relay with wire)
- Check valve actuator power (24V/230V)

### Problem: InfluxDB Not Logging

**Diagnosis**:
```bash
# Check InfluxDB status
sudo systemctl status influxdb

# Check connection
curl http://localhost:8086/health

# Check logs
journalctl -u influxdb -n 50
```

**Solutions**:
- Verify API token is correct in code
- Check network connectivity to InfluxDB
- Verify bucket name is correct
- Check disk space: `df -h`

## Security Considerations

### Network Security

```bash
# Configure firewall
sudo apt install -y ufw

# Allow SSH
sudo ufw allow 22

# Allow OpenPLC web interface (local network only)
sudo ufw allow from 192.168.1.0/24 to any port 8080

# Allow Grafana (local network only)
sudo ufw allow from 192.168.1.0/24 to any port 3000

# Enable firewall
sudo ufw enable
```

### Access Control

1. Change all default passwords
2. Use strong passwords (>12 characters)
3. Enable 2FA on Grafana if exposed externally
4. Restrict SSH access to key-based authentication
5. Regular security updates

## Performance Optimization

### Raspberry Pi Configuration

```bash
# Increase swap for compilation
sudo dphys-swapfile swapoff
sudo nano /etc/dphys-swapfile
# Set: CONF_SWAPSIZE=2048
sudo dphys-swapfile setup
sudo dphys-swapfile swapon

# Disable unnecessary services
sudo systemctl disable bluetooth
sudo systemctl disable wifi
sudo systemctl disable avahi-daemon
```

### InfluxDB Optimization

```bash
# Edit InfluxDB config
sudo nano /etc/influxdb/influxdb.conf

# Increase cache size
[data]
  cache-max-memory-size = "256m"
  cache-snapshot-memory-size = "64m"

# Restart
sudo systemctl restart influxdb
```

## Conclusion

You now have a fully operational Physics-Based Comfort Control System with:

✅ **Hardware**: Raspberry Pi PLC with 1-Wire sensors  
✅ **Control**: Physics-based heating control with learning  
✅ **Monitoring**: Real-time data logging to InfluxDB  
✅ **Visualization**: Grafana dashboards  
✅ **Reliability**: Automated backups and alerts  
✅ **Security**: Firewall and access controls  

The system is production-ready and will continuously improve performance through adaptive learning.

---

**For support**: Refer to main [README.md](./README.md) and individual component documentation.
