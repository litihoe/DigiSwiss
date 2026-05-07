# DigiSwiss - ERP Module Detailed Specification

## 1. Module Overview

The ERP module serves as the business backbone of DigiSwiss, managing the full lifecycle of customer orders from quotation through to invoicing and delivery. It is purpose-built for precision engineering manufacturers handling complex, high-value, low-volume production with strict traceability requirements.

---

## 2. Sub-Modules

### 2.1 Order Management

#### Features
| Feature | Description | User Story |
|---------|-------------|------------|
| Quotation Builder | Create detailed quotations with material, labour, and overhead costing | As a sales engineer, I want to quickly build accurate quotes so customers receive timely responses |
| Order Entry | Convert approved quotes to works orders with automatic BOM generation | As an admin, I want quotes to flow into orders without re-keying data |
| Order Tracking | Visual pipeline showing all orders from receipt to dispatch | As a production manager, I want to see all active orders and their status at a glance |
| Revision Control | Track order amendments with full audit trail | As a quality manager, I want to trace any changes to customer requirements |
| Priority Management | Flag urgent orders, re-prioritise production queue | As a planner, I want to escalate urgent jobs without disrupting the entire schedule |
| Customer Portal | Self-service order status for customers (optional) | As a customer, I want to check my order progress without calling |

#### Workflow: Quote-to-Order
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Enquiry │───▶│ Estimate │───▶│ Quotation│───▶│ Approval │───▶│  Works   │
│ Received │    │ & Cost   │    │  Issued  │    │ (Customer│    │  Order   │
└──────────┘    └──────────┘    └──────────┘    │  Accept) │    │ Created  │
                                                └──────────┘    └──────────┘
                                                      │
                                                      ▼ (Rejected)
                                                ┌──────────┐
                                                │  Archive │
                                                │ / Revise │
                                                └──────────┘
```

#### Data Model: Orders
```
Order
├── OrderId (GUID)
├── OrderNumber (string, auto-generated, e.g., "WO-2026-001234")
├── CustomerId (FK → Customer)
├── QuotationId (FK → Quotation, nullable)
├── Status (enum: Draft, Confirmed, InProduction, QualityHold, Complete, Dispatched, Invoiced)
├── Priority (enum: Standard, High, Urgent, Critical)
├── RequiredDate (datetime)
├── PromisedDate (datetime)
├── ActualDispatchDate (datetime, nullable)
├── TotalValue (decimal)
├── Currency (string, default "GBP")
├── Notes (string)
├── CreatedBy (FK → User)
├── CreatedAt (datetime)
├── ModifiedAt (datetime)
└── OrderLines[]
    ├── LineId (GUID)
    ├── PartNumber (string)
    ├── DrawingReference (string)
    ├── RevisionNumber (string)
    ├── Quantity (int)
    ├── UnitPrice (decimal)
    ├── Material (FK → Material)
    ├── Operations[] (FK → Operation)
    ├── DeliveryDate (datetime)
    └── Status (enum: Pending, InProgress, Complete, Shipped)
