# Watchdog Cron Event Template

Use this as the `systemEvent` text for a cron job that monitors a review/fix loop.
Fill in the `{{variables}}` before creating the cron.

If your environment already has a reusable markdown-safe watchdog template cron (for example `template-pr-review-fix-loop-markdown-v1`), reuse it instead of hand-writing new watchdog text.

---

## Template

```
PR #{{pr_number}} review/fix loop watchdog.

Check orchestration state for PR #{{pr_number}} ({{head_branch}} → {{base_branch}}).

Steps:
1) If no reviewer or fixer subagent is running, check the latest PR comment.
   - If it's a review with issues → spawn a fixer subagent ({{fixer_model}}).
   - If no review exists yet → spawn a reviewer subagent ({{reviewer_model}}).
2) If a fixer subagent completed and posted a chain-of-custody commit comment → spawn a reviewer subagent.
3) If a reviewer subagent completed and the review shows zero issues (Blocking + Non-blocking + Nice-to-haves + Deferred/Follow-ups ALL empty) → comment '@{{maintainer}} ready for merge' on the PR and disable this cron job.
4) If the review has ANY issues at all (including deferred/follow-up/optional items) → spawn a fixer subagent to resolve ALL of them. No deferrals.

Continue looping until the PR is clean.

Fixer requirements:
- Use proper markdown formatting in chain-of-custody PR comments (headers, bullets, code blocks).
- Commit messages must be clean conventional-commit format (plain text, no markdown).
```

## Cron Configuration

```json
{
  "name": "pr{{pr_number}}-review-fix-loop",
  "schedule": {
    "kind": "every",
    "everyMs": 300000
  },
  "payload": {
    "kind": "systemEvent",
    "text": "{{filled_template_above}}"
  },
  "sessionTarget": "main",
  "enabled": true
}
```

## Lifecycle

1. **Create** the cron when you spawn the first reviewer for a PR
2. **Disable** the cron when the PR is tagged ready for merge (or abandoned)
3. **Delete** stale crons periodically — don't leave them burning tokens

## Tuning

| Parameter | Default | Notes |
|-----------|---------|-------|
| `everyMs` | 300000 (5 min) | Increase for slower fixers, decrease for hot loops |
| Reviewer model | High-reasoning (Opus, o3) | Needs to find real issues |
| Fixer model | Fast coder (Codex, Sonnet) | Needs to execute fixes quickly |
