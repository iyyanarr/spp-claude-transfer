---
name: Working preferences
description: How the user likes to be assisted — collaboration style, design philosophy
type: feedback
---

## Design philosophy

- **Architectural rigor over speed.** Wants design decisions thought through
  before code. Asks for system-architect mode when designing larger structures.
- **Production-grade fixes, not band-aids.** When given the choice between
  "quick fix" and "clean refactor", consistently picks the clean refactor.
- **Bridge-first design.** When designing data shape, work backward from the
  Bridge sequence (what gets created on each system) rather than schema-first.
  Lesson learned: "getting the Bridge orchestration right is what matters most."
- **Pattern D for polymorphic doctypes:** normalized header + polymorphic child
  table + small JSON escape hatch for type-specific overflow. Chosen over fat
  doctype with conditional visibility, JSON blob, or per-type extension docs.
- **Single source of truth.** When two code paths drift (like the duplicated
  4-pass SE lookup), unify them rather than maintaining both.

## Collaboration style

- **Wants direct opinions and pushback when warranted.** "Tell me what you
  really think." Hedging and over-qualifying frustrates.
- **Concise, verifiable answers.** Prefers tables, decision trees, and explicit
  next-steps over long prose.
- **One question at a time when designing.** Said this explicitly during the
  Console Inspection design conversation — wants linear progression through
  decisions, not menus of options.
- **Cherry-pick external rule sets** rather than plugin-installing them
  wholesale. Took only 3 genuinely-new rules from
  forrestchang/andrej-karpathy-skills into a focused
  `~/.claude/rules/diff-discipline.md` instead of installing the whole CLAUDE.md.

## Quality gates

- **Test discipline:** Established 2026-04-30 — no code goes out without all
  tests passing first. Syntax+JSON checks are NOT a substitute. See
  `~/.claude/memory/rules.md` for full rule.
- **Memory hygiene:** Wants stale state proactively flagged. Asked whether
  the stale graphify graph was documented (it wasn't; was added on request).
- **Migration strategy:** Hard replacement preferred. When introducing a new
  doctype that supersedes 3 old ones, ignore old records and start fresh
  rather than running a migration script.

## Communication signals

- "ill take your recommendation" → user has approved the most recent
  recommendation; proceed with that exact plan, don't reconfirm.
- "what i would expect you delver for this test" → user wants concrete file
  paths, file lists, and test plans, not abstract descriptions.
- "no code goes out with all test passed" → strict test gate, treat as
  non-negotiable.
- "kindly fix it" → proceed with the fix, no further confirmation needed.
- Asks "what do you think?" → wants honest opinion with rationale, not
  agreement.
