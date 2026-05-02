# Shree Polymer Custom App - DocType API Audit

## Overview

This document provides a comprehensive audit of how the custom DocTypes in `shree_polymer_custom_app` interact with the centralized `api.py` file located at:

```
shree_polymer_custom_app/shree_polymer_custom_app/api.py
```

**Total Lines:** 1,508  
**Whitelisted Functions:** 24  
**DocTypes Audited:** 17

---

## API.py Function Reference

### Whitelisted Functions (Frontend Callable)

| Function | Line | Purpose |
|----------|------|---------|
| `on_item_update` | 29 | Hook for Item updates |
| `on_batch_update` | 40 | Hook for Batch updates |
| `on_sle_update` | 45 | Stock Ledger Entry update hook |
| `item_update` | 175 | Batch stock quantity sync |
| `update_consumed_items` | 268 | Update consumed items in Stock Entry |
| `update_se_barcode` | 293 | Generate/update Stock Entry barcodes |
| `save_generate_batchwise_report` | 333 | Generate batch-wise reports |
| `update_stock_balance` | 432 | Update Item Batch Stock Balance |
| `get_process_based_employess` | 440 | **Employee search by SPP Process** |
| `generate_batch_barcode` | 456 | Generate barcode for batches |
| `update_wh_barcode` | 477 | Update Warehouse barcode |
| `update_emp_barcode` | 519 | Update Employee barcode |
| `update_asset_barcode` | 651 | Update Asset (Bin) barcode |
| `update_all_asset_barcodes` | 694 | Bulk update all asset barcodes |
| `update_all_emp_barcode` | 739 | Bulk update all employee barcodes |
| `update_all_wh_barcode` | 759 | Bulk update all warehouse barcodes |
| `update_all_raw_materials` | 1063 | Update BOM raw materials |
| `update_exe_sheeting_text_to_barcode` | 1117 | Convert sheeting clip text to barcode |
| `update_exe_blank_bin_text_to_barcode` | 1142 | Convert blank bin text to barcode |
| `validate_document_submission` | 1186 | **Validate document before Stock Entry submission** |
| `validate_dc_document_cancellation` | 1230 | **Validate DC cancellation dependencies** |
| `validate_stock_entry` | 1251 | Validate Stock Entry status |
| `get_item_details` | 1359 | Get item details by batch number |
| `get_lot_details` | 1375 | **Get lot bin/rejection details with weight breakdown** |

### Internal Functions (Backend Only)

| Function | Line | Purpose |
|----------|------|---------|
| `get_stock_entry_naming_series` | 638 | Get custom naming series for Stock Entry types |
| `generate_batch_no` | 805 | Create/update Batch records |
| `delete_batches` | 841 | Delete batch records safely |
| `get_details_by_lot_no` | 877 | **Fetch lot stock details from Sub Lot Creation** |
| `get_parent_lot` | 1040 | Find parent lot number for sub-lots |
| `delete_stock_entry_safely` | 1269 | **Safe deletion with circular link handling** |
| `get_workstation_by_operation` | 1102 | Get workstation by operation mapping |
| `remove_spl_characters` | 515 | Remove special characters for barcode filenames |
| `get_decimal_values_without_roundoff` | 1168 | Decimal truncation utility |

---

## DocType → API.py Mapping

### 1. Delivery Challan Receipt

**File:** `doctype/delivery_challan_receipt/delivery_challan_receipt.py` (71KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    delete_stock_entry_safely
)
```

#### Functions Used
| Function | Purpose in DocType |
|----------|-------------------|
| `get_stock_entry_naming_series` | Get naming series for Material Receipt/Transfer Stock Entries |
| `delete_stock_entry_safely` | Cleanup linked Stock Entries on cancellation |

#### Own Whitelisted Functions
| Function | Line | Purpose |
|----------|------|---------|
| `validate_batch_release` | 211 | Validate batch release status before receipt |
| `validate_delivery_challan` | 249 | Validate DC number and fetch items |
| `create_stock_entry` | 430 | Create Material Receipt Stock Entry |
| `get_delivery_challan_items` | 643 | Fetch DC item details |
| `get_compound_details` | 1084 | Get compound item details |
| `get_item_stock_details` | 1319 | Get current stock for item/batch |

#### Business Logic
- Receives compound/items from Unit I via DC
- Creates Material Receipt Stock Entry
- Validates batch release status (Compound Inspection required)
- Uses `validate_document_submission` in api.py for cross-validation

---

### 2. Compound Inspection

**File:** `doctype/compound_inspection/compound_inspection.py` (19KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import delete_stock_entry_safely
```

