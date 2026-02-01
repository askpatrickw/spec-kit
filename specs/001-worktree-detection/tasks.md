# Tasks: Git Worktree-Aware Workflows

**Input**: Design documents from `/specs/001-worktree-detection/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Tests**: NOT requested in specification - acceptance scenarios serve as manual validation criteria

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `src/specify_cli/__init__.py` (main CLI module)
- **Scripts**: `scripts/bash/` and `scripts/powershell/`
- **Config**: `.specify/memory/config.json`

---

## Phase 1: Setup

**Purpose**: Project initialization and basic structure

- [X] T001 Verify branch `001-worktree-detection` exists and is checked out
- [X] T002 Verify `.specify/memory/` directory exists (created by existing init process)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core Git detection and config infrastructure - MUST be complete before ANY user story

**⚠️ CRITICAL**: All user stories depend on these foundational functions

- [X] T003 [P] Add `GitEnvironment` dataclass to src/specify_cli/__init__.py (fields: is_repo, is_worktree, is_bare, suggested_mode)
- [X] T004 [P] Add `is_git_worktree()` function to src/specify_cli/__init__.py (uses `git worktree list | grep`)
- [X] T005 [P] Add `is_bare_repo()` function to src/specify_cli/__init__.py (uses `git rev-parse --is-bare-repository`)
- [X] T006 Add `detect_git_environment()` function to src/specify_cli/__init__.py (combines all detections, returns GitEnvironment)
- [X] T007 [P] Add `get_config_path()` helper function to src/specify_cli/__init__.py (returns `.specify/memory/config.json` path)
- [X] T008 [P] Add `load_config()` function to src/specify_cli/__init__.py (reads and validates JSON config)
- [X] T009 [P] Add `save_config()` function to src/specify_cli/__init__.py (writes JSON config with validation)
- [X] T010 [P] Add `validate_config()` function to src/specify_cli/__init__.py (validates config structure and required fields per contracts/config-schema.json)

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Configure Source Management Mode (Priority: P1) 🎯 MVP

**Goal**: Enable developers to specify their preferred Git workflow during initialization so Spec-Kit behaves appropriately

**Independent Test**: Run `specify init` in a test directory and verify:
1. Config file `.specify/memory/config.json` is created
2. Config contains valid `source_management_flow` ("branch", "worktree", or "none")
3. If worktree mode: config includes `worktree_folder` path
4. System respects config in subsequent operations

**Implements**: FR-001, FR-001a, FR-001b, FR-002, FR-003, FR-003a, FR-016, FR-017

### Implementation for User Story 1

- [X] T011 [US1] Modify `init()` command in src/specify_cli/__init__.py to call `detect_git_environment()` before template download
- [X] T012 [US1] Add mode detection display logic in src/specify_cli/__init__.py `init()` to show "Detected: [environment]" message
- [X] T013 [US1] Add mode selection prompt in src/specify_cli/__init__.py `init()` using Rich Prompt (suggest detected mode, allow override)
- [X] T014 [US1] Add worktree folder prompt in src/specify_cli/__init__.py `init()` when mode=worktree (default: "./worktrees")
- [X] T015 [US1] Add non-bare repository warning in src/specify_cli/__init__.py `init()` when worktree mode selected in non-bare repo (FR-016)
- [X] T016 [US1] Add config creation logic in src/specify_cli/__init__.py `init()` after template extraction (calls `save_config()` with user selections)
- [X] T017 [US1] Add config validation in src/specify_cli/__init__.py `init()` to verify saved config matches schema
- [X] T018 [US1] Add error handling in src/specify_cli/__init__.py for mode=none scenario (skip Git operations gracefully)

**Checkpoint**: User Story 1 complete - initialization creates config, mode selection works, config persisted correctly

---

## Phase 4: User Story 2 - Automatic Worktree Detection (Priority: P1) 🎯 MVP

**Goal**: Enable Spec-Kit to automatically detect Git worktree environments so commands behave correctly without manual overrides

**Independent Test**: 
1. Create test worktree: `git worktree add ../test-wt test-branch`
2. Navigate to worktree: `cd ../test-wt`
3. Run Spec-Kit commands
4. Verify: System recognizes worktree environment and displays appropriate messages

**Implements**: FR-004, FR-005, FR-006, FR-013, FR-014

### Implementation for User Story 2

- [X] T019 [P] [US2] Add `get_current_mode()` helper function to src/specify_cli/__init__.py (reads config, returns mode with fallback to "branch")
- [X] T020 [P] [US2] Add mode indicator logging to src/specify_cli/__init__.py in relevant commands (displays "Worktree mode active", "Branch mode active", or "No Git mode")
- [X] T021 [US2] Add backward compatibility check to src/specify_cli/__init__.py for missing config (treat as "branch" mode with warning)
- [X] T022 [US2] Add error messages to src/specify_cli/__init__.py for malformed config (display config path and validation errors)
- [X] T023 [US2] Update Git operation wrappers in src/specify_cli/__init__.py to respect mode from config (skip Git ops when mode=none)

**Checkpoint**: User Story 2 complete - detection works in all environments, mode indicators appear, missing/invalid config handled gracefully

**🎉 MVP COMPLETE**: At this point, User Stories 1 & 2 provide core worktree detection and configuration. Stop here to validate MVP before proceeding.

---

## Phase 5: User Story 3 - Worktree Creation for New Features (Priority: P2)

**Goal**: Enable automatic worktree creation (not branches) when creating features in worktree mode

**Independent Test**:
1. Initialize project in worktree mode
2. Run feature creation command
3. Verify: New worktree created in configured folder with correct naming pattern (###-feature-name)

**Implements**: FR-007, FR-008, FR-009, FR-015, FR-015a, FR-015b

### Implementation for User Story 3

- [X] T024 [P] [US3] Add JSON config reading to scripts/bash/create-new-feature.sh (parse source_management_flow and worktree_folder using grep/sed)
- [X] T025 [P] [US3] Add JSON config reading to scripts/powershell/create-new-feature.ps1 (parse using ConvertFrom-Json)
- [X] T026 [US3] Add mode branching logic to scripts/bash/create-new-feature.sh (if mode=worktree, use `git worktree add`, else use `git checkout -b`)
- [X] T027 [US3] Add mode branching logic to scripts/powershell/create-new-feature.ps1 (if mode=worktree, use `git worktree add`, else use `git checkout -b`)
- [X] T028 [P] [US3] Add worktree folder existence check to scripts/bash/create-new-feature.sh (FR-015)
- [X] T029 [P] [US3] Add worktree folder existence check to scripts/powershell/create-new-feature.ps1 (FR-015)
- [X] T030 [US3] Add missing folder creation prompt to scripts/bash/create-new-feature.sh (FR-015a - ask permission before creating)
- [X] T031 [US3] Add missing folder creation prompt to scripts/powershell/create-new-feature.ps1 (FR-015a - ask permission before creating)
- [X] T032 [P] [US3] Add permission error handling to scripts/bash/create-new-feature.sh (FR-015b - clear error if folder not writable)
- [X] T033 [P] [US3] Add permission error handling to scripts/powershell/create-new-feature.ps1 (FR-015b - clear error if folder not writable)

**Checkpoint**: User Story 3 complete - worktree creation works in bash and PowerShell, folder validation complete, prompts functional

---

## Phase 6: User Story 4 - Worktree Branch Validation (Priority: P3)

**Goal**: Validate that worktree directory names match branch names to catch configuration mismatches early

**Independent Test**:
1. Create worktree with mismatched names (directory: 001-foo, branch: 002-bar)
2. Run validation command
3. Verify: Warning displays indicating mismatch

**Implements**: FR-010

### Implementation for User Story 4

- [X] T034 [P] [US4] Locate `check_feature_branch()` function in scripts/bash/common.sh (line 65) and verify worktree detection needs
- [X] T035 [US4] Add worktree detection to validation logic (check if current directory is worktree)
- [X] T036 [US4] Add worktree/branch name alignment validation (extract directory name and branch name, compare)
- [X] T037 [US4] Add mismatch warning messages (display clear warning when names don't align)
- [X] T038 [US4] Add success confirmation messages (indicate validation passed when names align)

**Checkpoint**: User Story 4 complete - validation catches mismatches, warnings clear and actionable

---

## Phase 7: User Story 5 - Clear Mode Indicators (Priority: P3)

**Goal**: Display clear visual indicators of which source management mode is active

**Independent Test**:
1. Run commands in each mode (branch/worktree/none)
2. Verify: Appropriate mode indicator appears in command output

**Implements**: FR-011

### Implementation for User Story 5

- [X] T039 [P] [US5] Add mode indicator to feature creation output in scripts/bash/create-new-feature.sh
- [X] T040 [P] [US5] Add mode indicator to feature creation output in scripts/powershell/create-new-feature.ps1
- [X] T041 [P] [US5] Add mode indicator to Git operation logging in src/specify_cli/__init__.py (any functions that call Git)
- [X] T042 [US5] Standardize mode indicator format across all output (consistent wording: "Worktree mode active", "Branch mode active", "No Git mode")
- [X] T043 [US5] Add mode indicator to init completion message in src/specify_cli/__init__.py `init()`

**Checkpoint**: User Story 5 complete - mode indicators consistent across all commands

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, cleanup, and final validation

- [X] T044 [P] Update main README.md with worktree mode documentation section
- [X] T045 [P] Update CHANGELOG.md with feature 001 entry
- [X] T046 [P] Verify all acceptance scenarios from spec.md against implementation
- [X] T047 Run `.specify/scripts/bash/update-agent-context.sh` to update AI agent context files
- [X] T048 Review code for consistency with existing Spec-Kit patterns
- [X] T049 Test initialization in all three modes (branch, worktree, none) manually
- [X] T050 Validate config.json format matches contracts/config-schema.json
- [X] T051 [P] Verify backward compatibility with existing projects (test in project directory without .specify/memory/config.json, verify defaults to branch mode without errors)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1: Setup**: No dependencies - can start immediately
- **Phase 2: Foundational**: Depends on Setup - BLOCKS all user stories
- **Phase 3: US1**: Depends on Foundational completion - Can start immediately after Phase 2
- **Phase 4: US2**: Depends on Foundational completion - Can start in parallel with Phase 3
- **Phase 5: US3**: Depends on Foundational AND US1 (needs config creation from US1)
- **Phase 6: US4**: Depends on Foundational AND US3 (needs worktree creation from US3)
- **Phase 7: US5**: Depends on Foundational - Can start in parallel with Phase 3/4
- **Phase 8: Polish**: Depends on all desired user stories

### User Story Dependencies

```
Foundational (Phase 2)
    ├── US1 (Phase 3) - Configure Source Management Mode
    │       └── US3 (Phase 5) - Worktree Creation (needs config from US1)
    │               └── US4 (Phase 6) - Validation (needs creation from US3)
    ├── US2 (Phase 4) - Automatic Detection (independent of US1)
    └── US5 (Phase 7) - Mode Indicators (independent of US1)
