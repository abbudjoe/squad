# Squad 🪖

**Autonomous AI code review/fix loops with parallel orchestration.**

Squad is a methodology and prompt toolkit for running multiple AI agents in parallel review/fix cycles on a shared codebase — with a single orchestrator agent managing the entire flow.

## The Problem

AI code agents can write code, but shipping quality code requires review cycles. Manually orchestrating "reviewer finds issues → fixer resolves them → re-review → repeat" is tedious. Doing it for multiple PRs simultaneously is chaos.

## The Solution

Squad defines:
1. **Roles** — Reviewer, Fixer, Orchestrator — with strict prompt templates
2. **A state machine** — each PR moves through defined stages with artifact gates
3. **A watchdog pattern** — cron-based monitors that keep loops progressing autonomously
4. **Parallel execution rules** — how to safely run N loops without conflicts

## How It Works

```
┌─────────────┐     ┌──────────┐     ┌─────────────┐
│ Orchestrator │────▶│ Reviewer │────▶│   Fixer     │
│   (main)     │◀────│  (opus)  │◀────│  (codex)   │
└──────┬───────┘     └──────────┘     └─────────────┘
       │
       ▼
  ┌─────────┐
  │ Watchdog │  ← cron every 3min
  │  (cron)  │
  └─────────┘
```

1. **Orchestrator** creates a branch, implements code, opens a PR
2. **Orchestrator** spawns a **Reviewer** subagent on the PR diff
3. Reviewer posts structured review as a PR comment
4. If issues found → Orchestrator spawns a **Fixer** subagent
5. Fixer resolves ALL issues, commits, pushes, posts chain-of-custody comment
6. Orchestrator spawns another Reviewer (re-review)
7. Loop continues until review is clean (all sections empty)
8. Orchestrator tags the PR ready for merge
9. **Watchdog cron** ensures the loop keeps progressing even if the orchestrator loses context

## Quick Start

### 1. Copy the prompt templates

The [`prompts/`](./prompts/) directory contains ready-to-use templates:

- [`reviewer.md`](./prompts/reviewer.md) — Reviewer subagent prompt
- [`fixer.md`](./prompts/fixer.md) — Fixer subagent prompt
- [`watchdog.md`](./prompts/watchdog.md) — Cron watchdog event text
- [`orchestrator.md`](./prompts/orchestrator.md) — Orchestrator decision logic

### 2. Define your "Definition of Done"

Edit [`config/definition-of-done.md`](./config/definition-of-done.md) with your project's standards.

### 3. Start a loop

See [`docs/single-loop.md`](./docs/single-loop.md) for running one PR through the cycle, or [`docs/parallel-loops.md`](./docs/parallel-loops.md) for running multiple PRs simultaneously.

## Architecture

### Roles

| Role | Model | Purpose |
|------|-------|---------|
| **Orchestrator** | Any (main session) | Spawns agents, tracks state, escalates blockers |
| **Reviewer** | High-reasoning (e.g. Opus, o3) | Finds real issues, enforces standards |
| **Fixer** | Fast coder (e.g. Codex, Sonnet) | Resolves all issues, commits, pushes |
| **Watchdog** | N/A (cron trigger) | Fires every N minutes to keep loops alive |

### PR State Machine

```
IMPLEMENTING → REVIEWING → FIXING → RE_REVIEWING → ... → CLEAN → READY_FOR_MERGE
```

Each transition requires **artifact gates**:
- `REVIEWING → FIXING`: Review comment URL must exist
- `FIXING → RE_REVIEWING`: Commit SHA + push + chain-of-custody comment URL must exist
- `RE_REVIEWING → CLEAN`: All review sections (Blocking, Non-blocking, Nice-to-haves) must be empty
- `CLEAN → READY_FOR_MERGE`: `@maintainer ready for merge` comment posted

### Clean Review Definition

A review is **clean** only when:
- Blocking Issues section is empty
- Non-blocking Issues section is empty  
- Nice-to-haves section is empty

An APPROVE verdict alone is **not sufficient** — the reviewer must have zero items in all sections.

## Key Principles

### No Deferrals
Every review item must be addressed in the current cycle. Nothing gets pushed to "future work" unless the human explicitly approves that specific deferral.

### No Rubber-Stamping
Reviewers are instructed to find real problems. Missing tests = BLOCKING. A review that misses obvious issues is a failed review.

### Artifact-Gated Transitions
State changes only happen when required artifacts exist. No "I think the fixer probably pushed" — verify the commit SHA, verify the PR comment URL.

### Watchdog Resilience
The cron watchdog is a safety net. If the orchestrator loses context (session restart, compaction), the watchdog fires and re-triggers the next action. It's not the primary driver — it's insurance.

## Capacity Planning

| Parallel Loops | Context Pressure | Rate Limit Risk | Recommended |
|:-:|:-:|:-:|:-:|
| 1-2 | Low | None | ✅ Easy mode |
| 3-4 | Medium | Low | ✅ Sweet spot |
| 5-6 | High | Medium | ⚠️ Needs wave batching |
| 7+ | Critical | High | ❌ Batch in waves of 3-4 |

### Merge Ordering
When multiple PRs target the same base branch, merge in dependency order. If PR B touches files near PR A, merge A first, then rebase B before its next review cycle.

## Lessons Learned

This methodology evolved from months of real-world AI agent orchestration. Key lessons:

1. **"Optional" items get skipped** — Subagents interpret "nice to have" as truly optional. You must explicitly state: "Nice-to-haves are NOT optional. Address ALL of them."
2. **Branch contamination is real** — Always start from a clean checkout. Include `git fetch && git checkout -b <branch> origin/<base>` in every task.
3. **Foreground services keep processes alive** — In mobile dev, losing a foreground service can cause process death. Same principle applies to agent loops: keep the watchdog alive.
4. **Review quality scales with context** — Don't over-compress diffs to save tokens. The reviewer needs enough context to catch real issues.
5. **Completion is push-based** — Don't poll for subagent status. Let completion events trigger the next action.
6. **Commit before config changes** — Any operation that might restart your orchestrator should be preceded by committing all in-flight work.

## License

MIT — use it however you want.

## Contributing

This is a living methodology. If you find improvements, open a PR. The only rule: changes must come from real-world experience, not theory.
