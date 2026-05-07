# DigiSwiss - MES Module Detailed Specification

## 1. Module Overview

The Manufacturing Execution System (MES) module bridges the gap between production planning (ERP) and the physical shop floor. It provides real-time visibility into job execution, enables paperless operations, enforces quality processes, and captures granular production data for continuous improvement.

Built for precision engineering environments where tolerances are tight, traceability is mandatory, and every operation matters.

---

## 2. Sub-Modules

### 2.1 Work Order Execution

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Digital Work Orders | Paperless job cards with all information at point of use | P0 |
| Operation Sequencing | Enforce correct operation order with dependencies | P0 |
| Start/Stop/Pause | Operator-driven job state transitions with timestamps | P0 |
| Setup Instructions | Machine-specific setup sheets, tool lists, fixture details | P1 |
| Drawing Viewer | Display engineering drawings at machine (zoom, annotate) | P1 |
| NC Program Linking | Associate CNC programs with operations, version controlled | P1 |
| Batch Splitting | Split a job into sub-batches for parallel processing | P2 |
| Rework Routing | Define alternate routes for non-conforming parts | P1 |

#### Workflow: Job Execution Lifecycle
```
┌───────────┐    ┌───────────┐    ┌───────────┐    ┌───────────┐
│  Released │───▶│   Queue   │───▶│   Setup   │───▶│  Running  │
│  (from    │    │ (at work  │    │ (operator │    │ (cutting/ │
│  planning)│    │  centre)  │    │  prepares)│    │  working) │
└───────────┘    └───────────┘    └───────────┘    └─────┬─────┘
                                                         │
                      ┌──────────────────────────────────┼──────────┐
                      │                                  │          │
                      ▼                                  ▼          ▼
               ┌───────────┐                     ┌───────────┐ ┌────────┐
               │   Paused  │                     │ Inspection│ │  Hold  │
               │ (break/   │                     │ (in-proc  │ │ (fault/│
               │  issue)   │                     │  check)   │ │  NCR)  │
               └───────────┘                     └─────┬─────┘ └────────┘
                                                       │
                                                       ▼
                                                ┌───────────┐    ┌───────────┐
                                                │ Complete  │───▶│   Next    │
                                                │(operation)│    │ Operation │
                                                └───────────┘    └───────────┘
```

#### Data Model: Work Orders
```
WorkOrder
├── WorkOrderId (GUID)
├── OrderId (FK → Order)
├── OrderLineId (FK → OrderLine)
├── WorkOrderNumber (string, e.g., "WO-2026-001234-01")
├── PartNumber (string)
├── DrawingRef (string)
├── Revision (string)
├── Quantity (int)
├── QuantityGood (int)
├── QuantityScrap (int)
├── Status (enum: Released, InProgress, Complete, OnHold, Cancelled)
├── Priority (enum: Standard, High, Urgent, Critical)
├── PlannedStartDate (datetime)
├── PlannedEndDate (datetime)
├── ActualStartDate (datetime, nullable)
├── ActualEndDate (datetime, nullable)
├── MaterialBatchId (FK → BatchNumber)
├── Notes (string)
└── Operations[]
    ├── OperationId (GUID)
    ├── SequenceNumber (int, e.g., 10, 20, 30)
    ├── WorkCentreId (FK → WorkCentre)
    ├── MachineId (FK → Machine, nullable)
    ├── OperatorId (FK → User, nullable)
    ├── OperationType (enum: Turning, Milling, Grinding, EDM, Drilling, Assembly, Inspection, HeatTreat, SurfaceFinish, Packing)
    ├── Description (string)
    ├── SetupTimeEstimate (TimeSpan)
    ├── RunTimeEstimate (TimeSpan)
    ├── ActualSetupTime (TimeSpan, nullable)
    ├── ActualRunTime (TimeSpan, nullable)
    ├── Status (enum: Pending, Queue, Setup, Running, Paused, Inspection, Complete, Hold)
    ├── NCProgramId (FK → NCProgram, nullable)
    ├── ToolListId (FK → ToolList, nullable)
    ├── SetupSheetId (FK → Document, nullable)
    ├── InspectionRequired (bool)
    └── TimeEntries[]
        ├── TimeEntryId (GUID)
        ├── OperatorId (FK → User)
        ├── Type (enum: Setup, Run, Idle, Rework)
        ├── StartTime (datetime)
        ├── EndTime (datetime, nullable)
        ├── Duration (TimeSpan, computed)
        └── Notes (string)
```

