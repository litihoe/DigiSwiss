# DigiSwiss - SaaS, Hybrid & Multi-Tenant Specification

## 1. Overview

This document provides the detailed specification for DigiSwiss's Software-as-a-Service (SaaS) delivery model, hybrid deployment capabilities, and multi-tenant architecture. It covers how the platform serves multiple manufacturing organisations simultaneously while maintaining strict data isolation, supporting on-premises requirements, and enabling scalable commercial operations.

---

## 2. Business Model

### 2.1 Revenue Model

| Revenue Stream | Description | Pricing Basis |
|---------------|-------------|---------------|
| Subscription | Monthly/annual platform access | Per tier (user + feature based) |
| Overage | Exceed included limits (storage, AI credits, users) | Per-unit above threshold |
| Professional Services | Implementation, training, custom integrations | Day rate / fixed price |
| Marketplace | Third-party integrations, custom modules | Revenue share (70/30) |
| Edge Hardware | Pre-configured edge nodes for hybrid customers | One-time purchase + support |
| Data Services | Advanced analytics, benchmarking (anonymised) | Add-on subscription |

### 2.2 Target Market Segments

| Segment | Size | Typical Tier | Deployment | Key Needs |
|---------|------|-------------|------------|-----------|
| Micro (1-10 employees) | Job shops, tool rooms | Starter | SaaS | Simple ERP + job tracking |
| Small (10-50 employees) | Subcontract manufacturers | Professional | SaaS | Full ERP + MES + basic SCADA |
| Medium (50-250 employees) | Specialist manufacturers | Professional/Enterprise | SaaS or Hybrid | All modules, multi-site |
| Large (250+ employees) | Aerospace/defence primes | Enterprise | Hybrid or Dedicated | Full platform, compliance, custom |

---

## 3. Multi-Tenant Architecture - Deep Dive

### 3.1 Tenant Context Propagation

```csharp
// Middleware: Tenant Resolution
public class TenantResolutionMiddleware
{
    public async Task InvokeAsync(HttpContext context, ITenantResolver resolver)
    {
        // Resolution chain (first match wins):
        // 1. JWT claim: "tenant_id"
        // 2. Subdomain: acme.digiswiss.io
        // 3. Custom domain: erp.acmemfg.co.uk (DNS lookup)
        // 4. Header: X-Tenant-Id (machine-to-machine only)
        
        var tenantContext = await resolver.ResolveAsync(context);
        
        if (tenantContext == null)
            throw new TenantNotFoundException();
            
        // Inject into DI scope
        context.RequestServices.GetRequiredService<ITenantContext>()
            .Set(tenantContext);
            
        await _next(context);
    }
}

// All DbContext queries automatically filtered
public class DigiSwissDbContext : DbContext
{
    private readonly ITenantContext _tenant;
    
    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Global query filter on all tenant-scoped entities
        builder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenant.TenantId);
        builder.Entity<WorkOrder>().HasQueryFilter(w => w.TenantId == _tenant.TenantId);
        builder.Entity<Machine>().HasQueryFilter(m => m.TenantId == _tenant.TenantId);
        // ... all tenant-scoped entities
    }
}
```

### 3.2 Database Isolation Strategies

| Strategy | Description | When to Use | Trade-offs |
|----------|-------------|------------|------------|
| **Shared DB + RLS** (Default) | All tenants in one database, Row-Level Security | Free, Starter, Professional | Cost-efficient, complex queries across tenants for platform analytics |
| **Shared DB + Separate Schema** | Each tenant gets own schema (erp_acme, erp_beta) | Professional (large) | Better logical isolation, slightly harder to manage |
| **Dedicated Database** | Separate SQL database per tenant | Enterprise | Full isolation, independent scaling, higher cost |
| **Dedicated Instance** | Separate SQL Server instance | Enterprise (regulated) | Maximum isolation, independent maintenance windows |

### 3.3 Row-Level Security Implementation

