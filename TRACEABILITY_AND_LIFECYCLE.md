# Battery Traceability, Electro-Chemical Telemetry & Lifecycle Specification (Deus Battery ERP)

---

## 1. Domain Scope & Inventory Classification

Manufacturing high-reliability battery packs for industrial robotics and battery platforms requires strict differentiation between raw electro-chemical cells, electronic BMS regulators, structural consumables, and fully serialized battery packs. **Deus Battery ERP** organizes inventory across standardized hardware domains, mapping each to a deterministic prefix and an accounting paradigm:

| Category Name | SKU Prefix | Accounting Type | Primary Inventory Classification |
|---|:---:|:---:|---|
| **Raw Battery Cells** | `CEL` | `BULK` | 21700 / 18650 Li-ion cells (Molicel, Samsung, LG, Panasonic) |
| **Battery Management Systems** | `BMS` | `BULK` | Smart balance boards, PCM protection circuits, CAN/SMBus telemetry |
| **Finished Battery Packs** | `BAT` | `PIECE` | Assembled multi-cell power units (e.g. 6S2P, 4S1P, 12S2P) |
| **Silicon Wiring & Leads** | `WIR` | `BULK` | High-temperature AWG silicone power wiring (10–18 AWG) |
| **Connectors & Hardware** | `CON` | `BULK` | Amass XT30, XT60, XT90, AS150 anti-spark, JST-XH balance leads |
| **Nickel Strip & Interconnects** | `NIC` | `BULK` | Pure nickel welding strips, copper-nickel sandwich busbars |
| **Insulation & Heat Shrink** | `INS` | `BULK` | PVC/PET heat shrink wrap, barley paper, Kapton polyimide tape |
| **Laboratory Tooling** | `TLS` | `BULK` | Precision spot welders, electronic balance chargers, internal resistance meters |
| **Testing Analyzers** | `EQP` | `PIECE` | Multi-channel programmable electronic DC loads, thermal imaging cameras |
| **General Workshop Ancillary** | `OTH` | `BULK` | Structural 3D printed end-caps, mounting brackets, silicone sealant |

---

## 2. Electro-Chemical Telemetry & Battery Health Models

Every serialized battery pack (`PIECE`) and discrete test batch records comprehensive physical, electrical, and degradation metrics:

```mermaid
graph TD
    subgraph ElectricalTelemetry [Electrical & Chemical Metrics]
        CAP["Capacity: capacity_mah (e.g., 8400 mAh)"]
        VOLT["Nominal Voltage: voltage_nominal (e.g., 22.2 V)"]
        CONF["Cell Topology: cell_configuration (e.g., 6S2P)"]
        CHEM["Chemistry: chemistry (e.g., Li-ion 21700, LiPo, LiFePO4)"]
        DISC["Discharge Rating: max_discharge_c (e.g., 35C / 45A)"]
    end

    subgraph DegradationTelemetry [Health & Degradation Tracking]
        IR["Internal Resistance: internal_resistance_mohm (e.g., 14.2 mΩ)"]
        CYC["Cycle Count: cycle_count (Charge/Discharge Cycles)"]
        SOH["State of Health: soh_percent (e.g., 96.5%)"]
        STAT["Operational Readiness: battery_status (Ready / Balancing / In Repair)"]
    end

    ElectricalTelemetry --> DegradationTelemetry
```

### 2.1. Parameter Definitions & Degradation Mechanics
- **Internal Resistance ($R_{\text{int}}$ in $m\Omega$):** Primary indicator of cell aging and thermal risk. Measured via 1kHz AC milliohm meter or DC delta-voltage method during acceptance testing. Values exceeding $150\%$ of baseline trigger automatic degradation to `G1` (Balancing/Maintenance) or `G2` (Cell Swap).
- **State of Health ($\text{SOH}\%$):** Calculated relative to factory rated capacity:
  $$\text{SOH} = \left(\frac{C_{\text{measured}}}{C_{\text{nominal}}}\right) \times 100\%$$
- **Cell Topology Notation:** Standardized series-parallel strings (`XS YP`):
  - *Example 6S2P:* 6 cells in series (nominal $6 \times 3.6\text{V} = 21.6\text{V}$, maximum $25.2\text{V}$), 2 strings in parallel doubling overall discharge capacity.

---

## 3. Dual-Ownership Accounting Paradigm

To accommodate master technicians fabricating personal equipment alongside commercial workshop throughput, the platform enforces architectural separation of ownership:

```mermaid
graph LR
    subgraph OwnershipModes [Ownership Paradigms]
        Corp["ZAHALNE (Company General Fund)"]
        Pers["OSOBYSTE (Master Personal Property)"]
    end

    subgraph CorpRules [Company Assets Governance]
        C1["Capitalized Corporate Inventory"]
        C2["Subject to Warehouse Financial Valuation"]
        C3["Eligible for Commercial Order Dispatch"]
        C4["Standard Centralized Assignment"]
    end

    subgraph PersRules [Personal Assets Governance]
        P1["Technician Personal Investment"]
        P2["Strictly Excluded from Company Balance Sheet"]
        P3["Ineligible for Commercial Orders (Requires Reclassification)"]
        P4["Dedicated Master Ownership Tag"]
    end

    Corp --> C1
    Corp --> C2
    Corp --> C3
    Corp --> C4

    Pers --> P1
    Pers --> P2
    Pers --> P3
    Pers --> P4
```

### Invariant Protection:
- **Financial Balance Protection:** Automated financial analytics services calculate warehouse balance strictly over items where `ownership_type == 'ZAHALNE'`. Personal units cannot artificially inflate corporate working capital.
- **Assembly Distinction:** In the BOM assembly engine, masters can explicitly designate an assembly run as personal via `is_personal = true`, creating the output asset with `ownership_type = 'OSOBYSTE'`.

