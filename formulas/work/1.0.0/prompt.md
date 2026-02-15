# Work - Multi-Project Git Worktree Manager

Generate two shell scripts that manage git worktrees across multiple projects:

1. **`work.sh`** - POSIX-compatible script for bash/zsh
2. **`work.nu`** - Nushell script using idiomatic patterns

Both scripts must be sourceable to allow directory changes in the calling shell.

---

## Product Overview

**Problem:** Developers working across multiple repositories need to context-switch between branches frequently. Traditional `git stash` / `git checkout` workflows are slow and error-prone.

**Solution:** A unified CLI that manages git worktrees across all your projects. Each branch gets its own directory, enabling true parallel development. All worktrees are stored centrally for easy discovery and cleanup.

---

## CLI Reference

| Command | Description |
|---------|-------------|
| `work` | Interactive project → worktree navigator |
| `work <branch>` | Create worktree and cd into it |
| `work ls` | List worktrees with PR status |
| `work rm [branch]` | Delete worktree |
| `work prune` | Clean up merged worktrees (all projects) |
| `work add [path]` | Register a project |

---

## Storage Layout

```
~/.config/work/
└── projects.nuon          # [{name: "web", path: "/path/to/web"}, ...]

~/.work/worktrees/
├── <project>/
│   ├── <branch>/          # Full git worktree
│   └── <branch>/
└── <project>/
    └── <branch>/
```

---

## Behavioral Specifications

### `work` — Interactive Navigator

**Entry point behavior depends on context:**

| Context | Initial View |
|---------|--------------|
| Outside any project | Project list |
| Inside a registered project | Worktree list for that project |

**Project list displays:**
- Project name (bold)
- Worktree count (e.g., "3 worktrees" or "no worktrees")
- Path (dimmed)
- Current marker if applicable

**Worktree list displays:**
- Branch name
- Age (e.g., "2 days ago")
- PR number + state (open/merged/closed)
- CI status (✓ passing, ✗ failing, ○ pending) with counts
- Current marker if applicable
- "main" option to return to main repo

**Navigation:**
- Select project → shows its worktrees
- Press ESC in worktree view → returns to project list
- Select worktree → cd to that directory

### `work <branch>` — Create Worktree

1. **Auto-register:** If not in a registered project but in a git repo, register it automatically
2. **Fetch:** Run `git fetch origin main`
3. **Create:**
   - If branch exists locally → create worktree from it
   - If branch doesn't exist → create new branch from `origin/main`
4. **Setup:** Create `.claude/settings.local.json` with `{"name": "<branch>"}`
5. **Optimize:** Run `git checkout` in background for faster perceived startup
6. **Navigate:** cd into the worktree

### `work ls` — List Worktrees

**Requires:** Must be inside a registered project

**Output format:**
```
<project> worktrees:

  <branch>  (2 days ago)  #123 open ✓ 5/5  <url>
  <branch>  (1 week ago)  #120 merged
  <branch>  (3 days ago) <- current
```

### `work rm [branch]` — Delete Worktree

**Branch resolution:**
- If branch provided → delete that worktree
- If no branch and inside a worktree → delete current worktree
- If no branch and not in worktree → error with usage hint

**Safety checks:**
1. If branch still exists on remote → warn user (PR may not be merged)
2. Require confirmation to proceed

**Cleanup:**
- Remove worktree directory
- Delete local branch
- If deleting current worktree → cd to main repo

### `work prune` — Cleanup Merged Worktrees

**Scope:** Operates across ALL registered projects

**For each project:**
1. Fetch from origin to update remote branch info
2. For each worktree:
   - Skip if branch still exists on remote
   - Skip (with warning) if has uncommitted changes or unpushed commits
   - Otherwise → remove worktree and local branch

**Output:**
```
web - fetching...
  mz-old-feature  removed
  mz-shipped      removed
zenpayroll - fetching...
  mz-done         removed

Pruned 3 worktree(s) across 2 project(s)
Skipped 1 with local changes
```

### `work add [path]` — Register Project

- Path defaults to current directory
- Validates path is a git repository
- Project name = directory basename
- Prevents duplicate registrations (by name or path)

---

## Project Detection Logic

To determine "current project":

1. Is pwd inside `~/.work/worktrees/<project>/...`? → that project
2. Is pwd inside any registered project's path? → that project
3. Otherwise → not in a project

---

## Display Standards

### Colors
| Element | Color |
|---------|-------|
| Open PR | Green |
| Merged PR | Purple |
| Closed PR | Red |
| Passing checks | Green |
| Failing checks | Red |
| Pending checks | Yellow |
| Current worktree | Green |
| PR numbers | Cyan |
| Paths, timestamps | Grey/dim |

### CI Status Format
```
✓ 5/5    # All passing (green)
✗ 3/5    # Has failures (red)
○ 4/5    # Has pending (yellow)
```

---

## Technical Requirements

### Bash/Zsh (`work.sh`)
- Define `work()` function with case statement
- Use `fzf` for fuzzy selection (with `--ansi` for colors)
- Use `jq` for JSON parsing
- Use background jobs (`&` + `wait`) for parallel PR fetching
- Store temp data in `$TMPDIR` or `/tmp`

### Nushell (`work.nu`)
- Define `def --env work` (env required for cd)
- Use `input list --fuzzy` for selection
- Use native `from json` for parsing
- Use `par-each` for parallel PR fetching
- Use `.nuon` format for config file

### Both
- `git` for worktree operations
- `gh` CLI for PR status
- Handle missing dependencies gracefully

---

## Edge Cases to Handle

- No projects registered → prompt to use `work add`
- No worktrees for project → show empty state
- `gh` CLI not available → skip PR status, show branch names only
- Branch has no PR → show branch without PR info
- Worktree has detached HEAD → handle gracefully
- User cancels selection → exit cleanly
