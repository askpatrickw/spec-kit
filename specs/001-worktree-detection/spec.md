# Feature Specification: Worktree-Aware Git Detection

**Feature Branch**: `001-worktree-detection`  
**Created**: 2026-01-31  
**Status**: Draft  
**Input**: User description: "Extend Spec-Kit's Git detection logic to support worktree-aware workflows"

## Clarifications

### Session 2026-01-31

- Q: When worktree flow is enabled but the current directory is not a registered worktree, should Spec-Kit fail or warn? → A: Fail the run with a blocking error.
- Q: When branch flow is enabled but the current directory is a registered worktree, how should Spec-Kit respond? → A: Fail the run with a blocking error.
- Q: When `source_management_flow` is set to `none` but Git is present, how should Spec-Kit respond? → A: Warn and continue (non-blocking).
- Q: If a bare repository is detected while worktree mode is enabled, how should Spec-Kit respond? → A: Continue in worktree mode and use the configured worktree location, guiding the user to create the worktree if it is missing.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Detect source management mode (Priority: P1)

As a developer running Spec-Kit in a repository, I want the tool to correctly identify whether the project uses branch flow, multiple working copies (worktrees), or no-Git workflows so that guidance and validations are appropriate.

**Why this priority**: The detection outcome drives all subsequent behavior and messaging.

**Independent Test**: Can be fully tested by running Spec-Kit in three environments (branch, worktree, no-Git) and observing mode detection results.

**Acceptance Scenarios**:

1. **Given** the repo is configured for branch-based flow, **When** Spec-Kit runs, **Then** it reports branch mode and uses branch-specific validation rules.
2. **Given** the repo is configured for worktree-based flow, **When** Spec-Kit runs, **Then** it reports worktree mode and uses worktree-specific validation rules.
3. **Given** branch flow is configured and the current directory is a registered worktree, **When** Spec-Kit validates the environment, **Then** validation fails with a blocking error that explains the mismatch and how to resolve it.

---

### User Story 2 - Validate worktree layout when enabled (Priority: P2)

As a developer using worktrees, I want Spec-Kit to validate that my current worktree and branch layout matches the expected worktree flow so I can quickly correct misconfigurations.

**Why this priority**: Worktree users need clear, actionable validation to avoid confusing errors later.

**Independent Test**: Can be fully tested by toggling the configured flow and observing validation results in a registered worktree vs an unregistered directory.

**Acceptance Scenarios**:

1. **Given** worktree flow is enabled and the current directory is a registered worktree, **When** Spec-Kit validates the environment, **Then** validation succeeds and messaging confirms worktree mode.
2. **Given** worktree flow is enabled and the current directory is not a registered worktree, **When** Spec-Kit validates the environment, **Then** validation fails with a blocking error that explains the mismatch and how to resolve it.

---

### User Story 3 - Handle bare or no-Git environments (Priority: P3)

As a developer running Spec-Kit in unusual repository states, I want clear messaging about bare or no-Git setups so I understand what functionality is available.

**Why this priority**: These environments are less common but can cause confusing failures without explicit handling.

**Independent Test**: Can be fully tested by running Spec-Kit in a bare repository and in a directory without Git metadata.

**Acceptance Scenarios**:

1. **Given** the root repository is bare, **When** Spec-Kit runs, **Then** it reports a bare repository state and limits checks that require a working tree.
2. **Given** no Git metadata is available, **When** Spec-Kit runs, **Then** it reports no-Git mode and avoids Git-dependent checks.
3. **Given** worktree mode is enabled and the repository is bare, **When** Spec-Kit runs, **Then** it continues using the configured worktree location and guides the user to create the worktree if it is missing.
4. **Given** `source_management_flow` is set to `none` and Git metadata is present, **When** Spec-Kit runs, **Then** it warns and continues without running Git-dependent checks.

---

### Edge Cases