```sql
-- Security predicate function
CREATE FUNCTION dbo.fn_TenantSecurityPredicate(@TenantId UNIQUEIDENTIFIER)
RETURNS TABLE
WITH SCHEMABINDING
AS
    RETURN SELECT 1 AS result
    WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS UNIQUEIDENTIFIER)
    OR IS_MEMBER('platform_admin') = 1;  -- Platform admins can see all
GO

-- Apply to all tenant-scoped tables
CREATE SECURITY POLICY TenantIsolationPolicy
    ADD FILTER PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON erp.Orders,
    ADD FILTER PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON erp.Customers,
    ADD FILTER PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON mes.WorkOrders,
    ADD FILTER PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON scada.Machines,
    ADD FILTER PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON iot.Devices,
    ADD BLOCK PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON erp.Orders AFTER INSERT,
    ADD BLOCK PREDICATE dbo.fn_TenantSecurityPredicate(TenantId) ON erp.Orders BEFORE UPDATE
WITH (STATE = ON, SCHEMABINDING = ON);
GO

-- Set tenant context on every connection
-- (Called by EF Core connection interceptor)
EXEC sp_set_session_context @key = N'TenantId', @value = @CurrentTenantId;
```

### 3.4 Cache Isolation (Redis)

```
Key Pattern: {tenant_id}:{module}:{entity}:{id}

Examples:
  acme-mfg:scada:machine:haas01:status       → "Running"
  acme-mfg:scada:machine:haas01:spindle_rpm  → "8500"
  acme-mfg:mes:workorder:WO-001234:status    → "InProgress"
  acme-mfg:erp:dashboard:kpi_cache           → {JSON blob}
  
  beta-eng:scada:machine:dmg01:status        → "Idle"
  beta-eng:mes:workorder:WO-000567:status    → "Complete"

Eviction: Per-tenant memory limits enforced via Redis ACLs
TTL: Varies by data type (real-time: 5s, dashboard: 60s, config: 1hr)
```

### 3.5 SignalR Multi-Tenant Isolation

```csharp
// Hub groups are tenant-scoped
public class MachineHub : Hub
{
    public async Task SubscribeToMachine(string machineId)
    {
        var tenantId = Context.User.FindFirst("tenant_id").Value;
        
        // Group name includes tenant to prevent cross-tenant data leakage
        var group = $"tenant:{tenantId}:machine:{machineId}";
        await Groups.AddToGroupAsync(Context.ConnectionId, group);
    }
}

// When broadcasting machine updates
public async Task BroadcastMachineStatus(string tenantId, string machineId, MachineStatus status)
{
    var group = $"tenant:{tenantId}:machine:{machineId}";
    await _hubContext.Clients.Group(group).MachineStatusChanged(machineId, status);
}
```

---

## 4. Subscription & Billing Engine

