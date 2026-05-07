# DigiSwiss - UI/UX Specification

## 1. Design Philosophy

### Principles
| Principle | Description |
|-----------|-------------|
| Manufacturing-First | Designed for factory environments: high contrast, large touch targets, glove-friendly |
| Role-Adaptive | Interface adapts complexity based on user role (manager vs operator) |
| Progressive Disclosure | Show essential info first, reveal detail on demand |
| Offline-Resilient | Core shop floor functions work without connectivity |
| Consistent Cross-Platform | Shared design language across Web, Desktop, iOS, Android |
| Accessibility | WCAG 2.1 AA compliance minimum |

### Design Tokens

| Token | Value | Usage |
|-------|-------|-------|
| Primary | `#1B4F72` (Dark Blue) | Headers, primary actions, navigation |
| Secondary | `#2E86C1` (Medium Blue) | Secondary actions, links |
| Success | `#1E8449` (Green) | Running, pass, complete, on-target |
| Warning | `#D4AC0D` (Amber) | Setup, warning, approaching limit |
| Danger | `#C0392B` (Red) | Fault, fail, overdue, critical alarm |
| Neutral | `#2C3E50` (Dark Grey) | Body text, borders |
| Surface | `#F8F9FA` (Light Grey) | Backgrounds, cards |
| Font (Web/Desktop) | Inter / Segoe UI | Clean, highly legible at all sizes |
| Font (Data) | JetBrains Mono | Numbers, codes, measurements |
| Border Radius | 8px (cards), 4px (inputs) | Consistent roundness |
| Spacing Unit | 8px base grid | All spacing in multiples of 8 |
| Min Touch Target | 48×48px | WCAG + glove-friendly |

---

## 2. Application Shell & Navigation

### 2.1 Web Application Layout

```
┌─────────────────────────────────────────────────────────────────────┐
│  ┌──────┐  DigiSwiss          [🔍 Search]  [🔔 3]  [👤 J.Smith ▾] │
│  │ LOGO │                                                           │
├──┴──────┴───────────────────────────────────────────────────────────┤
│  │          │                                                       │
│  │ SIDEBAR  │              MAIN CONTENT AREA                        │
│  │          │                                                       │
│  │ ▶ Dash   │  ┌─────────────────────────────────────────────────┐ │
│  │          │  │  Breadcrumb: Home > Orders > WO-2026-001234     │ │
│  │ ▶ Orders │  ├─────────────────────────────────────────────────┤ │
│  │ ▶ Produc │  │                                                 │ │
│  │ ▶ Quality│  │          Page Content                           │ │
│  │ ▶ Stock  │  │                                                 │ │
│  │ ▶ Machines│  │                                                 │ │
│  │ ▶ IoT    │  │                                                 │ │
│  │ ▶ AI     │  │                                                 │ │
│  │ ▶ Reports│  │                                                 │ │
│  │          │  │                                                 │ │
│  │ ─────── │  │                                                 │ │
│  │ ▶ Settings│ │                                                 │ │
│  │ ▶ Help   │  │                                                 │ │
│  │          │  └─────────────────────────────────────────────────┘ │
└──┴──────────┴───────────────────────────────────────────────────────┘
```

### 2.2 Navigation Structure

| Level 1 | Level 2 | Level 3 |
|---------|---------|---------|
| Dashboard | Overview, My Tasks, KPIs | - |
| Orders | All Orders, New Order, Quotes, Dispatch | Order Detail, Line Detail |
| Production | Schedule (Gantt), Work Orders, Shop Floor Board | WO Detail, Operation Detail |
| Quality | Inspections, NCRs, CAPA, Certificates | Inspection Form, NCR Detail |
| Inventory | Stock Items, Movements, Purchase Orders, Goods Receipt | Item Detail, PO Detail |
| Machines | Status Map, Machine List, OEE, Alarms | Machine Detail, Trend Viewer |
| IoT | Devices, Energy, Fleet, Predictive | Device Detail, Energy Dashboard |
| AI & Automation | Documents, Rules, Workflows, NLQ | Document Review, Rule Builder |
| Reports | Standard Reports, Custom Builder, Scheduled | Report Viewer |
| Settings | Users, Roles, System Config, Integrations | - |

### 2.3 Mobile Navigation (MAUI)

