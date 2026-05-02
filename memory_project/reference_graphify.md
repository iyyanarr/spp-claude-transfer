---
name: SPP graphify graph location
description: Where the graphify knowledge graph for the 4 SPP apps lives + when to refresh it
type: reference
originSessionId: 8147e470-c106-4fae-9f1b-fc2bd5f1ac67
---
**Location:** `/Users/alphaworkz/frappe-bench/.graphify_spp_workspace/graphify-out/`

**Scope:** 4 Frappe apps via symlink (follow_symlinks=True):
- shree_polymer_custom_app  (Legacy doctypes, 53-field Inspection Entry, MPE etc.)
- shree_polymer_console     (Console app + dash React)
- shree_polymer_ops         (MES Transaction Engine on Ops)
- shree_polymer_legacy_bridge  (Bridge: sync_api.py, populate_*_fields)

**Files:**
- `graph.json`         — raw graph (NetworkX node-link, ~5MB)
- `graph.html`         — interactive viz, opens in any browser
- `GRAPH_REPORT.md`    — community labels, god nodes, surprising connections
- `manifest.json`      — file fingerprint for `--update` incremental rebuilds
- `cost.json`          — token usage per run

**Refresh triggers:**
- After every wave/phase completion (e.g. Phase 1 inspection unification, Wave 2)
- After Bridge sync_api.py refactors (the populate_* helpers move around)
- Before any major design analysis (Pattern D decisions, gap analysis)
- When someone asks "show me what changed since last time" — the graph IS the diff

**How to refresh** (incremental — only re-extracts changed files):
```bash
cd /Users/alphaworkz/frappe-bench/.graphify_spp_workspace
graphify --update
```

**Force full rebuild** (if symlinks change, or after >50 file churn):
```bash
cd /Users/alphaworkz/frappe-bench/.graphify_spp_workspace
rm -rf graphify-out/
graphify .
```

**Known constraints:**
- Initial run on 2026-04-29 was AST-only because org monthly usage limit blocked
  the semantic extraction agents. The graph captures structural relationships
  (imports, classes, function defs) but NOT LLM-inferred call/data edges.
- Semantic extraction can be retried later via `--update --mode deep` once
  usage capacity is available.

**Drift watch:** if the most recent commit on `iyyanarr/develop` (Console) or
`rsvasanth/feat/*` (Bridge) is more than ~5 commits ahead of the date in
`cost.json`, the graph is meaningfully stale and should be refreshed before
using it for analysis.
