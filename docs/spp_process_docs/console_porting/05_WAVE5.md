# Wave 5 — Moulding Production Entry (The Target)

**Why fifth and final:** This is the doctype that consumes everything Waves 1-4 produce. MPE is the production booking event — it consumes Blank Bins (Wave 2), runs against a Job Card from Add On Work Planning (Wave 3), validates Line/Patrol Inspections (Wave 4), and emits the finished Stock Entry that triggers Deflashing/Packing downstream.

**Outcome:** Console operators can record moulding production end-to-end: scan supervisor + operator + lot, weigh produced output, scan consumed bins (with partial-bin balance handling), and submit. Both Legacy and Ops receive the production booking + Stock Entry (Manufacture).

**Doctypes in this wave:** 1 parent + 2 children = 3 total

| Doctype | Type | Naming | Source legacy doctype |
|---|---|---|---|
| `Console Moulding Production Entry` | parent | `CMPE-.YYYY.-.#####` | `Moulding Production Entry` |
| `Console Moulding Balance Bin` | child | n/a | `Moulding Balance Bin` |
| `Console Moulding Serial No` | child | n/a | `Moulding Serial No` |

---

## Why MPE is the highest-complexity wave

MPE has more business logic than any prior wave:

1. **Mass balance validation** — produced + scrap + leakage must equal total bin consumption (within tolerance). This is industrial-grade checking that catches operator errors.
2. **Partial bin handling** — if a bin is only partially used, operator weighs the remainder. Console computes net consumption.
3. **FIFO / specific-batch allocation** — when multiple bins of different batches are used, MPE must decide consumption order.
4. **Inspection prerequisite** — submission BLOCKS without Line + Patrol Inspections from Wave 4.
5. **Real-time stock validation** — must verify warehouse balance is sufficient (catches "negative stock" before submit).
6. **Batch # generation** — MPE generates the final Batch # (e.g., `T-LOT123-001`) and propagates it back to all related records (Inspection Entries get updated with this batch_no).
7. **Cascading SE submissions** — after MPE's Stock Entry submits, the rejection Stock Entries from Wave 4 inspections become eligible to submit and need to be triggered.
8. **Injection moulding variant** — separate logic for `purged_compound` and `compound_leakage` fields when `is_injection_moulding=1`.

This wave has the most thorough validation chain in the entire porting plan.

---

## Prerequisite checks

```bash
bench --site sppconsole-dev.local console <<EOF
import frappe
required_doctypes = [
    "Console Add On Work Planning",   # Wave 3
    "Console Blank Bin Issue",        # Wave 3
    "Console Inspection Entry",       # Wave 4
]
for dt in required_doctypes:
    assert frappe.db.exists("DocType", dt), f"Prerequisite missing: {dt}"

# Test data prerequisites
assert frappe.db.count("Console Add On Work Planning", {"docstatus": 1}) > 0, "No submitted planning records"
assert frappe.db.count("Console Blank Bin Issue", {"docstatus": 1}) > 0, "No bin issues recorded"
assert frappe.db.count("Console Inspection Entry", {"inspection_type": "Line Inspection", "docstatus": 1}) > 0, "No Line Inspections"
assert frappe.db.count("Console Inspection Entry", {"inspection_type": "Patrol Inspection", "docstatus": 1}) > 0, "No Patrol Inspections"
print("All prereqs OK — proceeding to Wave 5")
EOF
```

---

## Step-by-step instructions

### Step 5.1 — Console Moulding Production Entry parent (1 hour)