```
┌─────────────────────────────┐
│  DigiSwiss        [🔔] [≡]  │
├─────────────────────────────┤
│                             │
│      CONTENT AREA           │
│                             │
│                             │
│                             │
│                             │
│                             │
├─────────────────────────────┤
│ [🏠] [📋] [⚙️] [📊] [👤]  │
│ Home  Jobs Machines Dash  Me │
└─────────────────────────────┘
```

---

## 3. Key Screen Specifications

### 3.1 Dashboard (Management)

```
┌─────────────────────────────────────────────────────────────────────┐
│  DASHBOARD                                    Today: 7 May 2026     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐│
│  │ ORDERS   │ │   OEE    │ │  ON-TIME │ │  QUALITY │ │ ALARMS  ││
│  │  Active  │ │  Today   │ │ Delivery │ │  FPY     │ │ Active  ││
│  │   47     │ │  76.3%   │ │  94.2%   │ │  98.7%   │ │    3    ││
│  │ ▲3 vs yd │ │ ▲2.1%   │ │ ▼0.8%   │ │ ▲0.3%   │ │ 1 crit  ││
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └─────────┘│
│                                                                     │
│  ┌────────────────────────────────┐ ┌──────────────────────────┐  │
│  │  ORDERS DUE THIS WEEK         │ │  MACHINE STATUS          │  │
│  │                                │ │                          │  │
│  │  Mon: ████████░░ 8/10         │ │  Running:  7  ████████  │  │
│  │  Tue: ██████░░░░ 6/10         │ │  Setup:    2  ███       │  │
│  │  Wed: ████░░░░░░ 4/12         │ │  Idle:     2  ███       │  │
│  │  Thu: ░░░░░░░░░░ 0/8          │ │  Fault:    1  ██ ⚠️     │  │
│  │  Fri: ░░░░░░░░░░ 0/9          │ │  Offline:  0            │  │
│  │                                │ │                          │  │
│  └────────────────────────────────┘ └──────────────────────────┘  │
│                                                                     │
│  ┌────────────────────────────────┐ ┌──────────────────────────┐  │
│  │  RECENT ACTIVITY              │ │  ACTIONS REQUIRED        │  │
│  │                                │ │                          │  │
│  │  14:30 WO-1234 Op20 Complete  │ │  ⚠ 3 POs awaiting       │  │
│  │  14:15 NCR-0456 Raised        │ │    approval              │  │
│  │  14:02 GRN received (PO-789) │ │  ⚠ 2 Documents need      │  │
│  │  13:45 HAAS-02 Setup started  │ │    review                │  │
│  │  13:30 Order ORD-567 created  │ │  ⚠ 1 NCR pending        │  │
│  │                                │ │    disposition           │  │
│  └────────────────────────────────┘ └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Order List View

```
┌─────────────────────────────────────────────────────────────────────┐
│  ORDERS                              [+ New Order]  [Export]  [⚙]  │
├─────────────────────────────────────────────────────────────────────┤
│  [All] [Draft] [Confirmed] [In Production] [Complete] [Overdue]    │
│                                                                     │
│  🔍 Search orders...    Customer: [All ▾]  Date: [This Month ▾]   │
├─────────────────────────────────────────────────────────────────────┤
│  Order #     │ Customer      │ Due Date  │ Value    │ Status  │ ⚡ │
│  ─────────────┼───────────────┼───────────┼──────────┼─────────┼───│
│  WO-001234   │ Rolls-Royce   │ 12 May    │ £12,450  │ 🟢 Prod │   │
│  WO-001235   │ BAE Systems   │ 14 May    │ £8,200   │ 🟡 Conf │   │
│  WO-001236   │ Airbus UK     │ 10 May    │ £3,750   │ 🔴 Over │ ⚠ │
│  WO-001237   │ Siemens       │ 18 May    │ £22,100  │ 🟢 Prod │   │
│  WO-001238   │ GE Aviation   │ 20 May    │ £6,900   │ ⬜ Draft│   │
│  ...                                                                │
├─────────────────────────────────────────────────────────────────────┤
│  Showing 1-20 of 47          [◀ Prev]  Page 1 of 3  [Next ▶]      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Production Schedule (Gantt)

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRODUCTION SCHEDULE          [◀ Week]  7-11 May 2026  [Week ▶]    │
│  [Day] [Week] [Month]        [+ Add Job]  [🤖 AI Optimise]        │
├──────────────┬──────────────────────────────────────────────────────┤
│              │  Mon 7  │  Tue 8  │  Wed 9  │  Thu 10 │  Fri 11   │
│  Machine     │ AM │ PM │ AM │ PM │ AM │ PM │ AM │ PM │ AM │ PM   │
├──────────────┼────┼────┼────┼────┼────┼────┼────┼────┼────┼──────┤
│  HAAS VF-4   │████████████│░░░░│████████████████████│░░░░░░│      │
│  #01         │ WO-1234    │Idle│    WO-1245         │Setup │      │
├──────────────┼────────────┼────┼─────────────────────┼──────┼──────┤
│  HAAS VF-4   │████████│░░░░░░░░│████████████████████████████│      │
│  #02         │ WO-1236│  Setup  │       WO-1237              │      │
├──────────────┼────────┼─────────┼────────────────────────────┼──────┤
│  DMG MORI    │████████████████████████│░░│██████████████████│      │
│  5-Axis      │      WO-1238           │  │   WO-1240       │      │
├──────────────┼────────────────────────────────────────────────┼──────┤
│  MAZAK       │▓▓▓▓▓▓▓▓│████████████████████│░░░░░░░░░░░░░░│      │
│  QTN-250     │Maint   │     WO-1241        │   Available   │      │
└──────────────┴────────────────────────────────────────────────┴──────┘
│  Legend: ████ Running  ░░░░ Idle/Setup  ▓▓▓▓ Maintenance           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4. User Journeys

