---
name: manage-env
description: Add, remove, or edit shell aliases and small bash scripts in the env dotfiles repo (~/repos/env). Use when asked to create/add/remove an alias, or add/remove a small bash script/wrapper to PATH.
version: 1.0.0
allowed-tools: Bash, Read, Edit, Write, Glob, Grep
---

# Manage env repo (aliases & scripts)

Repo: `~/repos/env` (github: `xyzBart/env`). Not committed/pushed automatically — do that explicitly (see below).

## Where things live

- **Aliases file**: `~/repos/env/linux/aliases` — plain `alias name='...'` lines (and a couple of bash functions). Symlinked at `~/aliases` (do not edit through the symlink — `Edit`/`Write` refuse to write through it; always target the real path `~/repos/env/linux/aliases`). Sourced by `~/repos/env/linux/bashrc`, which is sourced by `~/.bashrc`.
- **Scripts dir**: `~/repos/env/bin` — this directory is on `$PATH` (added in `~/repos/env/linux/bashrc`). Any executable file dropped here is runnable by its filename alone, no extension needed in the invocation (e.g. `chatgptToMd.sh` is run as `chatgptToMd.sh`).

## Adding an alias

1. Read `~/repos/env/linux/aliases` to see existing style/grouping.
2. `Edit` the file (real path, not `~/aliases`) to add `alias name='command'` near related aliases.
3. If the command needs variable expansion to happen at *call time* (not at definition time), use single quotes for the alias body; escape inner `$` as `\$` if nesting double quotes for a sub-command like `awk`.
4. Test before committing: `bash -i -c 'source ~/aliases; <aliasname> ...'`
   - **Never** wrap the alias name with an external command like `timeout <n> <aliasname>` — `timeout` does an `execve` lookup for `<aliasname>` as a real binary on `$PATH`, but an alias only exists inside the interactive shell that sourced it, so this fails with `exit code 127: No such file or directory` even though the alias works fine. If a timeout is needed, put `timeout` *inside* the `bash -i -c` string instead: `bash -i -c 'source ~/aliases; timeout 180 <aliasname> ...'` — or rely on the `Bash` tool's own `timeout` parameter and skip shell-level `timeout` entirely.

## Removing an alias

1. Read the file, locate the `alias name=...` line (or function block for multi-line ones like `rgmd`).
2. `Edit` to delete it.

## Adding a small bash script

1. Write the script to `~/repos/env/bin/<name>.sh` (or no extension, matching existing conventions in that dir — check `ls ~/repos/env/bin` for similar wrappers first).
2. `chmod +x ~/repos/env/bin/<name>.sh`
3. Test by running it directly with its full path, or `source ~/repos/env/linux/bashrc` in a subshell and invoking by name.

## Removing a script

`rm ~/repos/env/bin/<name>` — confirm with the user first since this is a delete (see destructive-action guidance).

## Committing

Use the `commit` skill, scoped to just the files you touched (don't `git add -A` — this repo tends to have other unrelated pending changes). Push only if the user explicitly asks.
