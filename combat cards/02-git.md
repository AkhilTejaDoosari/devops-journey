# 🌿 GIT_CORE — Combat Card
> **80/20 Rule:** The daily loop + branching + one safe undo per situation.   
> **Pre-shift scan:** Daily loop → Branch workflow → Undo decision table.   
> **Source:** [02. Git](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/README.md)   
---

## 1. THE DAILY LOOP — Run This Every Single Day
> **Source:** [01-foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/01-foundations/README.md)   
> Muscle memory. You should be able to type this in the dark.

```
git status  →  git add .  →  git commit -m "type: message"  →  git push
```

| Command | What it does |
|---|---|
| `git status` | What changed? What's staged? Run before every `add` |
| `git add .` | Stage all changes in current directory |
| `git add <file>` | Stage one specific file only |
| `git restore --staged <file>` | Unstage a file — oops, wrong one |
| `git commit -m "type: message"` | Commit with a message |
| `git log --oneline` | Compact history — where am I in the timeline? |
| `git push` | Push commits to remote (origin) |
| `git pull` | Fetch + merge — sync before you start new work |

**Commit message types:** `feat:` `fix:` `docs:` `chore:` `refactor:`

**ShopStack context:**
```bash
# Start of every session
cd ~/shopstack
git pull
git status

# End of every session
git add .
git commit -m "feat: add webstore-db volume config"
git push
```

---

## 2. BRANCHING — Feature Work Never Touches Main
> **Source:** [03-history-branching](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/03-history-branching/README.md)   
> Every piece of work gets its own branch. Main is always clean.

| Command | What it does |
|---|---|
| `git switch -c feature/<name>` | Create AND switch to a new branch |
| `git switch <branch>` | Switch to an existing branch |
| `git branch` | List local branches — which one am I on? |
| `git branch -a` | List all branches including remote |
| `git merge <branch>` | Merge a branch into your current one |
| `git push -u origin <branch>` | Push branch to remote and set upstream |
| `git branch -d <branch>` | Delete a local branch after merging |

**The feature branch flow:**
```
git switch -c feature/add-volume-config    ← create branch
  ... do work ...
git add . && git commit -m "feat: ..."     ← commit on branch
git push -u origin feature/add-volume-config  ← push to remote
→ open PR on GitHub → review → merge to main
git switch main && git pull                ← sync main after merge
git branch -d feature/add-volume-config   ← clean up local branch
```

**ShopStack context:** Branch name format: `feature/shopstack-<what-you-changed>`. Never commit infra changes directly to main.

---

## 3. STASH — Park Work Without Committing
> **Source:** [02-stash-tags](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/02-stash-tags/README.md)   
> Urgent fix needed? Not ready to commit? Use stash.

| Command | What it does |
|---|---|
| `git stash push -m "WIP: description"` | Save in-progress work with a label |
| `git stash -u` | Stash including new untracked files |
| `git stash list` | See everything on the stash stack |
| `git stash pop` | Restore most recent stash and remove from stack |
| `git stash show -p` | Show full diff of most recent stash |

**The stash workflow:**
```
git stash push -m "WIP: updating db config"   ← park your work
git switch main                                ← fix the urgent thing
git add . && git commit -m "fix: ..."
git push
git switch feature/my-branch                  ← come back
git stash pop                                  ← restore exactly where you were
```

---

## 4. UNDO — The Decision Table
> **Source:** [05-undo-recovery](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/05-undo-recovery/README.md)   
> Before you type any recovery command: **has this been pushed?**

```
Not pushed  →  you can rewrite history
Pushed      →  you cannot rewrite shared history — use revert
```

| Situation | Command | Safe? |
|---|---|---|
| Discard changes in working directory | `git restore <file>` | ✅ Always safe |
| Unstage a file | `git restore --staged <file>` | ✅ Always safe |
| Fix the last commit message (not pushed) | `git commit --amend` | ✅ Local only |
| Undo last commit, keep the changes staged | `git reset --soft HEAD~1` | ✅ Local only |
| Undo last commit, keep changes unstaged | `git reset --mixed HEAD~1` | ✅ Local only |
| Undo a commit that was already pushed | `git revert <hash>` | ✅ Safe for shared history |
| Recover lost commits after a bad reset | `git reflog` then `git checkout <hash>` | ✅ Lifeline |

**The golden rule:** If it's pushed → `revert`. If it's local → `reset`. When in doubt → `reflog`.

---

## 5. INSPECT — Read The State
> **Source:** [01-foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/01-foundations/README.md)   

| Command | What it does |
|---|---|
| `git diff` | Changes in working dir vs staging |
| `git diff --staged` | Changes staged vs last commit |
| `git log --oneline --graph` | Visual branch history |
| `git show <hash>` | Full diff of a specific commit |
| `git remote -v` | Show all remotes (origin, upstream) |
| `git branch -vv` | Branches with their tracking remotes |

---

## ⚡ MASTER ONE-LINERS

```bash
# Start of session — always
git pull && git status

# Daily commit loop
git add . && git commit -m "fix: message" && git push

# Oops, wrong commit message (not pushed)
git commit --amend

# Oops, committed to main by accident (not pushed)
git reset --soft HEAD~1
git switch -c feature/correct-branch
git add . && git commit -m "..."

# See exactly what changed last commit
git show --stat HEAD
```

---

> **Own this page. The rest is details.**
