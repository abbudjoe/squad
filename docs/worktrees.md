# Worktree Isolation

When running parallel orchestrators on a shared filesystem, each orchestrator **must** have its own worktree to prevent file collisions.

## The Problem

Two orchestrators editing files in the same git checkout simultaneously will corrupt each other's work — even on different branches. Git checkout changes the working directory globally.

## Solution: git worktree

Each orchestrator creates an isolated worktree for its branch:

```bash
# Create a worktree for this orchestrator's branch
git worktree add /path/to/worktrees/feat-auth feat/auth

# Work happens in /path/to/worktrees/feat-auth (isolated from main checkout)
cd /path/to/worktrees/feat-auth
# ... edit files, build, test ...

# Clean up when done
git worktree remove /path/to/worktrees/feat-auth
```

All worktrees share the same `.git` objects — `git fetch` in one is visible to all. Branches checked out in a worktree are locked (can't be checked out elsewhere).

## Naming Convention

```
~/repo/                          # main checkout (conductor's home)
~/repo-worktrees/
├── feat-auth/                   # orchestrator A's worktree
├── feat-dashboard/              # orchestrator B's worktree
└── fix-memory-leak/             # orchestrator C's worktree
```

Or use a flat pattern:
```
~/repo-squad-101/                # orchestrator for PR #101
~/repo-squad-102/                # orchestrator for PR #102
```

## Orchestrator Lifecycle

### Setup (Phase 1 of orchestrator)

```bash
# Create worktree from base branch
git -C {{repo_path}} fetch origin
git -C {{repo_path}} worktree add {{worktree_path}} -b {{branch_name}} origin/{{base_branch}}
```

### Work (Phases 2-3)

All file edits, builds, and tests happen in `{{worktree_path}}`. The orchestrator passes this path to fixer subagents.

### Teardown (after merge)

```bash
git -C {{repo_path}} worktree remove {{worktree_path}}
git -C {{repo_path}} branch -d {{branch_name}}  # if merged
```

## Parallel Safety Guarantees

| Scenario | Safe? | Why |
|----------|-------|-----|
| Two orchestrators editing different worktrees | ✅ | Completely isolated filesystems |
| Orchestrator A builds while B edits | ✅ | Different directories |
| `git fetch` in any worktree | ✅ | Shared objects, no conflict |
| Two worktrees on the same branch | ❌ | Git locks branches to one worktree |
| Build artifacts in worktree A | ✅ | Build output stays in that worktree |

## When Worktrees Aren't Needed

- **Single orchestrator at a time** — no collision risk
- **Subagents on different machines** — naturally isolated
- **CLI workers in sandboxed environments** (e.g., Codex) — sandbox provides isolation
- **Review-only subagents** — they read diffs, don't edit files
