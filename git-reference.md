# Git Operations Reference

## 0. Configure Username and Email

Git requires a name and email to associate with your commits.

**Set globally (applies to all repositories on your machine):**
```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**Set locally (applies only to the current repository):**
```bash
git config user.name "Your Name"
git config user.email "you@example.com"
```

**Verify your configuration:**
```bash
git config --global user.name
git config --global user.email
```

**View all config settings:**
```bash
git config --list
```

> Local config overrides global config for that specific repository.

---

## 1. Adding a Repository

**Initialize a new local repository:**
```bash
git init
```

**Clone an existing remote repository:**
```bash
git clone <repository-url>
git clone <repository-url> <target-folder>   # clone into a specific folder
```

**Connect a local repo to a remote:**
```bash
git remote add origin <repository-url>
```

**Verify remote connection:**
```bash
git remote -v
```

---

## 2. Creating a New Branch

**Create a new branch:**
```bash
git branch <branch-name>
```

**Create and switch to a new branch in one step:**
```bash
git checkout -b <branch-name>
# or (modern syntax)
git switch -c <branch-name>
```

**Switch to an existing branch:**
```bash
git checkout <branch-name>
# or
git switch <branch-name>
```

**List all branches:**
```bash
git branch          # local branches
git branch -a       # local and remote branches
```

**Delete a branch:**
```bash
git branch -d <branch-name>   # safe delete (only if merged)
git branch -D <branch-name>   # force delete
```

---

## 3. Push

**Stage and commit changes before pushing:**
```bash
git add .                        # stage all changes
git add <file>                   # stage a specific file
git commit -m "your message"     # commit staged changes
```

**Push to remote:**
```bash
git push origin <branch-name>
```

**Push and set upstream (first-time push for a branch):**
```bash
git push -u origin <branch-name>
```

**After setting upstream, subsequent pushes can use:**
```bash
git push
```

**Force push (use with caution):**
```bash
git push --force origin <branch-name>
```

---

## 4. Pull

**Pull latest changes from remote (fetch + merge):**
```bash
git pull origin <branch-name>
```

**Pull with rebase instead of merge:**
```bash
git pull --rebase origin <branch-name>
```

**Fetch without merging (inspect before applying):**
```bash
git fetch origin
git diff origin/<branch-name>    # review changes
git merge origin/<branch-name>   # apply when ready
```

---

## Quick Reference

| Task | Command |
|---|---|
| Set username (global) | `git config --global user.name "Name"` |
| Set email (global) | `git config --global user.email "email"` |
| View config | `git config --list` |
| Init repo | `git init` |
| Clone repo | `git clone <url>` |
| Add remote | `git remote add origin <url>` |
| New branch | `git checkout -b <branch>` |
| Stage all | `git add .` |
| Commit | `git commit -m "message"` |
| Push (first time) | `git push -u origin <branch>` |
| Push | `git push` |
| Pull | `git pull` |
| List branches | `git branch -a` |
