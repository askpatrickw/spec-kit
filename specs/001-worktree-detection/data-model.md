# Data Model: Git Worktree-Aware Workflows

**Feature**: 001-worktree-detection  
**Date**: 2026-01-31  
**Status**: Phase 1 Design

This document defines the data structures, validation rules, and state transitions for the worktree-aware workflow feature.

---

## Entity 1: ConfigFile

**Purpose**: Persists user's source management mode selection and related settings.

**Location**: `.specify/memory/config.json`

**Structure**:
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

**Note**: Comments cannot be embedded in JSON. Documentation is provided in quickstart.md and error messages.

### Fields

| Field | Type | Required | Constraints | Default |
|-------|------|----------|-------------|---------|
| `version` | string | Yes | Must be "1.0" | "1.0" |
| `source_management_flow` | enum(string) | Yes | One of: "branch", "worktree", "none" | Auto-detected |
| `worktree_folder` | string (path) | Conditional | Required if flow=worktree; must be relative or absolute path | "./worktrees" |

### Validation Rules

1. **Schema Validation**:
   - `version` field must exist and equal "1.0"
   - `source_management_flow` must exist and be one of three valid values
   - `worktree_folder` must exist if and only if `source_management_flow == "worktree"`

2. **Path Validation** (when `worktree_folder` is present):
   - May be relative (e.g., `./worktrees`, `../worktrees`) or absolute
   - Should not contain special characters that are invalid on target OS
   - Validated for existence and writability before worktree operations (FR-015)

3. **Immutability** (FR-017):
   - Once created, `source_management_flow` cannot be changed
   - Attempts to modify result in error directing user to reinitialize
   - `worktree_folder` can technically be edited manually, but changing mode is blocked

### State Lifecycle

```
[No Config] 
    ↓
[specify init runs]
    ↓
[detect_git_environment() → suggests mode]
    ↓
[User confirms or overrides]
    ↓
[Config created with mode + folder (if worktree)]
    ↓
[Config exists - IMMUTABLE]
    ↓
[Option 1: Use normally] → Config unchanged
[Option 2: Mode change attempted] → Error: must reinitialize
[Option 3: Project deleted] → Config removed
```

### Example Configs

**Branch Mode** (standard Git workflow):
```json
{
  "version": "1.0",
  "source_management_flow": "branch"
}
```

**Worktree Mode** (isolated worktrees):
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

**None Mode** (no Git):
```json
{
  "version": "1.0",
  "source_management_flow": "none"
}
```

---

## Entity 2: GitEnvironment

**Purpose**: Represents the detected Git environment state (transient, not persisted).

**Lifecycle**: Created during `detect_git_environment()`, used to suggest mode, then discarded.

**Structure**:
```python
@dataclass
class GitEnvironment:
    has_git: bool         # Git command available and repo exists
    is_bare: bool         # Repository is bare (no working tree)
    is_worktree: bool     # Current directory is a registered worktree
```

### Fields

| Field | Type | Description | Detection Method |
|-------|------|-------------|------------------|
| `has_git` | boolean | Git is installed and current dir is in/can be a repo | `git rev-parse --is-inside-work-tree` (or init check) |
| `is_bare` | boolean | Repository has no working tree | `git rev-parse --is-bare-repository` |
| `is_worktree` | boolean | Current directory is a registered Git worktree | `git worktree list \| grep "$(pwd)"` |

### Derivation Logic

**Mode Suggestion Algorithm**:
```python
def suggest_mode(self) -> str:
    """
    Returns suggested source_management_flow based on environment.
    
    Decision tree:
    1. No Git → "none"
    2. Bare repo → "worktree" (bare repos are designed for worktrees)
    3. Currently in worktree → "worktree" (preserve existing setup)
    4. Standard Git repo → "branch" (default for normal repos)
    """
    if not self.has_git:
        return "none"
    if self.is_bare or self.is_worktree:
        return "worktree"
    return "branch"
```

### Valid State Combinations

