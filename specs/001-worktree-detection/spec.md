# Feature Specification: Git Worktree-Aware Workflows

**Feature Branch**: `001-worktree-detection`  
**Created**: 2026-01-31  
**Status**: Draft  
**Input**: User description: "Extend Spec-Kit's Git detection logic and branch management to support worktree-aware workflows"

## Clarifications

### Session 2026-01-31

- Q: How should the initialization process prompt for source management mode? → A: Auto-detect with confirmation - Detect Git setup automatically, suggest a mode, allow user to override
- Q: When initializing in worktree mode, what should be the default `worktree_folder` path? → A: Same directory as bare repo - `./worktrees` relative to bare repository root
- Q: How should the system respond when worktree mode is selected but the repository is not bare? → A: Warning with proceed option - Show warning about bare repo best practice, allow user to proceed or change mode
- Q: What should happen when creating a worktree but the configured `worktree_folder` path doesn't exist? → A: Auto-create with confirmation - Prompt user asking permission to create the directory
- Q: When a user changes `source_management_flow` from `branch` to `worktree` in an existing project config, what should happen? → A: Block with error - Prevent mode changes after initial setup, require fresh initialization

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Configure Source Management Mode (Priority: P1)

A developer initializing a new Spec-Kit project needs to specify their preferred Git workflow (branch-based, worktree-based, or no Git) so that all Spec-Kit commands behave appropriately for their setup.

**Why this priority**: This is foundational configuration that determines all subsequent Git operations. Without this, the system cannot know which workflow to support.

**Independent Test**: Can be fully tested by running `specify init` and verifying that a `.specify/memory/config.json` file is created with valid `source_management_flow` and `worktree_folder` settings. The system should respect these settings in all Git operations.

**Acceptance Scenarios**:

1. **Given** a new project directory, **When** developer runs initialization, **Then** system auto-detects Git environment and suggests appropriate mode (worktree if in worktree, bare repo, or standard Git; none if no Git)
2. **Given** system suggests a mode, **When** developer is prompted, **Then** developer can accept the suggestion or override with a different mode
3. **Given** developer accepts or selects worktree mode, **When** config is written, **Then** the config includes both `source_management_flow: worktree` and a valid `worktree_folder` path
4. **Given** developer accepts or selects branch mode, **When** config is written, **Then** the config includes `source_management_flow: branch` and omits worktree-specific settings
5. **Given** an existing config file, **When** developer views it, **Then** the settings are readable and clearly documented

---

### User Story 2 - Automatic Worktree Detection (Priority: P1)

A developer working in a Git worktree needs Spec-Kit to automatically detect this environment so that commands behave correctly without manual configuration overrides.

**Why this priority**: Detection is the core enabler for worktree support. Without reliable detection, the system cannot adapt its behavior.

**Independent Test**: Can be tested by creating a test worktree, navigating into it, and running Spec-Kit commands. The system should recognize it's in a worktree and display appropriate status messages.

**Acceptance Scenarios**:

1. **Given** developer is working in a registered Git worktree, **When** any Spec-Kit command runs, **Then** the system correctly identifies this as a worktree environment
2. **Given** developer is in a standard Git repository (not a worktree), **When** Spec-Kit checks the environment, **Then** the system identifies it as a standard repo
3. **Given** developer is in a bare repository, **When** Spec-Kit runs detection, **Then** the system identifies it as a bare repo and adjusts behavior accordingly
4. **Given** developer is in a directory with no Git repository, **When** detection runs, **Then** the system gracefully handles the absence of Git

---

### User Story 3 - Worktree Creation for New Features (Priority: P2)

A developer starting a new feature in worktree mode needs Spec-Kit to create a new worktree (not just a branch) following the established naming convention so that each feature has its own isolated workspace.

**Why this priority**: This builds on the detection capability (P1) to provide automated worktree creation, making the workflow seamless.

**Independent Test**: Can be tested by running the feature creation command in worktree mode and verifying that a new worktree directory is created in the configured location with the correct naming pattern.

**Acceptance Scenarios**:

1. **Given** config is set to worktree mode, **When** developer creates a new feature, **Then** a new worktree is created under the configured worktree folder
2. **Given** the naming standard is `###-feature-name`, **When** worktree is created, **Then** the worktree directory follows this exact pattern
3. **Given** worktree folder uses default path `./worktrees`, **When** new worktree is created, **Then** it appears in `/path/to/repo/worktrees/###-feature-name`
4. **Given** developer is already in a worktree, **When** creating another feature, **Then** the new worktree is created from the appropriate base branch

---

### User Story 4 - Worktree Branch Validation (Priority: P3)

A developer working in worktree mode needs validation that their current worktree and branch setup conforms to expected patterns so that they can identify and fix configuration mismatches early.

**Why this priority**: This is a quality-of-life enhancement that helps prevent errors but isn't required for basic functionality.

**Independent Test**: Can be tested by creating mismatched worktree/branch scenarios (e.g., worktree named X but branch named Y) and verifying that validation warnings appear.

**Acceptance Scenarios**:

1. **Given** developer is in a worktree with mismatched branch name, **When** validation runs, **Then** a warning indicates the mismatch
2. **Given** worktree and branch names align correctly, **When** validation runs, **Then** no warnings are shown
3. **Given** developer runs `check_feature_branch()` in worktree mode, **When** layout is correct, **Then** confirmation message indicates successful validation

---

### User Story 5 - Clear Mode Indicators (Priority: P3)

