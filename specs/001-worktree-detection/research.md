# Phase 0: Research & Decisions

**Feature**: Git Worktree-Aware Workflows  
**Branch**: 001-worktree-detection  
**Date**: 2026-01-31

This document resolves all "NEEDS CLARIFICATION" items from the Technical Context and documents technology decisions for implementation.

---

## Decision 1: JSON Config Management Best Practices

**Chosen Approach**: JSON with Python's built-in `json` module and native shell parsing

**Rationale**:
- **No new dependencies**: Python's `json` module is built-in; no external libraries needed
- **Shell script compatibility**: Both bash and PowerShell can parse JSON without external tools
  - Bash: Simple grep/sed patterns or `jq` (if available)
  - PowerShell: Built-in `ConvertFrom-Json` cmdlet
- **Cross-platform**: Works identically on Linux, macOS, Windows
- **Performance**: Adequate for small config files (<1KB) loaded once during operations
- **Still readable**: With proper formatting, JSON is clear enough for manual editing
- **Maintainability**: Standard format with ubiquitous tooling support

**Alternatives Considered**:
1. **YAML with PyYAML** - Rejected because:
   - Adds external dependency (PyYAML)
   - Shell scripts can't easily parse YAML (would need `yq` or inline Python)
   - Adds complexity for minimal readability gain (config has only 3 fields)
2. **YAML with inline Python parsing** - Rejected because:
   - Slower (subprocess overhead in shell scripts)
   - More complex shell script logic
   - Fragile across different Python environments
3. **TOML** - Rejected because:
   - Requires external library in Python (`tomli` or `toml`)
   - Poor shell script parsing support
4. **INI format** - Rejected because:
   - No native Python stdlib support for structured data
   - Harder to express nested config in future

**Implementation Notes**:

**Python parsing**:
```python
import json
from pathlib import Path

def load_config(path: Path = None) -> dict | None:
    config_path = path or Path.cwd() / ".specify" / "memory" / "config.json"
    if not config_path.exists():
        return None
    
    with open(config_path) as f:
        config = json.load(f)
    
    # Validate
    required_keys = {"source_management_flow", "version"}
    if not required_keys.issubset(config.keys()):
        raise ValueError(f"Missing required keys: {required_keys - config.keys()}")
    
    if config["source_management_flow"] not in {"branch", "worktree", "none"}:
        raise ValueError(f"Invalid source_management_flow: {config['source_management_flow']}")
    
    return config

def save_config(config: dict, path: Path = None) -> None:
    config_path = path or Path.cwd() / ".specify" / "memory" / "config.json"
    config_path.parent.mkdir(parents=True, exist_ok=True)
    
    with open(config_path, 'w') as f:
        json.dump(config, f, indent=2)
```

**Bash parsing** (simple, no jq needed):
```bash
CONFIG_FILE=".specify/memory/config.json"

# Extract source_management_flow
MODE=$(grep -o '"source_management_flow"[[:space:]]*:[[:space:]]*"[^"]*"' "$CONFIG_FILE" | sed 's/.*"\([^"]*\)".*/\1/')

# Extract worktree_folder
WORKTREE_FOLDER=$(grep -o '"worktree_folder"[[:space:]]*:[[:space:]]*"[^"]*"' "$CONFIG_FILE" | sed 's/.*"\([^"]*\)".*/\1/')

# Fallback to defaults if not found
MODE=${MODE:-branch}
WORKTREE_FOLDER=${WORKTREE_FOLDER:-./worktrees}
```

**PowerShell parsing**:
```powershell
$configPath = ".specify/memory/config.json"
if (Test-Path $configPath) {
    $config = Get-Content $configPath | ConvertFrom-Json
    $mode = $config.source_management_flow
    $worktreeFolder = $config.worktree_folder
} else {
    $mode = "branch"  # Default
}
```

**JSON format example**:
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

---

## Decision 2: Git Worktree Detection Methods

**Chosen Approach**: `git worktree list | grep "$(pwd)"` method

**Rationale**:
- **Reliability**: Works across all Git versions 2.13+ (when worktree became stable)
- **Simplicity**: Single command with straightforward output parsing
- **Cross-platform**: Works identically on Linux, macOS, and Windows (with Git Bash)
- **Accuracy**: Git's own worktree list is the authoritative source of registered worktrees
- **Maintainability**: Less fragile than checking internal Git directory structure

