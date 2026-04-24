[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# ArgoCD

**Hire importance at $25–40/hr:** 2/10 — know the concept. Grows to 7/10 at $55/hr.

**One line:** ArgoCD watches a Git repo and makes the cluster match what is in it. Git is the source of truth. Not kubectl. Not you.

> ⏳ Runbook notes not generated yet.
> Come back Day 14 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What GitOps is and why it exists

Before GitOps, deploying to Kubernetes meant someone running `kubectl apply` manually — on their laptop, with their credentials, in a terminal. No record of who deployed what. No easy rollback. No audit trail. Different engineers with different versions of the manifests.

GitOps solves this by making Git the single source of truth for cluster state. The cluster should look exactly like what is in the Git repo — no more, no less. ArgoCD is the tool that enforces this. It watches your Git repo continuously. When the repo changes, ArgoCD syncs the cluster to match. When the cluster drifts from the repo — someone ran `kubectl apply` manually — ArgoCD detects the drift and can revert it.

### The critical distinction — ArgoCD does not build

GitHub Actions builds the Docker image. ArgoCD deploys it. They are completely separate systems with completely separate responsibilities.

GitHub Actions runs on every push, builds the image, tags it, pushes it to Docker Hub. It does not touch the cluster.

ArgoCD watches the Git repo. When a manifest changes — for example, the image tag in `api-deployment.yaml` is updated from `v1.0` to `v1.1` — ArgoCD detects the change and applies the new manifest to the cluster. It does not build anything.

The connection between them is a manifest change in Git. CI updates the tag. ArgoCD sees the tag change. ArgoCD deploys the new image.

### The ShopStack GitOps flow

```
Developer pushes code to main
          ↓
GitHub Actions builds shopstack-api:a3f92c1
          ↓
CI updates api-deployment.yaml image tag to a3f92c1
          ↓
Git repo now has the new tag
          ↓
ArgoCD detects the manifest changed
          ↓
ArgoCD applies api-deployment.yaml to the cluster
          ↓
Kubernetes rolls out new Pods with the new image
          ↓
Old Pods terminate after new ones are healthy
```

### ShopStack files ArgoCD watches

```
shopstack/infra/k8s/                    ← ArgoCD watches this entire directory
shopstack/infra/k8s/api-deployment.yaml ← image tag change triggers deployment
shopstack/infra/k8s/api-service.yaml    ← Service definition
shopstack/infra/k8s/db-pvc.yaml         ← Postgres storage
ArgoCD Application manifest             ← defines what repo and path to watch
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- GitOps means Git is the source of truth — the cluster must match the repo
- ArgoCD watches Git and syncs the cluster — it does not build images
- GitHub Actions is CI — ArgoCD is CD — they are separate, connected by a manifest change
- Synced means Git and cluster match — OutOfSync means they differ
- Healthy means all resources are running — Degraded means something is broken
- Manual sync vs automated sync — the difference and when to use each
- Rollback in ArgoCD goes back to a previous Git commit — not `kubectl rollout undo`
- An Application manifest defines what repo, path, and cluster ArgoCD manages
- Drift — someone applied changes directly with kubectl — ArgoCD detects this
- The image tag in the manifest is the trigger — changing it starts a deployment

**When you can explain all ten without hesitation — stop. Move to Observability.**

### What unlocks higher pay

At $55/hr you add: ApplicationSets for managing multiple apps, sync waves and hooks for deployment ordering, custom health checks for non-standard resources, notifications when deployments fail, multi-cluster management from one ArgoCD instance, and App of Apps pattern for bootstrapping entire clusters.

📚 Deep dive → Coming Day 14 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

ArgoCD does not build. ArgoCD does not push images. ArgoCD watches a Git repo path and makes the cluster match what it finds there. The image is already built and pushed by GitHub Actions. ArgoCD sees the new image tag in the manifest and applies the manifest. The build and the deploy are completely separate pipelines connected only by a Git commit.

Synced means the cluster state matches the Git repo exactly. OutOfSync means they differ — either the repo changed and the cluster has not been updated yet, or someone ran `kubectl apply` directly bypassing Git. OutOfSync is not an error — it is information. It tells you the cluster needs to be updated or someone made a change outside of GitOps.

Healthy means all the Kubernetes resources in the application are in the expected state — Pods running, Services routing, PVCs bound. Degraded means something is broken — a Pod is CrashLoopBackOff, a Deployment has no available replicas, a PVC is unbound. Progressing means an update is in flight — rolling out new Pods.

Automated sync means ArgoCD applies changes the moment it detects a repo change — no human approval needed. Manual sync means someone must click Sync or run `argocd app sync` explicitly. Use automated sync for development environments. Use manual sync for production where a human should approve every deployment.

Rollback in ArgoCD means syncing to a previous Git commit — the manifests at that point in history. This is different from `kubectl rollout undo` which rolls back the Deployment object independently of Git. ArgoCD rollback keeps Git and the cluster in sync. `kubectl rollout undo` creates drift — the cluster no longer matches Git.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- App shows OutOfSync → compare Git manifest to cluster with `argocd app diff`
- App shows Degraded → `kubectl describe` the failing resource, read Events
- New image not deploying → was the image tag updated in the manifest in Git?
- Rollback needed → `argocd app rollback` to a previous history entry
- ArgoCD not detecting changes → check the repo connection and branch setting
- Cluster drifted from Git → someone ran kubectl directly — sync to restore

---

### Domain Awareness

- ApplicationSets — template multiple ArgoCD Applications from one definition
- App of Apps pattern — one ArgoCD Application manages other Applications
- Sync waves and hooks — control the order resources are applied during sync
- Notifications — alert Slack or email when deployments succeed or fail
- Multi-cluster — manage multiple Kubernetes clusters from one ArgoCD instance
- Image Updater — ArgoCD plugin that updates image tags automatically

---

### The one insight that separates copiers from understanders

ArgoCD makes the cluster a read-only output of Git.

If Git says 3 replicas, the cluster has 3 replicas. If someone manually scales to 5 with kubectl, ArgoCD detects the drift and can revert it to 3. The cluster is not the authority — Git is. This is the entire value of GitOps. Every change is a Git commit. Every deployment has an audit trail. Every rollback is a revert commit. The cluster is just the rendered output of what Git contains.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 14 evening.
> Use the prompt in `00-notes-pending.md` → ArgoCD section.
> Pull content from the generated notes section: `06-argocd-cli-reference`.

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
> Fill in based on your own experience during the ArgoCD week.

### Preview — Learn this yourself no matter what

- GitOps concept — Git is source of truth, cluster is the output — core to every interview
- ArgoCD does not build — GitHub Actions builds — never confuse them
- Synced vs OutOfSync vs Healthy vs Degraded — what each status means
- Why ArgoCD rollback is safer than kubectl rollout undo — keeps Git and cluster in sync
- The image tag in the manifest is the deployment trigger — not the image push itself

### Preview — Use AI for this

- Writing an ArgoCD Application manifest for ShopStack
- Configuring automated sync with pruning and self-heal options
- Setting up ArgoCD notifications for a specific chat platform
- ApplicationSet template syntax for multi-environment deployments

📚 Deep dive → Coming Day 14 evening — see `00-notes-pending.md`
