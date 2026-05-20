# Session 02 — Pods

**Goal:** Understand the YAML manifest structure, write Pod manifests for all ShopStack services, apply them, and prove why bare Pods are not enough — they don't self-heal.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why This Session Exists

You learn bare Pods today specifically to feel what breaks without a Deployment. Delete a Pod → it stays dead → that moment of realisation IS why Deployments exist. You can't appreciate Deployments until you've felt the pain of a bare Pod.

---

## Visual Map

```
Kubernetes Object Hierarchy:

  Deployment        ← Session 03 (you write this)
    └── ReplicaSet  ← K8s creates this automatically
          └── Pod   ← Session 02 (you understand this today)
                └── Container  ← your app runs here
```

---

## The Four Pillars of Every Manifest

Every Kubernetes manifest starts with the same four fields. API Server reads these first — one wrong and it rejects the entire file.

```yaml
apiVersion: v1       # which K8s API version — v1 for Pod/Service/ConfigMap/Secret
                     # apps/v1 for Deployment — different group

kind: Pod            # what object type — case sensitive, always capitalised

metadata:
  name: shopstack-api    # unique identity in the cluster
  labels:
    app: shopstack        # searchable tag — Services find Pods by these
    tier: api

spec:                # the blueprint — what goes inside
  containers:        # everything below is specific to the kind declared above
    - name: api
      image: akhiltejadoosari/shopstack-api:1.0
```

---

## Labels and Selectors — The Glue

Labels are stamps on objects. Selectors are search filters that find objects by their stamps.

```
Pod has label:        app: shopstack, tier: api
Service has selector: app: shopstack, tier: api
→ Service finds the Pod. Traffic flows.

One typo in either → they are invisible to each other.
```

**Why labels instead of names or IPs:**
- Pod names change every restart
- Pod IPs change every restart
- Labels stay the same across every replacement

This is how Kubernetes routes traffic to ephemeral Pods without losing them.

---

## The Files — ShopStack Pod Manifests

These files live in `infra/k8s/`. You apply them with `kubectl apply -f`.

### 📄 infra/k8s/api-pod.yaml

<details>
<summary>📄 Show full file — infra/k8s/api-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shopstack-api          # unique name in default namespace
  labels:
    app: shopstack              # Service selector matches this
    tier: api                   # distinguishes from other ShopStack pods
spec:
  containers:
    - name: api
      image: akhiltejadoosari/shopstack-api:1.0
      ports:
        - containerPort: 8080  # documentation — the port the app listens on
      env:
        - name: DB_HOST
          value: "db"          # Kubernetes DNS resolves "db" → db Service ClusterIP
        - name: DB_PORT
          value: "5432"        # quoted string — prevents K8s env var collision
        - name: DB_NAME
          value: "shopstack"
        - name: DB_USER
          value: "shopstack"
        - name: DB_PASS
          value: "shopstack_dev"   # moves to Secret in Session 05
```

</details>

---

### 📄 infra/k8s/db-pod.yaml

<details>
<summary>📄 Show full file — infra/k8s/db-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shopstack-db
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
          value: "shopstack_dev"    # moves to Secret in Session 05
```

</details>

---

### 📄 infra/k8s/frontend-pod.yaml

<details>
<summary>📄 Show full file — infra/k8s/frontend-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shopstack-frontend
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

### 📄 infra/k8s/worker-pod.yaml

<details>
<summary>📄 Show full file — infra/k8s/worker-pod.yaml</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shopstack-worker
  labels:
    app: shopstack
    tier: worker
spec:
  containers:
    - name: worker
      image: akhiltejadoosari/shopstack-worker:1.0
      env:
        - name: API_URL
          value: "http://api:8080"   # DNS resolves "api" → api Service ClusterIP
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — k3s is already running from Session 01
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Run the daily opening sequence

```bash
kubectl get nodes          # cluster responding?
kubectl get pods -n kube-system  # system pods healthy?
kubectl get pods           # anything already running?
```

---

### Step 2 — Create the manifest directory

```bash
ls ~/shopstack/infra/k8s/
```

Manifests already exist from previous work. Verify they're there.

---

### Step 3 — Apply the db Pod first