**Field schema:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "MOULDING_PRODUCTION_ENTRY") | yes |
| `moulding_date` | Date | | yes |
| `scan_lot_number` | Data | (input only) | no |
| `lot_no` | Data | (Link-style ref to Add On Work Plan Item.lot_number) | yes |
| `job_card` | Data | (auto-fetch from lot) | yes |
| `production_item` | Link | Console Item | (auto-fetch) | yes |
| `compound` | Link | Console Item | (auto-fetch from BOM) | yes |
| `bom` | Link | Console BOM | (auto-fetch) | yes |
| `s_source_warehouse` | Link | Console Warehouse | yes |
| `target_warehouse` | Link | Console Warehouse | yes (auto from template) |
| `scrap_warehouse` | Link | Console Warehouse | yes (auto from template) |
| `supervisor_id` | Link | Console Employee | yes |
| `operator_id` | Link | Console Employee | yes |
| `weight` | Float | | yes | Output FG weight in Kg |
| `purged_compound` | Float | (visible if is_injection_moulding) | | |
| `compound_leakage` | Float | (visible if is_injection_moulding) | | |
| `is_injection_moulding` | Check | (auto-detect from workstation name) | | |
| `total_consumed_weight` | Float | (read-only, sum from items + balances) | | |
| `total_required_weight` | Float | (read-only, calc: weight + rejection + purge + leakage) | | |
| `mass_balance_drift` | Float | (read-only, % difference) | | |
| `balance_bins` | Table | Console Moulding Balance Bin | (optional — only partial bins) | |
| `serial_nos` | Table | Console Moulding Serial No | (optional, for serialized items) | |
| `batch_no` | Data | (read-only, generated on submit, e.g., T-LOT123-001) | | |
| `stock_entry_reference` | Data | (filled by Bridge) | | |
| `inspection_entries_updated` | Long Text (JSON) | (list of Inspection Entry names tied back) | | |
| `batch_details` | Long Text (JSON) | Hidden — full bin/lot allocation map for replay | | |
| **All standard Console fields** | | | |

**Child — `Console Moulding Balance Bin`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `bin_code` | Link | Console Asset | yes |
| `bin_barcode` | Data | (fetched) | |
| `original_bin_weight` | Float | | yes | What was issued in Wave 3 |
| `weight_of_balance_bin` | Float | | yes | Weighed remainder after partial use |
| `bin_tare_weight` | Float | (auto-fetch from Asset) | yes |
| `net_consumption` | Float | (read-only, calc: original − remainder) | yes |
| `compound_in_bin` | Link | Console Item | (auto-fetch) | |

**Child — `Console Moulding Serial No`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `compound_code` | Data | | yes |
| `serial_no` | Int | | yes |
| `job_card_reference` | Data | | yes |
| `shift_series` | Data | | no |

### Step 5.2 — Critical validation hooks (1.5 hours)

This is the heaviest validate hook in the entire porting plan.