```

---

### 2.2 Inventory & Stock Management

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Stock Levels | Real-time view of raw materials, WIP, and finished goods | P0 |
| Reorder Points | Configurable min/max levels with automatic alerts | P0 |
| Material Traceability | Full batch/heat number tracking from receipt to finished part | P0 |
| Goods Receipt | Book in deliveries against purchase orders with inspection | P1 |
| Stock Movements | Track all movements with reason codes (issue, return, scrap, adjust) | P0 |
| Location Management | Warehouse/bay/bin location tracking | P1 |
| Stocktake | Periodic and perpetual inventory counting | P2 |
| Material Certificates | Store and link mill certs, test certificates to batches | P1 |

#### Data Model: Inventory
```
StockItem
├── StockItemId (GUID)
├── PartNumber (string)
├── Description (string)
├── Category (enum: RawMaterial, Consumable, Tooling, FinishedGoods, WIP)
├── UnitOfMeasure (string)
├── QuantityOnHand (decimal)
├── QuantityAllocated (decimal)
├── QuantityAvailable (computed: OnHand - Allocated)
├── QuantityOnOrder (decimal)
├── ReorderLevel (decimal)
├── ReorderQuantity (decimal)
├── UnitCost (decimal)
├── Location (FK → StockLocation)
├── BatchNumbers[]
│   ├── BatchId (GUID)
│   ├── BatchNumber (string)
│   ├── HeatNumber (string)
│   ├── SupplierCertRef (string)
│   ├── QuantityReceived (decimal)
│   ├── QuantityRemaining (decimal)
│   ├── ReceivedDate (datetime)
│   └── ExpiryDate (datetime, nullable)
└── Movements[]
    ├── MovementId (GUID)
    ├── Type (enum: Receipt, Issue, Return, Scrap, Adjustment, Transfer)
    ├── Quantity (decimal)
    ├── Reference (string - WO number, PO number, etc.)
    ├── ReasonCode (string)
    ├── PerformedBy (FK → User)
    └── Timestamp (datetime)
```

---

### 2.3 Production Planning & Scheduling

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Capacity Planning | Visual capacity view per machine/work centre | P0 |
| Job Scheduling | Drag-and-drop Gantt chart for job sequencing | P0 |
| Resource Allocation | Assign operators, machines, and tooling to jobs | P0 |
| Load Balancing | Highlight over/under-loaded work centres | P1 |
| What-If Scenarios | Simulate schedule changes before committing | P2 |
| Subcontract Management | Track external operations (heat treat, plating, etc.) | P1 |
| Due Date Alerts | Automatic notifications for at-risk deliveries | P0 |
| Setup Optimisation | Group similar jobs to minimise changeover time | P2 |

#### Workflow: Production Planning
```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Works Order │────▶│   Routing    │────▶│   Schedule   │
│   Created    │     │  Generated   │     │   (Assign    │
│              │     │ (Operations) │     │   to slots)  │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                     ┌──────────────┐     ┌──────┴───────┐
                     │   Release    │◀────│   Material   │
                     │   to Shop    │     │  Availability│
                     │    Floor     │     │    Check     │
                     └──────────────┘     └──────────────┘
```

#### Data Model: Production
```
WorkCentre
├── WorkCentreId (GUID)
├── Name (string)
├── Type (enum: CNC_Turning, CNC_Milling, Grinding, EDM, Inspection, Assembly, Manual)
├── Machines[] (FK → Machine)
├── CapacityHoursPerDay (decimal)
├── SetupTimeDefault (TimeSpan)
├── HourlyRate (decimal)
└── IsActive (bool)