### 4.1 Journey: Production Manager - Morning Review

```
Step 1: Login → Dashboard
  ┌─ View overnight summary (auto-generated)
  ├─ Check active alarms (1 critical → investigate)
  ├─ Review overdue orders (2 flagged → re-prioritise)
  └─ Check OEE trending down on HAAS-02

Step 2: Dashboard → Machine Detail (HAAS-02)
  ┌─ View OEE breakdown: Performance low (frequent stops)
  ├─ Check alarm history: 5 minor stops in last shift
  ├─ View spindle load trend: increasing over 2 weeks
  └─ Create maintenance ticket from dashboard

Step 3: Dashboard → Orders → Overdue
  ┌─ Identify 2 at-risk orders
  ├─ Open each → view remaining operations
  ├─ Drag-and-drop reschedule on Gantt
  └─ System confirms new delivery estimate → notify customer

Step 4: Dashboard → AI Scheduling
  ┌─ Request AI schedule optimisation for next week
  ├─ Review proposed changes (comparison view)
  ├─ Accept proposed schedule
  └─ System updates work centre queues

Total time: ~15 minutes (vs 45+ minutes with manual systems)
```

### 4.2 Journey: Operator - Start Shift & Work Job

```
Step 1: Scan QR Badge → Clock In
  ┌─ Operator dashboard loads
  ├─ Shows assigned jobs for today
  ├─ Any overnight alerts/notes from supervisor
  └─ Machine assigned: HAAS VF-4 #01

Step 2: Scan Work Order QR → Start Job
  ┌─ Job details load: WO-1234, Op 20, CNC Milling
  ├─ Drawing displayed (tap to zoom)
  ├─ Tool list shown (verify tools loaded)
  ├─ Material batch confirmed: HT-2026-0456
  └─ Tap [START SETUP] → Timer begins

Step 3: Setup Complete → Start Running
  ┌─ Tap [SETUP COMPLETE] → Setup time recorded
  ├─ Load NC program (displayed on screen)
  ├─ Tap [START RUNNING] → Production timer begins
  └─ Parts counter begins tracking

Step 4: First Article Inspection
  ┌─ After first part: Tap [INSPECT]
  ├─ Inspection form loads (from plan)
  ├─ Enter measurements for each characteristic
  ├─ System calculates pass/fail automatically
  ├─ All pass → [APPROVE & CONTINUE]
  └─ Production resumes

Step 5: Complete Operation
  ┌─ All parts done: Tap [COMPLETE]
  ├─ Enter: Good qty: 49, Scrap qty: 1
  ├─ Select scrap reason: "Tool breakage"
  ├─ Sign-off recorded
  └─ Next operation queued (or final inspection)

Total time per job: 5-10 seconds for each status change
```

