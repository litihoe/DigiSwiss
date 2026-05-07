# DigiSwiss - Product Specification

## 1. Executive Summary

**Product Name:** DigiSwiss  
**Domain:** Precision Engineering & Manufacturing  
**Platform Type:** Integrated Digital Manufacturing Platform (ERP + MES + SCADA + IoT)  
**Delivery Model:** SaaS with Hybrid deployment option  
**Tenancy:** Multi-Tenant architecture with tenant isolation  
**Target Platforms:** Web, Desktop (Windows), iOS, Android  
**Timeline:** 2-month initial delivery (MVP to Production)  
**Current State:** Prototype exists - requires stabilisation, extension, and scaling

---

## 2. Product Vision

DigiSwiss is an integrated digital manufacturing platform designed for precision engineering operations. It unifies Enterprise Resource Planning (ERP), Manufacturing Execution Systems (MES), Supervisory Control and Data Acquisition (SCADA), and Internet of Things (IoT) capabilities into a single, cohesive platform — enabling real-time operational visibility, automated workflows, and data-driven decision making.

Delivered as a **SaaS platform** with **hybrid deployment capabilities**, DigiSwiss serves multiple manufacturing organisations from a shared, scalable infrastructure while maintaining strict data isolation through a **multi-tenant architecture**. Customers who require on-premises data sovereignty (e.g., defence, aerospace) can deploy the edge/SCADA components locally while connecting to the cloud platform for AI, analytics, and collaboration.

---

## 3. Technology Stack

### Backend
| Layer | Technology |
|-------|-----------|
| Runtime | .NET 8 (LTS) |
| API Framework | ASP.NET Core Web API |
| Real-Time | SignalR |
| ORM | Entity Framework Core |
| Database | SQL Server |
| Architecture | Microservices / Event-Driven |
| Messaging | Azure Service Bus / RabbitMQ |
| Authentication | ASP.NET Identity + JWT / OAuth 2.0 |

### Frontend
| Platform | Technology |
|----------|-----------|
| Web | Blazor WebAssembly / Server |
| Desktop | .NET MAUI (Windows) |
| iOS | .NET MAUI |
| Android | .NET MAUI |

### AI & Automation
| Capability | Technology |
|-----------|-----------|
| Document Intelligence | Azure AI Document Intelligence / GPT-4 |
| OCR Pipeline | Custom AI-powered extraction |
| Automation Engine | SQL-triggered workflows + event-driven alerts |
| Predictive Analytics | ML.NET / Azure ML |

### Infrastructure
| Concern | Technology |
|---------|-----------|
| Cloud | Azure (primary) |
| CI/CD | GitHub Actions / Azure DevOps |
| Containerisation | Docker + Kubernetes |
| Monitoring | Application Insights / Serilog |
| API Gateway | Azure API Management / Ocelot |

---

## 4. Core Modules

### 4.1 ERP Module - Enterprise Resource Planning

**Purpose:** Manage business operations, resources, and financials.

| Feature | Description | Priority |
|---------|-------------|----------|
| Order Management | Create, track, and manage customer orders through production lifecycle | P0 |
| Inventory Management | Real-time stock levels, reorder points, material traceability | P0 |
| Production Planning | Schedule jobs, allocate resources, manage capacity | P0 |
| Purchasing & Procurement | Supplier management, purchase orders, goods receipt | P1 |
| Financial Overview | Cost tracking, job costing, margin analysis | P1 |
| Customer Management (CRM) | Customer records, communications, quotations | P1 |
| Document Management | Drawing revisions, specifications, quality documents | P2 |
| Reporting & Analytics | Customisable dashboards, KPI tracking, export capabilities | P1 |

### 4.2 MES Module - Manufacturing Execution System

**Purpose:** Bridge the gap between planning and shop floor execution.

| Feature | Description | Priority |
|---------|-------------|----------|
| Work Order Execution | Digital work orders with step-by-step instructions | P0 |
| Job Tracking | Real-time status of every job on the shop floor | P0 |
| Machine Allocation | Assign jobs to machines, track utilisation | P0 |
| Quality Control | In-process inspection, SPC, non-conformance management | P0 |
| Operator Interface | Tablet/kiosk-friendly UI for shop floor operators | P1 |
| Traceability | Full material and process traceability per part/batch | P1 |
| Tool Management | Tool life tracking, calibration schedules | P2 |
| Digital Travellers | Paperless job cards with sign-off workflows | P1 |

### 4.3 SCADA Module - Supervisory Control & Data Acquisition

**Purpose:** Monitor and control manufacturing equipment in real-time.

