[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Kubernetes

**Hire importance at $25–40/hr:** 6/10 — basics only at this tier. Grows to 9/10 at $55/hr.

**One line:** Docker runs one container. Kubernetes runs thousands — and keeps them alive, balanced, and updated without downtime.

> ⏳ Runbook notes not generated yet.
> Come back Day 7 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What a developer does that makes Kubernetes necessary

A developer builds a FastAPI application. It runs fine as one Docker container on one EC2 instance. Then traffic doubles. The single container cannot handle it — requests are slow, some time out. The developer says "we need more instances."

With Docker Compose you would manually spin up a second EC2, clone the repo, start another container, and put a load balancer in front. Do this for five services. Every deployment requires manual steps. Every container crash requires manual restart. Every scaling decision requires manual action.

Kubernetes solves this. You tell Kubernetes "I want 3 replicas of the API." Kubernetes creates 3 Pods, distributes traffic between them, and restarts any Pod that crashes — automatically, continuously, without your involvement.

### The developer's contract with Kubernetes

A developer writes a health check endpoint — `/api/health` — that returns 200 when the app is ready and healthy. Kubernetes calls this endpoint continuously. If it stops returning 200, Kubernetes restarts the container or removes it from the load balancer.

The developer writes the health endpoint. You configure the probe that calls it. Both sides must agree on the path and what HTTP status means healthy.

### The desired state mental model — the aha that makes everything click

Kubernetes is not a system that runs commands. It is a system that continuously reconciles reality with your declared intention.

You write a manifest — a YAML file — that says "I want 3 replicas of the API running at all times." Kubernetes stores this as the desired state. It then watches reality. If a Pod crashes, reality becomes 2 replicas. Kubernetes detects the gap between desired (3) and actual (2) and creates a new Pod. Always. Automatically.

This is why Kubernetes feels like magic. It is not magic. It is a control loop running every few seconds comparing desired state to actual state and fixing any difference.

```
You declare:     3 replicas of the API
Reality right now:  2 replicas (one crashed)
Kubernetes sees: gap of 1
Kubernetes acts: creates 1 new Pod
Reality now:     3 replicas
Loop repeats:    gap is 0, nothing to do
```

### ShopStack files Kubernetes touches

```
shopstack/infra/k8s/                    ← all manifests live here
shopstack/infra/k8s/api-deployment.yaml ← desired state for the API
shopstack/infra/k8s/api-service.yaml    ← stable endpoint for the API Pods
shopstack/infra/k8s/db-pvc.yaml         ← Postgres persistent storage
shopstack/infra/k8s/db-secret.yaml      ← database credentials
shopstack/services/api/src/main.py      ← where /api/health is defined
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- A Pod is the smallest deployable unit — one or more containers sharing a network and storage
- A Deployment manages Pods — it is what you create, not bare Pods
- Why you never run a bare Pod — no automatic restart, no rolling updates, no scaling
- A Service is a stable network endpoint — Pod IPs change, Service IP stays the same
- ClusterIP vs NodePort vs LoadBalancer — what each one exposes and to whom
- Kubernetes DNS — `api.shopstack.svc.cluster.local` — how services find each other
- Liveness probe vs readiness probe — what each one does when it fails
- PersistentVolumeClaim — how Postgres gets disk storage that survives Pod restarts
- `kubectl get`, `kubectl describe`, `kubectl logs`, `kubectl exec` — your four debugging tools
- CrashLoopBackOff — what it means and the first two commands you run

**When you can explain all ten without hesitation — stop. Move to GitHub Actions.**

### What unlocks higher pay

At $55/hr you add: writing manifests from scratch, Ingress controllers for external traffic, HorizontalPodAutoscaler for automatic scaling, resource requests and limits, namespaces for environment isolation, RBAC for access control, and network policies for Pod-to-Pod traffic restriction. At $80/hr you design multi-cluster architectures and own cluster-level reliability.

📚 Deep dive → Coming Day 7 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

A Pod is the smallest thing Kubernetes can deploy. It wraps one or more containers that share a network namespace and storage. Containers in the same Pod communicate via `localhost`. Containers in different Pods communicate via Services.

A Deployment is what you actually create. It manages a set of identical Pods. You tell the Deployment "I want 3 replicas." The Deployment creates a ReplicaSet, which creates the 3 Pods. When a Pod crashes, the ReplicaSet detects it and creates a replacement. You never create Pods directly in production — if the Pod crashes there is nothing to recreate it.

A Service is a stable network endpoint for a set of Pods. Pods are ephemeral — they crash, restart, get rescheduled, and get new IP addresses. A Service has a stable IP and DNS name that never changes. Other services talk to the Service, not to individual Pod IPs. Kubernetes DNS lets the API call `db` and Kubernetes resolves it to the db Service's stable IP.

ClusterIP is the default — the Service is only reachable from within the cluster. NodePort exposes the Service on a port on every Node — reachable from outside the cluster. LoadBalancer provisions a cloud load balancer — what you use in production on EKS to expose ShopStack to the internet.

Liveness probe checks if the container is alive — should it be restarted. If liveness fails, Kubernetes kills the container and starts a new one. Readiness probe checks if the container is ready to receive traffic — should it be in the load balancer rotation. If readiness fails, Kubernetes removes the Pod from the Service endpoints but does not restart it. A Pod can be alive but not ready — for example, still loading data on startup.

A PersistentVolumeClaim is how a Pod requests disk storage. The claim says "I need 5GB of storage." Kubernetes finds or provisions a PersistentVolume that satisfies the claim and binds them. The data on that volume survives Pod deletion and rescheduling — unlike the container's own filesystem which is destroyed with the Pod.

CrashLoopBackOff means a container is repeatedly crashing and Kubernetes keeps restarting it with increasing delays. The first command is `kubectl logs POD_NAME` — read what the container printed before it crashed. The second is `kubectl describe pod POD_NAME` — read the Events section at the bottom for what Kubernetes observed.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- Pod keeps crashing → CrashLoopBackOff → `kubectl logs` then `kubectl describe`
- Service not routing traffic → readiness probe failing → Pod not in endpoints
- Pod losing data on restart → no PVC attached → storage is on the container layer
- Service not reachable by name → wrong namespace or Service name mismatch
- New image not rolling out → check Deployment rollout status and events
- Pod stuck in Pending → insufficient cluster resources or PVC not bound

---

### Domain Awareness

- StatefulSets — for apps that need stable network identity and ordered deployment
- DaemonSets — run one Pod on every Node — log collectors, monitoring agents
- Jobs and CronJobs — run-to-completion workloads and scheduled tasks
- Helm — Kubernetes package manager — templated manifests for complex apps
- Custom Resource Definitions — extending Kubernetes with your own object types
- Service Mesh — Istio, Linkerd — advanced traffic management and mTLS

---

### The one insight that separates copiers from understanders

Kubernetes does not run your app. It continuously reconciles your declared intention with reality.

The manifest is not a command — it is a declaration. "This is the world I want." Kubernetes reads it, looks at what actually exists, and works to close any gap. This loop never stops. That is why a crashed Pod comes back — not because Kubernetes ran a restart command, but because the controller loop detected the gap and acted.

Once you understand this, every Kubernetes behaviour makes sense. OutOfSync means desired state diverged from actual. CrashLoopBackOff means Kubernetes keeps closing a gap that keeps reopening. Rollout means Kubernetes is transitioning actual state toward a new desired state — carefully, Pod by Pod.

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 7 evening.
> Use the prompt in `00-notes-pending.md` → Kubernetes section.
> Pull content from the generated notes section: `08-kubectl-command-reference`.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 7 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 7 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 7 evening.
> Fill in based on your own experience during the Kubernetes week.

### Preview — Learn this yourself no matter what

- What a Pod is vs a Deployment — Q1 in every Kubernetes interview
- The desired state mental model — the control loop — this is what separates understanders
- Liveness vs readiness probe — what each one does when it fails
- How Kubernetes DNS works — why `db` resolves to the db Service
- CrashLoopBackOff diagnosis sequence — cold, no notes

### Preview — Use AI for this

- Writing manifest files for a new service from scratch — first draft only, you read every line
- `kubectl` command syntax for specific filter and output flags
- Explaining a specific error from `kubectl describe` output

📚 Deep dive → Coming Day 7 evening — see `00-notes-pending.md`