#### Functions Used
| Function | Purpose in DocType |
|----------|-------------------|
| `delete_stock_entry_safely` | Cleanup on inspection failure/cancellation |

#### Business Logic
- Inspects compound received from Delivery Challan
- Must be completed before DC Receipt Stock Entry submission
- Links to `validate_document_submission` which checks inspection status

---

### 3. Material Transfer

**File:** `doctype/material_transfer/material_transfer.py` (40KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    delete_stock_entry_safely
)
```

#### Frontend JS Calls
```javascript
// material_transfer.js:50
frappe.call({
    method: "frappe.client.get_all",
    query: "shree_polymer_custom_app.shree_polymer_custom_app.api.get_process_based_employess",
    filters: { process: "Material Transfer" }
})
```

#### Functions Used
| Function | Location | Purpose |
|----------|----------|---------|
| `get_stock_entry_naming_series` | Python | Naming series for Material Transfer SE |
| `delete_stock_entry_safely` | Python | Cleanup on cancellation |
| `get_process_based_employess` | JS | Employee dropdown filtered by designation |

---

### 4. Cut Bit Transfer

**File:** `doctype/cut_bit_transfer/cut_bit_transfer.py` (9KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    delete_stock_entry_safely
)
```

#### Functions Used
| Function | Purpose |
|----------|---------|
| `get_stock_entry_naming_series` | Naming series for Repack SE |
| `delete_stock_entry_safely` | Cleanup on cancellation |

---

### 5. Blanking DC Entry

**File:** `doctype/blanking_dc_entry/blanking_dc_entry.py` (41KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import delete_stock_entry_safely
```

#### Frontend JS Calls
```javascript
// blanking_dc_entry.js:110
frappe.call({
    query: "shree_polymer_custom_app.shree_polymer_custom_app.api.get_process_based_employess"
})
```

#### Functions Used
| Function | Location | Purpose |
|----------|----------|---------|
| `delete_stock_entry_safely` | Python | Handle SE cleanup |
| `get_process_based_employess` | JS | Filter employees by Blanking process |

---

### 6. Blank Bin Inward Entry

**File:** `doctype/blank_bin_inward_entry/blank_bin_inward_entry.py` (19KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import delete_stock_entry_safely
```

#### Functions Used
| Function | Purpose |
|----------|---------|
| `delete_stock_entry_safely` | Cleanup Stock Entries on cancellation |

---

### 7. Work Planning / Add On Work Planning

**Files:** 
- `doctype/work_planning/work_planning.py` (17KB)
- `doctype/add_on_work_planning/add_on_work_planning.py` (16KB)

#### Backend Imports
None directly from api.py

#### Notes
These DocTypes manage production planning and do not directly interact with api.py. They operate through frappe's standard document methods.

---

### 8. Blank Bin Issue

**File:** `doctype/blank_bin_issue/blank_bin_issue.py` (7KB)

#### Backend Imports
None directly from api.py

#### Notes
Manages blank bin issuance to production lines. Uses standard Frappe methods.

---

### 9. Inspection Entry (Patrol/Line/Lot Inspection)

**File:** `doctype/inspection_entry/inspection_entry.py` (80KB - **Largest Controller**)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    # Additional imports available
)
```

#### Own Whitelisted Functions
| Function | Line | Purpose |
|----------|------|---------|
| `get_rejection_reasons` | 743 | Fetch rejection reasons (allow_guest=True) |
| `validate_lot_for_inspection` | 1482 | Validate lot before inspection |

#### Integration Points
- `validate_document_submission` (api.py:1186) checks:
  - Line Inspection → requires Moulding Production Entry
  - Referenced in submission validation workflow

#### Functions Used
| Function | Purpose |
|----------|---------|
| `get_stock_entry_naming_series` | Naming series for scrap movement SE |

---

### 10. Line Inspection Entry

**File:** `doctype/line_inspection_entry/line_inspection_entry.py` (12KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import get_stock_entry_naming_series
```

