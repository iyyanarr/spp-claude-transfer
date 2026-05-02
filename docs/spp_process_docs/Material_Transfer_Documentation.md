# Material Transfer - Business & Technical Documentation

## Table of Contents

1. [Overview](#overview)
2. [Business Purpose](#business-purpose)
3. [Process Flow](#process-flow)
4. [Data Model](#data-model)
5. [Frontend (JavaScript)](#frontend-javascript)
6. [Backend (Python)](#backend-python)
7. [Related DocTypes](#related-doctypes)
8. [Error Handling & Rollback](#error-handling--rollback)

---

## Overview

| Attribute | Value |
|-----------|-------|
| **DocType Name** | Material Transfer |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `MT-.YYYY.-.MM.-.DD.-.#####` |
| **Submittable** | Yes |
| **Created** | 2022-08-22 |

The **Material Transfer** DocType manages the internal movement of raw materials (Batches, Compounds, Cut Bits) between warehouses. It is primarily used to move stock from the main store to operational areas like Mixing Centers or Sheeting Warehouses.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Shop Floor Discipline**: Enforces the scanning of barcodes for every movement, ensuring real-time inventory accuracy.
2.  **Process-Specific Logic**: Handles different types of transfers:
    *   **Transfer Batches to Mixing Center**: Moving raw ingredients (rubber, chemicals) for mixing.
    *   **Transfer Compound to Sheeting Warehouse**: Moving mixed compound for sheeting, often splitting stock into "Main" and "Cut Bit" portions.
    *   **Final Batch Mixing**: Moving materials for final stage mixing.
3.  **Cut Bit Management**: Automatically calculates and separates "Cut Bits" (leftover trim) from the main batch based on item-specific percentage configurations, creating separate stock entries for consumption and re-issue.
4.  **Traceability**: Maintains the chain of custody by linking Material Transfer documents to underlying Stock Entries and Delivery Notes.

### Key Features

*   **Barcode Integration**: Automated item lookup and validation via `scan_spp_batch_number`.
*   **FIFO Enforcement (Partial)**: Validates scanned batches against stock expiry and ensures legitimate stock presence.
*   **Auto-Stock Entry**: On submission, it automatically creates `Stock Entry` documents (Material Transfer, Repack, or Material Issue/Receipt types).
*   **Clip Mapping**: For sheeting transfers, it associates "Sheeting Clips" with the material for downstream tracking.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Transfer] --> B{Select Type}
    B -->|To Mixing Center| C[Scan Batches]
    B -->|To Sheeting| D[Scan Compound & Clips]
    C --> E{Validate Stock}
    D --> E
    E -->|Frontend Validation| F[Add Items to Table]
    F --> G[Submit Document]
    G --> H{Server Side Logic}
    H -->|To Sheeting| I[Create Repack Entry]
    I --> J[Stock Entry: Move Compound]
    I --> K[Stock Entry: Issue/Receipt Cut Bits]
    H -->|To Mixing| L[Create Material Transfer (Stock Entry)]
    L --> M{Warehouses Different?}
    M -->|Yes| N[Create Delivery Note]
    M -->|No| O[Finish]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `material_transfer_type` | Select | "Transfer Batches to Mixing Center", "Transfer Compound to Sheeting Warehouse", "Final Batch Mixing". |
| `transfer_date` | Date | The posting date for the stock movement. |
| `source_warehouse` | Link | Origin warehouse (Default: `U3-Store - SPP INDIA`). |
| `target_warehouse` | Link | Destination warehouse. |
| `scan_spp_batch_number` | Data (Barcode) | Main input field for scanning items. |
| `scan_location` | Data | Optional scan for target warehouse validation. |
| `batches` | Table | Child table listing items to transfer. |
| `stock_entry_ref` | Data | Reference to the created Stock Entry. |
| `cutbit_qty` | Float | Calculated quantity of cut bits (for Sheeting transfers). |
| `sheeting_clip` | Table MultiSelect | selected clips for the sheeting process. |

### Child Table: Material Transfer Item (`batches`)

| Field | Type | Purpose |
|-------|------|---------|
| `scan_barcode` | Data | The raw scanned barcode text. |
| `item_code` | Link | Item being transferred. |
| `batch_no` | Link | System Batch Number. |
| `spp_batch_no` | Data | Custom SPP Batch Number. |
| `qty` | Float | Quantity to transfer. |
| `is_cut_bit_item` | Check | Flag indicating if this row represents a cut bit. |
| `qc_template` | Link | Quality Inspection Template (if applicable). |

---

## Frontend (JavaScript)

**File**: `material_transfer.js` (670 lines)

### Event Handlers

#### 1. Barcode Scanning (`scan_spp_batch_number`)
*   Checks for duplicates in the current list.
*   Calls `validate_spp_batch_no` backend method.
*   **Logic**:
    *   If valid, adds row to `batches`.
    *   If type is **Sheeting**, it may calculate `cutbit_qty` and potentially add a second row for "Cut Bits" if they exist in the response.

#### 2. Warehouse Logic
*   `material_transfer_type` change events filter the `target_warehouse` query to show only relevant process warehouses (e.g., Mixing/Warming warehouses).

#### 3. Manual Entry
*   If `enter_manually` is checked, a secondary field `manual_scan_spp_batch_number` becomes visible, bypassing some UI restrictions but triggering the same backend validation.

#### 4. Cut Bit Handling
*   `use_cut_bit`: If checked, fetches available cut bits from the backend (`get_cutbit_items`) for selection.
*   Auto-calculates `cutbit_qty` based on the item's configured percentage.

---

## Backend (Python)

**File**: `material_transfer.py` (848 lines)

### Key Methods

#### `validate_spp_batch_no` (Whitelist)
Validates scanned items:
1.  **Search**: Looks up `Stock Entry Detail` by `mix_barcode` or `barcode_text`.
2.  **Stock Check**: Verifies the item exists in the `source_warehouse` and fetches the actual available batch quantity using `get_batch_qty` (custom logic mimics FIFO availability).
3.  **BOM Validation**: For non-Sheeting transfers, ensures a valid BOM exists for the item.
4.  **Compound Inspection**: (Legacy/Commented) Logic to check if previous inspections were passed.
5.  **Returns**: Item details, Qty, and `is_cut_bit_item` flag.

#### `on_submit`
Orchestrates the transfer based on type:
1.  **To Mixing Center**:
    *   If Source == Target: Creates standard `Stock Entry` (Material Transfer).
    *   If Source != Target: Creates `Delivery Note` (for inter-warehouse logistics).
2.  **To Sheeting Warehouse**:
    *   Calls `create_sheeting_stock_entry`.
    *   Validates specific Sheeting Clips.

#### `create_stock_entry`
Generates a "Material Transfer" Stock Entry.
*   **Native Logic**: Replicates the item table.
*   **Cut Bit Logic**: If transferring cut bits, it intelligently searches `Stock Ledger Entry` for available batches in the Cut Bit Warehouse, sorting by creation date (FIFO), and splitting the request across multiple batches if necessary.

#### `create_sheeting_stock_entry`
Complex logic for splitting compound into usage and re-issue:
1.  **Validation**: Ensures sufficient Cut Bit stock exists before processing.
2.  **Repack Entry**: Creates a `Stock Entry` of type "Repack".
    *   Source Side: Consumes the Compound and Cut Bits.
    *   Target Side: Produces the new "Warmed" compound with a new serial number (`generate_w_serial_no`).
3.  **Issue/Receipt**: (Likely handled in referenced logic split between functions) Moves the Cut Bit portion to the Sheeting Warehouse if applicable.

#### `validate`
*   Ensures posting date is not in future.
*   Enforces Sheeting Clips selection for sheeting transfers.
*   Calculates `cutbit_qty` based on item master percentage.

### Helper Functions

*   `generate_w_serial_no`: Creates unique serials for sheeting batches.
*   `get_cut_bit_rate`: Calculates percentage-based weight.
*   `create_delivery_note`: Alternative path for inter-warehouse transfers.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Stock Entry** | Created on submit to actualize movement |
| **Delivery Note** | Created if moving between physically distant warehouses |
| **Item** | Source of Cut Bit Percentage and Aging info |
| **Sheeting Clip Mapping** | Tracks clips used in sheeting process |
| **SPP Settings** | Stores execution configuration (Default Warehouses, Naming Series) |

---

## Error Handling & Rollback

The system uses `try...except` blocks in creation methods (`create_stock_entry`, `create_sheeting_stock_entry`):
1.  **Log Error**: Captures the traceback in `Error Log`.
2.  **Rollback**: `frappe.db.rollback()` is called to revert changes.
3.  **Cleanup**: Where applicable, `delete_stock_entry_safely` is used to remove partially created documents to prevent orphaned records.
