# Quickstart: Worktree-Aware Git Detection

## Purpose

Verify that Spec-Kit correctly detects branch, worktree, and no-Git modes and enforces the expected validation behavior.

## Scenarios

1. **Branch mode**: Run Spec-Kit in a normal working tree with branch flow configured.
2. **Worktree mode**: Run Spec-Kit inside a registered worktree with worktree flow configured.
3. **Mode mismatch**: Run Spec-Kit in a worktree while branch flow is configured (expect blocking error).
4. **No-Git mode with Git present**: Set flow to none in a Git repo (expect warning, no Git checks).
5. **Bare repo with worktree mode**: Use a bare repo with worktree flow and confirm guidance to create the worktree.

## Expected Outcomes

- Correct mode is reported for each scenario.
- Mismatches produce blocking errors.
- No-Git mode warns and continues when Git is present.
- Bare repo in worktree mode continues using configured worktree location.
