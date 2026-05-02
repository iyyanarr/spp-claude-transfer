# Global memory

Persistent facts about the user that apply across ALL projects live at
`~/.claude/memory/`. Read `~/.claude/memory/MEMORY.md` (the index) at the
start of each session, then load any specific file you need:

- `identity.md` — names, emails, GitHub handles
- `career.md` — employer (Shree Polymer Industries), role, stack
- `projects.md` — SPP Console ecosystem (4 inter-dependent Frappe apps)
- `preferences.md` — collaboration style, design philosophy
- `rules.md` — cross-project rules (test gate, branch conventions, diff discipline)

Project-specific memory (e.g. `shree_polymer_console` branch rule) lives at
`~/.claude/projects/<project-key>/memory/` and OVERRIDES global memory when
they conflict.

# graphify
- **graphify** (`~/.claude/skills/graphify/SKILL.md`) - any input to knowledge graph. Trigger: `/graphify`
When the user types `/graphify`, invoke the Skill tool with `skill: "graphify"` before doing anything else.
