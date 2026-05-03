[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md) · [7. Interview](../day-7-interview/readme.md) · [8. K8s Architecture](../day-8-k8s-architecture/readme.md) · [9. Pods](../day-9-pods/readme.md) · **10. Deployments**

---

# Day 10 — Deployments

**Date:** May 3 2026   
**Read before session:** [03-deployments](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/05.%20Kubernetes/03-deployments)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Write Deployment manifests for all ShopStack services. Prove self-healing by deleting a Pod and watching it come back. Trigger a rolling update by changing the image tag. Roll it back with one command.   

---

## Knowledge — What Deployments Are Really About

### The problem you already proved

Yesterday you deleted a bare Pod and it stayed dead. Nobody noticed. Nobody fixed it. Nothing came back. That is the problem. In production, if a Pod crashes at 3 AM because of a memory spike, a bad config, or a flaky dependency — and nothing brings it back — that is downtime. Real downtime. Outage. On-call. Incident report.

A bare Pod has no guardian. That is why you never use one for anything that matters.

### The thermostat analogy — ReplicaSets

A ReplicaSet has exactly one job: make sure a specific number of identical Pods are running right now.

Think of it as a thermostat set to 2. The thermostat watches the room constantly. Pod crashes? Count drops to 1. The thermostat immediately turns on the heat and creates a new Pod to bring it back to 2. Three Pods somehow running? It kills one. It never stops watching. It never sleeps. It does not care why the count dropped — it just fixes it.

```
Desired state:  replicas = 2
Actual state:   running  = 1  ← a Pod crashed

ReplicaSet detects gap → creates 1 new Pod → running = 2 ✅
```

You almost never create a ReplicaSet directly. You create a Deployment, which creates and manages the ReplicaSet for you. The ReplicaSet is the mechanism. The Deployment is the manager.

### The building manager analogy — Deployments

If a ReplicaSet is the thermostat that keeps the count right, a Deployment is the building manager who controls everything about the thermostat — including how to safely upgrade it, roll it back, and configure it without kicking everyone out.

```
You (kubectl apply)
        │
        ▼
┌───────────────────┐
│    Deployment     │  ← You create this. Owns everything below.
└────────┬──────────┘
         │ creates and manages
         ▼
┌───────────────────┐
│    ReplicaSet     │  ← Enforces Pod count 24/7.
└────────┬──────────┘
         │ creates and manages
         ▼
┌─────────────────────────┐
│  Pod [api]  Pod [api]   │  ← Your app actually runs here.
└─────────────────────────┘
```

| Feature | Bare Pod | ReplicaSet | Deployment |
|---|---|---|---|
| Self-healing | ❌ | ✅ | ✅ |
| Scaling | ❌ | ✅ | ✅ |
| Rolling updates | ❌ | ❌ | ✅ |
| Rollbacks | ❌ | ❌ | ✅ |
| Update history | ❌ | ❌ | ✅ |

**The rule:** Bare Pod is for learning. ReplicaSet is for the machine. Deployment is what you write.

### The hotel renovation analogy — Rolling Updates

You need to renovate a 10-floor hotel without closing it. You cannot evict all guests at once. So you renovate floor by floor — move guests from floor 1 to a spare room, renovate floor 1, move them back, then floor 2. At no point is the hotel fully closed. Guests always have somewhere to stay.

Kubernetes does the exact same thing when you update your API image from v1.0 to v1.1.

```
BEFORE UPDATE         DURING UPDATE              AFTER UPDATE
[Pod v1.0]            [Pod v1.0] running         [Pod v1.1] ✅
[Pod v1.0]            [Pod v1.1] starting        [Pod v1.1] ✅
                      [Pod v1.0] terminated
                      [Pod v1.1] running
                      [Pod v1.0] terminated
```

Traffic keeps flowing the entire time. Zero downtime. The old ReplicaSet stays alive at 0 replicas — that is your rollback parachute.

### The selector — the critical link

A Deployment finds and manages its Pods using labels. The `selector.matchLabels` on the Deployment must exactly match the `labels` on the Pod template. This is how Kubernetes knows which Pods belong to which Deployment.

One typo here = the Deployment creates Pods it cannot manage. No self-healing. No rolling updates. Nothing works.

**Always check the selector when Pods are not appearing after `kubectl apply`.**

### ShopStack connection

Today you replace all the bare Pods from Day 9 with Deployment manifests. The same images, same ports, same env vars — but now wrapped in a Deployment that watches over them forever.

