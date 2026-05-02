# TEST RUNBOOK — Standard test cycle for the test machine

This is the playbook the test machine's Claude Code follows for every
test cycle the dev machine sends over.

---

## Cycle structure

```
1. Receive instruction from user (e.g., "verify commit 3ddb806")
2. Pull the commit on the test bench
3. Run targeted tests (specific module first, full suite if asked)
4. Report results in the structured format (see below)
5. If green: user pushes from dev. If red: investigate + report.
```

---

## Cycle 1 — The pending Console Inspection intent-aware aggregator fix

This is the first cycle. The dev machine has commit `3ddb806` ready to
push but blocked on tests.

### Step 1.1 — Pull the commit

```bash
cd ~/frappe-bench/apps/shree_polymer_console   # or whatever path
git fetch iyyanarr
git fetch --all  # if rsvasanth remote exists too

# The commit lives on dev machine's local develop only — the user needs to
# push it to a feature branch on rsvasanth/iyyanarr before you can fetch.
# OR copy the patch over manually:
#
# Option A: dev machine pushes to a temporary branch
#   git push iyyanarr develop:test/intent-aware-aggregator-3ddb806
#   (then on test machine:)
#   git fetch iyyanarr
#   git checkout iyyanarr/test/intent-aware-aggregator-3ddb806
#
# Option B: dev machine emails / shares the patch
#   git format-patch -1 3ddb806 -o /tmp/
#   (then on test machine:)
#   git apply /tmp/0001-fix-inspection-make-target_system-honest....patch

# Confirm you're on the right commit
git log --oneline -3
# Should show 3ddb806 (or its equivalent) at the top
```

### Step 1.2 — Migrate the test site (if doctype changes)

```bash
cd ~/frappe-bench
bench --site sppconsole-dev.local migrate
bench --site sppconsole-dev.local clear-cache
```

If migrate fails with a JSON error, the doctype JSON might be malformed.
Report immediately — DO NOT push.

### Step 1.3 — Run the targeted tests

```bash
cd ~/frappe-bench
bench --site sppconsole-dev.local run-tests \
    --module shree_polymer_console.shree_polymer_console.doctype.console_inspection.test_console_inspection
```

Or just the new aggregator class:

```bash
bench --site sppconsole-dev.local run-tests \
    --module shree_polymer_console.shree_polymer_console.doctype.console_inspection.test_console_inspection \
    --test TestConsoleInspectionIntentAwareAggregator
```

Capture full output.

### Step 1.4 — Report

Use the format in `KICKOFF.md`. If all 4 (or 8 with prior tests) pass:

```markdown
## Test report — commit 3ddb806

### Environment
- Site: sppconsole-dev.local
- Branch: <branch name>
- HEAD: 3ddb806

### Tests run
TestConsoleInspectionIntentAwareAggregator (4 tests)
TestConsoleInspection (4 prior tests, regression check)

### Results
✅ All 8 tests passed in <X>s

### Recommendation
Safe to push. Dev machine: `git push iyyanarr develop`
```

If any test fails:

```markdown
## Test report — commit 3ddb806

### Environment
- Site: sppconsole-dev.local
- Branch: <branch name>
- HEAD: 3ddb806

### Tests run
TestConsoleInspectionIntentAwareAggregator (4 tests)

### Results
✅ test_rejected_ci_with_ops_only_intent_auto_submits_when_ops_syncs
❌ test_accepted_ci_with_both_intent_still_requires_both_legs
✅ test_legacy_only_intent_works_symmetrically
✅ test_failed_intended_leg_marks_overall_failed

### Failure detail
```
File "test_console_inspection.py", line 287, in test_accepted_ci_with_both_intent_still_requires_both_legs
    self.assertEqual(partial.sync_status, "Partial")
AssertionError: 'Pending' != 'Partial'
```

### Diagnosis
The aggregator returns "Pending" instead of "Partial" when only one leg is
Synced and intent is "Both". Looks like the new intent-aware code path
treats "ops_partial" as "ops_required and ops_st == 'Synced'" but doesn't
account for legacy_st being None (vs "Pending"). Check api.py line ~1043
where target_system gets set — possibly target_system isn't being written
in time.

### Recommendation
HOLD push. Dev machine should investigate the Pending vs Partial logic.
```

---

## Cycle 2+ — General test runs

When the user says "verify <commit-or-branch>":

1. **Pull**: `git fetch && git checkout <ref>`
2. **Migrate** (if any doctype JSON or schema changed):
   `bench --site sppconsole-dev.local migrate`
3. **Identify the test module** from the commit message (look for "Tests added:" section)
4. **Run targeted tests** first — fail fast, isolate diagnostics
5. **If targeted passes, run regression**: `bench run-tests --app shree_polymer_console`
6. **Report** in the structured format
7. **DO NOT push** — that's the dev machine's call

