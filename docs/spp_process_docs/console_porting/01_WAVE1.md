# Wave 1 — Materialize the 5 Stub Doctypes

**Why first:** the Dash already submits MB Mixing, FB Mixing, Compound Deviation, Sheeting, and Cut Bit Transfer in production. Their Console-side audit trail is broken because the local doctypes don't exist as real DocTypes (only `__pycache__` exists). This wave makes the existing flows complete.

**Outcome:** every successful Dash submission for these 5 processes lands a `Console *` record locally before fanning out to Legacy + Ops.

**Doctypes in this wave:** 5 parents + 4 children = 9 total

| Doctype | Type | Naming | Source legacy doctype |
|---|---|---|---|
| `Console Master Batch Mixing` | parent | `CMBM-.YYYY.-.#####` | `Delivery Challan Receipt` (where `inward_material_type=Master Batch`) |
| `Console Master Batch Item` | child | n/a | `DC Item` |
| `Console Final Batch Mixing` | parent | `CFBM-.YYYY.-.#####` | `Delivery Challan Receipt` (where `inward_material_type=Compound`) |
| `Console Final Batch Item` | child | n/a | `DC Item` |
| `Console Compound Deviation` | parent | `CDEV-.YYYY.-.#####` | overlay on `Compound Inspection` (status=Rejected, with Quality Manager scan) |
| `Console Sheeting Process` | parent | `CSHT-.YYYY.-.#####` | new in Console arch — no legacy parent |
| `Console Sheeting Item` | child | n/a | new |
| `Console Cut Bit Transfer` | parent | `CCBT-.YYYY.-.#####` | `Cut Bit Transfer` |
| `Console Cut Bit Item` | child | n/a | `Cut Bit Item` |

**Note:** `Console Sheeting Clip` (already exists with JSON) stays as the per-clip child of `Console Sheeting Process`.

---

## Step-by-step instructions

### Step 1.1 — Pre-flight cleanup (5 minutes)

Run the cleanup from the master plan:

```bash
cd /Users/alphaworkz/frappe-bench/apps/shree_polymer_console
for dt in console_master_batch_mixing console_final_batch_mixing console_compound_deviation console_sheeting_process console_cut_bit_transfer; do
    rm -rf shree_polymer_console/doctype/$dt
done
# Verify
ls shree_polymer_console/doctype/ | grep -E "(master_batch|final_batch|compound_deviation|sheeting_process|cut_bit_transfer)" || echo "Clean"
```

Commit: `chore: remove orphan stub directories before Wave 1 doctype creation`

### Step 1.2 — Create Console Master Batch Mixing (15 minutes)

**Path:** `shree_polymer_console/shree_polymer_console/doctype/console_master_batch_mixing/`

**Field schema** (parent — extends standard Console fields):

| Field | Type | Options | Reqd | Notes |
|---|---|---|---|---|
| `process_code` | Data | (read-only) | yes | Auto: `MASTER_BATCH_MIXING` |
| `mix_barcode` | Data | (unique) | yes | The C_YYMMDDXn barcode (Console-generated) |
| `spp_batch_no` | Data | | yes | Mirrors `mix_barcode` for legacy compat |
| `produced_item` | Link | Console Item | yes | The Master Batch finished item |
| `produced_qty` | Float | | yes | FG quantity in Kg |
| `bom_no` | Link | Console BOM | no | Reference BOM if used |
| `target_warehouse` | Link | Console Warehouse | yes | Where the MB lands |
| `mixing_workstation` | Link | Console Workstation | yes | Which mixer was used |
| `mixing_operation` | Link | Console Operation | no | Defaults to "Master Batch Mixing" |
| `operator_id` | Link | Console Employee | yes | Scanned operator |
| `operator_name` | Data | (read-only, fetch from Employee) | | |
| `mixing_start_time` | Datetime | | yes | |
| `mixing_end_time` | Datetime | | yes | |
| `consumption_items` | Table | Console Master Batch Item | yes | The raw materials consumed |
| `barcode_image` | Attach | | | Generated barcode PNG |
| `from_warehouse` | Link | Console Warehouse | yes | Source warehouse for raw materials |
| `is_internal_mixing` | Check | | | Mirrors legacy DCR flag |
| **All standard Console fields** | | | | (see master plan) |

**Child schema — `Console Master Batch Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `item_code` | Link | Console Item | yes |
| `batch_no` | Data | | yes |
| `spp_batch_no` | Data | | no |
| `mix_barcode` | Data | | no |
| `qty` | Float | | yes |
| `s_warehouse` | Link | Console Warehouse | yes |
| `is_finished_item` | Check | (default 0) | |
| `allow_zero_valuation_rate` | Check | (default 1) | |

**Files to create:**

```
shree_polymer_console/shree_polymer_console/doctype/console_master_batch_mixing/
├── __init__.py                              # empty
├── console_master_batch_mixing.json         # the doctype definition
├── console_master_batch_mixing.py           # controller class
└── test_console_master_batch_mixing.py      # at least 2 tests

