# Module 04: Git & Version Control

> Part of the [DevOps Career Course](./README.md) by UncleJS

---

## Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Beginner: What is Git & Why It Matters](#beginner-what-is-git--why-it-matters)
- [Beginner: Core Git Commands](#beginner-core-git-commands)
- [Beginner: Branching & Merging](#beginner-branching--merging)
- [Beginner: Working with Remote Repositories](#beginner-working-with-remote-repositories)
- [Intermediate: Resolving Conflicts](#intermediate-resolving-conflicts)
- [Intermediate: Git Workflows](#intermediate-git-workflows)
- [Intermediate: Advanced Git Techniques](#intermediate-advanced-git-techniques)
- [Intermediate: Git Hooks](#intermediate-git-hooks)
- [Intermediate: Git for Infrastructure Code](#intermediate-git-for-infrastructure-code)
- [Tools & Commands Reference](#tools--commands-reference)
- [Hands-On Labs](#hands-on-labs)
- [Further Reading](#further-reading)

---

## Overview

Version control is the foundation of everything in DevOps. Git tracks every change to every file in your codebase — who made it, when, and why. It enables teams to work in parallel without stepping on each other, safely experiment with new features, and roll back to any previous state in seconds.

Infrastructure code, CI/CD pipelines, Kubernetes manifests, Terraform configs — all of it lives in Git. If it's not in Git, it doesn't exist.

[↑ Back to TOC](#table-of-contents)

---

## Learning Objectives

By the end of this module you will be able to:

- Initialize repositories and make commits with meaningful messages
- Use branches to develop features in isolation
- Merge branches and resolve conflicts confidently
- Push and pull code from remote repositories (GitHub, GitLab)
- Use pull requests / merge requests for code review
- Apply Git workflows used by professional engineering teams
- Use `rebase`, `cherry-pick`, `stash`, and `bisect` for advanced tasks
- Write Git hooks to automate pre-commit and pre-push checks
- Apply Git best practices to infrastructure-as-code repositories

[↑ Back to TOC](#table-of-contents)

---

## Beginner: What is Git & Why It Matters

Git is a **distributed** version control system — every developer has a full copy of the entire history locally. Changes are tracked as a series of **commits**, each with a unique SHA hash.

### Key Concepts

| Term | Definition |
|---|---|
| **Repository (repo)** | A directory tracked by Git, containing files and full change history |
| **Commit** | A snapshot of changes at a point in time, with a message |
| **Branch** | An independent line of development |
| **Remote** | A copy of the repository hosted elsewhere (e.g., GitHub) |
| **Clone** | A full local copy of a remote repository |
| **Stage / Index** | The area where changes are prepared before committing |
| **Working directory** | Your actual files on disk |

### The Three Areas of Git

```
Working Directory → Staging Area (Index) → Repository (.git)
       │                    │                      │
   (edit files)         (git add)             (git commit)
```

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Core Git Commands

### Initial Setup

```bash
# Configure your identity (do this once)
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "vim"
git config --global init.defaultBranch main

# View config
git config --list
```

### Starting a Repository

```bash
git init                    # Initialize a new repo in current directory
git clone https://github.com/user/repo.git   # Clone a remote repo
git clone https://... mydir  # Clone into a specific directory
```

### The Daily Workflow

```bash
git status                  # See what's changed
git diff                    # See unstaged changes
git diff --staged           # See staged changes

git add file.txt            # Stage a specific file
git add .                   # Stage all changes
git add -p                  # Interactively stage chunks (powerful!)

git commit -m "Add nginx config for production"   # Commit with message
git commit --amend          # Amend the last commit (before pushing!)

git log                     # Full commit history
git log --oneline           # Compact one-line history
git log --oneline --graph --all   # Visual branch graph
git show abc1234            # Show details of a specific commit
```

### Undoing Changes

```bash
git restore file.txt            # Discard unstaged changes to a file
git restore --staged file.txt   # Unstage a file (keep changes)
git revert abc1234              # Create a new commit that undoes a previous commit
git reset --soft HEAD~1         # Undo last commit, keep changes staged
git reset --mixed HEAD~1        # Undo last commit, keep changes unstaged
git reset --hard HEAD~1         # Undo last commit and discard all changes (dangerous!)
```

> ⚠️ **Warning**: `git reset --hard` permanently discards uncommitted changes. Never use `--hard` on commits that have already been pushed to a shared remote.

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Branching & Merging

Branches let you develop features in isolation without affecting the main codebase.

```bash
git branch                          # List local branches
git branch -a                       # List all branches (including remote)
git branch feature/add-login        # Create a new branch
git switch feature/add-login        # Switch to a branch
git switch -c feature/add-login     # Create AND switch in one command

git merge feature/add-login         # Merge feature branch into current branch
git merge --no-ff feature/add-login # Merge with a merge commit (keeps history clean)

git branch -d feature/add-login     # Delete a branch (after merging)
git branch -D feature/add-login     # Force delete (even if not merged)
```

### Merge Strategies

| Strategy | Command | When to Use |
|---|---|---|
| **Fast-forward** | `git merge` | Linear history, no divergence |
| **Merge commit** | `git merge --no-ff` | Preserve branch history |
| **Squash** | `git merge --squash` | Combine all branch commits into one |
| **Rebase** | `git rebase main` | Linear history, advanced use |

[↑ Back to TOC](#table-of-contents)

---

## Beginner: Working with Remote Repositories

```bash
git remote -v                       # List remotes
git remote add origin https://github.com/user/repo.git   # Add a remote

git push origin main                # Push local main to remote
git push origin feature/my-feature # Push a feature branch
git push -u origin main             # Set upstream and push
git push --tags                     # Push tags

git pull                            # Fetch + merge from remote
git pull --rebase                   # Fetch + rebase (cleaner history)
git fetch origin                    # Download remote changes without merging
git fetch --prune                   # Remove local refs to deleted remote branches

# Pull Request / Merge Request workflow
# 1. Create a branch: git switch -c feature/my-feature
# 2. Make commits
# 3. Push: git push -u origin feature/my-feature
# 4. Open PR on GitHub/GitLab
# 5. Request code review
# 6. Merge via the web UI
# 7. Delete the branch: git push origin --delete feature/my-feature
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Resolving Conflicts

Conflicts happen when two branches modify the same lines of the same file.

```bash
git merge feature/login
# AUTO-MERGING FAILED
# CONFLICT (content): Merge conflict in app/config.py

# Open the conflicted file — it looks like this:
<<<<<<< HEAD
DATABASE_HOST = "production-db.example.com"
=======
DATABASE_HOST = "dev-db.example.com"
>>>>>>> feature/login

# 1. Edit the file to the correct final state
# 2. Remove the conflict markers
# 3. Stage the resolved file
git add app/config.py
# 4. Complete the merge
git commit -m "Merge feature/login — resolved config conflict"

# Abort a merge if things go wrong
git merge --abort

# Use a visual merge tool
git mergetool     # Opens configured tool (vimdiff, meld, etc.)
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Git Workflows

### GitHub Flow (Simple — recommended for most teams)

```
main (always deployable)
  ├── feature/add-login     ← branch, PR, merge, delete
  ├── fix/crash-on-logout   ← branch, PR, merge, delete
  └── feature/new-dashboard ← branch, PR, merge, delete
```

1. `main` is always deployable to production
2. Create a feature branch from `main`
3. Commit and push to the branch
4. Open a Pull Request
5. Code review → merge → delete branch

### Git Flow (Enterprise — complex projects)

```
main         ← production releases only
develop      ← integration branch
  ├── feature/*   ← new features
  ├── release/*   ← pre-release stabilization
  └── hotfix/*    ← emergency production fixes
```

### Trunk-Based Development (CI/CD optimized)

- All developers commit directly to `main` (or short-lived branches < 1 day)
- Feature flags hide incomplete features
- Fast CI pipeline runs on every commit
- Requires strong test coverage

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Advanced Git Techniques

### Stash — Save Work Temporarily

```bash
git stash                   # Stash current changes
git stash push -m "WIP: login feature"  # Stash with a name
git stash list              # List all stashes
git stash pop               # Apply and remove top stash
git stash apply stash@{1}   # Apply a specific stash without removing
git stash drop stash@{1}    # Delete a specific stash
git stash clear             # Delete all stashes
```

### Rebase — Rewrite History

```bash
# Update your branch with the latest main (cleaner than merge)
git switch feature/my-feature
git rebase main

# Interactive rebase — squash, reorder, edit commits
git rebase -i HEAD~5        # Rebase last 5 commits interactively
# Commands in interactive rebase:
# pick   = keep commit as-is
# reword = keep but edit message
# squash = combine with previous commit
# fixup  = combine and discard message
# drop   = remove commit entirely
```

### Cherry-Pick — Apply a Specific Commit

```bash
git cherry-pick abc1234     # Apply commit abc1234 to current branch
git cherry-pick abc1234 def5678  # Apply multiple commits
git cherry-pick --no-commit abc1234  # Apply changes without committing
```

### Bisect — Find a Bug by Binary Search

```bash
git bisect start
git bisect bad                  # Current commit is broken
git bisect good v1.0.0          # v1.0.0 was working
# Git checks out a commit in the middle
# Test it — run your tests or reproduce the bug
git bisect good                 # This commit is good
git bisect bad                  # This commit is bad
# Git narrows down — repeat until found
git bisect reset                # Exit bisect mode
```

### Tags — Mark Releases

```bash
git tag v1.0.0                          # Lightweight tag
git tag -a v1.0.0 -m "Release v1.0.0"  # Annotated tag (preferred)
git tag                                 # List all tags
git push origin v1.0.0                  # Push a tag
git push origin --tags                  # Push all tags
git checkout v1.0.0                     # Checkout a tag (detached HEAD)
```

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Git Hooks

Git hooks are scripts that run automatically at specific points in the Git workflow. They live in `.git/hooks/` and must be executable.

### Useful Hook Examples

```bash
# .git/hooks/pre-commit — runs before every commit
#!/bin/bash
set -e

echo "Running pre-commit checks..."

# Run linter
if command -v shellcheck &>/dev/null; then
    find . -name "*.sh" -exec shellcheck {} \;
fi

# Prevent committing to main directly
BRANCH=$(git symbolic-ref HEAD 2>/dev/null | cut -d/ -f3)
if [ "$BRANCH" = "main" ]; then
    echo "ERROR: Direct commits to main are not allowed!"
    exit 1
fi

echo "Pre-commit checks passed."
```

```bash
# .git/hooks/commit-msg — enforce commit message format
#!/bin/bash
COMMIT_MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore|ci)(\(.+\))?: .{1,100}"

if ! echo "$COMMIT_MSG" | grep -qE "$PATTERN"; then
    echo "ERROR: Commit message must follow Conventional Commits format"
    echo "Example: feat(auth): add JWT token refresh"
    exit 1
fi
```

### Share Hooks with Your Team (use a tool)

- `.git/hooks/` is not tracked by Git
- Use [pre-commit](https://pre-commit.com/) framework to share hooks via `.pre-commit-config.yaml`
- Husky (Node.js projects) for shareable hooks in `package.json`

[↑ Back to TOC](#table-of-contents)

---

## Intermediate: Git for Infrastructure Code

### .gitignore for DevOps Repos

```gitignore
# Terraform / OpenTofu
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl

# Ansible
*.retry
vault_password.txt

# General secrets
.env
.env.local
*.pem
*.key
*_rsa
secrets/
credentials.json

# Editor
.vscode/
.idea/
*.swp
```

### Git for Kubernetes Manifests

```
infra/
├── kubernetes/
│   ├── production/
│   │   ├── deployments/
│   │   └── services/
│   └── staging/
├── terraform/
│   ├── modules/
│   └── environments/
└── ansible/
    ├── playbooks/
    └── roles/
```

- Every change to infrastructure goes through a Pull Request
- CI pipeline validates manifests (`kubectl dry-run`, `terraform validate`)
- Changes are reviewed before merging — this is GitOps

[↑ Back to TOC](#table-of-contents)

---

## Tools & Commands Reference

| Command | Purpose |
|---|---|
| `git init` / `git clone` | Start a repository |
| `git add` / `git commit` | Stage and commit changes |
| `git status` / `git diff` | Inspect working state |
| `git log --oneline --graph` | View branch history visually |
| `git branch` / `git switch` | Manage and navigate branches |
| `git merge` / `git rebase` | Integrate branch changes |
| `git push` / `git pull` | Sync with remote |
| `git stash` | Temporarily save work |
| `git cherry-pick` | Apply a specific commit |
| `git bisect` | Binary-search for a bug |
| `git tag` | Mark release points |
| `git revert` | Safely undo a commit |
| `git hook` scripts | Automate pre-commit checks |

[↑ Back to TOC](#table-of-contents)

---

## Hands-On Labs

### Lab 4.1 — First Repository

1. Create a new directory and initialize a Git repository
2. Create a `README.md` file and commit it
3. View the commit with `git log` and `git show`
4. Make a change, view it with `git diff`, stage and commit

### Lab 4.2 — Branching & Merging

1. Create a branch: `git switch -c feature/add-config`
2. Create a file `config.yaml` and commit it
3. Switch back to `main`
4. Make a different change on `main` and commit it
5. Merge the feature branch: `git merge --no-ff feature/add-config`
6. View the merge history: `git log --oneline --graph`

### Lab 4.3 — Conflict Resolution

1. Create two branches from the same base commit
2. Edit the same line of the same file in both branches
3. Attempt to merge — observe the conflict
4. Resolve the conflict manually
5. Complete the merge with a descriptive commit message

### Lab 4.4 — GitHub Workflow

1. Create a repo on GitHub
2. Push your local repo to it
3. Create a feature branch, make changes, push the branch
4. Open a Pull Request
5. Review and merge via the GitHub UI
6. Pull the changes locally: `git pull`

### Lab 4.5 — Git Hook

1. Write a `pre-commit` hook that prevents committing `.env` files
2. Test it by attempting to commit a `.env` file — the hook should reject it
3. Test that normal commits still work

[↑ Back to TOC](#table-of-contents)

---

## Further Reading

- [Pro Git (free book)](https://git-scm.com/book/en/v2) — Scott Chacon
- [Oh Shit, Git!](https://ohshitgit.com/) — Plain English solutions to common mistakes
- [Conventional Commits](https://www.conventionalcommits.org/) — Commit message standard
- [GitHub Flow Guide](https://docs.github.com/en/get-started/quickstart/github-flow)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)
- [Glossary: Branch](./glossary.md#b), [Fork](./glossary.md#f), [Pull Request](./glossary.md#p), [Repository](./glossary.md#r)

[↑ Back to TOC](#table-of-contents)

---

*© 2026 UncleJS — Licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)*
