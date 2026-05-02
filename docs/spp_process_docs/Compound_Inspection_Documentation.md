# Compound Inspection - Business & Technical Documentation

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
| **DocType Name** | Compound Inspection |
| **Module** | Shree Polymer Custom App |
| **Naming Rule** | `CM-INSP-.#####` |
| **Submittable** | Yes |
| **Created** | 2023-07-12 |

The **Compound Inspection** DocType is used to perform quality control checks on mixed compounds. It validates critical parameters (Hardness, Specific Gravity, TS2, TC 90) against defined standards and ensures the compound has undergone sufficient aging (maturation) before being released for production.

---

## Business Purpose

### What Problem Does It Solve?

1.  **Quality Assurance**: Ensures every batch of compound meets the strict physical property specifications defined in the `Quality Inspection Template`.
2.  **Maturation Control**: Enforces mandatory aging periods for compounds. If a compound is used before it has fully matured, it requires special approval.
3.  **Deviation Management**: Allows production to proceed with "out-of-spec" or "under-aged" compounds only if authorized by specific approvers (Quality Approver or Maturation Approver).
4.  **Traceability**: Links the inspection to the specific Stock Entry and Delivery Challan Receipt, creating a full audit trail from mixing to acceptance.

### Key Features

*   **Barcode Integration**: Scanning employee badges and compound labels for quick data entry and validation.
*   **Automatic Parameter Fetching**: Retrieves min/max acceptable values for Hardness, SG, etc., from the Item Master.
*   **Conditional Approvals**:
    *   **Maturation Approver**: Required if the compound hasn't aged enough.
    *   **Quality Approver**: Required if any observed parameter conflicts with the standard range.

---

## Process Flow

```mermaid
flowchart TD
    A[Scan Employee ID] --> B{Valid Inspector?}
    B -->|Yes| C[Scan Compound Barcode]
    B -->|No| Z[Error]
    C --> D{Validate Stock & DC Receipt}
    D -->|Valid| E[Fetch Item & Standards]
    D -->|Invalid| Z
    E --> F[Enter Observed Readings]
    F --> G{Check Aging}
    G -->|Not Enough Aging| H[Require Maturation Approver]
    F --> I{Check Parameters}
    I -->|Out of Spec| J[Require Quality Approver]
    H --> K[Submit Document]
    J --> K
    K --> L[Validate all Approvals]
    L --> M[Update Stock Entry Posting Date]
    M --> N[Create Quality Inspection (Accepted)]
    N --> O[Link QI to Compound Inspection]
```

---

## Data Model

### Main Fields

| Field | Type | Purpose |
|-------|------|---------|
| `scan_employee` | Data (Barcode) | ID of the employee performing the inspection |
| `scan_compound` | Data (Barcode) | Barcode of the compound batch being inspected |
| `compound_ref` | Link → Item | The Compound Item Code |
| `batch_no` | Data | SPP Batch Number |
| `qty` | Float | Quantity being inspected |
| `stock_id` | Data | Reference to the Stock Entry |
| `dc_receipt_id` | Data | Reference to the Delivery Challan Receipt |
| `posting_date` | Date | Date of inspection |
| `cavity_no` | Int | Cavity number used for testing |
| `qc_inspection_ref` | Data | Reference to the created Quality Inspection document |

### Validation Flags & Approvals

| Field | Type | Purpose |
|-------|------|---------|
| `no_enough_aging` | Check | Auto-set if aging check fails |
| `scan_approver` | Data (Barcode) | Maturation Approver's ID (Required if `no_enough_aging`) |
| `no_enough_hardness` | Check | Auto-set if Hardness is out of spec |
| `no_enough_sg` | Check | Auto-set if SG is out of spec |
| `no_enough_ts2` | Check | Auto-set if TS2 is out of spec |
| `no_enough_tc90` | Check | Auto-set if TC 90 is out of spec |
| `scan_quality_approver` | Data (Barcode) | Quality Approver's ID (Required if any parameter fails) |

### Quality Parameters

| Parameter | Min Field | Max Field | Observed Field |
|-----------|-----------|-----------|----------------|
| **Hardness** | `min_hardness` | `max_hardness` | `hardness_observed` |
| **Specific Gravity** | `sg_min` | `sg_max` | `sg_observed` |
| **TS2** | `ts2_min` | `ts2_max` | `ts2_observed` |
| **TC 90** | `tc_90_min` | `tc_90_max` | `tc_90_observed` |