### 4.3 Journey: Quality Manager - Handle NCR

```
Step 1: Alert Received → NCR Dashboard
  ┌─ Push notification: "NCR-0456 raised on WO-1234"
  ├─ Open NCR from notification
  └─ View details: 3 parts out of tolerance on OD

Step 2: NCR Detail → Investigation
  ┌─ View failed measurements (chart vs tolerance)
  ├─ Check traceability: material batch, operator, machine
  ├─ Review machine data: spindle load spike at 14:22
  ├─ View SPC chart: process drifting for last 10 parts
  └─ Attach photos of defective parts

Step 3: Root Cause & Disposition
  ┌─ Enter root cause: "Tool wear - exceeded life"
  ├─ Select disposition: Rework (re-machine to tolerance)
  ├─ Create corrective action: "Reduce tool life limit by 20%"
  ├─ Assign action to production supervisor
  └─ Set due date for verification

Step 4: Close Loop
  ┌─ Rework completed → re-inspect
  ├─ All pass → NCR status → Closed
  ├─ Corrective action verified (tool life updated in system)
  └─ Report generated for management review
```

### 4.4 Journey: Purchasing - AI Document Processing

```
Step 1: Supplier Email Arrives with PO Confirmation
  ┌─ Document Intelligence auto-processes attachment
  ├─ Classifies as "Purchase Order Confirmation"
  ├─ Extracts: PO number, line items, quantities, prices, dates
  └─ Confidence: 96% → auto-matched to our PO-789

Step 2: System Auto-Updates
  ┌─ PO-789 status → "Confirmed by Supplier"
  ├─ Expected delivery dates updated
  ├─ Any discrepancies flagged (price diff on line 3)
  └─ Notification to purchasing: "Price variance on PO-789 line 3"

Step 3: Purchasing Reviews (only the exception)
  ┌─ Open flagged document in review queue
  ├─ Side-by-side: our PO vs supplier confirmation
  ├─ Price difference: £12.50 vs £12.75 per unit
  ├─ Decision: Accept (within tolerance) → Approve
  └─ System updates PO with confirmed pricing

Time: 2 minutes (vs 15+ minutes manual processing per document)
```

---

## 5. Component Library

### 5.1 Status Indicators

| Component | States | Visual |
|-----------|--------|--------|
| Status Badge | Running, Setup, Idle, Fault, Offline, Complete | Coloured pill with icon |
| Priority Flag | Standard, High, Urgent, Critical | Grey → Blue → Orange → Red flag |
| Progress Bar | 0-100% | Segmented bar with percentage label |
| Health Score | 0-100 | Circular gauge (green/amber/red zones) |
| Trend Arrow | Up, Down, Flat | ▲ Green (good), ▼ Red (bad), ► Grey (flat) |
| Confidence Score | 0-100% | Colour-graded bar (red → amber → green) |

### 5.2 Data Display Components

| Component | Usage |
|-----------|-------|
| Data Card | KPI tiles on dashboards (value + trend + comparison) |
| Data Table | Sortable, filterable lists with row actions |
| Timeline | Activity feeds, audit history, status changes |
| Gantt Chart | Production scheduling, resource allocation |
| Trend Chart | Line/area charts for time-series data (Spindle load, temp) |
| SPC Chart | Control charts with UCL/LCL/mean lines |
| Floor Plan | Interactive factory layout with device/machine markers |
| Tree View | BOM structures, traceability chains, org hierarchies |

### 5.3 Input Components

| Component | Usage | Features |
|-----------|-------|----------|
| Smart Search | Global search, NLQ input | Autocomplete, recent, suggestions |
| Numeric Pad | Inspection measurements | Large buttons, decimal, +/- |
| QR Scanner | Camera-based code scanning | Auto-focus, torch, manual entry fallback |
| Date Picker | Due dates, schedule | Calendar view, quick presets |
| Signature Pad | Sign-off (touch draw) | Clear, undo, captures as image |
| Photo Capture | Defect photos, setup evidence | Camera + gallery, annotate |
| Dropdown Select | Status, categories, users | Search within, multi-select option |
| Toggle Switch | Enable/disable, yes/no | Clear on/off state |

---

## 6. Responsive Breakpoints

