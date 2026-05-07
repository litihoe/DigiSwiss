# DigiSwiss - API Specification

## 1. Overview

The DigiSwiss API follows RESTful principles with a microservices architecture. Each core module exposes its own API surface, unified behind an API Gateway. Real-time communication uses SignalR hubs.

**Base URL Pattern:** `https://api.digiswiss.io/v{version}/{service}/{resource}`

---

## 2. API Design Principles

| Principle | Implementation |
|-----------|---------------|
| Versioning | URL-based: `/v1/`, `/v2/` |
| Authentication | OAuth 2.0 + JWT Bearer tokens |
| Authorisation | RBAC claims in JWT, resource-level permissions |
| Pagination | Cursor-based (default 50, max 200) |
| Filtering | OData-style `$filter`, `$orderby`, `$select` |
| Error Format | RFC 7807 Problem Details |
| Rate Limiting | 1000 req/min per user, 5000 req/min per service account |
| Content Type | `application/json` (default), `application/pdf` for documents |
| HATEOAS | Links in responses for discoverability |
| Idempotency | `Idempotency-Key` header for POST/PUT operations |

---

## 3. Authentication & Authorisation

### 3.1 Auth Flow

```
┌────────┐    ┌──────────────┐    ┌────────────┐    ┌─────────┐
│ Client │───▶│ Identity     │───▶│  Token     │───▶│   API   │
│        │    │ Provider     │    │ (JWT)      │    │ Gateway │
│        │    │ (Azure AD B2C│    │            │    │         │
│        │    │  / Identity  │    │            │    │         │
│        │    │  Server)     │    │            │    │         │
└────────┘    └──────────────┘    └────────────┘    └─────────┘
```

### 3.2 JWT Claims

```json
{
  "sub": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "name": "John Smith",
  "email": "john.smith@company.com",
  "roles": ["ProductionManager", "QualityViewer"],
  "permissions": ["orders:read", "orders:write", "workorders:read", "machines:read"],
  "tenant_id": "tenant-001",
  "iss": "https://auth.digiswiss.io",
  "aud": "https://api.digiswiss.io",
  "exp": 1716500000,
  "iat": 1716496400
}
```

### 3.3 API Key (Machine-to-Machine)

```
Headers:
  X-API-Key: dgs_live_sk_a1b2c3d4e5f6...
  X-Device-Id: HAAS-01 (for IoT/SCADA devices)
```

---

## 4. Common Response Patterns

### 4.1 Success Response (Single Resource)
```json
{
  "data": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "type": "order",
    "attributes": { ... },
    "relationships": { ... }
  },
  "meta": {
    "timestamp": "2026-05-07T14:32:05Z",
    "version": "1"
  },
  "links": {
    "self": "/v1/erp/orders/a1b2c3d4..."
  }
}
```

### 4.2 Success Response (Collection)
```json
{
  "data": [ ... ],
  "meta": {
    "totalCount": 234,
    "pageSize": 50,
    "cursor": "eyJpZCI6MTAwfQ==",
    "hasMore": true
  },
  "links": {
    "self": "/v1/erp/orders?cursor=...",
    "next": "/v1/erp/orders?cursor=eyJpZCI6MTUwfQ==",
    "prev": "/v1/erp/orders?cursor=eyJpZCI6NTB9"
  }
}
```

### 4.3 Error Response (RFC 7807)
```json
{
  "type": "https://api.digiswiss.io/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "One or more fields failed validation.",
  "instance": "/v1/erp/orders",
  "traceId": "00-abc123-def456-01",
  "errors": [
    {
      "field": "requiredDate",
      "code": "DATE_IN_PAST",
      "message": "Required date must be in the future."
    }
  ]
}
```

---

## 5. ERP Service API

**Base:** `/v1/erp`

