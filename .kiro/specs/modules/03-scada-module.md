# DigiSwiss - SCADA Module Detailed Specification

## 1. Module Overview

The Supervisory Control and Data Acquisition (SCADA) module provides real-time monitoring, control, and historical analysis of manufacturing equipment. Purpose-built for precision engineering environments, it connects to CNC machines, PLCs, sensors, and auxiliary equipment to deliver a unified operational view of the entire shop floor.

---

## 2. Communication Protocols

### 2.1 Supported Protocols

| Protocol | Use Case | Machines/Devices | Data Rate |
|----------|----------|-----------------|-----------|
| OPC-UA | CNC machines, modern PLCs | Fanuc, Siemens, Haas, DMG Mori | 100-1000ms polling |
| MTConnect | CNC machine monitoring (read-only) | Haas, Mazak, Okuma, Doosan | 100ms-1s streaming |
| Modbus TCP | PLCs, power meters, sensors | Allen-Bradley, Schneider, ABB | 100-500ms polling |
| Modbus RTU | Legacy PLCs, serial devices | Older equipment over RS-485 | 500ms-1s polling |
| MQTT | Lightweight IoT sensors | Custom sensors, edge gateways | Event-driven |
| REST API | Modern equipment with HTTP interface | Newer CNC controllers, robots | 1-5s polling |
| Focas/Focas2 | Fanuc-specific deep integration | Fanuc CNC controllers | 100ms polling |
| EtherNet/IP | Rockwell/Allen-Bradley PLCs | ControlLogix, CompactLogix | 100ms polling |

### 2.2 Protocol Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                      SCADA SERVICE LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Protocol Abstraction Layer                    │   │
│  │     (Unified data model regardless of source protocol)    │   │
│  └───────┬────────┬────────┬────────┬────────┬──────────────┘   │
│          │        │        │        │        │                    │
│  ┌───────┴──┐ ┌──┴─────┐ ┌┴──────┐ ┌┴─────┐ ┌┴──────────┐     │
│  │ OPC-UA   │ │MTConnect│ │Modbus │ │ MQTT │ │  Focas2   │     │
│  │ Adapter  │ │ Adapter │ │Adapter│ │Bridge│ │  Adapter  │     │
│  └───────┬──┘ └──┬─────┘ └┬──────┘ └┬─────┘ └┬──────────┘     │
│          │        │        │         │        │                   │
└──────────┼────────┼────────┼─────────┼────────┼───────────────────┘
           │        │        │         │        │
    ┌──────┴──┐ ┌──┴────┐ ┌┴─────┐ ┌─┴────┐ ┌┴────────┐
    │Siemens  │ │ Haas  │ │ PLC  │ │Temp  │ │ Fanuc   │
    │ 840D    │ │ NGC   │ │Allen-│ │Sensor│ │ 31i     │
    │         │ │       │ │Brady │ │      │ │         │
    └─────────┘ └───────┘ └──────┘ └──────┘ └─────────┘