| Feature | Description | Priority |
|---------|-------------|----------|
| Real-Time Dashboards | Live machine status, utilisation, OEE | P0 |
| Alarm Management | Configurable alerts for machine faults, thresholds | P0 |
| Historical Data | Time-series storage for trend analysis | P1 |
| Machine Connectivity | OPC-UA, Modbus, MTConnect protocol support | P1 |
| Visual Process Maps | Graphical representation of production lines | P2 |
| Remote Monitoring | Mobile access to machine status and alerts | P1 |

### 4.4 IoT Module - Internet of Things

**Purpose:** Collect, process, and act on sensor/machine data.

| Feature | Description | Priority |
|---------|-------------|----------|
| Device Management | Register, configure, and monitor IoT devices | P0 |
| Data Ingestion | High-throughput sensor data collection | P0 |
| Edge Processing | Local data filtering and aggregation | P1 |
| Predictive Maintenance | AI-driven failure prediction from sensor patterns | P2 |
| Environmental Monitoring | Temperature, humidity, vibration tracking | P1 |
| Energy Monitoring | Power consumption per machine/line | P2 |

### 4.5 AI & Automation Module

**Purpose:** Reduce manual effort and enable intelligent operations.

| Feature | Description | Priority |
|---------|-------------|----------|
| Document Intelligence | AI-powered extraction from drawings, POs, specs (70% reduction target) | P0 |
| Automated Alerts | SQL-triggered operational notifications | P0 |
| QR-Based Operations | Onboarding, asset tracking, job allocation (40-60% time reduction) | P1 |
| Intelligent Scheduling | AI-assisted production scheduling optimisation | P2 |
| Anomaly Detection | Automatic identification of process deviations | P2 |
| Natural Language Queries | GPT-powered data querying for non-technical users | P3 |

---

## 5. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                               │
├──────────┬──────────┬──────────┬──────────┬────────────────────┤
│  Blazor  │  MAUI    │  MAUI    │  MAUI    │  Operator Kiosks   │
│  Web App │  Windows │  iOS     │  Android │  (Touch UI)        │
└────┬─────┴────┬─────┴────┬─────┴────┬─────┴────────┬───────────┘
     │          │          │          │              │
     └──────────┴──────────┴──────────┴──────────────┘
                           │
                    ┌──────┴──────┐
                    │ API Gateway │
                    └──────┬──────┘
                           │
     ┌─────────────────────┼─────────────────────────┐
     │              SERVICE LAYER                      │
     ├──────────┬──────────┬──────────┬──────────────┤
     │   ERP    │   MES    │  SCADA   │     IoT      │
     │ Service  │ Service  │ Service  │   Service    │
     └────┬─────┴────┬─────┴────┬─────┴──────┬───────┘
          │          │          │            │
     ┌────┴──────────┴──────────┴────────────┴────────┐
     │              MESSAGE BUS                         │
     │        (Azure Service Bus / RabbitMQ)           │
     └────┬──────────┬──────────┬────────────┬────────┘
          │          │          │            │
     ┌────┴────┐ ┌──┴───┐ ┌───┴───┐ ┌─────┴─────┐
     │SQL Server│ │Redis │ │Time-  │ │  Blob     │
     │(Primary) │ │Cache │ │Series │ │  Storage  │
     └─────────┘ └──────┘ │  DB   │ └───────────┘
                           └───────┘
