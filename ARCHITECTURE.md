# System Architecture Specification (Deus Battery ERP)
> **Structured following IEEE 1016 SDD conventions**

---

## 1. Architectural Goals & Design Philosophy

The **Deus Battery ERP** architecture is engineered around deterministic state management, transactional integrity, electro-chemical safety, and defense-in-depth security. High-throughput battery assembly workshops and cell testing laboratories require immediate real-time responsiveness with zero risk of concurrency-induced stock anomalies or unauthorized access to protected financial records.

### Core Architectural Principles:
1. **Modular Monolith:** Presentation, business logic, AI services, and persistence interfaces are co-located within a single cohesive deployment unit to eliminate network partitioning overhead while enforcing strict internal service boundaries.
2. **Strict Referential Integrity:** Foreign keys enforce explicit deletion rules (`ON DELETE RESTRICT` for assembled battery packs and consumed sub-components; `ON DELETE SET NULL` for template references), guaranteeing permanent production genealogy.
3. **Database-Level Concurrency Isolation:** Critical inventory decrements and state transitions utilize pessimistic row-level locking (`SELECT ... FOR UPDATE`) and advisory locking (`pg_advisory_xact_lock`) in PostgreSQL.
4. **Single Active Session State Governance:** Database-backed session verification enforces single active login tokens per user across workshop terminals.
5. **Isolated Protected File Storage:** Sensitive payment receipts and backups reside strictly outside public static mounts (`data/uploads/receipts`), accessible only via authenticated API routes.

---

## 2. Layered System Decomposition

