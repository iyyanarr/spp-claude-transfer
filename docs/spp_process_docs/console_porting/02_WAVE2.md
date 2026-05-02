# Wave 2 — Blanking Chain (Cut Bits/Clips → Blank Bins → Moulding Floor)

**Why second:** Wave 1 lands the upstream (compound creation, inspection, sheeting). Wave 2 turns those outputs into the blank bins that Moulding consumes. Without this, the Moulding chain has no input.

**Outcome:** Console can record the full path from Sheeting/Cut Bit Transfer → Blanking → Bin arrival at Moulding Unit.

**Doctypes in this wave:** 3 parents + 3 children = 6 total

| Doctype | Type | Naming | Source legacy doctype |
|---|---|---|---|
| `Console Bulk Clip Release` | parent | `CBCR-.YYYY.-.#####` | `Bulk Clip Release` |
| `Console Clip Release Item` | child | n/a | `Clip Release Item` |
| `Console Blanking DC Entry` | parent | `CBDE-.YYYY.-.#####` | `Blanking DC Entry` |
| `Console Blanking DC Item` | child | n/a | `Blanking DC Item` |
| `Console Blank Bin Inward Entry` | parent | `CBBIE-.YYYY.-.#####` | `Blank Bin Inward Entry` |
| `Console Blank Bin Inward Item` | child | n/a | `Blank Bin Inward Item` |

---

## Prerequisite checks (run these BEFORE Wave 2)

```bash
# Wave 1 must be on develop and migrated
cd /Users/alphaworkz/frappe-bench
git -C apps/shree_polymer_console log --oneline | grep -i "wave 1"
bench --site sppconsole-dev.local console <<EOF
import frappe
for dt in ["Console Master Batch Mixing", "Console Cut Bit Transfer"]:
    assert frappe.db.exists("DocType", dt), f"Wave 1 incomplete: {dt} missing"
print("Wave 1 OK — proceeding")
EOF
```

If Wave 1 isn't done, STOP and finish it first. Wave 2 child tables reference `Console Cut Bit Item` indirectly via Bulk Clip Release.

---

## Step-by-step instructions

### Step 2.1 — Console Bulk Clip Release + Item (20 minutes)

**Why this is in Wave 2 (not Wave 1):** Bulk Clip Release was originally part of Wave 1 in our earlier conversation, but it logically belongs *between* Sheeting and Blanking — clips produced from Sheeting → Bulk Clip Release → released into the Blanking pool. Moving it here keeps the Blanking chain together.

**Parent — `Console Bulk Clip Release`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "BULK_CLIP_RELEASE") | yes |
| `release_date` | Date | | yes |
| `released_from_warehouse` | Link | Console Warehouse | yes |
| `released_to_warehouse` | Link | Console Warehouse | yes |
| `released_clips` | Table | Console Clip Release Item | yes |
| `total_qty` | Float | (read-only, sum of clips) | |
| `operator_id` | Link | Console Employee | yes |
| **All standard Console fields** | | | |

**Child — `Console Clip Release Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `clip_barcode` | Data | | yes |
| `item_code` | Link | Console Item | yes |
| `batch_no` | Data | | yes |
| `spp_batch_no` | Data | | no |
| `qty` | Float | | yes |
| `source_clip` | Link | Console Sheeting Clip | no | links back to Wave 1 sheeting output |

**Validate:**
- `released_clips` must be non-empty
- Each `clip_barcode` must exist in `Console Sheeting Clip` and not already be released
- `total_qty` is recalculated on validate

### Step 2.2 — Console Blanking DC Entry + Item (30 minutes)

**Parent — `Console Blanking DC Entry`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "BLANKING_DC_ENTRY") | yes |
| `item_produced` | Link | Console Item | yes | The Blank/Mat being made |
| `posting_date` | Date | | yes |
| `scan_clip` | Data | (input only, not stored) | no |
| `scan_bin` | Data | (input only, not stored) | no |
| `scanned_item` | Data | | yes | Compound from clip |
| `available_quantity` | Float | | yes | From source clip |
| `spp_batch_number` | Data | | yes | From clip |
| `bin_code` | Link | Console Asset | yes | Target Blanking Bin |
| `gross_weight_kgs` | Float | | yes | Material + Bin |
| `bin_tare_weight` | Float | | yes | Empty bin weight (from Asset) |
| `net_weight_kgs` | Float | (read-only, calc) | yes | Material only |
| `clip_source` | Link | Console Sheeting Clip | yes | The clip being consumed |
| `bom_validated` | Check | | | Set by validate hook |
| `items` | Table | Console Blanking DC Item | yes | Produced blanks list |
| **All standard Console fields** | | | |

**Child — `Console Blanking DC Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `item_code` | Link | Console Item | yes |
| `qty` | Float | | yes |
| `bin_code` | Link | Console Asset | yes |
| `t_warehouse` | Link | Console Warehouse | yes |
| `is_finished_item` | Check | (default 1) | |