A developer running Spec-Kit commands needs clear visual indicators of which source management mode is active so that they understand why certain operations behave differently.

**Why this priority**: Improves user experience and reduces confusion, but the core functionality works without enhanced logging.

**Independent Test**: Can be tested by running commands in each mode (branch/worktree/none) and verifying that appropriate mode indicators appear in output.

**Acceptance Scenarios**:

1. **Given** config is set to worktree mode, **When** any Git operation runs, **Then** logs indicate "Worktree mode active"
2. **Given** config is set to branch mode, **When** Git operations run, **Then** logs indicate "Branch mode active"
3. **Given** Git is not available, **When** commands run, **Then** logs indicate "No Git mode - limited functionality"

---

### Edge Cases

- **Non-bare repository with worktree mode**: System displays warning message explaining bare repository best practices and potential issues, then prompts user to either change to branch mode or proceed with worktree mode anyway
- **Mode switching in existing project**: System blocks attempts to change `source_management_flow` after initial configuration and displays error message instructing user to reinitialize project with desired mode
- **Missing worktree folder**: When worktree folder doesn't exist, system prompts user for permission to create it; if user declines, operation aborts with clear error message
- **Unwritable worktree folder**: When worktree folder exists but isn't writable, system fails with permission error and instructs user to fix directory permissions
- **Unregistered worktree directory**: **Given** a directory exists at the configured worktree location but is not in `git worktree list`, **When** attempting to create a new worktree with the same name, **Then** Git's native error prevents creation and system displays error directing user to remove conflicting directory or use `git worktree repair`
- **Duplicate branch names across worktrees**: **Given** Git itself prevents multiple worktrees from sharing the same branch (returns error "branch already checked out"), **When** system attempts creation, **Then** system displays Git's error and instructs user to choose different feature name
- **Non-standard worktree locations**: **Deferred to Phase 2** - Current scope supports only worktrees created via `git worktree add` in configured folder; worktrees created manually or in arbitrary locations are out of scope for initial release

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST create a JSON configuration file at `.specify/memory/config.json` during project initialization containing a `source_management_flow` field accepting values: `branch`, `worktree`, or `none`
- **FR-001a**: System MUST auto-detect the current Git environment (worktree, bare repository, standard Git, or no Git) during initialization
- **FR-001b**: System MUST suggest an appropriate source management mode based on auto-detection and allow user to confirm or override the suggestion
- **FR-002**: Configuration file MUST include a `worktree_folder` field when `source_management_flow` is set to `worktree`
- **FR-002a**: Default value for `worktree_folder` MUST be `./worktrees` relative to the bare repository root when auto-configuring worktree mode
- **FR-003**: System MUST provide a detection function that identifies if the current directory is a registered Git worktree
- **FR-004**: System MUST provide a detection function that identifies if a repository is bare
- **FR-005**: System MUST read and respect the `source_management_flow` setting from config for all Git operations
- **FR-006**: System MUST create new worktrees (not branches) when in worktree mode and a new feature is initiated
- **FR-007**: New worktrees MUST be created in the directory specified by `worktree_folder` configuration
- **FR-008**: Worktree directory names MUST follow the same naming convention as feature branches (`###-feature-name`)
- **FR-009**: System MUST extend existing branch validation logic to validate worktree/branch alignment when in worktree mode
- **FR-010**: System MUST display mode-appropriate warnings and informational messages indicating active source management mode
- **FR-011**: System MUST maintain backward compatibility with existing branch-based workflows
- **FR-012**: System MUST gracefully handle environments where Git is not available (none mode)
- **FR-013**: Worktree detection MUST work regardless of whether the worktree was created with `git worktree add` using relative or absolute paths
- **FR-014**: System MUST validate that configured `worktree_folder` path exists and is accessible before attempting worktree operations
- **FR-014a**: When `worktree_folder` path doesn't exist, system MUST prompt user for permission to create it before proceeding
- **FR-014b**: When `worktree_folder` exists but lacks write permissions, system MUST fail with clear error message instructing user to fix permissions
- **FR-015**: System MUST display a warning when worktree mode is selected in a non-bare repository, explaining best practices and allowing user to proceed or change mode
- **FR-016**: System MUST prevent changes to `source_management_flow` after initial configuration and display error message directing user to reinitialize if mode change is needed

### Key Entities

- **ConfigFile**: JSON file at `.specify/memory/config.json` containing source management settings including flow type and worktree folder location
- **Worktree**: A Git worktree representing an isolated working directory for a feature, with attributes including path, associated branch, and registration status
- **Source Management Mode**: Enumeration of workflow types (branch, worktree, none) that determines how Spec-Kit manages version control operations

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Developers can initialize a project with worktree mode and have all subsequent feature creations automatically create worktrees
- **SC-002**: Worktree detection accuracy reaches 100% for worktrees created via `git worktree add` (standard configurations); non-standard worktree setups are out of scope and may not be detected
- **SC-003**: Zero breaking changes to existing branch-based workflows - all existing Spec-Kit projects continue functioning without modification
- **SC-004**: Configuration file is human-readable JSON with clear field names and can be manually edited with any text editor without requiring JSON schema validation tools
- **SC-005**: Mode indicator messages appear in command output 100% of the time, allowing developers to immediately understand which mode is active
- **SC-006**: Validation logic catches 100% of worktree/branch name mismatches when running in worktree mode
- **SC-007**: System handles all three modes (branch/worktree/none) without errors or degraded functionality specific to each mode's expected capabilities
