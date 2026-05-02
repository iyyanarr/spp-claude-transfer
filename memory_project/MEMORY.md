# Memory Index

- [PR workflow for shree_polymer_ops](feedback_pr_workflow.md) — Always use feat/* branches off main, PR to main; never commit directly to main
- [Test gate before push](feedback_test_gate.md) — No code goes out without all tests passing first; syntax+JSON checks are NOT a substitute
- [SPP graphify graph location](reference_graphify.md) — Graph lives at .graphify_spp_workspace/graphify-out/; refresh with `graphify --update` after each wave/phase

## shree_polymer_console Branch Rule (CRITICAL)

**ALWAYS work on the `develop` branch** for shree_polymer_console.
- Only switch to a `feat/*` branch when explicitly adding a **new feature**
- All bug fixes, tweaks, and improvements go directly to `develop`
- Push to `iyyanarr/develop` (the fork used by Frappe Cloud)
- Frappe Cloud deploys from `iyyanarr/shree_polymer_console:develop`

## Architecture Notes

- Three-site setup: sppconsole-dev.local (Console React), 25spp.local (Ops ERPNext v15), spp15.local (Legacy ERPNext v14)
- Built assets (shree_polymer_console/public/dash/assets/) must be committed — Frappe Cloud doesn't run `yarn build`
- GitHub Actions CI (`build-dash.yml`) auto-builds and commits assets on push
