# Work - Multi-Project Git Worktree Manager

Generate a Nushell script (`work.nu`) that manages git worktrees across multiple projects with GitHub PR status integration.

The script must be sourceable to allow directory changes in the calling shell.

## Overview

The tool enables working on multiple branches simultaneously across multiple git repositories. Projects are registered in a central config, and all worktrees are stored in a centralized location organized by project.

## Commands

| Command | Description |
|---------|-------------|
| `work` | Interactive project/worktree switcher with fuzzy search |
| `work <branch>` | Create worktree and cd into it (auto-registers project) |
| `work ls` | List worktrees for current project with PR/build status |
| `work rm [branch]` | Delete worktree (current worktree if no arg) |
| `work prune` | Clean up merged worktrees across ALL projects |
| `work add [path]` | Register a git repository as a project |

## Storage Layout

```
~/.config/work/
└── projects.nuon              # List of registered projects [{name, path}]

~/.work/worktrees/
├── web/                       # Project name (from directory basename)
│   ├── mz-feature-1/          # Full git worktree checkout
│   └── mz-bugfix/
└── zenpayroll/
    └── mz-other-feature/
```

## Command Behavior

### `work` (Interactive Switcher)

Two-level drill-down navigation:

1. **Outside a project**: Shows project list first
   - Displays: project name, worktree count, path
   - Select a project → enters worktree view

2. **Inside a project**: Shows worktree list first
   - Displays: branch name, age, PR number, PR state (open/merged/closed), build status (✓/✗/○)
   - Press ESC → goes back to project list
   - "main" option always present to return to main repo

### `work <branch>` (Create Worktree)

1. If not in a registered project, auto-registers current git repo
2. Fetches `origin/main`
3. Creates worktree:
   - If branch exists locally → uses existing branch
   - If branch doesn't exist → creates from `origin/main`
4. Sets up `.claude/settings.local.json` with session name
5. Runs `git checkout` in background for faster startup
6. cd's into the worktree

### `work ls` (List Worktrees)

Non-interactive list showing:
- Branch name
- Age (e.g., "2 days ago")
- PR number and state (from GitHub CLI)
- Build status with check counts (e.g., "✓ 5/5")
- Current worktree marker

### `work rm [branch]` (Delete Worktree)

- If no branch specified and inside a worktree, deletes current worktree
- Warns if branch still exists on remote (unmerged PR)
- Auto cd's to main repo if deleting current worktree
- Removes both worktree directory and local branch

### `work prune` (Cleanup)

Operates across ALL registered projects:

1. For each project with worktrees:
   - Fetches from origin (updates remote branch info)
   - For each worktree:
     - If branch no longer exists on remote → considered merged
     - If has local uncommitted/unpushed changes → skip with warning
     - Otherwise → remove worktree and local branch
2. Shows summary: "Pruned N worktree(s) across M project(s)"

### `work add [path]` (Register Project)

- Registers git repo at path (defaults to current directory)
- Project name derived from directory basename
- Validates it's a git repository
- Prevents duplicate registrations

## Project Detection

A "current project" is detected by checking (in order):
1. Is pwd inside `~/.work/worktrees/<project>/`? → that project
2. Is pwd inside any registered project's main repo path? → that project
3. Otherwise → null (not in a project)

## Features

### PR Status Integration
Uses `gh pr list` to fetch PR information for each branch:
- PR number and URL
- State: open (green), merged (purple), closed (red)
- CI status checks with pass/fail/pending counts

### Parallel Fetching
Fetches PR info for all worktrees concurrently using `par-each`.

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
- `✓ 3/5` - all checks passing (green)
- `✗ 3/5` - some checks failing (red)
- `○ 4/5` - checks pending (yellow)

### Safety Features
- Warns before deleting worktrees with branches still on remote
- Checks for uncommitted changes and unpushed commits
- Prompts for confirmation when deleting worktrees with local changes

## Integration Points

- **GitHub CLI (`gh`)**: Fetches PR info (number, state, status checks)
- **mise**: Auto-trusts config files when entering worktrees
- **Claude Code**: Sets up session name in `.claude/settings.local.json`

## Data Format

`projects.nuon` (Nushell structured data):
```nuon
[
  {name: "web", path: "/Users/user/workspace/web"},
  {name: "zenpayroll", path: "/Users/user/workspace/zenpayroll"}
]
```

## Script Structure

The script must be sourceable (not a standalone executable) because it needs to change the calling shell's directory.

Define commands using `def --env` for any command that changes directory:
```nushell
def --env work [arg?: string]: nothing -> nothing { ... }
def --env "work rm" [branch?: string]: nothing -> nothing { ... }
```

## Runtime Dependencies

- `git` - for worktree management
- `gh` - GitHub CLI for PR status
- Nushell's built-in `input list --fuzzy` for selection

## Example Usage

```shell
# Register current repo as a project
work add

# Start working on a new feature (auto-registers if needed)
work mz-new-feature

# Interactive switcher
work

# List all worktrees with status
work ls

# Clean up merged branches across all projects
work prune

# Delete current worktree
work rm
```

## Technical Notes

- Use `git worktree add --no-checkout` + background checkout for faster worktree creation
- Use `path expand` for all paths to handle `~` properly
- Handle edge cases: no projects registered, no worktrees, missing gh CLI
- All directory-changing functions must use `def --env`