**Alternatives Considered**:
1. **`git rev-parse --git-common-dir` + file existence check** - Rejected because:
   - More complex (requires multiple commands and path manipulation)
   - Relies on internal Git directory structure (may change across versions)
   - Harder to debug when issues occur
2. **Checking `.git/worktrees/` directory** - Rejected because:
   - Requires platform-specific path handling
   - Tightly coupled to Git's internal implementation
   - Would break if Git changes directory structure

**Implementation Notes**:
```python
def is_git_worktree() -> bool:
    """Check if current directory is a registered Git worktree."""
    try:
        # Get list of all worktrees
        result = subprocess.run(
            ["git", "worktree", "list"],
            check=True,
            capture_output=True,
            text=True,
            cwd=Path.cwd()
        )
        
        # Check if current directory appears in the list
        current_path = str(Path.cwd().resolve())
        for line in result.stdout.splitlines():
            # Format: /path/to/worktree [branch_name]
            worktree_path = line.split()[0] if line.strip() else ""
            if Path(worktree_path).resolve() == Path(current_path):
                return True
        
        return False
    except (subprocess.CalledProcessError, FileNotFoundError, IndexError):
        return False
```

**Minimum Git Version**: 2.13 (July 2017) - first version with stable worktree support

---

## Decision 3: Config File Location and Format Strategy

**Chosen Approach**: `.specify/memory/config.json` (JSON format inside memory subdirectory)

**Rationale**:
- **Consistency**: Follows existing Spec-Kit pattern of using `.specify/memory/` for user-specific state
- **Gitignore-friendly**: `.specify/memory/` is already in `.gitignore` templates, preventing accidental commits
- **Separation of concerns**: Distinguishes user config (memory/) from template files (.specify/ root)
- **Format choice (JSON)**: Shell scripts need to read this config; JSON is easily parseable in bash/PowerShell without external tools
- **Future-proof**: Leaves `.specify/config.json` available for project-wide (version-controlled) settings if needed later

**Alternatives Considered**:
1. **`.specify/config.json`** - Rejected because:
   - Would be version-controlled (not in gitignore)
   - Could conflict with template updates
   - Mixes user state with template structure
2. **`~/.config/specify/config.json`** (user home directory) - Rejected because:
   - Global config doesn't fit this use case (each project has its own mode)
   - Harder for users to discover and modify
   - Cross-platform path complexity (~/.config not standard on Windows)