### 5.1 Orders

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/orders` | List orders (paginated, filterable) | `orders:read` |
| GET | `/orders/{id}` | Get order details with lines | `orders:read` |
| POST | `/orders` | Create new order | `orders:write` |
| PUT | `/orders/{id}` | Update order (draft/confirmed only) | `orders:write` |
| PATCH | `/orders/{id}/status` | Change order status | `orders:write` |
| DELETE | `/orders/{id}` | Cancel order (soft delete) | `orders:delete` |
| GET | `/orders/{id}/lines` | Get order lines | `orders:read` |
| POST | `/orders/{id}/lines` | Add order line | `orders:write` |
| GET | `/orders/{id}/history` | Get audit history | `orders:read` |
| GET | `/orders/{id}/costs` | Get job costing breakdown | `finance:read` |

#### Create Order Request
```json
POST /v1/erp/orders
{
  "customerId": "cust-guid-here",
  "reference": "PO-12345",
  "requiredDate": "2026-06-15T00:00:00Z",
  "priority": "High",
  "notes": "Urgent repeat order",
  "lines": [
    {
      "partNumber": "SHAFT-ASM-001",
      "drawingRef": "DWG-001234",
      "revision": "C",
      "quantity": 50,
      "unitPrice": 45.00,
      "material": "316L Stainless Steel",
      "deliveryDate": "2026-06-15T00:00:00Z"
    }
  ]
}
```

### 5.2 Inventory

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/inventory/stock-items` | List stock items | `inventory:read` |
| GET | `/inventory/stock-items/{id}` | Get item with batches & movements | `inventory:read` |
| POST | `/inventory/stock-items` | Create stock item | `inventory:write` |
| POST | `/inventory/movements` | Record stock movement | `inventory:write` |
| GET | `/inventory/movements` | List movements (filterable) | `inventory:read` |
| GET | `/inventory/low-stock` | Items at or below reorder level | `inventory:read` |
| POST | `/inventory/stocktake` | Record stocktake count | `inventory:write` |

### 5.3 Customers

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/customers` | List customers | `customers:read` |
| GET | `/customers/{id}` | Get customer with contacts | `customers:read` |
| POST | `/customers` | Create customer | `customers:write` |
| PUT | `/customers/{id}` | Update customer | `customers:write` |
| GET | `/customers/{id}/orders` | Customer order history | `orders:read` |
| GET | `/customers/{id}/performance` | Delivery/quality metrics | `customers:read` |

### 5.4 Purchasing

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/purchasing/purchase-orders` | List POs | `purchasing:read` |
| POST | `/purchasing/purchase-orders` | Create PO | `purchasing:write` |
| PATCH | `/purchasing/purchase-orders/{id}/approve` | Approve PO | `purchasing:approve` |
| POST | `/purchasing/goods-receipts` | Record goods receipt | `purchasing:write` |
| GET | `/purchasing/suppliers` | List suppliers | `purchasing:read` |
| GET | `/purchasing/suppliers/{id}/scorecard` | Supplier performance | `purchasing:read` |

---

## 6. MES Service API

**Base:** `/v1/mes`

### 6.1 Work Orders

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/work-orders` | List work orders | `workorders:read` |
| GET | `/work-orders/{id}` | Get with operations & traveller | `workorders:read` |
| POST | `/work-orders` | Create work order | `workorders:write` |
| PATCH | `/work-orders/{id}/status` | Update status | `workorders:write` |
| GET | `/work-orders/{id}/traceability` | Full trace chain | `quality:read` |

### 6.2 Operations (Shop Floor)

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| POST | `/operations/{id}/start` | Start operation (begin timer) | `shopfloor:execute` |
| POST | `/operations/{id}/pause` | Pause with reason | `shopfloor:execute` |
| POST | `/operations/{id}/resume` | Resume paused operation | `shopfloor:execute` |
| POST | `/operations/{id}/complete` | Complete with qty good/scrap | `shopfloor:execute` |
| POST | `/operations/{id}/hold` | Place on hold (NCR, etc.) | `quality:write` |
| GET | `/operations/queue/{workCentreId}` | Jobs queued at work centre | `shopfloor:read` |
| GET | `/operations/active` | Currently running operations | `shopfloor:read` |

#### Start Operation Request
```json
POST /v1/mes/operations/{id}/start
{
  "operatorId": "operator-guid",
  "machineId": "machine-guid",
  "materialBatchNumber": "HT-2026-0456",
  "notes": "Using fixture F-003"
}
```

### 6.3 Quality & Inspection

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/inspections/plans/{partNumber}` | Get inspection plan | `quality:read` |
| POST | `/inspections/records` | Submit inspection results | `quality:write` |
| GET | `/inspections/records/{id}` | Get inspection record | `quality:read` |
| POST | `/ncrs` | Raise non-conformance report | `quality:write` |
| GET | `/ncrs` | List NCRs (filterable) | `quality:read` |
| PATCH | `/ncrs/{id}/disposition` | Set NCR disposition | `quality:approve` |
| POST | `/ncrs/{id}/corrective-actions` | Add corrective action | `quality:write` |

