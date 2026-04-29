[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Kubernetes

**Hire importance at $25–40/hr:** 6/10 — basics only at this tier. Grows to 9/10 at $55/hr.

**One line:** Docker runs one container. Kubernetes runs thousands — and keeps them alive, balanced, and updated without downtime.

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
You declare:        3 replicas of the API
Reality right now:  2 replicas (one crashed)
Kubernetes sees:    gap of 1
Kubernetes acts:    creates 1 new Pod
Reality now:        3 replicas
Loop repeats:       gap is 0, nothing to do
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

📚 Deep dive → [devops-runbook/notes/05. Kubernetes – Orchestration/](./notes/05-kubernetes/)

---

## 2 — The Bag

### Own These Cold

A Pod is the smallest thing Kubernetes can deploy. It wraps one or more containers that share a network namespace and storage. Containers in the same Pod communicate via `localhost`. Containers in different Pods communicate via Services.

A Deployment is what you actually create. It manages a set of identical Pods. You tell the Deployment "I want 3 replicas." The Deployment creates a ReplicaSet, which creates the 3 Pods. When a Pod crashes, the ReplicaSet detects it and creates a replacement. You never create Pods directly in production — if the Pod crashes there is nothing to recreate it.

A Service is a stable network endpoint for a set of Pods. Pods are ephemeral — they crash, restart, get rescheduled, and get new IP addresses. A Service has a stable IP and DNS name that never changes. Other services talk to the Service, not to individual Pod IPs. Kubernetes DNS lets the API call `db` and Kubernetes resolves it to the db Service's stable IP.

ClusterIP is the default — the Service is only reachable from within the cluster. NodePort exposes the Service on a port on every Node — reachable from outside the cluster. LoadBalancer provisions a cloud load balancer — what you use in production on EKS to expose ShopStack to the internet.

Liveness probe checks if the container is alive — should it be restarted. If liveness fails, Kubernetes kills the container and starts a new one. Readiness probe checks if the container is ready to receive traffic — should it be in the load balancer rotation. If readiness fails, Kubernetes removes the Pod from the Service endpoints but does not restart it. A Pod can be alive but not ready — for example, still loading data on startup.

A PersistentVolumeClaim is how a Pod requests disk storage. The claim says "I need 5GB of storage." Kubernetes finds or provisions a PersistentVolume that satisfies the claim and binds them. The data on that volume survives Pod deletion and rescheduling — unlike the container's own filesystem which is destroyed with the Pod.

CrashLoopBackOff means a container is repeatedly crashing and Kubernetes keeps restarting it with increasing delays. The first command is `kubectl logs POD_NAME --previous` — read what the container printed before it crashed. The second is `kubectl describe pod POD_NAME` — read the Events section at the bottom for what Kubernetes observed.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → Section 3 combat sheet.

- Pod keeps crashing → CrashLoopBackOff → `kubectl logs --previous` then `kubectl describe`
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

| What it does | Command | Example |
|---|---|---|
| Confirm node is Ready | `kubectl get nodes` | `kubectl get nodes` |
| Full cluster health scan | `kubectl get pods -A` | `kubectl get pods -A` |
| Check ShopStack pods | `kubectl get pods` | `kubectl get pods` |
| Filter pods by label | `kubectl get pods -l <label>` | `kubectl get pods -l tier=api` |
| Watch pod changes live | `kubectl get pods -w` | `kubectl get pods -w` |
| Full pod event log | `kubectl describe pod <n>` | `kubectl describe pod shopstack-api-xxx` |
| Container logs | `kubectl logs <n>` | `kubectl logs shopstack-api-xxx` |
| Logs from previous crash | `kubectl logs <n> --previous` | `kubectl logs shopstack-api-xxx --previous` |
| Follow logs live | `kubectl logs -f <n>` | `kubectl logs -f shopstack-api-xxx` |
| Enter a running container | `kubectl exec -it <n> -- /bin/sh` | `kubectl exec -it shopstack-api-xxx -- /bin/sh` |
| Apply a manifest | `kubectl apply -f <file>` | `kubectl apply -f infra/k8s/api-deployment.yaml` |
| Apply all manifests in folder | `kubectl apply -f <folder>` | `kubectl apply -f infra/k8s/` |
| Check all Deployments | `kubectl get deployments` | `kubectl get deployments` |
| Watch rolling update | `kubectl rollout status deployment/<n>` | `kubectl rollout status deployment/shopstack-api` |
| Emergency rollback | `kubectl rollout undo deployment/<n>` | `kubectl rollout undo deployment/shopstack-api` |
| Restart all pods in Deployment | `kubectl rollout restart deployment/<n>` | `kubectl rollout restart deployment/shopstack-api` |
| Scale up or down | `kubectl scale deployment/<n> --replicas=<n>` | `kubectl scale deployment/shopstack-api --replicas=4` |
| Update image — trigger rolling update | `kubectl set image deployment/<n> <c>=<image>` | `kubectl set image deployment/shopstack-api api=akhiltejadoosari/shopstack-api:1.1` |
| List all Services | `kubectl get services` | `kubectl get services` |
| Check which pods a Service routes to | `kubectl get endpoints` | `kubectl get endpoints` |
| List all PVCs | `kubectl get pvc` | `kubectl get pvc` |
| Decode a Secret value | `kubectl get secret <n> -o jsonpath='{.data.<key>}' \| base64 -d` | `kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' \| base64 -d` |
| Cluster event timeline | `kubectl get events --sort-by=.lastTimestamp` | `kubectl get events --sort-by=.lastTimestamp` |
| Get pods in a namespace | `kubectl get pods -n <n>` | `kubectl get pods -n kube-system` |
| Check probe config and endpoints | `kubectl describe service <n>` | `kubectl describe service api` |

📚 Full reference → [08-kubectl-reference](./notes/05-kubernetes/08-kubectl-reference/README.md)

---

## 4 — ShopStack Map

| Situation | File | Command |
|---|---|---|
| ShopStack not starting | `infra/k8s/` | `kubectl get pods` → find failing pod |
| API cannot reach DB | `infra/k8s/db-service.yaml` | `kubectl get endpoints db` → check if `<none>` |
| DB losing data on restart | `infra/k8s/db-pvc.yaml` | `kubectl get pvc` → must show `Bound` |
| Wrong DB password | `infra/k8s/db-secret.yaml` | `kubectl logs shopstack-db-xxx --previous` |
| Frontend not reachable in browser | `infra/k8s/frontend-service.yaml` | confirm NodePort 30080 in EC2 security group |
| API returning 500 | `shopstack/services/api/src/main.py` | `kubectl logs shopstack-api-xxx` |
| Pod stuck in Pending | any deployment manifest | `kubectl describe pod <n>` → Events → Insufficient resources |
| Rolling update stuck | `infra/k8s/api-deployment.yaml` | `kubectl rollout status deployment/shopstack-api` |
| New image not pulling | deployment manifest | `kubectl describe pod <n>` → Events → image pull error |
| CrashLoopBackOff on api | `infra/k8s/db-secret.yaml` | `kubectl logs shopstack-api-xxx --previous` → DB_HOST wrong |
| CrashLoopBackOff on db | `infra/k8s/db-secret.yaml` | `kubectl logs shopstack-db-xxx --previous` → missing POSTGRES_PASSWORD |
| Service selector mismatch | service + deployment manifests | `kubectl describe service api` → Endpoints: `<none>` |

📚 Full notes → [05. Kubernetes – Orchestration](./notes/05-kubernetes/README.md)

---

## 5 — Peace of Mind

10 questions. Toggle each answer. Say it out loud before revealing. 30 seconds per answer — no notes.

**Q1 — What is Kubernetes and why does it exist?**
<details><summary>Answer</summary>
Kubernetes is a container orchestration platform. Docker lets you run containers but cannot keep them running at scale — if a container crashes it stays down, scaling requires manual work. Kubernetes solves this. You declare desired state — "I want 2 copies of the ShopStack API" — and Kubernetes enforces it continuously. Crashed Pod? Replaced automatically. Need to scale? One command. New version? Rolling update with zero downtime.
</details>

**Q2 — What is a Pod?**
<details><summary>Answer</summary>
The smallest deployable unit in Kubernetes. A wrapper around one or more containers that share a network namespace and storage volumes. All containers in a Pod share the same IP and communicate via localhost. Kubernetes never runs a naked container — it always wraps it in a Pod first.
</details>

**Q3 — What is the difference between a Pod and a Deployment?**
<details><summary>Answer</summary>
A bare Pod has no guardian — delete it or it crashes, it stays dead. A Deployment declares desired state: "I want 2 replicas running at all times." The Deployment creates a ReplicaSet which watches the Pod count and creates or terminates Pods to match. In production you always use a Deployment — never bare Pods for anything that matters.
</details>

**Q4 — What is a Kubernetes Service and why is it needed?**
<details><summary>Answer</summary>
A stable network endpoint for a set of Pods. Pods are ephemeral — every restart gives a new IP. If the API talked to the DB Pod directly by IP, any DB restart would break the connection permanently. A Service has a stable ClusterIP and DNS name that never changes. The API connects to `db` — Kubernetes DNS resolves it to the db Service — the Service routes to whichever db Pod is currently running.
</details>

**Q5 — ClusterIP vs NodePort vs LoadBalancer?**
<details><summary>Answer</summary>
ClusterIP is the default — only reachable inside the cluster. I use it for the ShopStack API and DB. NodePort exposes the Service on a port on the EC2 instance — port 30080 for the ShopStack frontend during development. LoadBalancer provisions a cloud load balancer — on EKS this creates an AWS ALB. That is what you use in production to expose a service to the internet.
</details>

**Q6 — Liveness probe vs readiness probe?**
<details><summary>Answer</summary>
Liveness asks: is this container still functioning? Fail → container is killed and restarted. Readiness asks: is this container ready for traffic? Fail → Pod removed from Service endpoints, no restart. Critical difference: liveness failure causes a restart, readiness failure causes traffic removal. A Pod can be alive but not ready — still loading DB connections on startup. On ShopStack both probes hit `/api/health`.
</details>

**Q7 — What is a PVC and why does Postgres need one?**
<details><summary>Answer</summary>
A PersistentVolumeClaim is a request for disk storage. Without it, Postgres stores data on the container's writable layer — destroyed on every Pod restart, wiping the entire database. With a PVC, data lives on a separate volume that persists across restarts. New Pod mounts the same volume — Postgres continues from where it left off.
</details>

**Q8 — Secret vs ConfigMap?**
<details><summary>Answer</summary>
Both inject configuration into Pods as env vars. ConfigMap stores non-sensitive config — DB hostname, port, feature flags. Secret stores sensitive data — passwords, API keys. Values are base64 encoded — not encrypted, just encoded. In ShopStack, `DB_HOST=db` lives in a ConfigMap. `DB_PASSWORD` lives in a Secret. Neither gets committed to GitHub.
</details>

**Q9 — What is CrashLoopBackOff and how do you diagnose it?**
<details><summary>Answer</summary>
Container is repeatedly crashing and Kubernetes keeps restarting it with increasing delays. Two commands: first `kubectl logs <pod> --previous` — shows what the container printed before crashing, application error is usually right there. Second `kubectl describe pod <pod>` — scroll to Events — shows what Kubernetes observed, exit codes, OOMKilled signals. In ShopStack the most common cause is wrong DB_HOST — connection refused in logs, fix the env var, restart the Deployment.
</details>

**Q10 — What does kubectl apply do? Apply vs create?**
<details><summary>Answer</summary>
`kubectl apply` sends a manifest to the API Server. If the object does not exist it creates it. If it exists it updates only the changed fields. Idempotent — run it multiple times, same result. `kubectl create` only creates — if the object already exists it errors. In production always use `kubectl apply` — it supports the declarative GitOps workflow.
</details>

📚 Full interview prep → [99-interview-prep](./notes/05-kubernetes/99-interview-prep/README.md)

---

## 6 — AI Split

### Learn this yourself — no shortcuts

- The desired state mental model — the control loop — this is what separates understanders from copiers
- What a Pod is vs a Deployment — Q1 in every Kubernetes interview
- Liveness vs readiness probe — what each one does when it fails — interviewers love this distinction
- How Kubernetes DNS works — why `db` resolves to the db Service ClusterIP
- CrashLoopBackOff diagnosis sequence — `kubectl logs --previous` then `kubectl describe` — cold, no notes
- Reading `kubectl describe` Events section — this is your primary debugging tool
- Why base64 is not encryption — what actually makes Secrets safe

### Use AI for this

- First draft of a manifest for a new service — you read every line before applying
- Specific `kubectl` flag syntax you cannot recall — `jsonpath`, `output` formats
- Explaining a specific error from `kubectl describe` output you have not seen before
- Writing the `sed` command to update kubeconfig after EC2 IP changes
- Generating base64 encoded values for Secrets

### The line

If an interviewer can test it in 30 seconds with a verbal question — own it yourself. If it requires looking up syntax that changes by version — use AI.
