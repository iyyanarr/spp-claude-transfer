# Deflashing Receipt Entry - Business & Technical Documentation

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
| **DocType Name** | Deflashing Receipt Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `DRE-.#####` |
| **Submittable** | Yes |

The **Deflashing Receipt Entry** records the return of materials from a deflashing vendor (or internal section). It validates the weight of the received finished product and scrap, converting these weights back into quantities (Nos) to track yield and loss.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Inventory Recovery**: Moves stock back from the Vendor Warehouse to the `Unit-2` Warehouse (or designated FG warehouse).
2.  **Yield Analysis**: Calculates the efficiency of the deflashing process by comparing "Product Weight" vs. "Scrap Weight".
3.  **Manufacturing Completion**: Automatically creates and submits a `Work Order` and `Stock Entry (Manufacture)` to formally convert the "Moulded Item" (Raw Material) into the "Deflashed Item" (Finished Good).

### Key Features

*   **Barcode Validation**: Ensures the receiving lot matches a previously dispatched lot.
*   **Weight-to-Count Conversion**: Automatically calculates the number of pieces received (`Qty in Nos`) based on the standard weight per piece (from `UOM Conversion Detail`).
*   **Scrap Tracking**: Calculates expected vs. actual scrap to identify potential process issues or material theft.

---

## Process Flow

```mermaid
flowchart TD
    A[Scan Vendor] --> B[Scan Lot ID]
    B --> C{Validate Dispatch}
    C -->|Not Dispatched| D[Error: No Dispatch Entry found]
    C -->|Valid| E[Auto-fetch details matches]
    E --> F[Enter Product Weight]
    F --> G[Enter Scrap Weight]
    G --> H[Submit]
    H --> I[Create Work Order (Deflashing)]
    I --> J[Stock Entry (Manufacture)]
    J --> K[Update Job Cards]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `scan_deflashing_vendor` | Barcode | Identification of the vendor returning the goods. |
| `scan_lot_number` | Barcode | The lot being returned. |
| `product_weight` | Float | Actual weight of good finished parts (Kg). |
| `qty_in_nos` | Float | Calculated quantity in pieces (based on UOM). |
| `scrap_weight` | Float | Actual weight of waste material (Kg). |
| `stock_entry_reference` | Long Text | Link to the generated Manufacture Stock Entry. |
| `work_order_ref` | Data | Link to the generated Work Order. |

### Quantity & Scrap Tracking Fields

*   `qty_despatched_nos` vs. `qty_received_nos`: Tracks count discrepancies.
*   `total_scrap_expected_kg` vs. `actual_scrap_kg`: Tracks material usage efficiency.
*   `difference_kg_percentage`: Percentage deviation in scrap.

---

## Frontend (JavaScript)

**File**: `deflashing_receipt_entry.js`

### Key Features

#### 1. Lot Validation (`scan_lot_number`)
*   Calls `validate_lot_barcode` to ensure:
    *   The lot exists.
    *   It was previously dispatched (`Deflashing Despatch Entry`).
    *   It hasn't already been received.
*   Auto-populates: `Item`, `Batch`, `Qty`, `From Warehouse` (Vendor Warehouse).

#### 2. Percentage Styling
*   Highlights discrepancies (Difference % fields) in **Red** to alert the operator immediately if yield is poor.

---

## Backend (Python)

**File**: `deflashing_receipt_entry.py`

### Key Methods

#### `calculate_quantity_fields`
*   **UOM Conversion**: Fetches `UOM Conversion Detail` for the produced item to determine "Weight per Piece".
*   **Piece Calculation**: `Qty in Nos = (Product Weight / Weight per Piece) * 1000`.
*   **Blank Weight**: Fetches `avg_blank_wtproduct_gms` from `Mould Specification` to calculate expected yield from the original moulded blank.

#### `on_submit` -> `create_work_order`
*   Unlike standard ERPNext where a Work Order is made *before* production, here it is created **retroactively** upon receipt.
*   Creates a `Work Order` for the `Deflashed Item`.
*   Sets operations and stations based on `SPP Settings`.

#### `make_stock_entry`
*   Creates a **Manufacture** Stock Entry.
*   **Consumes**: The Moulded Item (from Vendor Warehouse).
*   **Produces**:
    1.  The Deflashed Item (to Unit-2 Warehouse).
    2.  Scrap Item (to Scrap Warehouse).
*   **Batching**: Generates a new Batch Number for the finished good.

#### `submit_inspection_entry` (Manual Submit)
*   **Integration**: If `Incoming Inspection` entries exist for this lot, it updates them with the final Batch Number and submits their linked Stock Entries.

### Error Handling
*   **Rollback Mechanism**: If the Work Order or Stock Entry creation fails, `rollback_entries` acts as a safety net to undo partial transactions and reset the document to Draft state.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Deflashing Despatch Entry** | Prerequisite. Must exist for the lot. |
| **Work Order** | Created automatically on submit. |
| **Stock Entry** | "Manufacture" entry created to update stock. |
| **UOM Conversion Detail** | Critical for converting Weight -> Nos. |
| **Mould Specification** | Source of "Blank Weight" for scrap calculation. |