```

---

## 6. SaaS, Hybrid & Multi-Tenant Architecture

### 6.1 Deployment Models

DigiSwiss supports three deployment models to accommodate different customer requirements:

| Model | Description | Target Customer | Data Location |
|-------|-------------|-----------------|---------------|
| **Full SaaS** | Everything hosted in DigiSwiss cloud (Azure UK) | SME manufacturers, general engineering | Cloud (Azure UK South/West) |
| **Hybrid** | Cloud platform + on-premises edge nodes for SCADA/IoT | Aerospace, defence, regulated industries | Sensitive data on-prem, analytics in cloud |
| **Dedicated** | Single-tenant cloud instance (isolated infrastructure) | Enterprise customers, high-compliance requirements | Dedicated Azure subscription |

### 6.2 SaaS Platform Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DIGISWISS SaaS PLATFORM                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     SHARED INFRASTRUCTURE                            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │API Gateway│  │  CDN     │  │  DNS /   │  │  Identity        │  │   │
│  │  │(per-tenant│  │ (Static  │  │  Custom  │  │  Provider        │  │   │
│  │  │ routing)  │  │  Assets) │  │  Domains)│  │  (Azure AD B2C)  │  │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     APPLICATION TIER (Shared)                        │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │   │
│  │  │  ERP    │ │  MES    │ │ SCADA   │ │  IoT    │ │   AI    │    │   │
│  │  │ Service │ │ Service │ │ Service │ │ Service │ │ Service │    │   │
│  │  │(tenant- │ │(tenant- │ │(tenant- │ │(tenant- │ │(tenant- │    │   │
│  │  │ aware)  │ │ aware)  │ │ aware)  │ │ aware)  │ │ aware)  │    │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      DATA TIER                                       │   │
│  │                                                                       │   │
│  │  STRATEGY: Shared Database, Separate Schemas (with RLS)              │   │
│  │                                                                       │   │
│  │  ┌───────────────────────────────────────────────────────────┐      │   │
│  │  │                  SQL Server (Elastic Pool)                  │      │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │      │   │
│  │  │  │ Tenant A │  │ Tenant B │  │ Tenant C │  │Tenant N │ │      │   │
│  │  │  │  (RLS)   │  │  (RLS)   │  │  (RLS)   │  │  (RLS)  │ │      │   │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └─────────┘ │      │   │
│  │  └───────────────────────────────────────────────────────────┘      │   │
│  │                                                                       │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐    │   │
│  │  │ Redis Cache  │  │ Time-Series │  │ Blob Storage (per tenant│    │   │
│  │  │(tenant-keyed)│  │(tenant-tagged)│ │  container isolation)  │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Hybrid Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CUSTOMER PREMISES (On-Prem)                            │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      EDGE NODE (Docker/K3s)                          │   │
│  │                                                                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │  SCADA   │  │   IoT    │  │  Local   │  │  Store & Forward │  │   │
│  │  │  Agent   │  │  Gateway │  │  Cache   │  │  Queue           │  │   │
│  │  │(OPC-UA,  │  │ (MQTT,   │  │ (Redis/  │  │ (RabbitMQ local) │  │   │
│  │  │ Modbus)  │  │  Sensor) │  │  SQLite) │  │                  │  │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  │                                                                       │   │
│  │  ┌──────────────────────────────────────────────────────────────┐   │   │
│  │  │  LOCAL DATA STORE (sensitive/real-time data stays on-prem)    │   │   │
│  │  │  • Raw machine data (high-frequency)                          │   │   │
│  │  │  • Proprietary process parameters                             │   │   │
│  │  │  • Defence/classified job details (if applicable)             │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│         │  Secure Tunnel (WireGuard / Azure Arc / VPN)                      │
│         │  Only aggregated / anonymised data crosses boundary               │
│         ▼                                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DIGISWISS CLOUD (Azure UK)                             │
│                                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   ERP    │  │    AI    │  │Analytics │  │ Billing  │  │  Admin   │   │
│  │ (Orders, │  │(Doc Intel│  │(Dashboards│  │(Subscript│  │ (Tenant  │   │
│  │ Planning)│  │ NLQ, ML) │  │ Reports) │  │  ions)   │  │  Mgmt)   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 Multi-Tenancy Strategy

#### Isolation Model

| Layer | Isolation Strategy | Implementation |
|-------|-------------------|----------------|
| **Identity** | Separate per tenant | Azure AD B2C with custom policies per tenant |
| **Application** | Shared services, tenant context injected | `TenantId` in JWT claims, middleware resolves |
| **Database** | Shared database with Row-Level Security | SQL Server RLS policies on all tenant-scoped tables |
| **Cache** | Tenant-prefixed keys | Redis key pattern: `{tenantId}:{entity}:{id}` |
| **Blob Storage** | Separate containers per tenant | `tenant-{id}-documents`, `tenant-{id}-backups` |
| **Time-Series** | Tenant-tagged data points | Tag: `tenant_id` on all IoT telemetry |
| **Message Bus** | Tenant-specific topics/subscriptions | Topic filter: `TenantId = '{id}'` |
| **Encryption** | Per-tenant encryption keys | Azure Key Vault, tenant-specific key for data-at-rest |
| **Logging** | Tenant-tagged logs | Structured logging with `TenantId` field |
| **Network** | Shared ingress, isolated egress | Tenant-specific VPN tunnels for hybrid |

#### Tenant Resolution Flow

```
Request → DNS/URL → API Gateway → Tenant Resolver → Service (with TenantContext)