```

**Key Insight**: US1 must complete before US3. US2 and US5 can proceed in parallel with US1.

### Within Each User Story

- Models/dataclasses before functions that use them (T003 before T004-T006)
- Helper functions before commands that use them (T007-T010 before T011-T018)
- Config reading before worktree operations (T024-T025 before T026-T027)
- Validation functions before scripts that call them (T034-T038 before feature scripts can validate)

### Parallel Opportunities

**Phase 2 Foundational** (after T003):
- T004, T005 can run in parallel (different functions)
- T007, T008, T009, T010 can run in parallel (different functions)

**Phase 3 US1**:
- T011-T014 are sequential (modify same init() function in order)
- T015-T018 can run after T014, some in parallel if modifying different sections

**Phase 4 US2**:
- T019, T020 can run in parallel (different functions)
- T021, T022, T023 can run in parallel (different code sections)

**Phase 5 US3**:
- T024 (bash) and T025 (PowerShell) can run in parallel (different files)
- T028 (bash) and T029 (PowerShell) can run in parallel (different files)
- T032 (bash) and T033 (PowerShell) can run in parallel (different files)

**Phase 6 US4**:
- T034 is blocking (must locate function first)
- T035-T038 are sequential (modify same validation logic)

**Phase 7 US5**:
- T039, T040, T041 can run in parallel (different files)

**Phase 8 Polish**:
- T044, T045, T046 can run in parallel (different files)

---

## Parallel Example: Foundational Phase

```bash
# After T003 completes, launch these in parallel:
Task T004: "Add is_git_worktree() function to src/specify_cli/__init__.py"
Task T005: "Add is_bare_repo() function to src/specify_cli/__init__.py"

