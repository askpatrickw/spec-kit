# Configuration Management API Contract

**Module**: `src/specify_cli/__init__.py`  
**Purpose**: Functions for loading, validating, and saving source management configuration  
**Version**: 1.0  
**Date**: 2026-01-31

---

## Function: `load_config()`

**Signature**:
```python
def load_config(path: Path | None = None) -> dict | None
```

**Purpose**: Loads and validates the source management configuration file.

**Parameters**:
- `path` (optional): Override config file location. Defaults to `.specify/memory/config.json` relative to current directory.

**Returns**:
- `dict`: Parsed and validated configuration if file exists and is valid
- `None`: If config file doesn't exist (not an error - project may not be initialized)

**Raises**:
- `ValueError`: If config file exists but is invalid (malformed JSON, schema violation)
- `FileNotFoundError`: Never raised (returns None instead for missing files)

**Validation Steps**:
1. Check if file exists → Return `None` if not
2. Parse JSON → Raise `ValueError` if syntax error
3. Validate schema:
   - `version` field exists and equals "1.0"
   - `source_management_flow` exists and is valid enum
   - `worktree_folder` present iff flow=="worktree"
4. Return validated dict

**Performance Target**: <50ms

**Example Usage**:
```python
# Check if project is initialized
config = load_config()
if config is None:
    console.print("[yellow]Project not initialized. Run 'specify init'[/yellow]")
    return

# Get mode
mode = config["source_management_flow"]
console.print(f"[cyan]Current mode:[/cyan] {mode}")

# Get worktree folder if applicable
if mode == "worktree":
    worktree_folder = Path(config["worktree_folder"])
    console.print(f"[cyan]Worktrees location:[/cyan] {worktree_folder}")
```

**Error Messages**:
```python
# Missing required field
ValueError("Config missing required field 'version' in .specify/memory/config.json")

# Invalid enum value
ValueError("Invalid source_management_flow 'invalid' (must be branch/worktree/none)")

# Conditional field violation
ValueError("worktree_folder required when source_management_flow=worktree")

# JSON syntax error
ValueError("Invalid JSON syntax in .specify/memory/config.json at line 3: ...")
```

**Test Scenarios** (if tests were requested):
1. Valid config (all three modes) → Returns dict
2. File doesn't exist → Returns None
3. Malformed JSON → Raises ValueError with line number
4. Missing required field → Raises ValueError
5. Invalid enum value → Raises ValueError
6. worktree mode without worktree_folder → Raises ValueError
7. branch mode with worktree_folder → Allowed but ignored (additionalProperties: false rejects unknown fields)

**Related Requirements**: FR-001, FR-002, FR-003, FR-006

---

## Function: `save_config()`

**Signature**:
```python
def save_config(config: dict, path: Path | None = None) -> None
```

**Purpose**: Validates and writes configuration to disk.

**Parameters**:
- `config`: Configuration dictionary matching schema (see `config-schema.json`)
- `path` (optional): Override config file location. Defaults to `.specify/memory/config.json`

**Returns**: None

**Raises**:
- `ValueError`: If config doesn't match schema
- `OSError`: If directory doesn't exist or lacks write permissions

**Validation**:
Same validation rules as `load_config()` applied before writing.

**Behavior**:
1. Validate config dict against schema
2. Create `.specify/memory/` directory if it doesn't exist
3. Write JSON with 2-space indentation for readability
4. Set file permissions to 0644 (readable by all, writable by owner)

**Performance Target**: <50ms

**JSON Output Format**:
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

Note: JSON format used for easy parsing by both Python and shell scripts. Documentation and warnings are provided in quickstart.md and command output (cannot embed comments in JSON).

**Example Usage**:
```python
# During specify init
config = {
    "version": "1.0",
    "source_management_flow": selected_mode
}

if selected_mode == "worktree":
    config["worktree_folder"] = "./worktrees"

try:
    save_config(config)
    console.print("[green]✓[/green] Configuration saved")
except ValueError as e:
    console.print(f"[red]Error:[/red] Invalid configuration: {e}")
    sys.exit(1)
except OSError as e:
    console.print(f"[red]Error:[/red] Cannot write config: {e}")
    sys.exit(1)
```

**Error Messages**:
```python
# Schema validation error
ValueError("Invalid source_management_flow 'invalid' (must be branch/worktree/none)")

# Directory creation failure
OSError("Cannot create .specify/memory/: Permission denied")

# Write permission error
OSError("Cannot write to .specify/memory/config.json: Permission denied")
```

**Test Scenarios** (if tests were requested):
1. Valid config (all modes) → File created successfully
2. Invalid config → Raises ValueError before writing
3. Directory doesn't exist → Creates directory, then writes file
4. Directory not writable → Raises OSError
5. Existing file → Overwrites (no backup - config is ephemeral user state)

**Related Requirements**: FR-001, FR-002, FR-003

---

## Function: `validate_config_immutable()`

