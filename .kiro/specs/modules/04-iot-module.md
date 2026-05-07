# DigiSwiss - IoT Module Detailed Specification

## 1. Module Overview

The Internet of Things (IoT) module manages the full lifecycle of connected devices across the manufacturing environment — from provisioning and configuration through data collection, edge processing, and cloud analytics. It extends beyond machine-level SCADA data to capture environmental, energy, asset-tracking, and auxiliary sensor data that drives predictive maintenance, process optimisation, and regulatory compliance.

---

## 2. IoT Architecture

### 2.1 Three-Tier Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLOUD TIER                                    │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │  IoT Hub   │  │  Stream    │  │  Storage   │  │  Analytics │   │
│  │ (Azure IoT │  │ Processing │  │ (Hot/Warm/ │  │  (ML/AI    │   │
│  │  Hub)      │  │ (ASA/Func) │  │  Cold)     │  │  Models)   │   │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘   │
│         │               │               │               │           │
│         └───────────────┴───────────────┴───────────────┘           │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │ (MQTT/AMQP/HTTPS)
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                         EDGE TIER│                                    │
│  ┌──────────────────────────────┴──────────────────────────────┐   │
│  │              Edge Gateway (Azure IoT Edge / Custom)          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │   │
│  │  │ Protocol │  │  Local   │  │  Filter  │  │  Store &  │  │   │
│  │  │ Adapters │  │Analytics │  │ & Enrich │  │  Forward  │  │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │   │
│  └──────────────────────────────┬──────────────────────────────┘   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │ (Modbus/BLE/Zigbee/LoRa/Ethernet)
                                  │
┌─────────────────────────────────┼───────────────────────────────────┐
│                       DEVICE TIER│                                    │
│  ┌─────────┐  ┌─────────┐  ┌───┴─────┐  ┌─────────┐  ┌────────┐ │
│  │ Temp &  │  │Vibration│  │ Power   │  │ Air     │  │ QR/NFC │ │
│  │Humidity │  │ Sensors │  │ Meters  │  │ Quality │  │Beacons │ │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Communication Flow

```
Device → Edge Gateway → IoT Hub → Stream Analytics → Storage/Actions

Detailed:
1. Device samples data at configured interval
2. Edge gateway receives via local protocol (BLE, Modbus, etc.)
3. Edge applies filtering rules (deadband, aggregation, anomaly check)
4. Edge forwards to cloud via MQTT/AMQP (with store-and-forward if offline)
5. IoT Hub authenticates, routes message to appropriate consumer
6. Stream Analytics processes in real-time (alerts, enrichment)
7. Data lands in appropriate storage tier (hot/warm/cold)
8. Dashboards and services consume from storage/real-time stream
```

---

## 3. Device Management

### 3.1 Device Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Register │───▶│Provision │───▶│  Active  │───▶│  Retire  │
│ (create  │    │(configure│    │(sending  │    │(decomm-  │
│  identity)│    │ & deploy)│    │  data)   │    │ ission)  │
└──────────┘    └──────────┘    └────┬─────┘    └──────────┘
                                     │
                           ┌─────────┼─────────┐
                           │         │         │
                           ▼         ▼         ▼
                    ┌──────────┐ ┌────────┐ ┌──────────┐
                    │Firmware  │ │Disabled│ │Maintenance│
                    │Update    │ │(temp   │ │(offline   │
                    │(OTA)     │ │ off)   │ │ for cal)  │
                    └──────────┘ └────────┘ └──────────┘
