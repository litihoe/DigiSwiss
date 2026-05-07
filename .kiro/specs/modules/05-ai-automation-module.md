# DigiSwiss - AI & Automation Module Detailed Specification

## 1. Module Overview

The AI & Automation module is the intelligence layer of DigiSwiss, responsible for reducing manual effort, enabling proactive decision-making, and automating repetitive workflows. It encompasses document intelligence (OCR/GPT-powered extraction), event-driven automation (SQL-triggered workflows), QR-based operations, intelligent scheduling, anomaly detection, and natural language interfaces.

**Key Targets:**
- 70% reduction in manual data entry via document intelligence
- 40-60% reduction in check-in/onboarding time via QR operations
- Real-time operational alerts with zero manual monitoring

---

## 2. Document Intelligence Pipeline

### 2.1 Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    DOCUMENT INTELLIGENCE PIPELINE                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐   ┌─────┐│
│  │  Input  │──▶│  Pre-    │──▶│  AI      │──▶│  Post-  │──▶│Store││
│  │ Sources │   │Processing│   │ Extract  │   │Process  │   │     ││
│  └─────────┘   └──────────┘   └──────────┘   └─────────┘   └─────┘│
│       │              │              │              │             │    │
│   • Email        • Deskew      • OCR (Azure   • Validate    • ERP  │
│   • Upload       • Enhance       Form Recog)  • Confidence   • MES  │
│   • Scanner      • Classify    • GPT-4 parse    check       • DMS  │
│   • File watch   • Split pages • Custom model • Human review        │
│   • API          • Rotate      • NER extract  • Enrich              │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 Supported Document Types

| Document Type | Extraction Fields | Confidence Target | Volume/Month |
|--------------|-------------------|-------------------|--------------|
| Purchase Orders (inbound) | PO number, lines, quantities, prices, dates, supplier | >95% | 200-500 |
| Engineering Drawings | Part number, revision, material, tolerances, dimensions | >85% | 100-300 |
| Material Certificates | Heat number, grade, test results, supplier, standards | >90% | 150-400 |
| Delivery Notes | DN number, quantities, PO reference, carrier | >95% | 200-500 |
| Invoices | Invoice number, lines, amounts, VAT, payment terms | >95% | 200-500 |
| Customer Specifications | Requirements, standards, special processes | >80% | 20-50 |
| Inspection Reports (external) | Dimensions, pass/fail, certificate references | >85% | 50-150 |
| Quotation Requests | Part details, quantities, materials, delivery requirements | >90% | 100-300 |

### 2.3 Pipeline Stages

#### Stage 1: Document Ingestion
```
DocumentIngestion
├── IngestId (GUID)
├── Source (enum: Email, Upload, Scanner, FileWatch, API)
├── OriginalFilename (string)
├── FileType (enum: PDF, TIFF, PNG, JPEG, DOCX, XLSX)
├── FileSize (long, bytes)
├── BlobReference (string)
├── ReceivedAt (datetime)
├── SenderEmail (string, nullable)
├── AssociatedEntity (string, nullable - e.g., supplier name, customer ref)
└── Status (enum: Received, Queued, Processing, Complete, Failed, ManualReview)
```

#### Stage 2: Classification
```
DocumentClassification
├── ClassificationId (GUID)
├── IngestId (FK → DocumentIngestion)
├── PredictedType (enum: PurchaseOrder, Drawing, MaterialCert, DeliveryNote, Invoice, Spec, InspectionReport, QuoteRequest, Unknown)
├── Confidence (decimal, 0-1)
├── AlternativeTypes[] (type + confidence pairs)
├── ClassifiedBy (enum: AIModel, Rule, Manual)
├── ModelVersion (string)
└── ProcessedAt (datetime)
```

#### Stage 3: Extraction
```
ExtractionResult
├── ExtractionId (GUID)
├── IngestId (FK → DocumentIngestion)
├── ClassificationType (enum)
├── ModelUsed (enum: AzureFormRecognizer, GPT4Vision, CustomModel, Hybrid)
├── Fields[]
│   ├── FieldName (string, e.g., "PurchaseOrderNumber")
│   ├── Value (string)
│   ├── Confidence (decimal, 0-1)
│   ├── BoundingBox (coordinates on page)
│   ├── PageNumber (int)
│   ├── ValidationStatus (enum: Valid, Invalid, NeedsReview)
│   └── ValidationRule (string, nullable - which rule failed)
├── Tables[]
│   ├── TableName (string, e.g., "OrderLines")
│   ├── Rows[]
│   │   └── Cells[] (column name + value + confidence)
│   └── RowCount (int)
├── OverallConfidence (decimal)
├── ProcessingTime (TimeSpan)
├── TokensUsed (int, for GPT calls)
└── Cost (decimal, estimated API cost)
```

