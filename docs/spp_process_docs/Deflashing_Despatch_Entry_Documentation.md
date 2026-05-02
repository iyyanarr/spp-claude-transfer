# Deflashing Despatch Entry - Business & Technical Documentation

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
| **DocType Name** | Deflashing Despatch Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `DDE-.#####` |
| **Submittable** | Yes |

The **Deflashing Despatch Entry** manages the transfer of moulded rubber products from the production floor to a "Deflashing Vendor" (internal or external). Deflashing is the process of removing excess rubber (flash) from the moulded parts.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Inventory Movement**: Moves stock from the `Moulding Warehouse` (Unit-2) to a specific `Vendor Warehouse`.
2.  **Quality Gate**: Ensures that **only** lots which have passed **Lot Inspection** can be dispatched.
3.  **Weight Verification**: Since the parts are moving to a vendor, the system verifies the weight again at dispatch.
    *   It calculates the theoretical weight (Pieces × Unit Weight).
    *   It captures the actual *Observed Weight*.
    *   If the difference exceeds **50 grams** (0.05 Kg), it triggers a "Weight Mismatch" alert and creates a tracker log.

### Key Features

*   **Vendor Scanning**: Scans a specific vendor/location barcode to set the `Target Warehouse`.
*   **Lot Validation**: Checks `Moulding Production Entry` and `Inspection Entry` status before allowing dispatch.
*   **Weight Tolerance**: Enforces strict weight checks to prevent pilferage or counting errors during handover.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Despatch] --> B[Scan Deflashing Vendor (Warehouse)]
    B --> C[Scan Lot Number]
    C --> D{Validate Lot}
    D -->|No Lot Inspection| E[Error: Complete Inspection First]
    D -->|Valid| F[Fetch Lot Details]
    F --> G[Enter Observed Weight]
    G --> H{Weight Diff > 50g?}
    H -->|Yes| I[Create Weight Mismatch Tracker]
    H -->|No| J[Add to List]
    J --> K[Submit]
    K --> L[Create Stock Entry (Transfer)]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `posting_date` | Date | Date of dispatch. |
| `scan_deflashing_vendor` | Data (Barcode) | Scanned ID of the receiving vendor/section. |
| `warehouse` | Data | Name of the target Vendor Warehouse. |
| `scan_lot_number` | Data (Barcode) | The Lot/Batch being transferred. |
| `stock_entry_reference` | Long Text | Linked Stock Entry ID(s). |
| `items` | Table | List of lots in this dispatch. |

### Child Table (`items`)

*   **DocType**: `Deflashing Despatch Entry Item`
*   **Fields**: `lot_number`, `batch_no`, `qty` (System Weight), `observed_weight`, `weight_difference`, `source_warehouse_id`.

---

## Frontend (JavaScript)

**File**: `deflashing_despatch_entry.js`

### Key Event Handlers

#### 1. Vendor Scanning (`scan_deflashing_vendor`)
*   Validates the vendor code against `SPP Settings` (allowed deflashing vendors).
*   Sets the `Target Warehouse` ID.

#### 2. Lot Scanning (`scan_lot_number`)
*   **Prerequisite Check**: Calls server to verify `Inspection Entry` (Lot Inspection) exists and is submitted.
*   **Validation**: Checks if `Moulding Production Entry` exists and if the lot is valid.
*   **Auto-Fetch**: key details like `Item`, `Batch No`, `Qty` (from Stock Entry), and `Source Warehouse`.

#### 3. Weight Verification (`show_observed_weight_dialog`)
*   Triggered when adding a lot.
*   Prompts user for **Observed Weight**.
*   Calculates `Difference = Observed - System`.
*   **Logic**:
    *   If `Abs(Difference) > 0.05 Kg` (50g): Shows a warning dialog. User must "Request Approval" (Creates `Weight Mismatch Tracker`) or cancel.
    *   If within tolerance: Adds item to the table.

---

## Backend (Python)

**File**: `deflashing_despatch_entry.py`

### Key Methods

#### `validate_lot_barcode`
*   Ensures the Lot isn't already dispatched.
*   Validation Chain:
    1.  `Job Card` (Completed)
    2.  `Moulding Production Entry` (Submitted)
    3.  `Line Inspection` & `Lot Inspection` (Submitted)
*   **Stock Check**: Verifies that the `qty` matches the current available batch quantity in the source warehouse.

#### `create_stock_entry` (`on_submit`)
*   **Purpose**: Official inventory transfer.
*   **Type**: `Material Transfer`.
*   **Source**: `Moulding Warehouse` (typically Unit-2).
*   **Target**: The scanned `Vendor Warehouse`.
*   **Status**: Submits the Stock Entry immediately.

#### `calculate_qty_in_nos`
*   Helper to reverse-calculate the number of pieces from weight.
*   Formula: `(Qty Kg / One Piece Weight)` where One Piece Weight comes from `Mould Specification`.
*   Used for reporting purposes (Despatch Note uses Kgs, but reports often need Nos).

---

## Advanced Logic

### Weight Mismatch Tracker
This is a separate logging system. If a dispatcher releases material that weighs significantly different from what the system says (e.g., system says 10kg, scale says 9.5kg), this event is logged in `Weight Mismatch Tracker`. This allows for audit trails on potential rubber theft or scale calibration issues without stopping the production flow entirely.

### Sub-Lot Logic
The system has provisions to handle "Sub-Lots" (`check_sublot`), allowing parts of a batch that were split or moved (e.g., via `Material Receipt`) to be dispatched correctly, keeping the lineage intact.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Inspection Entry** | Must be "Lot Inspection" completed. |
| **Moulding Prod. Entry** | Source of the Lot. |
| **Stock Entry** | Created to move stock to Vendor. |
| **Weight Mismatch Tracker** | Logs weight discrepancies. |
| **SPP Settings** | Defines valid Deflashing Vendors. |