- Worktree flow is configured but the current directory is a detached checkout or otherwise lacks a branch association.
- A repository is present but the root is a bare repository with no working tree.
- Git metadata is present but inaccessible due to permissions or environment constraints.
- The configured `source_management_flow` is set to `none` while Git is available.
- `source_management_flow` is set to `none` in a directory that has Git metadata.
- Worktree mode is enabled in a bare repository where the configured worktree location is missing.

## Assumptions

- The `source_management_flow` configuration is the primary switch for selecting branch, worktree, or no-Git behavior.
- If `source_management_flow` is unset, Spec-Kit defaults to existing branch-based behavior to preserve backward compatibility.

## Scope

In scope: Detect the active source management mode, validate the environment against that mode, and communicate the result clearly.
Out of scope: Changing existing Git workflows, creating or removing worktrees, or altering repository history.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST detect whether the current directory is within a Git repository and expose this state for downstream checks.
- **FR-002**: System MUST determine whether the root repository is bare and expose this state for downstream checks.
- **FR-003**: System MUST determine whether the current directory is a registered worktree when worktree flow is enabled.
- **FR-004**: System MUST respect the `source_management_flow` configuration to select `branch`, `worktree`, or `none` behavior.
- **FR-005**: System MUST apply branch-specific validation rules only when `source_management_flow` is `branch`.
- **FR-006**: System MUST apply worktree-specific validation rules only when `source_management_flow` is `worktree`.
- **FR-007**: System MUST skip Git-dependent validations when `source_management_flow` is `none` or when Git metadata is unavailable.
- **FR-008**: System MUST provide user-facing messaging that clearly states the detected source management mode.
- **FR-009**: When worktree flow is enabled and the current directory is not a registered worktree, the system MUST fail the run with a blocking error that describes the mismatch and expected setup.
- **FR-010**: When the root repository is bare, the system MUST warn that working-tree validations are unavailable.
- **FR-011**: When branch flow is enabled and the current directory is a registered worktree, the system MUST fail the run with a blocking error that describes the mismatch and expected setup.
- **FR-012**: When `source_management_flow` is `none` and Git metadata is present, the system MUST warn and continue without running Git-dependent checks.
- **FR-013**: When worktree mode is enabled and the repository is bare, the system MUST continue using the configured worktree location and guide the user to create the worktree if it is missing.

### Requirement Acceptance Criteria

- **AC-001**: FR-001 is satisfied when Spec-Kit reliably indicates whether Git metadata is present for the current directory.
- **AC-002**: FR-002 is satisfied when Spec-Kit identifies whether the repository is bare and adjusts checks accordingly.
- **AC-003**: FR-003 is satisfied when Spec-Kit can confirm if the current directory is a registered worktree when that mode is selected.
- **AC-004**: FR-004 is satisfied when the configured flow consistently determines which validations run.
- **AC-005**: FR-005 and FR-006 are satisfied when branch and worktree validations never run outside their configured modes.
- **AC-006**: FR-007 is satisfied when Git-dependent checks are skipped in no-Git mode or when Git is unavailable.
- **AC-007**: FR-008 through FR-013 are satisfied when user-facing messages clearly state the mode, mode mismatches fail with a blocking error, no-Git mode warns and continues when Git is present, and bare repositories in worktree mode continue using configured worktree locations with clear guidance.

### Key Entities

- **Source Management Flow**: Configuration setting that selects branch, worktree (separate working copies), or no-Git behavior.
- **Repository State**: Derived indicators such as Git presence, worktree registration, and bare repository status.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In test environments representing branch, worktree, and no-Git setups, mode detection is correct in 100% of runs.
- **SC-002**: When worktree flow is enabled, 100% of runs in registered worktrees pass validation without false warnings.
- **SC-003**: When worktree flow is enabled in a non-worktree directory, the mismatch warning appears in 100% of runs.
- **SC-004**: User support reports related to incorrect Git mode detection decrease by at least 50% within one release cycle.
