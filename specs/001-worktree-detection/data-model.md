# Phase 1 Data Model: Worktree-Aware Git Detection

## Entities

- **Source Management Flow**
  - Values: branch, worktree, none
  - Purpose: Selects which Git validation path to run

- **Repository State**
  - Attributes: git_present, is_bare_repo, is_registered_worktree
  - Purpose: Captures detected Git status used to drive validation and messaging

## Relationships

- Source Management Flow determines which Repository State checks are mandatory vs skipped.

## Validation Rules

- If Source Management Flow is worktree, is_registered_worktree must be true or validation fails.
- If Source Management Flow is branch, is_registered_worktree must be false or validation fails.
- If Source Management Flow is none, Git checks are skipped but presence of Git triggers a warning.