ProductionSchedule
├── ScheduleId (GUID)
├── OrderLineId (FK → OrderLine)
├── WorkCentreId (FK → WorkCentre)
├── MachineId (FK → Machine, nullable)
├── OperatorId (FK → User, nullable)
├── OperationId (FK → Operation)
├── PlannedStart (datetime)
├── PlannedEnd (datetime)
├── ActualStart (datetime, nullable)
├── ActualEnd (datetime, nullable)
├── Status (enum: Planned, Released, InProgress, Complete, OnHold)
├── SetupTime (TimeSpan)
├── RunTime (TimeSpan)
└── Notes (string)
```

---

### 2.4 Purchasing & Procurement

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Purchase Orders | Create, approve, and track POs | P0 |
| Supplier Management | Supplier database with performance ratings | P1 |
| Approval Workflows | Configurable approval thresholds (e.g., >£5000 needs director sign-off) | P1 |
| Goods Receipt | Book deliveries, trigger inspection if required | P0 |
| Invoice Matching | 3-way match (PO, GRN, Invoice) | P2 |
| Supplier Scorecard | On-time delivery, quality, pricing metrics | P2 |
| Blanket Orders | Framework agreements for repeat materials | P2 |
| RFQ Management | Request and compare quotes from multiple suppliers | P2 |

#### Data Model: Purchasing
```
PurchaseOrder
├── PurchaseOrderId (GUID)
├── PONumber (string, auto-generated)
├── SupplierId (FK → Supplier)
├── Status (enum: Draft, AwaitingApproval, Approved, Sent, PartReceived, Complete, Cancelled)
├── OrderDate (datetime)
├── RequiredDate (datetime)
├── ApprovedBy (FK → User, nullable)
├── ApprovedAt (datetime, nullable)
├── TotalValue (decimal)
├── DeliveryAddress (string)
├── Lines[]
│   ├── LineId (GUID)
│   ├── StockItemId (FK → StockItem)
│   ├── Description (string)
│   ├── Quantity (decimal)
│   ├── UnitPrice (decimal)
│   ├── QuantityReceived (decimal)
│   └── DueDate (datetime)
└── GoodsReceipts[]
    ├── GRNId (GUID)
    ├── ReceivedDate (datetime)
    ├── ReceivedBy (FK → User)
    ├── Lines[] (quantities per PO line)
    └── InspectionRequired (bool)

