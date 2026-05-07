# DigiSwiss - Data Model & Entity Relationships

## 1. Overview

This document defines the unified data model for DigiSwiss, showing how entities relate across all modules. The system uses SQL Server as the primary relational store, with supplementary Time-Series and Blob storage for specialised data.

### Database Strategy

| Database | Technology | Purpose | Entities |
|----------|-----------|---------|----------|
| Primary (OLTP) | SQL Server 2022 | Transactional data | Orders, WOs, Customers, Inventory, Quality |
| Time-Series | TimescaleDB / InfluxDB | Sensor & machine data | Telemetry, OEE, Energy readings |
| Cache | Redis | Real-time state, sessions | Machine status, live tags, user sessions |
| Document Store | Azure Blob + Cosmos DB | Files, AI extraction results | Drawings, certs, OCR results |
| Search Index | Azure AI Search | Full-text search, NLQ | Indexed copies of key entities |

---

## 2. Entity Relationship Diagram (High-Level)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CORE DOMAIN MODEL                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐ 1    * ┌──────────┐ 1    * ┌───────────┐ 1   * ┌─────────┐ │
│  │ Customer │───────▶│  Order   │───────▶│ OrderLine │──────▶│WorkOrder│ │
│  └──────────┘        └──────────┘        └───────────┘       └────┬────┘ │
│       │                    │                    │                   │      │
│       │ 1:*                │ 1:*                │ 1:1               │ 1:*  │
│       ▼                    ▼                    ▼                   ▼      │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐       ┌─────────┐ │
│  │ Contact  │        │ Document │        │ Material │       │Operation│ │
│  └──────────┘        └──────────┘        │ (Stock)  │       └────┬────┘ │
│                                           └──────────┘            │      │
│                                                │                  │ 1:*  │
│                                                │ 1:*              ▼      │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐    ┌───────────┐ │
│  │ Supplier │───────▶│ Purchase │        │  Batch   │    │ TimeEntry │ │
│  └──────────┘  1:*   │  Order   │        │ (Stock   │    └───────────┘ │
│                       └──────────┘        │  Movement│                   │
│                                           └──────────┘                   │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│                         PRODUCTION & QUALITY                              │
│                                                                          │
│  ┌──────────┐ 1    * ┌──────────┐        ┌───────────┐                 │
│  │ Machine  │───────▶│  Tag     │        │Inspection │                 │
│  └────┬─────┘        │(DataPoint│        │  Record   │                 │
│       │              └──────────┘        └─────┬─────┘                 │
│       │ 1:*                                    │                        │
│       ▼                                        │ 0:1                    │
│  ┌──────────┐        ┌──────────┐        ┌────┴──────┐                 │
│  │  Alarm   │        │   NCR    │◀───────│   NCR     │                 │
│  │Definition│        │          │        │(if fail)  │                 │
│  └──────────┘        └──────────┘        └───────────┘                 │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│                           IoT & DEVICES                                   │
│                                                                          │
│  ┌──────────┐ 1    * ┌──────────┐        ┌───────────┐                 │
│  │  Edge    │───────▶│IoTDevice │───────▶│ Telemetry │                 │
│  │ Gateway  │        └──────────┘  1:*   │  (Time-   │                 │
│  └──────────┘                            │   Series) │                 │
│                                          └───────────┘                 │
│                                                                          │
├──────────────────────────────────────────────────────────────────────────┤
│                         SYSTEM & AUTH                                     │
│                                                                          │
│  ┌──────────┐ *    * ┌──────────┐        ┌───────────┐                 │
│  │   User   │───────▶│   Role   │───────▶│Permission │                 │
│  └──────────┘        └──────────┘  1:*   └───────────┘                 │
│       │                                                                  │
│       │ 1:*                                                              │
│       ▼                                                                  │
│  ┌──────────┐                                                           │
│  │  Audit   │                                                           │
│  │  Trail   │                                                           │
│  └──────────┘                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Schema Definitions by Domain

### 3.1 Core / Shared Schema (`dbo` or `core`)

```sql
-- Tenant (for multi-tenancy support)
CREATE TABLE Tenants (
    TenantId        UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    Name            NVARCHAR(200) NOT NULL,
    Subdomain       NVARCHAR(50) UNIQUE NOT NULL,
    IsActive        BIT NOT NULL DEFAULT 1,
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

-- Users
CREATE TABLE Users (
    UserId          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    ExternalId      NVARCHAR(200) NOT NULL,  -- Azure AD Object ID
    Email           NVARCHAR(256) NOT NULL,
    FirstName       NVARCHAR(100) NOT NULL,
    LastName        NVARCHAR(100) NOT NULL,
    DisplayName     NVARCHAR(200) NOT NULL,
    EmployeeCode    NVARCHAR(20),
    Department      NVARCHAR(100),
    IsActive        BIT NOT NULL DEFAULT 1,
    LastLoginAt     DATETIME2,
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    INDEX IX_Users_Tenant (TenantId),
    INDEX IX_Users_Email (Email)
);

-- Roles & Permissions
CREATE TABLE Roles (
    RoleId          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    Name            NVARCHAR(100) NOT NULL,
    Description     NVARCHAR(500),
    IsSystem        BIT NOT NULL DEFAULT 0  -- Cannot be deleted
);

CREATE TABLE Permissions (
    PermissionId    UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    Code            NVARCHAR(100) NOT NULL UNIQUE,  -- e.g., 'orders:write'
    Module          NVARCHAR(50) NOT NULL,
    Description     NVARCHAR(200)
);

CREATE TABLE RolePermissions (
    RoleId          UNIQUEIDENTIFIER NOT NULL REFERENCES Roles(RoleId),
    PermissionId    UNIQUEIDENTIFIER NOT NULL REFERENCES Permissions(PermissionId),
    PRIMARY KEY (RoleId, PermissionId)
);

CREATE TABLE UserRoles (
    UserId          UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    RoleId          UNIQUEIDENTIFIER NOT NULL REFERENCES Roles(RoleId),
    AssignedAt      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    AssignedBy      UNIQUEIDENTIFIER REFERENCES Users(UserId),
    PRIMARY KEY (UserId, RoleId)
);

-- Audit Trail
CREATE TABLE AuditLog (
    AuditId         BIGINT IDENTITY(1,1) PRIMARY KEY,
    TenantId        UNIQUEIDENTIFIER NOT NULL,
    UserId          UNIQUEIDENTIFIER,
    Action          NVARCHAR(50) NOT NULL,   -- Create, Update, Delete, StatusChange
    EntityType      NVARCHAR(100) NOT NULL,  -- Order, WorkOrder, NCR, etc.
    EntityId        UNIQUEIDENTIFIER NOT NULL,
    OldValues       NVARCHAR(MAX),           -- JSON
    NewValues       NVARCHAR(MAX),           -- JSON
    IpAddress       NVARCHAR(45),
    UserAgent       NVARCHAR(500),
    Timestamp       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    Source          NVARCHAR(50),            -- UI, API, Automation, System
    INDEX IX_Audit_Entity (EntityType, EntityId),
    INDEX IX_Audit_User (UserId, Timestamp),
    INDEX IX_Audit_Timestamp (Timestamp)
);
```

### 3.2 ERP Schema (`erp`)

```sql
-- Customers
CREATE TABLE erp.Customers (
    CustomerId      UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    AccountCode     NVARCHAR(20) NOT NULL,
    CompanyName     NVARCHAR(200) NOT NULL,
    TradingName     NVARCHAR(200),
    Industry        NVARCHAR(50),
    PaymentTermsDays INT NOT NULL DEFAULT 30,
    CreditLimit     DECIMAL(12,2),
    CurrencyCode    NVARCHAR(3) NOT NULL DEFAULT 'GBP',
    VATNumber       NVARCHAR(20),
    IsActive        BIT NOT NULL DEFAULT 1,
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ModifiedAt      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, AccountCode)
);

-- Orders
CREATE TABLE erp.Orders (
    OrderId         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    OrderNumber     NVARCHAR(30) NOT NULL,
    CustomerId      UNIQUEIDENTIFIER NOT NULL REFERENCES erp.Customers(CustomerId),
    QuotationId     UNIQUEIDENTIFIER,
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Draft',
    Priority        NVARCHAR(10) NOT NULL DEFAULT 'Standard',
    CustomerRef     NVARCHAR(100),
    RequiredDate    DATETIME2 NOT NULL,
    PromisedDate    DATETIME2,
    ActualDispatchDate DATETIME2,
    TotalValue      DECIMAL(12,2) NOT NULL DEFAULT 0,
    Currency        NVARCHAR(3) NOT NULL DEFAULT 'GBP',
    Notes           NVARCHAR(MAX),
    CreatedBy       UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    ModifiedAt      DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, OrderNumber),
    INDEX IX_Orders_Customer (CustomerId),
    INDEX IX_Orders_Status (TenantId, Status),
    INDEX IX_Orders_RequiredDate (TenantId, RequiredDate)
);

-- Order Lines
CREATE TABLE erp.OrderLines (
    LineId          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    OrderId         UNIQUEIDENTIFIER NOT NULL REFERENCES erp.Orders(OrderId),
    LineNumber      INT NOT NULL,
    PartNumber      NVARCHAR(50) NOT NULL,
    DrawingRef      NVARCHAR(50),
    Revision        NVARCHAR(10),
    Description     NVARCHAR(500),
    Quantity        INT NOT NULL,
    UnitPrice       DECIMAL(10,4) NOT NULL,
    LineTotal       AS (Quantity * UnitPrice) PERSISTED,
    MaterialId      UNIQUEIDENTIFIER,
    DeliveryDate    DATETIME2,
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    UNIQUE (OrderId, LineNumber)
);

-- Stock Items
CREATE TABLE erp.StockItems (
    StockItemId     UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    PartNumber      NVARCHAR(50) NOT NULL,
    Description     NVARCHAR(500) NOT NULL,
    Category        NVARCHAR(30) NOT NULL,
    UnitOfMeasure   NVARCHAR(10) NOT NULL,
    QtyOnHand       DECIMAL(12,3) NOT NULL DEFAULT 0,
    QtyAllocated    DECIMAL(12,3) NOT NULL DEFAULT 0,
    QtyOnOrder      DECIMAL(12,3) NOT NULL DEFAULT 0,
    ReorderLevel    DECIMAL(12,3),
    ReorderQty      DECIMAL(12,3),
    UnitCost        DECIMAL(10,4),
    LocationId      UNIQUEIDENTIFIER,
    IsActive        BIT NOT NULL DEFAULT 1,
    UNIQUE (TenantId, PartNumber),
    INDEX IX_Stock_Reorder (TenantId, ReorderLevel)
);

-- Suppliers
CREATE TABLE erp.Suppliers (
    SupplierId      UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    Code            NVARCHAR(20) NOT NULL,
    Name            NVARCHAR(200) NOT NULL,
    ContactName     NVARCHAR(100),
    Email           NVARCHAR(256),
    Phone           NVARCHAR(30),
    PaymentTerms    NVARCHAR(50),
    IsApproved      BIT NOT NULL DEFAULT 0,
    Rating          DECIMAL(3,1),
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, Code)
);

-- Purchase Orders
CREATE TABLE erp.PurchaseOrders (
    PurchaseOrderId UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    PONumber        NVARCHAR(30) NOT NULL,
    SupplierId      UNIQUEIDENTIFIER NOT NULL REFERENCES erp.Suppliers(SupplierId),
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Draft',
    OrderDate       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    RequiredDate    DATETIME2,
    TotalValue      DECIMAL(12,2) NOT NULL DEFAULT 0,
    ApprovedBy      UNIQUEIDENTIFIER REFERENCES Users(UserId),
    ApprovedAt      DATETIME2,
    CreatedBy       UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    UNIQUE (TenantId, PONumber)
);
```



### 3.3 MES Schema (`mes`)

```sql
-- Work Orders
CREATE TABLE mes.WorkOrders (
    WorkOrderId     UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    OrderId         UNIQUEIDENTIFIER NOT NULL REFERENCES erp.Orders(OrderId),
    OrderLineId     UNIQUEIDENTIFIER NOT NULL REFERENCES erp.OrderLines(LineId),
    WorkOrderNumber NVARCHAR(30) NOT NULL,
    PartNumber      NVARCHAR(50) NOT NULL,
    DrawingRef      NVARCHAR(50),
    Revision        NVARCHAR(10),
    Quantity        INT NOT NULL,
    QtyGood         INT NOT NULL DEFAULT 0,
    QtyScrap        INT NOT NULL DEFAULT 0,
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Released',
    Priority        NVARCHAR(10) NOT NULL DEFAULT 'Standard',
    PlannedStart    DATETIME2,
    PlannedEnd      DATETIME2,
    ActualStart     DATETIME2,
    ActualEnd       DATETIME2,
    MaterialBatchId UNIQUEIDENTIFIER,
    Notes           NVARCHAR(MAX),
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, WorkOrderNumber),
    INDEX IX_WO_Status (TenantId, Status),
    INDEX IX_WO_PlannedStart (TenantId, PlannedStart)
);

-- Operations
CREATE TABLE mes.Operations (
    OperationId     UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    WorkOrderId     UNIQUEIDENTIFIER NOT NULL REFERENCES mes.WorkOrders(WorkOrderId),
    SequenceNumber  INT NOT NULL,
    WorkCentreId    UNIQUEIDENTIFIER NOT NULL REFERENCES mes.WorkCentres(WorkCentreId),
    MachineId       UNIQUEIDENTIFIER REFERENCES scada.Machines(MachineId),
    OperatorId      UNIQUEIDENTIFIER REFERENCES Users(UserId),
    OperationType   NVARCHAR(30) NOT NULL,
    Description     NVARCHAR(500),
    SetupTimeEst    INT,  -- minutes
    RunTimeEst      INT,  -- minutes
    ActualSetupTime INT,
    ActualRunTime   INT,
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    NCProgramId     UNIQUEIDENTIFIER,
    InspectionReq   BIT NOT NULL DEFAULT 0,
    CompletedAt     DATETIME2,
    UNIQUE (WorkOrderId, SequenceNumber),
    INDEX IX_Op_Status (Status),
    INDEX IX_Op_WorkCentre (WorkCentreId, Status)
);

-- Work Centres
CREATE TABLE mes.WorkCentres (
    WorkCentreId    UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    Name            NVARCHAR(100) NOT NULL,
    Type            NVARCHAR(30) NOT NULL,
    CapacityHrsDay  DECIMAL(5,2) NOT NULL DEFAULT 8.0,
    HourlyRate      DECIMAL(8,2),
    IsActive        BIT NOT NULL DEFAULT 1
);

-- Time Entries
CREATE TABLE mes.TimeEntries (
    TimeEntryId     UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    OperationId     UNIQUEIDENTIFIER NOT NULL REFERENCES mes.Operations(OperationId),
    OperatorId      UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    EntryType       NVARCHAR(10) NOT NULL,  -- Setup, Run, Idle, Rework
    StartTime       DATETIME2 NOT NULL,
    EndTime         DATETIME2,
    DurationMinutes AS DATEDIFF(MINUTE, StartTime, EndTime) PERSISTED,
    Notes           NVARCHAR(500),
    INDEX IX_Time_Operation (OperationId),
    INDEX IX_Time_Operator (OperatorId, StartTime)
);

-- Inspection Plans
CREATE TABLE mes.InspectionPlans (
    PlanId          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    PartNumber      NVARCHAR(50) NOT NULL,
    Revision        NVARCHAR(10) NOT NULL,
    PlanType        NVARCHAR(20) NOT NULL,  -- FirstArticle, InProcess, Final
    ApprovedBy      UNIQUEIDENTIFIER REFERENCES Users(UserId),
    ApprovedAt      DATETIME2,
    IsActive        BIT NOT NULL DEFAULT 1
);

-- Inspection Characteristics
CREATE TABLE mes.InspectionCharacteristics (
    CharId          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    PlanId          UNIQUEIDENTIFIER NOT NULL REFERENCES mes.InspectionPlans(PlanId),
    SeqNumber       INT NOT NULL,
    Description     NVARCHAR(200) NOT NULL,
    Nominal         DECIMAL(12,6),
    UpperTol        DECIMAL(12,6),
    LowerTol        DECIMAL(12,6),
    UOM             NVARCHAR(10),
    InstrumentType  NVARCHAR(50),
    IsCritical      BIT NOT NULL DEFAULT 0,
    UNIQUE (PlanId, SeqNumber)
);

-- Inspection Records
CREATE TABLE mes.InspectionRecords (
    RecordId        UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    WorkOrderId     UNIQUEIDENTIFIER NOT NULL REFERENCES mes.WorkOrders(WorkOrderId),
    OperationId     UNIQUEIDENTIFIER REFERENCES mes.Operations(OperationId),
    PlanId          UNIQUEIDENTIFIER NOT NULL REFERENCES mes.InspectionPlans(PlanId),
    InspectorId     UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    InspectionDate  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    OverallResult   NVARCHAR(20) NOT NULL,  -- Pass, Fail, ConditionalPass
    SerialNumber    NVARCHAR(50),
    SignedOffBy     UNIQUEIDENTIFIER REFERENCES Users(UserId),
    INDEX IX_Insp_WO (WorkOrderId)
);

-- Measurements
CREATE TABLE mes.Measurements (
    MeasurementId   UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    RecordId        UNIQUEIDENTIFIER NOT NULL REFERENCES mes.InspectionRecords(RecordId),
    CharId          UNIQUEIDENTIFIER NOT NULL REFERENCES mes.InspectionCharacteristics(CharId),
    ActualValue     DECIMAL(12,6) NOT NULL,
    IsInTolerance   AS (CASE WHEN ActualValue BETWEEN
                        (SELECT Nominal + LowerTol FROM mes.InspectionCharacteristics WHERE CharId = CharId) AND
                        (SELECT Nominal + UpperTol FROM mes.InspectionCharacteristics WHERE CharId = CharId)
                        THEN 1 ELSE 0 END),
    Notes           NVARCHAR(200)
);

-- Non-Conformance Reports
CREATE TABLE mes.NCRs (
    NCRId           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    NCRNumber       NVARCHAR(30) NOT NULL,
    WorkOrderId     UNIQUEIDENTIFIER REFERENCES mes.WorkOrders(WorkOrderId),
    PartNumber      NVARCHAR(50) NOT NULL,
    Quantity        INT NOT NULL,
    DetectedBy      UNIQUEIDENTIFIER NOT NULL REFERENCES Users(UserId),
    DetectedAt      NVARCHAR(20) NOT NULL,
    Description     NVARCHAR(MAX) NOT NULL,
    RootCause       NVARCHAR(MAX),
    Disposition     NVARCHAR(20) NOT NULL DEFAULT 'Pending',
    DispositionBy   UNIQUEIDENTIFIER REFERENCES Users(UserId),
    DispositionDate DATETIME2,
    CostImpact      DECIMAL(10,2),
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Open',
    CreatedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, NCRNumber),
    INDEX IX_NCR_Status (TenantId, Status),
    INDEX IX_NCR_Part (PartNumber)
);
```

### 3.4 SCADA Schema (`scada`)

```sql
-- Machines
CREATE TABLE scada.Machines (
    MachineId       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    MachineCode     NVARCHAR(20) NOT NULL,
    MachineName     NVARCHAR(100) NOT NULL,
    Type            NVARCHAR(30) NOT NULL,
    Manufacturer    NVARCHAR(100),
    Model           NVARCHAR(100),
    SerialNumber    NVARCHAR(50),
    Controller      NVARCHAR(100),
    Protocol        NVARCHAR(20) NOT NULL,
    IPAddress       NVARCHAR(45),
    Port            INT,
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Offline',
    WorkCentreId    UNIQUEIDENTIFIER REFERENCES mes.WorkCentres(WorkCentreId),
    Building        NVARCHAR(50),
    Bay             NVARCHAR(50),
    IsActive        BIT NOT NULL DEFAULT 1,
    UNIQUE (TenantId, MachineCode)
);

-- Data Points (Tags)
CREATE TABLE scada.DataPoints (
    TagId           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    MachineId       UNIQUEIDENTIFIER NOT NULL REFERENCES scada.Machines(MachineId),
    TagName         NVARCHAR(200) NOT NULL,
    Protocol        NVARCHAR(20) NOT NULL,
    Address         NVARCHAR(200) NOT NULL,
    DataType        NVARCHAR(10) NOT NULL,
    Unit            NVARCHAR(20),
    ScaleFactor     DECIMAL(10,6) DEFAULT 1.0,
    PollingMs       INT NOT NULL DEFAULT 1000,
    DeadbandValue   DECIMAL(10,4) DEFAULT 0,
    MinValue        DECIMAL(12,4),
    MaxValue        DECIMAL(12,4),
    IsEnabled       BIT NOT NULL DEFAULT 1,
    UNIQUE (MachineId, TagName)
);

-- Alarm Definitions
CREATE TABLE scada.AlarmDefinitions (
    AlarmId         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    MachineId       UNIQUEIDENTIFIER NOT NULL REFERENCES scada.Machines(MachineId),
    TagId           UNIQUEIDENTIFIER REFERENCES scada.DataPoints(TagId),
    AlarmCode       NVARCHAR(50) NOT NULL,
    Name            NVARCHAR(200) NOT NULL,
    Severity        NVARCHAR(10) NOT NULL,
    TriggerType     NVARCHAR(20) NOT NULL,
    Threshold       DECIMAL(12,4),
    TimeDelayMs     INT DEFAULT 0,
    IsEnabled       BIT NOT NULL DEFAULT 1,
    RequiresAck     BIT NOT NULL DEFAULT 1
);

-- Alarm Events
CREATE TABLE scada.AlarmEvents (
    EventId         UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    AlarmId         UNIQUEIDENTIFIER NOT NULL REFERENCES scada.AlarmDefinitions(AlarmId),
    MachineId       UNIQUEIDENTIFIER NOT NULL REFERENCES scada.Machines(MachineId),
    TriggerValue    DECIMAL(12,4),
    TriggeredAt     DATETIME2 NOT NULL,
    AcknowledgedAt  DATETIME2,
    AcknowledgedBy  UNIQUEIDENTIFIER REFERENCES Users(UserId),
    ClearedAt       DATETIME2,
    Status          NVARCHAR(15) NOT NULL DEFAULT 'Active',
    Notes           NVARCHAR(500),
    INDEX IX_Alarm_Active (MachineId, Status),
    INDEX IX_Alarm_Time (TriggeredAt DESC)
);

-- OEE Records
CREATE TABLE scada.OEERecords (
    RecordId        UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    MachineId       UNIQUEIDENTIFIER NOT NULL REFERENCES scada.Machines(MachineId),
    ShiftDate       DATE NOT NULL,
    ShiftType       NVARCHAR(10) NOT NULL,
    PlannedMinutes  INT NOT NULL,
    RunMinutes      INT NOT NULL,
    DowntimeMinutes INT NOT NULL,
    TotalPieces     INT NOT NULL,
    GoodPieces      INT NOT NULL,
    ScrapPieces     INT NOT NULL,
    Availability    DECIMAL(5,4),
    Performance     DECIMAL(5,4),
    Quality         DECIMAL(5,4),
    OEE             DECIMAL(5,4),
    UNIQUE (MachineId, ShiftDate, ShiftType),
    INDEX IX_OEE_Date (ShiftDate DESC)
);
```

### 3.5 IoT Schema (`iot`)

```sql
-- IoT Devices
CREATE TABLE iot.Devices (
    DeviceId        UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId        UNIQUEIDENTIFIER NOT NULL REFERENCES Tenants(TenantId),
    DeviceCode      NVARCHAR(50) NOT NULL,
    DisplayName     NVARCHAR(200) NOT NULL,
    DeviceType      NVARCHAR(30) NOT NULL,
    Manufacturer    NVARCHAR(100),
    Model           NVARCHAR(100),
    SerialNumber    NVARCHAR(100),
    FirmwareVersion NVARCHAR(30),
    Status          NVARCHAR(20) NOT NULL DEFAULT 'Registered',
    ConnectionState NVARCHAR(15) NOT NULL DEFAULT 'Disconnected',
    LastActivityAt  DATETIME2,
    Protocol        NVARCHAR(20) NOT NULL,
    GatewayId       UNIQUEIDENTIFIER REFERENCES iot.Devices(DeviceId),
    MachineId       UNIQUEIDENTIFIER REFERENCES scada.Machines(MachineId),
    Building        NVARCHAR(50),
    Zone            NVARCHAR(50),
    Position        NVARCHAR(100),
    ReportingIntervalSec INT NOT NULL DEFAULT 30,
    BatteryLevel    DECIMAL(5,2),
    CalibrationDate DATETIME2,
    CalibrationDue  DATETIME2,
    Configuration   NVARCHAR(MAX),  -- JSON
    IsActive        BIT NOT NULL DEFAULT 1,
    ProvisionedAt   DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    UNIQUE (TenantId, DeviceCode),
    INDEX IX_Device_Status (TenantId, Status),
    INDEX IX_Device_Gateway (GatewayId)
);

-- Energy Summaries (aggregated from time-series)
CREATE TABLE iot.EnergySummaries (
    SummaryId       UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    MachineId       UNIQUEIDENTIFIER NOT NULL REFERENCES scada.Machines(MachineId),
    Period          NVARCHAR(10) NOT NULL,  -- Hour, Day, Week, Month
    PeriodStart     DATETIME2 NOT NULL,
    TotalEnergy_kWh DECIMAL(12,4) NOT NULL,
    PeakDemand_kW   DECIMAL(10,4),
    AvgLoad_kW      DECIMAL(10,4),
    Cost_GBP        DECIMAL(10,4),
    PartsProduced   INT,
    CO2_kg          DECIMAL(10,4),
    UNIQUE (MachineId, Period, PeriodStart),
    INDEX IX_Energy_Period (PeriodStart DESC)
);
```

---

## 4. Cross-Module Relationships

### 4.1 Key Foreign Key Chains

```
Customer → Order → OrderLine → WorkOrder → Operation → TimeEntry
                                    │            │
                                    │            └→ InspectionRecord → Measurements
                                    │                      │
                                    │                      └→ NCR → CorrectiveActions
                                    │
                                    └→ Machine (via Operation.MachineId)
                                            │
                                            └→ DataPoints → Telemetry (time-series)
                                            └→ AlarmDefinitions → AlarmEvents
                                            └→ IoTDevices → Telemetry (time-series)
```

### 4.2 Traceability Chain

```
Supplier → PurchaseOrder → GoodsReceipt → StockBatch (HeatNumber)
    │
    └→ StockMovement (Issue to WO) → WorkOrder → Operation (who, what machine, when)
                                          │
                                          └→ InspectionRecord (measurements, pass/fail)
                                                │
                                                └→ Order → Customer (dispatch)
```

---

## 5. Indexing Strategy

### 5.1 Clustered Indexes
- All tables use `UNIQUEIDENTIFIER` PKs with `NEWSEQUENTIALID()` for clustered index performance
- Time-series related tables (AlarmEvents, OEE) also indexed by date descending

### 5.2 Key Non-Clustered Indexes

| Table | Index | Purpose |
|-------|-------|---------|
| Orders | (TenantId, Status, RequiredDate) | Dashboard queries, overdue |
| WorkOrders | (TenantId, Status, PlannedStart) | Production planning views |
| Operations | (WorkCentreId, Status) | Queue at work centre |
| Operations | (MachineId, Status) | Machine workload |
| TimeEntries | (OperatorId, StartTime) | Operator timesheets |
| AlarmEvents | (MachineId, Status) | Active alarm display |
| NCRs | (TenantId, Status, CreatedAt) | Quality dashboard |
| StockItems | (TenantId, QtyOnHand, ReorderLevel) | Low stock alerts |
| AuditLog | (EntityType, EntityId) | Entity history lookup |

### 5.3 Full-Text Indexes

| Table | Columns | Use Case |
|-------|---------|----------|
| Orders | Notes, CustomerRef | Search orders by reference |
| NCRs | Description, RootCause | Quality investigation |
| StockItems | Description, PartNumber | Inventory search |
| Customers | CompanyName, TradingName | Customer lookup |

---

## 6. Data Retention & Archiving

| Data Category | Active Retention | Archive | Delete |
|---------------|-----------------|---------|--------|
| Orders & Lines | Until archived (5 years) | 5-10 years (cold storage) | Never (regulatory) |
| Work Orders | 3 years | 3-10 years | Never |
| Time Entries | 2 years | 2-7 years | After 7 years |
| Inspection Records | 10 years (active) | Permanent | Never |
| NCRs | 10 years (active) | Permanent | Never |
| Alarm Events | 1 year (hot) | 7 years (warm) | After 7 years |
| Telemetry (raw) | 90 days | 2 years (aggregated) | Raw after 90 days |
| Audit Trail | 3 years | 7 years | After 7 years |
| User Sessions | 30 days | N/A | After 30 days |

---

## 7. Multi-Tenancy Model

DigiSwiss uses **row-level security** with a shared database:

```sql
-- Row-Level Security Policy
CREATE SECURITY POLICY TenantFilter
    ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON erp.Orders,
    ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON erp.Customers,
    ADD FILTER PREDICATE dbo.fn_TenantFilter(TenantId) ON mes.WorkOrders,
    -- ... all tenant-scoped tables
WITH (STATE = ON);
```

---

## 8. Migration Strategy

| Source | Migration Approach | Validation |
|--------|-------------------|------------|
| Existing prototype DB | Schema comparison + data migration scripts | Row count + checksum |
| Legacy CSV exports | ETL pipeline with data cleansing | Business rule validation |
| Paper records | Document Intelligence pipeline | Human verification |
| Spreadsheets | Structured import with mapping UI | Preview + confirm |

---

*Document Version: 1.0*  
*Parent Document: [Product Specification](product-specification.md)*