```
shopstack/infra/k8s/
├── api-pod.yaml          ← Day 9 — deleted today, replaced
├── frontend-pod.yaml     ← Day 9 — deleted today, replaced
├── db-pod.yaml           ← Day 9 — deleted today, replaced
├── api-deployment.yaml   ← Day 10 — 2 replicas, self-healing
├── frontend-deployment.yaml ← Day 10 — 2 replicas
├── worker-deployment.yaml   ← Day 10 — 1 replica
└── db-deployment.yaml       ← Day 10 — 1 replica (Secrets + PVC added Day 12)
```

---

# 📟 DAY 10 FULL CHEATSHEET — Deployments

## Writing and Applying

| What it does | Command | Example |
|---|---|---|
| [CORE] Apply manifest — create or update | `kubectl apply -f <file>` | `kubectl apply -f infra/k8s/api-deployment.yaml` |
| [CORE] Apply all manifests in a directory | `kubectl apply -f <directory>/` | `kubectl apply -f infra/k8s/` |
| [CORE] Delete Deployment and all its Pods | `kubectl delete deployment <name>` | `kubectl delete deployment shopstack-api` |

---

## Inspecting Deployments

| What it does | Command | Example |
|---|---|---|
| [CORE] Check all Deployments — ready count | `kubectl get deployments` | `kubectl get deployments` |
| [CORE] See the ReplicaSets behind Deployments | `kubectl get rs` | `kubectl get rs` |
| [CORE] See the Pods the RS created | `kubectl get pods` | `kubectl get pods` |
| [CORE] Watch Pods change live — Ctrl+C to stop | `kubectl get pods -w` | `kubectl get pods -w` |
| [CORE] Full event log — errors appear here | `kubectl describe deployment <name>` | `kubectl describe deployment shopstack-api` |
| [WIKI] All resources in one shot | `kubectl get all` | `kubectl get all` |

**Syntax breakdown — reading `kubectl get deployments` output:**
```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
shopstack-api      2/2     2            2           10m
                   ↑       ↑            ↑
            running/total  on latest    passing health checks
```

**Syntax breakdown — reading `kubectl get rs` output:**
```
NAME                       DESIRED   CURRENT   READY   AGE
shopstack-api-7d9f8b6c4    2         2         2       2m   ← active (new)
shopstack-api-5c6b7a8d9    0         0         0       10m  ← old, kept for rollback
```

---

## Rolling Updates

| What it does | Command | Example |
|---|---|---|
| [CORE] Watch rolling update in real time | `kubectl rollout status deployment/<name>` | `kubectl rollout status deployment/shopstack-api` |
| [CORE] Update image tag — triggers rolling update | `kubectl set image deployment/<name> <container>=<image>` | `kubectl set image deployment/shopstack-api api=akhiltejadoosari/shopstack-api:1.1` |
| [WIKI] Alternatively — edit the YAML and re-apply | `kubectl apply -f <file>` | `kubectl apply -f infra/k8s/api-deployment.yaml` |

**Syntax breakdown — `kubectl set image`:**
```
kubectl set image deployment/shopstack-api   api=akhiltejadoosari/shopstack-api:1.1
                 ↑                           ↑    ↑
            deployment name          container    new image:tag
                                     name in
                                     the manifest
```

---

## Rollbacks

| What it does | Command | Example |
|---|---|---|
| [CORE] See all previous revisions | `kubectl rollout history deployment/<name>` | `kubectl rollout history deployment/shopstack-api` |
| [CORE] Emergency rollback — go back one version | `kubectl rollout undo deployment/<name>` | `kubectl rollout undo deployment/shopstack-api` |
| [WIKI] Rollback to a specific revision number | `kubectl rollout undo deployment/<name> --to-revision=<n>` | `kubectl rollout undo deployment/shopstack-api --to-revision=1` |

---

## Scaling

| What it does | Command | Example |
|---|---|---|
| [CORE] Scale up or down | `kubectl scale deployment/<name> --replicas=<n>` | `kubectl scale deployment/shopstack-api --replicas=4` |

---

## The Deployment Manifest — Anatomy

