---
name: PR workflow for shree_polymer_ops
description: Branch and PR convention for the shree_polymer_ops repo
type: feedback
originSessionId: adb11a56-d1c0-4f2f-85d1-204add01f4dc
---
Always use a feature branch → PR workflow for shree_polymer_ops changes. Never commit directly to `main`.

**Why:** User confirmed this is the desired approach (April 2026). The `develop` branch is stale; `main` is the active base branch.

**How to apply:**
1. Before starting work: `git checkout -b feat/<short-description>` off `main`
2. Commit work to the feature branch
3. Push with `git push -u origin feat/<short-description>`
4. Create PR targeting `main` using `gh pr create --base main`
5. Report PR URL wrapped in `<pr-created>` tag

Branch naming: `feat/`, `fix/`, `refactor/` prefixes matching conventional commit types.