3. **`.specifyrc.json`** (repository root) - Rejected because:
   - Non-standard location (users won't expect it)
   - Pollutes repository root
   - Not consistent with existing .specify/ structure
4. **YAML format** - Rejected because:
   - Shell scripts can't parse YAML without external tools (`yq`)
   - Would require PyYAML dependency in Python
   - JSON is adequate for small config files

**Implementation Notes**:
- Create `.specify/memory/` directory during initialization if it doesn't exist
- Use JSON with 2-space indentation for readability
- Document config location prominently in quickstart.md
- Add config path to error messages when config is missing or invalid
- Ensure template's `.gitignore` includes `.specify/memory/` (verify existing templates)
- Shell scripts use simple grep/sed for JSON parsing (no jq dependency required)

---

## Decision 4: Interactive Prompt Library Selection

**Chosen Approach**: Rich's `Prompt` class with custom confirmation logic

**Rationale**:
- **Already available**: Rich is an existing dependency (no new dependencies needed)
- **Consistent UX**: Matches existing Spec-Kit prompt styling
- **Flexibility**: Rich provides full control over prompt formatting and validation
- **Integration**: Works well with existing console output and progress indicators
- **Cross-platform**: Rich handles platform quirks (Windows console, ANSI escapes, etc.)

**Alternatives Considered**:
1. **Typer's built-in `typer.confirm()` and `typer.prompt()`** - Rejected because:
   - Less control over formatting and appearance
   - Harder to customize for "auto-detect with confirmation" pattern
   - Minimal styling options compared to Rich
2. **readchar directly** - Rejected because:
   - Too low-level for our needs
   - Would require reimplementing prompt logic Rich already provides
   - Only needed for special cases (y/n single keystroke)
3. **questionary library** - Rejected because:
   - New dependency not justified for simple yes/no prompts
   - Rich provides equivalent functionality

**Implementation Notes**:
```python
from rich.prompt import Confirm, Prompt

def prompt_for_mode(suggested_mode: str) -> str:
    """Prompt user to confirm or override detected mode."""
    console.print(f"\n[cyan]Detected Git environment suggests:[/cyan] [bold]{suggested_mode}[/bold]")
    
    # Ask if user wants to use suggested mode
    use_suggested = Confirm.ask(
        f"Use '{suggested_mode}' mode?",
        default=True
    )
    
    if use_suggested:
        return suggested_mode
    
    # Let user select different mode
    console.print("\n[cyan]Available modes:[/cyan]")
    console.print("  [bold]branch[/bold]   - Traditional branch-based workflow")
    console.print("  [bold]worktree[/bold] - Isolated worktrees per feature")
    console.print("  [bold]none[/bold]     - No Git integration")
    
    mode = Prompt.ask(
        "Select mode",
        choices=["branch", "worktree", "none"],
        default=suggested_mode
    )
    
    return mode
```

---

## Decision 5: Bare Repository Detection Edge Cases

**Chosen Approach**: Hierarchical detection with explicit decision tree

**Rationale**:
- **Clarity**: Explicit order of checks prevents ambiguous states
- **Correctness**: Handles all Git environment types (bare, worktree, standard, none)
- **Debuggability**: Clear logic flow makes troubleshooting easier
- **User guidance**: Unambiguous classification enables appropriate mode suggestions

**Detection Decision Tree**:
```
1. Check if Git is installed
   └─ No → GitEnvironment(has_git=False, is_bare=False, is_worktree=False)
            Suggest: "none"

2. Check if in Git repository (git rev-parse --is-inside-work-tree)
   └─ No → GitEnvironment(has_git=True, is_bare=False, is_worktree=False)
            Suggest: "branch" (will initialize repo)

3. Check if current repo is bare (git rev-parse --is-bare-repository)
   └─ Yes → GitEnvironment(has_git=True, is_bare=True, is_worktree=False)
             Suggest: "worktree"

4. Check if current directory is a worktree (git worktree list | grep)
   └─ Yes → GitEnvironment(has_git=True, is_bare=False, is_worktree=True)
             Suggest: "worktree"
   └─ No → GitEnvironment(has_git=True, is_bare=False, is_worktree=False)
            Suggest: "branch"
```

**Edge Case Handling**:
- **Bare repo in worktree**: Cannot happen (bare repos don't have working trees)
- **Detached HEAD**: Treat as standard repo (doesn't affect worktree detection)
- **Shallow clone**: Treat as standard repo (worktree support works)
- **Submodule**: Treat as standard repo (each submodule has its own .git)

**Alternatives Considered**:
1. **Single check using git worktree list** - Rejected because:
   - Doesn't distinguish bare repos from non-worktree standard repos
   - Misses opportunity to suggest worktree mode for bare repos
2. **File system checks only** - Rejected because:
   - Fragile (relies on internal Git structure)
   - Doesn't work across Git versions
   - Can't detect shallow clones or special repo configurations

**Implementation Notes**:
```python
from dataclasses import dataclass

@dataclass
class GitEnvironment:
    has_git: bool
    is_bare: bool
    is_worktree: bool
    
    def suggest_mode(self) -> str:
        """Suggest source management mode based on environment."""
        if not self.has_git:
            return "none"
        if self.is_bare or self.is_worktree:
            return "worktree"
        return "branch"

def detect_git_environment() -> GitEnvironment:
    """Detect current Git environment and classify it."""
    # Check 1: Git installed and in a repo?
    try:
        subprocess.run(
            ["git", "rev-parse", "--is-inside-work-tree"],
            check=True,
            capture_output=True,
            cwd=Path.cwd()
        )
        has_git = True
    except (subprocess.CalledProcessError, FileNotFoundError):
        return GitEnvironment(has_git=False, is_bare=False, is_worktree=False)
    
    # Check 2: Is it a bare repository?
    is_bare = is_bare_repo()
    
    # Check 3: Is current directory a worktree?
    is_worktree = is_git_worktree() if not is_bare else False
    
    return GitEnvironment(has_git=has_git, is_bare=is_bare, is_worktree=is_worktree)
```

---

## Summary of Decisions

| Decision Area | Choice | Key Rationale |
|--------------|--------|---------------|
| Config Format | JSON (built-in `json` module) | No dependencies, shell-script parseable |
| Worktree Detection | `git worktree list` | Reliable, cross-version compatible |
| Config Location | `.specify/memory/config.json` | Follows existing pattern, gitignored |
| Prompt Library | Rich Prompt | Already available, consistent UX |
| Detection Logic | Hierarchical decision tree | Clear, unambiguous, debuggable |

All research complete. Ready for Phase 1: Design & Contracts.