#### Stage 4: Validation & Human Review
```
HumanReviewTask
├── TaskId (GUID)
├── ExtractionId (FK → ExtractionResult)
├── Reason (enum: LowConfidence, ValidationFailed, UnknownDocument, AmbiguousField)
├── FlaggedFields (string[] - which fields need review)
├── AssignedTo (FK → User, nullable)
├── OriginalValues (JSON - AI-extracted values)
├── CorrectedValues (JSON - human-corrected values, nullable)
├── Status (enum: Pending, InProgress, Completed, Skipped)
├── ReviewedBy (FK → User, nullable)
├── ReviewedAt (datetime, nullable)
├── TimeToReview (TimeSpan, nullable)
└── FeedbackForTraining (bool - flag for model improvement)
```

### 2.4 GPT Integration Strategy

| Use Case | Model | Approach | Cost Control |
|----------|-------|----------|--------------|
| Document classification | GPT-4o-mini | Few-shot prompt with examples | Cached prompts, batch classify |
| Field extraction (complex) | GPT-4o | Structured output with schema | Only for low-confidence OCR results |
| Drawing interpretation | GPT-4 Vision | Image analysis for title blocks | Only when OCR fails on drawings |
| Data validation | GPT-4o-mini | Cross-reference extracted data | Only flagged inconsistencies |
| Natural language queries | GPT-4o | RAG over manufacturing data | Rate-limited per user |

### 2.5 Confidence & Routing Rules

| Confidence Level | Action | Destination |
|-----------------|--------|-------------|
| ≥ 95% | Auto-accept, create records | Directly to ERP/MES |
| 85-94% | Auto-accept with highlight | To ERP/MES + notify user for spot-check |
| 70-84% | Needs review | Human review queue (priority: normal) |
| 50-69% | Needs review | Human review queue (priority: high) |
| < 50% | Reject/manual entry | Alert + manual processing |

---

## 3. SQL-Triggered Automation Engine

### 3.1 Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTOMATION ENGINE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────┐    ┌───────────────┐    ┌───────────────────┐   │
│  │  Trigger  │───▶│    Rule       │───▶│     Action        │   │
│  │  Sources  │    │   Engine      │    │    Executor       │   │
│  └───────────┘    └───────────────┘    └───────────────────┘   │
│       │                  │                       │               │
│   • SQL Change      • Condition            • Send email/SMS     │
│     Tracking          evaluation           • Create record      │
│   • Event Bus       • Time-based           • Update status      │
│   • Schedule          rules                • Push notification  │
│   • Webhook        • Complex event         • Trigger workflow   │
│   • Timer            processing            • Call API           │
│                                            • Generate document  │
│                                            • Escalate           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Trigger Types

| Trigger | Detection Method | Latency | Example |
|---------|-----------------|---------|---------|
| Data Change | SQL Change Tracking / CDC | < 5 seconds | Order status changed to "Dispatched" |
| Threshold Breach | Continuous query on live data | < 10 seconds | Stock level below reorder point |
| Schedule | Cron-based timer | At scheduled time | Daily overdue order report |
| Event | Message bus subscription | < 2 seconds | Machine fault raised |
| Absence | Watchdog timer | Configurable | No goods receipt for 3 days after PO due |
| Pattern | Complex Event Processing | < 30 seconds | 3 NCRs on same part within 7 days |

### 3.3 Automation Rule Data Model