### 6.4 Operator Actions

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| POST | `/operators/clock-in` | Clock in (QR scan) | `shopfloor:execute` |
| POST | `/operators/clock-out` | Clock out | `shopfloor:execute` |
| GET | `/operators/{id}/dashboard` | Operator's current jobs & queue | `shopfloor:read` |
| POST | `/operators/{id}/scan` | Process QR code scan | `shopfloor:execute` |

---

## 7. SCADA Service API

**Base:** `/v1/scada`

### 7.1 Machines

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/machines` | List all machines with current status | `machines:read` |
| GET | `/machines/{id}` | Machine details + live parameters | `machines:read` |
| GET | `/machines/{id}/status` | Current state only (lightweight) | `machines:read` |
| GET | `/machines/{id}/history` | Historical status changes | `machines:read` |
| GET | `/machines/{id}/oee` | OEE data (day/week/month) | `machines:read` |
| GET | `/machines/{id}/tags` | All configured data points | `scada:admin` |
| PUT | `/machines/{id}/tags/{tagId}` | Update tag configuration | `scada:admin` |

### 7.2 Alarms

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/alarms/active` | Currently active alarms | `alarms:read` |
| GET | `/alarms/history` | Historical alarms (filterable) | `alarms:read` |
| POST | `/alarms/{id}/acknowledge` | Acknowledge alarm | `alarms:write` |
| GET | `/alarms/definitions` | Alarm configurations | `scada:admin` |
| POST | `/alarms/definitions` | Create alarm definition | `scada:admin` |

### 7.3 Data & Trends

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/data/live/{tagId}` | Current value of a tag | `machines:read` |
| GET | `/data/history/{tagId}` | Historical values (time range) | `machines:read` |
| GET | `/data/aggregate/{tagId}` | Aggregated data (min/max/avg) | `machines:read` |
| POST | `/data/query` | Multi-tag query (batch) | `machines:read` |

#### Historical Data Query
```json
POST /v1/scada/data/query
{
  "tags": ["HAAS01.Spindle.Speed", "HAAS01.Spindle.Load"],
  "startTime": "2026-05-07T06:00:00Z",
  "endTime": "2026-05-07T14:00:00Z",
  "resolution": "1m",
  "aggregation": "avg"
}
```

---

## 8. IoT Service API

**Base:** `/v1/iot`

### 8.1 Devices

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/devices` | List all IoT devices | `iot:read` |
| GET | `/devices/{id}` | Device details + last telemetry | `iot:read` |
| POST | `/devices` | Register new device | `iot:admin` |
| PUT | `/devices/{id}` | Update device config | `iot:admin` |
| PATCH | `/devices/{id}/status` | Enable/disable device | `iot:admin` |
| DELETE | `/devices/{id}` | Decommission device | `iot:admin` |
| GET | `/devices/{id}/telemetry` | Recent telemetry data | `iot:read` |
| POST | `/devices/{id}/command` | Send command to device | `iot:admin` |