| Breakpoint | Width | Target | Layout |
|-----------|-------|--------|--------|
| Desktop XL | ≥1440px | 27"+ monitors, wall displays | Full sidebar + 3-column content |
| Desktop | 1024-1439px | Standard monitors | Sidebar + 2-column content |
| Tablet | 768-1023px | iPad, shop floor tablets | Collapsible sidebar, single column |
| Mobile | 320-767px | Phones (MAUI app) | Bottom nav, full-width cards |
| Kiosk | 1920×1080 | Shop floor displays | Custom full-screen layout |

---

## 7. Accessibility Requirements

| Requirement | Standard | Implementation |
|-------------|----------|----------------|
| Colour Contrast | WCAG 2.1 AA (4.5:1 text, 3:1 UI) | All colour pairs tested |
| Keyboard Navigation | Full keyboard access | Tab order, focus indicators, shortcuts |
| Screen Reader | ARIA labels on all interactive elements | Semantic HTML, live regions |
| Touch Targets | Minimum 48×48px | All buttons, links, controls |
| Font Scaling | Support 200% zoom without loss | Relative units (rem/em) |
| Motion | Respect `prefers-reduced-motion` | Disable animations if set |
| High Contrast | Windows High Contrast mode | Tested and functional |
| Error Messaging | Clear, actionable error text | Not colour-only indicators |

---

## 8. Platform-Specific Considerations

### 8.1 Web (Blazor)

| Concern | Approach |
|---------|----------|
| Loading Performance | Blazor WASM with pre-rendering, lazy load modules |
| State Management | Fluxor/Redux pattern for complex state |
| Offline Support | Service Worker for static assets + IndexedDB for queued actions |
| Real-Time | SignalR auto-reconnect with exponential backoff |
| Print | Dedicated print stylesheets for reports/travellers |

### 8.2 Mobile (MAUI - iOS/Android)

| Concern | Approach |
|---------|----------|
| Camera/QR | Native camera access for scanning |
| Push Notifications | Firebase (Android) + APNS (iOS) |
| Offline Mode | SQLite local DB, background sync |
| Biometrics | Face ID / fingerprint for quick auth |
| Haptics | Vibration feedback on scan success/failure |
| Dark Mode | Full dark theme support |

### 8.3 Desktop (MAUI - Windows)

| Concern | Approach |
|---------|----------|
| Multi-Window | Support detached windows (e.g., trend viewer on second monitor) |
| System Tray | Minimise to tray, badge for alerts |
| Keyboard Shortcuts | Full shortcut system for power users |
| USB/Serial | Direct connection to measurement devices (CMM, callipers) |
| Printing | Direct print to label printers (Zebra, Brother) |

### 8.4 Kiosk / Shop Floor Display

| Concern | Approach |
|---------|----------|
| Auto-Launch | Full-screen on boot, no browser chrome |
| Touch Only | No keyboard/mouse, large touch targets |
| Auto-Refresh | Page auto-cycles through relevant views |
| Brightness | High brightness for factory lighting |
| Authentication | QR badge scan only (no password typing) |

---

## 9. Notification System

### 9.1 Notification Types

| Type | Channel | Persistence | Example |
|------|---------|-------------|---------|
| Toast | In-app (bottom-right) | 5 seconds | "Job WO-1234 Op 20 complete" |
| Banner | In-app (top, dismissible) | Until dismissed | "System maintenance at 22:00" |
| Badge | Nav item counter | Until read/resolved | Orders (3) — 3 items need action |
| Push | Mobile push notification | Until opened | "Critical alarm: HAAS-01 fault" |
| Email | Email digest | N/A | Daily summary, overdue alerts |
| SMS | Text message | N/A | Critical alarms only |

### 9.2 Notification Preferences

Users can configure per notification type:
- In-app: On/Off
- Push: On/Off
- Email: Immediate / Digest (daily) / Off
- SMS: On/Off (critical only)
- Quiet Hours: Define do-not-disturb windows

---

## 10. Theming & White-Labelling

| Feature | Support |
|---------|---------|
| Company Logo | Configurable (sidebar + login screen) |
| Primary Colour | Adjustable accent colour |
| Dark Mode | Full dark theme (user preference) |
| Custom Dashboard | Drag-and-drop widget arrangement |
| Branded Reports | Company logo/header on generated documents |
| Login Page | Customisable background image + message |

---

*Document Version: 1.0*  
*Parent Document: [Product Specification](product-specification.md)*
