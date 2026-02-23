# Running a Single Review/Fix Loop

This guide walks through running one PR through the Squad review/fix cycle.

## Prerequisites

- An AI agent platform that supports subagent spawning (e.g., OpenClaw, Claude Code)
- A Git repository with a PR open against a base branch
- `gh` CLI authenticated for posting PR comments
- Access to at least two model tiers: a reasoning model (reviewer) and a coding model (fixer)

## Step 1: Open the PR

Create your branch, implement the feature/fix, and open a PR:

```bash
git checkout -b feat/my-feature origin/main
# ... implement ...
git add -A && git commit -m "feat(scope): implement feature X"
git push origin feat/my-feature
gh pr create --base main --head feat/my-feature --title "feat: feature X" --body "..."
```

## Step 2: Spawn the Reviewer

Using the [reviewer prompt template](../prompts/reviewer.md), spawn a subagent:

```
# Pseudocode — adapt to your platform
spawn_subagent(
    task = fill_template("prompts/reviewer.md", {
        project_name: "MyProject",
        pr_number: 42,
        head_branch: "feat/my-feature",
        base_branch: "main",
        pr_diff: $(gh pr diff 42),
        context_notes: "This PR adds...",
        repo: "owner/repo"
    }),
    model = "opus",       # or any high-reasoning model
    label = "pr42-reviewer-r1"
)
```

The reviewer will:
1. Analyze the diff
2. Post a structured review as a PR comment
3. Return completion

## Step 3: Check the Review

When the reviewer completes, read the PR comment:

```bash
gh pr view 42 --comments --json comments | jq -r '.comments[-1].body'
```

**If all sections are empty** → Skip to Step 6.

**If any issues exist** → Continue to Step 4.

## Step 4: Spawn the Fixer

Using the [fixer prompt template](../prompts/fixer.md), spawn a fixer subagent:

```
spawn_subagent(
    task = fill_template("prompts/fixer.md", {
        project_name: "MyProject",
        pr_number: 42,
        review_issues: "B1: Missing test for...\nNB1: Inconsistent naming...",
        repo_path: "/path/to/repo",
        branch_name: "feat/my-feature",
        key_files: "src/feature.ts, tests/feature.test.ts",
        build_command: "npm run build",
        test_command: "npm test",
        scope: "feature",
        repo: "owner/repo"
    }),
    model = "codex",      # or any fast coding model
    label = "pr42-fixer-r1"
)
```

The fixer will:
1. Resolve all issues
2. Commit and push
3. Post a chain-of-custody comment

## Step 5: Re-Review

After the fixer completes, spawn another reviewer:

```
spawn_subagent(
    task = fill_template("prompts/reviewer.md", {
        # Same as Step 2, but with updated diff and re-review instructions
        context_notes: "RE-REVIEW after fixer addressed R1 items. Verify each was fixed."
    }),
    model = "opus",
    label = "pr42-reviewer-r2"
)
```

**If clean** → Continue to Step 6.
**If issues remain** → Go back to Step 4.

## Step 6: Tag Ready for Merge

```bash
gh pr comment 42 --repo owner/repo --body "@maintainer ready for merge"
```

## Step 7: Set Up the Watchdog (Optional but Recommended)

If you want the loop to continue autonomously (e.g., while you sleep), create a watchdog cron using the [watchdog template](../prompts/watchdog.md).

The watchdog ensures:
- If a reviewer finishes and no fixer is spawned → it spawns one
- If a fixer finishes and no re-reviewer is spawned → it spawns one
- If the review is clean → it posts the ready-for-merge comment

## Typical Loop Timeline

```
T+0m   Reviewer spawned
T+2m   Review posted (3 blocking, 2 non-blocking, 1 nice-to-have)
T+2m   Fixer spawned
T+5m   Fixer committed, pushed, posted chain-of-custody
T+5m   Re-reviewer spawned
T+7m   Re-review posted (1 non-blocking remaining)
T+7m   Fixer spawned
T+9m   Fixer committed, pushed
T+9m   Re-reviewer spawned
T+11m  Review clean — all sections empty
T+11m  @maintainer ready for merge
```

Total: ~11 minutes for 3 review cycles, fully autonomous.
