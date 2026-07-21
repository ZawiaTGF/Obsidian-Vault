---
tags:
  - client/golden-gm
  - erpnext/workflow
  - process-map
---


This document outlines the complete operational workflow for **Golden GM** using **ERPNext**. It details the steps required to manage raw material procurement, inventory control, manufacturing (production), and sales/financial fulfillment.

## 📊 High-Level Process Overview

```mermaid
flowchart TD

    subgraph P1 [Phase 1 - Procurement & Billing]
        MR[Material Request]
        PO[Purchase Order]
        SUP[Supplier Confirmation]
        PR[Purchase Receipt]
        PI[Purchase Invoice]
        PAY[Supplier Payment Entry]
        MR --> PO
        PO --> SUP
        SUP -->|Goods Delivered| PR
        PR --> PI
        PO -->|Reference| PI
        PI --> PAY
    end

    subgraph P2 [Phase 2 - Inventory & Warehousing]
        RM_WH[Raw Material Warehouse]
        WIP_WH[WIP Warehouse]
        FG_WH[Finished Goods Warehouse]
        RM_WH -->|Material Transfer| WIP_WH
    end

    subgraph P3 [Phase 3 - Manufacturing]
        BOM[Bill of Materials]
        WO[Work Order]
        SE_ISSUE[Stock Entry: Material Issue]
        JC[Job Cards]
        SE_MFG[Stock Entry: Manufacture]
        BOM --> WO
        WO --> SE_ISSUE
        SE_ISSUE --> JC
        JC --> SE_MFG
    end

    subgraph P4 [Phase 4 - Sales & Order Booking]
        SQ[Sales Quotation]
        SO[Sales Order]
        SQ --> SO
    end

    subgraph P5 [Phase 5 - Logistics & Delivery]
        PS[Packing Slip]
        DN[Delivery Note]
        GP[Gate Pass]
        POD[Proof of Delivery]
        PS --> DN
        DN --> GP
        GP --> POD
    end

    subgraph P6 [Phase 6 - Financial Settlement]
        SI[Sales Invoice]
        PE[Customer Payment Entry]
        SI --> PE
    end

    %% Cross-phase connections
    PR --> RM_WH
    WIP_WH --> SE_ISSUE
    SE_MFG --> FG_WH
    FG_WH -->|Stock Reserved| SO
    SO --> PS
    POD --> SI

    %% Styling by phase
    style P1 fill:#e3f2fd,stroke:#1565c0
    style P2 fill:#e8f5e9,stroke:#2e7d32
    style P3 fill:#f3e5f5,stroke:#6a1b9a
    style P4 fill:#e1f5fe,stroke:#0277bd
    style P5 fill:#fff8e1,stroke:#f57f17
    style P6 fill:#fff3e0,stroke:#e65100

    style MR fill:#bbdefb,stroke:#1565c0
    style PO fill:#bbdefb,stroke:#1565c0
    style SUP fill:#bbdefb,stroke:#1565c0
    style PR fill:#c8e6c9,stroke:#2e7d32
    style PI fill:#ffe0b2,stroke:#e65100
    style PAY fill:#ffe0b2,stroke:#e65100
    style RM_WH fill:#c8e6c9,stroke:#2e7d32
    style WIP_WH fill:#fff9c4,stroke:#f9a825
    style FG_WH fill:#c8e6c9,stroke:#2e7d32
    style BOM fill:#e1bee7,stroke:#6a1b9a
    style WO fill:#e1bee7,stroke:#6a1b9a
    style SE_ISSUE fill:#e1bee7,stroke:#6a1b9a
    style JC fill:#e1bee7,stroke:#6a1b9a
    style SE_MFG fill:#c8e6c9,stroke:#2e7d32
    style SQ fill:#b3e5fc,stroke:#0277bd
    style SO fill:#b3e5fc,stroke:#0277bd
    style PS fill:#fff9c4,stroke:#f9a825
    style DN fill:#fff9c4,stroke:#f9a825
    style GP fill:#fff9c4,stroke:#f9a825
    style POD fill:#ffe0b2,stroke:#e65100
    style SI fill:#ffe0b2,stroke:#e65100
    style PE fill:#ffe0b2,stroke:#e65100
```

## Phase 1: Raw Material Procurement & Billing

This phase covers requesting raw materials, issuing purchase orders to suppliers, receiving goods into stock, and creating purchase invoices (billing).