**Critical validate hook (mirror legacy):**
- `net_weight_kgs == gross_weight_kgs - bin_tare_weight`
- `net_weight_kgs <= available_quantity` (can't blank more than the clip holds)
- `clip_source.qty -= net_weight_kgs` should be valid (Bridge handles actual stock movement)
- BOM validation: produced item should have a BOM that consumes the input compound
- `bin_code` Asset must be in status "Available" (not already filled)

### Step 2.3 — Console Blank Bin Inward Entry + Item (25 minutes)

**Parent — `Console Blank Bin Inward Entry`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `process_code` | Data | (read-only, "BLANK_BIN_INWARD") | yes |
| `posting_date` | Date | | yes |
| `blank_bin` | Data | (input only, not stored after scan) | no |
| `move_to_cut_bit_warehouse` | Check | | | If checked, repacks bin → cut bit batch |
| `scan_cut_bit_batch` | Data | (input only) | no | (only if move_to_cut_bit_warehouse) |
| `bin_code` | Link | Console Asset | yes |
| `item` | Link | Console Item | yes | Compound inside |
| `spp_batch_number` | Data | | yes | Validation batch |
| `gross_weight_kgs` | Float | | yes |
| `net_weight_kgs` | Float | (read-only, calc) | yes |
| `target_warehouse` | Link | Console Warehouse | yes | Moulding Unit warehouse |
| `items` | Table | Console Blank Bin Inward Item | yes |
| **All standard Console fields** | | | |

**Child — `Console Blank Bin Inward Item`:**

| Field | Type | Options | Reqd |
|---|---|---|---|
| `bin_code` | Link | Console Asset | yes |
| `item_code` | Link | Console Item | yes |
| `qty` | Float | | yes |
| `s_warehouse` | Link | Console Warehouse | yes |
| `t_warehouse` | Link | Console Warehouse | yes |
| `spp_batch_number` | Data | | no |

**Branching logic in validate:**
- IF `move_to_cut_bit_warehouse == 1`:
  - `scan_cut_bit_batch` is required
  - `target_warehouse` defaults to Cut Bit Warehouse
  - On submit: Bridge creates a Stock Entry (Repack) instead of Asset Movement (Receive Bin)
- ELSE:
  - On submit: Bridge creates an Asset Movement (Receive Bin) — bin physically arrives at Moulding Unit

### Step 2.4 — Update Dash (1 hour)

Wave 1 doctypes mapped to existing Dash screens. Wave 2 needs **3 NEW screens** (none exist yet):

```
dash/src/components/processes/
├── BulkClipReleaseEntry.tsx          ← already exists (per processRegistry); confirm wired to new doctype
├── BlankingDCEntry.tsx               ← NEW (build this)
└── BlankBinInwardEntry.tsx           ← NEW (build this)
```

For BulkClipReleaseEntry — verify it submits with `process_code: "BULK_CLIP_RELEASE"` and the payload matches the new doctype's field names. Update if drift.

For the 2 new screens, mirror the pattern from `MasterBatchMixingEntry.tsx`:
- Operator scan (Mixing Operator role for BDE, Material Handler role for BBIE)
- Source/target warehouse resolved from `template?.warehouses`
- Scan inputs (clip/bin/cut bit)
- Items table with editable rows
- `buildPayload()` returning the standard shape from Wave 1 with `process_code` set
- Submit via `initialize_submission` → `execute_submission_step` (same as MB Mixing)

Register in `processRegistry.tsx`:
```ts
'BLANKING_DC_ENTRY': BlankingDCEntry,
'BLANK_BIN_INWARD': BlankBinInwardEntry,
```

### Step 2.5 — Update seeders (15 minutes)

For each of the 3 new parents, add to `console_template_seeder.py`:
- A `Console Process Template` row
- Required field list (so Dash form validation works)
- Warehouse mappings (Source/Target/Scrap as applicable)

For each, add to `console_mapping_seeder.py`:
- `Legacy Sync Mapping` for `Bulk Clip Release` → `Bulk Clip Release` (same name in legacy)
- `Legacy Sync Mapping` for `Blanking DC Entry` → `Blanking DC Entry`
- `Legacy Sync Mapping` for `Blank Bin Inward Entry` → `Blank Bin Inward Entry` (with branching for cut-bit-warehouse mode)

### Step 2.6 — Migration + tests (15 minutes)

```bash
bench --site sppconsole-dev.local migrate
bench --site sppconsole-dev.local run-tests --module shree_polymer_console.shree_polymer_console.doctype.console_blanking_dc_entry.test_console_blanking_dc_entry
# repeat for the other 2 parents
```

### Step 2.7 — End-to-end staging test (1 hour)

Sequence — must run in this order to chain the data through:

1. **Bulk Clip Release** — pick a clip from a recent Sheeting submission (Wave 1 output), release it
2. **Blanking DC Entry** — scan that released clip, blank into a bin asset
3. **Blank Bin Inward Entry** — receive that bin at the Moulding Unit warehouse

Verify after each step:
- Console record `sync_status = Success`
- Both Legacy + Ops Transaction Log entries `Success`
- The asset (bin) shows the right Item + qty + warehouse on both ERPs

---

## Wave 2 Definition of Done — checklist

- [ ] All 6 doctypes created and migrated
- [ ] 3 new Dash screens (BulkClipRelease ✓, BlankingDC, BlankBinInward) implemented and registered
- [ ] Each parent has ≥2 unit tests passing
- [ ] Seeders updated and idempotent on re-run
- [ ] End-to-end staging test passes for all 3 processes in sequence
- [ ] PR opened, reviewed, merged to `develop`

## Estimated time

- Coding (doctypes): ~75 minutes
- Coding (Dash): ~60 minutes
- Tests + seeders: ~30 minutes
- Staging E2E: ~60 minutes
- **Total: ~3.5 hours**

## Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| `Console Sheeting Clip` doesn't track `is_released` flag yet | High | Add `is_released` field as part of Wave 2; update on Bulk Clip Release submit |
| BOM validation in Blanking DC Entry needs `Console BOM` to have full bidirectional links | Medium | Verify `Console BOM` already has the parent→item linkage; add if missing |
| Asset Movement is an ERPNext-native doctype, not custom — Console doesn't mirror it | Confirmed | Console only triggers AM via Bridge; doesn't store AM records |
| Branching logic in BBIE (move_to_cut_bit toggle) routes to different Stock Entry types | Medium | Bridge mapping for BBIE needs two variant configs; flag this in mapping seeder |

## Files to be created (final list)

```
shree_polymer_console/shree_polymer_console/doctype/
├── console_bulk_clip_release/         (4 files)
├── console_clip_release_item/         (3 files)
├── console_blanking_dc_entry/         (4 files)
├── console_blanking_dc_item/          (3 files)
├── console_blank_bin_inward_entry/    (4 files)
└── console_blank_bin_inward_item/     (3 files)

dash/src/components/processes/
├── BlankingDCEntry.tsx                (new ~600 LOC)
└── BlankBinInwardEntry.tsx            (new ~500 LOC)
```

Total: 6 new doctype directories (21 files), 2 new dash components.
