# Wave 4 — Inspection Entry (MPE Precondition)

**Why fourth:** Moulding Production Entry **blocks submission** if Line Inspection AND Patrol Inspection don't exist for the lot. This is enforced by `validate_lot_number()` in legacy `moulding_production_entry.py:line ~129`. Console must record these inspections before MPE can ship.

**Outcome:** Operators can record Line/Patrol/Lot inspections during a moulding shift via the Console Dash. MPE submission then has the prerequisite records to validate against.

**Doctypes in this wave:** 1 parent + 1 child = 2 total

| Doctype | Type | Naming | Source legacy doctype |
|---|---|---|---|
| `Console Inspection Entry` | parent | `CIE-.YYYY.-.#####` | `Inspection Entry` |
| `Console Inspection Entry Item` | child | n/a | `Inspection Entry Item` |

---

## Why this wave is small but high-stakes

Despite being only 2 doctypes, this wave has **high-impact validation logic**:

1. **Type uniqueness** — only ONE Lot Inspection per Lot, only ONE Patrol Inspection per Lot, only ONE Line Inspection per Lot. Duplicates must throw.
2. **Lot referential integrity** — `lot_no` must reference a submitted `Console Add On Work Planning` (Wave 3 output).
3. **Rejection cascade** — when Line + Patrol both submitted, system must auto-flag rejection stock entries for later submission (after MPE generates batch_no).
4. **Pending state across systems** — inspection records sit in "ready but waiting" state until MPE ties them to a final batch_no.

---

## Prerequisite checks

```bash
bench --site sppconsole-dev.local console <<EOF
import frappe
for dt in ["Console Add On Work Planning", "Console Blank Bin Issue"]:
    assert frappe.db.exists("DocType", dt), f"Wave 3 incomplete: {dt} missing"
print("Wave 3 OK — proceeding to Wave 4")
EOF
```

---

## Step-by-step instructions

### Step 4.1 — Console Inspection Entry + Item (45 minutes)

**Parent — `Console Inspection Entry`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "INSPECTION_ENTRY") | yes |
| `inspection_type` | Select: `Lot Inspection`/`Patrol Inspection`/`Line Inspection` | | yes |
| `inspection_date` | Date | | yes |
| `inspection_time` | Time | | yes |
| `lot_no` | Data | | yes | The Job Card / Lot # from Wave 3 |
| `production_item` | Link | Console Item | (auto-fetch from Lot) | |
| `job_card` | Data | (read-only) | (auto-fetch) | |
| `inspector_id` | Link | Console Employee | yes |
| `one_no_qty_equal_kgs` | Float | (read-only, fetch from Item) | | Used for Kg conversion |
| `inspected_qty_nos` | Float | | yes | Pieces checked |
| `total_inspected_qty` | Float | (read-only, calc: nos × one_no_qty_equal_kgs) | | |
| `total_rejected_qty_nos` | Float | (read-only, sum of items) | | |
| `total_rejected_qty_kg` | Float | (read-only, calc) | | |
| `batch_no` | Data | (read-only, filled BY MPE on submit) | | The final batch_no |
| `stock_entry_reference` | Data | (filled when rejection SE materializes) | | |
| `items` | Table | Console Inspection Entry Item | yes | Defect breakdown |
| **All standard Console fields** | | | |

**Child — `Console Inspection Entry Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `defect_type` | Link | Console Defect Type | yes |
| `rejected_nos` | Float | | yes |
| `rejected_kg` | Float | (read-only, calc per row) | yes |
| `remarks` | Small Text | | no |

**Note on `Console Defect Type`:** if not already a master data doctype, add it as part of this wave (very small — just `defect_name` + `description`). Sync from legacy's existing defect catalog.

### Step 4.2 — Critical validate hooks (30 minutes)

```python
def validate(self):
    # 1. Lot must exist as a submitted Add On Work Planning
    planning = frappe.db.get_value(
        "Console Add On Work Plan Item",
        {"lot_number": self.lot_no, "docstatus": 1},
        ["item_produced", "job_card"],
        as_dict=True
    )
    if not planning:
        frappe.throw(f"Lot {self.lot_no} has no submitted Work Planning record")
    self.production_item = planning.item_produced
    self.job_card = planning.job_card

    # 2. Inspection type uniqueness — block duplicates
    existing = frappe.db.exists("Console Inspection Entry", {
        "lot_no": self.lot_no,
        "inspection_type": self.inspection_type,
        "docstatus": ["!=", 2],  # not cancelled
        "name": ["!=", self.name],
    })
    if existing:
        frappe.throw(
            f"{self.inspection_type} already exists for Lot {self.lot_no}: {existing}"
        )

    # 3. Inspected qty (nos) is mandatory and > 0 for Lot Inspection specifically
    if self.inspection_type == "Lot Inspection" and (not self.inspected_qty_nos or self.inspected_qty_nos <= 0):
        frappe.throw("Lot Inspection requires inspected_qty_nos > 0")

    # 4. Compute totals
    self.total_inspected_qty = (self.inspected_qty_nos or 0) * (self.one_no_qty_equal_kgs or 0)
    self.total_rejected_qty_nos = sum(flt(i.rejected_nos) for i in self.items)
    for i in self.items:
        i.rejected_kg = flt(i.rejected_nos) * flt(self.one_no_qty_equal_kgs)
    self.total_rejected_qty_kg = sum(flt(i.rejected_kg) for i in self.items)

def on_submit(self):
    self._emit_transaction_log()
    self._check_inspection_completion()