```

### 2.3 Data Point Configuration

```
DataPoint (Tag)
├── TagId (GUID)
├── TagName (string, e.g., "HAAS01.Spindle.Speed")
├── MachineId (FK → Machine)
├── Protocol (enum: OpcUa, MTConnect, ModbusTcp, ModbusRtu, Mqtt, RestApi, Focas2)
├── Address (string - protocol-specific address)
│   ├── OPC-UA: "ns=2;s=Channel1.Device1.SpindleSpeed"
│   ├── MTConnect: "//*[@id='spindle_speed']"
│   ├── Modbus: "HR:40001" (Holding Register 40001)
│   ├── MQTT: "machines/haas01/spindle/speed"
│   └── Focas2: "SPINDLE/ACTUAL_SPEED"
├── DataType (enum: Float, Int, Bool, String, Enum)
├── Unit (string, e.g., "RPM", "mm", "°C", "%")
├── ScaleFactor (decimal, default 1.0)
├── Offset (decimal, default 0.0)
├── PollingInterval (TimeSpan)
├── DeadbandValue (decimal - minimum change to report)
├── MinValue (decimal, nullable)
├── MaxValue (decimal, nullable)
├── AlarmConfig (FK → AlarmDefinition, nullable)
├── IsEnabled (bool)
└── LastValue (object - cached current value)
```

---

## 3. Machine Monitoring

### 3.1 Machine Data Model

```
Machine
├── MachineId (GUID)
├── MachineName (string, e.g., "Haas VF-4 #01")
├── MachineCode (string, e.g., "HAAS-01")
├── Type (enum: CNC_Lathe, CNC_Mill_3Axis, CNC_Mill_5Axis, CNC_TurnMill, Grinder, EDM_Wire, EDM_Sinker, CMM, Manual)
├── Manufacturer (string)
├── Model (string)
├── SerialNumber (string)
├── YearOfManufacture (int)
├── Controller (string, e.g., "Fanuc 31i-B", "Siemens 840D sl")
├── ConnectionConfig
│   ├── Protocol (enum)
│   ├── IPAddress (string)
│   ├── Port (int)
│   ├── AuthCredentials (encrypted string, nullable)
│   └── ConnectionTimeout (TimeSpan)
├── Status (enum: Running, Idle, Setup, Fault, Maintenance, Offline)
├── CurrentState
│   ├── SpindleSpeed (decimal, RPM)
│   ├── FeedRate (decimal, mm/min)
│   ├── SpindleLoad (decimal, %)
│   ├── AxisPositions (X, Y, Z, A, B, C)
│   ├── ActiveProgram (string)
│   ├── ProgramProgress (decimal, %)
│   ├── CycleTime (TimeSpan)
│   ├── PartsCount (int)
│   ├── ToolInUse (int)
│   ├── CoolantLevel (decimal, %)
│   └── ActiveAlarms (string[])
├── Location
│   ├── Building (string)
│   ├── Bay (string)
│   └── Position (string)
└── MaintenanceSchedule (FK → MaintenanceSchedule)
```

### 3.2 Machine States & Transitions
```
                    ┌──────────┐
          ┌────────│ OFFLINE  │────────┐
          │        └──────────┘        │
          │ (power on)        (power off)
          ▼                            │
    ┌──────────┐                       │
    │   IDLE   │◀──────────────────────┤
    │          │◀─────────┐            │
    └──┬───┬───┘          │            │
       │   │              │            │
(start │   │(setup)  (complete)        │
 prog) │   │              │            │
       │   ▼              │            │
       │ ┌──────────┐    │            │
       │ │  SETUP   │────┘            │
       │ └──┬───────┘                 │
       │    │(start)                  │
       ▼    ▼                         │
    ┌──────────┐                      │
    │ RUNNING  │──────────────────────┤
    │          │                      │
    └──┬───┬───┘                      │
       │   │                          │
(alarm)│   │(pause)                   │
       │   │                          │
       ▼   ▼                          │
┌──────────┐  ┌──────────────┐        │
│  FAULT   │  │ MAINTENANCE  │────────┘
│  (alarm) │  │  (planned)   │
└──────────┘  └──────────────┘
```

---

## 4. Alarm Management

### 4.1 Alarm Hierarchy

| Level | Severity | Response | Example |
|-------|----------|----------|---------|
| 1 - Critical | Emergency | Immediate stop, evacuate if needed | Fire detection, safety guard breach |
| 2 - High | Urgent | Operator must respond within 5 min | Machine fault, spindle overload, collision |
| 3 - Medium | Warning | Respond within shift | Tool life warning, coolant low, temp high |
| 4 - Low | Advisory | Action at next maintenance | Filter change due, minor vibration increase |
| 5 - Info | Informational | No action required | Cycle complete, program loaded, door opened |

### 4.2 Alarm Configuration

```
AlarmDefinition
├── AlarmId (GUID)
├── AlarmCode (string, e.g., "ALM-HAAS01-SPINDLE-OVL")
├── Name (string, e.g., "Spindle Overload")
├── Description (string)
├── TagId (FK → DataPoint)
├── Severity (enum: Critical, High, Medium, Low, Info)
├── TriggerCondition
│   ├── Type (enum: HighLimit, LowLimit, Deviation, RateOfChange, StateChange, Boolean)
│   ├── Threshold (decimal)
│   ├── Deadband (decimal)
│   ├── TimeDelay (TimeSpan - must exceed for duration to trigger)
│   └── ComparisonTag (FK → DataPoint, nullable - for deviation alarms)
├── Actions[]
│   ├── NotifyRoles (string[] - e.g., ["Operator", "Supervisor", "Maintenance"])
│   ├── NotifyChannels (enum[]: Push, SMS, Email, SignalR, Siren)
│   ├── AutoAcknowledge (bool)
│   ├── AutoAcknowledgeAfter (TimeSpan, nullable)
│   └── EscalationRules[]
│       ├── EscalateAfter (TimeSpan)
│       ├── EscalateTo (string[] - roles)
│       └── EscalateChannel (enum)
├── IsEnabled (bool)
├── SuppressOnMaintenance (bool)
└── RequiresAcknowledgement (bool)