| has_git | is_bare | is_worktree | Interpretation | Suggested Mode |
|---------|---------|-------------|----------------|----------------|
| False | False | False | No Git environment | none |
| True | False | False | Standard Git repository | branch |
| True | True | False | Bare repository | worktree |
| True | False | True | Inside a worktree | worktree |
| True | True | True | **INVALID** (bare repos can't be worktrees) | N/A - shouldn't occur |

**Note**: The combination `has_git=True, is_bare=True, is_worktree=True` is impossible because bare repositories don't have working trees and thus cannot themselves be worktrees.

### Detection Order

Detection must follow this sequence to avoid ambiguity:

1. **Check Git availability**: Run `git rev-parse --is-inside-work-tree`
   - Failure → `has_git=False`, skip remaining checks
   - Success → `has_git=True`, continue

2. **Check if bare** (only if has_git): Run `git rev-parse --is-bare-repository`
   - Output "true" → `is_bare=True`, set `is_worktree=False` (skip worktree check)
   - Output "false" → `is_bare=False`, continue

3. **Check if worktree** (only if has_git and not bare): Run `git worktree list | grep`
   - Match found → `is_worktree=True`
   - No match → `is_worktree=False`

---

## Entity 3: WorktreeMetadata

**Purpose**: Represents a Git worktree's metadata (transient, used during validation and creation).

**Lifecycle**: Created during worktree operations, validated, then discarded. Not persisted.

**Structure**:
```python
@dataclass
class WorktreeMetadata:
    path: Path              # Absolute path to worktree directory
    branch_name: str        # Associated Git branch (format: ###-feature-name)
    registered: bool        # Whether Git worktree list includes this path
```

### Fields

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `path` | Path | Full filesystem path to worktree | Must be absolute; must exist for registered worktrees |
| `branch_name` | str | Git branch checked out in worktree | Must match `###-feature-name` pattern |
| `registered` | bool | Git knows about this worktree | True if in `git worktree list` output |

### Validation Rules (FR-010)

**Name Alignment Check**:
```python
def validate_name_alignment(self) -> bool:
    """
    Verify worktree directory name matches branch name.
    
    Examples:
      path=/repo/worktrees/001-user-auth, branch=001-user-auth → Valid
      path=/repo/worktrees/001-user-auth, branch=002-other → Invalid
    """
    expected_dirname = self.branch_name
    actual_dirname = self.path.name
    return actual_dirname == expected_dirname
```

**Location Validation** (FR-008):
```python
def validate_location(self, configured_folder: Path) -> bool:
    """
    Verify worktree is in configured worktree_folder.
    
    Example:
      configured_folder=/repo/worktrees
      path=/repo/worktrees/001-feature → Valid
      path=/tmp/001-feature → Invalid
    """
    return self.path.parent.resolve() == configured_folder.resolve()
```

### State Transitions

```
[Feature creation requested]
    ↓
[Branch name generated: ###-feature-name]
    ↓
[WorktreeMetadata created]
    - path = worktree_folder / branch_name
    - branch_name = ###-feature-name
    - registered = False (not created yet)
    ↓
[Validation checks]
    - Name alignment: N/A (creating new)
    - Location: Verify path is under worktree_folder
    - Folder exists: FR-015 check
    ↓
[Git worktree add command]
    ↓
[registered = True]
    ↓
[WorktreeMetadata discarded]
```

### Validation Scenarios

**Scenario 1: Creating new worktree**
- `registered=False` initially
- Validate location is under `worktree_folder`
- Check parent folder exists and is writable (FR-015a, FR-015b)
- After `git worktree add`, verify `registered=True`

**Scenario 2: Validating existing worktree** (FR-010)
- `registered=True` (found in `git worktree list`)
- Validate name alignment: `path.name == branch_name`
- Validate location: `path.parent == worktree_folder`
- Emit warning if either validation fails

**Scenario 3: Mismatch detection**
- Directory `/repo/worktrees/001-feature` exists
- Current branch is `002-other-feature`
- `validate_name_alignment()` returns False
- System warns: "Worktree directory name (001-feature) doesn't match branch (002-other-feature)"

---

## Relationships Between Entities

```
ConfigFile (persisted)
    ↓ read during init/operations
    ↓
Mode Decision
    ↓
    ├─ branch mode → Create branches (existing behavior)
    ├─ worktree mode → Create WorktreeMetadata → Git worktree operations
    └─ none mode → Skip Git operations

GitEnvironment (transient)
    ↓ used once during init
    ↓
suggest_mode()
    ↓
ConfigFile (created with suggested/selected mode)
```

**Key Insight**: `GitEnvironment` is only used during initialization to suggest a mode. Once `ConfigFile` is created, the system reads the config (not the environment) to determine behavior.

---

## Data Validation Summary

| Validation Type | When Applied | Entity | Rule Reference |
|----------------|--------------|--------|----------------|
| Schema validation | Config load | ConfigFile | Must match JSON schema |
| Mode enum check | Config load | ConfigFile | Must be branch\|worktree\|none |
| Conditional field | Config load | ConfigFile | worktree_folder required iff mode=worktree |
| Path existence | Before worktree ops | ConfigFile.worktree_folder | FR-015 |
| Write permissions | Before worktree ops | ConfigFile.worktree_folder | FR-015b |
| Name alignment | Validation command | WorktreeMetadata | FR-010 |
| Location check | Worktree creation | WorktreeMetadata | FR-008 |
| Immutability | Config modification attempt | ConfigFile.source_management_flow | FR-017 |

---

## Storage Format

**ConfigFile** (`.specify/memory/config.json`):
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

**GitEnvironment** and **WorktreeMetadata**: Not stored (ephemeral Python objects).

---

## Migration Considerations

**Future schema versions**:
- Version field allows detecting old config formats
- If `version != "1.0"`, can prompt for migration or reject
- For now, only version 1.0 is supported

**Config not found**:
- Treated as "uninitialized" state
- User must run `specify init` to create config

**Corrupt config**:
- JSON parse error → Clear error message with file path and line number
- Schema validation error → Explain which field is invalid
- Suggest deleting `.specify/memory/config.json` and re-running init

---

## Testing Considerations (if tests were requested)

1. **ConfigFile validation**:
   - Valid configs for all three modes
   - Invalid: missing fields, wrong enum values, conditional field violations
   - Malformed JSON syntax

2. **GitEnvironment detection**:
   - All valid state combinations (4 scenarios)
   - Edge cases: detached HEAD, shallow clone, submodule
   - Error cases: Git not installed, not in repo

3. **WorktreeMetadata validation**:
   - Name alignment: matching vs. mismatched
   - Location validation: inside vs. outside worktree_folder
   - Registered check: in list vs. orphaned directory

---

This data model supports all 22 functional requirements and provides clear validation boundaries for implementation.