```python
def validate(self):
    self._validate_lot_and_inspections()
    self._validate_mass_balance()
    self._validate_warehouse_stock()
    self._auto_detect_injection_moulding()

def _validate_lot_and_inspections(self):
    # 1. Lot must reference submitted Work Planning
    planning = frappe.db.get_value(
        "Console Add On Work Plan Item",
        {"lot_number": self.lot_no, "docstatus": 1},
        ["item_produced", "job_card", "bom", "work_station"],
        as_dict=True
    )
    if not planning:
        frappe.throw(f"Lot {self.lot_no} has no submitted Work Planning record")
    self.production_item = planning.item_produced
    self.job_card = planning.job_card
    self.bom = planning.bom

    # 2. Block if Line + Patrol inspections missing
    missing = []
    for itype in ["Line Inspection", "Patrol Inspection"]:
        if not frappe.db.exists("Console Inspection Entry", {
            "lot_no": self.lot_no, "inspection_type": itype, "docstatus": 1
        }):
            missing.append(itype)
    if missing:
        frappe.throw(
            f"The following inspections are missing for Lot {self.lot_no}: {', '.join(missing)}. "
            f"Complete inspections before booking production."
        )

    # 3. Compound (input) auto-fetch from BOM
    bom_input = frappe.db.get_value("Console BOM Item", {"parent": self.bom, "is_input": 1}, "item_code")
    self.compound = bom_input

def _validate_mass_balance(self):
    # Get bins issued for this lot
    issued = frappe.get_all("Console Blank Bin Issue Item",
        filters={"lot_number": self.lot_no, "is_consumed": 0},
        fields=["bin", "bin_weight", "compound"]
    )
    total_issued = sum(flt(b.bin_weight) for b in issued)

    # Calculate consumed (full bins) + balance (partial bins)
    balance_bin_codes = {b.bin_code for b in self.balance_bins}
    fully_consumed = sum(flt(b.bin_weight) for b in issued if b.bin not in balance_bin_codes)
    partial_consumed = sum(flt(b.net_consumption) for b in self.balance_bins)
    self.total_consumed_weight = fully_consumed + partial_consumed

    # Calculate required (output + scrap + injection waste)
    rejection = frappe.db.sql(
        "select sum(total_rejected_qty_kg) from `tabConsole Inspection Entry` "
        "where lot_no=%s and docstatus=1", self.lot_no
    )[0][0] or 0
    self.total_required_weight = (
        flt(self.weight) +
        flt(rejection) +
        (flt(self.purged_compound) if self.is_injection_moulding else 0) +
        (flt(self.compound_leakage) if self.is_injection_moulding else 0)
    )

    # Drift check (allow 1% tolerance — configurable)
    tolerance = 0.01
    if self.total_consumed_weight > 0:
        self.mass_balance_drift = abs(
            self.total_consumed_weight - self.total_required_weight
        ) / self.total_consumed_weight
        if self.mass_balance_drift > tolerance:
            frappe.throw(
                f"Mass imbalance: consumed {self.total_consumed_weight:.3f} kg "
                f"vs required {self.total_required_weight:.3f} kg "
                f"(drift {self.mass_balance_drift*100:.2f}%, tolerance {tolerance*100:.2f}%). "
                f"Verify partial bin weights or rejection quantities."
            )

def _validate_warehouse_stock(self):
    # Real-time stock check via Console Stock Cache
    available = frappe.db.get_value(
        "Console Stock Cache",
        {"item_code": self.compound, "warehouse": self.s_source_warehouse, "actual_qty": [">", 0]},
        "actual_qty"
    ) or 0
    if available < self.total_required_weight:
        frappe.throw(
            f"Insufficient stock: {available:.3f} kg available in {self.s_source_warehouse}, "
            f"need {self.total_required_weight:.3f} kg"
        )

def _auto_detect_injection_moulding(self):
    if not self.is_injection_moulding:
        # Check workstation name pattern
        ws = frappe.db.get_value("Console Add On Work Plan Item",
            {"lot_number": self.lot_no}, "work_station")
        if ws and "INJ" in ws.upper():
            self.is_injection_moulding = 1

def on_submit(self):
    self._generate_batch_no()
    self._update_inspection_entries()
    self._mark_bins_consumed()
    self._emit_transaction_log()

def _generate_batch_no(self):
    # Format: T-{lot_no}-{seq}
    seq = frappe.db.count("Console Moulding Production Entry",
        {"lot_no": self.lot_no, "docstatus": 1}) + 1
    self.batch_no = f"T-{self.lot_no}-{seq:03d}"
    self.db_set("batch_no", self.batch_no)

def _update_inspection_entries(self):
    # Wire batch_no back into all Line/Patrol/Lot inspections for this lot
    inspections = frappe.get_all("Console Inspection Entry",
        filters={"lot_no": self.lot_no, "docstatus": 1},
        pluck="name"
    )
    for ie in inspections:
        frappe.db.set_value("Console Inspection Entry", ie, "batch_no", self.batch_no)
    self.db_set("inspection_entries_updated", json.dumps(inspections))

def _mark_bins_consumed(self):
    # Full bins → fully consumed
    balance_bin_codes = {b.bin_code for b in self.balance_bins}
    issued = frappe.get_all("Console Blank Bin Issue Item",
        filters={"lot_number": self.lot_no, "is_consumed": 0},
        fields=["name", "bin"]
    )
    for b in issued:
        if b.bin not in balance_bin_codes:
            # Full consumption
            frappe.db.set_value("Console Blank Bin Issue Item", b.name, "is_consumed", 1)
        # Partial bins — leave is_consumed=0; bin still has remainder
```

### Step 5.3 — Bridge sequence (45 minutes)

MPE submission triggers a multi-step sequence:

```python
sequence = [
    {"step_id": "create_stock_entry_manufacture", "action": "create_doc", "params": {
        "doctype": "Stock Entry",
        "stock_entry_type": "Manufacture",
        "from_bom": 1,
        "bom_no": "{{ bom }}",
        "fg_completed_qty": "{{ weight }}",
        "items": [
            # Raw materials (consumption)
            {
                "item_code": "{{ compound }}",
                "qty": "{{ total_consumed_weight }}",
                "s_warehouse": "{{ s_source_warehouse }}",
                "is_finished_item": 0,
                # Per-batch breakdown from batch_details
            },
            # Scrap (purge + leakage for Injection)
            {
                "item_code": "{{ compound }}_SCRAP",
                "qty": "{{ purged_compound + compound_leakage }}",
                "is_scrap_item": 1,
                "t_warehouse": "{{ scrap_warehouse }}",
            },
            # Finished goods
            {
                "item_code": "{{ production_item }}",
                "qty": "{{ weight }}",
                "t_warehouse": "{{ target_warehouse }}",
                "batch_no": "{{ batch_no }}",
                "is_finished_item": 1,
            }
        ],
        "__submit": True
    }},
    {"step_id": "update_job_card", "action": "update_doc", "params": {
        "doctype": "Job Card",
        "name": "{{ job_card }}",
        "completed_qty": "{{ weight }}",
        "status": "Completed"
    }},
    {"step_id": "submit_pending_inspection_se", "action": "call_method", "params": {
        "method": "submit_inspection_stock_entries_from_moulding",
        "args": {"lot_no": "{{ lot_no }}", "batch_no": "{{ batch_no }}"}
    }},
]
```

