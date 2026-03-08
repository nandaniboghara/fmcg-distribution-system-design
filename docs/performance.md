# ⚡ Performance Optimizations

Real optimizations made on a system processing 100,000+ daily orders.

---

## 1. Order Dashboard: 4s → 50ms

**Problem:** Admin dashboard filtered orders by status and date. At 5M+ rows, queries were taking 3–8 seconds — full sequential scans.

**Fix:**
```sql
-- Before: Seq Scan, cost ~18,000
-- After:
CREATE INDEX idx_orders_status_date ON orders(status, delivery_date);
-- Index Scan, cost ~84
```

**Result:** ~4s → ~50ms. The single highest-ROI change I made on query performance.

---

## 2. N+1 Queries on Order List API: 51 queries → 3

**Problem:** Order list serializer accessed `order.customer` and `order.items[n].product` inside a loop. 50 orders = 51 DB round trips.

**Fix:**
```python
# Before
orders = Order.objects.filter(status='pending')

# After
orders = Order.objects.filter(status='pending').select_related(
    'customer', 'sales_agent'
).prefetch_related(
    'items__product'
)
```

**Result:** 51 queries → 3 queries. API response: ~2s → ~120ms.

---

## 3. Inventory Dashboard: 3.2s → 8ms

**Problem:** Inventory summary aggregated stock across all warehouses on every page load. Expensive `GROUP BY` on every request.

**Fix:** Materialized view refreshed async via Celery on inventory writes:

```sql
CREATE MATERIALIZED VIEW inventory_summary AS
SELECT product_id,
       SUM(quantity_available) AS total_available,
       SUM(quantity_reserved)  AS total_reserved
FROM inventory GROUP BY product_id;

CREATE UNIQUE INDEX ON inventory_summary(product_id);
```

```python
@app.task
def refresh_inventory_summary():
    with connection.cursor() as c:
        c.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY inventory_summary")
```

`CONCURRENTLY` — reads never blocked during refresh. Safe at our write volume.

**Result:** 3.2s → 8ms dashboard load.

---

## 4. Report Generation: 30s timeout → async + immediate response

**Problem:** Bonus and settlement reports aggregated months of data. Synchronous API calls timed out at 30s.

**Fix:** Async Celery task pattern:
```
POST /reports/generate/  →  { "task_id": "abc123" }   (immediate)
GET  /tasks/abc123/       →  { "status": "processing", "progress": 45 }
GET  /tasks/abc123/       →  { "status": "complete", "download_url": "..." }
```

**Result:** API response immediate. Reports reliable, retriable, never timeout.

---

## 5. IoT Logs: Consistent query performance over time

**Problem:** Cold storage logs grow at ~1,440 rows/sensor/day. Date-range queries degraded as table grew.

**Fix:** Monthly range partitioning:
```sql
CREATE TABLE cold_storage_logs (...) PARTITION BY RANGE (recorded_at);

CREATE TABLE cold_storage_logs_2024_03
    PARTITION OF cold_storage_logs
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

Date-range queries scan only the relevant partition. Old partitions archived and dropped without locking.

---

## 6. Peak Traffic: Zero downtime during delivery hours

**Problem:** Traffic spiked 4–5x during morning delivery hours (7–10am) as 100+ agents synced simultaneously.

**Fix:** AWS Auto Scaling Group:
```
CPU > 70% for 2 min  →  Scale OUT (add EC2 instance)
CPU < 30% for 10 min →  Scale IN  (remove instance)
Min: 2 instances | Max: 8 instances
```

**Result:** Zero downtime during peak hours. Cost-efficient off-peak (scales back to 2).

---

## Monitoring Approach

- **RDS slow query log** — all queries > 500ms logged and reviewed weekly
- **Django Debug Toolbar** in staging — catches N+1 before it hits production
- **`EXPLAIN ANALYZE`** — run on every new query touching tables > 100K rows before merge
- **CloudWatch dashboards** — API p95 latency, DB connection pool, CPU per instance, error rate
- **Latency spike alert** → triggers on-call notification → rollback if post-deploy regression
