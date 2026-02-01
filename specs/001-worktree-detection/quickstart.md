# Quickstart: Git Worktree-Aware Workflows

**Feature**: 001-worktree-detection  
**Audience**: Spec-Kit users wanting to understand and use worktree mode  
**Last Updated**: 2026-01-31

---

## Overview

Spec-Kit now supports three source management modes:

- **branch**: Traditional Git branch workflow (default for most projects)
- **worktree**: Isolated working directories per feature (recommended for bare repositories)
- **none**: No Git integration (for non-Git projects)

The mode is auto-detected during initialization and saved in `.specify/memory/config.json`.

---

## For New Projects

### Quick Start

```bash
# Initialize a new project
specify init my-project
cd my-project

# Spec-Kit auto-detects your Git environment and suggests a mode
# Just press Enter to accept the suggestion, or choose a different mode
```

### What Gets Auto-Detected?

| Your Setup | Detection Result | Suggested Mode |
|------------|------------------|----------------|
| No Git installed | "No Git environment" | **none** |
| Standard Git repo | "Standard repository" | **branch** |
| Bare repository | "Bare repository" | **worktree** |
| Inside a worktree | "Worktree environment" | **worktree** |

### Manual Mode Selection

If the auto-detected mode isn't what you want:

```bash
# During initialization, when prompted:
Detected Git environment suggests: branch
Use 'branch' mode? (Y/n): n

Available modes:
  branch   - Traditional branch-based workflow
  worktree - Isolated worktrees per feature
  none     - No Git integration

Select mode: worktree
```

---

## Checking Your Current Mode

```bash
# View your configuration
cat .specify/memory/config.json
```

Example output:
```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}
```

---

## Worktree Mode Setup

### Prerequisites for Worktree Mode

Worktree mode works best with **bare repositories**. Here's how to set one up:

```bash
# Create a bare repository
mkdir my-project.git
cd my-project.git
git init --bare

# Create worktrees directory (Spec-Kit will use this by default)
mkdir worktrees

# Now initialize Spec-Kit
specify init
# Auto-detects bare repo → suggests worktree mode → creates config
```

### Using Worktree Mode in Non-Bare Repos

You CAN use worktree mode in non-bare repos, but you'll get a warning:

```
⚠ Warning: Worktree mode works best with bare repositories.
Proceeding may cause confusion if you work in both the main repo
and worktrees simultaneously.

Proceed anyway? (Y/n):
```

**Recommendation**: Convert to bare repo first, or use branch mode instead.

---

## Creating Features in Different Modes

### Branch Mode (Default)

```bash
# Create a new feature
/speckit.specify "Add user authentication"

# System creates:
# - New branch: 001-user-auth
# - Spec directory: specs/001-user-auth/
# - Switches to the branch
```

### Worktree Mode

```bash
# Create a new feature
/speckit.specify "Add user authentication"

# System creates:
# - New worktree: <repo>/worktrees/001-user-auth/
# - New branch: 001-user-auth (checked out in the worktree)
# - Spec directory: specs/001-user-auth/ (in worktree)
# - You're now IN the worktree directory
```

### None Mode

```bash
# Create a new feature
/speckit.specify "Add user authentication"

# System creates:
# - Spec directory: specs/001-user-auth/
# - NO Git operations (no branch, no commits)
```

---

## Working with Worktrees

### Switching Between Worktrees

```bash
# List all worktrees
git worktree list

# Output example:
# /project.git                     (bare)
# /project.git/worktrees/001-auth  [001-auth]
# /project.git/worktrees/002-ui    [002-ui]

# Navigate to a different worktree
cd /project.git/worktrees/002-ui
```

### Worktree Directory Structure

```
my-project.git/              # Bare repository
├── worktrees/               # All feature worktrees (configured in config.json)
│   ├── 001-user-auth/       # Feature 001 worktree
│   │   ├── specs/           # Spec for feature 001
│   │   ├── src/             # Source code
│   │   └── .specify/        # Spec-Kit files
│   │
│   └── 002-dashboard/       # Feature 002 worktree
│       └── ...
│
├── refs/                    # Git internals
├── objects/                 # Git internals
└── .specify/                # Shared Spec-Kit config (optional)
    └── memory/
        └── config.json      # Source management mode
```