shree_polymer_console/shree_polymer_console/doctype/console_master_batch_item/
├── __init__.py                              # empty
├── console_master_batch_item.json           # is_child_table: 1
└── console_master_batch_item.py             # minimal controller
```

**Controller (`console_master_batch_mixing.py`) outline:**

```python
import frappe
from frappe.model.document import Document

class ConsoleMasterBatchMixing(Document):
    def before_insert(self):
        self.process_code = "MASTER_BATCH_MIXING"
        if not self.sync_status:
            self.sync_status = "Pending"
        if not self.target_system:
            self.target_system = "Both"

    def validate(self):
        if not self.consumption_items:
            frappe.throw("At least one consumption item required")
        if self.produced_qty <= 0:
            frappe.throw("Produced quantity must be > 0")
        # Mass balance sanity check (allow 0.5% drift)
        consumed = sum(flt(i.qty) for i in self.consumption_items)
        if abs(consumed - self.produced_qty) / max(self.produced_qty, 1e-6) > 0.005:
            frappe.throw(f"Mass imbalance: consumed {consumed} vs produced {self.produced_qty}")

    def on_submit(self):
        self._emit_transaction_log()

    def _emit_transaction_log(self):
        from shree_polymer_console.shree_polymer_console.api import log_transaction
        log_transaction(
            process_type=self.process_code,
            reference_id=self.reference_id or self.name,
            status="Pending",
            target_system=self.target_system,
        )

    @frappe.whitelist()
    def apply_bridge_result(self, target, status, remote_name=None, error=None):
        # Bridge calls this after each leg completes
        if target == "Legacy":
            self.db_set("legacy_reference", remote_name)
        elif target == "Ops":
            self.db_set("ops_reference", remote_name)
        # Recompute aggregate sync_status
        if self.legacy_reference and self.ops_reference:
            self.db_set("sync_status", "Success")
        elif self.legacy_reference or self.ops_reference:
            self.db_set("sync_status", "Partial")
        if error:
            self.db_set("error_log_summary", str(error)[:500])
```

**Tests (`test_console_master_batch_mixing.py`) — minimum 2:**

```python
import frappe
import unittest

class TestConsoleMasterBatchMixing(unittest.TestCase):
    def setUp(self):
        # Ensure prerequisites: a Console Item, Warehouse, Employee fixture
        pass

    def test_create_and_submit(self):
        doc = frappe.get_doc({
            "doctype": "Console Master Batch Mixing",
            "mix_barcode": "C_TESTMB1",
            "spp_batch_no": "C_TESTMB1",
            "produced_item": "_Test Master Batch Item",
            "produced_qty": 100.0,
            "target_warehouse": "_Test Warehouse",
            "mixing_workstation": "_Test Mixer",
            "operator_id": "_Test Employee",
            "from_warehouse": "_Test Source Warehouse",
            "mixing_start_time": "2026-01-01 09:00:00",
            "mixing_end_time": "2026-01-01 09:30:00",
            "consumption_items": [{
                "item_code": "_Test Raw Material",
                "batch_no": "RAW-001",
                "qty": 100.0,
                "s_warehouse": "_Test Source Warehouse",
            }],
        }).insert(ignore_permissions=True)
        doc.submit()
        self.assertEqual(doc.docstatus, 1)
        self.assertEqual(doc.process_code, "MASTER_BATCH_MIXING")
        self.assertEqual(doc.sync_status, "Pending")

    def test_validate_mass_balance(self):
        with self.assertRaises(frappe.ValidationError):
            frappe.get_doc({
                "doctype": "Console Master Batch Mixing",
                # ... fields with consumed != produced
                "produced_qty": 100.0,
                "consumption_items": [{"item_code": "X", "batch_no": "Y", "qty": 50.0, "s_warehouse": "_Test Source Warehouse"}],
            }).insert(ignore_permissions=True)
```

### Step 1.3 — Create Console Final Batch Mixing (15 minutes)

Same shape as Master Batch Mixing, with these differences:

| Difference | Reason |
|---|---|
| `process_code = "FINAL_BATCH_MIXING"` | Different process |
| Naming series: `CFBM-.YYYY.-.#####` | |
| Mandatory: `master_batch_input` (Link to `Console Master Batch Mixing` row by mix_barcode) | FB consumes a previously created MB |
| Add `compound_specification` | The compound formula being made |

Otherwise identical schema, identical controller pattern.

### Step 1.4 — Create Console Compound Deviation (15 minutes)