```
AutomationRule
├── RuleId (GUID)
├── Name (string, e.g., "Overdue Order Alert")
├── Description (string)
├── Category (enum: OrderManagement, Inventory, Quality, Production, Maintenance, Financial, HR)
├── IsEnabled (bool)
├── Priority (int, execution order when multiple rules fire)
├── Trigger
│   ├── Type (enum: DataChange, Threshold, Schedule, Event, Absence, Pattern)
│   ├── Source (string - table name, event type, or cron expression)
│   ├── Condition (string - SQL predicate or expression)
│   └── Parameters (JSON - trigger-specific config)
├── Conditions[]
│   ├── Field (string)
│   ├── Operator (enum: Equals, NotEquals, GreaterThan, LessThan, Contains, In, Between)
│   ├── Value (string)
│   └── LogicalOperator (enum: AND, OR)
├── Actions[]
│   ├── ActionId (GUID)
│   ├── SequenceNumber (int)
│   ├── Type (enum: SendEmail, SendSMS, PushNotification, CreateRecord, UpdateField, CallAPI, GenerateDocument, TriggerWorkflow, Escalate, Webhook)
│   ├── Configuration (JSON - action-specific settings)
│   ├── DelayBefore (TimeSpan, nullable)
│   └── OnFailure (enum: Continue, Abort, Retry)
├── Cooldown (TimeSpan - minimum time between firings)
├── MaxFiresPerDay (int, nullable - rate limiting)
├── ExecutionLog[]
│   ├── ExecutionId (GUID)
│   ├── FiredAt (datetime)
│   ├── TriggerData (JSON - what caused the fire)
│   ├── ActionsExecuted (int)
│   ├── Status (enum: Success, PartialFailure, Failed)
│   ├── Duration (TimeSpan)
│   └── ErrorMessage (string, nullable)
└── CreatedBy (FK → User)
```

### 3.4 Pre-Built Automation Templates

| Template | Trigger | Action | Category |
|----------|---------|--------|----------|
| Overdue Order Alert | Order.RequiredDate < Today AND Status != Complete | Email to sales + production manager | Orders |
| Low Stock Alert | StockItem.QuantityAvailable ≤ ReorderLevel | Email purchasing + auto-create draft PO | Inventory |
| NCR Escalation | NCR created with cost > £500 | Push notification to director | Quality |
| Machine Fault Alert | Machine.Status changed to "Fault" | SMS to maintenance + supervisor | Production |
| Delivery Confirmation | GoodsReceipt created | Email to requisitioner + update PO status | Purchasing |
| Inspection Overdue | Inspection due but not started after 4 hours | Alert to quality manager | Quality |
| Job Complete | WorkOrder all operations complete | Email customer + generate dispatch note | Production |
| Invoice Ready | Order status → Dispatched + delivery confirmed | Create draft invoice in accounts | Financial |
| Tool Life Warning | Tool.LifeRemainingPercent < 15% | Alert operator + create replacement PO | Maintenance |
| Calibration Due | Tool.CalibrationDueDate within 7 days | Email to quality + block usage warning | Quality |

---

## 4. QR-Based Operations

### 4.1 QR Code Use Cases

| Use Case | QR Content | Action on Scan | Time Saving |
|----------|-----------|----------------|-------------|
| Operator Clock-In | Employee badge QR | Start shift, show dashboard | 40% vs manual |
| Job Start | Work order QR (on traveller) | Start timer, load job details | 50% vs searching |
| Machine Login | Machine-mounted QR | Associate operator to machine | 60% vs manual entry |
| Material Issue | Stock label QR | Record issue to work order | 45% vs paper forms |
| Inspection Point | Part/batch QR | Open inspection form with context | 40% vs manual lookup |
| Asset Identification | Equipment label QR | Show maintenance history, manuals | 55% vs filing cabinets |
| Visitor Check-In | Visitor pass QR | Register arrival, notify host, H&S brief | 60% vs paper sign-in |
| Tool Check-Out | Tool crib QR | Record borrowing, track location | 50% vs logbook |

### 4.2 QR Code Architecture

```
QR Code Content Format:
  digiswiss://{entity_type}/{entity_id}[?action={default_action}]

Examples:
  digiswiss://employee/a1b2c3d4-e5f6-7890-abcd-ef1234567890?action=clockin
  digiswiss://workorder/WO-2026-001234?action=start
  digiswiss://machine/HAAS-01?action=login
  digiswiss://stock/BATCH-HT2026-0456?action=issue
  digiswiss://tool/T-00123?action=checkout
```

### 4.3 QR Data Model

```
QRCodeRegistry
├── QRId (GUID)
├── EntityType (enum: Employee, WorkOrder, Machine, StockItem, Tool, Asset, Visitor, Location)
├── EntityId (GUID)
├── Code (string - encoded content)
├── DefaultAction (string)
├── GeneratedAt (datetime)
├── PrintedAt (datetime, nullable)
├── IsActive (bool)
├── ExpiresAt (datetime, nullable - for temporary codes like visitors)
└── ScanLog[]
    ├── ScanId (GUID)
    ├── ScannedBy (FK → User)
    ├── ScannedAt (datetime)
    ├── DeviceInfo (string - phone model, app version)
    ├── Location (GPS coordinates, nullable)
    ├── ActionTaken (string)
    └── Result (enum: Success, Failed, Expired, Unauthorized)
```

