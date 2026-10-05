# wt

A small git worktree manager. It figures out which repository you are in and
uses a config file to decide which branch to base new worktrees on and where to
put them.

## Install

Clone the repo and create a link in a folder that $PATH is reading:

```bash
ln -sf <repo path>/wt ~/.bin/wt
```

## Config

Lives at `~/.config/wt/config` (override with the `WT_CONFIG` env var). It is
created with sensible defaults on first run.

```ini
# Worktrees are created at <worktree_root>/<repo>/<name>
worktree_root = ~/Work/worktrees

[repos]
<repo1> = <branch1>
<repo2> = <branch2>
<repo3> = <branch3>
<repo4> = <branch4>

# Optional: override worktree_root per repository
[roots]
<repo2> = ~/Work/<repo2>-trees
```

The key under `[repos]` is the directory name of the repository. Running `wt`
anywhere inside `~/Work/<repo1>` resolves to `<repo1>`, so new worktrees branch
off `<branch1>`.

## Setup hooks

To run extra setup after a worktree is created, drop an executable script named
after the repository next to the config file:

```bash
~/.config/wt/<repo>.setup.sh
```

If it exists, `wt create` runs it once the worktree is in place. The script runs
with the new worktree as its working directory and gets:

| arg | env | value |
| --- | --- | --- |
| `$1` | `WT_WORKTREE` | the new worktree path |
| `$2` | `WT_REPO` | repository name |
| `$3` | `WT_BRANCH` | the new branch |
| `$4` | `WT_REPO_ROOT` | the main checkout |

Example `~/.config/wt/repo1.setup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

worktree="$1"
repo_root="$4"

cp "$repo_root/local.properties" "$worktree/local.properties"
cp "$repo_root/.env" "$worktree/.env"
```

Make it executable (`chmod +x`); otherwise `wt` warns and runs it with `bash`.
A failing script is reported as a warning — the worktree is kept.

## Usage

```
wt create <name> [--from <branch>]   create a worktree off the configured base branch
wt list                              list worktrees for the current repository
wt delete <name|index> [--force]     delete one worktree (and its branch, if merged)
wt delete --all [--force]            delete every worktree for this repository
wt path <name|index>                 print a worktree path
wt config [show|path|edit]           inspect or edit the config file
```

### Examples

```bash
cd ~/Work/<repo1>
wt create fix-bug-2      # new worktree + branch, based on origin/<branch1>
wt list                  # 1  fix-bug-2   fix-bug-2
wt delete 1              # delete by index
wt delete fix-bug-2      # or by name
wt delete --all          # delete all of them (asks for confirmation)
```

## Behaviour notes

- `create` fetches the base branch from `origin` first, so worktrees start from
  the current upstream tip rather than a stale local ref. If the fetch fails it
  falls back to the local `origin/<base>` ref, then to a local branch.
- `wt` works the same from inside the main checkout or from inside a worktree —
  it always resolves back to the main repository.
- `list` numbers worktrees and marks dirty ones with `*`, missing ones with
  `(missing)`. The main checkout is never listed and never deleted.
- `delete` removes the branch too, but only via `git branch -d`, so an unmerged
  branch is kept and you get a warning. `--force` uses `-D` and discards
  uncommitted changes.
- `delete` refuses to remove the worktree you are currently standing in.