### 8.2 Fleet Management

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/fleet/summary` | Fleet health overview | `iot:read` |
| GET | `/fleet/offline` | Disconnected devices | `iot:read` |
| GET | `/fleet/firmware` | Firmware version summary | `iot:admin` |
| POST | `/fleet/firmware/update` | Initiate OTA update | `iot:admin` |
| GET | `/fleet/firmware/update/{id}` | Update rollout status | `iot:admin` |

### 8.3 Energy

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/energy/current` | Real-time power consumption | `energy:read` |
| GET | `/energy/summary` | Energy summary (day/week/month) | `energy:read` |
| GET | `/energy/by-machine/{id}` | Per-machine energy data | `energy:read` |
| GET | `/energy/cost` | Cost analysis and projections | `energy:read` |

---

## 9. AI & Automation Service API

**Base:** `/v1/ai`

### 9.1 Document Intelligence

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| POST | `/documents/upload` | Upload document for processing | `documents:write` |
| GET | `/documents/{id}/status` | Processing status | `documents:read` |
| GET | `/documents/{id}/result` | Extraction results | `documents:read` |
| POST | `/documents/{id}/approve` | Approve extracted data | `documents:approve` |
| POST | `/documents/{id}/correct` | Submit corrections | `documents:write` |
| GET | `/documents/review-queue` | Documents awaiting review | `documents:approve` |

#### Upload Document
```json
POST /v1/ai/documents/upload
Content-Type: multipart/form-data

file: (binary)
metadata: {
  "source": "email",
  "expectedType": "PurchaseOrder",
  "associatedCustomer": "customer-guid",
  "priority": "normal"
}
```

### 9.2 Automation Rules

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| GET | `/automation/rules` | List automation rules | `automation:read` |
| GET | `/automation/rules/{id}` | Rule details + execution history | `automation:read` |
| POST | `/automation/rules` | Create automation rule | `automation:write` |
| PUT | `/automation/rules/{id}` | Update rule | `automation:write` |
| PATCH | `/automation/rules/{id}/toggle` | Enable/disable rule | `automation:write` |
| GET | `/automation/rules/{id}/logs` | Execution logs | `automation:read` |
| POST | `/automation/rules/{id}/test` | Test rule (dry run) | `automation:write` |

### 9.3 Natural Language Query

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| POST | `/nlq/query` | Submit natural language question | `nlq:use` |
| GET | `/nlq/history` | User's query history | `nlq:use` |
| POST | `/nlq/feedback` | Rate query response quality | `nlq:use` |

#### NLQ Request
```json
POST /v1/ai/nlq/query
{
  "question": "What's the current status of order WO-2026-001234?",
  "context": {
    "currentPage": "orders",
    "recentEntities": ["WO-2026-001234"]
  }
}
```

#### NLQ Response
```json
{
  "answer": "Work order WO-2026-001234 is currently In Production. Operation 20 (CNC Milling) is running on HAAS-01, 78% complete. Expected completion: today at 16:30.",
  "confidence": 0.95,
  "sources": [
    { "type": "WorkOrder", "id": "wo-guid", "field": "Status" },
    { "type": "Operation", "id": "op-guid", "field": "Progress" }
  ],
  "suggestedActions": [
    { "label": "View Work Order", "url": "/mes/work-orders/wo-guid" },
    { "label": "View Machine", "url": "/scada/machines/haas-01-guid" }
  ],
  "visualisation": null
}
```

### 9.4 Scheduling

| Method | Endpoint | Description | Permissions |
|--------|----------|-------------|-------------|
| POST | `/scheduling/optimise` | Request schedule optimisation | `scheduling:write` |
| GET | `/scheduling/optimise/{id}` | Get optimisation result | `scheduling:read` |
| POST | `/scheduling/optimise/{id}/accept` | Accept proposed schedule | `scheduling:write` |
| GET | `/scheduling/current` | Current active schedule | `scheduling:read` |
| POST | `/scheduling/what-if` | Run what-if scenario | `scheduling:read` |

---

## 10. Real-Time API (SignalR Hubs)

### 10.1 Hub Endpoints

| Hub | URL | Purpose |
|-----|-----|---------|
| MachineHub | `/hubs/machines` | Live machine status & parameters |
| ProductionHub | `/hubs/production` | Job status changes, completions |
| AlertHub | `/hubs/alerts` | Real-time alarms and notifications |
| DashboardHub | `/hubs/dashboard` | KPI updates, OEE, throughput |