---

## 5. Intelligent Scheduling (AI-Assisted)

### 5.1 Scheduling Optimisation Engine

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Current    │───▶│  Constraint  │───▶│     ML       │───▶│  Optimised   │
│   Schedule   │    │  Definition  │    │  Optimiser   │    │  Schedule    │
│  + New Jobs  │    │              │    │              │    │  (Proposed)  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
        │                  │                   │                    │
    • Open orders      • Due dates        • Genetic algo      • Gantt view
    • Machine avail    • Machine caps     • Constraint        • KPI impact
    • Operator skills  • Setup times        satisfaction      • What-if
    • Material avail   • Dependencies     • Historical        • Accept/reject
    • Current WIP      • Priorities         learning
```

### 5.2 Scheduling Constraints

| Constraint Type | Description | Hard/Soft |
|----------------|-------------|-----------|
| Due Date | Job must complete by customer required date | Hard |
| Machine Capability | Job can only run on specific machine types | Hard |
| Operator Qualification | Operator must be qualified for operation | Hard |
| Material Availability | Material must be in stock before start | Hard |
| Operation Sequence | Operations must follow defined order | Hard |
| Setup Minimisation | Group similar jobs to reduce changeover | Soft |
| Priority Weighting | Higher priority jobs scheduled earlier | Soft |
| Load Balancing | Distribute work evenly across machines | Soft |
| Operator Preference | Consider operator skill level (faster on familiar parts) | Soft |
| Energy Cost | Prefer off-peak scheduling for energy-intensive ops | Soft |

### 5.3 Scheduling Data Model

```
ScheduleOptimisation
├── OptimisationId (GUID)
├── RequestedBy (FK → User)
├── RequestedAt (datetime)
├── Horizon (TimeSpan - how far ahead to schedule)
├── InputJobs (int - number of jobs considered)
├── Constraints (JSON - active constraints and weights)
├── Algorithm (enum: GeneticAlgorithm, ConstraintProgramming, Heuristic)
├── ComputeTime (TimeSpan)
├── Results
│   ├── ProposedSchedule[] (job → machine → time slot assignments)
│   ├── OnTimeDeliveryRate (decimal, predicted %)
│   ├── MachineUtilisation (decimal, average %)
│   ├── TotalSetupTime (TimeSpan)
│   ├── Violations[] (any soft constraint violations with impact)
│   └── ComparisonVsCurrent (JSON - improvement metrics)
├── Status (enum: Computing, Ready, Accepted, Rejected, Expired)
├── AcceptedBy (FK → User, nullable)
└── AcceptedAt (datetime, nullable)
```

---

## 6. Anomaly Detection

### 6.1 Anomaly Types

| Anomaly | Detection Method | Data Source | Response |
|---------|-----------------|-------------|----------|
| Process Drift | SPC control chart rules (Western Electric) | Inspection measurements | Alert quality + operator |
| Machine Degradation | Trend analysis on vibration/temperature | IoT sensors | Maintenance alert |
| Unusual Cycle Time | Statistical deviation from baseline | MES time entries | Alert supervisor |
| Energy Spike | Sudden increase vs baseline profile | Power meters | Investigate + log |
| Material Defect Pattern | Clustering of NCRs by batch/supplier | Quality data | Alert purchasing + QA |
| Operator Performance | Significant deviation from peer average | MES productivity data | Supervisor review (sensitive) |
| Cost Overrun | Job cost trending above estimate | ERP costing data | Alert PM + finance |

### 6.2 Anomaly Detection Models

```
AnomalyDetectionConfig
├── ConfigId (GUID)
├── Name (string)
├── DataSource (enum: Inspection, IoTSensor, MESTime, Energy, Quality, Cost)
├── Method (enum: StatisticalSPC, IsolationForest, LSTM_Autoencoder, ZScore, MovingAverage)
├── Parameters
│   ├── BaselinePeriod (TimeSpan - training window)
│   ├── Sensitivity (decimal - how many σ for alert)
│   ├── MinDataPoints (int - minimum before active)
│   ├── EvaluationWindow (TimeSpan)
│   └── SeasonalityPeriod (TimeSpan, nullable)
├── AlertConfig
│   ├── Severity (enum: Low, Medium, High)
│   ├── NotifyRoles (string[])
│   ├── Cooldown (TimeSpan)
│   └── AutoCreateNCR (bool - for quality anomalies)
├── IsEnabled (bool)
└── LastEvaluated (datetime)

