# Conductor Prompt Template

The **Conductor** is the top-level agent (your main session). It does NOT manage review/fix loops directly. Instead, it spawns **Orchestrator subagents** that each own one PR end-to-end.

## Responsibilities

1. **Decompose work** into independent, non-overlapping tasks
2. **Spawn Orchestrator subagents** (one per task/PR) as persistent sessions
3. **Monitor health** — check if orchestrators are stuck or need human input
4. **Handle merge sequencing** — when orchestrators tag ready, merge in the right order
5. **Escalate blockers** to the human
6. **Stay lean** — don't hold PR diffs or review details in context

## Spawning an Orchestrator

Each orchestrator is a **persistent session** (`mode=session`), not a one-shot run. This lets it:
- React to its own watchdog cron events
- Manage multiple review/fix rounds over time
- Maintain its own state without polluting the conductor's context

```
spawn_subagent(
    mode = "session",        # persistent — survives across turns
    model = "{{coder_model}}",
    label = "squad-pr{{N}}",
    task = fill_template("prompts/orchestrator-subagent.md", {
        task_description: "...",
        repo: "owner/repo",
        repo_path: "/path/to/repo",
        base_branch: "main",
        branch_name: "feat/...",
        reviewer_model: "opus",
        fixer_model: "codex",
        build_command: "...",
        test_command: "...",
        definition_of_done: contents_of("config/definition-of-done.md"),
        maintainer: "@username"
    })
)
```

## State Tracking

The conductor tracks minimal state per orchestrator:

```markdown
| # | PR | Orchestrator | Status | Last Update |
|---|-----|-------------|--------|-------------|
| 1 | #101 | squad-pr101 | running | R2 review in progress |
| 2 | #102 | squad-pr102 | ready | tagged @maintainer |
| 3 | #103 | squad-pr103 | blocked | flaky test, escalated |
```

**Don't track internal loop state** (which review round, what issues were found). That's the orchestrator's job.

## Merge Sequencing

When orchestrators report ready-for-merge:

1. **Check for conflicts** — if two PRs touch nearby files, merge the simpler one first
2. **Merge** the PR
3. **Notify remaining orchestrators** to rebase if needed:
   ```
   sessions_send(sessionKey="squad-pr102", message="PR #101 was merged. Rebase your branch: git fetch origin && git rebase origin/main")
   ```
4. **Close the orchestrator's session** after merge is confirmed

## Health Monitoring

Check on orchestrators periodically (or via a conductor-level cron):

```
subagents(action="list")
```

Signs of trouble:
- Orchestrator has been in the same stage for >30 minutes
- Orchestrator reports a blocker (flaky test, rate limit, compile error it can't fix)
- Watchdog cron fires but orchestrator doesn't respond

Recovery:
- Send a steering message: `sessions_send(sessionKey="squad-pr101", message="Status check — what stage are you in?")`
- If truly stuck, kill and re-spawn with updated context

## Capacity

| Orchestrators | Conductor Load | Notes |
|:---:|:---:|---|
| 1-3 | Minimal | Easy to track, merge conflicts unlikely |
| 4-6 | Low | Sweet spot for a sprint |
| 7-10 | Medium | Use wave batching (merge first batch before spawning next) |
| 10+ | High | Definitely wave batch in groups of 4-6 |

## Anti-Patterns

- ❌ **Reading PR diffs in the conductor** — that's the orchestrator's job
- ❌ **Spawning fixers/reviewers directly** — delegate to orchestrators
- ❌ **Polling subagent lists in a loop** — completion is push-based
- ❌ **Holding review details in conductor context** — wastes tokens, causes compaction
- ❌ **Merging without checking for conflicts** — always verify independence first
