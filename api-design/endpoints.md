# 🔌 API Design Documentation

## Design Principles

- RESTful resource-based URLs
- URL versioning: `/api/v1/`, `/api/v2/`
- Consistent JSON response envelope on all endpoints
- Cursor-based pagination on all list endpoints
- JWT Bearer token authentication on all protected routes
- `X-Tenant-ID` header resolves DB schema per request

---

## Response Envelope

```json
{
  "status": "success | error",
  "data": { } | [ ],
  "message": "Human-readable message",
  "errors": null | { "field": ["detail"] },
  "meta": {
    "next_cursor": "base64encodedcursor==",
    "prev_cursor": "base64encodedcursor==",
    "count": 100
  }
}
```

---

## Authentication

```
POST   /api/v1/auth/login/              Obtain access + refresh tokens
POST   /api/v1/auth/token/refresh/      Rotate access token
POST   /api/v1/auth/logout/             Revoke refresh token
```

All protected routes:
```
Authorization: Bearer <access_token>
X-Tenant-ID: tz
```

---

## Orders API

```
GET    /api/v1/orders/                  List (cursor paginated, filterable)
POST   /api/v1/orders/                  Create draft order
GET    /api/v1/orders/{id}/             Order detail
PATCH  /api/v1/orders/{id}/             Update (role-restricted fields)
POST   /api/v1/orders/{id}/confirm/     Confirm — triggers credit check
POST   /api/v1/orders/{id}/dispatch/    Attach TSP route + assign agent
POST   /api/v1/orders/{id}/deliver/     Mark delivered, capture proof
POST   /api/v1/orders/{id}/cancel/      Cancel with reason
```

**Create Order:**
```json
POST /api/v1/orders/
{
  "customer_id": "uuid",
  "delivery_date": "2024-03-15",
  "items": [
    { "product_id": "uuid", "quantity": 10, "discount_pct": 5.0 }
  ]
}
→ { "status": "success", "data": { "id": "uuid", "status": "draft", "total_amount": "450.00", "currency": "TZS" } }
```

---

## Payment & Settlement API
*(Endpoints I designed and owned)*

```
GET    /api/v1/payments/collections/              List collections (filter by agent, status, date)
POST   /api/v1/payments/collections/              Log cash collection
POST   /api/v1/payments/collections/{id}/verify/  Supervisor verifies (role: supervisor only)
GET    /api/v1/payments/gap-analysis/             Credit / discount gap report
POST   /api/v1/payments/settlements/              Batch reconcile + generate TRA receipts
GET    /api/v1/payments/settlements/{id}/         Settlement detail
GET    /api/v1/payments/tra-receipts/{id}/pdf/    Download TRA receipt as PDF
```

**Log Cash Collection:**
```json
POST /api/v1/payments/collections/
{
  "delivery_id": "uuid",
  "amount_collected": "150.00",
  "collection_method": "cash"
}
→ { "status": "success", "data": { "id": "uuid", "status": "pending" } }
```

---

## Bonus API
*(Endpoints I designed and owned)*

```
GET    /api/v1/bonuses/                       List bonuses by period
GET    /api/v1/bonuses/{agent_id}/progress/   Real-time progress (WebSocket also available)
GET    /api/v1/bonuses/{id}/log/              Full calculation_log for transparency
POST   /api/v1/bonuses/calculate/             Trigger async recalculation (admin)
POST   /api/v1/bonuses/{id}/approve/          Approve for payment (finance)
```

**Progress Response:**
```json
GET /api/v1/bonuses/{agent_id}/progress/
{
  "status": "success",
  "data": {
    "deliveries_completed": 78,
    "deliveries_target": 100,
    "on_time_rate": 0.91,
    "current_bonus_amount": "45000.00",
    "currency": "TZS",
    "progress_pct": 78,
    "period_end": "2024-03-31"
  }
}
```

---

## Delivery & Route API

```
GET    /api/v1/deliveries/                    List deliveries
GET    /api/v1/deliveries/{id}/               Detail with route
POST   /api/v1/deliveries/{id}/complete/      Mark delivered + capture proof → triggers bonus Celery task
POST   /api/v1/deliveries/{id}/fail/          Mark failed with reason
GET    /api/v1/deliveries/routes/today/       Today's TSP-optimized routes per agent
POST   /api/v1/deliveries/routes/recalculate/ Re-trigger TSP for urgent new orders
```

---

## IoT & Cold Storage API

```
GET    /api/v1/iot/cold-storage/              Latest readings per warehouse
GET    /api/v1/iot/cold-storage/alerts/       Active unacknowledged temperature alerts
POST   /api/v1/iot/cold-storage/alerts/{id}/acknowledge/   Admin dismisses alert
GET    /api/v1/iot/cold-storage/history/      Historical readings with date range filter
```

---

## WebSocket Events

Real-time push via Django Channels:

| Event | Trigger | Recipient |
|---|---|---|
| `bonus.progress_updated` | Delivery completed → Celery task | Agent mobile app |
| `cold_storage.alert` | Temp threshold breached | Admin dashboard |
| `delivery.status_changed` | Agent updates delivery status | Admin dashboard |
| `order.status_changed` | Order state transition | Customer / sales agent |

---

## Error Reference

| Code | Meaning | Example |
|---|---|---|
| 400 | Validation error | Missing required field |
| 401 | Unauthenticated | Expired token |
| 403 | Unauthorized | Agent trying to verify collection |
| 404 | Not found | Order ID doesn't exist in tenant |
| 409 | State conflict | Order already confirmed |
| 422 | Business rule violation | Credit limit exceeded |
| 500 | Server error | Unexpected exception |
