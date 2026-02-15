# Work - Git Worktree Manager

Write a Rust CLI tool called `work` that manages git worktrees for parallel branch development with GitHub PR status integration.

## Overview

The tool enables working on multiple branches simultaneously by creating isolated worktrees in a sibling directory (`<repo>-worktrees/`). Each worktree is a full working copy of the repository on a specific branch.

## Commands

### `work` (no arguments)
Opens an interactive fuzzy picker to switch between existing worktrees. Prints the selected worktree path to stdout for shell integration.

### `work go <branch>`
Creates a new worktree for the branch if it doesn't exist. The binary only creates - shell wrappers intercept this command and use `work path` to cd afterward.
- New worktrees are created from `origin/main`
- Uses `--no-checkout` for faster creation
- Auto-configures Claude Code session name (`.claude/settings.local.json`)
- Spawns background checkout process

### `work list` or `work ls`
Lists all worktrees with rich status information:
- Branch name (highlighted green if current)
- Age (e.g., "2 days ago")
- Current worktree marker (`<- current`)
- PR number, state, and URL
- CI status with check counts (e.g., `✓ 3/5`)

### `work switch` or `work sw`
Interactive fuzzy picker showing all worktrees with the same rich formatting as `work list`. Type to filter, press Enter to select. Prints selected path to stdout.

### `work path <branch>`
Prints the path where a worktree for the given branch would be located. Does not create the worktree or check if it exists. Useful for shell integration.

### `work delete <branch>`
Deletes a worktree and its local branch.
- If no branch specified and currently in a worktree, deletes the current one
- Warns if branch still exists on remote (PR may not be merged)
- Prints main repo path to stdout if deleting current worktree

### `work prune`
Cleans up worktrees whose branches have been merged (no longer on origin).
- Fetches from origin first to ensure up-to-date info
- Checks for local changes before deleting
- Prompts for confirmation if worktree has uncommitted/unpushed changes

## Features

### PR Status Integration
Uses `gh pr list` to fetch PR information for each branch:
- PR number and URL
- State: open (green), merged (purple), closed (red)
- CI status checks with pass/fail/pending counts

### Parallel Fetching
Fetches PR info for all worktrees concurrently using async/tokio, significantly improving speed with multiple worktrees.

### Color Coding
- Green: open PRs, passing checks, current worktree
- Red: closed PRs, failing checks
- Yellow: pending checks
- Purple: merged PRs
- Cyan: PR numbers
- Grey: URLs and timestamps

### Status Check Display
Shows CI status as icon + count:
- `✓ 5/5` - all checks passing (green)
- `✗ 3/5` - some checks failing (red)
- `○ 4/5` - checks pending (yellow)

### Safety Features
- Warns before deleting worktrees with branches still on remote
- Checks for uncommitted changes and unpushed commits
- Prompts for confirmation when deleting worktrees with local changes

## Directory Structure

```
~/code/myrepo/              # Main repository
~/code/myrepo-worktrees/    # Worktrees directory
  ├── feature-a/            # Worktree for feature-a branch
  ├── feature-b/            # Worktree for feature-b branch
  └── bugfix-123/           # Worktree for bugfix-123 branch
```

## Shell Integration

Shell wrappers intercept `work go` to add directory changing:

```bash
# Bash/Zsh
work() {
  command work "$@"
  if [[ "$1" == "go" && -n "$2" ]]; then
    local path=$(command work path "$2")
    [[ -d "$path" ]] && cd "$path"
  fi
}
```

```nu
# Nushell
def --env work [...args] {
  ^work ...$args
  if ($args | first | default "") == "go" and ($args | length) > 1 {
    let branch = ($args | get 1)
    let path = (^work path $branch | str trim)
    if ($path | path exists) {
      cd $path
    }
  }
}
```

The `work path <branch>` command returns the worktree path for scripting:
```bash
# Check if worktree exists
if [[ -d "$(work path my-feature)" ]]; then
  echo "Worktree exists"
fi
```

## Runtime Dependencies

- `git` - for worktree management
- `gh` - GitHub CLI for PR status (optional but recommended)

## Example Usage

```sh
# Start working on a new feature (shell wrapper handles cd)
work go mz-new-feature

# List all worktrees with status
work list

# Quick switch with fuzzy finder (or just `work`)
work sw

# Clean up merged branches
work prune

# Delete a specific worktree
work delete old-branch
```

## Technical Requirements

- Language: Rust
- Use `clap` v4 with derive macros for CLI parsing
- Use `tokio` for async runtime and parallel PR fetching
- Use `skim` for interactive fuzzy selection
- Use `colored` for terminal colors
- Use `git2` or shell out to `git` for git operations
- Use `chrono` or `humantime` for relative time formatting
- Shell out to `gh` CLI for GitHub API calls
- The binary name must be `work`
- Compile with `cargo build --release`
