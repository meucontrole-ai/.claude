# Pre-PR

Full verification before opening a Pull Request. Orchestrates 4 specialized agents in sequence.

## Constraints

| Rule | Details |
| --- | --- |
| Order | metaspec → review → docs → tests |
| Blocking | If code-reviewer returns RED, STOP and fix before continuing |
| Autonomy | Fix found issues automatically when possible |
| Permission | Ask for approval before opening the PR |

## Workflow

| Step | Agent | Action |
| --- | --- | --- |
| 1 | `branch-metaspec-checker` | Verify alignment with meta specs |
| 2 | `branch-code-reviewer` | Review quality, bugs, patterns |
| 3 | - | Fix critical issues from review |
| 4 | `branch-documentation-writer` | Update documentation |
| 5 | `branch-test-planner` | Verify/write missing tests |
| 6 | - | Run final tests |
| 7 | - | Ask approval and open PR |

### Step 1: MetaSpec Check

Invoke `branch-metaspec-checker`. If critical misalignments exist, report to user before continuing.

### Step 2: Code Review

Invoke `branch-code-reviewer`. Analyze traffic light:
- **GREEN**: continue
- **YELLOW**: list caveats, continue
- **RED**: STOP — fix all critical issues before proceeding

### Step 3: Fixes

For each critical issue from review:
1. Read the file with the problem
2. Apply fix
3. Run `go build ./...` to verify

### Step 4: Documentation

Invoke `branch-documentation-writer`. Apply proposed updates.

### Step 5: Tests

Invoke `branch-test-planner`. Write missing tests identified.

### Step 6: Final Validation

```bash
go test ./... -race
golangci-lint run
```

If failures: fix and repeat.

### Step 7: PR

Present summary to user:
- Changes made
- Issues fixed
- Tests added
- Docs updated

Ask permission. If approved, use `/create-pr`.

## Validation

- [ ] All 4 agents ran
- [ ] Critical issues fixed
- [ ] Tests passing with `-race`
- [ ] Lint passing
- [ ] PR opened (with user approval)
