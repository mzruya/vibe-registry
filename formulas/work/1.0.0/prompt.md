# Work - Git Worktree Manager

Write a Nushell script called `work.nu` that manages git worktrees for parallel branch development with GitHub PR status integration.

## Overview

The tool enables working on multiple branches simultaneously by creating isolated worktrees in a sibling directory (`<repo>-worktrees/`). Each worktree is a full working copy of the repository on a specific branch.

## Commands

### `work` (no arguments)
Opens an interactive fuzzy picker to switch between existing worktrees. Changes to the selected worktree directory.

### `work go <branch>`
Creates a new worktree for the branch if it doesn't exist and changes to it.
- New worktrees are created from `origin/main`
- Auto-configures Claude Code session name (`.claude/settings.local.json`)

### `work list` or `work ls`
Lists all worktrees with rich status information:
- Branch name (highlighted green if current)
- Age (e.g., "2 days ago")
- Current worktree marker (`<- current`)
- PR number, state, and URL
- CI status with check counts (e.g., `3/5`)

### `work switch` or `work sw`
Interactive fuzzy picker showing all worktrees. Type to filter, press Enter to select. Changes to selected directory.

### `work path <branch>`
Prints the path where a worktree for the given branch would be located. Does not create the worktree or check if it exists. Useful for scripting.

### `work delete <branch>`
Deletes a worktree and its local branch.
- If no branch specified and currently in a worktree, deletes the current one
- Warns if branch still exists on remote (PR may not be merged)
- Returns to main repo if deleting current worktree

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
Fetches PR info for all worktrees concurrently using `par-each`, significantly improving speed with multiple worktrees.

### Color Coding
Use ANSI codes for colors:
- Green: open PRs, passing checks, current worktree
- Red: closed PRs, failing checks
- Yellow: pending checks
- Purple: merged PRs
- Cyan: PR numbers
- Grey: URLs and timestamps

### Status Check Display
Shows CI status as icon + count:
- `3/5` - all checks passing (green)
- `3/5` - some checks failing (red)
- `4/5` - checks pending (yellow)

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

## Script Structure

The script should define an `--env` function to allow directory changes:

```nu
# Main entry point
def --env work [
  command?: string  # Command to run (go, list, ls, switch, sw, path, delete, prune)
  ...args: string   # Additional arguments
] {
  # Implementation
}
```

Use subcommands pattern or match on the command argument.

## Runtime Dependencies

- `git` - for worktree management
- `gh` - GitHub CLI for PR status (optional but recommended)
- `fzf` or `sk` - for fuzzy selection (use whichever is available)

## Example Usage

```nu
# Start working on a new feature
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

## Technical Notes

- Use `git worktree list --porcelain` for parsing worktree info
- Use `gh pr list --json` for structured PR data
- Use `par-each` for parallel operations
- Handle edge cases: no worktrees, missing gh CLI, no fzf/sk
- The script should work when sourced: `source work.nu`