```

### 3.2 Device Data Model

```
IoTDevice
├── DeviceId (GUID)
├── DeviceCode (string, e.g., "TEMP-BAY3-001")
├── DisplayName (string, e.g., "Bay 3 Temperature Sensor")
├── DeviceType (enum: TemperatureSensor, HumiditySensor, VibrationSensor, PowerMeter, AirQualitySensor, FlowMeter, PressureSensor, ProximitySensor, Beacon, Gateway)
├── Manufacturer (string)
├── Model (string)
├── SerialNumber (string)
├── FirmwareVersion (string)
├── Status (enum: Registered, Provisioning, Active, Disabled, Maintenance, Firmware_Update, Retired)
├── ConnectionState (enum: Connected, Disconnected, Unknown)
├── LastActivityTime (datetime)
├── Location
│   ├── Building (string)
│   ├── Zone (string, e.g., "CNC Bay", "Grinding", "Stores")
│   ├── MachineId (FK → Machine, nullable - if attached to machine)
│   ├── Position (string, e.g., "Spindle bearing", "Coolant tank")
│   ├── Latitude (decimal, nullable)
│   └── Longitude (decimal, nullable)
├── Configuration
│   ├── ReportingInterval (TimeSpan, e.g., 30 seconds)
│   ├── SamplingRate (TimeSpan, e.g., 1 second)
│   ├── Thresholds (JSON - device-specific alert thresholds)
│   ├── CalibrationOffset (decimal)
│   ├── CalibrationDate (datetime)
│   └── CalibrationDueDate (datetime)
├── Connectivity
│   ├── Protocol (enum: MQTT, BLE, Zigbee, LoRaWAN, WiFi, Ethernet, Modbus)
│   ├── GatewayId (FK → IoTDevice[Gateway], nullable)
│   ├── NetworkAddress (string)
│   ├── AuthMethod (enum: SymmetricKey, X509Certificate, SAS)
│   └── EncryptionEnabled (bool)
├── Metadata (JSON - flexible key/value)
├── Tags (string[])
├── BatteryLevel (decimal, nullable - for wireless devices)
├── SignalStrength (int, dBm, nullable)
└── ProvisionedAt (datetime)

DeviceTelemetry (message schema)
├── DeviceId (string)
├── Timestamp (datetime, UTC)
├── MessageId (GUID)
├── Readings[]
│   ├── MetricName (string, e.g., "temperature", "vibration_rms")
│   ├── Value (double)
│   ├── Unit (string)
│   └── Quality (enum: Good, Suspect, Bad)
└── DeviceMetadata
    ├── BatteryLevel (decimal, nullable)
    ├── SignalStrength (int, nullable)
    └── FirmwareVersion (string)
```

### 3.3 Device Provisioning Flow

| Step | Action | System |
|------|--------|--------|
| 1 | Register device in portal (type, location, config) | DigiSwiss Admin |
| 2 | Generate device credentials (key/cert) | IoT Hub DPS |
| 3 | Flash firmware with credentials + config | Provisioning Tool |
| 4 | Device powers on, connects to gateway/hub | Device |
| 5 | Device reports initial telemetry | Device → Hub |
| 6 | System confirms connectivity, marks Active | DigiSwiss |
| 7 | Monitoring begins, alerts configured | Automated |

---

## 4. Edge Computing

### 4.1 Edge Gateway Responsibilities

| Function | Description | Benefit |
|----------|-------------|---------|
| Protocol Translation | Convert BLE/Zigbee/Modbus to MQTT | Unified cloud communication |
| Local Filtering | Apply deadband, suppress noise | Reduce bandwidth 60-80% |
| Aggregation | Compute min/max/avg over windows | Meaningful data reduction |
| Store & Forward | Buffer data during connectivity loss | Zero data loss guarantee |
| Local Alerting | Trigger immediate alerts without cloud | Sub-100ms response |
| Edge Analytics | Run lightweight ML models locally | Real-time anomaly detection |
| Device Management | Manage connected devices, health checks | Simplified maintenance |
| Security | TLS termination, device authentication | Defence in depth |

### 4.2 Edge Processing Rules

```
EdgeRule
├── RuleId (GUID)
├── Name (string, e.g., "Vibration Spike Detection")
├── GatewayId (FK → IoTDevice[Gateway])
├── InputDevices (GUID[] - which devices this rule applies to)
├── Condition
│   ├── Type (enum: Threshold, RateOfChange, Pattern, Aggregation, Absence)
│   ├── MetricName (string)
│   ├── Operator (enum: GreaterThan, LessThan, Equals, Between, Outside)
│   ├── Value (double)
│   ├── SecondaryValue (double, nullable - for Between/Outside)
│   ├── WindowDuration (TimeSpan - evaluation window)
│   └── MinOccurrences (int - how many times in window)
├── Actions[]
│   ├── Type (enum: ForwardToCloud, LocalAlert, SuppressData, EnrichData, TriggerDevice)
│   ├── Priority (enum: Immediate, Normal, Batch)
│   └── Parameters (JSON)
├── IsEnabled (bool)
└── ExecutionCount (long - for monitoring)
```

### 4.3 Edge Gateway Hardware Specifications

| Tier | Use Case | Example Hardware | Capacity |
|------|----------|-----------------|----------|
| Light | Single machine, few sensors | Raspberry Pi 4 / Intel NUC | 10-20 devices |
| Medium | Machine bay, mixed protocols | Dell Edge Gateway 3000 | 50-100 devices |
| Heavy | Full factory floor, ML models | Azure Stack Edge Mini R | 200+ devices |

### 4.4 Offline Resilience

```
Normal Operation:
  Device → Edge → Cloud (real-time)