**Critical:** the third step (submit pending inspection SEs) is what releases the rejection Stock Entries from Wave 4 — they were waiting for `batch_no` to materialize.

### Step 5.4 — Update Dash (2 hours)

`MouldingProductionEntry.tsx` already exists per the registry — review and adapt to the new doctype shape.

**Key UX considerations:**

- Scan supervisor + operator at top
- Scan Lot # → fetch full Job Card context, show:
  - Production Item, Compound (input), BOM, Workstation, Target Qty
  - Issued Bins list (from Wave 3 Blank Bin Issue, filtered by lot_no, is_consumed=0)
  - **Inspection status indicator** — Line + Patrol checkmarks (red if missing)
- Enter Produced Weight (FG)
- IF injection moulding: enter Purged + Leakage
- For each issued bin: checkbox "Fully Consumed?" or scan barcode to enter Balance Bin row
- For Balance Bin rows: enter remainder weight, system computes net_consumption
- Real-time mass balance display: Required vs Consumed, drift %
- Submit only enabled when:
  - Mass balance within tolerance
  - All required inspections present
  - At least 1 bin recorded

This is the most complex dash screen in the entire porting effort. Allocate full 2 hours.

### Step 5.5 — Tests (1 hour)

```python
def test_blocks_without_line_inspection():
    # Try to submit MPE without Line Inspection → ValidationError
    ...

def test_blocks_without_patrol_inspection():
    # Same with Patrol → ValidationError
    ...

def test_mass_balance_within_tolerance_passes():
    # consumed=100, required=100.5 → drift=0.5%, within 1% tolerance → OK
    ...

def test_mass_balance_outside_tolerance_blocks():
    # consumed=100, required=110 → drift=10% → ValidationError
    ...

def test_partial_bin_calculation():
    # Bin issued 50kg, balance 12kg → net_consumption=38kg
    ...

def test_batch_no_generation():
    # Submit MPE for Lot LOT-20260101-001 → batch_no=T-LOT-20260101-001-001
    ...

def test_batch_no_propagates_to_inspections():
    # After submit, all Inspection Entries for the lot have batch_no set
    ...

def test_warehouse_stock_check():
    # If source warehouse has 50kg, try to submit needing 100kg → ValidationError
    ...

def test_full_bin_marked_consumed():
    # Fully consumed bins from Blank Bin Issue → is_consumed=1 after MPE submit
    ...

def test_injection_moulding_includes_purge_leakage():
    # is_injection_moulding=1 with purge=2, leakage=1 → total_required includes them
    ...

def test_non_injection_excludes_purge_leakage():
    # is_injection_moulding=0 → purge/leakage NOT included in total_required
    ...
```

Aim for ≥10 tests on this doctype given its complexity.

### Step 5.6 — End-to-end staging test (1.5 hours)

The most thorough end-to-end test in the porting plan.

