# work - Git Worktree Manager

A CLI that makes working with multiple git branches effortless. Each branch gets its own directory - no more stashing or juggling commits.

## The Problem

You're deep in a feature when a bug needs fixing on another branch. Your options: stash your work, commit half-done code, or clone the repo again. Git worktrees solve this but the commands are verbose and forgettable.

## Directory Structure

Worktrees live as siblings to the main repo with a `-worktrees` suffix:

```
~/code/
├── myproject/                    # main repo
└── myproject-worktrees/
    ├── my-feature/               # worktree for my-feature branch
    ├── bugfix-login/             # worktree for bugfix-login branch
    └── experiment/               # worktree for experiment branch
```

## Commands

### `work <branch>`

Creates a new worktree and cd's into it. If the worktree already exists, just cd's there.

**Critical UX requirement: This command MUST return instantly (under 1 second) regardless of repository size.** The user should be in their new directory immediately. Any slow operations like fetching or populating files must happen asynchronously after the prompt returns. This is the entire point of the tool - fast context switching.

**Creating a new worktree:**
```
~/code/myproject $ work my-feature
Created worktree at ~/code/myproject-worktrees/my-feature
Checking out files in background...
~/code/myproject-worktrees/my-feature $
```

The user is now in the worktree directory. Files appear in the background over the next few seconds while they can already start working.

**Switching to existing worktree:**
```
~/code/myproject $ work my-feature
Switching to existing worktree: my-feature
~/code/myproject-worktrees/my-feature $
```

New branches are created from origin/main (using whatever is locally available - no fetching).

"Created worktree" and "Switching to existing worktree" messages are green.
"Checking out files in background..." is dimmed/gray.

### `work list`

Shows all worktrees with age and PR status.

```
~/code/myproject $ work list
Worktrees in ~/code/myproject-worktrees:

  my-feature     (2 hours ago)    #423 open ✓    github.com/org/repo/pull/423
  bugfix-login   (3 days ago)     #419 merged
  experiment     (1 week ago)
```

- Branch names are cyan and bold
- Age in parentheses is dimmed/gray
- PR number and status: "open ✓" is green, "merged" is purple, "closed" is red
- PR URL is dimmed/gray
- If `gh` CLI is not available, just skip the PR info columns

Age format: "just now", "2 minutes ago", "1 hour ago", "3 days ago", "2 weeks ago"

### `work prune`

Removes worktrees whose branches no longer exist on origin (PR was merged/closed).

```
~/code/myproject $ work prune
Fetching from origin...
Pruned: bugfix-login (branch deleted from origin)
Skipped: experiment (has uncommitted changes)
Pruned 1 worktree
```

- "Fetching from origin..." is dimmed/gray
- "Pruned:" is green, followed by branch name
- "Skipped:" is yellow, followed by branch name and reason in parentheses

Never deletes worktrees with uncommitted changes or unpushed commits without asking.

### `work delete <branch>` or `work -d <branch>`

Deletes a specific worktree.

**Critical UX requirement: This command MUST return instantly (under 1 second) regardless of repository size.** The user should be back at their prompt immediately. Any slow operations like removing files must happen asynchronously after the prompt returns.

**Behavior depends on current location:**
- If inside the worktree being deleted: cd to the main repo first, then delete the worktree
- If inside a different worktree or the main repo: delete without changing location
- If not in a git repository: show "Error: Not in a git repository" (red)

**Clean deletion (branch already merged):**
```
~/code/myproject $ work -d bugfix-login
Deleted: bugfix-login
```

**With uncommitted changes:**
```
~/code/myproject $ work -d my-feature
Warning: Worktree has uncommitted changes
Delete anyway? [y/N] n
Aborted.
```

**Branch still on origin:**
```
~/code/myproject $ work -d my-feature
Warning: Branch still exists on origin (PR may not be merged)
Delete anyway? [y/N]
```

- "Warning:" is yellow and bold
- "Deleted:" is green
- "Aborted." is dimmed/gray
- Default for confirmation prompts is always No

### `work home`

Navigate back to the main repository directory.

**From a worktree:**
```
~/code/myproject-worktrees/my-feature $ work home
~/code/myproject $
```

**Already in the main repo:**
```
~/code/myproject $ work home
~/code/myproject $
```

- If inside a worktree: cd to the main repo
- If already in the main repo: stay (no-op, no output)
- If not in a git repository: show "Error: Not in a git repository" (red)

### `work init`

Sets up shell integration so the command can cd into worktrees.

```
$ work init
Detected shell: zsh
Add this to your ~/.zshrc:

  eval "$(work init)"

Then restart your shell or run: source ~/.zshrc
```

**With explicit shell:**
```
$ work init --shell fish
Detected shell: fish
Add this to your ~/.config/fish/config.fish:

  work init | source

Then restart your shell.
```

Must support: bash, zsh, fish, and nushell. Auto-detect from $SHELL if --shell not provided.

### `work --wait <branch>` or `work -w <branch>`

Blocking mode - the opposite of the default instant behavior. Fetches from origin first, waits for all files to be checked out, then returns. Takes 10-30+ seconds on large repos. Useful for scripts or CI where you need files immediately.

```
~/code/myproject $ work --wait my-feature
Fetching from origin...
Created worktree at ~/code/myproject-worktrees/my-feature
~/code/myproject-worktrees/my-feature $
```

This is intentionally slow because it ensures everything is ready before returning.

## Additional Behavior

- A worktree "exists" if it has a `.git` file or directory inside it
- Create `.claude/settings.local.json` with `{"sessionName": "<branch>"}` in each new worktree for Claude Code session naming
- If not in a git repository, show: "Error: Not in a git repository" (red)

## Technical Constraints

- Language: Rust
- Binary name: `work`
- Shell out to `git` and `gh` commands (don't use libgit2)
- Use `clap` for argument parsing
- Use `colored` crate for terminal colors
- Use `dialoguer` for confirmation prompts