```mermaid
flowchart TD
    %% Roles & Nodes
    subgraph Procurement_Dept [Procurement Department]
        MR[Material Request]
        PO[Purchase Order]
    end

    subgraph External_Supplier [Supplier]
        SUP[Supplier Confirmation]
    end

    subgraph Warehouse_Dept [Warehouse / Receiving]
        PR[Purchase Receipt]
    end

    subgraph Finance_Dept [Accounts / Finance]
        PI[Purchase Invoice]
        PAY[Payment Entry]
    end

    %% Workflow Connections
    MR -->|Auto/Manual Trigger| PO
    PO -->|Send PO| SUP
    SUP -->|Acknowledge & Ship| PR
    PR -->|Updates Physical Stock| PI
    PO -->|Reference Doc| PI
    PI -->|Generates Accounts Payable| PAY

    %% Styling
    style MR fill:#e1f5fe,stroke:#0288d1
    style PO fill:#e1f5fe,stroke:#0288d1
    style PR fill:#e8f5e9,stroke:#388e3c
    style PI fill:#fff3e0,stroke:#f57c00
    style PAY fill:#fff3e0,stroke:#f57c00
```

### Detailed Steps:

1. **Material Request (MR):**
    
    - **User Role:** Warehouse Manager / Production Planner
        
    - **ERPNext DocType:** `Material Request` (Type: Purchase)
        
    - **Description:** Triggered when raw material stock levels fall below reorder thresholds or for a specific production run.
        
2. **Purchase Order (PO):**
    
    - **User Role:** Procurement Officer
        
    - **ERPNext DocType:** `Purchase Order`
        
    - **Description:** Generated from the Material Request and sent to the selected vendor/supplier with negotiated terms, unit prices, and expected delivery dates.
        
3. **Purchase Receipt (PR / GRN):**
    
    - **User Role:** Storekeeper / Warehouse Manager
        
    - **ERPNext DocType:** `Purchase Receipt`
        
    - **Description:** Logged when raw materials physically arrive. This updates the `Raw Material Warehouse` stock balance.
        
4. **Purchase Invoice (Bill Creation):**
    
    - **User Role:** Accounts Payable / Accountant
        
    - **ERPNext DocType:** `Purchase Invoice`
        
    - **Description:** The supplier's bill is recorded against the Purchase Receipt/Order. Creates an entry in Accounts Payable and logs tax/vendor balance.
        

## Phase 2: Inventory & Warehousing

Managing raw material storage, stock movements, and issuing goods to the production floor.

```mermaid
flowchart LR
    RM_WH[Raw Material Warehouse] -->|Stock Entry: Material Transfer| PROD_WH[Production / WIP Warehouse]
    PROD_WH -->|Stock Entry: Manufacture| FG_WH[Finished Goods Warehouse]

    style RM_WH fill:#e8f5e9,stroke:#388e3c
    style PROD_WH fill:#fff8e1,stroke:#ffa000
    style FG_WH fill:#e8f5e9,stroke:#388e3c
```

### Key Controls:

- **Batch & Serial Tracking:** Enable batch numbers for raw ingredients to maintain full traceability.
    
- **Warehouse Structure:**
    
    - `Stores / Raw Materials`
        
    - `Work In Progress (WIP) / Production Floor`
        
    - `Finished Goods Warehouse`
        

## Phase 3: Production / Manufacturing Workflow

This stage handles transforming raw materials into final packaged products (e.g., edible oil, consumer goods).

```mermaid
flowchart TD
    BOM[Bill of Materials - BOM] --> WO[Work Order]
    WO -->|Issue Materials| SE_ISSUE[Stock Entry: Material Issue to WIP]
    SE_ISSUE --> JC[Job Cards / Operations Tracking]
    JC --> SE_MFG[Stock Entry: Manufacture / Production Entry]
    SE_MFG --> FG[Finished Goods Stock Updated]

    style BOM fill:#f3e5f5,stroke:#7b1fa2
    style WO fill:#f3e5f5,stroke:#7b1fa2
    style JC fill:#f3e5f5,stroke:#7b1fa2
    style SE_MFG fill:#e8f5e9,stroke:#388e3c
```

### Detailed Steps:

1. **Bill of Materials (BOM):**
    
    - Multi-level recipe defining all raw materials, packaging supplies, and operational costs required to yield a standard unit of finished product.
        
2. **Work Order:**
    
    - Initiates production for a specified quantity of finished goods. Auto-reserves required raw materials.
        
3. **Job Cards:**
    
    - Tracks operator tasks, workstation allocation, machine usage, and labor time spent during manufacturing.
        
4. **Stock Entry (Manufacture):**
    
    - Deducts raw materials from `WIP Warehouse` and increments finished product inventory in `Finished Goods Warehouse`. Calculates actual cost of production.
        

## Phase 4: Sales & Order Booking

Capturing customer orders and planning fulfillment.

