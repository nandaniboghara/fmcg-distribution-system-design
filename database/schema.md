# 🗄️ Database Schema Design

## Multi-Tenant Strategy

Each country gets its own **PostgreSQL schema**. The public schema holds shared config only.

```
postgres
├── public/         → tenants, countries, currencies, global config
├── tenant_tz/      → Tanzania: all business data isolated here
└── tenant_ke/      → Kenya: fully isolated, independent migrations
```

Tenant resolved from JWT on every request via Django middleware:
```python
# Middleware sets schema before every DB operation
connection.set_schema(tenant_id)  # → SET search_path TO tenant_tz, public
```

**Why schema-per-tenant over row-level tenancy:**
Row-level is simpler but requires `WHERE tenant_id = ?` on every query — easy to miss, risk of data leaks. Schema isolation is enforced at the DB level. For a financial system processing 100K+ orders/day, data isolation was non-negotiable.

---

## Core Tables

### users
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| email | VARCHAR(255) | unique per tenant |
| phone | VARCHAR(20) | |
| role | ENUM | admin, warehouse_manager, sales_agent, delivery_agent, finance_officer |
| country_id | FK | |
| is_active | BOOLEAN | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

---

### products
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| name | VARCHAR(255) | |
| sku | VARCHAR(100) | unique |
| category_id | FK | |
| base_price | DECIMAL(12,2) | |
| requires_cold_storage | BOOLEAN | triggers temp monitoring |
| config | JSONB | country-specific pricing rules, tax codes |
| is_deleted | BOOLEAN | soft delete |

---

### inventory
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| product_id | FK | |
| warehouse_id | FK | |
| quantity_available | INTEGER | |
| quantity_reserved | INTEGER | locked for confirmed orders |
| reorder_threshold | INTEGER | triggers low-stock alert |
| last_updated | TIMESTAMPTZ | |

**Materialized view** `inventory_summary` aggregates across warehouses.
Refreshed via Celery task on every inventory write — `CONCURRENTLY` to avoid read locks.

---

### orders
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| customer_id | FK | |
| sales_agent_id | FK | |
| status | ENUM | draft→confirmed→packed→dispatched→delivered→settled→cancelled |
| total_amount | DECIMAL(12,2) | |
| discount_amount | DECIMAL(12,2) | |
| credit_used | DECIMAL(12,2) | |
| currency | VARCHAR(3) | ISO 4217 |
| delivery_date | DATE | |
| created_at | TIMESTAMPTZ | |
| updated_at | TIMESTAMPTZ | |

**Indexes:**
```sql
CREATE INDEX idx_orders_status_date ON orders(status, delivery_date);
CREATE INDEX idx_orders_agent_created ON orders(sales_agent_id, created_at DESC);
```
These two indexes reduced dashboard query time from ~4s to ~50ms.

---

### order_items
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| order_id | FK | |
| product_id | FK | |
| quantity | INTEGER | |
| unit_price | DECIMAL(12,2) | **price at time of order** — never recalculated |
| discount_pct | DECIMAL(5,2) | |
| line_total | DECIMAL(12,2) | computed on write |

---

### deliveries
| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| order_id | FK | |
| agent_id | FK | |
| route_id | FK | TSP-generated route assignment |
| status | ENUM | assigned, in_transit, delivered, failed |
| scheduled_at | TIMESTAMPTZ | |
| delivered_at | TIMESTAMPTZ | |
| gps_lat | DECIMAL(9,6) | location at delivery confirmation |
| gps_lng | DECIMAL(9,6) | |
| proof_type | ENUM | signature, photo |
| proof_url | TEXT | S3 URL |

---

### cash_collections
*Designed end-to-end by me — core of the settlement workflow.*

| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| delivery_id | FK | |
| agent_id | FK | |
| amount_collected | DECIMAL(12,2) | |
| collection_method | ENUM | cash, mobile_money, selcom |
| status | ENUM | pending → verified → reconciled → settled |
| verified_by | FK → users | supervisor only |
| verified_at | TIMESTAMPTZ | |
| reconciled_by | FK → users | finance only |
| reconciled_at | TIMESTAMPTZ | |
| tra_receipt_no | VARCHAR(100) | Tanzania Revenue Authority compliance |
| dispute_note | TEXT | populated if discrepancy found |

**Design principle:** Each status transition is append-only and timestamped. No status can be skipped. Role-based access enforced at API layer — agents can only create, supervisors can only verify, finance can only reconcile.

---

### agent_bonuses
*Designed end-to-end by me — rules engine + calculation log.*

| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| agent_id | FK | |
| period_start | DATE | |
| period_end | DATE | |
| deliveries_completed | INTEGER | |
| on_time_rate | DECIMAL(5,2) | % deliveries on schedule |
| bonus_amount | DECIMAL(12,2) | final calculated amount |
| status | ENUM | calculating → calculated → approved → paid |
| calculation_log | JSONB | full step-by-step breakdown for transparency |
| rule_snapshot | JSONB | rules at calculation time — immutable record |

**Why JSONB for calculation_log:** Bonus rules change over time. Storing the rule snapshot alongside the calculation means we can always reconstruct exactly how a bonus was calculated, even after rules change — critical for dispute resolution.

---

### cold_storage_logs
*Partitioned by month — high-volume IoT polling data.*

| Column | Type | Notes |
|---|---|---|
| id | UUID | PK |
| warehouse_id | FK | |
| sensor_id | VARCHAR(50) | IoT device ID |
| temperature_c | DECIMAL(5,2) | |
| humidity_pct | DECIMAL(5,2) | |
| recorded_at | TIMESTAMPTZ | partition key |
| is_alert | BOOLEAN | true if outside safe threshold |
| alert_acknowledged | BOOLEAN | admin dismissed alert |

```sql
-- Monthly range partitioning
CREATE TABLE cold_storage_logs (...)
PARTITION BY RANGE (recorded_at);

CREATE TABLE cold_storage_logs_2024_03
    PARTITION OF cold_storage_logs
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

Older partitions archived and dropped without locking the parent table.

---

## Key Indexes Summary

```sql
-- Order dashboard (most critical — 100K+ rows)
CREATE INDEX idx_orders_status_date ON orders(status, delivery_date);
CREATE INDEX idx_orders_agent_created ON orders(sales_agent_id, created_at DESC);

-- Cash collection reconciliation
CREATE INDEX idx_cash_status_reconciled ON cash_collections(status, reconciled_at);
CREATE INDEX idx_cash_agent ON cash_collections(agent_id, created_at DESC);

-- Inventory lookups
CREATE INDEX idx_inventory_product_warehouse ON inventory(product_id, warehouse_id);

-- IoT time-series queries
CREATE INDEX idx_cold_logs_warehouse_time ON cold_storage_logs(warehouse_id, recorded_at DESC);

-- Delivery assignment
CREATE INDEX idx_deliveries_agent_date ON deliveries(agent_id, scheduled_at);
```

---

## Materialized Views

### inventory_summary
```sql
CREATE MATERIALIZED VIEW inventory_summary AS
SELECT
    product_id,
    SUM(quantity_available)   AS total_available,
    SUM(quantity_reserved)    AS total_reserved,
    COUNT(warehouse_id)       AS warehouse_count,
    MIN(quantity_available)   AS min_warehouse_stock
FROM inventory
GROUP BY product_id;

CREATE UNIQUE INDEX ON inventory_summary(product_id);
```

Refreshed via Celery task triggered on every inventory write:
```python
@app.task
def refresh_inventory_summary():
    with connection.cursor() as cursor:
        cursor.execute(
            "REFRESH MATERIALIZED VIEW CONCURRENTLY inventory_summary"
        )
```
`CONCURRENTLY` means reads are never blocked during refresh — safe at 100K+ orders/day scale.
