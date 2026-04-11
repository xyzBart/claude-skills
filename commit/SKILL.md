---
name: commit
description: Create a git commit following project conventions. Does NOT add Co-Authored-By attribution lines.
version: 1.0.0
allowed-tools: Bash, Read, Glob, Grep
---

# Git Commit Skill

Create a git commit. Follow the steps below exactly.

## Workflow

### 1. Gather state (run in parallel)

```bash
git status          # see untracked/modified files
git diff            # see staged and unstaged changes
git log --oneline -10  # learn commit message style of this repo
```

### 2. Draft the commit message

- Summarize the *why*, not just the *what*
- Keep the subject line under 72 characters
- Add a blank line + body only if extra context is needed
- **Do NOT add any "Co-Authored-By" or attribution trailer lines**

### 3. Stage and commit

Stage specific relevant files (avoid `git add -A` / `git add .` to prevent accidentally including secrets or binaries).

Pass the commit message via HEREDOC to preserve formatting:

```bash
git commit -m "$(cat <<'EOF'
Your commit message here
EOF
)"
```

### 4. Confirm

Run `git status` and report the result to the user.

## Rules

- Never amend a published commit
- Never skip hooks (`--no-verify`)
- Never force-push unless the user explicitly requests it
- Never commit files that look like secrets (`.env`, credentials, private keys)
- **Never add Co-Authored-By or any AI attribution trailers**
