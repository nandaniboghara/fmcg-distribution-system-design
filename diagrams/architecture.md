# 🗺️ Architecture Diagrams

All diagrams use [Mermaid](https://mermaid.js.org/) — they render natively on GitHub. Paste into [mermaid.live](https://mermaid.live) to view locally.

---

## 1. High-Level System Architecture

```mermaid
graph TB
    subgraph Clients
        MA[📱 Mobile App\nDelivery Agents]
        WD[🖥️ Web Dashboard\nAdmin / Finance]
    end

    subgraph API["API Layer — Django REST Framework"]
        GW[API Gateway\nJWT Auth + Tenant Middleware\nRate Limiting + Versioning]
    end

    subgraph Services
        US[User & Auth]
        OS[Order & Delivery]
        IS[Inventory]
        PS[Payment & Settlement]
        BS[Bonus Engine]
    end

    subgraph Async["Async Layer"]
        CL[Celery Workers]
        RD[(Redis Broker)]
    end

    subgraph Data
        PG[(PostgreSQL\nSchema-per-Tenant)]
        S3[AWS S3\nProof / Receipts]
    end

    subgraph External
        IOT[🌡️ IoT Cold Storage\nPolled every 5min]
        TSP[🗺️ TSP Route Engine\nML Team]
        SEL[💳 Selcom Gateway]
        TRA[🧾 TRA Receipt API]
    end

    subgraph AWS
        EC2[EC2 Auto Scaling]
        RDS[RDS PostgreSQL]
        CW[CloudWatch]
    end

    MA -->|HTTPS| GW
    WD -->|HTTPS + WebSocket| GW
    GW --> US & OS & IS & PS & BS
    OS & IS & PS & BS --> PG
    OS --> S3
    BS --> RD --> CL
    CL --> PG
    CL -->|WebSocket push| WD
    OS --> TSP
    PS --> SEL & TRA
    CL --> IOT
    PG --> RDS
    EC2 --> CW
```

---

## 2. Order Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Draft : Sales agent creates
    Draft --> Confirmed : Agent confirms\n(credit check passed)
    Draft --> Cancelled : Agent cancels
    Confirmed --> Packed : Warehouse packs
    Confirmed --> Cancelled : Admin cancels
    Packed --> Dispatched : Assigned to delivery agent\n(TSP route attached)
    Dispatched --> Delivered : Agent marks delivered\n(proof captured)
    Dispatched --> Failed : Delivery failed
    Failed --> Dispatched : Re-assigned
    Delivered --> Settled : Cash collected\n& reconciled
    Settled --> [*]
    Cancelled --> [*]
```

---

## 3. Cash Collection & Settlement Workflow
*(Designed end-to-end by me)*

```mermaid
sequenceDiagram
    participant A as 🚚 Delivery Agent
    participant App as Mobile App
    participant API as Backend API
    participant S as 👤 Supervisor
    participant F as 💼 Finance

    A->>App: Collect cash from customer
    App->>API: POST /payments/collections/\n{amount, method, delivery_id}
    API-->>App: status: pending ✓

    S->>API: GET /payments/collections/?status=pending
    API-->>S: List of pending collections
    S->>API: POST /payments/collections/{id}/verify/
    API-->>S: status: verified ✓

    F->>API: GET /payments/gap-analysis/
    API-->>F: Credit / discount gap report
    F->>API: POST /payments/settlements/ (batch)
    API->>TRA: Generate TRA receipt
    TRA-->>API: receipt_no
    API-->>F: status: settled, tra_receipt_no ✓
```

---

## 4. Bonus Calculation Engine Flow
*(Designed end-to-end by me)*

```mermaid
sequenceDiagram
    participant D as Delivery Completed
    participant API as Backend API
    participant C as Celery Worker
    participant DB as PostgreSQL
    participant WS as WebSocket
    participant App as Agent Mobile App

    D->>API: POST /deliveries/{id}/complete/
    API->>DB: Update delivery status → delivered
    API->>C: Trigger bonus_recalculate task (async)
    API-->>D: 200 OK (immediate response)

    C->>DB: Fetch agent deliveries + rules snapshot
    C->>C: Evaluate rules\n(count × rate × multiplier)
    C->>DB: UPDATE agent_bonuses\n(amount + calculation_log)
    C->>WS: Push progress update to agent
    WS-->>App: 🔔 "You're at 78% of your bonus target!"
```

---

## 5. IoT Cold Storage Polling Architecture

```mermaid
sequenceDiagram
    participant CB as Celery Beat\n(every 5 min)
    participant C as Celery Worker
    participant IOT as IoT Device API
    participant DB as PostgreSQL
    participant WS as WebSocket
    participant D as Admin Dashboard

    CB->>C: Trigger poll_cold_storage task
    C->>IOT: GET /sensors/{warehouse_id}/readings
    IOT-->>C: {temp: 8.2°C, humidity: 85%}
    C->>DB: INSERT cold_storage_logs
    C->>C: Check threshold\n(safe: 2–6°C)
    alt Temperature breach
        C->>C: Trigger alert task
        C->>DB: UPDATE is_alert = true
        C->>WS: Push alert to dashboard
        WS-->>D: 🚨 "Warehouse TZ-03: Temp 8.2°C — exceeds safe range"
    end
```

---

## 6. Multi-Tenant Request Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant MW as Tenant Middleware
    participant DB as PostgreSQL

    C->>MW: Request + JWT\n{tenant_id: "tz", role: "finance"}
    MW->>MW: Decode JWT\nExtract tenant_id = "tz"
    MW->>DB: SET search_path TO tenant_tz, public
    MW->>DB: SELECT * FROM cash_collections\nWHERE status = 'pending'
    DB-->>MW: Rows from tenant_tz only\n(Kenya data physically isolated)
    MW-->>C: Response ✓
```

---

## 7. CI/CD & Deployment Pipeline

```mermaid
graph LR
    A[Git Push] --> B[Bitbucket PR]
    B --> C[CI Pipeline\npytest unit + integration\nDocker build check]
    C --> D{Tests pass?}
    D -->|No| E[PR blocked ❌]
    D -->|Yes| F[Code Review\nI reviewed all junior PRs]
    F --> G[Merge to main]
    G --> H[Deploy to Staging\nDocker Compose]
    H --> I[Smoke Tests\n+ Manual validation]
    I --> J{Go / No-Go\nmy decision}
    J -->|Go| K[Deploy to Production\nEC2 Auto Scaling]
    K --> L[CloudWatch monitors\nlatency + error rate]
    L --> M{Regression?}
    M -->|Yes| N[Rollback to\nprevious image ↩️]
    M -->|No| O[Release complete ✅]
```
