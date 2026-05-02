# Global Memory Index

Persistent facts about the user that apply across ALL projects, not just one.
Project-specific memory still lives at `~/.claude/projects/<project-key>/memory/`.

- [Identity & accounts](identity.md) — names, emails, GitHub handles, machine
- [Work & career](career.md) — employer, role, stack
- [Active projects](projects.md) — top-level inventory across repos
- [Working preferences](preferences.md) — how the user likes to be assisted
- [Cross-project rules](rules.md) — instructions that apply broadly

When project-specific memory disagrees with global memory, the project-specific
entry wins (e.g. `shree_polymer_console` branch rule overrides any general
"work on main" preference).
