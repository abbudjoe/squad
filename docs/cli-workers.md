# CLI Worker Alternative

Instead of using platform subagents for fixers, you can use CLI-based AI agents (e.g., Codex CLI, Claude CLI). The orchestrator subagent handles all git and GitHub operations that the CLI worker can't.

## Architecture

```
Orchestrator (persistent subagent, has full tool access)
├── creates/manages worktree on build node
├── launches CLI worker in that worktree
├── monitors for completion (cron or polling)
├── git add + commit + push (after worker finishes)
├── posts chain-of-custody PR comment (via gh)
└── spawns reviewer for the next round
```

The CLI worker **only edits code**. It never needs to:
- `git push`
- Post PR comments
- Access external APIs
- Manage state

This makes it compatible with sandboxed environments (Codex) and minimizes the trust surface.

## Worker Types

### Codex CLI (`codex exec`)

```bash
# Orchestrator launches on build node:
cd {{worktree_path}}
codex exec --full-auto "{{fixer_prompt}}"
```

**Pros:**
- Sandboxed — can't push, post, or leak
- Gets its own filesystem view (no collision risk)
- Good at mechanical fixes

**Cons:**
- Can't `git push` or `gh pr comment` — orchestrator must handle
- Can't run `gh api` or network calls inside sandbox
- Fire-and-forget — needs completion monitoring

### Claude CLI (`claude -p`)

```bash
# Orchestrator launches on build node:
cd {{worktree_path}}
echo "{{fixer_prompt}}" | claude -p
```

**Pros:**
- Full filesystem access in the worktree
- Can run `gh` commands if authenticated
- Good at complex reasoning tasks

**Cons:**
- Not sandboxed — can make unintended changes outside worktree
- Fire-and-forget — needs completion monitoring
- Shares filesystem (needs worktree isolation)

## Orchestrator Git Workflow

After a CLI worker finishes editing files, the orchestrator handles git:

```bash
# 1. Check what the worker changed
cd {{worktree_path}}
git status --short
git diff

# 2. Stage and commit
git add -A
git commit -m "fix({{scope}}): address PR #{{pr_number}} review items

- Fixed B1: {{description}}
- Fixed NB1: {{description}}
- Fixed NH1: {{description}}"

# 3. Push
git push origin {{branch_name}}

# 4. Post chain-of-custody comment
COMMIT_SHA=$(git rev-parse HEAD)
gh pr comment {{pr_number}} --repo {{repo}} --body "## Fixer Report
### Commit
\`${COMMIT_SHA}\`
### Changes
- B1: {{what_was_fixed}}
- NB1: {{what_was_fixed}}
### Verification
- Build: ✅
- Tests: ✅"
```

## Completion Monitoring

CLI workers are fire-and-forget. The orchestrator must monitor them.

### Option A: Polling (simple)

```bash
# Check if the worker process is still running
ps aux | grep -E 'codex|claude' | grep -v grep

# Check output
tail -20 /tmp/worker-pr{{pr_number}}.log

# Check for file changes
cd {{worktree_path}} && git status --short
```

### Option B: Watchdog Cron (resilient)

Create a cron that checks worker status every 2-3 minutes:

```
"Check CLI worker for PR #{{pr_number}}.
1. Run: ps aux | grep 'codex\|claude' | grep {{worktree_path}}
2. If still running → do nothing.
3. If finished → check git status in {{worktree_path}}.
   - If changes exist → commit, push, post comment, spawn reviewer.
   - If no changes → worker may have failed. Check logs."
```

### Option C: File-based signal (most reliable)

Have the CLI worker create a sentinel file when done:

```bash
# In the fixer prompt:
"After completing all fixes, create a file called DONE.txt with a summary."

# Orchestrator checks:
if [ -f {{worktree_path}}/DONE.txt ]; then
    # Worker finished — proceed with git operations
    rm {{worktree_path}}/DONE.txt
fi
```

## Hybrid Pattern: Subagent Reviewers + CLI Fixers

The sweet spot for many teams:

- **Reviewers** → Platform subagents (they just read diffs and post comments — no file edits)
- **Fixers** → CLI workers (they need to edit code, run builds/tests — benefit from local toolchain)
- **Orchestrator** → Platform subagent (persistent session, handles git, monitors workers)

This gives you:
- Push-based reviewer completion (no monitoring needed)
- Local build environment for fixers (JDK, SDK, etc.)
- Reliable git operations from the orchestrator

## Failure Modes

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Worker crashes mid-edit | No DONE.txt, partial changes in worktree | Orchestrator reverts: `git checkout .` and re-launches |
| Worker makes no changes | Empty `git status` | Orchestrator logs failure, may re-launch with more context |
| Worker edits wrong files | `git diff` shows unexpected paths | Orchestrator reverts and re-launches with clearer scope |
| Push fails (conflict) | Non-zero exit from `git push` | Orchestrator rebases: `git pull --rebase origin/{{base}}` |
| Worker times out | Process still running after N minutes | Orchestrator kills and re-launches |
