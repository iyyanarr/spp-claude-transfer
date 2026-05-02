# Blank Bin Inward Entry - Business & Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Business Purpose](#business-purpose)
3. [Process Flow](#process-flow)
4. [Data Model](#data-model)
5. [Frontend (JavaScript)](#frontend-javascript)
6. [Backend (Python)](#backend-python)
7. [Related DocTypes](#related-doctypes)
8. [Error Handling](#error-handling)

---

## Overview

| Attribute | Value |
|-----------|-------|
| **DocType Name** | Blank Bin Inward Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `BBIE-.#####` |
| **Submittable** | Yes |
| **Created** | 2023-01-28 |

The **Blank Bin Inward Entry** manages the return or inward movement of Blanking Bins. It primarily handles two scenarios: receiving used bins back into the blanking stock loop, or diverting bins containing reusable material to the Cut Bit Warehouse.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Asset Tracking**: Ensures precise tracking of Bin Assets as they return to the Blanking department.
2.  **Cut Bit Repurposing**: Provides a specific workflow ("Move To Cut Bit Warehouse") to identify material in bins that should be reclassified as "Cut Bits" for reprocessing, rather than standard stock.
3.  **Inventory Updates**: Automatically updates the `Item Bin Mapping` to reflect the current quantity and status of materials inside the bins.
4.  **Stock Movement**: Automates "Repack" entries when materials are diverted to the Cut Bit Warehouse.

### Key Features

*   **Dual Workflow**: unique checkbox `Move To Cut Bit Warehouse` toggles between standard inward (Asset Movement only) and reprocessing (Asset Movement + Stock Repack).
*   **Bin Validation**: Scans Bin Assets to verify they contain the expected items and batches.
*   **Batch Validation**: When moving to Cut Bits, enforces scanning a valid "Cut Bit Batch" to ensure traceability.

---

## Process Flow

```mermaid
flowchart TD
    A[Scan Bin] --> B{Validate Bin}
    B -->|Valid| C[Fetch Bin & Item Details]
    C --> D{Move to Cut Bit?}
    D -- No --> E[Standard Inward]
    E --> F[Enter Weight]
    F --> G[Submit]
    G --> H[Create Asset Movement (To Location -> From Location)]
    D -- Yes --> I[Scan Cut Bit Batch]
    I --> J[Valid Batch?]
    J -- Yes --> K[Enter Weight]
    K --> L[Submit]
    L --> M[Create Asset Movement (Release Bin)]
    L --> N[Create 'Repack' Stock Entry]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `blank_bin` | Data (Barcode) | Input field for scanning the Bin Asset. |
| `move_to_cut_bit_warehouse` | Check | Toggles the Cut Bit workflow. |
| `scan_cut_bit_batch` | Data (Barcode) | Input field for target Cut Bit Batch (if checked). |
| `bin_code` | Link (Asset) | The identified Bin Asset. |
| `item` | Link | The Item (Compound) inside the bin. |
| `spp_batch_number` | Data | Validation batch number. |
| `gross_weight_kgs` | Float | Measured Gross Weight. |
| `net_weight_kgs` | Float | Calculated Net Weight (Material only). |
| `items` | Table | List of processed bins. |
| `stock_entry_reference` | Long Text | ID(s) of generated Stock Entries. |

---

## Frontend (JavaScript)

**File**: `blank_bin_inward_entry.js` (207 lines)

### Event Handlers

#### 1. Bin Scanning (`blank_bin`)
*   Calls `validate_bin_barcode`.
*   Checks if the bin is already in the list.
*   If valid, populates Bin, Item, Batch, and Weight details.
*   Enables the "Add" button (unless "Move to Cut Bit" is checked, which requires an extra step).

#### 2. Cut Bit Batch Scanning (`scan_cut_bit_batch`)
*   Only active if `move_to_cut_bit_warehouse` is True.
*   Calls `validate_cutbit_batch_barcode`.
*   Verifies the scanned batch matches the Compound in the bin.

#### 3. Weight Validation (`gross_weight_kgs`)
*   Calculates `Net Weight`.
*   Validates:
    *   Net Weight $\le$ Available Stock.
    *   Gross Weight $>$ Bin Weight.

#### 4. Add Button
*   Validates presence of Bin, Weight, and (if applicable) Cut Bit Batch.
*   Adds row to `items` table and clears inputs.

---

## Backend (Python)

**File**: `blank_bin_inward_entry.py` (450 lines)

### Key Methods

#### `validate_bin_barcode` (Whitelist)
1.  **Query**: Fetches Asset and `Item Bin Mapping`.
2.  **Status Check**: Ensures the bin is *not* retired (i.e., it has active stock).
3.  **Stock Validation**: Compares Bin Qty against `Item Batch Stock Balance` to ensure system integrity.

#### `on_submit`
*   **Scenario A: Move to Cut Bit**
    *   Calls `make_stock_entry`.
*   **Scenario B: Standard Inward**
    *   Calls `asset_movement` for each item.

#### `make_stock_entry`
1.  **Stock Entry**: Creates a "Repack" entry.
    *   **Source**: Sheeting Warehouse (consumes original batch).
    *   **Target**: Cut Bit Warehouse (produces into Cut Bit Batch).
2.  **Asset Update**: Calls `asset_movement` with "release" type to retire the bin mapping.

#### `asset_movement`
1.  **Update Type**:
    *   `update`: Updates `Item Bin Mapping` qty.
    *   `release`: Marks `Item Bin Mapping` as retired (`is_retired=1`) and updates qty.
2.  **Movement**: Creates `Asset Movement` doc.
    *   **Standard**: Moves from "To Location" $\rightarrow$ "From Location" (Reverse logisitics).
    *   **Release**: Moves from "From Location" $\rightarrow$ "To Location".

### Helper Functions

*   `rollback_entries`: Manually reverts `Item Bin Mapping` changes and deletes Stock Entries if submission fails.
*   `validate_cutbit_batch_barcode`: Ensures the batch belongs to the correct Item.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Asset Movement** | Core logic for moving Bins. |
| **Stock Entry** | Used for Cut Bit Repacking. |
| **Item Bin Mapping** | Tracks content of the bins. |
| **Batch** | Validates Source and Target batches. |
| **SPP Settings** | Configures Warehouse and Location defaults. |

---

## Error Handling

*   **Atomic Rollback**: `on_submit` is wrapped in a try-except block that calls `rollback_entries` to attempt a clean state restoration upon failure.
*   **Constraint Checks**: Explicit checks for Bin Weight vs Gross Weight prevent negative inventory logic.
