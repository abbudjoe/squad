# Local Configuration

The templates in this repo are generic by design — they use `{{variables}}` and describe patterns, not specific infrastructure. To actually run a Squad sprint, you need a **local configuration layer** that fills in your project's specifics.

## What to Create

Mirror the repo structure in a private local directory:

```
your-workspace/
├── squad/                        # ← this public repo (generic templates)
├── squad-local/                  # ← your private config (project-specific)
│   ├── config/
│   │   └── definition-of-done.md # Your project's DoD (build commands, test suites, PR conventions)
│   └── prompts/
│       ├── conductor.md          # Sprint checklist with your merge workflow
│       ├── orchestrator-subagent.md  # Filled template with your repo paths, models, branches
│       ├── reviewer.md           # Reviewer prompt with your project context and patterns
│       ├── fixer.md              # Fixer prompt with your git conventions
│       └── watchdog.md           # Cron templates with your timing preferences
└── SQUAD.md                      # Top-level quick-reference (infra, models, devices)
```

## What Goes in Local Config

Your local prompts should fill in everything the generic templates leave as `{{variables}}`:

### SQUAD.md (quick-reference)
- Repo URL and local paths
- Build node details (IPs, SSH users, paths)
- Model assignments (which model for each role)
- Build and test commands
- Device/hardware details (if applicable)
- Branch naming conventions

### Local Definition of Done
- Your specific build commands (not `{{build_command}}`)
- Your specific test suites and known pre-existing failures to exclude
- Your PR conventions (title tags, description format, review requirements)
- Any project-specific quality gates (lint, coverage thresholds, etc.)

### Local Prompts
- Project context: language, framework, key architectural patterns, test utilities
- Repo paths filled in (no more `{{repo_path}}`)
- Model names filled in (no more `{{reviewer_model}}`)
- Git conventions specific to your team (branch naming, commit format, PR tags)
- Environment constraints (e.g., "no JDK on the orchestrator machine — build on the remote node")

## Keep It Private

Your local config will contain project-specific details:
- Internal hostnames, IPs, node identifiers
- Repo paths and directory structures
- Device serial numbers or hardware identifiers
- Team conventions and workflow preferences

**Do not commit `squad-local/` or `SQUAD.md` to a public repository.** Add them to your `.gitignore`:

```gitignore
# Squad — private local config
/squad-local/
/SQUAD.md
```

If your workspace is version-controlled, ensure the ignore rules are in place before your first commit.

## Workflow

1. **Read the generic template** in `squad/prompts/` to understand the structure and required sections
2. **Read your local version** in `squad-local/prompts/` for the filled-in values
3. **Spawn the agent** using the local version (all variables resolved)
4. **Update the generic repo** if you discover a pattern improvement that isn't project-specific

The generic templates evolve with methodology improvements. Your local config evolves with your infrastructure. Keep them in sync but separate.