AnomalyEvent
├── EventId (GUID)
├── ConfigId (FK → AnomalyDetectionConfig)
├── DetectedAt (datetime)
├── EntityType (string - Machine, Part, Operator, Supplier)
├── EntityId (GUID)
├── Description (string)
├── ExpectedValue (decimal)
├── ActualValue (decimal)
├── DeviationScore (decimal - how anomalous, in σ)
├── Context (JSON - surrounding data points for investigation)
├── Status (enum: New, Investigating, Confirmed, FalsePositive, Resolved)
├── ResolvedBy (FK → User, nullable)
└── Resolution (string, nullable)
```

---

## 7. Natural Language Interface

### 7.1 Capabilities

| Query Type | Example | Data Source | Response Format |
|-----------|---------|-------------|-----------------|
| Status | "What's the status of order 1234?" | ERP | Structured card |
| Metrics | "What was our OEE last week?" | SCADA/MES | Chart + number |
| Search | "Show me all overdue orders for Rolls-Royce" | ERP | Table |
| Comparison | "Compare scrap rates this month vs last" | Quality | Chart |
| Forecast | "Will WO-5678 meet its due date?" | MES + AI | Prediction + confidence |
| Action | "Create a purchase order for 100kg 316L" | ERP | Guided form (confirm) |
| Explanation | "Why was NCR-0456 raised?" | Quality | Summary from records |

### 7.2 Architecture (RAG - Retrieval Augmented Generation)

```
┌──────────┐    ┌───────────────┐    ┌──────────────┐    ┌──────────┐
│  User    │───▶│   Intent      │───▶│   Query      │───▶│  Format  │
│  Query   │    │ Classification│    │  Execution   │    │ Response │
│(natural  │    │ + Entity      │    │ (SQL/API/    │    │ (text +  │
│ language)│    │  Extraction   │    │  search)     │    │  visual) │
└──────────┘    └───────────────┘    └──────────────┘    └──────────┘
                       │                     │
                       ▼                     ▼
                ┌─────────────┐      ┌─────────────┐
                │  Permission │      │   Context   │
                │   Check     │      │  Enrichment │
                │(user can see│      │(add relevant│
                │ this data?) │      │ background) │
                └─────────────┘      └─────────────┘
```

### 7.3 Security & Guardrails

| Rule | Implementation |
|------|---------------|
| Data access control | NLQ respects same RBAC as UI - operators can't query financial data |
| Action confirmation | Any write operation requires explicit user confirmation |
| Query auditing | All queries logged with user, timestamp, and data accessed |
| Rate limiting | Max 20 queries/user/hour to control API costs |
| Scope limitation | Cannot access data outside user's permitted entities |
| Hallucination prevention | All responses cite source records; "I don't know" for uncertain |

---

## 8. Workflow Automation (Low-Code)

### 8.1 Visual Workflow Builder

Users can create custom automation workflows via a drag-and-drop interface.

```
Workflow Components:
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  TRIGGERS        LOGIC             ACTIONS          OUTPUTS       │
│  ┌────────┐     ┌────────┐       ┌────────┐      ┌────────┐   │
│  │On Event│     │If/Else │       │Send    │      │Create  │   │
│  │On Time │     │Switch  │       │ Email  │      │ Record │   │
│  │On Data │     │Loop    │       │Update  │      │Generate│   │
│  │On API  │     │Delay   │       │ Field  │      │ Report │   │
│  │Manual  │     │Parallel│       │Call API│      │Webhook │   │
│  └────────┘     └────────┘       └────────┘      └────────┘   │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 Workflow Data Model