```bash
kubectl apply -f ~/shopstack/infra/k8s/db-pod.yaml
kubectl get pods -w
```

Watch it start. Press `Ctrl+C` when STATUS shows `Running`.

---

### Step 4 — Apply remaining Pods

```bash
kubectl apply -f ~/shopstack/infra/k8s/api-pod.yaml
kubectl apply -f ~/shopstack/infra/k8s/frontend-pod.yaml
kubectl apply -f ~/shopstack/infra/k8s/worker-pod.yaml

kubectl get pods
```

Expected:
```
NAME                 READY   STATUS    AGE
shopstack-api        1/1     Running   30s
shopstack-db         1/1     Running   60s
shopstack-frontend   1/1     Running   20s
shopstack-worker     1/1     Running   15s
```

---

### Step 5 — Inspect a Pod

```bash
# See the full picture — events are at the bottom
kubectl describe pod shopstack-api

# See what the app printed on startup
kubectl logs shopstack-api

# Follow logs live
kubectl logs -f shopstack-api
```

---

### Step 6 — Get a shell inside a Pod

```bash
kubectl exec -it shopstack-db -- sh

# Inside the pod
psql -U shopstack -d shopstack -c "\l"
exit
```

---

### Step 7 — Prove bare Pods don't self-heal (the critical lesson)

```bash
# Delete the API Pod
kubectl delete pod shopstack-api

# Watch what happens
kubectl get pods -w
```

Expected: `shopstack-api` disappears and DOES NOT come back.

This is why Deployments exist. A bare Pod has no watchdog. When it dies, it stays dead. In production, every app runs in a Deployment — never as a bare Pod.

---

### Step 8 — Clean up all Pods

```bash
kubectl delete pod shopstack-api shopstack-db shopstack-frontend shopstack-worker
kubectl get pods
```

Expected: no pods in default namespace.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Apply db pod
kubectl apply -f ~/shopstack/infra/k8s/db-pod.yaml
sleep 20
kubectl get pods | grep shopstack-db
```
Expected: `shopstack-db   1/1   Running`

---

```bash
# 2. Apply API pod
kubectl apply -f ~/shopstack/infra/k8s/api-pod.yaml
sleep 10
kubectl get pods | grep shopstack-api
```
Expected: `shopstack-api   1/1   Running`

---

```bash
# 3. Logs work
kubectl logs shopstack-db | tail -3
```
Expected: Postgres startup messages

---

```bash
# 4. Describe works
kubectl describe pod shopstack-api | grep -A 5 "Events:"
```
Expected: Events section shows `Pulled`, `Created`, `Started`

---

```bash
# 5. Bare Pod does NOT self-heal — the critical lesson
kubectl delete pod shopstack-api
sleep 5
kubectl get pods | grep shopstack-api
```
Expected: no output — Pod is gone and not coming back

---

```bash
# 6. Clean up
kubectl delete pod shopstack-db shopstack-frontend shopstack-worker 2>/dev/null || true
kubectl get pods
```
Expected: no pods

**All 6 green?** → Save file → push → Session 03 — Deployments.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Pod stuck in `Pending` | No resources or PVC not bound | `kubectl describe pod NAME` → read Events |
| Pod in `CrashLoopBackOff` | App crashes on start | `kubectl logs NAME` → read error |
| Pod stuck in `ImagePullBackOff` | Image doesn't exist on Docker Hub | Check image name and tag |
| `error: invalid apiVersion` | Typo in apiVersion | Pod = `v1`, Deployment = `apps/v1` |
| Labels don't match selector | Typo in label or selector | `kubectl describe service NAME` → check Endpoints |

---

## Quick Reference

| What | Command |
|---|---|
| Apply manifest | `kubectl apply -f FILE` |
| Apply all in folder | `kubectl apply -f FOLDER/` |
| List pods | `kubectl get pods` |
| Watch pods | `kubectl get pods -w` |
| Describe pod | `kubectl describe pod NAME` |
| Pod logs | `kubectl logs NAME` |
| Follow logs | `kubectl logs -f NAME` |
| Shell in pod | `kubectl exec -it NAME -- sh` |
| Delete pod | `kubectl delete pod NAME` |
| Delete all pods | `kubectl delete pods --all` |