### 4.1 Billing Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                      BILLING ENGINE                                 │
├───────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │ Metering │───▶│  Usage   │───▶│ Invoice  │───▶│ Payment  │   │
│  │ (Track   │    │ Aggreg-  │    │ Generator│    │ Gateway  │   │
│  │  usage)  │    │  ation   │    │          │    │ (Stripe) │   │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘   │
│       │                                                │          │
│       ▼                                                ▼          │
│  ┌──────────┐                                   ┌──────────┐    │
│  │ Quota    │                                   │ Dunning  │    │
│  │ Enforce- │                                   │ (Failed  │    │
│  │ ment     │                                   │ payment) │    │
│  └──────────┘                                   └──────────┘    │
└───────────────────────────────────────────────────────────────────┘
```

### 4.2 Metered Resources

| Resource | Included (per tier) | Overage Price | Measurement |
|----------|-------------------|---------------|-------------|
| Active Users | Tier-dependent | £15/user/mo | Unique logins in billing period |
| Storage (GB) | Tier-dependent | £0.50/GB/mo | Peak usage in period |
| API Calls | 100K-Unlimited | £0.001/call | Monthly aggregate |
| AI Credits | Tier-dependent | £0.05/credit | 1 credit = 1 document processed |
| IoT Messages | 1M-Unlimited | £0.0001/msg | Monthly aggregate |
| SignalR Connections | 10-Unlimited | £5/10 connections | Peak concurrent |
| Edge Nodes | 0-Unlimited | £50/node/mo | Active registered nodes |

### 4.3 Subscription Data Model

```sql
CREATE TABLE platform.Subscriptions (
    SubscriptionId      UNIQUEIDENTIFIER PRIMARY KEY,
    TenantId            UNIQUEIDENTIFIER NOT NULL REFERENCES platform.Tenants(TenantId),
    StripeSubscriptionId NVARCHAR(100),
    Tier                NVARCHAR(20) NOT NULL,
    Status              NVARCHAR(20) NOT NULL,  -- Active, PastDue, Cancelled, Paused
    BillingCycle        NVARCHAR(10) NOT NULL,  -- Monthly, Annual
    CurrentPeriodStart  DATETIME2 NOT NULL,
    CurrentPeriodEnd    DATETIME2 NOT NULL,
    MRR                 DECIMAL(10,2) NOT NULL,
    ARR                 AS (MRR * 12) PERSISTED,
    TrialEnd            DATETIME2,
    CancelledAt         DATETIME2,
    CancelReason        NVARCHAR(500),
    CreatedAt           DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE platform.UsageRecords (
    RecordId            BIGINT IDENTITY(1,1) PRIMARY KEY,
    TenantId            UNIQUEIDENTIFIER NOT NULL,
    ResourceType        NVARCHAR(30) NOT NULL,  -- Users, Storage, APICalls, AICredits, IoTMessages
    Period              DATE NOT NULL,
    Quantity            DECIMAL(18,4) NOT NULL,
    IncludedQuantity    DECIMAL(18,4) NOT NULL,
    OverageQuantity     AS (CASE WHEN Quantity > IncludedQuantity THEN Quantity - IncludedQuantity ELSE 0 END) PERSISTED,
    INDEX IX_Usage_Tenant_Period (TenantId, Period, ResourceType)
);
```

### 4.4 Quota Enforcement

| Enforcement Level | Behaviour | User Experience |
|------------------|-----------|-----------------|
| **Soft Limit (80%)** | Warning notification | Banner: "You've used 80% of your AI credits this month" |
| **Hard Limit (100%)** | Block new usage, allow existing | Cannot process new documents, existing jobs continue |
| **Grace Period (110%)** | 7-day grace with overage billing | "You've exceeded your plan. Upgrade or usage will be restricted." |
| **Suspended (unpaid)** | Read-only access | "Your account is suspended. Please update payment to continue." |
| **Terminated** | No access, data retained 30 days | "Account closed. Contact support within 30 days to recover data." |

---

## 5. Hybrid Architecture - Deep Dive

### 5.1 Edge-to-Cloud Synchronisation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SYNC PROTOCOL                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  EDGE → CLOUD (Upstream)                                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 1. Machine status changes (event-driven, immediate)           │   │
│  │ 2. OEE summaries (every 5 minutes)                           │   │
│  │ 3. Alarm events (immediate)                                   │   │
│  │ 4. Production completions (event-driven)                      │   │
│  │ 5. IoT aggregated telemetry (configurable: 30s-5min)         │   │
│  │ 6. Energy summaries (hourly)                                  │   │
│  │ 7. Edge health/diagnostics (every 60 seconds)                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  CLOUD → EDGE (Downstream)                                           │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ 1. Work order assignments (event-driven)                      │   │
│  │ 2. Schedule updates (event-driven)                            │   │
│  │ 3. Configuration changes (event-driven)                       │   │
│  │ 4. User/permission updates (every 5 minutes)                  │   │
│  │ 5. Firmware update packages (on-demand)                       │   │
│  │ 6. AI model updates (on-demand)                               │   │
│  │ 7. Alert rules / threshold changes (event-driven)             │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  CONFLICT RESOLUTION                                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Strategy: Last-Writer-Wins with vector clocks                 │   │
│  │ Priority: Edge wins for machine/production data               │   │
│  │           Cloud wins for business/planning data               │   │
│  │ Conflicts logged for manual review if both sides modified     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Offline Operation Capabilities

| Scenario | Duration Supported | What Works Offline | What Doesn't |
|----------|-------------------|-------------------|--------------|
| Momentary (< 5 min) | Automatic | Everything continues | Real-time cloud dashboards stale |
| Short (5 min - 1 hr) | Automatic | SCADA, IoT, MES operator actions | AI document processing, cross-site views |
| Extended (1-24 hrs) | With warning | All shop floor operations | New orders, purchasing, reports, mobile app |
| Prolonged (> 24 hrs) | Degraded mode | Machine monitoring, basic job execution | Most cloud features, user management |

### 5.3 Edge Node Management

```sql
CREATE TABLE platform.EdgeNodes (
    EdgeNodeId          UNIQUEIDENTIFIER PRIMARY KEY,
    TenantId            UNIQUEIDENTIFIER NOT NULL REFERENCES platform.Tenants(TenantId),
    NodeName            NVARCHAR(100) NOT NULL,
    SiteLocation        NVARCHAR(200) NOT NULL,
    Status              NVARCHAR(20) NOT NULL DEFAULT 'Provisioning',
    LastHeartbeat       DATETIME2,
    SoftwareVersion     NVARCHAR(30),
    OSVersion           NVARCHAR(100),
    
    -- Hardware Info
    CPUCores            INT,
    RAMTotal_GB         DECIMAL(5,1),
    DiskTotal_GB        DECIMAL(8,1),
    DiskUsed_GB         DECIMAL(8,1),
    
    -- Connectivity
    PublicIP            NVARCHAR(45),
    VPNStatus           NVARCHAR(20),
    LastSyncAt          DATETIME2,
    SyncLagSeconds      INT,
    QueuedMessages      INT,
    
    -- Connected Devices
    MachineCount        INT DEFAULT 0,
    IoTDeviceCount      INT DEFAULT 0,
    
    -- Security
    CertificateThumbprint NVARCHAR(64),
    CertificateExpiry   DATETIME2,
    LastSecurityScan    DATETIME2,
    
    ProvisionedAt       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

---

## 6. Platform Operations

### 6.1 Tenant Onboarding Automation

```
Step 1: Self-Service Signup (Web)
  ├── Email verification
  ├── Company details form
  ├── Industry selection (sets default config)
  └── Choose subscription tier

Step 2: Automated Provisioning (< 30 seconds)
  ├── Create tenant record
  ├── Provision identity (Azure AD B2C)
  ├── Create storage containers
  ├── Seed default data (roles, permissions, demo data option)
  ├── Configure DNS/subdomain
  ├── Apply feature flags for tier
  └── Send welcome email with login link

Step 3: Guided Onboarding (First Login)
  ├── Welcome wizard (5 steps)
  ├── Import existing data (CSV upload)
  ├── Configure company settings
  ├── Invite team members
  └── Connect first machine (guided)

Step 4: Ongoing Success
  ├── In-app tooltips and guides
  ├── Weekly usage summary email
  ├── Proactive support for stuck users
  └── Upgrade prompts when hitting limits
```

### 6.2 Multi-Region Deployment

| Region | Azure Region | Purpose | Tenants |
|--------|-------------|---------|---------|
| UK Primary | UK South (London) | Main production | All UK/EU tenants |
| UK DR | UK West (Cardiff) | Disaster recovery | Failover for UK South |
| EU | West Europe (Netherlands) | EU data residency | EU tenants (GDPR) |
| US (Future) | East US 2 | US expansion | US tenants |

### 6.3 Release Management

```
Release Pipeline:
  1. Development → Feature branches → PR review
  2. Staging → Full regression testing (shared tenant simulation)
  3. Canary → Deploy to 5% of Enterprise tenants (monitoring)
  4. Rolling → Deploy region by region (UK South → UK West → EU)
  5. Complete → All tenants on new version within 4 hours

Database Migrations:
  - Always backwards-compatible (expand-contract pattern)
  - New columns: nullable with defaults
  - Removed columns: mark deprecated, remove next release
  - Schema changes: applied before application deployment
  - Rollback: application code handles both old and new schema
```

### 6.4 SaaS Metrics & KPIs

| Metric | Target | Measurement |
|--------|--------|-------------|
| MRR (Monthly Recurring Revenue) | Growth >10% MoM | Stripe reporting |
| Churn Rate | < 5% monthly | Lost tenants / total tenants |
| Net Revenue Retention | > 110% | Expansion - Churn |
| Trial-to-Paid Conversion | > 15% | Conversions / trial signups |
| Time to Value | < 1 hour | First meaningful action after signup |
| Tenant Health Score | > 70 average | Composite: usage + engagement + NPS |
| Platform Uptime | 99.9% | Across all tenants |
| Mean Tenant Provisioning Time | < 30 seconds | Signup to usable system |
| Support Ticket Resolution | < 4 hours (Professional+) | Ticket close time |
| CAC Payback Period | < 6 months | CAC / Average MRR |

---

## 7. Security & Compliance

### 7.1 Tenant Data Protection

| Control | Implementation |
|---------|---------------|
| Data encryption at rest | Per-tenant keys in Azure Key Vault (AES-256) |
| Data encryption in transit | TLS 1.3 for all connections |
| Cross-tenant access prevention | RLS + application filter + integration tests |
| Tenant data export | Self-service GDPR export (full JSON/CSV) |
| Tenant data deletion | Complete purge within 30 days, certified |
| Backup encryption | Tenant-specific backup encryption keys |
| Key rotation | Automatic quarterly rotation, manual on-demand |
| Penetration testing | Annual third-party test + continuous scanning |

### 7.2 Tenant Security Testing

```
Automated Cross-Tenant Security Tests (run on every deploy):
  1. Attempt to access Tenant B data with Tenant A credentials → MUST FAIL
  2. Attempt to subscribe to Tenant B SignalR groups → MUST FAIL
  3. Attempt to read Tenant B blob storage → MUST FAIL
  4. Attempt to query Tenant B cache keys → MUST FAIL
  5. SQL injection with tenant_id manipulation → MUST FAIL
  6. JWT token manipulation (change tenant claim) → MUST FAIL
  7. Subdomain enumeration → Rate limited, no data leakage
  8. API access without tenant context → Rejected with 401/403
```

### 7.3 Compliance Automation

| Requirement | Automation |
|-------------|------------|
| GDPR Data Subject Requests | Self-service data export + deletion API |
| Right to be Forgotten | Automated cascade deletion across all stores |
| Data Processing Records | Auto-generated from audit log |
| Breach Notification | Automated detection + tenant notification pipeline |
| Annual Audits | Continuous compliance monitoring dashboards |
| Certifications | Azure compliance inheritance + custom controls |

---

## 8. Integration with Core Modules

### 8.1 Module Behaviour by Deployment Model

| Module | SaaS | Hybrid | Notes |
|--------|------|--------|-------|
| ERP | Cloud | Cloud | Always in cloud (multi-site collaboration) |
| MES | Cloud | Cloud + Edge (operator UI) | Edge caches work orders for offline |
| SCADA | Cloud | **Edge primary** + Cloud sync | Real-time data stays local, summaries to cloud |
| IoT | Cloud | **Edge primary** + Cloud sync | Raw telemetry local, aggregates to cloud |
| AI | Cloud | Cloud (with edge pre-processing) | GPU compute in cloud, classification on edge |
| Billing | Cloud | Cloud | Platform concern, always centralised |
| Admin | Cloud | Cloud + Edge admin panel | Edge node management from both |

### 8.2 Feature Flags by Tenant

```json
{
  "tenantId": "acme-mfg",
  "tier": "Professional",
  "features": {
    "modules": {
      "erp": true,
      "mes": true,
      "scada": true,
      "iot": true,
      "ai_document_intelligence": true,
      "ai_nlq": false,
      "ai_scheduling": false
    },
    "capabilities": {
      "custom_domain": true,
      "sso_saml": true,
      "api_webhooks": true,
      "white_labelling": false,
      "dedicated_support": false,
      "multi_site": true,
      "advanced_reporting": true,
      "predictive_maintenance": false
    },
    "limits": {
      "max_users": 50,
      "max_machines": 25,
      "max_iot_devices": 100,
      "storage_gb": 250,
      "ai_credits_monthly": 5000,
      "signalr_connections": 100,
      "api_rate_limit_per_min": 5000,
      "data_retention_years": 5
    }
  }
}
```

---

## 9. Business Rules

### Multi-Tenancy Rules
1. Tenant data is NEVER accessible by another tenant (zero-trust between tenants)
2. Platform admin access to tenant data requires explicit consent + audit log
3. Tenant deletion triggers 30-day grace period before permanent purge
4. Suspended tenants retain read-only access for 14 days, then full lock-out
5. Cross-tenant analytics only use anonymised, aggregated data

### Billing Rules
1. Downgrade takes effect at end of current billing period
2. Upgrade takes effect immediately with pro-rated charges
3. Failed payment retries: Day 1, Day 3, Day 7, then suspend
4. Annual subscriptions get 20% discount (2 months free)
5. Cancelled tenants can reactivate within 30 days with data intact

### Hybrid Rules
1. Edge nodes must sync at least once per 24 hours or trigger alert
2. Data older than configured retention (default: 72 hours) auto-purged from edge
3. Edge nodes auto-update during configured maintenance windows
4. Loss of cloud connectivity does not affect local SCADA/IoT operations
5. Security certificates on edge nodes must be renewed before expiry (30-day warning)

---

*Module Version: 1.0*  
*Parent Document: [Product Specification](product-specification.md)*
