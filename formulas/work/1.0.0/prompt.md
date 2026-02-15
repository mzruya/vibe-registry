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

**Navigation:**
- Select project → shows its worktrees
- Press ESC in worktree view → returns to project list
- Select worktree → cd to that directory

**TUI Mockup — Project List:**
```
Select project:
> web           3 worktrees   ~/workspace/web
  zenpayroll    1 worktree    ~/workspace/zenpayroll
  api           no worktrees  ~/workspace/api    <- current
```

**TUI Mockup — Worktree List:**
```
web [esc=back]:
> main                (main repo)
  mz-feature-auth     (2 days ago)   #142 open   ✓ 12/12
  mz-fix-login        (5 days ago)   #138 merged
  mz-refactor-api     (1 week ago)   #135 open   ✗ 8/12
  mz-add-tests        (3 days ago)              <- current
```

### `work <branch>` — Create Worktree

1. **Auto-register:** If not in a registered project but in a git repo, register it automatically
2. **Fetch:** Run `git fetch origin main`
3. **Create:**
   - If branch exists locally → create worktree from it
   - If branch doesn't exist → create new branch from `origin/main`
4. **Setup:** Create `.claude/settings.local.json` with `{"name": "<branch>"}`
5. **Optimize:** Run `git checkout` in background for faster perceived startup
6. **Navigate:** cd into the worktree

**TUI Mockup — New branch:**
```
~/workspace/web $ work mz-new-feature
web > mz-new-feature
Fetching origin/main...
Creating new branch from origin/main...
~/.work/worktrees/web/mz-new-feature $
```

**TUI Mockup — Auto-register:**
```
~/workspace/newproject $ work mz-first-branch
Registering 'newproject'...
newproject > mz-first-branch
Fetching origin/main...
Creating new branch from origin/main...
~/.work/worktrees/newproject/mz-first-branch $
```

**TUI Mockup — Existing branch:**
```
~/workspace/web $ work mz-existing-feature
web > mz-existing-feature
Fetching origin/main...
Creating from existing branch...
~/.work/worktrees/web/mz-existing-feature $
```

### `work ls` — List Worktrees

**Requires:** Must be inside a registered project

**TUI Mockup:**
```
~/workspace/web $ work ls
web worktrees:

  mz-feature-auth   (2 days ago)   #142 open ✓ 12/12  https://github.com/org/web/pull/142
  mz-fix-login      (5 days ago)   #138 merged
  mz-refactor-api   (1 week ago)   #135 open ✗ 8/12   https://github.com/org/web/pull/135
  mz-add-tests      (3 days ago)   <- current

```

**TUI Mockup — No worktrees:**
```
~/workspace/api $ work ls
No worktrees for api
```

**TUI Mockup — Not in project:**
```
~/random/dir $ work ls
Error: Not in a registered project
Use work add to register a project
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

**TUI Mockup — Delete current (merged):**
```
~/.work/worktrees/web/mz-old-feature $ work rm
Removing 'mz-old-feature'...
Removed mz-old-feature
~/workspace/web $
```

**TUI Mockup — Delete by name:**
```
~/workspace/web $ work rm mz-old-feature
Removed mz-old-feature
```

**TUI Mockup — Branch still on remote:**
```
~/.work/worktrees/web/mz-wip $ work rm
Warning: Branch still exists on remote
Delete anyway? [y/N] y
Switching to main repo...
Removed mz-wip
~/workspace/web $
```

**TUI Mockup — Not in worktree:**
```
~/workspace/web $ work rm
Usage: work rm <branch>
```

### `work prune` — Cleanup Merged Worktrees

**Scope:** Operates across ALL registered projects

**For each project:**
1. Fetch from origin to update remote branch info
2. For each worktree:
   - Skip if branch still exists on remote
   - Skip (with warning) if has uncommitted changes or unpushed commits
   - Otherwise → remove worktree and local branch

**TUI Mockup — Normal operation:**
```
$ work prune
web - fetching...
  mz-old-feature      removed
  mz-shipped          removed
zenpayroll - fetching...
  mz-done             removed

Pruned 3 worktree(s) across 2 project(s)
```

**TUI Mockup — With local changes:**
```
$ work prune
web - fetching...
  mz-old-feature      removed
  mz-wip              has local changes - skipped
zenpayroll - fetching...

Pruned 1 worktree(s) across 1 project(s)
Skipped 1 with local changes
```

**TUI Mockup — Nothing to prune:**
```
$ work prune
web - fetching...
zenpayroll - fetching...

No worktrees to prune
```

**TUI Mockup — No projects:**
```
$ work prune
No projects registered
```

### `work add [path]` — Register Project

- Path defaults to current directory
- Validates path is a git repository
- Project name = directory basename
- Prevents duplicate registrations (by name or path)

**TUI Mockup — Register current directory:**
```
~/workspace/web $ work add
Registered 'web'
```

**TUI Mockup — Register by path:**
```
$ work add ~/workspace/api
Registered 'api'
```

**TUI Mockup — Already registered:**
```
~/workspace/web $ work add
Project 'web' already registered
```

**TUI Mockup — Not a git repo:**
```
~/random/dir $ work add
Error: Not a git repository
```

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
