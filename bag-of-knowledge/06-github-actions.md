[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# GitHub Actions

**Hire importance at $25–40/hr:** 5/10 — basic pipeline only. Grows to 8/10 at $55/hr.

**One line:** Every code change that reaches production passes through a pipeline. GitHub Actions is where that pipeline lives for most teams.

> ⏳ Runbook notes not generated yet.
> Come back Day 14 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What a developer does that makes GitHub Actions necessary

A developer pushes code to GitHub. Before GitHub Actions, someone had to manually pull the latest code, build the Docker image, test it, and push it to Docker Hub. Every deployment was a manual checklist. Every team member had a different version of "how to deploy." Things broke because steps were skipped or done in the wrong order.

GitHub Actions automates that checklist. The moment a developer pushes to `main`, a workflow runs automatically — it checks out the code, builds the Docker image, runs tests, pushes the image to Docker Hub, and can trigger deployment. Nobody manually runs these steps. The pipeline is the checklist — consistent, repeatable, and recorded.

### The developer's contract with CI

A developer writes tests. They expect CI to run them on every push and block a merge if tests fail. They push code and expect an image to appear in Docker Hub tagged with the commit SHA. They write a `Dockerfile` and expect CI to build it correctly in a clean environment.

Your job is to wire that expectation into a workflow file. The workflow is infrastructure — it runs the developer's code in a controlled environment and produces the artifacts they need for deployment.

### The disposable runner mental model — the aha that prevents the biggest mistakes

A GitHub Actions runner is a disposable Linux VM that starts blank on every run.

Every run starts with a fresh Ubuntu machine. No files. No installed tools beyond what Ubuntu ships with. No memory of previous runs. When the job finishes — success or failure — the VM is destroyed.

This is why `actions/checkout@v4` is the first step in every workflow. Without it, the runner has no code — it is a blank machine. The step literally clones your repository onto the runner.

This is why caching matters. Without caching, every run reinstalls all dependencies from scratch. A 5-minute install on every commit. With caching, the first run is slow. Every subsequent run finds the cache and skips the install.

This is why secrets cannot live in the workflow file. The runner is ephemeral but logs are permanent. Secrets must live in GitHub repository settings and be injected as environment variables at runtime.

### ShopStack pipeline — what it does

```
Developer pushes to main
          ↓
GitHub Actions runner starts — blank Ubuntu VM
          ↓
actions/checkout — clones shopstack repo onto runner
          ↓
docker/login-action — authenticates to Docker Hub using secrets
          ↓
docker/build-push-action — builds API image, pushes to Docker Hub
          ↓
Image is now available at:
akhiltejadoosari/shopstack-api:COMMIT_SHA
          ↓
ArgoCD detects new image tag in manifest — syncs cluster
```

GitHub Actions built it. ArgoCD deployed it. They are not the same thing.

### ShopStack files GitHub Actions touches

```
.github/workflows/build.yml          ← the pipeline definition
shopstack/services/api/Dockerfile     ← what gets built
GitHub repo Settings → Secrets        ← DOCKER_USERNAME, DOCKER_PASSWORD
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- A workflow is a YAML file in `.github/workflows/` — it defines when and what runs
- A runner is a disposable Linux VM — it resets completely on every run
- `actions/checkout` is mandatory — without it the runner has no code
- Trigger events — `push`, `pull_request`, `workflow_dispatch` — what starts the workflow
- Jobs run on runners — steps run inside jobs — the hierarchy
- Secrets live in GitHub settings — never in the workflow file — referenced as `${{ secrets.NAME }}`
- Environment variables — `env:` block for workflow-level, `$GITHUB_ENV` for step-level
- `docker/login-action` authenticates to Docker Hub — runs before build and push
- `docker/build-push-action` builds and pushes in one step — replaces two manual commands
- Reading a failed job — GitHub Actions tab → click the run → click the failed step

**When you can explain all ten without hesitation — stop. Move to ArgoCD.**

### What unlocks higher pay

At $55/hr you add: multi-job pipelines with job dependencies, caching strategies for Docker layers and package managers, matrix builds for testing across multiple versions, environment promotion — build once deploy to staging then prod, reusable workflows shared across repositories, and self-hosted runners for private infrastructure.

📚 Deep dive → Coming Day 14 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

A workflow file lives at `.github/workflows/FILENAME.yml`. GitHub watches this directory and triggers runs based on the events you declare. The `on:` block at the top defines triggers. `push: branches: [main]` means every push to main starts the workflow. `workflow_dispatch:` adds a manual trigger button in the GitHub UI.

A runner is a disposable Linux VM. It starts blank. It runs your steps. It gets destroyed. Every run is a fresh machine. This is both the power and the constraint — clean environment every time, but nothing persists between runs. You must explicitly checkout code, install tools, and cache dependencies on every run.

`actions/checkout@v4` is the first step of every workflow. It clones your repository onto the blank runner. Skip it and every subsequent step fails because there are no files to work with.

Secrets are stored in GitHub repository Settings → Secrets and variables → Actions. They are never visible in logs — GitHub masks them. Inside the workflow you reference them as `${{ secrets.SECRET_NAME }}`. The value is injected at runtime as an environment variable. Never hardcode credentials in the workflow file — the file is version controlled and visible to anyone with repo access.

`docker/login-action` authenticates the runner to Docker Hub using your credentials from secrets. `docker/build-push-action` builds the Docker image from the Dockerfile and pushes it to Docker Hub in one step. These two actions replace the manual `docker login`, `docker build`, `docker tag`, `docker push` sequence.

GitHub Actions and ArgoCD are not the same thing. GitHub Actions builds the image — CI. ArgoCD deploys the image to the cluster — CD. They are separate systems with separate responsibilities. Confusing them in an interview is a red flag.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- Workflow not triggering → check `on:` trigger matches the branch and event
- Runner cannot find files → `actions/checkout` missing or in wrong position
- Build fails with auth error → Docker Hub credentials wrong or secret name mismatch
- Build slow every run → no layer caching configured
- Secret visible in logs → secret was echoed directly — never `echo $SECRET`
- Job fails but I cannot see why → GitHub UI → Actions → click the run → click failed step

---

### Domain Awareness

- Self-hosted runners — your own machine as a runner — for private network access
- Reusable workflows — share pipeline logic across multiple repositories
- Matrix strategy — test across multiple OS versions or language versions simultaneously
- OIDC authentication — passwordless auth to AWS and GCP — no long-lived secrets
- GitHub Environments — approval gates before deploying to production
- Composite actions — package multiple steps into a reusable custom action

---

### The one insight that separates copiers from understanders

GitHub Actions does not know what your project needs — you tell it every time.

The runner starts blank. Every dependency, every tool, every credential must be explicitly provided in the workflow. This is not a limitation — it is the guarantee of a clean, reproducible environment. The workflow file IS the documentation of exactly what your build requires. If it works in CI, it works because you made it explicit. If it fails locally but works in CI — or vice versa — the difference is what was explicitly declared.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 14 evening.
> Use the prompt in `00-notes-pending.md` → GitHub Actions section.
> Pull content from the generated notes section: `06-debugging-failed-pipelines`.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 14 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 14 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 14 evening.
> Fill in based on your own experience during the GitHub Actions week.

### Preview — Learn this yourself no matter what

- The runner mental model — disposable, blank, starts fresh — why checkout is mandatory
- Why GitHub Actions is CI only — ArgoCD is CD — never confuse them in an interview
- How secrets work — stored in GitHub settings, referenced as `${{ secrets.NAME }}`
- What triggers a workflow — the `on:` block and what each event means
- Reading a failed pipeline — where to find the exact error in the GitHub UI

### Preview — Use AI for this

- Writing a complete workflow file for a new project — first draft, you read every line
- Cache configuration for Docker layers or a specific package manager
- Setting up OIDC authentication to AWS — exact YAML, you verify the permissions
- Syntax for matrix builds, conditional steps, job dependencies

📚 Deep dive → Coming Day 14 evening — see `00-notes-pending.md`