Resolution Methods:
1. Subdomain:    acme.digiswiss.io        → TenantId = "acme"
2. Custom domain: erp.acmemfg.co.uk       → DNS CNAME → mapped to TenantId
3. Header:       X-Tenant-Id: acme        → For API/machine-to-machine
4. JWT Claim:    tenant_id: "guid-here"   → For authenticated user requests
```

#### Tenant Data Model

```sql
CREATE TABLE platform.Tenants (
    TenantId            UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantCode          NVARCHAR(50) NOT NULL UNIQUE,     -- URL-safe slug (e.g., "acme-mfg")
    CompanyName         NVARCHAR(200) NOT NULL,
    DisplayName         NVARCHAR(100) NOT NULL,
    CustomDomain        NVARCHAR(200),                    -- e.g., "erp.acmemfg.co.uk"
    
    -- Subscription & Billing
    SubscriptionTier    NVARCHAR(20) NOT NULL,            -- Free, Starter, Professional, Enterprise
    SubscriptionStatus  NVARCHAR(20) NOT NULL DEFAULT 'Active',
    BillingCycle        NVARCHAR(10) NOT NULL DEFAULT 'Monthly',
    TrialEndsAt         DATETIME2,
    MRR                 DECIMAL(10,2),                    -- Monthly Recurring Revenue
    
    -- Configuration
    DeploymentModel     NVARCHAR(20) NOT NULL DEFAULT 'SaaS',  -- SaaS, Hybrid, Dedicated
    DataRegion          NVARCHAR(20) NOT NULL DEFAULT 'UK-South',
    MaxUsers            INT NOT NULL DEFAULT 10,
    MaxMachines         INT NOT NULL DEFAULT 5,
    MaxIoTDevices       INT NOT NULL DEFAULT 20,
    StorageQuota_GB     INT NOT NULL DEFAULT 50,
    AICreditsMonthly    INT NOT NULL DEFAULT 1000,
    
    -- Features (feature flags per tenant)
    EnabledModules      NVARCHAR(MAX) NOT NULL,           -- JSON: ["ERP","MES","SCADA","IoT","AI"]
    FeatureFlags        NVARCHAR(MAX),                    -- JSON: {"advancedScheduling": true, ...}
    
    -- Branding
    LogoUrl             NVARCHAR(500),
    PrimaryColour       NVARCHAR(7),                      -- Hex: "#1B4F72"
    ThemeMode           NVARCHAR(10) DEFAULT 'Light',
    
    -- Hybrid Configuration
    EdgeNodeCount       INT DEFAULT 0,
    VPNEndpoint         NVARCHAR(200),
    EdgeSyncInterval    INT DEFAULT 60,                   -- seconds
    
    -- Metadata
    Industry            NVARCHAR(50),
    Country             NVARCHAR(50) DEFAULT 'United Kingdom',
    Timezone            NVARCHAR(50) DEFAULT 'Europe/London',
    CurrencyCode        NVARCHAR(3) DEFAULT 'GBP',
    IsActive            BIT NOT NULL DEFAULT 1,
    CreatedAt           DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
    SuspendedAt         DATETIME2,
    DeletedAt           DATETIME2
);