AlarmEvent
├── EventId (GUID)
├── AlarmId (FK → AlarmDefinition)
├── MachineId (FK → Machine)
├── TriggeredAt (datetime)
├── Value (decimal - the value that triggered)
├── AcknowledgedAt (datetime, nullable)
├── AcknowledgedBy (FK → User, nullable)
├── ClearedAt (datetime, nullable)
├── Duration (TimeSpan, computed)
├── Notes (string, nullable)
└── Status (enum: Active, Acknowledged, Cleared)
```

### 4.3 Alarm Workflow
```
┌───────────┐     ┌───────────┐     ┌───────────────┐     ┌──────────┐
│ Condition │────▶│   Alarm   │────▶│  Notification │────▶│ Operator │
│  Detected │     │  Raised   │     │  Dispatched   │     │ Responds │
└───────────┘     └───────────┘     └───────────────┘     └────┬─────┘
                                                                │
                                          ┌─────────────────────┤
                                          │                     │
                                          ▼                     ▼
                                   ┌────────────┐       ┌────────────┐
                                   │Acknowledge │       │ No Response│
                                   │  & Resolve │       │ (Escalate) │
                                   └──────┬─────┘       └──────┬─────┘
                                          │                    │
                                          ▼                    ▼
                                   ┌────────────┐       ┌────────────┐
                                   │   Alarm    │       │ Supervisor │
                                   │  Cleared   │       │  Notified  │
                                   └────────────┘       └────────────┘
```

---

## 5. Dashboard Specifications

### 5.1 Executive Overview Dashboard

**Purpose:** High-level plant performance for management  
**Refresh Rate:** 5 seconds  
**Access:** Management, Production Manager

```
┌─────────────────────────────────────────────────────────────────────┐
│  DIGISWISS PLANT OVERVIEW                          Live │ 14:32:05  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │      OEE        │  │  AVAILABILITY   │  │  PERFORMANCE    │    │
│  │                 │  │                 │  │                 │    │
│  │     76.3%       │  │     89.2%       │  │     91.5%       │    │
│  │   ▲ 2.1% WoW   │  │   ▲ 1.3% WoW   │  │   ▼ 0.5% WoW   │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │    QUALITY      │  │  ACTIVE ALARMS  │  │   THROUGHPUT    │    │
│  │                 │  │                 │  │                 │    │
│  │     99.2%       │  │       3         │  │   47 parts/hr   │    │
│  │   ▲ 0.1% WoW   │  │  (1 critical)   │  │   ▲ 5% vs plan │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
│  MACHINE STATUS MAP                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  ████ Running (7)  ░░░░ Idle (2)  ▓▓▓▓ Setup (1)          │   │
│  │  ╳╳╳╳ Fault (1)    ---- Offline (1)                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  PRODUCTION vs TARGET (Today)                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  █████████████████████████░░░░░░░░░  72% (43/60 jobs)      │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Machine Detail Dashboard

**Purpose:** Deep-dive into individual machine performance  
**Refresh Rate:** 1 second  
**Access:** Operators, Supervisors, Maintenance