---

### 2.2 Operator Interface

#### Design Principles
- **Touch-first:** Large buttons, swipe gestures, minimal text input
- **Gloves-compatible:** Oversized touch targets (min 48px)
- **High-contrast:** Readable in bright shop floor lighting
- **Minimal training:** Intuitive icons, colour-coded status
- **Offline-capable:** Queue actions when connectivity drops, sync when restored

#### Screen Specifications

**Operator Dashboard (Home)**
```
┌─────────────────────────────────────────────────────┐
│  [Operator Name]          [Shift: Day]    [Logout]  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │  CURRENT    │  │   MY QUEUE  │  │   ALERTS   │ │
│  │    JOB      │  │   (3 jobs)  │  │    (1)     │ │
│  │             │  │             │  │            │ │
│  │ WO-001234   │  │ Next: WO-.. │  │ Tool wear  │ │
│  │ Op 20 Mill  │  │             │  │ warning    │ │
│  │ [RUNNING]   │  │             │  │            │ │
│  └─────────────┘  └─────────────┘  └────────────┘ │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  [PAUSE]  [COMPLETE]  [QUALITY]  [PROBLEM]  │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Job Detail Screen**
```
┌─────────────────────────────────────────────────────┐
│  WO-2026-001234  │  Op 20: CNC Milling             │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Part: SHAFT-ASM-001  Rev: C  Qty: 50              │
│  Material: 316L SS  Batch: HT-2026-0456            │
│  Machine: Haas VF-4  Program: PRG-001234-R2        │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  TABS: [Drawing] [Setup] [Tools] [Quality]  │  │
│  │                                              │  │
│  │  (Content area - drawing viewer, setup       │  │
│  │   instructions, tool list, or inspection     │  │
│  │   requirements displayed here)               │  │
│  │                                              │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Timer: 02:34:15 (running)   Good: 32  Scrap: 1   │
│                                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │  [PAUSE]  [LOG SCRAP]  [INSPECT]  [DONE]    │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### Operator Interactions
| Action | Input Method | System Response |
|--------|-------------|-----------------|
| Clock On/Off | QR badge scan | Record shift start/end, show dashboard |
| Start Job | Tap "Start" + confirm | Begin timer, update job status, notify planning |
| Pause Job | Tap "Pause" + reason select | Pause timer, log reason, update board |
| Complete Operation | Tap "Done" + enter good/scrap qty | Stop timer, trigger next op or inspection |
| Log Scrap | Tap "Scrap" + qty + reason | Update counts, generate NCR if threshold exceeded |
| Report Problem | Tap "Problem" + category + photo | Create maintenance ticket, alert supervisor |
| First-Off Inspection | Tap "Quality" + enter measurements | Record results, pass/fail decision, release batch |

---

### 2.3 Quality Control & Inspection

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| First Article Inspection (FAI) | Full dimensional inspection on first part of batch | P0 |
| In-Process Inspection | Periodic checks during production run | P0 |
| Final Inspection | Complete inspection before dispatch | P0 |
| SPC (Statistical Process Control) | Control charts, Cp/Cpk calculations | P1 |
| Non-Conformance Reports (NCR) | Raise, investigate, disposition non-conforming parts | P0 |
| Corrective Actions (CAPA) | Track root cause and preventive actions | P1 |
| Inspection Plans | Define what to measure, tolerances, instruments | P0 |
| Certificate of Conformity | Auto-generate CoC from inspection data | P1 |
| Gauge Management | Calibration schedules, R&R studies | P2 |
| Customer Complaints | Track and resolve quality issues from customers | P2 |

