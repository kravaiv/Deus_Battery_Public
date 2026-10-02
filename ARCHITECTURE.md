# System Architecture Specification (Deus Battery ERP)
> **Structured following IEEE 1016 SDD conventions**

---

## 1. Architectural Goals & Design Philosophy

The **Deus Battery ERP** architecture is engineered around the core principles of deterministic state management, transactional integrity, electro-chemical safety, and operational simplicity. High-throughput battery assembly workshops and cell testing laboratories require immediate real-time response times with zero risk of concurrency-induced stock anomalies (e.g., negative stock, double-spending of cells during parallel bench assemblies, or race conditions during order dispatch).

### Core Architectural Principles:
1. **Modular Monolith:** Presentation, business logic, domain services, and persistence interfaces are co-located within a single cohesive deployment unit to eliminate network partitioning overhead and latency while enforcing strict internal service boundaries.
2. **Strict Referential Integrity:** Foreign keys enforce explicit cascading and deletion rules (`ON DELETE RESTRICT` for assembled battery packs and consumed sub-components), guaranteeing that historical production records can never be orphaned or corrupted.
3. **Database-Level Concurrency Isolation:** Critical inventory decrementing and state transitions utilize row-level pessimistic locking (`SELECT ... FOR UPDATE`) in PostgreSQL, preventing "lost update" anomalies during simultaneous multi-bench operations.
4. **Single Active Session State Governance:** Database-backed session verification enforces single active login tokens per user, preventing account credential sharing across simultaneous workstations.
5. **Client-Agnostic REST Contracts:** Clear architectural decoupling between the backend API and the presentation layer, allowing standard web browsers, mobile PWA clients, and automated battery testing analyzers to consume the same secure endpoints.

---

## 2. Layered System Decomposition

The system is organized into four distinct, loosely coupled architectural tiers:

```mermaid
graph TD
    subgraph Presentation [Client / Presentation Tier]
        PWA["Responsive PWA Shell (Vanilla JS / CSS Grid)"]
        SW["Service Worker (Offline Shell & Asset Cache)"]
        Scanner["Camera Optical Scanner (Html5-QRCode)"]
        VirtList["Virtualized Chunked Renderer (35-Item Sentinel)"]
    end

    subgraph Transport [Transport & Security Tier]
        FastAPI["FastAPI REST Router"]
        OAuth["JWT Authentication Middleware (OAuth2 Bearer)"]
        SessionGuard["Single-Session Active Token Verifier"]
        RBAC["Role-Based Access Control Dependency Injection"]
        AuditMW["Audit Logging Interceptor"]
    end

    subgraph Domain [Domain & Service Tier]
        ItemService["Item & Battery Inventory Ledger Service"]
        BOMService["BOM Assembly Orchestration Engine"]
        OrderService["Order Dispatch & CRM Service"]
        FinService["Financial Valuation & Margin Engine"]
        AssignmentService["Asset Assignment Service (Issue/Return)"]
        RepairService["Maintenance & Balancing Service"]
        BackupService["System Backup & Recovery Service"]
        ExportService["Document & Spreadsheet Generation Service"]
    end

    subgraph Persistence [Data Persistence Tier]
        UnitOfWork["SQLAlchemy 2.0 Session / Unit of Work"]
        Locking["Row (FOR UPDATE) & Advisory Lock Manager"]
        PostgreSQL[("PostgreSQL Relational Engine (11 Tables, ACID, WAL)")]
        FS["File Storage (Receipts & Backups)"]
    end

    PWA --> SW
    Scanner --> PWA
    VirtList --> PWA
    SW --> FastAPI
    FastAPI --> OAuth
    OAuth --> SessionGuard
    SessionGuard --> RBAC
    RBAC --> AuditMW
    AuditMW --> ItemService
    AuditMW --> BOMService
    AuditMW --> OrderService
    AuditMW --> FinService
    AuditMW --> AssignmentService
    AuditMW --> RepairService
    AuditMW --> BackupService
    AuditMW --> ExportService
    ItemService --> UnitOfWork
    BOMService --> UnitOfWork
    OrderService --> UnitOfWork
    FinService --> UnitOfWork
    AssignmentService --> UnitOfWork
    RepairService --> UnitOfWork
    BackupService --> UnitOfWork
    ExportService --> UnitOfWork
    UnitOfWork --> Locking
    Locking --> PostgreSQL
    BackupService --> FS
    OrderService --> FS
```

---

## 3. Relational Schema & Entity-Relationship Model

The PostgreSQL database encompasses **11 normalized relational tables** enforcing strict relational constraints:

```mermaid
erDiagram
    users ||--o{ audit_logs : "performed_by"
    users ||--o{ items : "responsible_person"

    items ||--o{ issue_return : "item_id (CASCADE)"
    items ||--o{ repairs : "item_id (CASCADE)"
    items ||--o{ write_offs : "item_id (CASCADE)"
    items ||--o{ purchases : "item_id (SET NULL)"

    bom_templates ||--o{ bom_items : "template_id (CASCADE)"
    bom_templates ||--o{ assembly_logs : "template_id (SET NULL)"

    assembly_logs ||--|| items : "assembled_item_id (RESTRICT)"
    assembly_logs ||--|{ assembly_consumed_items : "assembly_log_id (CASCADE)"
    items ||--o{ assembly_consumed_items : "consumed_item_id (SET NULL)"

    users {
        int id PK
        string username UK
        string full_name
        enum role "admin, leader, worker"
        boolean is_active
        datetime created_at
    }

    items {
        int id PK
        string inventory_number UK
        string category_name
        string category_prefix
        enum accounting_type "SHTUCHNYI, HRUPOVYI"
        text name
        text model
        decimal quantity
        string unit
        enum readiness_class "G0, G1, G2, G3"
        enum completeness "KOMPLEKTNYI, NE_KOMPLEKTNYI"
        enum status "SPRAVNYI, DOUKOMPLEKT, V_REMONTI, DONOR, NOVYI, SPYSANYI"
        decimal min_quantity
        text storage_location
        text responsible_person
        enum ownership_type "ZAHALNE, OSOBYSTE"
        text serial_number
        decimal price
        int capacity_mah
        decimal voltage_nominal
        string cell_configuration
        string chemistry
        decimal internal_resistance_mohm
        int cycle_count
        decimal soh_percent
        decimal max_discharge_c
        string battery_status
        text note
        datetime created_at
        datetime updated_at
    }

    issue_return {
        int id PK
        int item_id FK "CASCADE"
        string operation_type
        decimal quantity
        text recipient
        text issued_by
        date planned_return_date
        date actual_return_date
        enum status_on_return
        text note
        datetime created_at
    }

    repairs {
        int id PK
        int item_id FK "CASCADE"
        date transfer_date
        text defect
        enum class_before_repair
        text performer
        string repair_status
        date completion_date
        enum class_after_repair
        decimal cost
        text note
        datetime created_at
    }

    write_offs {
        int id PK
        int item_id FK "CASCADE"
        date write_off_date
        decimal quantity
        text reason
        text basis_act
        text responsible_person
        text note
        datetime created_at
    }

    purchases {
        int id PK
        int item_id FK "SET NULL"
        date request_date
        text name
        text characteristics
        decimal quantity
        string unit
        string priority
        text initiator
        text supplier
        decimal price
        string status
        text note
        datetime created_at
    }

    audit_logs {
        int id PK
        int user_id FK "SET NULL"
        string username
        string action
        string entity_type
        int entity_id
        string entity_code
        text details
        datetime created_at
    }

    orders {
        int id PK
        string order_number UK
        string customer
        string order_type
        string status
        string priority
        date target_date
        date completed_date
        string assigned_to
        decimal cost_price
        decimal customer_price
        boolean is_paid
        boolean items_deducted
        text items_json
        text note
        datetime created_at
        datetime updated_at
    }

    bom_templates {
        int id PK
        string name UK
        string target_category_name
        string target_category_prefix
        text target_model
        text description
        datetime created_at
    }

    bom_items {
        int id PK
        int template_id FK "CASCADE"
        enum accounting_type
        string category_name
        string category_prefix
        text model_match
        string specific_inventory_number
        decimal quantity
        string unit
    }

    assembly_logs {
        int id PK
        int assembled_item_id FK "RESTRICT"
        int template_id FK "SET NULL"
        string assembler_name
        date assembly_date
        text storage_location
        text notes
        datetime created_at
    }

    assembly_consumed_items {
        int id PK
        int assembly_log_id FK "CASCADE"
        int consumed_item_id FK "SET NULL"
        string inventory_number
        text item_name
        decimal quantity
        string unit
        string accounting_type
        decimal unit_cost
    }
```

---

## 4. Concurrency Control & State Consistency

In a high-intensity battery pack assembly workshop, race conditions present critical operational risks (e.g., negative component inventory, double-allocation of cells, conflicting order numbering, or split-brain user sessions).

### 4.1. Pessimistic Row-Level Locking (`FOR UPDATE`)
To guarantee zero negative stock and eliminate "lost update" anomalies during simultaneous multi-bench cell deduction, the system implements **Pessimistic Row-Level Locking** via SQLAlchemy `with_for_update()` in PostgreSQL:

```python
# Atomic row locking pattern implemented across all mutating endpoints
with session.begin():
    item = (
        session.query(Item)
        .filter(Item.id == item_id)
        .with_for_update()  # Acquires exclusive PostgreSQL row-level lock (FOR UPDATE)
        .first()
    )
    if not item:
        raise HTTPException(status_code=404, detail="Entity not found")
        
    if item.accounting_type == AccountingTypeEnum.HRUPOVYI:  # BULK
        if item.quantity < requested_decrement:
            raise HTTPException(status_code=400, detail="Insufficient stock quantity")
        item.quantity -= requested_decrement
    else:
        # Atomic status mutation for piece-wise serialized assets
        if item.status != StatusEnum.SPRAVNYI:
            raise HTTPException(status_code=400, detail="Item is not available for operation")
        item.status = TargetStatusEnum.V_REMONTI
```

### 4.2. Transactional Advisory Locking for Sequence Generation
Sequential identifiers (such as order numbers `ORD-YYYY-XXXX`) require monotonic serialization across concurrent database transactions. Standard `SELECT MAX(...)` queries suffer from race conditions under load. The engine implements PostgreSQL transactional advisory locks:

```python
# Acquire advisory transaction lock scoped to sequence generation
db.execute(
    text("SELECT pg_advisory_xact_lock(:sequence_lock_id)"),
    {"sequence_lock_id": 10001}
)
# Safely compute next sequence number without duplicate key collisions
```

### 4.3. Single Active Session Concurrency Guard
To prevent credential sharing across multiple workshop terminals, every user authentication event generates a unique session token persisted to the user record and embedded in the signed JWT payload.

```mermaid
sequenceDiagram
    autonumber
    actor TechA as Technician (Terminal A)
    actor TechB as Technician (Terminal B - Concurrent)
    participant Auth as Auth Service
    participant DB as PostgreSQL Engine
    participant API as Protected Endpoint

    TechA->>Auth: Authenticate (credentials)
    Auth->>Auth: Generate unique session token
    Auth->>DB: Persist active session token for user
    Auth-->>TechA: Returns JWT with session token

    TechA->>API: Request protected resource (Bearer JWT with session token 1)
    API->>DB: Validate session token against database state
    DB-->>API: Valid (Match)
    API-->>TechA: 200 OK (Data Payload)

    Note over TechB, Auth: Technician logs in from Terminal B using same account
    TechB->>Auth: Authenticate (credentials)
    Auth->>Auth: Generate new session token
    Auth->>DB: Overwrite active session token for user
    Auth-->>TechB: Returns JWT with session token 2

    TechA->>API: Request protected resource (Bearer JWT with session token 1)
    API->>DB: Validate session token against database state
    DB-->>API: Invalid (Database contains session token 2)
    API-->>TechA: 401 Unauthorized (Session terminated from another device)
```

---

## 5. Bill of Materials (BOM) Assembly Execution Workflow

The assembly workflow coordinates sub-component deduction, battery pack instantiation, dynamic prime cost calculation, and append-only (application-level) audit linkage within a single ACID transaction:

```mermaid
sequenceDiagram
    autonumber
    actor Master as Assembly Master / Technician
    participant Router as Assembly Service
    participant LockMgr as Row-Level Lock Manager
    participant DB as PostgreSQL (Unit of Work)
    participant Audit as Audit Interceptor

    Master->>Router: Trigger Assembly Build (template_id, consumed_parts, warehouse, is_personal)
    Router->>DB: Begin immediate transaction
    
    loop For each consumed component in request
        Router->>LockMgr: SELECT * FROM items WHERE id = item_id FOR UPDATE
        LockMgr-->>Router: Acquired exclusive row lock
        Router->>Router: Verify readiness_class in (G0, NEW) and quantity >= required
        Router->>DB: Deduct quantity (BULK) or mutate status to CONSUMED (PIECE)
        Router->>Router: Accumulate prime cost += (quantity * unit_cost)
    end

    Router->>DB: INSERT INTO items (new assembled battery pack, price=prime_cost, ownership_type)
    Router->>DB: INSERT INTO assembly_logs (assembled_item_id, template_id, assembler_name, warehouse)
    
    loop For each consumed item
        Router->>DB: INSERT INTO assembly_consumed_items (assembly_log_id, consumed_item_id, qty, unit_cost)
    end

    Router->>Audit: Log ASSEMBLY_EXECUTED event with full component genealogy
    Router->>DB: Commit transaction (All locks released)
    Router-->>Master: 200 OK (Assembled Battery Pack Passport & New Token)
```

---

## 6. Specifications Reference

- **[System Landing & Project Overview](README.md)**
- **[Traceability & Battery Lifecycle Specification](TRACEABILITY_AND_LIFECYCLE.md)**
- **[Security, Governance & Resilience Specification](SECURITY_AND_RELIABILITY.md)**