**Path:** `shree_polymer_console/shree_polymer_console/doctype/console_compound_deviation/`

**Note:** No child table — deviation overlays an existing `Console Compound Inspection` record. We just need fields capturing the QA decision.

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "COMPOUND_DEVIATION") | yes |
| `source_inspection` | Link | Console Compound Inspection | yes |
| `item_code` | Link | Console Item | yes (fetched from source) |
| `batch_no` | Data | | yes (fetched) |
| `spp_batch_no` | Data | | yes (fetched) |
| `mix_barcode` | Data | | yes (fetched) |
| `available_qty` | Float | | yes |
| `deviation_status` | Select: `Accepted`/`Rejected`/`Rework` | | yes |
| `deviation_reason` | Small Text | | yes |
| `quality_manager` | Link | Console Employee | yes |
| `quality_manager_name` | Data | (read-only fetch) | |
| `destination_warehouse` | Link | Console Warehouse | yes (depends on status) |
| **All standard Console fields** | | | |

**Validate:**
- `source_inspection` must have `status = "Rejected"` (deviation only applies to rejected CIs)
- `available_qty > 0`
- `deviation_reason` minimum 10 chars

### Step 1.5 — Create Console Sheeting Process + Item (20 minutes)

**Parent — `Console Sheeting Process`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "SHEETING") | yes |
| `compound_input` | Link | Console Compound Inspection | yes |
| `item_code` | Link | Console Item | yes |
| `batch_no` | Data | | yes |
| `mix_barcode` | Data | | yes |
| `input_qty` | Float | | yes |
| `produced_clips` | Table | Console Sheeting Clip | yes (already exists) |
| `produced_items` | Table | Console Sheeting Item | yes |
| `cut_bit_qty` | Float | | no |
| `scrap_qty` | Float | | no |
| `from_warehouse` | Link | Console Warehouse | yes |
| `to_warehouse` | Link | Console Warehouse | yes |
| `operator_id` | Link | Console Employee | yes |
| **All standard Console fields** | | | |

**Child — `Console Sheeting Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `item_code` | Link | Console Item | yes |
| `qty` | Float | | yes |
| `t_warehouse` | Link | Console Warehouse | yes |
| `clip_barcode` | Data | | no |
| `is_finished_item` | Check | (default 1) | |

### Step 1.6 — Create Console Cut Bit Transfer + Item (20 minutes)

**Parent — `Console Cut Bit Transfer`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "CUT_BIT_TRANSFER") | yes |
| `transfer_from` | Select: `Warming`/`Blanking` | | yes |
| `source_warehouse` | Link | Console Warehouse | yes (auto from transfer_from) |
| `target_warehouse` | Link | Console Warehouse | yes |
| `scan_clip_or_bin` | Data | (input only, not stored) | no |
| `employee` | Link | Console Employee | yes |
| `items` | Table | Console Cut Bit Item | yes |
| `stock_entry_reference` | Data | | no (filled by Bridge) |
| **All standard Console fields** | | | |

**Child — `Console Cut Bit Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `item_code` | Link | Console Item | yes |
| `batch_no` | Data | | yes |
| `spp_batch_no` | Data | | no |
| `clip_name` | Data | | no |
| `qty` | Float | | yes |
| `ct_source_warehouse` | Link | Console Warehouse | no |

### Step 1.7 — Update seeders (10 minutes)

Edit `shree_polymer_console/shree_polymer_console/console_template_seeder.py`:

For each of the 5 new parents, ensure a `Console Process Template` row exists with:
- `process_code` matching
- `target_system = "Both (Parallel)"`
- Warehouse purposes (Source, Target, Scrap as applicable)
- Required fields registered for the Dash to render

Edit `shree_polymer_console/shree_polymer_console/console_mapping_seeder.py`:

Register the Legacy and Ops sync mappings for each. Example for MB Mixing:

```python
def seed_master_batch_mixing_mapping():
    upsert_mapping(
        process_code="MASTER_BATCH_MIXING",
        target_system="Legacy",
        legacy_doctype="Delivery Challan Receipt",
        legacy_field_map={
            "scan_barcode": "{{ mix_barcode }}",
            "inward_material_type": "Master Batch",
            "dc_items": "{{ consumption_items }}",
            # ... full map
        }
    )
    upsert_mapping(
        process_code="MASTER_BATCH_MIXING",
        target_system="Ops",
        ops_doctype="Stock Entry",
        ops_field_map={ ... }
    )
```

### Step 1.8 — Run migration + smoke test (15 minutes)

```bash
cd /Users/alphaworkz/frappe-bench
bench --site sppconsole-dev.local migrate
bench --site sppconsole-dev.local clear-cache
# Confirm doctypes exist
bench --site sppconsole-dev.local console <<EOF
import frappe
for dt in ["Console Master Batch Mixing", "Console Final Batch Mixing", "Console Compound Deviation", "Console Sheeting Process", "Console Cut Bit Transfer"]:
    print(dt, "exists:", frappe.db.exists("DocType", dt))
EOF
```