```mermaid
flowchart TD
    SQ[Sales Quotation] --> SO[Sales Order]

    style SQ fill:#e1f5fe,stroke:#0288d1
    style SO fill:#e1f5fe,stroke:#0288d1
```

### Detailed Steps:

1. **Sales Quotation:**
    
    - **User Role:** Sales Executive / Manager
    - **ERPNext DocType:** `Sales Quotation`
    - **Description:** Sent to prospective customers showing pricing and availability.
        
2. **Sales Order (SO):**
    
    - **User Role:** Sales Executive
    - **ERPNext DocType:** `Sales Order`
    - **Description:** Confirms customer agreement, items requested, delivery schedules, and payment terms. Auto-reserves finished goods stock if available.


## Phase 5: Logistics & Customer Delivery

Managing the physical packaging, dispatching, and transportation of products to the customer.

```mermaid
flowchart TD
    SO[Sales Order] --> PS[Packing Slip]
    PS --> DN[Delivery Note]
    DN --> GP[Gate Pass]
    GP --> POD[Proof of Delivery]

    style SO fill:#e1f5fe,stroke:#0288d1
    style PS fill:#e8f5e9,stroke:#388e3c
    style DN fill:#e8f5e9,stroke:#388e3c
    style GP fill:#e8f5e9,stroke:#388e3c
    style POD fill:#fff8e1,stroke:#ffa000
```

### Detailed Steps:

1. **Packing Slip:**
    
    - **User Role:** Warehouse Packer / Storekeeper
    - **ERPNext DocType:** `Packing Slip`
    - **Description:** Tracks individual packages, carton details, and specific quantities packed before shipment.
        
2. **Delivery Note (DN):**
    
    - **User Role:** Logistics Coordinator / Dispatch Officer
    - **ERPNext DocType:** `Delivery Note`
    - **Description:** Decrements stock from `Finished Goods Warehouse` upon shipment. Acts as the official dispatch document.
        
3. **Gate Pass:**
    
    - **User Role:** Security / Gatekeeper
    - **ERPNext DocType:** `Gate Pass` (Type: Outward)
    - **Description:** Authorizes transport vehicle loading and exit from factory/warehouse premises.
        
4. **Proof of Delivery (POD):**
    
    - **User Role:** Delivery Driver / Logistics Team
    - **ERPNext DocType:** `Delivery Note` (Attach signed POD receipt)
    - **Description:** Scanned copy of the physical receipt signed by the customer upon delivery, attached to the Delivery Note in ERPNext.


## Phase 6: Financial Settlement (Billing & Payment)

Generating invoicing and recording payments from customers.

```mermaid
flowchart TD
    DN[Delivery Note] --> SI[Sales Invoice]
    SI --> PE[Payment Entry]

    style DN fill:#e8f5e9,stroke:#388e3c
    style SI fill:#fff3e0,stroke:#f57c00
    style PE fill:#fff3e0,stroke:#f57c00
```

### Detailed Steps:

1. **Sales Invoice (SI):**
    
    - **User Role:** Accounts Receivable / Accountant
    - **ERPNext DocType:** `Sales Invoice`
    - **Description:** Generates the financial invoice, updates Accounts Receivable, and records Cost of Goods Sold (COGS).
        
2. **Payment Entry (PE):**
    
    - **User Role:** Cashier / Finance Accountant
    - **ERPNext DocType:** `Payment Entry`
    - **Description:** Logs incoming customer payment via bank transfer, cash, check, or credit.


## 🛠 Summary Table of ERPNext Document Mapping

|Stage|Process Step|Primary ERPNext DocType|Primary User / Role|
|---|---|---|---|
|**Procurement**|Material Request|`Material Request`|Warehouse Manager / Production Planner|
|**Procurement**|Purchase Order|`Purchase Order`|Procurement Manager|
|**Procurement**|Goods Receipt|`Purchase Receipt`|Storekeeper|
|**Billing**|Supplier Invoicing|`Purchase Invoice`|Accounts Payable|
|**Inventory**|Stock Issue / Transfer|`Stock Entry`|Warehouse User|
|**Manufacturing**|Product Recipe|`BOM`|Production Head|
|**Manufacturing**|Production Order|`Work Order`|Production Supervisor|
|**Sales**|Customer Order|`Sales Order`|Sales Executive|
|**Logistics**|Packing details|`Packing Slip`|Warehouse Packer|
|**Logistics**|Shipping/Dispatch|`Delivery Note`|Logistics Coordinator / Dispatch Officer|
|**Logistics**|Vehicle Exit|`Gate Pass`|Gatekeeper|
|**Finance**|Customer Invoicing|`Sales Invoice`|Accounts Receivable|
|**Finance**|Payment Settlement|`Payment Entry`|Accountant|

_Created for Golden GM ERPNext Implementation._