#### Workflow: Quality Inspection
```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│  Inspection   │────▶│   Perform     │────▶│   Record      │
│  Triggered    │     │  Measurements │     │   Results     │
│(first-off/    │     │  (per plan)   │     │               │
│ in-proc/final)│     │               │     │               │
└───────────────┘     └───────────────┘     └───────┬───────┘
                                                    │
                                          ┌─────────┴─────────┐
                                          │                   │
                                          ▼                   ▼
                                   ┌────────────┐     ┌────────────┐
                                   │    PASS    │     │    FAIL    │
                                   │            │     │            │
                                   │ Release to │     │ Raise NCR  │
                                   │ next op /  │     │ Hold stock │
                                   │ dispatch   │     │ Notify QM  │
                                   └────────────┘     └──────┬─────┘
                                                             │
                                                             ▼
                                                      ┌────────────┐
                                                      │Disposition │
                                                      │            │
                                                      │• Rework    │
                                                      │• Concession│
                                                      │• Scrap     │
                                                      │• Return    │
                                                      └────────────┘
```

#### Data Model: Quality
```
InspectionPlan
├── PlanId (GUID)
├── PartNumber (string)
├── Revision (string)
├── PlanType (enum: FirstArticle, InProcess, Final, Receiving)
├── Characteristics[]
│   ├── CharacteristicId (GUID)
│   ├── SequenceNumber (int)
│   ├── Description (string, e.g., "OD at datum A")
│   ├── Nominal (decimal)
│   ├── UpperTolerance (decimal)
│   ├── LowerTolerance (decimal)
│   ├── UnitOfMeasure (string)
│   ├── InstrumentType (string, e.g., "Micrometer 0-25mm")
│   ├── SampleSize (int)
│   ├── SampleFrequency (string, e.g., "Every 10th part")
│   ├── IsCritical (bool)
│   └── DrawingBalloon (string, nullable)
└── ApprovedBy (FK → User)

InspectionRecord
├── RecordId (GUID)
├── WorkOrderId (FK → WorkOrder)
├── OperationId (FK → Operation)
├── PlanId (FK → InspectionPlan)
├── InspectorId (FK → User)
├── InspectionDate (datetime)
├── PartSerialNumber (string, nullable)
├── Result (enum: Pass, Fail, ConditionalPass)
├── Measurements[]
│   ├── MeasurementId (GUID)
│   ├── CharacteristicId (FK → Characteristic)
│   ├── ActualValue (decimal)
│   ├── IsInTolerance (bool, computed)
│   ├── Deviation (decimal, computed)
│   └── Notes (string)
└── SignOff (FK → User, nullable)

NonConformanceReport
├── NCRId (GUID)
├── NCRNumber (string, auto-generated, e.g., "NCR-2026-0123")
├── WorkOrderId (FK → WorkOrder)
├── PartNumber (string)
├── Quantity (int)
├── DetectedBy (FK → User)
├── DetectedAt (enum: InProcess, FinalInspection, CustomerReturn, GoodsReceipt)
├── Description (string)
├── RootCause (string, nullable)
├── Disposition (enum: Pending, Rework, Concession, Scrap, ReturnToSupplier)
├── DispositionBy (FK → User, nullable)
├── DispositionDate (datetime, nullable)
├── CostImpact (decimal)
├── Photos[] (string[] - blob references)
├── CorrectiveActions[]
│   ├── ActionId (GUID)
│   ├── Description (string)
│   ├── AssignedTo (FK → User)
│   ├── DueDate (datetime)
│   ├── Status (enum: Open, InProgress, Complete, Verified)
│   └── CompletedDate (datetime, nullable)
└── Status (enum: Open, UnderInvestigation, Dispositioned, Closed)
```

---