### 10.2 MachineHub Events

```csharp
// Server → Client
interface IMachineHubClient
{
    Task MachineStatusChanged(string machineId, MachineStatus newStatus);
    Task ParameterUpdated(string machineId, string tagName, double value, DateTime timestamp);
    Task AlarmRaised(AlarmEvent alarm);
    Task AlarmCleared(string alarmId);
    Task CycleCompleted(string machineId, int partCount, TimeSpan cycleTime);
}

// Client → Server
interface IMachineHub
{
    Task SubscribeToMachine(string machineId);
    Task UnsubscribeFromMachine(string machineId);
    Task SubscribeToAll();
    Task AcknowledgeAlarm(string alarmId, string notes);
}
```

### 10.3 ProductionHub Events

```csharp
// Server → Client
interface IProductionHubClient
{
    Task JobStarted(string workOrderId, string operationId, string machineId);
    Task JobCompleted(string workOrderId, string operationId, int goodQty, int scrapQty);
    Task JobPaused(string workOrderId, string operationId, string reason);
    Task QueueUpdated(string workCentreId, WorkOrderSummary[] queue);
    Task OEEUpdated(string machineId, decimal oee, decimal availability, decimal performance, decimal quality);
}
```

---

## 11. Webhook API

### 11.1 Webhook Registration

```json
POST /v1/webhooks
{
  "url": "https://external-system.com/callbacks/digiswiss",
  "events": ["order.created", "order.dispatched", "ncr.raised", "machine.fault"],
  "secret": "whsec_abc123...",
  "isActive": true
}
```

### 11.2 Webhook Payload

```json
{
  "id": "evt_a1b2c3d4",
  "type": "order.dispatched",
  "timestamp": "2026-05-07T14:32:05Z",
  "data": {
    "orderId": "order-guid",
    "orderNumber": "WO-2026-001234",
    "customerId": "customer-guid",
    "dispatchDate": "2026-05-07",
    "trackingNumber": "RM1234567890GB"
  },
  "signature": "sha256=abc123..."
}
```

### 11.3 Available Events

| Category | Events |
|----------|--------|
| Orders | `order.created`, `order.confirmed`, `order.dispatched`, `order.completed` |
| Production | `workorder.started`, `workorder.completed`, `operation.completed` |
| Quality | `ncr.raised`, `ncr.dispositioned`, `inspection.failed` |
| Machines | `machine.fault`, `machine.offline`, `machine.maintenance_due` |
| Inventory | `stock.low`, `stock.received`, `stock.movement` |
| Documents | `document.processed`, `document.review_required` |

---

## 12. API Gateway Configuration

### 12.1 Routing

```yaml
routes:
  - path: /v1/erp/**
    service: erp-service
    port: 5001
  - path: /v1/mes/**
    service: mes-service
    port: 5002
  - path: /v1/scada/**
    service: scada-service
    port: 5003
  - path: /v1/iot/**
    service: iot-service
    port: 5004
  - path: /v1/ai/**
    service: ai-service
    port: 5005
  - path: /hubs/**
    service: realtime-service
    port: 5010
    websocket: true
```

### 12.2 Rate Limiting Tiers

| Tier | Requests/Min | Burst | Use Case |
|------|-------------|-------|----------|
| Free/Trial | 60 | 10 | Evaluation |
| Standard User | 1,000 | 50 | Normal UI usage |
| Power User | 5,000 | 100 | Integration/reporting |
| Service Account | 10,000 | 200 | Machine-to-machine |
| IoT Device | 120 | 5 | Telemetry submission |

---

## 13. Health & Monitoring Endpoints

| Endpoint | Purpose | Auth Required |
|----------|---------|---------------|
| GET `/health` | Basic alive check | No |
| GET `/health/ready` | Readiness (DB, dependencies) | No |
| GET `/health/detailed` | Full dependency health | Admin |
| GET `/metrics` | Prometheus metrics | Internal |
| GET `/info` | Version, build, environment | Admin |

---

*Document Version: 1.0*  
*Parent Document: [Product Specification](product-specification.md)*