```
┌─────────────────────────────────────────────────────────────────────┐
│  HAAS VF-4 #01  │  Status: RUNNING  │  Program: O1234  │  87%     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LIVE PARAMETERS                     AXIS POSITIONS                 │
│  ┌───────────────────────┐          ┌─────────────────────────┐   │
│  │ Spindle: 8,500 RPM    │          │ X: -125.432 mm          │   │
│  │ Feed:    2,400 mm/min │          │ Y:   45.678 mm          │   │
│  │ Load:    42%   ████░░ │          │ Z: -234.567 mm          │   │
│  │ Coolant: 78%   █████░ │          │ A:    0.000°            │   │
│  │ Tool:    T12 (D12mm)  │          │ B:   45.000°            │   │
│  └───────────────────────┘          └─────────────────────────┘   │
│                                                                     │
│  SPINDLE LOAD TREND (Last 4 hours)                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 100%│                                                       │   │
│  │     │         ╱╲    ╱╲                                     │   │
│  │  50%│───╱╲──╱──╲──╱──╲──╱╲───── ← Alarm threshold        │   │
│  │     │ ╱    ╲╱    ╲╱    ╲╱  ╲╱╲                            │   │
│  │   0%│___________________________________________            │   │
│  │     10:00   11:00   12:00   13:00   14:00                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  TODAY'S SUMMARY          │  ACTIVE ALARMS                         │
│  Parts: 32 good / 1 scrap │  ⚠ Tool life 85% (T12) - Medium      │
│  Uptime: 87%              │  ℹ Coolant top-up recommended - Low    │
│  Cycles: 33               │                                        │
│  Avg Cycle: 12m 34s       │                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 OEE Analytics Dashboard

**Purpose:** Detailed OEE breakdown and loss analysis  
**Refresh Rate:** 1 minute  
**Access:** Production Manager, CI Team

| Widget | Content |
|--------|---------|
| OEE Waterfall | Breakdown: Availability × Performance × Quality |
| Loss Pareto | Top 10 downtime reasons ranked by duration |
| OEE by Machine | Comparative bar chart across all machines |
| OEE Trend | 30-day rolling OEE with target line |
| Shift Comparison | OEE by shift (Day vs Night) |
| Planned vs Unplanned | Downtime categorisation pie chart |

---

## 6. Historical Data & Trending

### 6.1 Data Storage Strategy

| Data Type | Storage | Retention | Resolution |
|-----------|---------|-----------|------------|
| Real-time values | In-memory (Redis) | 24 hours | Raw (100ms-1s) |
| Short-term history | Time-series DB (InfluxDB/TimescaleDB) | 90 days | 1-second aggregates |
| Medium-term history | Time-series DB | 2 years | 1-minute aggregates |
| Long-term archive | Blob storage (compressed) | 7 years | 1-hour aggregates |
| Alarm history | SQL Server | 7 years | Full detail |
| Event log | SQL Server | 7 years | Full detail |

### 6.2 Data Aggregation

```
RawDataPoint (in-memory / short-term)
├── TagId (FK → DataPoint)
├── Timestamp (datetime, UTC)
├── Value (double)
├── Quality (enum: Good, Bad, Uncertain)
└── Source (string)

AggregatedDataPoint (historical)
├── TagId (FK → DataPoint)
├── PeriodStart (datetime)
├── PeriodEnd (datetime)
├── Resolution (enum: OneSecond, OneMinute, OneHour, OneDay)
├── MinValue (double)
├── MaxValue (double)
├── AvgValue (double)
├── SumValue (double)
├── Count (int)
├── FirstValue (double)
├── LastValue (double)
└── Quality (enum: Good, Partial, Bad)
```

### 6.3 Trend Viewer Capabilities

| Feature | Description |
|---------|-------------|
| Multi-tag overlay | Plot multiple tags on same chart (dual Y-axis) |
| Time range selection | Predefined (1h, 8h, 24h, 7d, 30d) and custom range |
| Zoom & pan | Mouse-driven zoom into specific time periods |
| Cursor readout | Hover to see exact values at any point in time |
| Annotations | Mark events (tool changes, alarms, shift changes) on timeline |
| Export | CSV/Excel export of historical data |
| Comparison | Overlay same period from different days/weeks |
| Threshold lines | Display alarm limits and target values |

---

## 7. OEE Calculations

### 7.1 Formulas

```
OEE = Availability × Performance × Quality

Availability = Run Time / Planned Production Time
  Where:
    Planned Production Time = Shift Duration - Planned Stops (breaks, meetings)
    Run Time = Planned Production Time - Unplanned Stops (breakdowns, changeovers)

Performance = (Ideal Cycle Time × Total Pieces) / Run Time
  Where:
    Ideal Cycle Time = Theoretical best cycle time for the part
    Total Pieces = Good + Scrap

