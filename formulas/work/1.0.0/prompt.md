# work - Git Worktree Manager

A bash script that makes working with multiple git branches in parallel effortless. Instead of stashing changes or juggling commits, each branch gets its own directory.

## The Problem

When you're working on a feature and need to quickly fix a bug on another branch, you either stash your work, commit half-done code, or clone the repo again. Git worktrees solve this but the commands are verbose and forgettable.

## What Success Looks Like

**Starting work on a new branch:**
```
$ work my-feature
Created worktree at ~/code/myproject-worktrees/my-feature
cd ~/code/myproject-worktrees/my-feature
```
The user copies the cd command (or wraps `work` in a shell function to auto-cd).

**Seeing what's in flight:**
```
$ work list
Worktrees in ~/code/myproject-worktrees:

  my-feature     (2 hours ago)   #423 open ✓   github.com/org/repo/pull/423
  bugfix-login   (3 days ago)    #419 merged
  experiment     (1 week ago)
```
At a glance: what branches exist, how old they are, and PR status if one exists.

**Cleaning up after PRs merge:**
```
$ work prune
Pruned: bugfix-login (branch deleted from origin)
Skipped: experiment (has local changes)
Pruned 1 worktree
```
Automatically removes worktrees whose branches no longer exist on origin (PR was merged/closed). Warns before deleting anything with uncommitted work.

**Deleting a specific worktree:**
```
$ work delete my-feature
```
Or `work -d my-feature`. If the branch still exists on origin, confirm first (PR might not be merged yet).

## Behavior Details

- Worktrees live in `<repo>-worktrees/<branch>/` as a sibling to the main repo
- `work <branch>` creates a new worktree from origin/main, or navigates to an existing one
- Since scripts can't change the parent shell's directory, print a `cd` command the user can execute
- `work list` shows age (human-readable like "3 days ago"), and fetches PR info from GitHub using `gh` if available
- `work prune` fetches from origin first, then removes worktrees whose branches are gone from remote
- Before deleting, check for uncommitted changes or unpushed commits - ask for confirmation if found
- By default, `work <branch>` should return immediately (non-blocking):
  - Skip `git fetch` (use existing local refs - slightly stale is fine for speed)
  - Use `git worktree add --no-checkout` which creates the worktree structure instantly without checking out files
  - Create `.claude/settings.local.json` with the branch name for Claude Code session naming
  - Print the cd command
  - Spawn `git checkout HEAD` in background using `&` (this populates files while user is already in the directory)
  - Print a message like "Checking out files in background..."
- Add a `--wait` or `-w` flag for blocking mode when needed:
  - Run `git fetch origin` before creating the worktree
  - Use regular `git worktree add` (with checkout) and wait for completion
  - Useful for scripting or when you need to chain commands
- To check if a worktree already exists, check for `.git` file/directory inside it (not just the directory existing)

## Technical Notes

- Language: Bash (#!/bin/bash)
- Use ANSI escape codes for colored output (green for success, yellow for warnings, red for errors)
- Use `read -p` for confirmations
- Handle missing `gh` gracefully - just skip PR info (check with `command -v gh`)
- The script must be named `work` (no .sh extension)
- Replace `/` with `-` in branch names for directory names (e.g., `feature/foo` becomes `feature-foo`)
- For human-readable time: calculate seconds since modification, convert to "X minutes/hours/days/weeks ago"