# After T006 completes, launch these in parallel:
Task T007: "Add get_config_path() helper to src/specify_cli/__init__.py"
Task T008: "Add load_config() function to src/specify_cli/__init__.py"
Task T009: "Add save_config() function to src/specify_cli/__init__.py"
Task T010: "Add validate_config() function to src/specify_cli/__init__.py"
```

---

## Parallel Example: User Story 3

```bash
# Launch bash and PowerShell config reading in parallel:
Task T024: "Add JSON config reading to scripts/bash/create-new-feature.sh"
Task T025: "Add JSON config reading to scripts/powershell/create-new-feature.ps1"

# Launch folder checks in parallel:
Task T028: "Add worktree folder existence check to scripts/bash/create-new-feature.sh"
Task T029: "Add worktree folder existence check to scripts/powershell/create-new-feature.ps1"
```

---

## Implementation Strategy

### MVP First (US1 + US2 Only) - RECOMMENDED

1. ✅ Complete Phase 1: Setup (T001-T002)
2. ✅ Complete Phase 2: Foundational (T003-T010) - CRITICAL blocking phase
3. ✅ Complete Phase 3: User Story 1 (T011-T018) - Config and mode selection
4. ✅ Complete Phase 4: User Story 2 (T019-T023) - Detection and indicators
5. **STOP and VALIDATE**: Test initialization with all three modes, verify detection works
6. If validation passes: MVP ready to merge/deploy

**Why stop here?**: US1+US2 provide the core capability (mode configuration and detection). US3-US5 are enhancements that can be added incrementally.

### Incremental Delivery (Full Feature)

1. Complete Setup + Foundational → Foundation ready
2. Add US1 → Test independently → Config creation works ✓
3. Add US2 → Test independently → Detection works ✓
4. Deploy/demo MVP (US1+US2)
5. Add US3 → Test independently → Worktree creation works ✓
6. Add US4 → Test independently → Validation works ✓
7. Add US5 → Test independently → Indicators complete ✓
8. Polish → Final release

### Parallel Team Strategy

With multiple developers (after Foundational completes):

1. **Developer A**: US1 (T011-T018) - Focus on init() command modifications
2. **Developer B**: US2 (T019-T023) - Focus on detection and mode reading
3. **Developer C**: US5 (T039-T043) - Focus on mode indicators (independent of US1)

After US1 completes:
4. **Developer A**: US3 (T024-T033) - Worktree creation scripts
5. **Developer C**: US4 (T034-T038) - Validation logic

---

## Notes

- **[P] tasks**: Different files or independent code sections, no dependencies
- **[Story] labels**: Map tasks to user stories for traceability
- Each user story is independently testable (see "Independent Test" sections)
- Stop at any checkpoint to validate story independently
- Config format: JSON (no new dependencies, shell-parseable)
- Detection method: `git worktree list | grep` (reliable, cross-version)
- Config location: `.specify/memory/config.json` (gitignored, user-specific)
- Mode switching blocked after init (FR-017) - simplifies implementation

**Total Tasks**: 51
- Setup: 2 tasks
- Foundational: 8 tasks (BLOCKING)
- US1: 8 tasks (P1 - MVP)
- US2: 5 tasks (P1 - MVP)
- US3: 10 tasks (P2)
- US4: 5 tasks (P3)
- US5: 5 tasks (P3)
- Polish: 7 tasks

**MVP Scope**: 23 tasks (Setup + Foundational + US1 + US2)
**Full Feature**: 50 tasks (all phases)

**Parallel Opportunities Identified**: 18 tasks marked [P] across all phases

**Independent Test Criteria**: Each user story (US1-US5) includes "Independent Test" section with specific validation steps
