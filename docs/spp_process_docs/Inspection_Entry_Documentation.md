# Inspection Entry - Business & Technical Documentation

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
| **DocType Name** | Inspection Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `INSP-.#####` |
| **Submittable** | Yes |
| **Key Types** | `Lot Inspection`, `Line Inspection`, `Patrol Inspection`, `Incoming Inspection` |

The **Inspection Entry** is the universal quality control document. It handles three distinct critical checks:
1.  **Line Inspection**: Station-based check during production.
2.  **Patrol Inspection**: Roving Inspector check during production.
3.  **Lot Inspection**: Final verification of the entire produced lot/batch.
4.  **Incoming Inspection**: Verification of items returning from external deflashing vendors.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Multi-Stage Quality Control**: Ensures products are checked at multiple stages (Machine, Roving, Post-Production Lot).
2.  **Stock Release Gatekeeper**: The system enforces a **"Two-Key" security model**. Rejection Stock Entries are only submitted when *both* **Line Inspection** AND **Lot Inspection** are completed for a batch. This prevents premature inventory movements.
3.  **Rejection Accountability**: Tracks rejections by weight (Kgs) to maintain inventory mass balance.
4.  **Traceability**: Every inspection is linked to a specific `Job Card` (Batch Code), enabling full defect traceability to the operator and machine.

### Key Features

*   **Integrated Workflow**: "Lot Inspection" relies on "Line Inspection" data and vice versa for stock validation.
*   **Unit Conversion**: Automatically converts "Inspected Pieces" to "Kilograms" using `Mould Specification` weights, ensuring accurate inventory deduction.
*   **Defect Standardization**: Uses a common "Defect Code" list (Bubbles, Short Fill, Flow Marks) across all inspection types.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Inspection] --> B{Select Type}
    B -->|Line/Patrol| C[During Production Check]
    B -->|Lot Inspection| D[Post-Production Check]
    C --> E[Scan Inspector & Lot]
    D --> E
    E --> F[Enter Inspected Qty (Nos)]
    F --> G[System Calcs Weight (Kgs)]
    G --> H[Enter Rejections (Nos)]
    H --> I[Submit Entry]
    I --> J{Check Completion}
    J -->|Line + Lot Done?| K[Yes: Trigger Rejection Stock Entry]
    J -->|Only one done?| L[Wait in Queue]
    K --> M[Move Rejection to Scrap Warehouse]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `inspection_type` | Select | "Lot Inspection", "Patrol Inspection", "Line Inspection". |
| `lot_no` | Data (Barcode) | The Job Card / Batch Code. |
| `inspected_qty_nos` | Float | **Mandatory**. Quantity checked in pieces. |
| `total_inspected_qty` | Float | Calculated quantity in Kgs. |
| `total_rejected_qty_kg` | Float | Total rejection weight to be deducted. |
| `stock_entry_reference` | Link | The generated Stock Entry (Rejection). |
| `batch_no` | Link | The final Batch Number (linked after Moulding Entry submission). |

### Child Tables

*   `items` (`Inspection Entry Item`): breakdown of specific defects (e.g., "Air Bubble - 5 Nos").

---

## Frontend (JavaScript)

**File**: `inspection_entry.js`

### Key Event Handlers

#### 1. Lot Scanning (`scan_production_lot` / `lot_no`)
*   **Validation**: Checks if an inspection of this *Type* already exists for this *Lot*. Prevents duplicates.
*   **Zero Logic**: For Lot Inspection, `inspected_qty_nos` MUST be entered.
*   **Weight Calc**: `inspected_qty_nos` * `one_no_qty_equal_kgs` (fetched from backend) = `total_inspected_qty`.

#### 2. Rejection Entry
*   User enters "Rejected Nos" in the child table against defect types.
*   Script calculates "Rejected Kgs" row-by-row and updates the document header totals.

---

## Backend (Python)

**File**: `inspection_entry.py`

### Key Methods

#### `validate_lot_number`
*   **Cross-Check**: Verifies the Lot exists in `Job Card`.
*   **Duplicate Check**: Ensures you don't create two "Lot Inspections" for the same batch code.

#### `on_submit`
*   **Incoming Inspection Logic**:
    *   Checks if `total_rejected_qty_kg > 0`. If yes, calls `make_inc_stock_entry` to create a Rejection Stock Entry (Unit-2 -> Rejection Warehouse).
    *   **Trigger**: Calls `submit_deflash_receipt_entry` to finalize the linked `Deflashing Receipt Entry`.
*   **Line/Lot/Patrol Logic ("Two-Key")**:
    1.  Checks if `total_rejected_qty_kg > 0`. If so, calls `make_stock_entry` (Creates *Draft* Stock Entry).
    2.  **Trigger Condition**: Queries if **Line Inspection** AND **Lot Inspection** are now both submitted for this `lot_no`.
    3.  If `Count >= 2` (meaning both are done):
        *   Calls `submit_inspection_stock_entry_immediately`.

#### `submit_inspection_stock_entry_immediately`
This function orchestrates the final inventory movement:
1.  **Fetches Moulding Entry**: Finds the parent `Moulding Production Entry` to get the real **Batch Number** (e.g., "T-1234").
2.  **Updates All**: Loops through *all* inspections (Line, Patrol, Lot) for this job.
3.  **Assigns Batch**: Updates their draft Stock Entries with the real Batch Number.
4.  **Submits**: Submits all pending Rejection Stock Entries.
    *   *Result*: Rejections are deducted from inventory *only* when the production cycle is fully verified (Moulding + Line + Lot).

---

## Advanced Logic

### Why wait for both inspections?
*   **Accuracy**: "Line Inspection" happens during the shift. "Lot Inspection" happens after.
*   **Inventory Safety**: Deducting stock based on just one opinion might be premature. The system waits for the "Lot Inspection" (Final Check) to confirm the total rejection count before finalizing the inventory deduction for the entire batch.

### Unit Conversion
*   **Formula**: `(Avg Blank Weight + Shell Weight) / 1000` = `Weight per Piece (Kg)`.
*   This factor is stored in `one_no_qty_equal_kgs` and is vital for converting "5 Rejected Pieces" into "0.450 Kg" of stock deduction.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Job Card** | The parent document defining the Lot. |
| **Moulding Production Entry** | The source of the Batch Number. |
| **Stock Entry** | The financial document moving Rejected Stock. |
| **Mould Specification** | Source of 'Weight per Piece' data. |
