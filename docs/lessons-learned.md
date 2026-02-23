# Lessons Learned

Hard-won lessons from months of running AI agent review/fix loops in production.

## Subagent Behavior

### "Optional" means "skip" to an AI
**Problem:** When a review says "Nice-to-have: consider adding X," subagents interpret this as truly optional and skip it.

**Fix:** Every fixer prompt must include: *"Nice-to-haves are NOT optional. Address ALL of them. No deferrals."*

### Branch contamination
**Problem:** Subagents inherit the orchestrator's working directory state. Untracked files, staged changes, and even other branches' modifications can leak into the subagent's commits.

**Fix:** Every fixer task must start with:
```bash
git fetch origin
git checkout {{branch}}
git status  # verify clean state
```

### Subagents don't push (sandbox environments)
**Problem:** Some agent platforms (Codex sandbox, etc.) block outbound network. The subagent makes changes but can't `git push`.

**Fix:** Always verify the push happened. If using sandboxed environments, the orchestrator must handle `git add`, `git commit`, and `git push` after the subagent completes.

### Fire-and-forget CLI workers need monitoring
**Problem:** Background CLI processes (`nohup codex exec ...`) don't send completion events. Their work sits uncommitted indefinitely.

**Fix:** Create a cron monitor immediately when spawning any background worker. Check `ps aux`, tail log files, and complete git operations when done. Delete the cron when idle.

## Review Quality

### Rubber-stamp reviews are the default
**Problem:** Without explicit anti-rubber-stamp instructions, reviewers tend to approve everything with superficial comments.

**Fix:** Always include: *"Do NOT rubber-stamp. Missing tests = BLOCKING. If something looks wrong, say so."*

### APPROVE verdict doesn't mean clean
**Problem:** A reviewer can say "APPROVE" while listing 3 non-blocking issues. The orchestrator interprets APPROVE as ready-to-merge.

**Fix:** Ignore the verdict label. A review is clean ONLY when Blocking + Non-blocking + Nice-to-haves sections are ALL empty.

### Review quality degrades with compressed context
**Problem:** Summarizing diffs to save tokens causes reviewers to miss issues they'd catch with full context.

**Fix:** Always include the full diff in reviewer prompts. Save tokens elsewhere (shorter context notes, not shorter diffs).

## Orchestration

### Completion events must trigger action, not status updates
**Problem:** When a subagent completes, the orchestrator posts a status message but forgets to spawn the next stage.

**Fix:** First action on completion = spawn next stage (or record blocker). Status update comes second.

### Config changes kill active subagents
**Problem:** Restarting the gateway (e.g., `config.patch`) wipes the session store. Running subagents lose their work.

**Fix:** Always commit and push before any operation that might restart the orchestrator.

### Process death chains are real
**Problem:** A seemingly minor state change (overlay deactivation → foreground service stops → process killed → ViewModel lost → chat empty). Each step seems harmless; the chain is catastrophic.

**Fix:** Trace the full lifecycle of state changes. Ask: "If this value changes, what depends on it? What depends on *that*?"

## Git Workflow

### Merge ordering matters
**Problem:** Merging PR A causes PR B to have conflicts, even though they seemed independent.

**Fix:** Before starting parallel loops, verify file independence. After merging, rebase remaining PRs before their next review cycle.

### Commit before everything
**Problem:** Work gets lost when sessions restart, subagents crash, or config changes trigger restarts.

**Fix:** Commit early, commit often. Push before any disruptive operation. Uncommitted work is ephemeral.

## Capacity

### 3-4 parallel loops is the sweet spot
**Problem:** Running 7+ loops simultaneously causes context overflow, rate limits, and merge chaos.

**Fix:** Batch in waves of 3-4. Complete and merge each wave before starting the next. The slight serialization is worth the stability.

### Watchdogs are insurance, not primary drivers
**Problem:** Relying on watchdog crons as the main orchestration mechanism leads to 3-minute delays between every stage.

**Fix:** Use completion events as the primary trigger. Watchdogs are backup for when the orchestrator loses context (compaction, restarts, crashes).
