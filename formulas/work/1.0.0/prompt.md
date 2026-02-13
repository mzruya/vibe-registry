# work - Git Worktree Manager

A CLI that makes working with multiple git branches effortless. Each branch gets its own directory - no more stashing or juggling commits.

## The Problem

You're deep in a feature when a bug needs fixing on another branch. Your options: stash your work, commit half-done code, or clone the repo again. Git worktrees solve this but the commands are verbose and forgettable.

## Commands

### `work <branch>`

Creates a new worktree and cd's into it. If the worktree already exists, just cd's there.

```
~/code/myproject $ work my-feature
Creating worktree for branch: my-feature
Checking out files in background...
~/code/myproject-worktrees/my-feature $
```

The command returns instantly. Files are checked out in the background while you're already in the directory. New branches are created from origin/main.

### `work list`

Shows all worktrees with age and PR status (if `gh` CLI is available).

```
~/code/myproject $ work list
Worktrees in ~/code/myproject-worktrees:

  my-feature     (2 hours ago)   #423 open ✓   github.com/org/repo/pull/423
  bugfix-login   (3 days ago)    #419 merged
  experiment     (1 week ago)
```

### `work prune`

Removes worktrees whose branches no longer exist on origin (PR was merged/closed).

```
~/code/myproject $ work prune
Fetching from origin...
Pruned: bugfix-login (branch deleted from origin)
Skipped: experiment (has uncommitted changes)
Pruned 1 worktree
```

Warns before deleting anything with uncommitted changes or unpushed commits.

### `work delete <branch>` or `work -d <branch>`

Deletes a specific worktree. Asks for confirmation if:
- Branch still exists on origin (PR might not be merged)
- There are uncommitted changes
- There are unpushed commits

### `work init`

Sets up shell integration. The user runs this once, follows the instructions, and the `work` command can cd them into worktrees from then on.

```
$ work init
Detected shell: zsh
Add this to your ~/.zshrc:

  eval "$(work init)"

Then restart your shell.
```

Must support: bash, zsh, fish, and nushell.

## Behavior Details

- Worktrees live in `<repo>-worktrees/<branch>/` as a sibling to the main repo
- `work <branch>` should return instantly - no blocking on git fetch or file checkout
- Add a `--wait` flag for blocking mode when needed (fetches first, waits for checkout)
- A worktree "exists" if it has a `.git` file/directory inside it
- Create `.claude/settings.local.json` with `{"sessionName": "<branch>"}` for Claude Code
- Handle missing `gh` gracefully - just skip PR info
- Use colors: green for success, yellow for warnings, red for errors

## Technical Constraints

- Language: Rust
- The binary must be named `work`
- Shell out to `git` and `gh` (don't use libgit2)
