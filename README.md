# SPP Claude Transfer Bundle

Context bundle to bootstrap Claude Code on a new machine working on the
Shree Polymer Console project ecosystem.

**Audience:** A second Claude Code workstation (typically the production-replica
test machine) that needs the same memory, rules, and project docs as the
dev machine.

## Quick start (on the destination machine)

```bash
git clone https://github.com/<owner>/spp-claude-transfer.git
cd spp-claude-transfer
# Read 01_SETUP.md and follow the 7 steps
```

## Files

| File | Purpose |
|---|---|
| `KICKOFF.md` | First message to feed the destination machine's Claude Code |
| `01_SETUP.md` | One-time setup steps (memory install, sed-replace, verify) |
| `TEST_RUNBOOK.md` | Standard test-cycle playbook for the test machine |
| `CLAUDE_global_index.md` | Reference for `~/.claude/CLAUDE.md` |
| `manifest.json` | File inventory with SHA-256 checksums |
| `memory_global/` | 6 files → install to `~/.claude/memory/` |
| `memory_project/` | 4 files → install to `~/.claude/projects/<key>/memory/` |
| `rules/` | 9 files → install to `~/.claude/rules/` |
| `docs/spp_process_docs/` | Process docs, porting plan, factory testing → sibling of `frappe-bench/` |

## Updating

When the dev machine has new memory or docs to share, refresh and push:

```bash
# On dev machine:
cd /Users/alphaworkz/spp-claude-transfer
# (re-run the bundle creation script — see /Users/alphaworkz/.claude/sessions/ for the prep commands)
git add -A
git commit -m "refresh: <what changed>"
git push
```

On the test machine: `git pull` and re-run any setup steps that touch
the changed files (typically `01_SETUP.md` Step 2 + Step 3).

## Privacy

**This repo is PRIVATE.** It contains internal SPP project context including
branch names, business rules, and architectural decisions. Do not make public.
