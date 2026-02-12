# work - Git Worktree Manager

Write a Rust CLI tool called `work` that manages git worktrees for parallel branch development. It replaces the tedious `git worktree add/remove` workflow with simple commands.

## Worktree layout

Worktrees are stored in a sibling directory to the main repo. If the main git repo is at `/code/myproject`, worktrees go into `/code/myproject-worktrees/<branch>/`.

The "main repo" is always the first worktree returned by `git worktree list --porcelain` (the line starting with `worktree `).

## Commands

### `work <branch>` (default action)

Create a new worktree or navigate to an existing one.

1. If already inside the target worktree, print a message and exit
2. If the worktree directory exists, print its path so the user can `cd` to it
3. If it doesn't exist:
   a. Run `git fetch origin main` from the main repo
   b. Create the worktrees directory if needed
   c. If the branch already exists locally: `git worktree add --no-checkout <path> <branch>`
   d. If the branch is new: `git worktree add --no-checkout -b <branch> <path> origin/main`
   e. Create `.claude/settings.local.json` in the worktree with `{"name": "<branch>"}` for Claude Code session naming
   f. Add `.claude/` to the worktree's `.git/info/exclude` so it doesn't show as untracked
   g. Run `git checkout HEAD` in the new worktree
   h. Print the worktree path

Since a CLI can't change the caller's directory, print a line like:
```
cd /path/to/worktree
```
so the user can wrap it in a shell alias like `cd $(work mybranch)`.

### `work list` (also `work ls`)

List all worktrees with useful context.

For each directory in `<main-repo>-worktrees/`:
- Show the branch name
- Show a human-readable age ("3 days ago", "2 weeks ago") based on directory modification time
- Show PR info by running `gh pr list --head <branch> --state all --json number,url,state,statusCheckRollup --limit 1`
  - PR number and state (open/merged/closed) with color
  - Build status: passing (green check), failing (red x), pending (yellow circle)
  - PR URL
- Highlight the current worktree if the user is inside one

Sort worktrees by age (newest first).

Output format example:
```
Worktrees in ~/code/myproject-worktrees:

  fix-login  (3 days ago)  #142 open  passing  https://github.com/org/repo/pull/142
  new-feature  (1 week ago)  #138 merged  https://github.com/org/repo/pull/138
  experiment  (2 weeks ago)
```

### `work delete <branch>` (also `work -d <branch>`)

Delete a worktree and its local branch.

1. If no branch specified and currently inside a worktree, infer the branch from the current directory
2. If the branch still exists on remote (`git ls-remote --heads origin <branch>`), warn the user and ask for confirmation
3. If the branch is gone from remote, proceed without asking (PR was likely merged)
4. Run `git worktree remove <path> --force` and `git branch -D <branch>`

### `work prune`

Automatically clean up worktrees whose branches have been merged.

1. Run `git fetch origin --prune`
2. For each worktree, check if the branch still exists on remote
3. If the branch is gone from remote:
   - Check for local changes (uncommitted files via `git status --porcelain`, or unpushed commits via `git log origin/<branch>..HEAD`)
   - If there are local changes, ask for confirmation before deleting
   - If clean, delete automatically
4. Print a summary of how many worktrees were pruned/skipped

### `work` (no arguments)

Print usage help.

## Technical Requirements

- Language: Rust
- Use `clap` v4 with derive macros for CLI parsing
- Use `colored` crate for terminal colors
- Shell out to `git` and `gh` commands using `std::process::Command` (do NOT use libgit2)
- Parse command output as needed (e.g., `git worktree list --porcelain`, `gh pr list --json`)
- Use `serde` and `serde_json` for parsing `gh` JSON output
- Use `chrono` for time-ago formatting from file modification timestamps
- Use `dialoguer` for interactive confirmation prompts (yes/no)
- The binary name must be `work`
- Create a complete `Cargo.toml` with package name "work"
- All git operations should use `-C <main_repo>` to operate from the main repo context
- Handle errors gracefully with helpful messages
- The code should compile with `cargo build --release`

## Edge cases to handle

- Not in a git repository: print error and exit
- Worktree directory doesn't exist yet: create it
- `gh` not installed: skip PR info gracefully, don't crash
- Branch name inference when user is inside a worktree and runs `work delete` with no args
- User is currently inside a worktree being deleted: warn them to cd out first