```yaml
apiVersion: apps/v1         # ← NOT v1 — Deployments live in apps/v1
                            #   Pods use v1. Deployments use apps/v1.
                            #   Wrong version = API Server rejects immediately.
kind: Deployment

metadata:
  name: shopstack-api
  labels:
    app: shopstack
    tier: api

spec:
  replicas: 2               # ← How many Pod copies to keep alive at all times.
                            #   ReplicaSet enforces this number 24/7.

  selector:                 # ← How this Deployment finds and claims its Pods.
    matchLabels:
      app: shopstack        # ← MUST match the Pod template labels below.
      tier: api             #   One typo = Deployment owns no Pods.

  template:                 # ← The Pod blueprint. Every new Pod uses this.
    metadata:
      labels:
        app: shopstack      # ← Must match selector.matchLabels exactly.
        tier: api
    spec:
      containers:
        - name: api
          image: akhiltejadoosari/shopstack-api:1.0
                            # ← Always pin a version. Never use 'latest'.
                            #   You cannot roll back latest to latest.
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "db"
            - name: DB_NAME
              value: "shopstack"
            - name: DB_USER
              value: "shopstack"
            - name: DB_PASSWORD
              value: "shopstack_dev"  # ← Moves to a Secret on Day 12
```

**The three structural sections of a Deployment `spec`:**
```
spec:
  replicas: 2          ← how many
  selector:            ← which Pods I own (matchLabels)
  template:            ← the Pod blueprint (labels must match selector)
```

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| Deployment created but no Pods appear | `selector.matchLabels` does not match Pod template `labels` | `kubectl describe deployment shopstack-api` → read Events |
| Rolling update hangs at "1 out of 2 updated" | New Pod is failing to start — bad image tag or bad env vars | `kubectl get pods` → find stuck Pod → `kubectl describe pod <name>` |
| `kubectl rollout undo` does nothing | Already at revision 1 — nothing to roll back to | `kubectl rollout history deployment/<name>` to confirm |
| Scale up — Pods stuck in `Pending` | EC2 t3.micro out of CPU or RAM | `kubectl describe pod <name>` → Events → look for `Insufficient cpu` |
| Deleted a Deployment — Pods vanished too | Correct — Deployment deletion cascades to RS and Pods | Recreate with `kubectl apply -f` |
| `apps/v1` error on apply | Used `v1` instead of `apps/v1` in apiVersion | Fix the YAML — Pods use `v1`, Deployments use `apps/v1` |

---

## Checklist

> 🧪 Practice Only — Not a DevOps Skill
> Delete the bare Pods from Day 9 before writing Deployments. This is cleanup — not a DevOps operation.
> ```bash
> kubectl delete pod shopstack-api shopstack-frontend shopstack-db
> kubectl get pods   # confirm clean
> ```
> ↩️ Back to DevOps — resume normal checklist

- [ ] Read `03-deployments` in your runbook — all sections
- [ ] Write `infra/k8s/api-deployment.yaml` from scratch — 2 replicas
- [ ] Write `infra/k8s/frontend-deployment.yaml` from scratch — 2 replicas
- [ ] Write `infra/k8s/db-deployment.yaml` from scratch — 1 replica
- [ ] Write `infra/k8s/worker-deployment.yaml` from scratch — 1 replica
- [ ] `kubectl apply -f infra/k8s/`
- [ ] `kubectl get deployments` — read every column out loud: READY, UP-TO-DATE, AVAILABLE
- [ ] `kubectl get pods` — confirm 2 API pods, 2 frontend pods, 1 db, 1 worker
- [ ] `kubectl get rs` — understand what you are looking at
- [ ] Delete one API Pod by name: `kubectl delete pod <name>`
- [ ] `kubectl get pods -w` — watch it come back automatically
- [ ] Write in your notebook: what just happened and who made it happen
- [ ] Change the image tag in `api-deployment.yaml` from `1.0` to `1.1`
- [ ] `kubectl apply -f infra/k8s/api-deployment.yaml`
- [ ] `kubectl rollout status deployment/shopstack-api` — watch rolling update live
- [ ] `kubectl get rs` — confirm two RS exist — one at 2, one at 0
- [ ] `kubectl rollout history deployment/shopstack-api` — see both revisions
- [ ] `kubectl rollout undo deployment/shopstack-api` — roll back to 1.0
- [ ] `kubectl get pods` — confirm rollback completed
- [ ] `kubectl get rs` — old RS scaled back up, new RS at 0
- [ ] Answer out loud: what is the difference between a Pod and a Deployment?

**Session win condition:** Self-healing proven. Rolling update done. Rollback done. You can write a Deployment manifest from scratch without looking at notes.

---

Ready to test yourself? → [Test](./test.md)