**Pre-test setup:**
1. Wave 1-4 must all be complete and migrated
2. Have at least one chain of records ready:
   - Submitted MB Mixing → CI → Sheeting → Cut Bit Transfer (Wave 1)
   - Submitted Bulk Clip Release → Blanking DC Entry → Blank Bin Inward (Wave 2)
   - Submitted Add On Work Planning (Lot # generated) → Blank Bin Issue (bins assigned) (Wave 3)
   - Submitted Line Inspection + Patrol Inspection for the lot (Wave 4)

**Test sequence:**
1. **Open MPE Dash screen**, scan supervisor + operator
2. **Scan Lot #** — verify all auto-fetched fields populate, both inspections show as ✓
3. **Enter weights:** weight=80, purge=0 (assume not injection moulding for first test)
4. **Mark all issued bins as fully consumed** OR add a partial bin balance
5. **Verify mass balance display** is within tolerance
6. **Submit** — verify:
   - `Console Moulding Production Entry` record created with `batch_no` set
   - `Console Inspection Entry` records updated with same `batch_no`
   - `Console Blank Bin Issue Item` rows marked `is_consumed=1`
   - Both Legacy + Ops have:
     - Stock Entry (Manufacture) submitted with batch_no
     - Job Card status=Completed
     - Inspection rejection Stock Entries (if any rejections) submitted
7. **Verify Console Transaction Log** — TWO entries (Legacy + Ops), both `Status=Success`
8. **Negative test:** try to submit a 2nd MPE for same Lot # without all bins consumed → expect failure or partial-bin handling

---

## Wave 5 Definition of Done — checklist

- [ ] 3 doctypes created and migrated
- [ ] All validation hooks implemented and unit-tested
- [ ] Bridge sequence supports multi-step Stock Entry + Job Card update + cascade
- [ ] MouldingProductionEntry.tsx adapted (or rewritten) for new schema
- [ ] ≥10 unit tests passing
- [ ] Mass balance, inspection prereq, stock check, batch_no generation all verified
- [ ] End-to-end staging: full chain Wave 1 → 5 results in successful production booking on both ERPs
- [ ] Cascade verified: rejection SE from Wave 4 inspections submits after MPE
- [ ] PR opened, reviewed, merged to `develop`

## Estimated time

- Coding (doctypes): 1 hour
- Validation hooks: 1.5 hours
- Bridge sequence: 45 minutes
- Coding (Dash): 2 hours
- Tests: 1 hour
- Staging E2E: 1.5 hours
- **Total: ~7.5 hours**

This is the longest wave. Plan it for a dedicated day.

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| Mass balance tolerance too strict — operators get blocked on legitimate small variances | High | Start at 1%, make configurable in `Console Stock Settings`, monitor first week |
| Bin assigned to multiple Job Cards (data corruption) | Low | Wave 3 already validates `is_consumed=0` before issue; Wave 5 marks consumed atomically |
| Batch # collisions if two MPEs submit simultaneously for same Lot | Medium | Use DB-level unique index on `(lot_no, batch_no)` |
| Real-time stock check is wrong (stale Console Stock Cache) | Medium | Force refresh from cache before validate; if cache age >5min, fail and ask operator to refresh |
| Rejection SE cascade (step 3 in Bridge sequence) fails silently | High | Bridge must surface this failure prominently; consider as separate transaction log entry per inspection SE submitted |
| Injection moulding auto-detect by name pattern is fragile | Medium | Add explicit `is_injection_moulding` field to Console Workstation master data; auto-fetch instead of regex |
| Job Card name from legacy doesn't roundtrip (different naming series in legacy vs ops) | Medium | Bridge must reconcile both Job Card names — store both `legacy_job_card` and `ops_job_card` separately |

## Files to be created (final list)

```
shree_polymer_console/shree_polymer_console/doctype/
├── console_moulding_production_entry/    (4 files)
├── console_moulding_balance_bin/         (3 files)
└── console_moulding_serial_no/           (3 files)

dash/src/components/processes/
└── MouldingProductionEntry.tsx           (adapt existing — likely full rewrite ~800 LOC)

apps/shree_polymer_legacy_bridge/sync_api.py
└── may need: support for chained __submit and call_method actions
```

Total: 3 new doctype directories (10 files), 1 dash component rewrite, possible bridge enhancement.

---

## Wave 5 milestone — what "done" means at the program level

When Wave 5 ships:

✅ Console operators can run a complete shop-floor shift on Console alone — from raw material receipt (DCR/MB Mixing) through compound creation, inspection, sheeting, blanking, planning, bin issue, and final moulding production booking.

✅ Every step has its own Console record (audit trail), syncs to both Legacy and Ops automatically, and survives partial sync failures.

✅ Master Batch Mixing → Moulding Production Entry chain is fully Console-native. Legacy DCR forms become read-only audit archives going forward.

✅ Downstream waves (Deflashing, Receive Deflashing, Sub Lot Creation, Lot Resource Tagging, Packing) become independent epics, each modeled on the same Wave 1-5 patterns.

This is the architectural cutover point.
