---
name: Active projects
description: Top-level inventory across SPP repos and tooling
type: project
---

## SPP Console ecosystem (4 inter-dependent Frappe apps)

### shree_polymer_console
Primary Console MES app. Frappe Python backend + React/TypeScript dash UI
(built into `shree_polymer_console/public/dash/assets/`). Includes Bridge
orchestrator that fans transactions out to Legacy + Ops in parallel via
`initialize_submission` / `execute_submission_step` / `apply_bridge_result`.
- **Repo:** `iyyanarr/shree_polymer_console` (deploy) ← `rsvasanth/shree_polymer_console` (dev)
- **Branch convention:** work directly on `develop` (CRITICAL rule); only use `feat/*` for genuine new features
- **Current work:** Console Inspection unification (Phases 1-4 deployed 2026-05-01; Phase 5 gated on tests; Phase 6 pending)

### shree_polymer_legacy_bridge
Sync engine on the Legacy ERPNext bench. Reads Console payloads, creates
corresponding Legacy docs (Compound Inspection, Stock Entry, etc.).
- **Repo:** `iyyanarr/shree_polymer_legacy_bridge` (deploy) ← `rsvasanth/shree_polymer_legacy_bridge` (dev)
- **Branch convention:** feature branches → PR to `master`; user manually syncs iyyanarr fork after merge
- **Recent:** 2026-04-30 refactor (commit `1df2de8`) extracted 5 helpers — `_resolve_compound_identity`, `_lookup_legacy_stock_entry`, `_apply_compound_identity_to_doc`, `_apply_compound_readings_to_doc`, `_apply_deviation_extras` — and collapsed populate_compound_inspection_fields + populate_compound_deviation_fields into thin wrappers. Eliminates ~200 lines of duplicated 4-pass SE lookup logic. Deployed 2026-05-02.

### shree_polymer_ops
Ops-side ERPNext v15 app containing the MES Transaction Engine. Receives
Bridge POSTs, creates Stock Entry / Quality Inspection records.
- **Repo:** `iyyanarr/shree_polymer_ops`
- **Branch convention:** PR to `main` (per saved feedback)

### shree_polymer_custom_app
Legacy custom app on the older ERPNext v14 install. Defines 100+ doctypes
including `Compound Inspection`, `Inspection Entry` (53-field polymorphic
with 6 inspection_types), `Moulding Production Entry`, `Delivery Challan
Receipt`. Source of truth referenced by the Console unification design.

## Site setup
Three-site bench mirrors three Frappe Cloud staging URLs:

| Role | Local | Staging |
|---|---|---|
| Console | `sppconsole-dev.local` | `console.m.frappe.cloud` |
| Ops | `25spp.local` | `newspp.m.frappe.cloud` |
| Legacy | `spp15.local` | `sppmaster.frappe.cloud` |

- Built dash assets MUST be committed (Frappe Cloud doesn't run yarn build)
- GitHub Actions `build-dash.yml` auto-builds + commits assets on push

## Tooling

### graphify knowledge graph
Located at `/Users/alphaworkz/frappe-bench/.graphify_spp_workspace/graphify-out/`
Symlinks 4 SPP apps via follow_symlinks=True. Refresh trigger: after every
wave/phase completion. See project memory `reference_graphify.md` for details
including the `detect_incremental` symlinks bug + workaround.

### Console Inspection Unification Plan
Wave/phase porting plan documented in
`/Users/alphaworkz/frappe-bench/spp_process_docs/console_porting/`:
00_PORTING_PLAN.md, 01_WAVE1.md … 06_INTEGRATION_TESTS_TODO.md.
Goal: port the production chain Master Batch Mixing → Moulding Production Entry
to Console-local doctypes via the unified `Console Inspection` doctype
(replaces 3 per-type doctypes).
