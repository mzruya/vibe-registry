# work - Git Worktree Manager

A CLI tool that makes working with multiple git branches in parallel effortless. Instead of stashing changes or juggling commits, each branch gets its own directory.

## The Problem

When you're working on a feature and need to quickly fix a bug on another branch, you either stash your work, commit half-done code, or clone the repo again. Git worktrees solve this but the commands are verbose and forgettable.

## What Success Looks Like

**First-time setup (one time only):**
```
$ work init
Detected shell: zsh
Add this to your ~/.zshrc:

  eval "$(work init)"

Then restart your shell or run: source ~/.zshrc
```
The `init` command detects the user's shell and outputs a shell-specific wrapper function. This wrapper enables the `work` command to cd into worktrees (which a subprocess normally can't do). The user adds one line to their shell config and they're set up forever.

**Starting work on a new branch:**
```
$ work my-feature
Created worktree at ~/code/myproject-worktrees/my-feature
Checking out files in background...
~/code/myproject-worktrees/my-feature $
```
The command automatically cd's into the new worktree while files populate in the background.

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
- `work list` shows age (human-readable like "3 days ago"), and fetches PR info from GitHub using `gh` if available
- `work prune` fetches from origin first, then removes worktrees whose branches are gone from remote
- Before deleting, check for uncommitted changes or unpushed commits - ask for confirmation if found
- By default, `work <branch>` should return immediately (non-blocking):
  - Skip `git fetch` (use existing local refs - slightly stale is fine for speed)
  - Use `git worktree add --no-checkout` which creates the worktree structure instantly without checking out files
  - Create `.claude/settings.local.json` with the branch name for Claude Code session naming
  - Output a special `__CD__:/path/to/worktree` directive (the shell wrapper parses this to cd)
  - Spawn `git checkout HEAD` in background (this populates files while user is already in the directory)
  - Print a message like "Checking out files in background..."
- Add a `--wait` or `-w` flag for blocking mode when needed:
  - Run `git fetch origin` before creating the worktree
  - Use regular `git worktree add` (with checkout) and wait for completion
  - Useful for scripting or when you need to chain commands
- To check if a worktree already exists, check for `.git` file/directory inside it (not just the directory existing)

## Shell Integration (the `init` command)

The `work init` command enables the CLI to cd the user into worktrees. Here's how it works:

1. **Detect the shell** - check `$SHELL` env var for bash, zsh, or fish
2. **Output shell-specific wrapper** - a function that:
   - Calls the actual `work` binary
   - Captures output and parses for `__CD__:/path` directive
   - Prints all output except the `__CD__` line
   - cd's to the path if present

**Example output for `work init` (zsh/bash):**
```bash
work() {
  local output
  output=$(command work "$@")  # 'command' bypasses the function, calls the binary
  local exit_code=$?
  local cd_path=$(echo "$output" | grep "^__CD__:" | cut -d: -f2)
  echo "$output" | grep -v "^__CD__:"
  [[ -n "$cd_path" ]] && cd "$cd_path"
  return $exit_code
}
```

**Example output for `work init --shell fish`:**
```fish
function work
  set -l output (command work $argv)  # 'command' bypasses the function, calls the binary
  set -l exit_code $status
  set -l cd_path (echo $output | grep "^__CD__:" | cut -d: -f2)
  echo $output | grep -v "^__CD__:"
  test -n "$cd_path" && cd $cd_path
  return $exit_code
end
```

The `init` command should:
- Auto-detect shell if `--shell` not provided
- Print instructions for adding to shell config
- Support `--shell bash`, `--shell zsh`, `--shell fish`

## Technical Notes

- Language: Rust
- Use `clap` for CLI parsing, `colored` for terminal output, `dialoguer` for confirmations
- Shell out to `git` and `gh` commands (don't use libgit2)
- Handle missing `gh` gracefully - just skip PR info
- The binary is named `work` - the shell wrapper uses `command work` to call the binary (bypasses the function)
- Output `__CD__:/path/to/worktree` on its own line when the user should be cd'd somewhere
- For background processes, use `std::process::Command` with `.spawn()` and don't wait for it
- For human-readable time: calculate seconds since modification, convert to "X minutes/hours/days/weeks ago"
