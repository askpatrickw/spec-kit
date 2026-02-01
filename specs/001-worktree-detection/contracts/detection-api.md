# Git Detection API Contract

**Module**: `src/specify_cli/__init__.py`  
**Purpose**: Functions for detecting Git environment state  
**Version**: 1.0  
**Date**: 2026-01-31

---

## Function: `is_git_worktree()`

**Signature**:
```python
def is_git_worktree() -> bool
```

**Purpose**: Detects if the current directory is a registered Git worktree.

**Returns**:
- `True` if current directory appears in `git worktree list` output
- `False` otherwise (not a worktree, Git error, or not in repo)

**Implementation Strategy**:
```bash
# Equivalent Git command:
git worktree list | grep "$(pwd)"
```

**Performance Target**: <100ms

**Error Handling**:
- Git not installed → Return `False`
- Not in a Git repository → Return `False`
- Command execution error → Return `False`
- Never raises exceptions (defensive design)

**Cross-Platform Notes**:
- Uses `Path.resolve()` for consistent path comparison across OS
- Works on Windows (with Git Bash), Linux, macOS

**Example Usage**:
```python
if is_git_worktree():
    console.print("[cyan]Worktree mode active[/cyan]")
else:
    console.print("[cyan]Standard repository[/cyan]")
```

**Test Scenarios** (if tests were requested):
1. Inside a registered worktree → Returns `True`
2. In standard Git repo (not worktree) → Returns `False`
3. Git not installed → Returns `False`
4. Not in any Git repo → Returns `False`

**Related Requirements**: FR-004, FR-014

---

## Function: `is_bare_repo()`

**Signature**:
```python
def is_bare_repo() -> bool
```

**Purpose**: Detects if the current Git repository is bare (has no working tree).

**Returns**:
- `True` if `git rev-parse --is-bare-repository` outputs "true"
- `False` otherwise (not bare, Git error, or not in repo)

**Implementation Strategy**:
```bash
# Equivalent Git command:
git rev-parse --is-bare-repository
# Output: "true" or "false"
```

**Performance Target**: <50ms

**Error Handling**:
- Git not installed → Return `False`
- Not in a Git repository → Return `False`
- Command execution error → Return `False`
- Never raises exceptions (defensive design)

**Example Usage**:
```python
if is_bare_repo():
    console.print("[yellow]Bare repository detected[/yellow]")
    console.print("Worktree mode recommended for bare repos")
```

**Test Scenarios** (if tests were requested):
1. Inside bare repository → Returns `True`
2. Standard Git repo with working tree → Returns `False`
3. Inside a worktree → Returns `False`
4. Git not installed → Returns `False`

**Related Requirements**: FR-005

---

## Function: `detect_git_environment()`

**Signature**:
```python
@dataclass
class GitEnvironment:
    has_git: bool
    is_bare: bool
    is_worktree: bool
    
    def suggest_mode(self) -> str:
        """Returns suggested source_management_flow value."""
        ...

def detect_git_environment() -> GitEnvironment
```

**Purpose**: Combines all Git detection checks into a single classification of the current environment.

**Returns**: `GitEnvironment` object with three boolean fields:
- `has_git`: Git is installed and current directory is (or can be) a Git repo
- `is_bare`: Repository is bare
- `is_worktree`: Current directory is a registered worktree

**Detection Algorithm**:
```python
1. Check if Git is available and directory is/can be a repo
   └─ No → GitEnvironment(has_git=False, is_bare=False, is_worktree=False)
   
2. If has_git, check if bare: is_bare_repo()
   └─ Yes → GitEnvironment(has_git=True, is_bare=True, is_worktree=False)
            (Skip worktree check - bare repos can't be worktrees)
   
3. If has_git and not bare, check worktree: is_git_worktree()
   └─ GitEnvironment(has_git=True, is_bare=False, is_worktree=<result>)
```

**Performance Target**: <200ms (combines all checks)

**Mode Suggestion Logic**:
```python
GitEnvironment.suggest_mode() -> str:
    if not self.has_git:
        return "none"
    if self.is_bare or self.is_worktree:
        return "worktree"
    return "branch"
```

**Example Usage**:
```python
env = detect_git_environment()
suggested = env.suggest_mode()

console.print(f"[cyan]Detected environment:[/cyan]")
console.print(f"  Git available: {env.has_git}")
console.print(f"  Bare repo: {env.is_bare}")
console.print(f"  In worktree: {env.is_worktree}")
console.print(f"[cyan]Suggested mode:[/cyan] [bold]{suggested}[/bold]")
```

**Test Scenarios** (if tests were requested):

| Scenario | has_git | is_bare | is_worktree | suggest_mode() |
|----------|---------|---------|-------------|----------------|
| No Git installed | False | False | False | "none" |
| Standard Git repo | True | False | False | "branch" |
| Bare repository | True | True | False | "worktree" |
| Inside worktree | True | False | True | "worktree" |

**Related Requirements**: FR-001a, FR-001b, FR-006

---

## Integration Example

**During `specify init` command**:

```python
def init(project_name: str, ...):
    # ... existing setup code ...
    
    # NEW: Detect Git environment
    env = detect_git_environment()
    suggested_mode = env.suggest_mode()
    
    # NEW: Prompt user (auto-detect with confirmation)
    console.print(f"\n[cyan]Detected Git environment:[/cyan]")
    console.print(f"  Suggested mode: [bold]{suggested_mode}[/bold]")
    
    use_suggested = Confirm.ask(
        f"Use '{suggested_mode}' mode?",
        default=True
    )
    
    if use_suggested:
        selected_mode = suggested_mode
    else:
        selected_mode = Prompt.ask(
            "Select mode",
            choices=["branch", "worktree", "none"],
            default=suggested_mode
        )
    
    # NEW: Create config
    config = {
        "version": "1.0",
        "source_management_flow": selected_mode
    }
    
    if selected_mode == "worktree":
        config["worktree_folder"] = "./worktrees"
        
        # FR-016: Warn if not bare
        if not env.is_bare:
            console.print("[yellow]⚠ Warning:[/yellow] Worktree mode works best with bare repositories.")
            proceed = Confirm.ask("Proceed anyway?", default=True)
            if not proceed:
                # Let user re-select mode
                ...
    
    save_config(config)
    
    # ... continue with existing init logic ...
```

---

## Error Handling Contract

All detection functions follow this contract:

1. **Never raise exceptions** - Always return bool or object
2. **Fail safe** - Unknown states default to False
3. **Silent failures** - Log errors internally if needed, but don't break flow
4. **Fast failures** - Timeout after 5 seconds for subprocess calls

This defensive design ensures `specify init` never crashes due to Git environment issues.

---

## Performance Budget

| Function | Target | Max Acceptable |
|----------|--------|----------------|
| `is_git_worktree()` | <100ms | 500ms |
| `is_bare_repo()` | <50ms | 200ms |
| `detect_git_environment()` | <200ms | 1000ms |

**Rationale**: Git commands are fast; these targets allow for subprocess overhead while maintaining responsive CLI experience.

---

## Dependencies

- **Python stdlib**: `subprocess`, `pathlib`
- **Git**: Version 2.13+ (for stable worktree support)
- **Existing functions**: `is_git_repo()` (can be reused/refactored)

---

This API contract supports FR-004, FR-005, FR-001a, FR-001b, FR-006, FR-014, and provides the foundation for mode-aware Git operations.
