# Receive Deflashing Entry - Business & Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Business Purpose](#business-purpose)
3. [Process Flow](#process-flow)
4. [Data Model](#data-model)
5. [Frontend (JavaScript)](#frontend-javascript)
6. [Backend (Python)](#backend-python)
7. [Advanced Logic](#advanced-logic)
8. [Related DocTypes](#related-doctypes)

---

## Overview

| Attribute | Value |
|-----------|-------|
| **DocType Name** | Receive Deflashing Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `RDPU-.#####` |
| **Submittable** | Yes |

> [!NOTE]
> Despite the name "Receive Deflashing", this DocType is primarily the **Receiving** counterpart to the `Despatch To U1 Entry`. It is used to acknowledge receipt of goods at the destination unit (Unit-1) that were sent from the source unit (Unit-2).

---

## Business Purpose

### What Problem Does It Solve?

1.  **Inter-Unit Receipt**: Completes the inventory transfer cycle started by `Despatch To U1 Entry`.
2.  **Quantity & Weight Verification**: Ensures that the goods physically received match what was dispatched.
3.  **Discrepancy Tracking**: Identifies and logs any weight or quantity mismatches (e.g., loss during transit) via `Weight Mismatch Tracker`.
4.  **Stock Regularization**: Moves stock from the "Transit" warehouse to the final "Store" warehouse.

### Key Features

*   **Despatch Integration**: Pulls all item details directly from the source `Despatch To U1 Entry`.
*   **Weight Tolerance Check**: Enforces a strict weight tolerance (configured in `SPP Settings`). If the observed weight deviates too much, it forces a mismatch workflow.
*   **One-Click Stock Entry**: Automatically generates `Material Transfer` stock entries for verified items.
*   **Discrepancy Reporting**: Built-in dialog to report items missing or not listed in the delivery challan (DC).

---

## Process Flow

```mermaid
flowchart TD
    A[Start Receipt] --> B[Enter Deflash Despatch No (DPU)]
    B --> C[Fetch DPU Details]
    C --> D[Scan Lot Number]
    D --> E[Weight Check Dialog]
    E --> F{Within Tolerance?}
    F -->|Yes| G[Mark as Received]
    F -->|No| H{Confirm Mismatch?}
    H -->|Yes| I[Create Weight Mismatch Tracker]
    H -->|No| J[Cancel Scan]
    G & I --> K[Add to Item Table]
    K --> L[Repeat for all lots]
    L --> M[Create Stock Entries]
    M --> N[Close/Submit Document]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `dd_number` | Data | Reference to the `Despatch To U1 Entry` (Source Document). |
| `receiving_warehouse` | Link (Warehouse) | The physical warehouse where goods are being received (e.g., U1-Store). |
| `scan_lot_no` | Barcode | Input field for scanning incoming lots. |
| `total_received_lots` | Int | Count of processed lots. |
| `received_stock_entry_ref` | Table MultiSelect | Links to all created Stock Entries. |
| `status` | Select | Open, Closed, Pending. |

### Child Table (`items`)

*   **DocType**: `Receive Deflashing Item`
*   **Fields**: `lot_no`, `product_ref`, `qty_nos`, `weight_kgs`, `received_weight`, `weight_difference`, `status` (Received/Miss Match), `stock_entry_status`.

---

## Frontend (JavaScript)

**File**: `receive_deflashing_entry.js`

### Key Features

#### 1. Fetching Despatch Info
*   User enters `dd_number`.
*   Call `get_despatch_info`.
*   Displays a summary (Vehicle No, Total Lots, etc.) and prepares the scanning interface.

#### 2. Weight Verification Dialog (`show_observed_weight_dialog`)
*   **Trigger**: Scanning a valid lot.
*   **Input**: User enters `observed_weight` and `observed_qty_nos`.
*   **Validation**:
    *   Calculates `weight_difference`.
    *   Fetches `tolerance` from `SPP Settings`.
    *   **Logic**:
        *   If `diff <= tolerance`: Accepted.
        *   If `diff > tolerance`: Prompts user.
            *   *Action*: Create `Weight Mismatch Tracker`.
            *   *Result*: Item added with status "Miss Match".

#### 3. Stock Entry Creation (`create_stock_entries`)
*   Filters for items with status "Received" or "Miss Match" that haven't been processed.
*   Calls backend `create_stock_entries`.
*   Updates references and status upon success.

#### 4. Discrepancy Reporting
*   Provides a "Report Discrepancy" button to file a `Discrepancy Report` for issues like "Items Not Found" or "Items Found But Not Listed".

---

## Backend (Python)

**File**: `receive_deflashing_entry.py`

### Key Methods

#### `create_stock_entries` (Ajax Method)
*   **Input**: List of items to process.
*   **Loop**: Iterates through each item.
*   **Action**: Calls internal `_create_stock_entry`.
*   **Stock Entry Details**:
    *   **Type**: `Material Transfer`.
    *   **Source**: `SPP Settings.p_target_warehouse` (The "Transit" warehouse where `Despatch To U1` left the goods).
    *   **Target**: `doc.receiving_warehouse` (The final destination).
    *   **Item**: Transfers the specific `batch_no` and `spp_batch_number` (Lot No).
*   **Post-Processing**: Updates the `Receive Deflashing Entry` with the list of created Stock Entries.

#### `get_despatch_info`
*   Validation wrapper to fetch `Despatch To U1 Entry` details safely.

---

## Advanced Logic

### Weight Mismatch Tracker integration
The system is designed to not block operations when weights don't match exactly. Instead, it "forks" the process:
*   **Perfect Match**: Standard Stock Transfer.
*   **Mismatch**: Creates a permanent audit record (`Weight Mismatch Tracker`) linked to the lot and dispatch, then proceeds with the transfer (often transferring the *actual* observed weight or system weight depending on business rules, though the code here suggests items are added to the table with observed details).

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Despatch To U1 Entry** | Source Document. |
| **Stock Entry** | Created document (Inventory Movement). |
| **Weight Mismatch Tracker** | Exception handling. |
| **Discrepancy Report** | Exception handling. |
| **SPP Settings** | Configuration (Tolerance, Transit Warehouse). |