Quality = Good Pieces / Total Pieces
```

### 7.2 Downtime Categories

| Category | Type | Examples |
|----------|------|----------|
| Planned Maintenance | Planned | Scheduled PM, calibration |
| Changeover/Setup | Semi-planned | Job change, fixture swap |
| Breakdown | Unplanned | Machine fault, component failure |
| Minor Stops | Unplanned | Jam, misalignment, sensor trip |
| Material Wait | Unplanned | No material available |
| Operator Wait | Unplanned | No operator assigned |
| Quality Issue | Unplanned | Inspection hold, rework |
| Tool Change | Semi-planned | Planned/unplanned tool replacement |
| No Order | Planned | No work scheduled |

### 7.3 OEE Data Model

```
OEERecord
├── RecordId (GUID)
├── MachineId (FK → Machine)
├── ShiftDate (date)
├── ShiftType (enum: Day, Night, Weekend)
├── PlannedProductionMinutes (int)
├── ActualRunMinutes (int)
├── DowntimeMinutes (int)
├── IdealCycleSeconds (decimal)
├── TotalPieces (int)
├── GoodPieces (int)
├── ScrapPieces (int)
├── Availability (decimal, computed)
├── Performance (decimal, computed)
├── Quality (decimal, computed)
├── OEE (decimal, computed)
├── DowntimeEvents[]
│   ├── EventId (GUID)
│   ├── Category (enum - from table above)
│   ├── StartTime (datetime)
│   ├── EndTime (datetime)
│   ├── DurationMinutes (decimal)
│   ├── Reason (string)
│   └── Notes (string)
└── CalculatedAt (datetime)
```

---

## 8. Remote Monitoring & Mobile Access

### 8.1 Mobile SCADA Features

| Feature | Description |
|---------|-------------|
| Machine Status | Colour-coded status of all machines (grid/list view) |
| Push Notifications | Configurable alerts for alarms, completion, downtime |
| Live Values | View key parameters for selected machine |
| Alarm Acknowledge | Acknowledge and add notes to alarms remotely |
| Trend Viewer | Simplified trend display for mobile screens |
| Shift Summary | End-of-shift performance summary |

### 8.2 Notification Rules

```
NotificationRule
├── RuleId (GUID)
├── Name (string)
├── Trigger (enum: AlarmRaised, MachineStateChange, OEEThreshold, CycleComplete, ShiftEnd)
├── Conditions
│   ├── MachinIds (GUID[] - which machines)
│   ├── Severity (enum[] - which severities)
│   ├── TimeWindow (TimeSpan - only during certain hours)
│   └── Cooldown (TimeSpan - minimum time between notifications)
├── Recipients[]
│   ├── UserId (FK → User)
│   ├── Channel (enum: Push, SMS, Email)
│   └── Priority (enum: Always, WorkingHoursOnly, UrgentOnly)
└── IsEnabled (bool)
```

---

## 9. Business Rules

### Machine Monitoring Rules
1. Machine status updates must be within 5 seconds of actual state change
2. Communication failure triggers "OFFLINE" status after 30 seconds of no response
3. Three consecutive communication failures raise a connectivity alarm
4. Machine data is never deleted - only archived after retention period

### Alarm Rules
1. Critical alarms cannot be auto-acknowledged
2. Unacknowledged alarms escalate per escalation rules
3. Alarm flooding protection: suppress repeated alarms within deadband period
4. Maintenance mode suppresses non-critical alarms but logs them
5. Alarm count per shift contributes to machine health score

### OEE Rules
1. OEE calculations run at end of each shift automatically
2. Real-time OEE estimates update every 5 minutes during shift
3. OEE < 60% for any machine triggers production manager alert
4. Downtime events > 15 minutes must have a reason code assigned
5. OEE targets are configurable per machine and product family

---

## 10. Integration Points

| System/Module | Direction | Method | Data |
|---------------|-----------|--------|------|
| MES | Bi-directional | SignalR + Event Bus | Machine state → job status, job assignment → machine context |
| IoT | Inbound | MQTT/Event Stream | Sensor data supplementing machine data |
| ERP (Planning) | Inbound | Event Bus | Scheduled maintenance windows, production targets |
| AI Module | Outbound | API/Event Stream | Historical data for predictive analytics |
| Maintenance (CMMS) | Outbound | Event Bus | Alarm events, downtime logs, maintenance triggers |
| Historian | Outbound | Time-series write | All tag values for long-term storage |

---

## 11. Security Considerations

| Concern | Mitigation |
|---------|------------|
| Network segmentation | SCADA on isolated VLAN, firewall between IT and OT networks |
| Protocol security | OPC-UA with certificates, MQTT with TLS, no plain Modbus externally |
| Access control | Read-only for most users, write (control) only for authorised roles |
| Audit trail | All control actions logged with user, timestamp, before/after values |
| Failsafe | Loss of SCADA connection does not stop machines (monitoring only) |
| Data integrity | Checksums on time-series data, tamper detection on alarm logs |

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](../product-specification.md)*
