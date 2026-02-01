# Phase 0 Research: Worktree-Aware Git Detection

## Decision 1: Git worktree detection approach

- **Decision**: Use Git CLI commands that are already expected by Spec-Kit workflows, relying on `git worktree list` for worktree awareness and `git rev-parse --is-bare-repository` for bare detection.
- **Rationale**: These commands are supported in standard Git installations and align with existing CLI-based detection patterns.
- **Alternatives considered**: Parsing `.git` metadata directly; avoided to reduce edge cases and stay consistent with existing Git checks.

## Decision 2: Mode mismatch handling

- **Decision**: Treat mode mismatches as blocking errors (branch vs worktree), warn and continue when `source_management_flow` is `none` but Git is present, and continue in worktree mode for bare repos using the configured worktree location.
- **Rationale**: Matches clarified stakeholder expectations; avoids hybrid ambiguity while preserving explicit no-Git intent.
- **Alternatives considered**: Auto-switching modes; rejected due to ambiguity and loss of explicit configuration intent.
