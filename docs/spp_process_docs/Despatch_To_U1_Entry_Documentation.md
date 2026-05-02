# Despatch To U1 Entry - Business & Technical Documentation

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
| **DocType Name** | Despatch To U1 Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `DPU-.#####` |
| **Submittable** | Yes |

The **Despatch To U1 Entry** handles the transfer of finished/semi-finished goods from the "Unit-2" warehouse (Production) back to "Unit-1" (or another designated target warehouse). This typically happens after deflashing and final inspection.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Inter-Unit Transfer**: Facilitates the movement of goods between separate factory units (Unit-2 -> Unit-1).
2.  **Valuations Tracking**: Ensures the value of the goods (Valuation Rate * Qty) is correctly transferred between accounts.
3.  **Final Quality Check**: Validates that the lot has completed necessary preceding steps (like `Incoming Inspection` or `Deflashing Receipt`) before it can leave Unit-2.

### Key Features

*   **Vendor/Warehouse Validation**: Ensures that items being dispatched are coming from the correct "Vendor" warehouse where they were last processed (e.g., Deflashing Vendor).
*   **Batch Traceability**: Maintains full traceability by linking `Lot No`, `Batch No`, and `SPP Batch No`.
*   **Auto-Stock Entry**: Automatically creates a "Material Transfer" Stock Entry upon submission.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Despatch] --> B[Scan Lot Number]
    B --> C{Validate Lot}
    C -->|Not Found| D[Error]
    C -->|Found| E{Check Pre-requisites}
    E -->|No Incoming Inspection| F[Error: Complete Inspection]
    E -->|Valid| G[Fetch Lot Details]
    G --> H[Add to List]
    H --> I[Repeat for multiple lots]
    I --> J[Submit]
    J --> K[Create Stock Entry (Material Transfer)]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `posting_date` | Date | Date of despatch. |
| `vehicle_no` | Data | **Mandatory**. Vehicle number transporting the goods. |
| `scanned_lot_number` | Barcode | Input field for scanning. |
| `vendor` | Link (Warehouse) | The vendor/warehouse the goods are currently at. |
| `total_qty_nos` | Float | Sum of all items in numbers. |
| `total_qty_kgs` | Float | Sum of all items in Kg. |
| `stock_entry_reference` | Text | Link to the created Stock Entry. |

### Child Table (`items`)

*   **DocType**: `Despatch To U1 Entry Item`
*   **Fields**: `lot_no`, `product_ref`, `qty_nos`, `weight_kgs`, `batch_no`, `spp_batch_no`.

---

## Frontend (JavaScript)

**File**: `despatch_to_u1_entry.js`

### Key Event Handlers

#### `scan_lot_number`
*   **Server Call**: `validate_lot_number`
*   **Validations**:
    *   **Vendor Match**: Checks if the scanned lot belongs to the same Vendor as previously scanned lots (cannot mix vendors in one despatch).
    *   **Duplicate Check**: Prevents scanning the same lot twice.
*   **Auto-Population**: If valid, it populates `Product`, `Qty`, `Weight`, `Batch No`, etc., from the server response.

#### `add_html` (Add Button)
*   Adds the validated lot to the `items` child table.
*   Updates key totals (`total_qty_nos`, `total_qty_kgs`).
*   Resets the scan fields for the next entry.

---

## Backend (Python)

**File**: `despatch_to_u1_entry.py`

### Key Methods

#### `validate_lot_number`
*   **Checks Lineage**:
    1.  Looks for `Incoming Inspection Entry` (must be submitted).
    2.  If found, looks for `Deflashing Receipt Entry` to get stock details.
    3.  If not found there, checks `Job Card` path (for specific workflows).
    4.  Checks `Stock Entry Detail` to get specific `Batch No` and `Spp Batch No` used in previous steps.
*   **Stock Check**: Calls `check_available_stock` to ensure the item physically exists in the `From Warehouse`.
*   **UOM Conversion**: Calculates/Verify `Product Weight` from `Qty in Nos` using UOM Conversion factors.

#### `create_stock_entry` (`on_submit`)
*   **Purpose**: Creates the actual inventory movement.
*   **Type**: `Material Transfer`.
*   **Source**: `SPP Settings.unit_2_warehouse`.
*   **Target**: `SPP Settings.p_target_warehouse`.
*   **Address Logic**: Auto-assigns "Bill From/To" and "Ship From/To" addresses based on Factory 1 / Factory 2 presets.
*   **Validation**: Ensures `vehicle_no` is captured.

---

## Advanced Logic

### Sub-Lot Handling (`check_sublot`)
The logic includes a specifically robust `check_sublot` function. If a lot was split or transferred via `Material Receipt` (instead of standard production flow), this function traces the lot back through `Material Receipt` or `Deflashing Receipt` parents to find the correct origin and stock availability. This ensures that even "odd" flows (e.g., returned goods, split batches) can be dispatched correctly.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Incoming Inspection** | Mandatory prerequisite. |
| **Deflashing Receipt** | Source of stock data. |
| **Stock Entry** | Created document. |
| **SPP Settings** | Defines Source/Target warehouses. |
