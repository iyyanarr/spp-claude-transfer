# Integration Test Harness — TODO

**Status:** Parked (2026-04-30). Pick up after current urgent work.

## Goal

Per-process real-data integration tests that prove Console + Ops + Legacy all
hold the same truth after a submission. Catches the cross-system bugs that
unit tests can't (race conditions, payload drift, Stock Ledger mismatches).

## Site setup (confirmed)

| Role | Local site | Staging |
|---|---|---|
| Console | `sppconsole-dev.local` | `console.m.frappe.cloud` |
| Ops | `25spp.local` | `newspp.m.frappe.cloud` |
| Legacy | `spp15.local` | `sppmaster.frappe.cloud` |

## Deliverable scope (when resumed)

Files to create in `apps/shree_polymer_console/shree_polymer_console/integration_tests/`:

```
├── __init__.py
├── README.md                                      # runbook
├── runner.py                                      # generic harness
├── preflight_master_batch_mixing.py               # precondition checker
├── scenario_master_batch_mixing.py                # the actual test
├── fixtures/
│   └── seed_master_batch_mixing.py                # creates test items/BOM/stock
├── checks/
│   ├── console_checks.py
│   ├── ops_checks.py
│   └── legacy_checks.py
└── reports/                                       # gitignored, runtime output
```

## Confirmed test data names

- Produced (FG): `MB_TEST_6122`
- Raw (consumed): `B_TEST_6122`
- BOM: `BOM-MB_TEST_6122-001`
- Workstation: `Two Roll Mixing Mill`
- Operation: `Master Batch Mixing`
- Operator: `HR-EMP-00101` (role: Mixing Operator)
- Item Group: `Master Batch`
- Warehouse mapping: `Mixing - SPP` (Ops) ↔ `U3-Store - SPP INDIA` (Legacy)
- Test mix_barcode pattern: `TEST_MIX_<unix_ms>` (unique per run)

## Confirmed approach

1. **Start on local benches first** (`sppconsole-dev.local` + `25spp.local` + `spp15.local`).
2. **Auto-seed missing stock** via Material Receipt (don't just report).
3. **MB Mixing first**, then Final Batch Mixing, then the rest of Wave 1, then Waves 2–5 in order.

## Time estimate

- MB Mixing first time (incl. building framework): **~3.5 hours**
- Each subsequent process scenario: **~45 min** (framework reused, only scenario file + delta seeds change)
- Full Wave 1 (5 scenarios): **~5.5 hours total**

## Why this matters

Catches bugs unit tests can't:
- Sync race conditions (the kind we just fixed in apply_bridge_result)
- Stock Ledger Entry mismatches between Ops and Legacy
- Bridge sequence template drift (e.g. `{{qty}}` vs `transfer_qty`)
- Cross-system field drift (Console says batch X, Ops says batch Y)
- Auto-submit failing silently → records stuck Draft

## When to resume

After current urgent work clears. Reference:
- This file for scope
- Conversation transcript for full design (the "Deliverables for MB Mixing test" section)
