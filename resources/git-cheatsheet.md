# Git Cheatsheet

## Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

---

## Starting a Repository

```bash
git init                          # Initialise a new local repo
git clone <url>                   # Clone a remote repo locally
git clone <url> my-folder         # Clone into a specific folder
```

---

## Core Workflow

```bash
git status                        # Show working tree status
git add file.txt                  # Stage a specific file
git add .                         # Stage all changes
git add -p                        # Stage changes interactively (chunk by chunk)
git commit -m "message"           # Commit staged changes
git commit --amend                # Edit the last commit (before pushing)
git push origin main              # Push to remote
git pull origin main              # Fetch and merge from remote
```

---

## Branching

```bash
git branch                        # List local branches
git branch -a                     # List all branches (local + remote)
git branch feature/login          # Create a new branch
git checkout feature/login        # Switch to a branch
git checkout -b feature/login     # Create and switch in one step
git switch feature/login          # Modern alternative to checkout
git switch -c feature/login       # Create and switch (modern)
git branch -d feature/login       # Delete a merged branch
git branch -D feature/login       # Force delete (even if not merged)
```

---

## Merging and Rebasing

```bash
git merge feature/login           # Merge a branch into current branch
git merge --no-ff feature/login   # Merge with a merge commit (no fast-forward)
git rebase main                   # Rebase current branch onto main
git rebase --abort                # Abort a rebase in progress
git rebase --continue             # Continue rebase after resolving conflicts
```

**When to use which:**
- `merge` — preserves full history, good for long-lived branches
- `rebase` — creates a clean linear history, good for feature branches before merging

---

## Viewing History

```bash
git log                           # Full commit history
git log --oneline                 # Compact one-line history
git log --oneline --graph         # Branch graph view
git log -n 10                     # Last 10 commits
git log --author="Name"           # Commits by a specific author
git diff                          # Unstaged changes
git diff --staged                 # Staged changes (about to be committed)
git diff main..feature/login      # Difference between two branches
git show abc1234                  # Show a specific commit
```

---

## Undoing Changes

```bash
git restore file.txt              # Discard unstaged changes to a file
git restore --staged file.txt     # Unstage a file (keep changes in working tree)
git reset HEAD~1                  # Undo last commit, keep changes staged
git reset --hard HEAD~1           # Undo last commit and discard all changes
git revert abc1234                # Create a new commit that undoes a past commit
```

**Safe rule:** Use `revert` on shared branches. Use `reset` only on local branches.

---

## Remote Operations

```bash
git remote -v                     # List remotes
git remote add origin <url>       # Add a remote called origin
git fetch origin                  # Download changes without merging
git pull origin main              # Fetch and merge
git push origin main              # Push local commits to remote
git push -u origin feature/login  # Push and set upstream tracking
git push origin --delete branch   # Delete a remote branch
```

---

## Stashing

```bash
git stash                         # Save uncommitted changes to stash
git stash save "WIP: login form"  # Stash with a label
git stash list                    # List all stashes
git stash pop                     # Apply and remove the most recent stash
git stash apply stash@{1}         # Apply a specific stash without removing it
git stash drop stash@{1}          # Delete a specific stash
git stash clear                   # Delete all stashes
```

---

## Tags

```bash
git tag                           # List all tags
git tag v1.0.0                    # Create a lightweight tag
git tag -a v1.0.0 -m "Release"   # Create an annotated tag (preferred)
git push origin v1.0.0            # Push a specific tag
git push origin --tags            # Push all tags
git tag -d v1.0.0                 # Delete a local tag
```

---

## .gitignore

```bash
# Ignore by extension
*.log
*.env
*.pem

# Ignore a folder
node_modules/
.terraform/
__pycache__/

# Ignore a specific file
terraform.tfvars
secrets.json

# Ignore everything except specific files
*
!.gitignore
!requirements.txt
```

```bash
# Check what is currently ignored
git check-ignore -v filename

# Force add an ignored file (use sparingly)
git add -f filename
```

---

## Useful One-Liners

```bash
# DANGER: Permanently destroys all uncommitted changes and untracked files. No undo.
# Only use this if you are absolutely sure you want to discard everything.
git reset --hard && git clean -fd

# See which branches have been merged into main
git branch --merged main

# Find which commit introduced a bug (binary search)
git bisect start
git bisect bad                   # Current commit is broken
git bisect good v1.0.0           # This tag was working
# Git checks out commits for you to test until it finds the culprit
git bisect reset                 # Done

# Clean up remote-tracking references to deleted branches
git fetch --prune

# Show contributors sorted by number of commits
git shortlog -sn
```

---

## Commit Message Best Practices

```
<type>: <short summary (50 chars max)>

<optional body — explain why, not what (72 chars per line)>
```

**Types:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

**Good:** `fix: prevent race condition in session token refresh`
**Bad:** `fixed stuff`, `update`, `WIP`
