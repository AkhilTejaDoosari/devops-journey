# Session 03 — Deployments

**Goal:** Write Deployment manifests for all ShopStack services, prove self-healing works, do a rolling update and a rollback.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Deployments Exist

Session 02 proved that bare Pods don't self-heal. A Deployment wraps Pods with three superpowers:

```
Self-healing    → Pod dies → Deployment creates a replacement immediately
Rolling updates → New version deployed one Pod at a time — zero downtime
Rollback        → One command to undo the last deployment
```

In production you never run bare Pods. Every app runs in a Deployment.

---

## Visual Map

```
You (kubectl apply)
        │
        ▼
┌───────────────────┐
│    Deployment     │  ← you write this — owns everything below
└────────┬──────────┘
         │ creates
         ▼
┌───────────────────┐
│    ReplicaSet     │  ← K8s creates this — enforces Pod count
└────────┬──────────┘
         │ creates
         ▼
  ┌────────────┐ ┌────────────┐
  │  Pod (api) │ │  Pod (api) │  ← replicas: 2
  └────────────┘ └────────────┘
```

---

## Deployment vs ReplicaSet vs Pod

| Feature | Bare Pod | Deployment |
|---|---|---|
| Self-healing | ❌ | ✅ |
| Scaling | ❌ | ✅ |
| Rolling updates | ❌ | ✅ |
| Rollbacks | ❌ | ✅ |
| Version history | ❌ | ✅ |

---

## The Files — All ShopStack Deployment Manifests

### 📄 infra/k8s/api-deployment.yaml

<details>
<summary>📄 Show full file — infra/k8s/api-deployment.yaml</summary>

```yaml
apiVersion: apps/v1       # Deployments use apps/v1 — NOT v1
kind: Deployment
metadata:
  name: shopstack-api
  labels:
    app: shopstack
    tier: api
spec:
  replicas: 2             # keep 2 copies running at all times
  selector:
    matchLabels:
      app: shopstack      # THIS must exactly match Pod template labels below
      tier: api           # one typo = Deployment cannot manage its own Pods
  template:               # Pod blueprint — every Pod created uses this
    metadata:
      labels:
        app: shopstack    # must match selector.matchLabels above
        tier: api
    spec:
      containers:
        - name: api
          image: akhiltejadoosari/shopstack-api:1.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              value: "db"
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: "shopstack"
            - name: DB_USER
              value: "shopstack"
            - name: DB_PASS
              value: "shopstack_dev"   # moves to Secret in Session 05
```

</details>

---

### 📄 infra/k8s/db-deployment.yaml

<details>
<summary>📄 Show full file — infra/k8s/db-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-db
  labels:
    app: shopstack
    tier: db
spec:
  replicas: 1             # always 1 for Postgres — multiple replicas need StatefulSet
  selector:
    matchLabels:
      app: shopstack
      tier: db
  template:
    metadata:
      labels:
        app: shopstack
        tier: db
    spec:
      containers:
        - name: db
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: "shopstack"
            - name: POSTGRES_USER
              value: "shopstack"
            - name: POSTGRES_PASSWORD
              value: "shopstack_dev"   # moves to Secret in Session 05
```

</details>

---

### 📄 infra/k8s/frontend-deployment.yaml

<details>
<summary>📄 Show full file — infra/k8s/frontend-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-frontend
  labels:
    app: shopstack
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shopstack
      tier: frontend
  template:
    metadata:
      labels:
        app: shopstack
        tier: frontend
    spec:
      containers:
        - name: frontend
          image: akhiltejadoosari/shopstack-frontend:1.0
          ports:
            - containerPort: 80
```

</details>

---

### 📄 infra/k8s/worker-deployment.yaml

<details>
<summary>📄 Show full file — infra/k8s/worker-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-worker
  labels:
    app: shopstack
    tier: worker