Connectivity Loss:
  Device → Edge → Local Buffer (SQLite/LevelDB)
                → Local Alerting continues
                → Dashboard shows "Cloud Disconnected"

Reconnection:
  Edge → Replay buffered data → Cloud (chronological order)
       → Reconcile any missed alerts
       → Resume normal operation

Buffer Capacity: Minimum 72 hours of data at normal reporting rates
```

---

## 5. Data Pipeline Architecture

### 5.1 Ingestion Pipeline

```
┌─────────┐    ┌─────────┐    ┌──────────────┐    ┌─────────────┐
│ Devices │───▶│ IoT Hub │───▶│   Message    │───▶│  Consumer   │
│         │    │         │    │   Router     │    │  Groups     │
└─────────┘    └─────────┘    └──────┬───────┘    └──────┬──────┘
                                     │                    │
                    ┌────────────────┼────────────────────┤
                    │                │                    │
                    ▼                ▼                    ▼
             ┌───────────┐   ┌───────────┐       ┌───────────┐
             │  Stream   │   │  Alert    │       │   Cold    │
             │ Analytics │   │  Engine   │       │  Storage  │
             │(real-time)│   │           │       │  (Blob)   │
             └─────┬─────┘   └─────┬─────┘       └───────────┘
                   │               │
                   ▼               ▼
            ┌───────────┐   ┌───────────┐
            │ Time-Series│   │ SignalR   │
            │    DB     │   │ (Push to  │
            │           │   │  clients) │
            └───────────┘   └───────────┘