```mermaid
graph TD
    subgraph Presentation ["Client / Presentation Tier"]
        PWA["Responsive PWA Shell (Dark/Light Tokens, Resizable Modals)"]
        SW["Service Worker (Offline Shell & Cache-Busted Assets)"]
        Scanner["Camera Optical Scanner (Html5-QRCode)"]
        UX["100-Item Pagination, Unified Confirm Engine & Double-Click Guard"]
    end

    subgraph Transport ["Transport & Hardened Security Tier"]
        FastAPI["FastAPI REST Router"]
        OAuth["JWT Auth + Fail-Fast Production Guard"]
        SessionGuard["Single-Session Active Token Verifier"]
        RBAC["Role-Based Access Control Dependency Injection"]
        AuditMW["Audit Logging Interceptor"]
    end

    subgraph AI ["Embedded AI Tier"]
        AIRouter["AI Router (/api/v1/ai)"]
        AIService["AI Assistant Service (Self-Assignment & Context)"]
        ChatEnc["Fernet Encrypted Chat Store"]
    end

    subgraph Domain ["Domain & Service Tier"]
        ItemService["Battery Inventory & Customization Service"]
        BOMService["BOM v1.1 Assembly & Replenishment Engine"]
        OrderService["Order Dispatch, Services & Receipt Service"]
        FinService["Financial Valuation, Margin & Performer Engine"]
        AssignmentService["Asset Assignment Service (Issue/Return)"]
        RepairService["Maintenance & Balancing Service"]
        BackupService["System Backup & Client Download Service"]
        ExportService["Document & Spreadsheet Generation Service"]
    end

    subgraph Persistence ["Data Persistence Tier"]
        UnitOfWork["SQLAlchemy 2.0 + Alembic Migrations"]
        Locking["Row (FOR UPDATE) & Advisory Lock Manager"]
        PostgreSQL[("PostgreSQL 16 Relational Engine (15 Tables, ACID, WAL)")]
        FS["Isolated Storage (data/uploads/receipts & backups)"]
    end

    PWA --> SW
    Scanner --> PWA
    UX --> PWA
    SW --> FastAPI
    FastAPI --> OAuth
    OAuth --> SessionGuard
    SessionGuard --> RBAC
    RBAC --> AuditMW
    AuditMW --> AIRouter
    AIRouter --> AIService
    AIService --> ChatEnc
    ChatEnc --> UnitOfWork
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

The PostgreSQL 16 database is versioned via **Alembic** (`alembic/versions/`) and encompasses **15 normalized relational tables**:

```mermaid
erDiagram
    users ||--o{ audit_logs : "performed_by"
    users ||--o{ items : "responsible_person"
    users ||--o{ ai_chat_sessions : "owns"

    ai_chat_sessions ||--o{ ai_chat_messages : "contains (CASCADE)"

    items ||--o{ issue_return : "item_id (CASCADE)"
    items ||--o{ repairs : "item_id (CASCADE)"
    items ||--o{ write_offs : "item_id (CASCADE)"
    items ||--o{ purchases : "item_id (SET NULL)"

    bom_templates ||--o{ bom_items : "template_id (CASCADE)"
    bom_templates ||--o{ assembly_logs : "template_id (SET NULL)"
    bom_templates ||--o{ items : "assembly_template_id (SET NULL)"

    assembly_logs ||--|| items : "assembled_item_id (RESTRICT)"
    assembly_logs ||--|{ assembly_consumed_items : "assembly_log_id (CASCADE)"
    items ||--o{ assembly_consumed_items : "consumed_item_id (SET NULL)"

    orders ||--o{ order_services : "order_id (CASCADE)"

    users {
        int id PK
        string username UK
        string full_name
        enum role "admin, leader, worker"
        string active_session_token
        boolean is_active
        datetime created_at
    }

    items {
        int id PK
        string inventory_number UK
        string category_name
        string category_prefix
        enum accounting_type "PIECE, BULK"
        text name
        text model
        decimal quantity
        enum readiness_class "G0, G1, G2, G3"
        enum status "OPERATIONAL, INCOMPLETE, IN_REPAIR, DONOR, NEW, WRITTEN_OFF"
        enum ownership_type "GENERAL, PERSONAL"
        int assembly_template_id FK "SET NULL"
        decimal price
        int capacity_mah
        decimal voltage_nominal
        string cell_configuration
        string chemistry
        decimal internal_resistance_mohm
        int cycle_count
        decimal soh_percent
        decimal max_discharge_c
    }

    orders {
        int id PK
        string order_number UK
        string customer
        string status
        string assigned_to
        decimal cost_price
        decimal customer_price
        boolean is_paid
        text items_json
        datetime created_at
    }

    order_services {
        int id PK
        int order_id FK "CASCADE"
        string service_name
        string performer
        decimal cost
        boolean is_paid
        datetime created_at
    }

    bom_templates {
        int id PK
        string name UK
        string target_category_prefix
        text target_model
        datetime created_at
    }

    bom_items {
        int id PK
        int template_id FK "CASCADE"
        enum accounting_type
        string category_prefix
        string specific_inventory_number "VARCHAR(64)"
        decimal quantity
    }

    ai_chat_sessions {
        int id PK
        int user_id FK
        string title
        datetime created_at
    }

    ai_chat_messages {
        int id PK
        int session_id FK "CASCADE"
        string role
        text encrypted_content
        datetime created_at
    }
```

---

## 4. Concurrency Control & State Consistency

### 4.1. Pessimistic Row-Level Locking (`FOR UPDATE`)
All mutating inventory endpoints acquire exclusive row locks via `session.query(Item).filter(...).with_for_update()` inside an atomic transaction boundary, preventing negative stock and lost updates during simultaneous multi-bench cell deductions.

### 4.2. Transactional Advisory Locking for Sequence Generation
Sequential order identifiers (`ORD-YYYY-XXXX`) are serialized across concurrent transactions via PostgreSQL `SELECT pg_advisory_xact_lock(:sequence_lock_id)`.

### 4.3. Single Active Session Concurrency Guard
Every authentication event generates a fresh session token persisted on `users` and embedded in the signed JWT. Concurrent logins from a second terminal immediately invalidate the prior session token (`401 Unauthorized`), preventing shared-credential access.

---

## 5. Embedded AI Assistant & Encrypted Persistence (`app/ai/*`)

- **Authenticated Operator Context:** When an operator requests order creation via the AI assistant ("assign to me" or omitted assignee), the post-processing layer resolves `assigned_to = current_user.full_name` deterministically from the verified JWT identity.
- **Encrypted Chat History:** Conversations are persisted in `ai_chat_sessions` and `ai_chat_messages` using Fernet symmetric encryption, isolating dialog history at rest.

---

## 6. UI State Engineering & Database Versioning

1. **Alembic Schema Versioning:** Relational migrations are tracked under `alembic/versions/` with baseline and additive schema upgrades tested against production snapshots.
2. **Incremental Pagination & Collapsible Filters:** 100-row chunked pagination across operational registries paired with collapsible filter panels displaying live active-filter counts.
3. **Modal Persistence & Safety:** Modal dialogs support drag-resizing persisted per modal ID in `localStorage`, disable accidental backdrop-click dismissal, and coordinate via a unified promise-based confirmation controller.
4. **Double-Click Idempotency & WebKit OOM Mitigation:** Action buttons enforce in-flight locks; mobile viewports (`<= 768px`) render exclusive card layouts and unmount/hide inactive overlays.

---

## 7. Specifications Reference

- **[System Landing & Project Overview](README.md)**
- **[Traceability & Battery Lifecycle Specification](TRACEABILITY_AND_LIFECYCLE.md)**
- **[Security, Governance & Resilience Specification](SECURITY_AND_RELIABILITY.md)**
