[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Git

**Hire importance at $25–40/hr:** 8/10

**One line:** Every pipeline starts with a push. Every deployment traces back to a commit. No Git fluency — no job.

📚 Full runbook → [Git & GitHub – Version Control](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/README.md)

---

## 0 — Before You Open A Terminal

### What a developer does that makes Git necessary

A developer writes code. They change files. They break things and fix them. Without Git, there is no record of what changed, who changed it, or how to go back. Git records every change as a permanent snapshot — a commit. Every commit has a message, a timestamp, an author, and a unique fingerprint called a SHA hash.

In a team, multiple developers work on the same codebase simultaneously. Git manages that — branching, merging, conflict resolution. Without it, two developers editing the same file at the same time overwrite each other's work.

### Why every DevOps tool depends on Git

Git is not just version control. It is the source of truth that every other tool in your stack reads from.

GitHub Actions triggers on a Git commit — a push to main starts your CI pipeline. Docker images are tagged with Git commit SHAs so you always know which code is inside the image. ArgoCD watches a Git repo and deploys whatever manifest is in it — that is GitOps. Terraform state is version controlled. Everything downstream depends on what is in the repo.

### The three zones — the mental model that makes every command make sense

```
Working Directory → Staging Area → Local Repo → Remote (GitHub)
      edit             git add       git commit      git push

◄── git restore --staged  ◄── git reset      ◄── git fetch / pull
```

Every Git command moves data between these four zones. Know where data lives — every command becomes logical. This is not a detail. This is the entire mental model.

### ShopStack files Git touches

```
~/shopstack/                         ← the entire repo lives here
~/shopstack/.gitignore               ← what Git never sees
~/shopstack/infra/docker-compose.yml ← every change tracked
~/shopstack/services/                ← app code — every commit records changes
.github/workflows/                   ← CI pipeline triggered by every push to main
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- A commit is a snapshot — not a diff, a complete picture of the project at that moment
- The three zones — working directory, staging area, local repo — and what moves between them
- Why the staging area exists — you control exactly what each commit contains
- `HEAD` is a pointer to where you currently are — it moves when you commit or switch branches
- A branch is a lightweight pointer to a commit — not a copy of the code
- Merge vs revert vs reset — what each one does and when to use each
- `git push` requires a remote — `git commit` is only local
- `.gitignore` must exist before the first commit — secrets in history are permanent
- `git stash` saves work in progress without a commit
- A merge conflict is not an error — it is Git asking you to decide

**When you can explain all ten without hesitation — stop. Move to Networking.**

### What unlocks higher pay

At $55/hr you add: rebase for linear history, cherry-pick for selective commits, git hooks for automation, signed commits for security, advanced branching strategies like trunk-based development, and Git in CI/CD pipelines — understanding exactly why a pipeline triggers and how commit SHAs trace to deployed images.

📚 Deep dive → [Git & GitHub Runbook](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/README.md)

---

## 2 — The Bag

### Own These Cold

A commit is a complete snapshot of the project — not a diff. Every commit has a SHA hash, a message, an author, and a pointer to its parent. This chain of snapshots is your entire history.

The three zones are everything. Working directory is where you edit. Staging area is where you choose what goes into the next commit. Local repo is committed history. Remote is GitHub. Every Git command moves data between these zones. Know where data is and every command makes sense.

The staging area exists so you control exactly what each commit contains. You edited five files but only three are ready — you stage those three and commit them as one logical change. Without staging, every commit would include everything you touched.

`HEAD` is a pointer to where you currently are. When you commit, HEAD's branch moves forward. When you switch branches, HEAD moves to that branch. When HEAD is detached, you checked out a commit directly — switch back to a branch immediately.

A branch is just a pointer to a commit. When you create a branch, Git creates a new pointer at your current commit. When you commit on that branch, only that pointer moves. `main` stays where it was. Branches are cheap — create them freely.

`.gitignore` must be created before the first `git add .`. Git history is immutable. If you commit a secret, it exists in history permanently even after you delete the file. Anyone who clones the repo can find it. Create `.gitignore` first. Always.

`git revert` creates a new commit that undoes a previous one — safe because it does not rewrite history. `git reset` moves HEAD backward — dangerous on pushed commits because it rewrites history. Use revert for pushed commits. Use reset only for local unpushed work.

📚 Deep dive → [Git Foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/01-foundations/README.md)

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI.

- Need to save work without committing → `git stash`
- Need to undo a pushed commit safely → `git revert`
- Need to undo a local unpushed commit → `git reset`
- Need to work on a feature without touching main → `git branch` + `git switch`
- Need to bring feature work into main → `git merge`
- Need to find what changed and when → `git log` + `git diff`
- Need to recover something deleted → `git reflog`
- CI pipeline is not triggering → check if push actually went to the right remote branch

---

### Domain Awareness

- Rebase — rewrites commit history to appear linear — never on pushed commits
- Cherry-pick — apply a single commit from one branch to another
- Git hooks — scripts that run automatically on git events like commit or push
- Signed commits — GPG verification that you are who you say you are
- Trunk-based development — everyone merges to main frequently, feature flags replace long-lived branches

---

### The one insight that separates copiers from understanders

A commit is a snapshot, not a diff.

Once you know this, `git reset --hard`, `git stash`, `git revert`, and merge conflicts all become logical operations on snapshots — not scary magic. You are always moving between complete pictures of the project. Nothing is lost until you explicitly destroy it. Even then, `git reflog` keeps a record for 90 days.

---

## 3 — Combat Sheet

The combat sheet is not a memorisation list. It is your pressure-proof reference. When something goes wrong — open this, find the situation, run the command.

---

### Daily Drivers

These are the commands you type every single working day. Drill these until your hands type them without thinking.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Don't know what changed or where I am | `git status` | Run before every add and commit |
| Need to save a snapshot of work | `git add -A && git commit -m "type: message"` | `git commit -m "fix: api health check port"` |
| Need to send commits to GitHub | `git push origin main` | After every working commit on ShopStack |
| Need latest changes from GitHub | `git pull origin main` | Before starting work every morning |
| Need to read recent history | `git log --oneline` | See last 10 commits on ShopStack |
| Need to work on a feature in isolation | `git switch -c feature/name` | `git switch -c feat/healthcheck-api` |

---

### Situational

You recognise the situation. You open this table. You find the command. You close it.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Mid-feature — urgent fix needed | `git stash` then fix then `git stash pop` | Stash compose changes while fixing a critical bug |
| Committed wrong message or forgot a file | `git commit --amend` | Only before pushing — never after |
| Need to undo a pushed commit safely | `git revert HEAD` | Revert a bad docker-compose change |
| Need to undo a local unpushed commit | `git reset --soft HEAD~1` | Keep changes staged, undo the commit |
| Merge conflict after `git merge` | Edit file, remove markers, `git add`, `git commit` | Two people edited docker-compose.yml |
| Need to see exact changes in a commit | `git show HASH` | `git show a1b2c3d` |
| Need to compare local vs remote | `git diff main origin/main` | Is remote ahead of local |
| Accidentally deleted a branch | `git reflog` then `git branch NAME HASH` | Recover deleted feature branch |
| Need to clone ShopStack on new EC2 | `git clone https://github.com/AkhilTejaDoosari/shopstack` | Fresh EC2 setup |
| Push rejected — non-fast-forward | `git pull origin main` first then push | Someone else pushed while you were working |
| Need to see all branches | `git branch -a` | See local and remote branches |
| Need to delete a merged branch | `git branch -d branch-name` | Clean up after PR merge |

---

## 4 — ShopStack Map

| Situation | Command or File | Knowledge Needed |
|---|---|---|
| Save docker-compose.yml changes | `git add infra/docker-compose.yml && git commit -m "fix: port mapping"` | Stage specific file, not everything |
| Roll back a bad Dockerfile change | `git revert HEAD` | Revert vs reset — revert is safe after push |
| Create branch to test new healthcheck | `git switch -c feat/healthcheck-api` | Branch creation, isolation |
| See what changed in nginx.conf last week | `git log --oneline services/frontend/nginx.conf` | git log with path filter |
| Push Dockerfile change to trigger CI build | `git push origin main` | Push triggers GitHub Actions |
| Clone ShopStack onto a new EC2 | `git clone https://github.com/AkhilTejaDoosari/shopstack` | Clone, not pull |
| Check if remote has newer changes | `git fetch && git diff main origin/main` | fetch vs pull |
| Find who last changed docker-compose.yml | `git log --oneline infra/docker-compose.yml` | Git log with file path |
| Secret accidentally committed | `git rm --cached .env` + add to `.gitignore` + new commit + rotate the secret | gitignore, cached tracking |
| CI pipeline not triggering | Confirm push went to correct branch on remote with `git log --oneline origin/main` | Remote tracking |

📚 Deep dive → [Git Undo & Recovery](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/05-undo-recovery/README.md)

---

## 5 — Peace of Mind

> Answer every question out loud before opening the toggle.

> If you stumble — go to the terminal. Not back to the notes.

---

**Q1. What is the difference between `git add` and `git commit`? Why do both exist?**

<details>
<summary>Answer</summary>

`git add` moves changes from the working directory into the staging area — you are choosing what goes into the next snapshot. `git commit` seals the staging area as a permanent snapshot in the local repo.

Both exist because the staging area gives you control. You edited five files but only three are ready — you stage those three and commit them as one logical change. Without `git add`, every commit would include everything you touched since the last commit, which makes history messy and hard to read.

</details>

---

**Q2. You committed to `main` directly and pushed a broken change. How do you undo it safely?**

<details>
<summary>Answer</summary>

`git revert HEAD` creates a new commit that undoes the last commit. It does not rewrite history — it adds a new snapshot that is the inverse of the bad one. This is safe after pushing because teammates who already pulled are not affected.

I never use `git reset` on a pushed commit because reset rewrites history. If teammates already pulled those commits, their local history diverges from the remote and they have problems.

</details>

---

**Q3. What does `git stash` do and when do you use it in real work?**

<details>
<summary>Answer</summary>

`git stash` saves your uncommitted changes temporarily and returns the working directory to the last commit state. The stash is a stack — you can push multiple stashes and pop them back in order.

Real situation: I am mid-way through changing `docker-compose.yml` to add a new service. My teammate says the API health check is broken in production — fix it now. I run `git stash` to save my in-progress work, fix the health check, commit and push, then `git stash pop` to bring my original work back. No half-finished changes mixed into the production fix.

</details>

---

**Q4. What is a merge conflict and how do you resolve one?**

<details>
<summary>Answer</summary>

A merge conflict happens when two branches modified the same lines in the same file and Git cannot automatically decide which version to keep. It is not an error — it is Git asking you to make the decision.

Git marks the conflict in the file with `<<<<<<<`, `=======`, and `>>>>>>>` markers. Above the `=======` is what is on the current branch. Below is what is coming in from the other branch. I open the file, read both versions, keep the correct one, remove all three markers, run `git add` on the resolved file, then `git commit` to complete the merge.

</details>

---

**Q5. What is the difference between `git fetch` and `git pull`?**

<details>
<summary>Answer</summary>

`git fetch` downloads changes from the remote but does not touch your working directory or local branches. It just updates your knowledge of what is on the remote. You can inspect what changed before deciding to merge.

`git pull` is `git fetch` followed by `git merge` in one step. It downloads and immediately merges into your current branch.

I use `git fetch` first in sensitive situations — `git fetch && git diff main origin/main` shows me exactly what would change before I commit to the merge. `git pull` is fine for daily work when I know the remote is safe.

</details>

---

**Q6. You deleted a feature branch by accident after working on it for two days. Is the work gone?**

<details>
<summary>Answer</summary>

No. `git reflog` shows the full history of every HEAD movement — every commit, every branch switch, every reset. The commits from the deleted branch are still in the local repo for 90 days. I run `git reflog` to find the hash of the last commit on the deleted branch, then `git branch recovery-branch HASH` to recreate the branch pointing at that commit. All work is recovered.

</details>

---

**Q7. What does `HEAD` mean in Git?**

<details>
<summary>Answer</summary>

HEAD is a pointer to where I currently am in the repo. Normally it points to the tip of the current branch — the latest commit. When I commit, HEAD's branch moves forward automatically. When I switch branches, HEAD moves to that branch.

Detached HEAD means HEAD is pointing directly at a commit instead of a branch. This happens when I checkout a commit hash directly. Any commits I make in this state are not on any branch — they become orphaned when I switch away. I get back by running `git switch main` to reattach HEAD to a branch.

</details>

---

**Q8. A teammate says "the CI pipeline did not trigger on your push." What do you check?**

<details>
<summary>Answer</summary>

First: `git log --oneline origin/main` — did the push actually reach the remote on the right branch. If it shows my commit, the push succeeded.

Second: check the GitHub Actions workflow trigger — does it fire on `push` to `main` or a different branch. If I pushed to a feature branch but the workflow only triggers on main, that is the cause.

Third: check the `.github/workflows/` file for syntax errors — a broken YAML file silently prevents the workflow from running.

</details>

---

**Q9. What is the difference between `git reset --soft` and `git reset --hard`?**

<details>
<summary>Answer</summary>

Both move HEAD backward to a previous commit — undoing commits.

`git reset --soft HEAD~1` undoes the last commit but keeps all the changes staged. The files are exactly as they were — nothing is lost, just uncommitted. Good for when I committed too early and want to reorganise.

`git reset --hard HEAD~1` undoes the last commit and erases all changes from the working directory and staging area. The files revert to the state of the previous commit. This is destructive — uncommitted work is gone permanently.

I only use reset on commits that have not been pushed. On pushed commits I always use `git revert`.

</details>

---

**Q10. What goes in `.gitignore` and why must it be created before the first commit?**

<details>
<summary>Answer</summary>

Secrets and credentials like `.env` files and API keys. Build output directories like `dist/` and `build/`. Runtime data like `*.log` files. Dependencies like `node_modules/`. Infrastructure state files like `terraform.tfstate`.

It must be created before the first `git add .` because Git history is immutable. If a secret is committed, it exists in the history permanently even after the file is deleted. Anyone who clones the repo can access it by inspecting the commit history. The only fix after committing a secret is to rotate it — the history cannot be cleanly erased once pushed to a shared repo.

</details>

---

## 6 — AI Split

### Own this yourself — never outsource

- The three zones — working directory, staging area, local repo — and what every command does to them
- What a commit actually is — a snapshot, not a diff — this is tested in every interview
- Why `.gitignore` must exist before the first commit — consequences of getting this wrong
- `git revert` vs `git reset` — which is safe after pushing and which is not
- Reading `git log --oneline` and understanding what you are looking at
- What HEAD is and what detached HEAD means

### Use AI for this — guilt free

- Exact syntax for `git log` filter flags — `--since`, `--author`, `--grep`
- Writing a `.gitignore` file for a specific project type — Python, Node, Go
- Exact flags for `git stash` — `-u` for untracked, `--include-untracked`
- Setting up SSH key authentication for GitHub step by step
- Complex rebase scenarios — interactive rebase, squashing commits

### The right questions to ask AI for Git

```
"I committed a file that should be in .gitignore.
The commit has not been pushed yet.
How do I remove it from tracking without losing the file?"

"I need to undo my last 3 commits but keep all the changes staged.
What is the exact command?"

"I have this git log output: [paste output].
Which commit introduced the change to docker-compose.yml
and how do I see exactly what changed in that commit?"

"Write a .gitignore for a Python FastAPI project
that also has a .env file and a terraform/ directory."
```

📚 Full Git runbook → [Git & GitHub – Version Control](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/02.%20Git%20%26%20GitHub%20%E2%80%93%20Version%20Control/README.md)