```

### 5.2 Message Routing Rules

| Route | Condition | Destination | Latency Target |
|-------|-----------|-------------|----------------|
| Hot Path | All real-time telemetry | Stream Analytics → SignalR | < 2 seconds |
| Alert Path | Threshold breaches | Alert Engine → Notifications | < 5 seconds |
| Warm Path | Aggregated summaries | Time-Series DB | < 30 seconds |
| Cold Path | Raw archive | Blob Storage (Parquet) | < 5 minutes |
| Analytics Path | ML model input | ML Pipeline (batch) | Hourly |
| Device Events | Status changes, errors | Event Hub → SQL | < 10 seconds |

### 5.3 Data Volume Estimates

| Scenario | Devices | Msg/Device/Min | Total Msg/Min | Daily Volume |
|----------|---------|----------------|---------------|-------------|
| Pilot (1 bay) | 20 | 2 | 40 | ~58K msgs / ~50MB |
| Phase 1 (floor) | 100 | 2 | 200 | ~288K msgs / ~250MB |
| Full Scale | 500 | 2 | 1,000 | ~1.44M msgs / ~1.2GB |
| Future (dense) | 2,000 | 4 | 8,000 | ~11.5M msgs / ~10GB |

---

## 6. Sensor Types & Use Cases

### 6.1 Environmental Monitoring

| Sensor | Metric | Range | Accuracy | Use Case |
|--------|--------|-------|----------|----------|
| Temperature | °C | -20 to 80 | ±0.5°C | Machine shop ambient, coolant temp |
| Humidity | %RH | 0-100% | ±2% | Corrosion prevention, CMM room |
| Air Pressure | mbar | 800-1200 | ±1 mbar | Cleanroom differential pressure |
| Particulate | μg/m³ | 0-500 | ±10% | Air quality, grinding dust |
| Noise Level | dB(A) | 30-130 | ±1.5dB | H&S compliance, machine health |

### 6.2 Machine Health Monitoring

| Sensor | Metric | Range | Accuracy | Use Case |
|--------|--------|-------|----------|----------|
| Vibration (accelerometer) | mm/s RMS, g | 0-50g | ±2% | Bearing wear, spindle health |
| Current Clamp | Amps | 0-100A | ±1% | Motor load, power anomalies |
| Temperature (contact) | °C | -40 to 300 | ±1°C | Bearing temp, hydraulic oil |
| Flow Meter | L/min | 0-50 | ±3% | Coolant flow, hydraulic pressure |
| Pressure Transducer | bar | 0-400 | ±0.5% | Hydraulic/pneumatic systems |
| Oil Quality | TAN, viscosity | - | - | Lubricant degradation |

### 6.3 Energy Monitoring

| Sensor | Metric | Range | Accuracy | Use Case |
|--------|--------|-------|----------|----------|
| Power Meter (3-phase) | kW, kWh, PF | 0-500kW | ±0.5% | Per-machine energy consumption |
| CT Clamp | kW | 0-100kW | ±2% | Sub-metering by bay/circuit |
| Gas Meter | m³/hr | 0-100 | ±1% | Compressed air consumption |
| Water Meter | L/hr | 0-1000 | ±2% | Coolant water usage |

### 6.4 Asset Tracking

| Technology | Range | Accuracy | Battery Life | Use Case |
|-----------|-------|----------|--------------|----------|
| BLE Beacons | 30m | ±2m | 2-5 years | Tool/fixture location |
| UWB Tags | 50m | ±10cm | 6-12 months | High-precision tracking |
| NFC/QR | Contact | N/A | Passive | Asset identification, check-in |
| LoRa GPS | Outdoor | ±5m | 5+ years | Vehicle/delivery tracking |

---

## 7. Predictive Maintenance

### 7.1 ML Model Pipeline

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│Historical│───▶│ Feature  │───▶│  Model   │───▶│  Deploy  │───▶│  Predict │
│  Data    │    │Engineering│    │ Training │    │ (Edge/   │    │ & Alert  │
│ (6+ mo)  │    │           │    │ (ML.NET) │    │  Cloud)  │    │          │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

### 7.2 Predictive Models

| Model | Input Data | Output | Accuracy Target |
|-------|-----------|--------|-----------------|
| Bearing Failure | Vibration RMS, temperature, frequency spectrum | Days to failure | ±7 days |
| Spindle Health | Current draw, vibration, temperature | Health score 0-100 | >85% precision |
| Tool Wear | Cutting force, vibration, cycle count | Remaining life % | ±10% |
| Coolant Degradation | pH, conductivity, temperature, time | Days to change | ±3 days |
| Air Compressor | Pressure, temperature, duty cycle, vibration | Maintenance window | ±5 days |

### 7.3 Predictive Maintenance Data Model

```
PredictiveModel
├── ModelId (GUID)
├── Name (string)
├── Version (string)
├── TargetAssetType (enum: Spindle, Bearing, Tool, Coolant, Compressor)
├── InputFeatures (string[] - required sensor metrics)
├── TrainingDataRange (DateRange)
├── Accuracy (decimal)
├── DeploymentLocation (enum: Cloud, Edge, Both)
├── LastTrainedAt (datetime)
├── IsActive (bool)
└── Predictions[]
    ├── PredictionId (GUID)
    ├── MachineId (FK → Machine)
    ├── DeviceId (FK → IoTDevice)
    ├── PredictedEvent (string, e.g., "Bearing failure")
    ├── Confidence (decimal, 0-1)
    ├── PredictedDate (datetime)
    ├── CurrentHealthScore (decimal, 0-100)
    ├── RecommendedAction (string)
    ├── GeneratedAt (datetime)
    └── Outcome (enum: Pending, Confirmed, FalsePositive, Missed, nullable)