spec:
  replicas: 1             # one worker is enough — pings health every 10s
  selector:
    matchLabels:
      app: shopstack
      tier: worker
  template:
    metadata:
      labels:
        app: shopstack
        tier: worker
    spec:
      containers:
        - name: worker
          image: akhiltejadoosari/shopstack-worker:1.0
          env:
            - name: API_URL
              value: "http://api:8080"   # DNS resolves "api" → api Service
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — k3s running, manifests exist
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Daily opening sequence

```bash
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get pods
```

---

### Step 2 — Apply all Deployments

```bash
kubectl apply -f ~/shopstack/infra/k8s/db-deployment.yaml
kubectl apply -f ~/shopstack/infra/k8s/api-deployment.yaml
kubectl apply -f ~/shopstack/infra/k8s/frontend-deployment.yaml
kubectl apply -f ~/shopstack/infra/k8s/worker-deployment.yaml

kubectl get deployments
kubectl get pods
```

Expected:
```
NAME               READY   UP-TO-DATE   AVAILABLE
shopstack-api      2/2     2            2
shopstack-db       1/1     1            1
shopstack-frontend 2/2     2            2
shopstack-worker   1/1     1            1
```

---

### Step 3 — Prove self-healing

```bash
# Get the name of one API pod
kubectl get pods | grep api

# Delete it
kubectl delete pod shopstack-api-XXXXX

# Watch the Deployment create a replacement immediately
kubectl get pods -w
```

Expected: deleted Pod disappears, new Pod appears within seconds. This is self-healing.

---

### Step 4 — Inspect Deployment rollout history

```bash
kubectl rollout history deployment/shopstack-api
```

Expected: revision 1 listed

---

### Step 5 — Do a rolling update

```bash
# Update the API image tag (simulate a new release)
kubectl set image deployment/shopstack-api api=akhiltejadoosari/shopstack-api:latest

# Watch the rolling update — one Pod at a time
kubectl rollout status deployment/shopstack-api
```

Expected: `Waiting for rollout to finish: 1 out of 2 new replicas have been updated...` then `successfully rolled out`

---

### Step 6 — Roll back

```bash
# Undo the last deployment
kubectl rollout undo deployment/shopstack-api

# Verify rollback
kubectl rollout status deployment/shopstack-api
kubectl get pods
```

Expected: Deployment rolls back to the previous image tag.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. All Deployments running
kubectl get deployments
```
Expected: all 4 Deployments with READY matching desired count

---

```bash
# 2. Correct number of pods
kubectl get pods | grep -c Running
```
Expected: `6` (2 api + 1 db + 2 frontend + 1 worker)

---

```bash
# 3. Self-healing works
API_POD=$(kubectl get pods -l tier=api -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $API_POD
sleep 10
kubectl get pods -l tier=api
```
Expected: 2 API pods running (one new, one old)

---

```bash
# 4. Rollout history exists
kubectl rollout history deployment/shopstack-api
```
Expected: at least 1 revision listed

---

```bash
# 5. Rollback works
kubectl rollout undo deployment/shopstack-api
kubectl rollout status deployment/shopstack-api
```
Expected: `successfully rolled out`

**All 5 green?** → Save file → push → Session 04 — Services.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Deployment shows 0/2 ready | Pods crashing — check logs | `kubectl logs PODNAME` |
| `selector does not match` error | Mismatch between selector and pod labels | Labels in template must match selector exactly |
| Rolling update stuck | New Pod won't start | `kubectl describe pod NEWPOD` → read Events |
| `apps/v1` rejected | Using wrong apiVersion | Deployments need `apps/v1` not `v1` |

---

## Quick Reference

| What | Command |
|---|---|
| Apply deployment | `kubectl apply -f FILE` |
| List deployments | `kubectl get deployments` |
| Describe deployment | `kubectl describe deployment NAME` |
| Scale deployment | `kubectl scale deployment NAME --replicas=3` |
| Update image | `kubectl set image deployment/NAME CONTAINER=IMAGE:TAG` |
| Rollout status | `kubectl rollout status deployment/NAME` |
| Rollout history | `kubectl rollout history deployment/NAME` |
| Rollback | `kubectl rollout undo deployment/NAME` |
| Delete deployment | `kubectl delete deployment NAME` |
