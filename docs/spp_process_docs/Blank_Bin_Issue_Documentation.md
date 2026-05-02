# Blank Bin Issue - Business & Technical Documentation

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
| **DocType Name** | Blank Bin Issue |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `PE-.#####` |
| **Submittable** | Yes |

The **Blank Bin Issue** DocType manages the issuance of material (stored in "Blanking Bins") to a specific production job "Job Card". It acts as the "Material Issue" step in the manufacturing execution system (MES) for the Moulding process.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Material Traceability**: Links a specific physical bin of material (Compound/Mat) to a specific production run (Job Card/Lot). This ensures that the final product can be traced back to the raw material batch.
2.  **Validation**: Prevents errors by ensuring:
    *   The scanned Job Card is valid and in progress.
    *   The scanned Bin contains the *correct* Compound required by the BOM for the product being made.
    *   The Bin is active (not retired) and has weight.

### Key Features

*   **Barcode Scanning**: Efficiently captures "Job Card" (Lot Number) and "Bin" barcodes.
*   **BOM Verification**: Checks that the material in the bin matches the Bill of Materials (BOM) for the Work Order item.
*   **Asset Location Check**: Verifies the bin is present in the correct "From Location" (e.g., Blanking Store) before issue.

---

## Process Flow

```mermaid
flowchart TD
    A[Start Issue] --> B[Scan Job Card (Lot #)]
    B --> C{Validate Job Card}
    C -->|Invalid| D[Error Message]
    C -->|Valid| E[Scan Blank Bin]
    E --> F{Validate Bin}
    F -->|Mismatch/Empty| G[Error Message]
    F -->|Valid| H[Fetch Bin Details]
    H --> I[Validate Compound vs BOM]
    I -->|Match| J[Add to List]
    J --> K[Submit Issue]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `scan_production_lot` | Data (Barcode) | Input for scanning Job Card/Lot Number. |
| `scan_bin` | Data (Barcode) | Input for scanning Bin Asset. |
| `job_card` | Link (Job Card) | The identified production job. |
| `bin` | Link (Asset) | The identified material bin. |
| `production_item` | Link (Item) | Item being produced by the Job Card. |
| `compound` | Link (Item) | Material inside the scanned bin. |
| `bin_weight` | Float | Weight of material in the bin. |
| `qty_to_manufacture` | Float | Target quantity for the job. |
| `items` | Table | List of issued bins (Child Table: `Blank Bin Issue Item`). |

### Child Table (`items`)

*   **DocType**: `Blank Bin Issue Item`
*   **Fields**: `job_card`, `production_item`, `press`, `mould`, `bin`, `compound`, `bin_weight`, `spp_batch_number`.

---

## Frontend (JavaScript)

**File**: `blank_bin_issue.js`

### Event Handlers

#### 1. Scan Job Card (`scan_production_lot`)
*   Calls `validate_blank_issue_barcode` (Type: `scan_production_lot`).
*   Checks for duplicates in the current list.
*   Populates Job Card details (`press`, `mould`, `production_item`).
*   **Freezes** the scanned Job Card value to prevent change until the Bin is scanned.

#### 2. Scan Bin (`scan_bin`)
*   Calls `validate_blank_issue_barcode` (Type: `scan_bin`).
*   Checks for duplicates.
*   Populates Bin details (`weight`, `compound`, `asset_name`).
*   Triggers `add_values` or `append_values` to push data to the child table.

#### 3. Add / HTML Button
*   Provides a custom UI interactions to handle the sequential scanning workflow (Job Card -> Bin -> Add).

### Logic
*   The form enforces a specific sequence: **Job Card first, then Bin**.
*   Validates that the scanned Bin matches the Job Card requirements before adding the row.

---

## Backend (Python)

**File**: `blank_bin_issue.py`

### Key Methods

#### `validate_blank_issue_barcode` (Whitelist)

**Type: `scan_production_lot`**
1.  **Status Check**: Ensures Job Card status is "Work In Progress".
2.  **Asset Fetch**: Resolves `mould_reference` to `Item Code`.

**Type: `scan_bin`**
1.  **Bin Query**: Fetches details from `tabAsset` and `tabItem Bin Mapping`.
2.  **Availability Check**:
    *   `is_retired`: Must be 0 (Active).
    *   `bin_weight`: Must be > 0.
3.  **Location Check**: Verifies Bin is in the `to_location` defined in `SPP Settings`.
4.  **Material Validation**:
    *   Calls `validate_compound`.
    *   Checks if `bin.compound` matches any component in the **BOM** of the `production_item`.

#### `validate_compound`
*   Queries the **BOM** of the `production_item`.
*   Iterates through BOM Items (`tabBOM Item`) to find a match for the Bin's `compound`.
*   Returns `True` if a valid match is found, ensuring only correct materials are used.

#### `validate`
*   Re-constructs the child table (`items`) from the header fields immediately before save to ensure data consistency.
*   Enforces mandatory fields (`scan_production_lot`, `scan_bin`).

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Job Card** | The work order/lot receiving the material. |
| **Asset (Bin)** | The container holding the material. |
| **Item Bin Mapping** | Real-time content of the bin. |
| **BOM** | Used to validate material compatibility. |
| **SPP Settings** | Configures default locations. |

---

## Error Handling

*   **Invalid Status**: Prevents issue to Job Cards that are not WIP.
*   **BOM Mismatch**: Explicit error if the Compound in the bin is not in the Product's BOM.
*   **Wrong Location**: Prevents issuing bins that are not physically in the correct staging area.