```

---

## 8. Device Fleet Management

### 8.1 Fleet Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│  IoT DEVICE FLEET                                     245 devices   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  STATUS OVERVIEW                                                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐     │
│  │  ONLINE    │ │  OFFLINE   │ │  WARNING   │ │   ERROR    │     │
│  │    231     │ │     8      │ │     4      │ │     2      │     │
│  │   (94%)    │ │   (3%)     │ │   (2%)     │ │   (1%)     │     │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘     │
│                                                                     │
│  BY TYPE                          ALERTS                           │
│  Temperature:     45              ⚠ TEMP-BAY2-003: Battery 12%    │
│  Vibration:       30              ⚠ VIB-HAAS02: Signal weak       │
│  Power Meters:    12              ✗ FLOW-GRIND1: No data 2hrs     │
│  Air Quality:      8              ✗ PWR-BAY4: Communication fail  │
│  Flow:            15                                               │
│  Other:          135              FIRMWARE                         │
│                                   12 devices need update (v2.3.1)  │
│  HEALTH MAP (Floor Plan)                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  [Interactive floor plan with device locations,              │  │
│  │   colour-coded by status: green/amber/red]                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.2 Firmware Over-the-Air (OTA) Updates

```
FirmwareUpdate
├── UpdateId (GUID)
├── TargetDeviceType (enum)
├── TargetFirmwareVersion (string)
├── PackageUrl (string - blob storage)
├── PackageSize (long, bytes)
├── Checksum (string, SHA-256)
├── ReleaseNotes (string)
├── RolloutStrategy
│   ├── Type (enum: Immediate, Staged, Scheduled)
│   ├── StagePercentages (int[], e.g., [10, 25, 50, 100])
│   ├── ScheduledTime (datetime, nullable)
│   ├── MaxFailureRate (decimal - abort if exceeded)
│   └── RollbackOnFailure (bool)
├── Status (enum: Draft, InProgress, Complete, RolledBack, Failed)
├── TargetDevices (GUID[])
├── Results[]
│   ├── DeviceId (GUID)
│   ├── Status (enum: Pending, Downloading, Installing, Success, Failed, Skipped)
│   ├── StartedAt (datetime)
│   ├── CompletedAt (datetime, nullable)
│   └── ErrorMessage (string, nullable)
└── CreatedAt (datetime)
```

---

## 9. Energy Management

### 9.1 Energy Dashboard Widgets

| Widget | Description |
|--------|-------------|
| Total Consumption | Real-time kW draw across facility |
| Per-Machine Cost | £/hour running cost per machine |
| Energy per Part | kWh consumed per part produced |
| Peak Demand | Track and alert on peak demand charges |
| Carbon Footprint | CO2 equivalent from energy usage |
| Shift Comparison | Energy efficiency by shift |
| Idle Cost | Cost of machines powered but not cutting |
| Compressed Air | Monitor expensive compressed air usage |

### 9.2 Energy Data Model

```
EnergyReading
├── ReadingId (GUID)
├── DeviceId (FK → IoTDevice[PowerMeter])
├── MachineId (FK → Machine, nullable)
├── Timestamp (datetime)
├── ActivePower_kW (decimal)
├── ReactivePower_kVAR (decimal)
├── ApparentPower_kVA (decimal)
├── PowerFactor (decimal)
├── Voltage_V (decimal)
├── Current_A (decimal)
├── Frequency_Hz (decimal)
├── Energy_kWh (decimal, cumulative)
└── CostRate_GBP (decimal - time-of-use rate)