---

## 4. Readiness Classification Finite State Machine (FSM)

Physical battery packs and sub-assemblies are governed by a multi-state finite state machine evaluating operational and safety readiness (`G0`–`G3`):

### 4.1. Readiness Classes (`readiness_class`)
- **`G0` (Operational):** Battery pack passes full acceptance testing: cell voltage delta $\Delta V \le 15\text{mV}$, internal resistance within factory nominal bounds, and physical insulation intact.
- **`G1` (Routine Servicing / Cell Balancing):** Pack functional but exhibits voltage drift ($\Delta V > 30\text{mV}$ between parallel groups), balance lead wire wear, or minor outer heat-shrink abrasions. Requires balancing charger service.
- **`G2` (Workshop Repair / Cell Group Replacement):** Pack unserviceable due to an underperforming parallel cell group, damaged BMS, or high internal resistance. Requires disassembly and rebuild.
- **`G3` (Retired / Donor Cells):** Pack degraded below $70\%\text{ SOH}$ or suffering physical impact damage. Disassembled strictly to salvage viable connectors, balance leads, or individual functional cells.

### 4.2. FSM State Transition Workflow

```mermaid
stateDiagram-v2
    [*] --> NEW: Cell Procurement & Initial Quality Control
    NEW --> G0_OPERATIONAL: Bench Acceptance Test (Capacity & IR Verified)
    
    state "G0: Operational" as G0_OPERATIONAL
    state "G1: Routine Balancing" as G1_BALANCING
    state "G2: Workshop Maintenance" as G2_REPAIR
    state "G3: Donor Storage" as G3_DONOR
    state "Scrapped / Disposed" as SCRAPPED

    G0_OPERATIONAL --> ISSUED: Assignment Check-out (Deployment)
    ISSUED --> G0_OPERATIONAL: Return (Nominal Voltage Balance)
    ISSUED --> G1_BALANCING: Return (Voltage Delta > 30mV)
    ISSUED --> G2_REPAIR: Return (Cell Failure / Over-discharge)
    
    G1_BALANCING --> G0_OPERATIONAL: Balancing Complete (Delta V <= 15mV)
    
    G2_REPAIR --> G0_OPERATIONAL: Group Swap Complete & Re-tested
    G2_REPAIR --> G3_DONOR: Irreparable Internal Damage
    
    G3_DONOR --> SCRAPPED: Component Stripping Complete
    SCRAPPED --> [*]
```

---

## 5. End-to-End Battery Pack Passport & Genealogy

Every assembled battery pack receives an append-only digital passport linking the finished product to its exact sub-components:

```mermaid
graph TD
    subgraph Pack [Assembled Battery Pack: BAT-0042]
        P["BAT-0042 (6S2P 8400mAh, Class: G0, Status: SPRAVNYI)"]
    end

    subgraph ConsumedComponents [Consumed Sub-Assembly Traceability]
        C1["Cells: 12x 21700 Molicel P42A (Batch: CEL-2026-Q3-01)"]
        C2["BMS: Smart BMS 6S 60A (SN: BMS-6S-9912)"]
        C3["Nickel: 0.2mm Pure Nickel Strip (Batch: NIC-0012)"]
        C4["Lead: XT90-S Anti-Spark Connector (Batch: CON-0084)"]
        C5["Wire: 12 AWG Silicon Wire 0.3m (Batch: WIR-0021)"]
        C6["Wrap: PVC Heat Shrink 85mm (Batch: INS-0045)"]
    end

    P --> C1
    P --> C2
    P --> C3
    P --> C4
    P --> C5
    P --> C6
```

### Traceability Applications:
- **Forward Impact Analysis (Cell Batch Recall):** If laboratory testing flags an entire inbound cell batch (`CEL-2026-Q3-01`) as susceptible to thermal degradation, an automated query identifies all assembled packs containing cells from that batch:
  $$\text{AffectedPacks} = \Pi_{\text{assembled\_item\_id}}(\sigma_{\text{consumed\_inventory\_number} = \text{'CEL-2026-Q3-01'}}(\text{AssemblyConsumedItem}))$$
- **Backward Forensic Analysis:** Following an operational power failure, technicians inspect the digital passport to extract the precise spot-welding technician, assembly date, initial internal resistance telemetry, and component supply invoices.

---

## 6. Production Request & Order Lifecycle (CRM FSM)

Inbound battery manufacturing and issue requests follow a deterministic Finite State Machine:

```mermaid
stateDiagram-v2
    [*] --> NEW: Requirement Registered
    NEW --> IN_PROGRESS: BOM Reserved & Bench Assembly Underway
    IN_PROGRESS --> READY: Pack Acceptance Tested & G0 Assigned
    READY --> ISSUED: Assignment Transferred & Payment Verified
    ISSUED --> [*]
```

### Operational Stages:
1. **`NEW`**: Registered requirement specifying target pack configuration, voltage, capacity, customer entity, target completion date, and priority tag (`LOW`, `NORMAL`, `URGENT`, `CRITICAL`).
2. **`IN_PROGRESS`**: Assigned to lead assembly technician. Sub-component BOM cell reservation is verified in inventory.
3. **`READY`**: Battery pack assembly complete, balancing test verified, prime cost (`cost_price`) calculated, and customer price confirmed.
4. **`ISSUED`**: Physical handover executed. Linked to payment verification (`is_paid = true`) and settlement confirmation.

---

## 7. Specifications Reference

- **[System Landing & Project Overview](README.md)**
- **[System Architecture Specification](ARCHITECTURE.md)**
- **[Security, Governance & Resilience Specification](SECURITY_AND_RELIABILITY.md)**