### Removing a Worktree

```bash
# When done with a feature
cd /project.git  # Return to bare repo
git worktree remove worktrees/001-user-auth

# Or force remove if there are uncommitted changes
git worktree remove --force worktrees/001-user-auth
```

---

## Understanding Mode Indicators

When you run Spec-Kit commands, you'll see mode indicators:

```bash
$ specify check
[Worktree mode active]
✓ Git worktree: valid
✓ Branch name: matches worktree directory
...

$ specify check  # In branch mode
[Branch mode active]
✓ Git branch: 001-user-auth
...

$ specify check  # In none mode
[No Git mode - limited functionality]
✓ Spec directory: exists
...
```

---

## Common Operations

### Checking Worktree/Branch Alignment

In worktree mode, Spec-Kit validates that your worktree directory name matches the branch:

```bash
# Valid: Directory and branch match
/project.git/worktrees/001-user-auth → branch: 001-user-auth  ✓

# Invalid: Mismatch detected
/project.git/worktrees/001-user-auth → branch: 002-other      ⚠
Warning: Worktree directory name (001-user-auth) doesn't match branch (002-other)
```

**How to fix**: Check out the correct branch in the worktree:
```bash
git checkout 001-user-auth
```

### Changing Worktree Folder Location

Edit `.specify/memory/config.json`:

```json
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "/custom/path/to/worktrees"
}
```

**Note**: This only affects NEW worktrees. Existing worktrees aren't moved.

---

## Troubleshooting

### "Config file not found"

**Problem**: `.specify/memory/config.json` doesn't exist.

**Solution**: Run `specify init` to create the config.

### "Cannot create worktree folder: Permission denied"

**Problem**: Spec-Kit can't create the worktrees directory.

**Solution**:
```bash
# Create the directory manually with correct permissions
mkdir -p ./worktrees
chmod 755 ./worktrees
```

### "Worktree folder does not exist"

**Problem**: When creating a new worktree, the configured folder doesn't exist.

**What happens**: Spec-Kit will prompt you:
```
The configured worktree folder does not exist: ./worktrees
Create this directory? (y/N):
```

**Options**:
- Type `y` to let Spec-Kit create the folder
- Type `n` (or press Enter for default) to cancel the operation

**Default behavior**: Defaults to `N` (No) for safety - won't create directories without explicit permission

**Exact prompt specification** (for FR-014a):
- **Prompt text**: `"The configured worktree folder does not exist: {path}\nCreate this directory? (y/N): "`
- **Default**: `N` (No) - pressing Enter without input cancels
- **Valid responses**: `y`, `Y`, `yes`, `Yes` → creates directory; anything else → cancels
- **On timeout/EOF**: Treated as `N` (cancels operation)

### "Worktree mode selected but repository is not bare"

**Problem**: You selected worktree mode in a non-bare repository.

**Options**:
1. **Convert to bare repo** (recommended):
   ```bash
   # Backup your work first!
   cd /path/to/project
   git clone --bare . ../project.git
   cd ../project.git
   specify init  # Will suggest worktree mode
   ```

2. **Use branch mode instead**:
   ```bash
   # Delete config and reinitialize
   rm .specify/memory/config.json
   specify init
   # Select "branch" when prompted
   ```

3. **Proceed anyway** (not recommended):
   - Accept the warning during init
   - Be careful not to work in both the main repo and worktrees

### "Cannot change source_management_flow"

**Problem**: You edited config.json to change the mode.

**Why blocked**: Mode switching requires complex migration (converting branches to worktrees or vice versa), which is not supported in the initial implementation.

**Solution**:
```bash
# Option 1: Reinitialize (loses existing feature branches)
rm -rf .specify/
specify init

# Option 2: Manually migrate (advanced)
# 1. Create new bare repo with desired mode
# 2. Copy specs/ directory
# 3. Recreate branches/worktrees as needed
```

