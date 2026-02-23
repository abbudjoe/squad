# Definition of Done

A PR is considered **done** when ALL of the following are true:

## Code Quality
- [ ] Code compiles without errors or new warnings
- [ ] All existing tests pass
- [ ] New features have corresponding tests
- [ ] Bug fixes have regression tests
- [ ] No TODO/FIXME/HACK comments without corresponding issues

## Review
- [ ] At least one structured review has been posted as a PR comment
- [ ] ALL blocking issues resolved
- [ ] ALL non-blocking issues resolved
- [ ] ALL nice-to-have items resolved
- [ ] No deferred items (unless human explicitly approved the specific deferral)
- [ ] Latest review has empty Blocking, Non-blocking, AND Nice-to-have sections

## Git
- [ ] Clean commit history (squash fixups if needed)
- [ ] Conventional commit message format
- [ ] Branch is up to date with base branch
- [ ] No merge conflicts

## CI
- [ ] All CI checks pass (green)
- [ ] No new linting errors
- [ ] Test coverage maintained or improved

## Documentation
- [ ] Code comments for non-obvious logic
- [ ] Public API changes have updated docs/KDoc/JSDoc
- [ ] Breaking changes documented in PR description

## Sign-off
- [ ] `@maintainer ready for merge` comment posted
- [ ] Human maintainer has been notified

---

## Customization

This file is a starting point. Modify it to match your project's standards:

- **Mobile projects:** Add "APK builds successfully", "No new Lint errors"
- **Backend projects:** Add "Database migrations are reversible", "API versioning respected"
- **Libraries:** Add "Changelog updated", "Semver bump appropriate"
- **Monorepos:** Add "Only affected packages are modified", "Cross-package tests pass"
