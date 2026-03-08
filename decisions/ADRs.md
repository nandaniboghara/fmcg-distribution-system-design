# 📋 Architecture Decision Records (ADRs)

These are real decisions I made independently during the project — covering DB design, API contracts, async architecture, and deployment strategy.

---

## ADR-001: Schema-per-Tenant vs Row-Level Tenancy

**Status:** Accepted | **Decided by:** Me (independently)

### Context
Two countries needed isolated data with different currencies, tax rules, and payment gateways. I evaluated two approaches before committing to the schema design.

### Options

| | Row-Level Tenancy | Schema-per-Tenant |
|---|---|---|
| **Isolation** | Logical only — relies on `WHERE tenant_id` everywhere | Physical — DB enforces isolation |
| **Data leak risk** | High if any query misses the filter | Zero — wrong schema = empty results |
| **Query complexity** | Every query needs tenant filter | Clean queries, no tenant filter needed |
| **Migration ops** | Single migration run | Must run per tenant |
| **Cross-tenant reporting** | Easy JOIN | Application-level or `dblink` |

### Decision
**Schema-per-tenant.** For a financial system with cash collections, settlements, and tax receipts, I couldn't accept the data leak risk of row-level tenancy. The migration overhead was automated via a management command — acceptable tradeoff.

### Consequences
- Migrations automated: `python manage.py migrate_all_tenants`
- Cross-tenant reports handled at application layer only
- Adding a new country = create schema + run migrations — well-defined process

---

## ADR-002: Cursor-based vs Offset Pagination

**Status:** Accepted | **Decided by:** Me

### Context
At 100,000+ orders/day, the order list API was getting slow. Standard `LIMIT x OFFSET y` degrades severely at high offsets (PostgreSQL must scan and discard all preceding rows).

### Decision
**Cursor-based pagination** using `(created_at, id)` as composite cursor, base64-encoded for clients.

### Why
`OFFSET 50000` on a 5M-row table = scan 50,000 rows and discard them. Cursor jumps directly to the right row via index. Performance is O(1) regardless of page depth.

### Consequences
- No random page access — clients can't jump to page 500 (acceptable for mobile infinite scroll)
- Consistent sub-100ms list performance at any data volume
- Cursor is opaque to clients — implementation can change without breaking API contract

---

## ADR-003: Async Bonus Calculation via Celery

**Status:** Accepted | **Decided by:** Me

### Context
Bonus calculations for 100+ agents involve aggregating months of delivery data, applying multi-factor rules, and storing detailed logs. Running this synchronously in the API caused timeouts.

### Options
- **Synchronous in API** — simple, but 15–30s timeouts for large agents, bad UX, no retry on failure
- **Async Celery task** — triggered on delivery completion, runs in background, retriable

### Decision
**Async Celery.** Delivery completion API returns immediately (fast UX). Bonus calculation runs in background. Agents see real-time progress via WebSocket push on each task completion.

### Consequences
- Added Redis as a dependency (already used for caching — no new infra)
- Celery task must be idempotent — re-running for same period must produce same result
- `CELERY_TASK_ALWAYS_EAGER=True` in test settings for synchronous test execution

---

## ADR-004: JSONB for Bonus Rule Snapshots

**Status:** Accepted | **Decided by:** Me

### Context
Bonus rules change periodically (new country rules, adjusted multipliers). If we only store the calculated amount, we can't reconstruct *how* it was calculated after rules change — creating disputes.

### Decision
Store two JSONB columns on `agent_bonuses`:
- `calculation_log` — step-by-step breakdown of the calculation
- `rule_snapshot` — exact rules in effect at calculation time

### Consequences
- Any historical bonus is fully reconstructable regardless of current rules
- Dispute resolution is immediate — show the agent the exact log
- Storage overhead is acceptable vs. the operational cost of unresolvable disputes

---

## ADR-005: Payment Gateway Abstraction Layer

**Status:** Accepted | **Decided by:** Me

### Context
Tanzania uses Selcom. Adding Kenya would require a different gateway. I didn't want payment logic tightly coupled to a specific provider.

### Decision
Abstract base class `PaymentGateway` with a standard interface: `charge()`, `verify()`, `refund()`, `reconcile()`. Each gateway is a concrete implementation.

```
PaymentService
    └── PaymentGatewayFactory.get(country)
            ├── SelcomGateway(PaymentGateway)   ← Tanzania
            └── FutureGateway(PaymentGateway)   ← Kenya, etc.
```

### Consequences
- Adding a new country's gateway = one new class, zero changes to business logic
- Gateway responses normalized to internal `PaymentResult` object
- Selcom-specific error codes mapped to internal error taxonomy in the adapter layer
- Easy to mock in tests — inject `MockGateway` without touching real Selcom API

---

## ADR-006: Polling vs Webhook for IoT Cold Storage

**Status:** Accepted | **Decided by:** Me

### Context
IoT devices monitor cold storage temperature. We needed real-time visibility for perishable goods.

### Options
- **IoT device pushes to our API (webhook)** — real-time but requires IoT devices to know our API endpoint; complex firewall/network config in warehouse environments
- **We poll the IoT API (polling)** — simpler integration, we control the schedule, works regardless of IoT network config

### Decision
**Polling via Celery Beat every 5 minutes.** The IoT team's devices expose a simple read API. 5-minute polling frequency was sufficient for the alert SLA (warehouse staff response time is 15–30 min).

### Consequences
- Maximum alert latency = 5 minutes (acceptable for this use case)
- Polling task must handle device unavailability gracefully — log failure, don't crash
- If real-time requirements tighten in future, webhook pattern is the migration path

---

## ADR-007: Docker for Environment Parity

**Status:** Accepted | **Decided by:** Me (as part of deployment ownership)

### Context
"Works on my machine" bugs were causing staging surprises. With a 10+ person team, environment drift was a growing problem.

### Decision
Dockerize the full application stack. `docker-compose.yml` for local dev mirrors staging and production configuration exactly. CI builds and validates the Docker image on every PR.

### Consequences
- Onboarding new developers: `docker compose up` — fully running local environment in minutes
- Staging surprises dropped significantly post-Docker adoption
- Production deployments use the exact image validated in CI — no environment drift
- Rollbacks are image rollbacks — fast and reliable