### 2.4 Traceability

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Material Traceability | Link finished parts to raw material batch/heat numbers | P0 |
| Process Traceability | Record who, what, when, where for every operation | P0 |
| Serial Number Tracking | Individual part serialisation for critical components | P1 |
| Genealogy Tree | Visual parent-child relationship for assemblies | P2 |
| Forward/Backward Trace | From material to customer, or customer complaint to source | P0 |
| Certificate Pack | Auto-compile full traceability documentation per order | P1 |
| Recall Support | Identify all affected orders from a material batch | P1 |

#### Traceability Data Chain
```
Material Certificate (Mill Cert)
    │
    ▼
Goods Receipt (Batch/Heat Number recorded)
    │
    ▼
Stock Issue to Work Order (Batch allocated)
    │
    ▼
Operation Records (Operator, Machine, Program, Time, Parameters)
    │
    ▼
Inspection Records (Measurements, Inspector, Instruments)
    │
    ▼
Final Inspection & Certificate of Conformity
    │
    ▼
Dispatch Record (Customer, Delivery Note, Date)
```

---

### 2.5 Digital Travellers (Paperless Job Cards)

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Digital Job Card | Complete job information on tablet/screen | P0 |
| Electronic Sign-Off | Operator and inspector sign-off per operation | P0 |
| Photo Attachment | Capture setup photos, defect images | P1 |
| Revision Alerts | Notify operators if drawing revision changes mid-job | P0 |
| Offline Mode | Continue working during network interruption | P1 |
| Print Option | Generate printable traveller for backup/customer requirement | P2 |
| Custom Fields | Configurable per operation type (e.g., surface finish RA value) | P1 |

#### Data Model: Digital Traveller
```
Traveller
├── TravellerId (GUID)
├── WorkOrderId (FK → WorkOrder)
├── Status (enum: Active, Complete, Archived)
├── Steps[]
│   ├── StepId (GUID)
│   ├── OperationId (FK → Operation)
│   ├── SequenceNumber (int)
│   ├── Instruction (string)
│   ├── SignOffRequired (bool)
│   ├── SignedOffBy (FK → User, nullable)
│   ├── SignedOffAt (datetime, nullable)
│   ├── InspectionRequired (bool)
│   ├── InspectionRecordId (FK → InspectionRecord, nullable)
│   ├── Attachments[] (photo/document references)
│   ├── CustomFields (JSON - flexible key/value)
│   └── Notes (string)
└── CompletedAt (datetime, nullable)
```

---

### 2.6 Tool Management

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Tool Inventory | Register all cutting tools, fixtures, gauges | P1 |
| Tool Life Tracking | Monitor usage cycles/minutes, predict replacement | P1 |
| Tool Lists per Job | Define required tooling for each operation | P1 |
| Calibration Schedule | Track calibration due dates, alert on expiry | P1 |
| Tool Crib Management | Check-in/check-out system for shared tooling | P2 |
| Preset Data | Store tool offset/geometry data per tool assembly | P2 |
| Wear Compensation | Auto-adjust based on SPC trend data | P3 |

#### Data Model: Tooling
```
Tool
├── ToolId (GUID)
├── ToolNumber (string)
├── Description (string)
├── Category (enum: CuttingTool, Fixture, Gauge, MeasuringInstrument, Jig)
├── Location (string)
├── Status (enum: Available, InUse, Maintenance, Calibration, Scrapped)
├── MaxLifeCycles (int, nullable)
├── MaxLifeMinutes (int, nullable)
├── CurrentLifeCycles (int)
├── CurrentLifeMinutes (int)
├── LifeRemainingPercent (computed)
├── CalibrationDueDate (datetime, nullable)
├── CalibrationInterval (int, days, nullable)
├── LastCalibratedDate (datetime, nullable)
├── CalibratedBy (string)
├── CertificateRef (string, nullable)
└── AssignedToMachine (FK → Machine, nullable)
```

---

## 3. Shop Floor Visibility Board