Supplier
├── SupplierId (GUID)
├── Name (string)
├── Code (string)
├── ContactName (string)
├── Email (string)
├── Phone (string)
├── Address (string)
├── PaymentTerms (string)
├── Currency (string)
├── IsApproved (bool)
├── ApprovalDate (datetime, nullable)
├── Rating (decimal, 1-5)
├── Categories[] (string[])
└── Notes (string)
```

---

### 2.5 Financial Overview & Job Costing

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Job Costing | Actual vs estimated cost per job (material, labour, overhead) | P0 |
| Material Cost Tracking | Cost per batch, FIFO/weighted average | P1 |
| Labour Cost Tracking | Time × rate per operation | P1 |
| Overhead Allocation | Configurable overhead rates per work centre | P1 |
| Margin Analysis | Profit/loss per job, customer, product family | P1 |
| Cost Variance Reporting | Highlight jobs exceeding budget thresholds | P1 |
| Invoice Generation | Generate invoices from completed/dispatched orders | P2 |
| Accounts Integration | Export to Sage/Xero/QuickBooks via API | P2 |

#### Data Model: Costing
```
JobCost
├── JobCostId (GUID)
├── OrderId (FK → Order)
├── OrderLineId (FK → OrderLine)
├── EstimatedMaterialCost (decimal)
├── ActualMaterialCost (decimal)
├── EstimatedLabourCost (decimal)
├── ActualLabourCost (decimal)
├── EstimatedOverheadCost (decimal)
├── ActualOverheadCost (decimal)
├── SubcontractCost (decimal)
├── TotalEstimated (computed)
├── TotalActual (computed)
├── Variance (computed: Actual - Estimated)
├── VariancePercentage (computed)
├── CostEntries[]
│   ├── EntryId (GUID)
│   ├── Type (enum: Material, Labour, Overhead, Subcontract, Sundry)
│   ├── Description (string)
│   ├── Quantity (decimal)
│   ├── UnitCost (decimal)
│   ├── TotalCost (decimal)
│   ├── OperationId (FK → Operation, nullable)
│   └── Timestamp (datetime)
└── Margin (computed: OrderLine.Value - TotalActual)
```

---

### 2.6 Customer Relationship Management (CRM)

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Customer Database | Full customer records with contacts and addresses | P0 |
| Communication Log | Track emails, calls, meetings per customer | P1 |
| Quotation History | All quotes with win/loss tracking | P1 |
| Customer Pricing | Customer-specific pricing agreements | P2 |
| Delivery Performance | On-time delivery metrics per customer | P1 |
| Credit Management | Credit limits, payment terms, overdue tracking | P2 |
| Customer Categories | Segment customers (aerospace, medical, automotive, etc.) | P1 |

#### Data Model: Customer
```
Customer
├── CustomerId (GUID)
├── AccountCode (string)
├── CompanyName (string)
├── TradingName (string, nullable)
├── Industry (enum: Aerospace, Medical, Automotive, Defence, Oil&Gas, General)
├── Contacts[]
│   ├── ContactId (GUID)
│   ├── FirstName (string)
│   ├── LastName (string)
│   ├── Role (string)
│   ├── Email (string)
│   ├── Phone (string)
│   └── IsPrimary (bool)
├── Addresses[]
│   ├── AddressId (GUID)
│   ├── Type (enum: Billing, Delivery, Registered)
│   ├── Lines (string[])
│   ├── City (string)
│   ├── PostCode (string)
│   └── Country (string)
├── PaymentTerms (int, days)
├── CreditLimit (decimal)
├── CurrentBalance (decimal)
├── CurrencyCode (string)
├── VATNumber (string, nullable)
├── IsActive (bool)
└── CreatedAt (datetime)
```

---

### 2.7 Document Management

#### Features
| Feature | Description | Priority |
|---------|-------------|----------|
| Drawing Register | Version-controlled engineering drawings (PDF, DXF, STEP) | P0 |
| Revision Control | Track drawing revisions with approval workflows | P0 |
| Document Linking | Attach documents to orders, parts, operations | P1 |
| Search & Filter | Full-text search across document metadata | P1 |
| Access Control | Role-based document access (e.g., restrict customer drawings) | P1 |
| Expiry Tracking | Alert when certifications/approvals expire | P2 |
| Template Library | Standard templates for reports, NCRs, certificates | P2 |

---

## 3. Business Rules

### Order Management Rules
1. Orders cannot move to "InProduction" without material availability confirmation
2. Priority changes on active orders trigger re-scheduling notification
3. Order amendments after production start require manager approval
4. Completed orders cannot be modified (create credit note / new order)
5. Auto-archive orders 12 months after final invoice

### Inventory Rules
1. Stock cannot go negative (block issue if insufficient)
2. Allocated stock is reserved and cannot be issued to other jobs
3. Batch/heat numbers are mandatory for aerospace and medical orders
4. Goods receipt triggers inspection workflow if supplier is "conditional approved"
5. Reorder alerts fire when available quantity ≤ reorder level

### Financial Rules
1. Purchase orders >£5,000 require director approval
2. Job cost variance >15% triggers automatic alert to production manager
3. Credit hold: no new orders if customer exceeds credit limit by >10%
4. All pricing in GBP unless customer-specific agreement

---

## 4. Reporting Requirements

| Report | Frequency | Audience |
|--------|-----------|----------|
| Order Book Summary | Real-time dashboard | Management |
| Works-in-Progress (WIP) | Daily | Production, Finance |
| On-Time Delivery Performance | Weekly | Management, Sales |
| Stock Valuation | Monthly | Finance |
| Supplier Performance | Monthly | Purchasing |
| Job Profitability | Per job completion | Management, Finance |
| Capacity Utilisation | Weekly | Production |
| Overdue Orders | Daily alert | Sales, Production |
| Material Usage vs Forecast | Monthly | Purchasing, Finance |
| Customer Revenue Analysis | Quarterly | Management, Sales |

---

## 5. Integration Points

| External System | Direction | Method | Data |
|----------------|-----------|--------|------|
| Accounting (Sage/Xero) | Bi-directional | REST API | Invoices, payments, nominal codes |
| CAD Systems | Inbound | File watcher / API | Drawing files, BOMs |
| Email (Outlook/SMTP) | Outbound | API | Order confirmations, dispatch notes |
| Shipping (Royal Mail/DPD) | Outbound | REST API | Labels, tracking numbers |
| Bank Feed | Inbound | Open Banking API | Payment reconciliation |

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](../product-specification.md)*