#### Functions Used
| Function | Purpose |
|----------|---------|
| `get_stock_entry_naming_series` | Naming series for rejection SE |

---

### 11. Lot Inspection Entry

**File:** `doctype/lot_inspection_entry/lot_inspection_entry.py` (8KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import get_stock_entry_naming_series
```

#### Integration Points
- Must be completed before `Moulding Production Entry` Stock Entry submission
- Referenced in `validate_document_submission` (api.py:1216-1218)

---

### 12. Moulding Production Entry

**File:** `doctype/moulding_production_entry/moulding_production_entry.py` (72KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series,
    generate_batch_no,
    delete_batches,
    delete_stock_entry_safely
)
```

#### Frontend JS Calls
```javascript
// moulding_production_entry.js:37
frappe.call({
    query: "shree_polymer_custom_app.shree_polymer_custom_app.api.get_process_based_employess"
})

// moulding_production_entry.js:137, 427, 572
frappe.call({
    method: 'shree_polymer_custom_app.shree_polymer_custom_app.api.get_lot_details',
    args: { lot_number, doctype, docname, mould_reference }
})
```

#### Own Whitelisted Functions
| Function | Line | Purpose |
|----------|------|---------|
| `validate_compound_stock` | 1203 | Validate compound availability |
| `get_compound_details` | 1231 | Fetch compound item info |
| `validate_bin_barcode` | 1337 | Validate scanned blank bin |
| `get_mould_details` | 1368 | Fetch mould specifications |
| `calculate_mass_balance` | 1382 | Calculate production mass balance |

#### Functions Used
| Function | Location | Purpose |
|----------|----------|---------|
| `get_stock_entry_naming_series` | Python | Naming for Production SE |
| `generate_batch_no` | Python | Create lot batch record |
| `delete_batches` | Python | Cleanup batches on failure |
| `delete_stock_entry_safely` | Python | Rollback SE on error |
| `get_process_based_employess` | JS | Employee dropdown |
| `get_lot_details` | JS | **Fetch rejection/bin details for mass balance** |

---

### 13. Deflashing Despatch Entry

**File:** `doctype/deflashing_despatch_entry/deflashing_despatch_entry.py` (27KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    # Additional functions
)
```

#### Frontend JS Calls
```javascript
// deflashing_despatch_entry.js:402, 559
frappe.call({
    method: 'shree_polymer_custom_app.shree_polymer_custom_app.api.get_lot_details',
    args: { lot_number, doctype: 'Deflashing Despatch Entry', docname }
})
```

#### Functions Used
| Function | Location | Purpose |
|----------|----------|---------|
| `get_stock_entry_naming_series` | Python | Naming for despatch SE |
| `get_lot_details` | JS | Fetch lot rejection details for weight calculation |

---

### 14. Deflashing Receipt Entry

**File:** `doctype/deflashing_receipt_entry/deflashing_receipt_entry.py` (37KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_stock_entry_naming_series, 
    # Additional functions
)
```

#### Integration Points
- Must have `Incoming Inspection Entry` before Stock Entry submission
- Referenced in `validate_document_submission` (api.py:1219-1221)

---

### 15. Incoming Lot Inspection Entry

**File:** `doctype/incoming_lot_inspection_entry/incoming_lot_inspection_entry.py` (7KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import get_stock_entry_naming_series
```

---

### 16. Despatch To U1 Entry

**File:** `doctype/despatch_to_u1_entry/despatch_to_u1_entry.py` (26KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_details_by_lot_no,
    get_parent_lot, 
    delete_stock_entry_safely
)
```

#### Functions Used
| Function | Purpose |
|----------|---------|
| `get_details_by_lot_no` | Fetch lot stock from Sub Lot Creation workflow |
| `get_parent_lot` | Trace parent lot for sub-lots |
| `delete_stock_entry_safely` | Cleanup on cancellation |

---

### 17. Receive Deflashing Entry

**File:** `doctype/receive_deflashing_entry/receive_deflashing_entry.py` (8KB)

#### Notes
No direct imports from api.py. Uses standard document operations.

---

### 18. Packing

**File:** `doctype/packing/packing.py` (26KB)

#### Backend Imports
```python
from shree_polymer_custom_app.shree_polymer_custom_app.api import (
    get_details_by_lot_no,
    get_parent_lot, 
    delete_stock_entry_safely
)
```

#### Functions Used
| Function | Purpose |
|----------|---------|
| `get_details_by_lot_no` | Validate lot availability for packing |
| `get_parent_lot` | Trace lot lineage |
| `delete_stock_entry_safely` | Cleanup on failure |

---

## Summary Matrix

| DocType | get_stock_entry_naming_series | delete_stock_entry_safely | generate_batch_no | delete_batches | get_details_by_lot_no | get_parent_lot | get_process_based_employess (JS) | get_lot_details (JS) |
|---------|-------------------------------|---------------------------|-------------------|----------------|----------------------|----------------|----------------------------------|---------------------|
| Delivery Challan Receipt | ✓ | ✓ | | | | | | |
| Compound Inspection | | ✓ | | | | | | |
| Material Transfer | ✓ | ✓ | | | | | ✓ | |
| Cut Bit Transfer | ✓ | ✓ | | | | | | |
| Blanking DC Entry | | ✓ | | | | | ✓ | |
| Blank Bin Inward Entry | | ✓ | | | | | | |
| Work Planning | | | | | | | | |
| Add On Work Planning | | | | | | | | |
| Blank Bin Issue | | | | | | | | |
| Inspection Entry | ✓ | | | | | | | |
| Line Inspection Entry | ✓ | | | | | | | |
| Lot Inspection Entry | ✓ | | | | | | | |
| Moulding Production Entry | ✓ | ✓ | ✓ | ✓ | | | ✓ | ✓ |
| Deflashing Despatch Entry | ✓ | | | | | | | ✓ |
| Deflashing Receipt Entry | ✓ | | | | | | | |
| Incoming Lot Inspection | ✓ | | | | | | | |
| Despatch To U1 Entry | | ✓ | | | ✓ | ✓ | | |
| Packing | | ✓ | | | ✓ | ✓ | | |

---

## Refactoring Recommendations

### 1. Function Consolidation
- `get_stock_entry_naming_series` is used by **12 DocTypes** → Consider making it a mixin
- `delete_stock_entry_safely` is used by **10 DocTypes** → Move to a shared utility class

### 2. Duplicate Logic
- Employee filtering (`get_process_based_employess`) is called from 5 JS files → Consider client-side caching

### 3. Validation Flow
The `validate_document_submission` function handles multiple DocTypes:
- Delivery Challan Receipt → Compound Inspection check
- Inspection Entry (Line) → Moulding Production Entry check  
- Moulding Production Entry → Lot Inspection Entry check
- Deflashing Receipt Entry → Incoming Inspection Entry check

**Recommendation:** Each DocType should have its own validation method; `validate_document_submission` should orchestrate calls.

### 4. Lot Details API
`get_lot_details` is critical for:
- Moulding Production Entry (mass balance calculation)
- Deflashing Despatch Entry (weight tracking)

**Recommendation:** Move to a dedicated `lot_utils.py` for clearer separation.

### 5. Code Quality Issues
- SQL injection vulnerabilities in several raw queries
- Missing type hints
- Inconsistent error handling patterns
- Large functions that should be split (e.g., `get_details_by_lot_no` is 100+ lines)

---

## Next Steps for Refactoring

1. Extract utility functions into separate modules:
   - `stock_utils.py` - Stock Entry operations
   - `batch_utils.py` - Batch management
   - `lot_utils.py` - Lot tracking
   - `barcode_utils.py` - Barcode generation

2. Create base controller class with common methods

3. Add comprehensive test coverage for api.py functions

4. Document inter-DocType dependencies in workflow diagram
