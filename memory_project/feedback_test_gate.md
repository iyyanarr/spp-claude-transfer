---
name: Test gate before push
description: User's hard rule — no code change ships to a remote without tests passing first
type: feedback
originSessionId: 8147e470-c106-4fae-9f1b-fc2bd5f1ac67
---
**Rule: No code goes out without all tests passing.**

**Why:** User wants a discipline gate before any push to `iyyanarr/develop` (Console)
or `rsvasanth/feat/*` (Legacy Bridge). Pushing untested code → cascading bugs on
staging → user-visible breakage like the Compound Deviation drift saga.

**How to apply:**
1. Before `git push`, the relevant test suite must run successfully:
   - For Console-app changes: `bench --site sppconsole-dev.local run-tests --module shree_polymer_console.shree_polymer_console.doctype.<doctype>.test_<doctype>`
   - For Bridge changes: equivalent on the Bridge site
2. If the worktree has no bench attached (cannot run tests locally), do NOT push.
   Instead: commit locally, prepare the test command for the user, and wait for
   the user to either run tests themselves or grant explicit go-ahead.
3. Syntax check + JSON validation are NOT a substitute for tests.
4. If a test doesn't exist for the change, write one before pushing.

**Established:** 2026-04-30 during Phase 5 of the Console Inspection unification,
after the Compound Deviation race-condition saga where multiple "fixes" shipped
without proper test coverage.