---

## Special cycles

### Bridge end-to-end test (real spp15 + 25spp data)

After Console-side tests pass, the dev machine may ask you to verify the
ACTUAL Bridge sync end-to-end with real production data:

```bash
# 1. On Console site, submit a test record via API
bench --site sppconsole-dev.local console <<'PY'
import frappe
import time

ref_id = str(int(time.time() * 1000))
# Pick a real Rejected CI batch from production data...
# (the user will provide the specific batch_no / mix_barcode)

doc = frappe.get_doc({
    "doctype": "Console Inspection",
    "inspection_type": "COMPOUND_INSPECTION",
    "reference_id": ref_id,
    "mix_barcode": "<real_mix_barcode>",
    "spp_batch_no": "<real_spp_batch>",
    "item_code": "<real_item>",
    "batch_no": "<real_batch>",
    "posting_date": frappe.utils.today(),
    "inspector_id": "<real_inspector_id>",
    "status": "Rejected",
    "target_system": "Ops",
    "readings": [
        {"specification": "Hardness", "reading_value": "55", ...},
        # ... 4 readings
    ],
})
doc.flags.from_local_cache = True
doc.insert(ignore_permissions=True)
print(f"Created: {doc.name}, ref_id: {ref_id}")
PY

# 2. Verify on Ops site
bench --site 25spp.local console <<'PY'
import frappe
qi = frappe.get_all("Quality Inspection",
    filters={"reference_name": "<the SE this CI references>"},
    fields=["name", "status", "creation"])
print(qi)
PY

# 3. Confirm Legacy was NOT touched (Rejected → Ops only)
bench --site spp15.local console <<'PY'
import frappe
ci = frappe.db.exists("Compound Inspection", {"scan_compound": "<mix_barcode>"})
assert not ci, "Legacy should NOT have a CI for a Rejected one"
print("Confirmed: Legacy correctly skipped")
PY
```

Report all three results in your structured format.

### Smoke test of the Compound Deviation queue

Once the intent-aware fix is in, a real Rejected CI should auto-submit
and appear in the Deviation queue:

```bash
# 1. Trigger apply_bridge_result for the Ops leg of an existing record
bench --site sppconsole-dev.local console <<'PY'
import frappe
doc = frappe.get_doc("Console Inspection", "INSP-2026-00001")
doc.db_set("target_system", "Ops", update_modified=False)
doc.apply_bridge_result(target="Ops", status="Success",
                        remote_name=doc.ops_reference or "BACKFILL")
frappe.db.commit()

# 2. Verify state
print(f"sync_status: {doc.sync_status}")
print(f"docstatus: {doc.docstatus}")
assert doc.sync_status == "Success"
assert doc.docstatus == 1

# 3. Verify it appears in deviation queue
from shree_polymer_console.shree_polymer_console.api import get_rejected_compound_inspections
result = get_rejected_compound_inspections()
matching = [r for r in result["data"] if r["name"] == "INSP-2026-00001"]
assert len(matching) == 1, "Should appear in Deviation queue after auto-submit"
print(f"✅ Found in queue: {matching[0]}")
PY
```

---

## What to do when a test fails

1. **DON'T push.**
2. **Capture full traceback** (last 30 lines of `bench run-tests` output).
3. **Identify the exact assertion** that failed.
4. **Inspect related state**: query the DB for the record involved, check
   the Console Transaction Log, check the Bridge Log on Legacy.
5. **Hypothesize a cause** based on what you know about the architecture.
6. **Report all of the above** — the dev machine's Claude will pattern-match
   your diagnosis against the code and propose a patch.

---

## What to do when tests pass

1. **Confirm the bench is in a clean state**:
   ```bash
   bench --site sppconsole-dev.local console -c "import frappe; frappe.db.rollback(); print('clean')"
   ```
2. **Note the Frappe + Python versions**:
   ```bash
   bench version | head -5
   python3 --version
   ```
3. **Report green** with the structured format from KICKOFF.md.
4. **Wait for the user to push from dev machine** (don't push yourself).

---

## Lifecycle hygiene

- **After each test cycle**: clean up any test records you created via DB queries:
  ```bash
  bench --site sppconsole-dev.local console <<'PY'
  import frappe
  for ref in frappe.get_all("Console Inspection",
      filters={"reference_id": ["like", "TEST-%"]}, pluck="name"):
      frappe.delete_doc("Console Inspection", ref, force=1)
  frappe.db.commit()
  PY
  ```
- **After 5-10 cycles**: refresh the graphify graph if you're using it for analysis:
  ```bash
  cd /path/to/.graphify_spp_workspace
  graphify --update
  ```
- **Daily**: `bench --site sppconsole-dev.local clear-cache` to avoid stale doctype metadata.