CREATE TABLE platform.TenantSubscriptionHistory (
    HistoryId           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    TenantId            UNIQUEIDENTIFIER NOT NULL REFERENCES platform.Tenants(TenantId),
    PreviousTier        NVARCHAR(20),
    NewTier             NVARCHAR(20) NOT NULL,
    ChangeReason        NVARCHAR(100),
    EffectiveFrom       DATETIME2 NOT NULL,
    ChangedBy           NVARCHAR(200),
    CreatedAt           DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### 6.5 SaaS Subscription Tiers

| Feature | Free / Trial | Starter | Professional | Enterprise |
|---------|-------------|---------|--------------|------------|
| **Price** | £0 (14 days) | £299/mo | £799/mo | Custom |
| **Users** | 3 | 10 | 50 | Unlimited |
| **Machines** | 2 | 5 | 25 | Unlimited |
| **IoT Devices** | 5 | 20 | 100 | Unlimited |
| **Storage** | 5 GB | 50 GB | 250 GB | Custom |
| **AI Credits/mo** | 100 | 1,000 | 5,000 | Unlimited |
| **Modules** | ERP only | ERP + MES | All | All |
| **SCADA** | - | Basic | Full | Full |
| **IoT** | - | - | Full | Full |
| **AI Document Intel** | - | Basic | Full | Full + custom models |
| **Real-Time (SignalR)** | 5s refresh | 2s refresh | 1s refresh | Sub-second |
| **Data Retention** | 30 days | 1 year | 5 years | Custom |
| **Support** | Community | Email (48h) | Priority (4h) | Dedicated (1h SLA) |
| **Deployment** | SaaS only | SaaS only | SaaS or Hybrid | Any (incl. Dedicated) |
| **Custom Domain** | - | - | Yes | Yes |
| **SSO (SAML/OIDC)** | - | - | Yes | Yes |
| **API Access** | Read-only | Full | Full | Full + webhooks |
| **White-Labelling** | - | - | Logo + colours | Full rebrand |
| **SLA** | Best effort | 99.5% | 99.9% | 99.95% |
| **Compliance Certs** | - | - | ISO 27001 | ISO + custom |

### 6.6 Tenant Lifecycle

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Sign Up │───▶│ Trial    │───▶│  Active  │───▶│ Upgrade/ │───▶│  Churn   │
│  (Self-  │    │ (14 days │    │  (Paying │    │ Downgrade│    │  Mgmt    │
│  Service)│    │  free)   │    │  Sub)    │    │          │    │          │
└──────────┘    └────┬─────┘    └────┬─────┘    └──────────┘    └────┬─────┘
                     │               │                                │
                     │ (no convert)  │ (payment fail)                 │
                     ▼               ▼                                ▼
              ┌──────────┐    ┌──────────┐                     ┌──────────┐
              │  Expired │    │ Suspended│                     │  Delete  │
              │  (read-  │    │  (grace  │                     │  (GDPR   │
              │  only)   │    │  period) │                     │  wipe)   │
              └──────────┘    └──────────┘                     └──────────┘
```

#### Tenant Provisioning Steps (Automated)

| Step | Action | Duration |
|------|--------|----------|
| 1 | Create tenant record in platform DB | < 1 second |
| 2 | Provision Azure AD B2C directory/policies | 5-10 seconds |
| 3 | Create blob storage container | < 1 second |
| 4 | Seed database with tenant default data (roles, permissions, config) | 2-3 seconds |
| 5 | Configure DNS/subdomain routing | < 5 seconds |
| 6 | Send welcome email with onboarding guide | Async |
| 7 | Enable feature flags based on subscription tier | < 1 second |
| **Total** | **Self-service signup to usable system** | **< 30 seconds** |

### 6.7 Data Isolation & Compliance

#### Data Sovereignty Controls

| Requirement | Implementation |
|-------------|---------------|
| Data residency | All tenant data stored in specified Azure region (default: UK South) |
| Cross-tenant isolation | Row-Level Security + application-level TenantId filtering |
| Encryption isolation | Per-tenant encryption keys in Azure Key Vault |
| Backup isolation | Tenant-specific backup schedules, separate restore capability |
| Deletion guarantee | Full data purge within 30 days of tenant deletion (GDPR) |
| Export rights | Tenant can export all their data in standard formats (JSON/CSV) at any time |
| Audit isolation | Tenant can only see their own audit logs |

#### Compliance Matrix

| Standard | SaaS | Hybrid | Dedicated |
|----------|------|--------|-----------|
| GDPR | Yes | Yes | Yes |
| ISO 27001 | Professional+ | Yes | Yes |
| Cyber Essentials | All tiers | Yes | Yes |
| SOC 2 Type II | Enterprise | Enterprise | Yes |
| ITAR / defence | No | With controls | Yes |
| AS9100 (Aerospace QMS) | Professional+ | Yes | Yes |
| ISO 13485 (Medical) | Enterprise | Yes | Yes |

### 6.8 Hybrid Deployment Details

#### What Stays On-Premises

| Component | Reason | Syncs to Cloud |
|-----------|--------|----------------|
| SCADA Agent | Low-latency machine connectivity required | Aggregated status, OEE (not raw data) |
| IoT Gateway | Local sensor data collection | Filtered/aggregated telemetry |
| Local Cache | Offline operation during connectivity loss | Queued operations replayed on reconnect |
| Sensitive Job Data | Defence/classified restrictions | Metadata only (no classified content) |
| Raw Machine Parameters | IP-sensitive process data | Statistical summaries only |

#### What Runs in Cloud

| Component | Reason | Benefit |
|-----------|--------|---------|
| ERP (Orders, CRM, Finance) | Multi-site collaboration | Accessible anywhere, always up-to-date |
| AI & Document Intelligence | GPU compute required | Cost-effective, auto-scaling |
| Analytics & Reporting | Cross-site aggregation | Unified view across factories |
| User Management & SSO | Centralised identity | Single sign-on across all sites |
| Billing & Licensing | Platform concern | Automated, self-service |
| Mobile Apps | Internet access needed | Push notifications, remote monitoring |

#### Edge Node Specification

```
Edge Node (Minimum Spec per Site):
├── Hardware: Intel NUC i5 / Dell Edge Gateway 5200 / Custom
├── OS: Ubuntu 22.04 LTS / Windows Server IoT
├── Runtime: Docker + K3s (lightweight Kubernetes)
├── Containers:
│   ├── scada-agent         (OPC-UA/Modbus/MTConnect adapter)
│   ├── iot-gateway         (MQTT broker + device manager)
│   ├── local-cache         (Redis for real-time state)
│   ├── store-forward       (RabbitMQ for offline queuing)
│   ├── edge-analytics      (ML.NET models for local inference)
│   └── sync-agent          (Cloud synchronisation manager)
├── Storage: 500GB SSD (72-hour buffer minimum)
├── Network: Dual NIC (factory LAN + internet/VPN)
└── Security: TPM 2.0, encrypted disk, certificate-based auth

Connectivity:
├── Outbound: HTTPS/WSS to DigiSwiss cloud (port 443 only)
├── Inbound: None required (edge initiates all connections)
├── VPN: WireGuard tunnel for management access
└── Bandwidth: Minimum 10 Mbps upload (typical: 50-100 KB/s)
```

### 6.9 Multi-Tenant Platform Administration

#### Platform Admin Portal (Super-Admin)

| Feature | Description |
|---------|-------------|
| Tenant Dashboard | Overview of all tenants: status, usage, revenue, health |
| Onboarding Monitor | Track new tenant provisioning, activation rates |
| Usage Analytics | Per-tenant resource consumption (storage, API calls, AI credits) |
| Billing Management | Subscription changes, invoicing, payment failures |
| Feature Flag Control | Enable/disable features per tenant or globally |
| Health Monitoring | Per-tenant error rates, performance metrics |
| Support Ticketing | Customer support queue with tenant context |
| Compliance Reports | Data residency, access logs, security events |
| Announcements | Platform-wide or tier-specific notifications |
| Release Management | Rolling deployments, canary releases, feature rollouts |

#### Tenant Self-Service Admin

| Feature | Description |
|---------|-------------|
| User Management | Add/remove users, assign roles, manage invitations |
| Subscription | View plan, upgrade/downgrade, billing history |
| Branding | Upload logo, set colours, custom domain |
| Integrations | API keys, webhook configuration, SSO setup |
| Data Export | Full data export in JSON/CSV |
| Usage Dashboard | Current usage vs limits (users, storage, AI credits) |
| Edge Nodes | Monitor connected edge nodes (hybrid only) |
| Audit Log | View all actions within their tenant |

### 6.10 SaaS Operational Concerns

#### Noisy Neighbour Prevention

| Control | Implementation |
|---------|---------------|
| API Rate Limiting | Per-tenant rate limits based on subscription tier |
| Database Resource Governor | SQL Server Resource Governor pools per tier |
| Compute Isolation | Kubernetes resource quotas per tenant pod |
| Storage Throttling | Blob storage IOPS limits per container |
| SignalR Connection Limits | Max connections per tenant |
| Background Job Queues | Separate queues with priority by tier |
| AI Credit System | Monthly credit allocation prevents runaway costs |

#### Zero-Downtime Deployments

```
Deployment Strategy:
1. Blue/Green deployment for API services
2. Rolling updates for background workers
3. Database migrations: backwards-compatible only (expand-contract pattern)
4. Feature flags for gradual rollout (% of tenants → all)
5. Canary releases to Enterprise tenants first (they have best monitoring)
6. Automatic rollback on error rate > 1%
```

#### Tenant Data Backup & Recovery

| Tier | Backup Frequency | Retention | Recovery SLA |
|------|-----------------|-----------|--------------|
| Free/Trial | Daily | 7 days | Best effort |
| Starter | Daily | 30 days | 24 hours |
| Professional | Hourly | 90 days | 4 hours |
| Enterprise | Continuous (point-in-time) | 1 year | 1 hour |

---

## 7. Integration Requirements

### 7.1 Machine Connectivity
- OPC-UA for CNC machines (e.g., Fanuc, Siemens, Haas)
- Modbus TCP/RTU for PLCs and sensors
- MTConnect for standardised machine data
- Custom REST/MQTT adapters for legacy equipment

### 7.2 External System Integration
| System | Integration Type | Purpose |
|--------|-----------------|---------|
| Accounting Software | API / CSV Import | Financial data sync |
| CAD/CAM Systems | File watchers / API | Drawing import, program management |
| Supplier Portals | API | Automated purchasing |
| Shipping/Logistics | API | Dispatch and tracking |
| Email/SMS | SMTP / Twilio | Notifications and alerts |

### 7.3 Legacy Migration
- Replace CSV/FTPS workflows with API-driven integrations
- Data migration strategy from existing systems
- Parallel running period for validation

---

## 8. Non-Functional Requirements

### 8.1 Performance
| Metric | Target |
|--------|--------|
| API Response Time | < 200ms (95th percentile) |
| Real-Time Data Latency | < 1 second (SignalR) |
| Dashboard Load Time | < 2 seconds |
| Concurrent Users | 100+ simultaneous |
| IoT Data Ingestion | 10,000+ messages/minute |

### 8.2 Reliability & Availability
| Metric | Target |
|--------|--------|
| Uptime SLA | 99.9% |
| RTO (Recovery Time Objective) | < 1 hour |
| RPO (Recovery Point Objective) | < 5 minutes |
| Failover | Automatic with health checks |

### 8.3 Security
- Role-Based Access Control (RBAC) with granular permissions
- Data encryption at rest (AES-256) and in transit (TLS 1.3)
- Audit trail for all critical operations
- GDPR compliance for personal data
- API key management for machine-to-machine communication
- Multi-factor authentication for admin users

### 8.4 Scalability
- Horizontal scaling via containerised microservices
- Database read replicas for reporting workloads
- Message queue-based decoupling for burst handling
- CDN for static assets and global access

---

## 9. User Roles & Permissions

| Role | Access Level | Primary Functions |
|------|-------------|-------------------|
| System Administrator | Full | Configuration, user management, system health |
| Production Manager | High | Planning, scheduling, reporting, approvals |
| Quality Manager | High | QC processes, NCRs, calibration, audits |
| Shop Floor Operator | Medium | Job execution, inspections, time logging |
| Machine Operator | Medium | Machine operation, status updates, fault reporting |
| Purchasing Manager | Medium | Procurement, supplier management, stock |
| Sales/Estimating | Medium | Quotations, order entry, customer management |
| Finance | Medium | Costing, invoicing, financial reports |
| Viewer/Guest | Low | Read-only dashboards and reports |

---

## 10. Delivery Roadmap

### Phase 1: Stabilisation & Core (Weeks 1-3)
- [ ] Prototype review and architecture assessment
- [ ] Codebase stabilisation and technical debt resolution
- [ ] Core ERP: Order management, inventory basics
- [ ] Core MES: Work order execution, job tracking
- [ ] Authentication & authorisation framework
- [ ] CI/CD pipeline setup
- [ ] Database schema design and migration strategy

### Phase 2: Real-Time & Automation (Weeks 4-5)
- [ ] SignalR real-time dashboards
- [ ] SCADA: Machine status monitoring
- [ ] AI Document Intelligence pipeline
- [ ] SQL-triggered automation engine
- [ ] QR-based operations (onboarding, allocation)
- [ ] Automated alert system

### Phase 3: Multi-Platform & IoT (Weeks 6-7)
- [ ] .NET MAUI mobile app (iOS + Android)
- [ ] Desktop application (Windows)
- [ ] IoT device management and data ingestion
- [ ] Edge processing implementation
- [ ] Machine connectivity (OPC-UA/Modbus)

### Phase 4: Polish & Launch (Week 8)
- [ ] Integration testing across all platforms
- [ ] Performance optimisation and load testing
- [ ] Security audit and penetration testing
- [ ] User acceptance testing
- [ ] Documentation and training materials
- [ ] Production deployment and monitoring setup

---

## 11. Key Performance Indicators (KPIs)

| KPI | Target | Measurement |
|-----|--------|-------------|
| Manual Data Entry Reduction | 70% | Before/after comparison |
| Check-in/Onboarding Time | 40-60% reduction | Time tracking |
| Machine Utilisation Visibility | 100% of connected machines | Dashboard coverage |
| Order-to-Delivery Tracking | End-to-end | Process completion rate |
| System Uptime | 99.9% | Monitoring alerts |
| User Adoption | 90%+ within 30 days | Login/usage analytics |

---

## 12. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Legacy system integration complexity | High | Medium | Phased migration, parallel running |
| Machine connectivity challenges | High | Medium | Protocol adapters, vendor consultation |
| Two-month timeline pressure | Medium | High | MVP approach, prioritised backlog |
| Data migration accuracy | High | Medium | Validation scripts, reconciliation reports |
| User adoption resistance | Medium | Medium | Training, intuitive UX, champion users |
| IoT data volume scaling | Medium | Low | Cloud auto-scaling, edge filtering |

---

## 13. Assumptions & Dependencies

### Assumptions
1. Existing prototype provides a working foundation (not a complete rewrite)
2. Machine/sensor hardware is available or can be procured
3. Azure cloud hosting is acceptable
4. UK-based operations (timezone, compliance, currency)
5. English language interface (initially)

### Dependencies
1. Access to existing prototype source code and documentation
2. Machine specifications and communication protocols
3. Business process documentation / SME availability
4. Test environment with representative hardware
5. Azure subscription and resource provisioning

---

## 14. Success Criteria

The platform will be considered successful when:

1. **Operational:** All P0 features deployed and operational across web platform
2. **Automated:** Document processing pipeline achieving 70% reduction in manual entry
3. **Real-Time:** Live dashboards displaying machine/production status with < 1s latency
4. **Multi-Platform:** Functional apps on web, desktop, iOS, and Android
5. **Integrated:** API-driven connections replacing legacy CSV/FTPS workflows
6. **Adopted:** Shop floor operators actively using the system for daily operations

---

## 15. Next Steps

1. **Prototype Review** - Access and assess current codebase, architecture, and database
2. **Requirements Workshop** - Detailed walkthrough of business processes with stakeholders
3. **Architecture Decision Records** - Finalise technology choices and patterns
4. **Sprint Planning** - Break Phase 1 into 1-week sprints with deliverables
5. **Environment Setup** - Development, staging, and production environments

---

## 16. Detailed Specification Documents

This product specification is supported by detailed module-level documents that provide comprehensive technical depth for each area of the platform.

### Module Specifications

| # | Document | Path | Contents |
|---|----------|------|----------|
| 1 | **ERP Module** | [modules/01-erp-module.md](modules/01-erp-module.md) | Order management workflows, inventory data models, production planning, purchasing, job costing, CRM, document management. 7 sub-modules with full entity definitions and business rules. |
| 2 | **MES Module** | [modules/02-mes-module.md](modules/02-mes-module.md) | Work order execution lifecycle, operator interface (screen mockups), quality control & inspection, traceability chains, digital travellers, tool management. Shop floor visibility board design. |
| 3 | **SCADA Module** | [modules/03-scada-module.md](modules/03-scada-module.md) | 8 communication protocols (OPC-UA, MTConnect, Modbus, MQTT, Focas2, etc.), machine state model, 5-level alarm hierarchy, 3 dashboard specifications, OEE calculations, historical data strategy, remote monitoring. |
| 4 | **IoT Module** | [modules/04-iot-module.md](modules/04-iot-module.md) | 3-tier architecture (Device/Edge/Cloud), device lifecycle & provisioning, edge computing rules, data pipeline with volume estimates, sensor catalogue, predictive maintenance ML pipeline, energy management, firmware OTA. |
| 5 | **AI & Automation Module** | [modules/05-ai-automation-module.md](modules/05-ai-automation-module.md) | Document Intelligence pipeline (8 document types, GPT integration), SQL-triggered automation engine (10 templates), QR-based operations, intelligent scheduling, anomaly detection, natural language interface, visual workflow builder. |

### Cross-Cutting Specifications

| # | Document | Path | Contents |
|---|----------|------|----------|
| 6 | **API Specification** | [06-api-specification.md](06-api-specification.md) | RESTful API design principles, OAuth 2.0 + JWT auth, 5 service APIs with full endpoint tables and request/response examples, SignalR real-time hubs (typed interfaces), Webhook API, API Gateway routing, rate limiting. |
| 7 | **Data Model** | [07-data-model.md](07-data-model.md) | Database strategy (5 storage types), ER diagrams, full SQL schema (Core, ERP, MES, SCADA, IoT schemas), indexing strategy, cross-module relationships, data retention policies, multi-tenancy (row-level security), migration strategy. |
| 8 | **UI/UX Specification** | [08-ui-ux-specification.md](08-ui-ux-specification.md) | Design tokens, application shell, navigation structure, key screen wireframes, 4 user journey flows, component library, responsive breakpoints, accessibility (WCAG 2.1 AA), platform-specific notes (Web/Mobile/Desktop/Kiosk), notifications, theming. |
| 9 | **SaaS, Hybrid & Multi-Tenant** | [09-saas-multitenancy.md](09-saas-multitenancy.md) | Deployment models, multi-tenant isolation strategy, subscription tiers & billing, hybrid edge architecture, tenant lifecycle, platform admin, noisy-neighbour prevention, data sovereignty, compliance matrix. |

### Document Map

```
.kiro/specs/
├── product-specification.md          ← This document (overview & index)
├── 06-api-specification.md           ← API contracts & real-time hubs
├── 07-data-model.md                  ← Database schema & relationships
├── 08-ui-ux-specification.md         ← UI/UX design & user journeys
├── 09-saas-multitenancy.md           ← SaaS, Hybrid & Multi-Tenant spec
└── modules/
    ├── 01-erp-module.md              ← ERP detailed spec
    ├── 02-mes-module.md              ← MES detailed spec
    ├── 03-scada-module.md            ← SCADA detailed spec
    ├── 04-iot-module.md              ← IoT detailed spec
    └── 05-ai-automation-module.md    ← AI & Automation detailed spec
```

---

## 17. Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | May 2026 | Initial | Product specification created with all module overviews |
| 1.1 | May 2026 | Refined | Added 8 detailed specification documents covering all modules, API contracts, data model, and UI/UX |
| 1.2 | May 2026 | Enhanced | Added SaaS delivery model, hybrid deployment architecture, multi-tenant strategy, subscription tiers, tenant lifecycle, and compliance matrix |

---

*Document Version: 1.2*  
*Created: May 2026*  
*Last Updated: May 2026*  
*Status: Draft - Awaiting Stakeholder Review*  
*Total Specification Pages: ~180+ (across all documents)*
