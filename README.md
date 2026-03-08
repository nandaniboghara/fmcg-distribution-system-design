# 🏭 FMCG Management & Food Distribution Platform

> A production-grade, multi-tenant food distribution backend system processing **100,000+ daily orders** across **2 countries** — built to digitize end-to-end FMCG operations including inventory, order processing, cold-chain logistics, delivery route optimization, cash collection, and financial settlements.

**Role:** Senior Backend Developer (Python/Django) — *end-to-end ownership of Payment Integration, Delivery & Route Optimization, and Cash Collection & Settlement*
**Team Size:** 10+ engineers (cross-functional: backend, frontend, ML, IoT, DevOps)
**Duration:** March 2023 – January 2026

---

## 📌 Table of Contents
- [Problem Statement](#problem-statement)
- [System Architecture](#system-architecture)
- [Tech Stack & Why](#tech-stack--why)
- [Core Modules](#core-modules)
- [What I Personally Owned](#what-i-personally-owned)
- [Database Design](#database-design)
- [API Design](#api-design)
- [Key Engineering Challenges](#key-engineering-challenges)
- [Performance Optimizations](#performance-optimizations)
- [Testing Strategy](#testing-strategy)
- [Production Incidents & War Stories](#production-incidents--war-stories)
- [DevOps & Deployment](#devops--deployment)
- [Leadership & Mentoring](#leadership--mentoring)
- [Results & Impact](#results--impact)
- [Lessons Learned](#lessons-learned)

---

## 🧩 Problem Statement

A large-scale FMCG distributor operating across Tanzania and Kenya relied on fragmented, mostly manual systems for order management, delivery, and financial reconciliation — resulting in:

- ❌ No real-time inventory visibility across multiple warehouses
- ❌ Manual cash collection with zero digital reconciliation trail
- ❌ Delivery routes planned manually — high fuel cost, missed time windows
- ❌ No cold-storage temperature monitoring for perishable goods
- ❌ No unified system supporting multi-currency, multi-tax, multi-gateway operations across countries
- ❌ Bonus calculations for 100+ field agents done manually in spreadsheets

**Goal:** Build a single unified platform that digitizes the entire distribution chain — warehouse to doorstep to financial settlement — at 100,000+ order/day scale, across 2 countries.

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                │
│     Mobile App (Delivery Agents)  │  Web Dashboard (Admin/Finance)  │
└──────────────────┬───────────────────────────────┬───────────────────┘
                   │ HTTPS / REST API               │
┌──────────────────▼───────────────────────────────▼───────────────────┐
│                    API GATEWAY — Django REST Framework                │
│         JWT Auth │ Tenant Middleware │ Rate Limiting │ Versioning     │
└───────┬──────────┬──────────────┬──────────────┬────────────┬────────┘
        │          │              │              │            │
┌───────▼──┐ ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼─────┐ ┌───▼──────┐
│  User &  │ │  Order &  │ │ Inventory │ │ Payment & │ │  Bonus & │
│   Auth   │ │  Delivery │ │ Warehouse │ │Settlement │ │Incentives│
└──────────┘ └───────────┘ └───────────┘ └───────────┘ └──────────┘
        │          │              │              │            │
┌───────▼──────────▼──────────────▼──────────────▼────────────▼───────┐
│                  PostgreSQL — Schema-per-Tenant                       │
│             tenant_tz (Tanzania) │ tenant_ke (Kenya)                 │
└──────────────────────────────────────────────────────────────────────┘
        │                     │                      │
┌───────▼──────┐   ┌──────────▼────────┐   ┌────────▼────────────────┐
│  Celery +    │   │  External Services │   │    AWS Infrastructure   │
│  Redis Queue │   │  Selcom Gateway   │   │    EC2 Auto Scaling     │
│  Async Tasks │   │  TSP ML Engine    │   │    RDS PostgreSQL       │
│              │   │  IoT Cold Storage │   │    S3 (Proof/Receipts)  │
│              │   │  TRA Receipt API  │   │    CloudWatch           │
└──────────────┘   └───────────────────┘   └─────────────────────────┘
```

---

## 🛠️ Tech Stack & Why

| Layer | Technology | Why Chosen |
|---|---|---|
| **Backend** | Python 3.10 + Django 4.x | Mature ORM, rapid module development, team expertise |
| **API** | Django REST Framework | ViewSets, serializers, throttling, versioning out of the box |
| **Async Tasks** | Celery + Redis | Bonus calculation, report generation, WebSocket event dispatch |
| **Real-time** | Django Channels + WebSockets | Live delivery tracking, cold-storage alerts on dashboard |
| **Containers** | Docker + Docker Compose | Environment parity across dev/staging/production |
| **Database** | PostgreSQL 14 | ACID compliance, schema isolation for multi-tenancy, JSONB |
| **Cloud** | AWS EC2 + RDS + S3 | Managed DB, scalable compute, durable proof-of-delivery storage |
| **Auto Scaling** | AWS Auto Scaling Groups | Handle 4-5x traffic spikes during morning delivery hours |
| **Auth** | JWT (DRF SimpleJWT) | Stateless, works seamlessly across mobile and web |
| **Monitoring** | CloudWatch | Infra metrics, alarm-based scaling, slow query alerts |
| **CI/CD** | Bitbucket Pipelines | Automated test runs on PR, staged deployments |
| **Testing** | pytest + pytest-django | Unit + integration test suite across all core modules |

---

## 📦 Core Modules

### 1. 👤 User & Auth
Role-based access control across 5 roles: Admin, Warehouse Manager, Sales Agent, Delivery Agent, Finance Officer. JWT with refresh token rotation. Country-specific user config per tenant.

### 2. 🛒 Product & Inventory
Multi-warehouse real-time stock tracking. Low-stock alerts, automated reorder triggers. Product variants, bulk pricing, country-specific pricing rules. Materialized views for instant dashboard aggregations.

### 3. 📦 Order Management
Full lifecycle: Draft → Confirmed → Packed → Dispatched → Delivered → Settled. Credit limit enforcement, promotional pricing engine, discount gap tracking.

### 4. 🚚 Delivery & Route Optimization *(personally owned)*
TSP-based route optimization consuming live GPS + order location data. Polled IoT cold-storage sensors per warehouse to monitor perishable goods in real time. WebSocket push for live delivery status on admin dashboard. Proof of delivery (photo/signature) stored on S3.

### 5. 💰 Payment & Settlement *(personally owned)*
Selcom payment gateway integration (Tanzania). Multi-step cash collection workflow with full audit trail. TRA receipt generation for tax compliance. Credit/discount gap analysis and reconciliation reporting.

### 6. 🧮 Bonus & Incentive Engine *(personally designed & owned)*
Real-time bonus calculation engine based on delivery performance KPIs. Configurable rules per country/region. Async Celery processing for heavy calculations. Live progress dashboards via WebSocket for agents to track their own bonus status.

---

## 🎯 What I Personally Owned

These weren't just contributions — I designed, built, deployed, and maintained these end-to-end:

| Area | My Ownership |
|---|---|
| **Cash Collection & Settlement** | Designed the entire workflow from scratch — agent collection → supervisor verification → finance reconciliation → TRA receipt generation |
| **Bonus Calculation Engine** | Designed the rules engine, async processing architecture, and real-time progress visibility |
| **Payment Gateway (Selcom)** | Full integration including failure handling, retry logic, and reconciliation |
| **Delivery & TSP Integration** | Built the live data feed consumed by the ML team's TSP solver, and integrated its output back into delivery assignment APIs |
| **IoT Cold Storage Polling** | Designed the polling architecture, alert thresholds, and WebSocket push for real-time dashboard alerts |
| **DB Schema & API Design** | Made independent decisions on schema design, indexing strategy, and API contract changes throughout the project |
| **Release Management** | Owned deployment decisions — coordinated staging validation, managed safe production rollouts, and executed rollbacks when needed |

---

## 🗄️ Database Design

See [`/database/schema.md`](./database/schema.md) for full schema.

**Key design decisions:**
- **Schema-per-tenant** — Tanzania and Kenya get fully isolated PostgreSQL schemas; tenant resolved from JWT on every request via middleware (`SET search_path TO tenant_tz, public`)
- **Soft deletes** on all business entities (`is_deleted`, `deleted_at`)
- **Full audit trail** — `created_by`, `updated_by`, `created_at`, `updated_at` on every table
- **JSONB columns** for flexible country-specific configuration (tax rules, gateway config, bonus rules)
- **Materialized views** for inventory aggregations — refreshed async on write events via Celery
- **Monthly range partitioning** on `cold_storage_logs` — high-volume IoT append-only table

---

## 🔌 API Design

See [`/api-design/endpoints.md`](./api-design/endpoints.md) for full API documentation.

**Principles:**
- RESTful, resource-based URLs with URL versioning (`/api/v1/`, `/api/v2/`)
- Consistent response envelope: `{ status, data, message, errors, meta }`
- Cursor-based pagination on all list endpoints (performance at 100K+ orders/day scale)
- All schema and API design decisions owned independently — backward compatibility enforced via versioning policy

---

## ⚙️ Key Engineering Challenges

### Challenge 1: Designing Cash Collection & Settlement from Scratch
**Problem:** Cash collected by 100+ field agents had no digital trail. Disputes, lost cash, and settlement delays were common business pain points before I joined.

**My Design:**
A 4-stage audited workflow — every state transition timestamped and immutable:

```
Agent logs collection → Supervisor verifies → Finance reconciles → TRA receipt generated
     (pending)              (verified)            (reconciled)          (settled)
```

Each stage has role-based access control — agents can only create, supervisors can only verify, finance can only reconcile. No stage can be skipped. The `calculation_log` JSONB column stores the full breakdown at reconciliation time for dispute resolution.

**Impact:** Eliminated manual cash discrepancy investigations. Full audit trail from collection to settlement for every order.

---

### Challenge 2: Bonus Calculation Engine at Scale
**Problem:** Bonus calculations for 100+ field agents were done in spreadsheets. Rules were complex — based on delivery count, on-time rate, customer satisfaction, and region-specific multipliers. Needed to be real-time visible to agents.

**My Design:**
- Configurable rules stored in DB (not hardcoded) — country admins can adjust without deployments
- Calculations run as async Celery tasks (not blocking API)
- `calculation_log` JSONB stores the full step-by-step breakdown per agent per period
- Agents see live progress toward their bonus via WebSocket — updated on every delivery completion

```
Delivery completed → Celery task triggered → Rules evaluated →
Bonus progress updated → WebSocket event pushed to agent's mobile app
```

---

### Challenge 3: TSP Route Optimization Integration
**Problem:** Delivery routes were manually planned. With 100,000+ daily orders across multiple agents, this was a massive inefficiency.

**My Role:** Built the data feed API consumed by the ML team's TSP solver (live order locations, agent GPS, delivery windows), and integrated the solver's output back into the delivery assignment system.

```
Morning cron → Fetch live orders + agent locations →
POST to TSP engine → Receive optimized routes →
Store route assignments → Agents receive routes via mobile app
```

Designed the integration to be re-triggerable for urgent same-day orders added after the morning run.

---

### Challenge 4: IoT Cold Storage Monitoring
**Problem:** Temperature excursions for perishable goods went undetected until product was already spoiled.

**My Design:** Polling architecture hitting IoT device APIs every 5 minutes per warehouse. Readings stored in partitioned `cold_storage_logs`. Threshold breach triggers a Celery task that pushes a WebSocket alert to the admin dashboard in real time.

```
Celery beat (5min) → Poll IoT API per warehouse →
Store reading → Check threshold → Breach? →
Celery alert task → WebSocket push to dashboard
```

---

### Challenge 5: Multi-Tenant Architecture Across Countries
**Problem:** Tanzania and Kenya needed isolated data (different currencies, tax rules, payment gateways) but shared infrastructure.

**Decision (mine):** Schema-per-tenant over row-level tenancy — stronger data isolation for a financial system, cleaner queries, per-tenant migrations. Accepted the tradeoff of higher migration ops complexity and automated it via a management command.

---

### Challenge 6: New Country Go-Live Under Pressure *(July 2024 Recognition)*
**Problem:** A new country launch deadline coincided with a major client demo commitment. This required payment gateway validation, data migration, environment setup, and end-to-end testing — all within a compressed timeline.

**What I did:** Stretched working hours, independently coordinated across backend, DevOps, and client-facing teams, resolved blocking issues without escalating to senior management, and delivered the go-live on time. Recognized by Moweb Technologies with a Monthly Appreciation Award for this delivery.

---

## 🚀 Performance Optimizations

| Problem | Technique | Result |
|---|---|---|
| Order dashboard slow (3–8s) | Composite index on `(status, delivery_date)` | Query time: 4s → ~50ms |
| N+1 on order list API | `select_related` + `prefetch_related` audit | 51 queries → 3 queries per request |
| Inventory aggregation on every load | Materialized view + async Celery refresh | Dashboard: 3.2s → 8ms |
| Report generation timeouts (30s+) | Async Celery tasks + polling endpoint | API response: immediate; reports reliable |
| IoT log query degradation over time | Monthly range partitioning on `cold_storage_logs` | Consistent query time regardless of data age |
| Delivery-hour traffic spikes (4–5x) | AWS Auto Scaling on CPU > 70% | Zero downtime during peak hours |

See [`/docs/performance.md`](./docs/performance.md) for deep-dive on each optimization.

---

## 🧪 Testing Strategy

**Coverage:** Unit tests + integration tests across all core modules.

```
tests/
├── unit/
│   ├── test_bonus_calculation.py     ← Rules engine edge cases
│   ├── test_cash_collection.py       ← Workflow state transitions
│   ├── test_payment_gateway.py       ← Selcom integration (mocked)
│   ├── test_order_lifecycle.py       ← State machine validation
│   └── test_inventory.py            ← Stock reservation logic
└── integration/
    ├── test_order_to_delivery.py     ← Full order → delivery flow
    ├── test_settlement_workflow.py   ← Collection → reconciliation → receipt
    └── test_multi_tenant.py         ← Schema isolation validation
```

**Key testing principles:**
- Payment gateway tests use mocked Selcom responses — never hit real gateway in CI
- Bonus calculation tests cover all rule combinations and edge cases (zero deliveries, partial periods, multiplier stacking)
- Multi-tenant integration tests verify schema isolation — a query in `tenant_tz` must never return `tenant_ke` data
- Celery tasks tested with `CELERY_TASK_ALWAYS_EAGER=True` in test settings

---

## 🔥 Production Incidents & War Stories

### Incident 1: Payment Gateway Silent Failures
**What happened:** Selcom gateway was returning HTTP 200 with a failure payload for a subset of transactions. Our system was marking orders as paid when they weren't.

**How I resolved it:** Added deep payload validation (not just HTTP status checking), implemented idempotency keys on all payment requests, and built a reconciliation job that cross-checks our payment records against Selcom's transaction log nightly.

---

### Incident 2: Migration Conflict in Production
**What happened:** Two parallel junior developer branches both modified the same model. Merging caused a corrupted migration state in production — the DB schema was ahead of Django's migration history.

**How I resolved it:** Manually reconstructed the migration state using `--fake` on already-applied migrations, squashed the conflicting migrations, validated schema state against production DB, and deployed a clean migration. Introduced a migration review checklist for all future PRs.

---

### Incident 3: Performance Degradation Under Load
**What happened:** Dashboard response times degraded from ~200ms to 8+ seconds during peak morning delivery hours as the order table grew past 5M rows.

**How I resolved it:** Used `EXPLAIN ANALYZE` to identify sequential scans, added composite indexes, introduced materialized views for aggregations, and moved heavy report queries to async Celery tasks. Resolved without any downtime.

---

### Incident 4: Deployment Rollback
**What happened:** A release introduced a regression in the order confirmation flow — a missing `select_related` caused N+1 queries at scale, bringing API response time to 30+ seconds under load.

**How I resolved it:** Identified the regression within 10 minutes of deploy via CloudWatch latency spike alert, executed a rollback to the previous Docker image, and deployed a hotfix with the fix within 2 hours. Introduced a staging load test requirement for any PR touching the order query path.

---

## ☁️ DevOps & Deployment

```
Developer → Git Push → Bitbucket PR
                           │
                    CI Pipeline runs:
                    - pytest (unit + integration)
                    - Docker build validation
                           │
                    Code Review (I reviewed all junior PRs)
                           │
                    Merge → Staging Deploy (Docker)
                           │
                    Staging validation + smoke tests
                           │
                    Production Deploy (EC2 Auto Scaling)
                           │
                    CloudWatch monitors for regressions
                    (rollback executed if latency spikes)
```

**My deployment responsibilities:**
- Owned all release decisions — go/no-go calls on production deployments
- Managed safe rollout sequencing (migrations before code, never after)
- Executed rollbacks when regressions detected post-deploy
- Maintained Docker Compose environments across dev/staging/production parity

---

## 👩‍💻 Leadership & Mentoring

As the team grew to 10+ engineers, my role expanded beyond individual contribution:

**What I did with junior developers:**
- **Onboarding** — walked new joiners through codebase, architecture, and our module patterns
- **Task assignment & delivery management** — broke down features into clear tasks, assigned with context, tracked delivery
- **Pair programming** — sat with juniors on complex problems (multi-tenant bugs, payment integration edge cases)
- **PR reviews with teaching intent** — every review included not just "what's wrong" but "why" and "how to improve"
- **Migration conflict resolution** — personally resolved several complex conflicts caused by parallel junior contributions, then turned each incident into a team learning moment

**April 2024 Recognition:** Acknowledged for independently handling technical architecture decisions, DB schema and API design changes, production bug prioritization, and release management — without requiring senior involvement. This recognition reflected the trust the organization placed in my judgment across the full engineering lifecycle.

---

## 📈 Results & Impact

| Metric | Result |
|---|---|
| **Scale** | 100,000+ daily orders processed reliably across 2 countries |
| **Uptime** | Zero downtime during peak delivery hours via AWS Auto Scaling |
| **Settlement accuracy** | Full digital audit trail from cash collection to TRA receipt — eliminated manual disputes |
| **Agent visibility** | Real-time bonus progress via WebSocket — measurable improvement in field agent engagement |
| **Query performance** | Key dashboard queries: 4s → 50ms via indexing + materialized views |
| **Perishables** | Real-time cold-storage alerts reduced undetected temperature excursions |
| **Delivery efficiency** | TSP-optimized routes replaced 100% manual route planning |
| **Team growth** | Mentored and managed delivery of multiple junior developers across the project lifecycle |

---

## 💡 Lessons Learned

1. **Design for auditability from day one** — the cash collection workflow taught me that financial systems need immutable audit trails baked into the schema, not bolted on later
2. **Async everything that can wait** — moving bonus calculations and report generation to Celery was the highest-ROI change I made
3. **Schema-per-tenant is powerful but needs automation** — migrations must be scripted to run per-tenant; manual execution doesn't scale past 2 tenants
4. **Production rollback is a skill, not a failure** — having a practiced rollback procedure saved us from a 30-second API regression becoming a multi-hour outage
5. **IoT integrations need defensive coding** — sensors drop connections; always design for partial data, retries, and threshold-based alerting rather than raw value streaming
6. **Independent decision-making at senior level means owning consequences** — making architecture, schema, and release decisions without senior oversight taught me to think through second-order effects before committing

---

## 👩‍💻 Author

**Nandani Kanani** — Senior Software Engineer
[LinkedIn](https://linkedin.com/in/nandani-b-63a29018b) | [Email](mailto:pnandani126@gmail.com)

> *This repository documents the architecture, design decisions, and engineering challenges of a production system built at Moweb Technologies Pvt. Ltd. No proprietary code is shared in compliance with company policy.*