def _check_inspection_completion(self):
    """When Line + Patrol both submitted for same Lot, flag for rejection SE creation."""
    line_done = frappe.db.exists("Console Inspection Entry", {
        "lot_no": self.lot_no, "inspection_type": "Line Inspection", "docstatus": 1
    })
    patrol_done = frappe.db.exists("Console Inspection Entry", {
        "lot_no": self.lot_no, "inspection_type": "Patrol Inspection", "docstatus": 1
    })
    if line_done and patrol_done:
        # Mark both as "Ready for SE creation"
        # The rejection Stock Entry creation happens later, triggered by MPE submission
        # because batch_no isn't known yet
        for ie_name in [line_done, patrol_done]:
            frappe.db.set_value("Console Inspection Entry", ie_name, "rejection_se_pending", 1)
```

Add `rejection_se_pending` Check field to the parent schema (defaults 0).

### Step 4.3 — Update Dash (30 minutes)

```
dash/src/components/processes/
└── InspectionEntryEntry.tsx     ← NEW
```

Single screen handles all 3 inspection types via a Select dropdown at the top.

**Key UX:**
- Inspection Type dropdown (top)
- Scan Lot # → auto-fetch production_item, job_card, one_no_qty_equal_kgs
- Inspector scan
- Inspected Nos (mandatory)
- Items table: Defect Type (Searchable Select on Console Defect Type), Rejected Nos
- Auto-calc Rejected Kg per row, totals at bottom
- Submit only enabled when at least 1 defect row OR Line/Patrol type

Register:
```ts
'INSPECTION_ENTRY': InspectionEntryEntry,
```

### Step 4.4 — Bridge mapping (15 minutes)

Mapping is straightforward — `Console Inspection Entry` → `Inspection Entry` on legacy. 1:1 field map. The rejection Stock Entry generation happens *later* (after MPE submission), so this wave's Bridge step is a simple doc creation.

In `console_mapping_seeder.py`:
```python
def seed_inspection_entry_mapping():
    upsert_mapping(
        process_code="INSPECTION_ENTRY",
        target_system="Legacy",
        legacy_doctype="Inspection Entry",
        legacy_field_map={
            "inspection_type": "{{ inspection_type }}",
            "lot_no": "{{ lot_no }}",
            "inspected_qty_nos": "{{ inspected_qty_nos }}",
            "total_inspected_qty": "{{ total_inspected_qty }}",
            "items": "{{ items }}",
            # ... full map
        }
    )
    # Ops mapping similar
```

### Step 4.5 — Tests (30 minutes)

```python
def test_duplicate_inspection_type_blocked():
    # Submit a Line Inspection for Lot X, then try another → should throw
    ...

def test_lot_inspection_requires_qty():
    # Submit Lot Inspection with inspected_qty_nos=0 → should throw
    ...

def test_invalid_lot_blocked():
    # Submit with lot_no that doesn't exist in Add On Work Planning → should throw
    ...

def test_rejection_se_pending_flag_set():
    # Submit Line, then Patrol → both should have rejection_se_pending=1
    ...

def test_rejected_kg_calculated():
    # Submit with 5 rejected nos × 0.025 kg/no → rejected_kg=0.125
    ...
```

### Step 4.6 — End-to-end staging test (45 minutes)

Sequence (chains from Wave 3):

1. Have a Lot # from a recent Wave 3 Add On Work Planning submission
2. **Submit Line Inspection** for that Lot — verify Console record + both ERPs
3. **Submit Patrol Inspection** for the same Lot — verify both records have `rejection_se_pending=1`
4. Try to **resubmit** Line Inspection for same Lot → expect ValidationError
5. **Lot Inspection** — submit with inspected_qty_nos=500, defect rows totaling 12 rejected → verify totals

---

## Wave 4 Definition of Done — checklist

- [ ] 2 doctypes + master data `Console Defect Type` (if missing) created
- [ ] All validation hooks implemented (uniqueness, lot ref, qty math)
- [ ] 1 new Dash screen handling all 3 inspection types
- [ ] ≥5 unit tests passing
- [ ] Seeder updated for new mapping
- [ ] End-to-end test: Line + Patrol → both flagged for SE creation
- [ ] PR opened, reviewed, merged to `develop`

## Estimated time

- Coding (doctypes): 45 minutes
- Coding (Dash): 30 minutes
- Tests + seeders: 30 minutes
- Staging E2E: 45 minutes
- **Total: ~2.5 hours**

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| `Console Defect Type` master data not yet synced | Medium | Add to master data sync job before this wave |
| Patrol/Line/Lot uniqueness conflicts with concurrent shop floor entries | Low | Use DB-level unique index on `(lot_no, inspection_type)` for non-cancelled records |
| `one_no_qty_equal_kgs` field on Console Item not populated for all items | Medium | Add to master data sync; default to 0 with warning if missing |
| Bridge fails to create legacy Inspection Entry mid-shift → Console says success but legacy missing | Medium | Bridge already has retry logic; verify it covers Inspection Entry's mapping |
| Rejection Stock Entry creation depends on MPE submission — race if MPE submits before all 3 inspections done | Confirmed by docs | Wave 5 MPE validate must check that *required* inspections (Line + Patrol) exist; Lot Inspection is optional unless explicitly required |

## Files to be created (final list)

```
shree_polymer_console/shree_polymer_console/doctype/
├── console_inspection_entry/         (4 files)
├── console_inspection_entry_item/    (3 files)
└── console_defect_type/              (3 files, master data — only if not already exists)

dash/src/components/processes/
└── InspectionEntryEntry.tsx          (new ~500 LOC)
```

Total: 2-3 new doctype directories (~7-10 files), 1 new dash component.
