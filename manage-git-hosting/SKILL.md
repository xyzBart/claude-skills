---
name: manage-git-hosting
description: Manage repos/projects on GitHub and GitLab via their CLIs (gh, glab). Use when asked to check whether gh/glab are installed and authenticated, list repos or projects, or init/push a local repo to GitHub or GitLab.
version: 1.0.0
allowed-tools: Bash
---

# Manage git hosting (GitHub + GitLab)

Two separate CLIs, one per host: `gh` (GitHub) and `glab` (GitLab). Neither depends on the other being installed.

## Check both are present and authenticated

```bash
command -v gh >/dev/null && gh --version || echo "gh: NOT INSTALLED"
echo "---"
command -v glab >/dev/null && glab --version || echo "glab: NOT INSTALLED"
echo "---"
gh auth status 2>&1
echo "---"
glab auth status 2>&1
```

- `gh auth status` reports per-host login and token scopes.
- `glab auth status` reports the logged-in user, which protocol (https/ssh) git operations use, and whether a token is configured.
- A tool can be installed but not authenticated — always check `auth status`, not just `--version`.

## Known environment facts (uzer-Latitude-5420)

- GitHub: the SSH key in the agent (`ssh-add -l`) is registered with GitHub, so `git@github.com:...` remotes work.
- GitLab: that same SSH key is **not** registered with GitLab → `git@gitlab.com:...` push fails with `Permission denied (publickey)`. `glab auth status` shows GitLab is configured for **https** protocol with a stored token instead. For any new GitLab remote, use the HTTPS form (`https://gitlab.com/<namespace>/<repo>.git`) unless the user has since added the SSH key to GitLab's account SSH keys settings — re-check with `ssh -T git@gitlab.com` if unsure.
- GitLab namespaces: a personal account namespace (e.g. `gitlab.com/xyzBart`) can hold projects directly but **cannot** have subgroups — subgroups are a group-only feature. For nested/folder-style organization, use (or create) an actual GitLab group instead (this account already has one: `xyzbart-group`).

## Listing repos/projects

```bash
gh repo list xyzBart --limit 100
glab repo list
```

## Init + push a new local repo

Pick the remote form per the SSH/HTTPS note above.

```bash
cd <path>
git init -b main
git add -A
git commit -m "Initial commit"
```

GitHub, repo doesn't exist yet (creates it and pushes in one step):
```bash
gh repo create xyzBart/<name> --private --source=. --push
```

GitHub or GitLab, repo already exists remotely:
```bash
git remote add origin git@github.com:xyzBart/<name>.git   # GitHub (SSH works)
# or
git remote add origin https://gitlab.com/<namespace>/<name>.git   # GitLab (use HTTPS)
git push -u origin main
```