```
Workflow
├── WorkflowId (GUID)
├── Name (string)
├── Description (string)
├── Category (string)
├── Version (int)
├── Status (enum: Draft, Active, Paused, Archived)
├── Trigger
│   ├── Type (enum)
│   └── Configuration (JSON)
├── Steps[]
│   ├── StepId (GUID)
│   ├── Type (enum: Action, Condition, Delay, Parallel, SubWorkflow)
│   ├── Name (string)
│   ├── Configuration (JSON)
│   ├── NextStepOnSuccess (GUID, nullable)
│   ├── NextStepOnFailure (GUID, nullable)
│   └── Timeout (TimeSpan, nullable)
├── CreatedBy (FK → User)
├── LastModifiedAt (datetime)
└── ExecutionHistory[]
    ├── ExecutionId (GUID)
    ├── StartedAt (datetime)
    ├── CompletedAt (datetime, nullable)
    ├── Status (enum: Running, Completed, Failed, Cancelled, TimedOut)
    ├── TriggerData (JSON)
    ├── StepResults[] (step-by-step execution log)
    └── ErrorDetails (string, nullable)
```

---

## 9. Performance & Cost Management

### 9.1 AI Cost Tracking

| Service | Pricing Model | Monthly Budget | Monitoring |
|---------|--------------|----------------|------------|
| Azure Form Recognizer | Per page | £200 | Per-document tracking |
| GPT-4o | Per token (input/output) | £500 | Per-request logging |
| GPT-4o-mini | Per token | £100 | Per-request logging |
| GPT-4 Vision | Per image + tokens | £150 | Per-document tracking |
| Azure AI Search | Per unit + transactions | £100 | Monthly |
| ML.NET (self-hosted) | Compute only | Included in infra | CPU/memory monitoring |

### 9.2 Cost Optimisation Strategies

| Strategy | Implementation | Savings |
|----------|---------------|---------|
| Model cascading | Use mini for simple, full for complex | 40-60% |
| Caching | Cache identical document layouts | 20-30% |
| Batch processing | Group similar documents | 15-20% |
| Confidence routing | Only use GPT when OCR confidence is low | 30-50% |
| Template matching | Use rules for known document formats | 70%+ (no AI cost) |
| Edge classification | Classify locally before cloud processing | 10-15% |

---

## 10. Business Rules

### Document Intelligence Rules
1. Documents with overall confidence <50% are rejected and require manual entry
2. Financial documents (invoices, POs) require 95%+ confidence for auto-processing
3. All AI-extracted data is flagged as "AI-Generated" in audit trail until human-verified
4. Human corrections are fed back into training pipeline (with consent)
5. Maximum 24-hour SLA for human review queue items

### Automation Engine Rules
1. All automation rules must have an owner and are disabled if owner leaves
2. Rules firing >100 times/day trigger review notification
3. Failed actions retry 3 times with exponential backoff before alerting
4. Automation cannot delete records - only create, update, or flag
5. All automated changes are tagged with rule ID in audit trail

### AI Safety Rules
1. AI never makes financial commitments without human approval
2. Natural language interface cannot execute DELETE operations
3. Predictive outputs always show confidence level
4. AI decisions affecting quality/safety require human confirmation
5. Monthly accuracy review of all AI models with performance report

---

## 11. Integration Points

| System/Module | Direction | Method | Data |
|---------------|-----------|--------|------|
| ERP | Bi-directional | API + Event Bus | Extracted document data → records; triggers from data changes |
| MES | Outbound | Event Bus | Schedule optimisation results, anomaly alerts |
| SCADA | Inbound | Event Stream | Machine data for anomaly detection |
| IoT | Inbound | Event Stream | Sensor data for predictive models |
| Azure AI Services | Outbound | REST API | Document processing, GPT calls |
| Email (Exchange/SMTP) | Bi-directional | Graph API / SMTP | Inbound docs, outbound notifications |
| Blob Storage | Bi-directional | SDK | Document storage and retrieval |
| Azure AI Search | Bi-directional | REST API | Indexed manufacturing data for RAG |

---

## 12. Metrics & KPIs

| Metric | Target | Measurement |
|--------|--------|-------------|
| Document auto-processing rate | >70% (no human touch) | Processed / Total documents |
| Extraction accuracy | >92% average across all types | Correct fields / Total fields |
| Human review turnaround | <4 hours average | Time from queue to review complete |
| Automation rule success rate | >98% | Successful executions / Total fires |
| Alert response time | <15 minutes (critical) | Time from alert to acknowledgement |
| NLQ response accuracy | >90% | Correct answers / Total queries |
| Cost per document | <£0.15 average | Total AI spend / Documents processed |
| Model drift detection | Monthly check | Accuracy trend over rolling 30 days |

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](../product-specification.md)*
