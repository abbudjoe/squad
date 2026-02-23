# Running Parallel Review/Fix Loops

This guide covers running multiple PRs through Squad loops simultaneously.

## When to Parallelize

Parallelize when you have multiple independent workstreams:
- Different files/modules with no overlap
- Different areas of the codebase
- Bug fixes that don't touch shared code

**Do NOT parallelize when:**
- Two PRs modify the same files
- PR B depends on PR A's changes
- Changes touch shared interfaces that affect multiple consumers

## Setup

### 1. Identify Independent Workstreams

```
Workstream 1: PR #101 — feat/auth (touches: src/auth/*, tests/auth/*)
Workstream 2: PR #102 — feat/dashboard (touches: src/ui/dashboard/*, tests/ui/*)
Workstream 3: PR #103 — fix/memory-leak (touches: src/core/cache.ts)
```

Verify independence: no file overlap between workstreams.

### 2. Create a State Tracking Table

In your daily notes or a tracking file:

```markdown
| PR | Branch | Stage | Reviewer | Fixer | Watchdog |
|----|--------|-------|----------|-------|----------|
| #101 | feat/auth | REVIEWING | pr101-reviewer-r1 | — | pr101-loop |
| #102 | feat/dashboard | FIXING | — | pr102-fixer-r1 | pr102-loop |
| #103 | fix/memory-leak | RE_REVIEWING | pr103-reviewer-r2 | — | pr103-loop |
```

### 3. Create Watchdog Crons

One cron per PR, using the [watchdog template](../prompts/watchdog.md):

```
Cron: pr101-review-fix-loop (every 3min)
Cron: pr102-review-fix-loop (every 3min)
Cron: pr103-review-fix-loop (every 3min)
```

### 4. Spawn Initial Reviewers

Launch all reviewers in parallel — they're independent:

```
spawn_subagent(label="pr101-reviewer-r1", model="opus", ...)
spawn_subagent(label="pr102-reviewer-r1", model="opus", ...)
spawn_subagent(label="pr103-reviewer-r1", model="opus", ...)
```

## Orchestration Rules

### Completion Events

When a subagent completes:
1. **Identify which PR** it belongs to (from the label)
2. **Transition that PR's state** to the next stage
3. **Spawn the next subagent** for that PR
4. **Do NOT block other PRs** — if one is waiting, work the others

### The "Never Idle" Rule

If one loop is blocked (e.g., waiting for a slow fixer), immediately check if other loops need action. The orchestrator should always be advancing *something*.

### Context Management

With 3+ parallel loops, context fills fast. Strategies:

1. **Write state to files** — Don't rely on in-context memory for PR stages
2. **Use consistent labels** — `pr{N}-{role}-r{round}` makes it easy to identify subagents
3. **Let watchdogs recover state** — After compaction, the watchdog cron fires and re-triggers the correct next action
4. **Summarize, don't repeat** — When spawning subagents, include the diff but summarize context

### Rate Limit Awareness

Multiple fixers hitting the same API simultaneously can trigger rate limits:
- **Stagger spawns** by 30-60 seconds if possible
- **Use different model providers** for different loops if available
- **Have the watchdog retry** — if a subagent fails due to rate limits, the next watchdog tick will re-spawn it

## Merge Sequencing

When multiple PRs are ready for merge:

### Simple Case (No Conflicts)
Merge in any order. Each PR is independent.

### Adjacent Files
If PRs touch nearby code (same directory, related modules):
1. Merge the simplest PR first
2. Rebase the next PR: `git rebase origin/main`
3. If conflicts arise, the fixer handles them in the next cycle
4. Re-run the review loop after rebase

### Dependency Chain
If PR B depends on PR A:
1. Complete PR A's loop first
2. Merge PR A
3. Rebase PR B onto the updated base
4. Start/restart PR B's review loop

## Capacity Planning

### Sweet Spot: 3-4 Parallel Loops

This balances:
- **Throughput** — 3-4x faster than sequential
- **Context pressure** — manageable with state files
- **Rate limits** — unlikely to hit with standard API plans
- **Merge conflicts** — rare with good workstream selection

### Wave Batching for Larger Sprints

For 10+ PRs:

```
Wave 1: PR #101, #102, #103 (parallel loops)
  ↓ merge all
Wave 2: PR #104, #105, #106 (parallel loops)
  ↓ merge all
Wave 3: PR #107, #108, #109 (parallel loops)
  ↓ merge all
```

Each wave runs 3-4 loops in parallel. Merge everything before starting the next wave.

## Example Timeline

```
T+0m    Spawn 3 reviewers (PR #101, #102, #103)
T+2m    #101 review done → spawn fixer
T+3m    #102 review done → spawn fixer
T+3m    #103 review done → clean! → tag ready for merge
T+5m    #101 fixer done → spawn re-reviewer
T+6m    #102 fixer done → spawn re-reviewer
T+7m    #101 re-review → clean! → tag ready for merge
T+8m    #102 re-review → 1 issue → spawn fixer
T+10m   #102 fixer done → spawn re-reviewer
T+12m   #102 re-review → clean! → tag ready for merge

Total: 12 minutes for 3 PRs, ~7 review cycles across all loops
Sequential estimate: ~33 minutes
Speedup: ~2.75x
```

## Failure Modes

| Failure | Symptom | Recovery |
|---------|---------|----------|
| Subagent timeout | No completion event | Watchdog re-spawns next tick |
| Rate limit | Subagent returns error | Watchdog retries next tick |
| Context compaction | Orchestrator forgets state | Watchdog + state files recover |
| Merge conflict | Rebase fails | Fixer resolves in next cycle |
| Flaky test | Fixer can't get green | Orchestrator escalates to human |
