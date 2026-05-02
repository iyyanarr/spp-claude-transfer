# Sub Lot Creation - Business & Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Business Purpose](#business-purpose)
3. [Process Flow](#process-flow)
4. [Data Model](#data-model)
5. [Backend Logic](#backend-logic)
6. [Frontend Logic](#frontend-logic)
7. [Related DocTypes](#related-doctypes)

---

## Overview

| Attribute | Value |
|-----------|-------|
| **DocType Name** | Sub Lot Creation |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `SUBL-.YYYY.-.#####` |
| **Submittable** | Yes |

The **Sub Lot Creation** DocType is used to split a parent lot into smaller sub-lots. This is often required during the finishing process when a batch is processed in smaller quantities or split across different workstations/operators.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Batch Splitting**: Allows a single large production batch (e.g., from Moulding) to be divided into smaller manageable units for finishing operations (Trimming, Printing, etc.).
2.  **Traceability**: Maintains a parent-child relationship between the original lot and the new sub-lots, ensuring full traceability of materials.
3.  **Inventory Segregation**: Physically moves the split quantity to a specific warehouse via a `Repack` Stock Entry.
4.  **Resource Tagging Integration**: Can optionally carry over "Resource Tagging" (LRT) data to the new sub-lot.

---

## Process Flow

```mermaid
flowchart TD
    A[Start] --> B[Scan Parent Lot No]
    B --> C{Validate Lot}
    C -->|Invalid| D[Error]
    C -->|Valid| E[Fetch Details]
    E --> F[Enter Split Qty]
    F --> G[Submit]
    G --> H[Create Sub-Lot Record]
    H --> I[Create Stock Entry (Repack)]
    I --> J[Generate New Barcode]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `scan_lot_no` | Data (Barcode) | The source lot being split. |
| `sub_lot_no` | Data | The newly generated sub-lot number (Auto-generated). |
| `qty` | Float | The quantity (in Nos/Kg) to be split into the new sub-lot. |
| `warehouse` | Link (Warehouse) | The warehouse where the sub-lot resides (Source & Target). |
| `item_code` | Link (Item) | The item being split. |
| `stock_entry_reference` | Link (Stock Entry) | Link to the "Repack" Stock Entry created. |
| `first_parent_lot_no` | Data | The original ancestor lot (Moulding Lot). |

---

## Backend Logic

**File**: `sub_lot_creation.py`

### Key Methods

#### `validate_lot` (Whitelist)
*   **Checks**:
    *   **Stock Availability**: Verifies `qty` in `Item Batch Stock Balance`.
    *   **Draft Checks**: Ensures no other draft documents (like `Despatch To U1`, `Inspection Entry`) exist for this lot, preventing concurrent modifications.
    *   **Parent Tracing**: Finds the `first_parent_lot_no` by tracing back through `Material Receipt` or `Deflashing Receipt` to the original `Moulding Production Entry`.

#### `make_repack_entry`
*   **Trigger**: `on_submit`
*   **Action**: Creates a **Stock Entry** of type `Repack`.
*   **Inventory Movement**:
    *   **Source**: Removes `qty` of Parent Batch from `warehouse`.
    *   **Target**: Adds `qty` of New Sub-Lot Batch (suffixed with `-1`, `-2` etc.) to the same `warehouse`.
*   **Barcode**: Generates a new barcode image for the `sub_lot_no`.

#### `update_sublot`
*   **Logic**: Generates the new sub-lot number naming convention.
    *   Format: `ParentLotNo-SequentialNumber`.
    *   Example: If Parent is `L123`, Sub-lot becomes `L123-1`.

#### `update_lrt`
*   **Purpose**: If the parent lot had "Lot Resource Tagging" (LRT) data, this function copies the *pending* operations to the new sub-lot, creating new `Lot Resource Tagging` documents automatically.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Lot Resource Tagging** | Can trigger creation of new tagging records. |
| **Stock Entry** | Created as "Repack" to manage inventory. |
| **Moulding Production Entry** | The ultimate ancestor for traceability. |
