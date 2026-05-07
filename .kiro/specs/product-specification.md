# DigiSwiss - Product Specification

## 1. Executive Summary

**Product Name:** DigiSwiss  
**Domain:** Precision Engineering & Manufacturing  
**Platform Type:** Integrated Digital Manufacturing Platform (ERP + MES + SCADA + IoT)  
**Target Platforms:** Web, Desktop (Windows), iOS, Android  
**Timeline:** 2-month initial delivery (MVP to Production)  
**Current State:** Prototype exists - requires stabilisation, extension, and scaling

---

## 2. Product Vision

DigiSwiss is an integrated digital manufacturing platform designed for precision engineering operations. It unifies Enterprise Resource Planning (ERP), Manufacturing Execution Systems (MES), Supervisory Control and Data Acquisition (SCADA), and Internet of Things (IoT) capabilities into a single, cohesive platform — enabling real-time operational visibility, automated workflows, and data-driven decision making.

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

## 6. Integration Requirements

### 6.1 Machine Connectivity
- OPC-UA for CNC machines (e.g., Fanuc, Siemens, Haas)
- Modbus TCP/RTU for PLCs and sensors
- MTConnect for standardised machine data
- Custom REST/MQTT adapters for legacy equipment

### 6.2 External System Integration
| System | Integration Type | Purpose |
|--------|-----------------|---------|
| Accounting Software | API / CSV Import | Financial data sync |
| CAD/CAM Systems | File watchers / API | Drawing import, program management |
| Supplier Portals | API | Automated purchasing |
| Shipping/Logistics | API | Dispatch and tracking |
| Email/SMS | SMTP / Twilio | Notifications and alerts |

### 6.3 Legacy Migration
- Replace CSV/FTPS workflows with API-driven integrations
- Data migration strategy from existing systems
- Parallel running period for validation

---

## 7. Non-Functional Requirements

### 7.1 Performance
| Metric | Target |
|--------|--------|
| API Response Time | < 200ms (95th percentile) |
| Real-Time Data Latency | < 1 second (SignalR) |
| Dashboard Load Time | < 2 seconds |
| Concurrent Users | 100+ simultaneous |
| IoT Data Ingestion | 10,000+ messages/minute |

### 7.2 Reliability & Availability
| Metric | Target |
|--------|--------|
| Uptime SLA | 99.9% |
| RTO (Recovery Time Objective) | < 1 hour |
| RPO (Recovery Point Objective) | < 5 minutes |
| Failover | Automatic with health checks |

### 7.3 Security
- Role-Based Access Control (RBAC) with granular permissions
- Data encryption at rest (AES-256) and in transit (TLS 1.3)
- Audit trail for all critical operations
- GDPR compliance for personal data
- API key management for machine-to-machine communication
- Multi-factor authentication for admin users

### 7.4 Scalability
- Horizontal scaling via containerised microservices
- Database read replicas for reporting workloads
- Message queue-based decoupling for burst handling
- CDN for static assets and global access

---

## 8. User Roles & Permissions

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

## 9. Delivery Roadmap

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

## 10. Key Performance Indicators (KPIs)

| KPI | Target | Measurement |
|-----|--------|-------------|
| Manual Data Entry Reduction | 70% | Before/after comparison |
| Check-in/Onboarding Time | 40-60% reduction | Time tracking |
| Machine Utilisation Visibility | 100% of connected machines | Dashboard coverage |
| Order-to-Delivery Tracking | End-to-end | Process completion rate |
| System Uptime | 99.9% | Monitoring alerts |
| User Adoption | 90%+ within 30 days | Login/usage analytics |

---

## 11. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| Legacy system integration complexity | High | Medium | Phased migration, parallel running |
| Machine connectivity challenges | High | Medium | Protocol adapters, vendor consultation |
| Two-month timeline pressure | Medium | High | MVP approach, prioritised backlog |
| Data migration accuracy | High | Medium | Validation scripts, reconciliation reports |
| User adoption resistance | Medium | Medium | Training, intuitive UX, champion users |
| IoT data volume scaling | Medium | Low | Cloud auto-scaling, edge filtering |

---

## 12. Assumptions & Dependencies

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

## 13. Success Criteria

The platform will be considered successful when:

1. **Operational:** All P0 features deployed and operational across web platform
2. **Automated:** Document processing pipeline achieving 70% reduction in manual entry
3. **Real-Time:** Live dashboards displaying machine/production status with < 1s latency
4. **Multi-Platform:** Functional apps on web, desktop, iOS, and Android
5. **Integrated:** API-driven connections replacing legacy CSV/FTPS workflows
6. **Adopted:** Shop floor operators actively using the system for daily operations

---

## 14. Next Steps

1. **Prototype Review** - Access and assess current codebase, architecture, and database
2. **Requirements Workshop** - Detailed walkthrough of business processes with stakeholders
3. **Architecture Decision Records** - Finalise technology choices and patterns
4. **Sprint Planning** - Break Phase 1 into 1-week sprints with deliverables
5. **Environment Setup** - Development, staging, and production environments

---

## 15. Detailed Specification Documents

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

### Document Map

```
.kiro/specs/
├── product-specification.md          ← This document (overview & index)
├── 06-api-specification.md           ← API contracts & real-time hubs
├── 07-data-model.md                  ← Database schema & relationships
├── 08-ui-ux-specification.md         ← UI/UX design & user journeys
└── modules/
    ├── 01-erp-module.md              ← ERP detailed spec
    ├── 02-mes-module.md              ← MES detailed spec
    ├── 03-scada-module.md            ← SCADA detailed spec
    ├── 04-iot-module.md              ← IoT detailed spec
    └── 05-ai-automation-module.md    ← AI & Automation detailed spec
```

---

## 16. Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | May 2026 | Initial | Product specification created with all module overviews |
| 1.1 | May 2026 | Refined | Added 8 detailed specification documents covering all modules, API contracts, data model, and UI/UX |

---

*Document Version: 1.1*  
*Created: May 2026*  
*Last Updated: May 2026*  
*Status: Draft - Awaiting Stakeholder Review*  
*Total Specification Pages: ~150+ (across all documents)*
