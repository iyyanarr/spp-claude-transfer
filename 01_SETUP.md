# SETUP — Manual steps for the human on the test machine

Run these commands ONCE on the test machine before starting Claude Code there.
Estimated time: 5 minutes.

---

## Step 1 — Clone the bundle from GitHub

The transfer bundle lives at https://github.com/iyyanarr/spp-claude-transfer
(private repo — make sure you're signed in to GitHub as a collaborator).

On the **test machine**:

```bash
cd ~
git clone https://github.com/iyyanarr/spp-claude-transfer.git
cd spp-claude-transfer
ls
# Should show: KICKOFF.md, 01_SETUP.md, TEST_RUNBOOK.md, README.md,
#              memory_global/, memory_project/, rules/, docs/,
#              CLAUDE_global_index.md, manifest.json
```

> **GitHub auth:** if the clone prompts for credentials, use a Personal
> Access Token (PAT) with repo scope. Generate at
> https://github.com/settings/tokens → "Tokens (classic)" → Generate new
> token. The PAT is your password when git asks.

### Future updates

When the dev machine pushes new context (refreshed memory, new test plans,
updated docs), pull on the test machine:

```bash
cd ~/spp-claude-transfer
git pull
# Then re-run any setup steps below that touch changed files
# (typically Step 2 + Step 3 if memory_global/ or memory_project/ changed)
```

The dev machine's update workflow is documented in `README.md`.

---

## Step 2 — Install global memory + rules

On the **test machine**:

```bash
# Create the Claude config directory if it doesn't exist
mkdir -p ~/.claude/memory
mkdir -p ~/.claude/rules

# Copy global memory (works across all projects on this machine)
cp ~/spp-claude-transfer/memory_global/*.md ~/.claude/memory/

# Copy rules
cp ~/spp-claude-transfer/rules/*.md ~/.claude/rules/

# Update / create the global CLAUDE.md so memory auto-loads each session
if [ ! -f ~/.claude/CLAUDE.md ]; then
    cp ~/spp-claude-transfer/CLAUDE_global_index.md ~/.claude/CLAUDE.md
else
    echo "WARNING: ~/.claude/CLAUDE.md already exists. Review it manually:"
    echo "  diff ~/.claude/CLAUDE.md ~/spp-claude-transfer/CLAUDE_global_index.md"
    echo "Merge the 'Global memory' section into the existing file."
fi

# Verify
ls ~/.claude/memory/
ls ~/.claude/rules/
cat ~/.claude/CLAUDE.md | head -20
```

Expected: 6 memory files, 9 rule files, CLAUDE.md mentions `~/.claude/memory/`.

---

## Step 3 — Install project-scoped memory

The project memory is keyed by the bench path. On the **test machine**, find
your Frappe bench path first:

```bash
# Likely one of these — pick the one that exists:
ls ~/frappe-bench/apps/shree_polymer_console 2>/dev/null && echo "FOUND: ~/frappe-bench"
ls /opt/frappe-bench/apps/shree_polymer_console 2>/dev/null && echo "FOUND: /opt/frappe-bench"
```

Whichever path holds the Console app, that's `<BENCH>` for the next step.
Compute the Claude project key:

```bash
# Replace <BENCH> with your actual path, then:
BENCH=~/frappe-bench  # ← edit this

# Build the project key (Claude derives it from the path with slashes → dashes)
PROJECT_KEY="-$(echo $BENCH/apps/shree_polymer_console | sed 's|/|-|g' | sed 's|^-||')"
echo "Project key: $PROJECT_KEY"

# Create the project memory dir
mkdir -p "$HOME/.claude/projects/$PROJECT_KEY/memory"

# Copy the project-scoped memory files
cp ~/spp-claude-transfer/memory_project/*.md "$HOME/.claude/projects/$PROJECT_KEY/memory/"

# Verify
ls "$HOME/.claude/projects/$PROJECT_KEY/memory/"
```

Expected: 4 files (MEMORY.md, feedback_pr_workflow.md, feedback_test_gate.md, reference_graphify.md).

> **Note:** if the path on this machine is `/opt/frappe-bench/apps/shree_polymer_console`,
> the project key becomes `-opt-frappe-bench-apps-shree-polymer-console`. Adjust accordingly.

---

## Step 4 — Patch path references in memory files

The memory files contain references to `/Users/alphaworkz/frappe-bench/...`
which won't exist on this machine. Quick global sed-replace:

```bash
# Discover this machine's bench path (set in Step 3)
BENCH=~/frappe-bench  # ← edit if different

# Replace all dev-machine paths in the COPIED memory (don't touch the bundle)
find ~/.claude/memory -name "*.md" -exec sed -i.bak \
    "s|/Users/alphaworkz/frappe-bench|$BENCH|g" {} \;

PROJECT_KEY="-$(echo $BENCH/apps/shree_polymer_console | sed 's|/|-|g' | sed 's|^-||')"
find "$HOME/.claude/projects/$PROJECT_KEY/memory" -name "*.md" -exec sed -i.bak \
    "s|/Users/alphaworkz/frappe-bench|$BENCH|g" {} \;

# Verify (should show no /Users/alphaworkz/ refs anymore)
grep -r "/Users/alphaworkz" ~/.claude/memory/ ~/.claude/projects/ 2>/dev/null

# Cleanup .bak files once you confirm
find ~/.claude/memory -name "*.md.bak" -delete
find "$HOME/.claude/projects/$PROJECT_KEY/memory" -name "*.md.bak" -delete
```

If the grep returns no output, paths are clean.

---

## Step 5 — Copy the project docs to the test machine's bench

The `spp_process_docs/` folder is referenced by memory and contains the
process documentation, porting plan, factory testing docs, and integration
test plans. Put it next to the bench (NOT inside it — the docs aren't a
Frappe app):

```bash
# Same BENCH variable from Step 3
cp -r ~/spp-claude-transfer/docs/spp_process_docs $(dirname $BENCH)/

# Verify
ls $(dirname $BENCH)/spp_process_docs/
```

Expected: ~24 files including `console_porting/`, `factory_testing/`,
`detailed_process_flow.md`, etc.

---

## Step 6 — Confirm git remotes on the apps

The test machine needs to be able to fetch feature branches from the dev
machine's GitHub forks. Check each app's remotes:

```bash
cd $BENCH/apps/shree_polymer_console
git remote -v
# Expected:
#   iyyanarr  https://github.com/iyyanarr/shree_polymer_console.git (fetch+push)
#   rsvasanth https://github.com/rsvasanth/shree_polymer_console.git (fetch+push, optional)

cd $BENCH/apps/shree_polymer_legacy_bridge
git remote -v
# Expected:
#   origin    https://github.com/rsvasanth/shree_polymer_legacy_bridge.git
#   iyyanarr  https://github.com/iyyanarr/shree_polymer_legacy_bridge.git
```

If any expected remote is missing, add it:

```bash
git remote add iyyanarr https://github.com/iyyanarr/shree_polymer_console.git
```

---

## Step 7 — Start Claude Code on the test machine

From the bench directory:

```bash
cd $BENCH/apps/shree_polymer_console
claude
```

In the first chat message, paste:

```
Read /home/<user>/spp-claude-transfer/KICKOFF.md and follow its instructions.
```

(replace `<user>` with the actual user on this machine)

The Claude there will read the kickoff doc, then the memory files, then
acknowledge readiness.

---

## Verification checklist

Tick each before declaring setup done:

- [ ] Bundle transferred and extracted on test machine
- [ ] `~/.claude/memory/` has 6 files (global)
- [ ] `~/.claude/rules/` has 9 files
- [ ] `~/.claude/CLAUDE.md` mentions `~/.claude/memory/`
- [ ] Project memory dir exists with 4 files
- [ ] Path references in memory are sed-replaced to test-machine paths
- [ ] `spp_process_docs/` is sibling of `frappe-bench/` on the test machine
- [ ] Git remotes on Console + Legacy Bridge apps include both forks
- [ ] Claude Code starts in the Console app dir
- [ ] First Claude chat reads KICKOFF.md and acknowledges readiness

When all 10 are ticked, the test machine is ready for the first test cycle.

---

## Troubleshooting

**"Memory files reference /Users/alphaworkz/ paths that don't exist"**
You skipped Step 4. Run the sed-replace.

**"Claude on test machine doesn't know about the SPP project"**
Check `~/.claude/CLAUDE.md` exists and points to `~/.claude/memory/`.
Restart the Claude session.

**"bench run-tests can't find the test module"**
Confirm Console app version on test machine matches the dev machine.
Run `cd <BENCH>/apps/shree_polymer_console && git log --oneline -5`
to see latest commits — should match dev machine's `iyyanarr/develop`
HEAD plus pending feature commits.

**"Production data on spp15 is missing recent records"**
The "production replica" was made at some date. If the dev machine has
done work since that snapshot, the data may not include very recent
batches. Clarify with the user when the snapshot was taken.