*Note: If a Quality Approver is required, they must also provide their own independent readings (e.g., `q_hardness_observed`).*

---

## Frontend (JavaScript)

**File**: `compound_inspection.js` (208 lines)

### Event Handlers

#### 1. Inspector Validation (`scan_employee`)
Validates that the scanned employee has the designation allowing "Compound Inspector" operations via `validate_inspector_barcode`.

#### 2. Compound Scanning (`scan_compound`)
When a compound barcode is scanned:
*   Calls `validate_compound_barcode` backend method.
*   Populates `compound_ref`, `batch_no`, `qty`, `stock_id`, `dc_receipt_id`.
*   Fetches and sets parameter min/max values (`set_parameters`).

#### 3. Approver Scanning
*   `scan_approver`: Validates "Compound Maturation Approver" designation.
*   `scan_quality_approver`: Validates "Compound Inspection Approver" designation.

### UX Logic
*   **Hide/Show Info**: `hide_show_info` toggles visibility of the parameter section based on whether a compound has been scanned.
*   **Custom Button**: Adds a "View Quality Inspection" button if a QC Inspection reference exists.

---

## Backend (Python)

**File**: `compound_inspection.py` (398 lines)

### Key Methods

#### `validate_compound_barcode` (Whitelist)
Checks the validity of the scanned compound:
1.  Ensures `Compound Inspection` doesn't already exist for this barcode.
2.  Fetches `Stock Entry Detail` to find the parent Stock Entry.
3.  Ensures Stock Entry is not Cancelled.
4.  Checks the source `Delivery Challan Receipt` status (must be Submitted).
5.  Fetches `Quality Inspection Template` from the Item Master to retrieve parameter standards.

#### `validate_inspector_barcode` (Whitelist)
Verifies if the scanned employee is Active and maps to the required SPP Process designation (e.g., "Compound Inspector", "Compound Maturation Approver").

#### `validate`
*   Ensures all mandatory observations are entered.
*   **Aging Check**: Calls `get_aging_diff()` to compare current time vs. mixing time against the Item's aging requirement. Sets `no_enough_aging` flag.
*   **Parameter Comparison**: Calls `compare_parameters()` to check observed values against min/max. Sets flags like `no_enough_hardness`.

#### `on_submit`
*   **Approval Enforcement**: Throws errors if flags (`no_enough_aging`, etc.) are set but corresponding Approver IDs are missing.
*   **Readings Check**: If Quality Approver is involved, ensures they have entered their own readings (`check_quality_approver_reading`).
*   **Finalization**: Calls `submit_dc_receipt_stock`.
    *   Submits the referenced **Stock Entry** (if it was Draft).
    *   Updates Stock Entry's `posting_date` and `time` to match the mixing time.
    *   Creates a generic system **Quality Inspection** document with "Accepted" status and links it.

### Helper Functions

*   `get_aging_diff()`: specific logic to calculate aging based on DC Receipt Mixing Date/Time or Stock Entry Posting Date.
*   `rollback_entries()`: Safe rollback mechanism.
*   `create_quality_inspection_entry()`: Generates the standard standard ERPNext Quality Inspection document.

---

## Related DocTypes

| DocType | Relationship |
|---------|--------------|
| **Item** | Source of `item_aging` and `Quality Inspection Template` |
| **Stock Entry** | The document being inspected (updated on submit) |
| **Delivery Challan Receipt** | Provides the original Mixing Date & Time |
| **Quality Inspection** | Created automatically upon successful Compound Inspection |
| **Employee** | Validates inspectors and approvers |
| **SPP Settings** | Maps designators to specific scan operations |

---

## Error Handling & Rollback

If any error occurs during the final submission (e.g., creating the Quality Inspection fails):

1.  **`rollback_entries`** is called.
2.  Deletes the partially created **Quality Inspection**.
3.  Safely deletes/reverts the **Stock Entry** using `delete_stock_entry_safely`.
4.  Resets the `Compound Inspection` to **Draft** state (`docstatus=0`).
5.  Logs the traceback in Error Log.
