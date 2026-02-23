# Orchestrator Decision Logic

This document describes how the orchestrator (main session) manages one or more review/fix loops.

## Per-PR State Machine

Track each active PR in exactly one stage:

```
IMPLEMENTED → REVIEWING → FIXING → RE_REVIEWING → CLEAN → READY_FOR_MERGE
```

### Stage Transitions

| From | To | Trigger | Required Artifact |
|------|----|---------|-------------------|
| IMPLEMENTED | REVIEWING | Reviewer subagent spawned | — |
| REVIEWING | FIXING | Review posted with issues | Review comment URL |
| REVIEWING | CLEAN | Review posted, all sections empty | Review comment URL |
| FIXING | RE_REVIEWING | Fixer committed and pushed | Commit SHA + chain-of-custody comment URL |
| RE_REVIEWING | FIXING | Re-review found issues | Review comment URL |
| RE_REVIEWING | CLEAN | Re-review clean | Review comment URL |
| CLEAN | READY_FOR_MERGE | `@maintainer ready for merge` posted | Comment URL |

### Artifact Gates

**Never mark a stage complete without verifying artifacts:**

- **Review stage complete** → Review comment URL exists on the PR
- **Fix stage complete** → Commit SHA exists, branch is pushed, chain-of-custody PR comment URL exists
- **Clean** → All three sections (Blocking, Non-blocking, Nice-to-haves) are empty in the latest review

### Verdict Normalization

If the reviewer says `APPROVE` but any issue section has items, treat it as **not clean**. Immediately spawn a fixer.

## Completion Event Handling

When a subagent completes:

1. **First action:** Spawn the next stage (or record explicit blocker)
2. **Second action:** Post a status update

Never post a status update without first ensuring the next stage is launched.

## Watchdog Event Handling

When a watchdog cron fires:

1. **First action:** Check orchestration state + spawn next stage (or record blocker)
2. **Second action:** Post status/reminder if needed

## Parallel Loop Management

### Starting Multiple Loops

```
Loop 1: PR #101 (feat/auth)        → REVIEWING
Loop 2: PR #102 (feat/dashboard)   → FIXING  
Loop 3: PR #103 (fix/memory-leak)  → RE_REVIEWING
```

### Rules for Parallel Loops

1. **Each PR gets its own watchdog cron** — named `pr{N}-review-fix-loop`
2. **Each PR gets its own branch** — no shared branches
3. **Subagent labels include PR number** — `pr101-reviewer-r1`, `pr102-fixer-r3`
4. **Don't poll subagent lists in loops** — completion is push-based
5. **If one loop is blocked, work another** — never idle waiting

### Merge Ordering

When multiple PRs are ready:
1. Merge the simplest/most independent first
2. If PR B touches files near PR A, merge A first
3. After merging, rebase remaining PRs: `git fetch origin && git rebase origin/<base>`
4. If rebase conflicts arise, the fixer handles them in the next fix cycle

### Context Pressure Management

With 3+ active loops, context fills fast. Strategies:
- Keep PR state in a daily memory file, not just in-context
- Summarize diffs when passing to reviewers (include full diff, but summarize context)
- Use the watchdog as a state-recovery mechanism after compaction
- Batch waves: run 3-4 loops, merge, then start the next batch

## Status Update Format

When posting progress updates, use this cadence:

```
1. "Code committed, PR opened, reviewer spawned"        → REVIEWING
2. "Reviewer finished, fixer spawned"                     → FIXING
3. "Fixer finished, re-review spawned"                    → RE_REVIEWING
4. Repeat 2↔3 until clean
5. "Review clean, tagged @maintainer ready for merge"     → READY_FOR_MERGE
```