### Step 1.9 — Run unit tests (5 minutes)

```bash
bench --site sppconsole-dev.local run-tests --module shree_polymer_console.shree_polymer_console.doctype.console_master_batch_mixing.test_console_master_batch_mixing
# repeat for the other 4 parents
```

### Step 1.10 — End-to-end staging test (manual, ~30 minutes)

**Pre-conditions:**
- The 4 staging URLs you mentioned earlier are open and logged in.
- Master data sync has run (so Console has fresh items, batches, employees).

**Test for each of the 5 processes (one submission each):**

1. Navigate to `console.m.frappe.cloud/dash`, open the process screen
2. Submit a test record
3. Verify in `console.m.frappe.cloud/app/console-master-batch-mixing` (or the appropriate listview) — record exists with `sync_status = Pending` then `Success`
4. Verify in `console.m.frappe.cloud/app/console-transaction-log` — TWO entries (Legacy + Ops), both `Status = Success`
5. Verify in `sppmaster.frappe.cloud` — corresponding legacy doc exists, batch landed
6. Verify in `newspp.m.frappe.cloud` — corresponding Ops doc exists

**Known risks:**
- Bridge mapping for COMPOUND_DEVIATION may still hit the four-pass SE lookup band-aids — that's expected per today's earlier discussion. The new Console payload shape from this morning's refactor should make Pass 1 succeed cleanly.

### Step 1.11 — Commit & PR

Single feature branch off `develop`:

```bash
git checkout -b feat/wave1-console-process-doctypes
# ... commits ...
git push -u origin feat/wave1-console-process-doctypes
gh pr create --base develop --title "feat: Wave 1 — Materialize 5 Console process doctypes (MB, FB, Deviation, Sheeting, CutBit)"
```

PR description should include:
- Wave 1 link from this plan
- List of 9 doctypes added
- Migration command shown
- Per-doctype test pass screenshots
- Staging smoke test results (5 records, all `Success`)

---

## Wave 1 Definition of Done — checklist

- [ ] Pre-flight cleanup committed
- [ ] 5 parent + 4 child doctypes created at correct path
- [ ] All 9 controllers implement standard hooks
- [ ] All 5 parents have ≥2 unit tests passing
- [ ] `console_template_seeder.py` updated for all 5
- [ ] `console_mapping_seeder.py` updated with Legacy + Ops mappings for all 5
- [ ] `bench migrate` clean on `sppconsole-dev.local`
- [ ] Each of the 5 Dash screens submits successfully and creates a Console record
- [ ] Each submission produces 2 Console Transaction Logs (Legacy + Ops), both `Success`
- [ ] PR opened, reviewed, merged to `develop`
- [ ] CI build of dash assets succeeds (Frappe Cloud auto-deploy)

## Estimated total time

- Coding: ~2 hours (15-20 min × 5 parents + child overhead)
- Testing: ~1 hour
- Staging smoke + PR: ~1 hour
- **Total: ~4 hours of focused work**

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| `console_template_seeder.py` already has stale rows for these 5 process_codes | High | Audit seeder runs idempotent; prefer `upsert` over `insert` |
| Bridge mapping doesn't recognize new doctype names | Medium | Mapping seeder runs first; verify rows in `Legacy Sync Mapping` listview before testing |
| Child table autoname collides with legacy DC Item naming | Low | Use Console-prefixed naming `CMBI-.#####` for child tables that need names |
| Dash payload field names drift from doctype field names | Medium | Cross-check against today's MB Mixing payload from `MasterBatchMixingEntry.tsx` line 478-510 |
| `bench migrate` slow due to 5 new doctypes | Low | Run during low-traffic window |

## Files to be created (final list)

```
shree_polymer_console/shree_polymer_console/doctype/
├── console_master_batch_mixing/
│   ├── __init__.py
│   ├── console_master_batch_mixing.json
│   ├── console_master_batch_mixing.py
│   └── test_console_master_batch_mixing.py
├── console_master_batch_item/
│   ├── __init__.py
│   ├── console_master_batch_item.json
│   └── console_master_batch_item.py
├── console_final_batch_mixing/                  (same 4-file pattern)
├── console_final_batch_item/                    (same 3-file pattern)
├── console_compound_deviation/                  (same 4-file pattern)
├── console_sheeting_process/                    (same 4-file pattern)
├── console_sheeting_item/                       (same 3-file pattern)
├── console_cut_bit_transfer/                    (same 4-file pattern)
└── console_cut_bit_item/                        (same 3-file pattern)
```

Total: 9 new directories, 32 new files.