### Real-Time Production Board
A wall-mounted display (large TV/monitor) showing live production status.

```
┌─────────────────────────────────────────────────────────────────────┐
│  DIGISWISS - SHOP FLOOR STATUS          14:32  │  Shift: Day       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MACHINE STATUS                                                     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │ HAAS-01 │ │ HAAS-02 │ │ DMG-01  │ │ MAZAK-1 │ │ GRIND-1 │    │
│  │ ██████  │ │ ██████  │ │ ██████  │ │ ██████  │ │ ██████  │    │
│  │RUNNING  │ │ SETUP   │ │RUNNING  │ │  IDLE   │ │  FAULT  │    │
│  │WO-1234  │ │WO-1278  │ │WO-1256  │ │         │ │ ALARM   │    │
│  │78% done │ │         │ │45% done │ │         │ │ !!      │    │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │
│                                                                     │
│  TODAY'S TARGETS                        ALERTS                      │
│  ┌──────────────────────────┐          ┌────────────────────────┐  │
│  │ Completed: 12 / 18 jobs  │          │ ⚠ WO-1299 overdue     │  │
│  │ On-Time:   89%           │          │ ⚠ GRIND-1 fault       │  │
│  │ Scrap:     0.3%          │          │ ℹ Tool change due H-02│  │
│  │ OEE:       76%           │          │                        │  │
│  └──────────────────────────┘          └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. Business Rules

### Work Order Rules
1. Operations must be completed in sequence (Op 10 before Op 20) unless explicitly flagged as parallel
2. An operation cannot start until the previous operation's inspection is signed off (if required)
3. Operators can only work on jobs assigned to their qualified work centres
4. Maximum 1 active job per operator at any time (prevents time booking conflicts)
5. Paused jobs exceeding 4 hours auto-escalate to supervisor

### Quality Rules
1. First Article Inspection is mandatory for all new parts and new revisions
2. NCRs exceeding £500 cost impact require director review
3. Three NCRs on same part/operation within 30 days triggers mandatory process review
4. Inspection instruments must have valid calibration - system blocks usage if expired
5. Customer-returned parts always generate NCR with mandatory root cause analysis

### Traceability Rules
1. Material batch number must be recorded at first manufacturing operation
2. Aerospace/medical parts require individual serial numbers
3. All traceability records are immutable (append-only)
4. Traceability chain gaps trigger quality alert
5. Full traceability pack must be complete before dispatch sign-off

---

## 5. Reporting Requirements

| Report | Frequency | Audience |
|--------|-----------|----------|
| OEE (Overall Equipment Effectiveness) | Real-time | Production, Management |
| Operator Efficiency | Daily/Weekly | Supervisors |
| Scrap Rate by Part/Machine/Operator | Weekly | Quality, Production |
| First-Pass Yield | Weekly | Quality, Management |
| NCR Summary & Trends | Monthly | Quality, Management |
| On-Time Delivery (from MES perspective) | Daily | Production |
| Setup Time Analysis | Weekly | Production, CI team |
| SPC Control Charts | Real-time | Quality, Operators |
| Tool Life Summary | Weekly | Production, Purchasing |
| Downtime Analysis (Pareto) | Weekly | Production, Maintenance |

---

## 6. Integration Points

| System/Module | Direction | Method | Data |
|---------------|-----------|--------|------|
| ERP (Orders) | Inbound | Event Bus | Work orders, material allocations |
| ERP (Inventory) | Outbound | Event Bus | Stock movements, scrap, completions |
| SCADA | Inbound | SignalR/MQTT | Machine status, cycle counts, alarms |
| IoT | Inbound | Event Stream | Sensor data, environmental readings |
| AI Module | Bi-directional | API | Document extraction, anomaly alerts |
| CAM Systems | Inbound | File Watcher | NC programs, tool paths |
| CMM (Coordinate Measuring) | Inbound | File Import | Inspection results (DMIS/QIF format) |

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](../product-specification.md)*