EnergySummary (hourly/daily/monthly)
├── SummaryId (GUID)
├── MachineId (FK → Machine)
├── Period (enum: Hour, Day, Week, Month)
├── PeriodStart (datetime)
├── TotalEnergy_kWh (decimal)
├── PeakDemand_kW (decimal)
├── AverageLoad_kW (decimal)
├── Cost_GBP (decimal)
├── PartsProduced (int)
├── EnergyPerPart_kWh (decimal, computed)
└── CO2_kg (decimal, computed - using grid carbon intensity)
```

---

## 10. Business Rules

### Device Management Rules
1. All devices must be registered before connecting (no auto-discovery in production)
2. Devices with no telemetry for >2× reporting interval trigger connectivity alert
3. Battery level <20% triggers replacement notification
4. Calibration-expired devices flag data as "Suspect" quality
5. Decommissioned devices retain historical data but stop accepting new data

### Data Quality Rules
1. Values outside physical range (e.g., temperature -50°C) are flagged "Bad" quality
2. Stuck values (no change for >10× expected) are flagged "Suspect"
3. Missing data gaps >5 minutes are logged as data quality events
4. Aggregations require >80% good-quality source data to be marked "Good"
5. Duplicate messages (same device + timestamp) are deduplicated at ingestion

### Predictive Maintenance Rules
1. Models require minimum 6 months historical data before deployment
2. Predictions with confidence <60% are suppressed from user-facing alerts
3. All predictions are tracked for accuracy (outcome feedback loop)
4. Models are retrained monthly with latest data
5. False positive rate >20% triggers model review

### Energy Rules
1. Machine idle >30 minutes sends "consider power-down" notification
2. Peak demand approaching contracted maximum triggers load-shedding alert
3. Energy cost anomalies >20% vs baseline trigger investigation alert
4. Compressed air leaks (flow with no production) trigger maintenance ticket
5. Monthly energy reports auto-generated for management review

---

## 11. Integration Points

| System/Module | Direction | Method | Data |
|---------------|-----------|--------|------|
| SCADA | Bi-directional | MQTT/Event Bus | Supplementary machine data, shared tags |
| MES | Outbound | Event Bus | Environmental data for traceability, energy per job |
| ERP (Maintenance) | Outbound | Event Bus | Predictive maintenance work orders |
| AI Module | Outbound | API/Event Stream | Raw data for model training and inference |
| Azure IoT Hub | Bi-directional | MQTT/AMQP | Device telemetry, device twin, commands |
| Time-Series DB | Outbound | Direct write | All processed telemetry |
| Blob Storage | Outbound | SDK | Raw data archive (Parquet/CSV) |
| Power BI | Outbound | DirectQuery | Energy reporting and analytics |

---

## 12. Security & Compliance

| Concern | Implementation |
|---------|---------------|
| Device Authentication | X.509 certificates (production), SAS tokens (dev) |
| Data Encryption | TLS 1.3 in transit, AES-256 at rest |
| Network Isolation | IoT devices on dedicated VLAN, no internet access |
| Firmware Signing | Code-signed firmware packages, verified before install |
| Access Control | Device-level permissions (read telemetry, send commands) |
| Audit Trail | All device provisioning/configuration changes logged |
| Data Sovereignty | All data stored in UK Azure region |
| Vulnerability Scanning | Monthly scan of edge gateways and firmware |

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](../product-specification.md)*