**Signature**:
```python
def validate_config_immutable(current_config: dict) -> None
```

**Purpose**: Checks if source_management_flow has been modified and blocks changes (FR-017).

**Parameters**:
- `current_config`: Existing config loaded from disk

**Returns**: None (success)

**Raises**:
- `RuntimeError`: If mode change is detected

**Usage Context**:
This function would be called if/when we add a `specify config` command to modify settings. For the initial implementation, we're not supporting config modification at all, so this is primarily a design placeholder.

**Example (Future Enhancement)**:
```python
def update_config(new_settings: dict):
    """Hypothetical future command to update config."""
    current = load_config()
    if current is None:
        raise ValueError("No config to update. Run 'specify init' first.")
    
    # FR-017: Block mode changes
    if "source_management_flow" in new_settings:
        if new_settings["source_management_flow"] != current["source_management_flow"]:
            raise RuntimeError(
                "Cannot change source_management_flow after initialization.\n"
                "To switch modes, delete .specify/memory/config.json and run 'specify init' again."
            )
    
    # Merge and save
    updated = {**current, **new_settings}
    save_config(updated)
```

**Related Requirements**: FR-017

---

## Helper Function: `get_config_path()`

**Signature**:
```python
def get_config_path(override: Path | None = None) -> Path
```

**Purpose**: Returns absolute path to config file with optional override.

**Parameters**:
- `override` (optional): Custom path to config file

**Returns**: Absolute `Path` object

**Behavior**:
```python
if override:
    return override.resolve()
else:
    return (Path.cwd() / ".specify" / "memory" / "config.json").resolve()
```

**Example Usage**:
```python
config_path = get_config_path()
console.print(f"[dim]Config location: {config_path}[/dim]")
```

---

## Integration with Git Detection

**Typical initialization flow**:

```python
def init_command():
    # 1. Detect Git environment
    env = detect_git_environment()
    suggested_mode = env.suggest_mode()
    
    # 2. Prompt user with auto-detect suggestion
    selected_mode = prompt_for_mode(suggested_mode, env)
    
    # 3. Create config
    config = create_config(selected_mode)
    
    # 4. FR-016: Warn if worktree mode in non-bare repo
    if selected_mode == "worktree" and not env.is_bare:
        warn_non_bare_worktree()
        if not confirm_proceed():
            # Restart mode selection
            return init_command()
    
    # 5. Save config
    save_config(config)
    
    # 6. Continue with template download, etc.
    ...
```

**Typical command execution flow**:

```python
def create_feature_command(feature_name: str):
    # 1. Load config
    config = load_config()
    if config is None:
        console.print("[red]Error:[/red] Project not initialized")
        sys.exit(1)
    
    # 2. Get mode
    mode = config["source_management_flow"]
    
    # 3. Branch based on mode
    if mode == "none":
        console.print("[yellow]Git operations disabled in 'none' mode[/yellow]")
        return
    elif mode == "worktree":
        create_worktree(feature_name, config["worktree_folder"])
    else:  # branch mode
        create_branch(feature_name)
```

---

## Configuration Schema Reference

See `config-schema.json` for JSON Schema definition. Summary:

```json
{
  "version": "1.0",
  "source_management_flow": "branch",
  "worktree_folder": "./worktrees"
}
```

- `version`: Required, must be "1.0"
- `source_management_flow`: Required, enum: "branch"|"worktree"|"none"
- `worktree_folder`: Conditional - required iff flow="worktree"

---

## Error Handling Philosophy

1. **Missing config = uninitialized** (not an error): Return `None` from `load_config()`
2. **Invalid config = corruption** (error): Raise `ValueError` with clear fix instructions
3. **Permission errors = environmental** (error): Raise `OSError` with actionable message
4. **Mode changes = policy violation** (error): Raise `RuntimeError` explaining immutability

---

## Performance Budget

| Function | Target | Max Acceptable |
|----------|--------|----------------|
| `load_config()` | <50ms | 200ms |
| `save_config()` | <50ms | 200ms |
| `get_config_path()` | <1ms | 10ms |

**Rationale**: File I/O for small JSON files (<1KB) is fast; targets account for filesystem overhead.

---

## Dependencies

- **Python stdlib**: `pathlib`, `os`, `json` (built-in)
- **Rich**: For error message formatting (existing dependency)

**No new external dependencies** - `json` module is part of Python standard library.

---

## Future Enhancements (Out of Scope for Initial Implementation)

1. **Config migration**: Support upgrading from version 1.0 to future versions
2. **Config editing**: `specify config set worktree_folder /new/path` (currently blocked)
3. **Config validation command**: `specify config validate` to check for corruption
4. **Environment variable overrides**: `SPECIFY_MODE=worktree` to override config
5. **Project-wide config**: `.specify/project-config.json` (version-controlled, shared settings)

---

This API contract supports FR-001, FR-002, FR-003, FR-003a, FR-006, FR-015, FR-016, FR-017, and provides the foundation for mode-aware command behavior.