### "Worktree already exists for branch"

**Problem**: Git prevents creating multiple worktrees for the same branch.

**Solution**: Remove the old worktree first:
```bash
git worktree list  # Find the existing worktree
git worktree remove <path>
```

---

## Best Practices

### When to Use Worktree Mode

✅ **Use worktree mode when**:
- Working on multiple features simultaneously
- Using a bare repository
- Want isolated environments per feature
- Working on long-running features that shouldn't block each other

❌ **Avoid worktree mode when**:
- Just starting with Git (stick to branch mode - it's simpler)
- Project has frequent cross-feature changes (switching worktrees is slower than switching branches)
- Using non-bare repo (can cause confusion)

### When to Use Branch Mode

✅ **Use branch mode when**:
- Standard Git workflow (most common case)
- Working on one feature at a time
- Prefer fast branch switching over isolated directories

### When to Use None Mode

✅ **Use none mode when**:
- Prototyping without version control
- Git is not available in your environment
- Project is managed with a different VCS

---

## Migration Guide (Future)

**Current limitation**: Changing modes after initialization is NOT supported. You must reinitialize.

**Planned for future versions**:
- Automatic branch → worktree conversion
- Automatic worktree → branch conversion
- Safe migration with backup/restore

For now, if you need to switch modes:
1. Commit all work
2. Delete `.specify/memory/config.json`
3. Run `specify init` again
4. Manually recreate features in the new mode

---

## Configuration Reference

**Location**: `.specify/memory/config.json`

**Schema**:
```json
{
  "version": "1.0",
  "source_management_flow": "<mode>",
  "worktree_folder": "<path>"
}
```

**Fields**:
- `version`: Config version (for future upgrades) - must be "1.0"
- `source_management_flow`: Required - must be "branch", "worktree", or "none"
- `worktree_folder`: Required for worktree mode only

**Example configurations**:

```json
// Branch mode (simplest)
{
  "version": "1.0",
  "source_management_flow": "branch"
}

// Worktree mode with default folder
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "./worktrees"
}

// Worktree mode with custom folder
{
  "version": "1.0",
  "source_management_flow": "worktree",
  "worktree_folder": "/mnt/ssd/worktrees"
}

// None mode (no Git)
{
  "version": "1.0",
  "source_management_flow": "none"
}
```

**Important**: 
- This file is in `.gitignore` (it's user-specific, not project configuration)
- JSON does not support comments - examples above use `//` for illustration only
- No trailing commas allowed in JSON

---

## FAQ

**Q: Can I manually edit config.json?**  
A: Yes, but only `worktree_folder` is safe to change. Don't change `source_management_flow` (mode switching is blocked). Be careful with JSON syntax (proper quotes, no trailing commas).

**Q: Is the config shared in Git?**  
A: No, it's in `.specify/memory/` which is gitignored. Each developer configures their own mode.

**Q: What if my team uses different modes?**  
A: That's fine! One developer can use branch mode while another uses worktree mode. The specs are compatible.

**Q: Can I use worktrees without Spec-Kit managing them?**  
A: Yes, but you won't get the naming validation and automatic creation. Set mode to `branch` or `none` and manage worktrees manually.

**Q: Does this work on Windows?**  
A: Yes, worktree mode works on Windows with Git Bash. Paths use forward slashes in config (`./worktrees`), Git handles the conversion.

**Q: What's the minimum Git version?**  
A: Git 2.13+ (July 2017) for stable worktree support. Use `git --version` to check.

---

## Next Steps

- **Learn the workflow**: Try creating a feature in your chosen mode
- **Read the spec**: See `specs/001-worktree-detection/spec.md` for technical details
- **Explore edge cases**: Check the Edge Cases section in the spec for advanced scenarios
- **Provide feedback**: Report issues or suggest improvements

---

**Related Documentation**:
- [Feature Specification](./spec.md)
- [Implementation Plan](./plan.md)
- [Data Model](./data-model.md)
- [API Contracts](./contracts/)
