# Diff Discipline

Three rules for tighter diffs and fewer wasted iterations. Cherry-picked from
Karpathy's LLM coding guidelines (2026-05) — the bits not already covered by
coding-style.md, testing.md, or the system prompt's "no scope creep" rules.

## Surface uncertainty BEFORE coding

When a request is ambiguous, don't pick silently:

- If multiple plausible interpretations exist, **list them and ask before
  choosing**. The cost of a clarifying question is one turn; the cost of
  building the wrong thing is hours.
- If a simpler approach exists than what the user asked for, say so before
  starting. **Push back when warranted** — being agreeable about an
  overcomplicated approach isn't helpful.
- If you find yourself guessing at a field name, file location, or business
  rule, stop and ask. Naming what you don't know is faster than searching.

This is NOT the same as "ask permission for every action." Trivial choices
should still be made. The rule applies when reasonable engineers would
disagree, not when the answer is obvious.

## The diff test

After making changes, before committing, run this test:

> **Every changed line should trace directly to the user's request.**

If you can't justify a line by pointing at the request, remove it. This kills:

- Drive-by formatting changes ("while I'm here let me fix this")
- Refactors of adjacent code that wasn't broken
- Comments explaining what well-named code already says
- Speculative error handling for cases that can't happen
- Renaming variables you didn't need to touch

This is the dual of the "no scope creep" rule in the system prompt — same
principle, applied as a post-hoc audit instead of a pre-commitment.

## Plan-with-verify for multi-step tasks

For non-trivial work (3+ logical steps, or anywhere you'd otherwise hand-wave
"and then I'll figure it out"), state a brief numbered plan with explicit
verify steps BEFORE starting:

```
1. [What I'll do]   → verify: [How I'll know it worked]
2. [What I'll do]   → verify: [How I'll know it worked]
3. [What I'll do]   → verify: [How I'll know it worked]
```

This is stronger than vanilla TodoWrite — it forces a verification gate per
step, so independence is bounded by checks instead of by trust. Strong
verify clauses let you loop autonomously; weak ones ("make it work") force
constant clarification.

Examples of strong verify clauses:
- "Run `bench migrate` clean"
- "Existing test X still passes; new test Y passes"
- "grep for old field name returns zero hits"
- "Three sample payloads round-trip correctly"

Examples of weak verify clauses (rewrite these):
- "It looks right"
- "Tests pass" (which tests?)
- "Working" (working how?)

## Origin

These three rules came from a one-time review of forrestchang/andrej-karpathy-skills.
The other rules in that file were already covered by my existing setup.
