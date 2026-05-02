---
name: Cross-project rules
description: Instructions that apply broadly across all SPP projects
type: feedback
---

## Test gate before push (CRITICAL)

**Established:** 2026-04-30 during Phase 5 of the Console Inspection unification,
after the Compound Deviation race-condition saga where multiple "fixes" shipped
without proper test coverage.

**Rule:** No code goes out without all tests passing.

**Why:** Pushing untested code → cascading bugs on staging → user-visible
breakage. The Compound Deviation drift saga was the trigger — multiple
sequential "fixes" each introduced new bugs because none were test-verified.

**How to apply:**
1. Before `git push`, the relevant test suite must run successfully:
   - Console: `bench --site sppconsole-dev.local run-tests --module shree_polymer_console.shree_polymer_console.doctype.<doctype>.test_<doctype>`
   - Bridge: equivalent on the Bridge site
2. If the worktree has no bench attached (cannot run tests locally), do NOT
   push. Instead: commit locally, prepare the test command for the user,
   and wait for the user to either run tests themselves or grant explicit
   go-ahead.
3. Syntax check + JSON validation are NOT a substitute for tests.
4. If a test doesn't exist for the change, write one before pushing.

## Branch & PR conventions per repo

| Repo | Convention |
|---|---|
| `shree_polymer_console` | Work on `develop` directly. Push to `iyyanarr/develop`. Frappe Cloud auto-deploys. Only use `feat/*` for genuinely new features. |
| `shree_polymer_legacy_bridge` | Feature branches → PR to `master`. User manually syncs `iyyanarr` fork after merge. |
| `shree_polymer_ops` | Feature branches → PR to `main`. Never commit directly to `main`. The `develop` branch is stale; `main` is active. |

## Diff discipline (from `~/.claude/rules/diff-discipline.md`)

These three rules apply to every codebase, not just SPP:

1. **Surface uncertainty before coding** — if multiple plausible
   interpretations exist, list them and ask before picking silently.
2. **The diff test** — every changed line should trace directly to the user's
   request. Strip drive-by formatting changes.
3. **Plan-with-verify for multi-step tasks** — for non-trivial work, state a
   numbered plan with explicit verify steps before starting.

## Memory hygiene

When something stateful (like a generated artifact, cache, or graph) might
drift from current state, **proactively flag it** to the user. Don't wait
for them to ask. The user explicitly raised this expectation 2026-05-01
when noticing the graphify graph wasn't refreshed after multiple commits.
