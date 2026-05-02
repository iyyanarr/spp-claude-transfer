# Moulding Production Entry - Business & Technical Documentation

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
| **DocType Name** | Moulding Production Entry |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `MLDPE-.#####` |
| **Submittable** | Yes |

The **Moulding Production Entry** is the central document for recording the output of the Moulding Process. It acts as the point where "Raw Material" (Compound in Bins) is consumed and "Finished Goods" (Moulded Products) are created in the system.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Production Booking**: Officially records the quantity produced (Weight) for a specific Job Card (Lot).
2.  **Consumption Traceability**: Links the consumed raw material batches (from "Blanking Bins") to the finished product, ensuring full forward/backward traceability.
3.  **Inventory Mass Balance**:
    *   **Consumption**: Deducts material from the `Sheeting Warehouse`.
    *   **Production**: Adds material to the `FG Warehouse`.
    *   **Scrap/Waste**: Accounts for "Purged Compound" and "Leakage" (specifically for Injection Moulding) and moves them to a Scrap warehouse.
4.  **Process Enforcement**:
    *   Ensures that **Line Inspection** and **Patrol Inspection** are completed *before* production can be booked.
    *   Validates that the weight of consumed bins matches the produced weight (plus tolerance).

### Key Features

*   **Barcode Scanning**: rapid entry of Job Card (Lot/Batch) and Supervisor/Operator IDs.
*   **Bin Balancing**: Users must scan the bins used. If a bin is only partially used, they scan it as a "Balance Bin" and weigh the remainder to calculate exact consumption.
*   **Immediate Submission**: Upon booking, the system *immediately* creates and submits the Stock Entry, updates the Work Order, and completes the Job Card.
*   **Inspection Trigger**: Automatically triggers the submission of pending "Rejection Stock Entries" for related Line/Patrol inspections once the batch is finalized.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Production Entry] --> B[Scan Supervisor & Operator]
    B --> C[Scan Job Card (Lot #)]
    C --> D{Validate Inspections}
    D -->|Missing Line/Patrol| E[Error: Complete Inspections First]
    D -->|Valid| F[Enter Produced Weight]
    F --> G[Enter Purge/Leakage (if Inj. Moulding)]
    G --> H[Scan Consumed Bins]
    H --> I{Partial Bin?}
    I -->|Yes| J[Weigh Balance Bin & Record Net Usage]
    I -->|No| K[Full Bin Consumed]
    K --> L[Validate Mass Balance]
    L --> M[Submit Entry]
    M --> N[Backend: Create Stock Entry (Manufacture)]
    N --> O[Update Work Order & Job Card]
    O --> P[Trigger Inspection Stock Entries]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `moulding_date` | Date | Posting date of production. |
| `scan_lot_number` | Data (Barcode) | The Job Card / Batch being processed. |
| `weight` | Float | The actual output weight (F/G). |
| `purged_compound` | Float | Material wasted during setup (Injection). |
| `compound_leakage` | Float | Material leaked/wasted (Injection). |
| `balance_bins` | Table | List of bins partially consumed. |
| `job_card` | Link | Ref to the production job. |
| `stock_entry_reference` | Data | Linked Stock Entry ID. |
| `is_injection_moulding` | Check | Auto-detected from Workstation name. |
| `batch_details` | Long Text (JSON) | Hidden field storing complex lot/bin JSON data. |

### Child Tables

*   `balance_bins` (`Moulding Balance Bin`): Tracks `bin_code`, `weight_of_balance_bin`, `net_weight` (used).

---

## Frontend (JavaScript)

**File**: `moulding_production_entry.js`

### Key Event Handlers

#### 1. Lot Scanning (`scan_lot_number`)
*   Calls `validate_lot_number` (Backend).
*   **Logic**:
    *   Populates Job Card details, Item, Compound.
    *   **Crucial**: Fetches "Bin Details" (which bins were issued to this job). Stores this in `batch_details` JSON for local validation logic.
    *   Visualizes "Weight Breakdown" using `bin_tracking_display` HTML field.

#### 2. Bin Scanning (`scan_bin`)
*   Used for **Balance Bins** (bins not fully consumed).
*   Validates that the scanned bin was actually issued to this Job Card.
*   Fetches current bin weight.

#### 3. Add Balance Bin (`add`)
*   Calculates `Net Weight` (Gross Weight - Empty Bin Weight is NOT how it works; it's `Gross Weight - Bin Tare` usually, but here logic calculates `Net Consumption` = `Original Qty` - `Remaining Weight`).
*   **Logic**: Updates the local `batch_details` JSON to reflect that this bin is "Partially Consumed".

---

## Backend (Python)

**File**: `moulding_production_entry.py`

### Key Methods

#### `validate_lot_number` (Whitelist)
*   **Prerequisite Check**: Queries `Inspection Entry` to ensure `Line Inspection` and `Patrol Inspection` exist for this lot. **Blocks production** if missing.
*   **Bin Fetching**: Retrieves all `Blank Bin Issue` items for this job to understand available material.

#### `validate_comsumption_details`
*   **Mass Balance**:
    *   Total Required = `Produced Weight` + `Rejection` + `Purge` + `Leakage`.
    *   Total Available = Sum of Consumed Bins.
*   **Algorithm**: Iterates through issued bins (FIFO or specific batch logic within `batch_details`) to allocate "Consumed" vs "Balance" quantities. Ensures `Required <= Available`.

#### `make_stock_entry`
*   **Core Execution**:
    1.  Generates a new **Batch No** (e.g., `T-LOT123...`).
    2.  Updates `Job Card` logs and `Work Order` operations.
    3.  Creates **Stock Entry (Manufacture)**:
        *   **FG Item**: Added to Target Warehouse.
        *   **Raw Material**: Added to Source Warehouse (deducted).
        *   **Scrap**: Adds `purged_compound` / `leakage` as Consumption items but marks them as scrap/waste.
    4.  **Submission**: Submits the Stock Entry *immediately*.

#### `submit_inspection_stock_entries_from_moulding`
*   **Integration**:
    *   Since Inspections (Line/Patrol) happen *before* the Batch # is known (it's generated here in Moulding), their Stock Entries (for rejections) are held in draft or pending state.
    *   This function finds those Inspection Entries, updates them with the newly generated `batch_no`, and **Submits** their Stock Entries.
    *   This ensures all rejections are booked against the finalized production batch.

#### `append_source_details`
*   Complex helper to construct the "Raw Material Consumption" rows for the Stock Entry.
*   Handles the logic of splitting batches if multiple bins of different batches were used.
*   Calculates specific "Purge" quantities to be deducted from the consumed batches.

---

## Error Handling

*   **Missing Inspections**: returns specific error "The following inspections are missing: Line Inspection...".
*   **Inventory Shortage**: `validate_actual_warehouse_stock` checks real-time warehouse balance before submission to prevent "Negative Stock" errors.
*   **Rollback**: `rollback_entries` provides a safe way to reverse the entire transaction (delete Stock Entry, reset Job Card) if a failure occurs mid-transaction.
