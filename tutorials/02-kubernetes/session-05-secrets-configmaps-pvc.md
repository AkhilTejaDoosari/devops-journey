# Session 05 — Secrets, ConfigMaps, PVC

**Goal:** Move hardcoded credentials out of Deployment manifests into Secrets and ConfigMaps, add a PVC so Postgres data survives Pod restarts, prove data persistence works.

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

Session 03 Deployments had passwords in plain YAML. If that file goes to GitHub — credentials are exposed. Session 05 fixes this properly.

```
Session 03 (wrong):
  env:
    - name: DB_PASS
      value: "shopstack_dev"    ← plain text in version control

Session 05 (correct):
  envFrom:
    - secretRef:
        name: db-secret         ← credentials live in Secret, not in the manifest
```

And Postgres data was dying on every Pod restart because no PVC was attached. Session 05 fixes that too.

---

## Visual Map

```
db-configmap.yaml    → non-sensitive config  → injected into api + db Deployments
db-secret.yaml       → sensitive credentials → injected into api + db Deployments
db-pvc.yaml          → disk claim            → mounted at /var/lib/postgresql/data

db Deployment (updated):
  envFrom:
    - secretRef:    db-secret      (POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB)
    - configMapRef: db-config      (DB_HOST, DB_PORT, DB_NAME)
  volumeMounts:
    - db-pvc → /var/lib/postgresql/data

Pod restarts → new Pod mounts same PVC → Postgres continues with existing data
```

---

## ConfigMap vs Secret — The Rule

| What | Use | Why |
|---|---|---|
| `DB_HOST=db` | ConfigMap | Would you put this in a public GitHub repo? Yes. |
| `DB_PORT=5432` | ConfigMap | Yes — it's not sensitive. |
| `DB_NAME=shopstack` | ConfigMap | Yes. |
| `DB_PASSWORD=shopstack_dev` | Secret | No — never commit passwords. |
| `DB_USER=shopstack` | Secret | No — usernames are sensitive too. |

---

## The Files — All Three Objects

### 📄 infra/k8s/db-configmap.yaml

**What it does:** stores non-sensitive DB connection config. Injected into both API and DB Deployments.

<details>
<summary>📄 Show full file — infra/k8s/db-configmap.yaml</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config              # referenced by name in Deployment envFrom
data:
  DB_HOST: "db"                # Service name — Kubernetes DNS resolves this
  DB_PORT: "5432"              # quoted string — Kubernetes injects as string
  DB_NAME: "shopstack"         # database name — not sensitive
```

</details>

---

### 📄 infra/k8s/db-secret.yaml

**What it does:** stores credentials as base64-encoded values. Never commit this file to GitHub with real passwords.

**Encoding the values:**
```bash
echo -n "shopstack" | base64
# → c2hvcHN0YWNr

echo -n "shopstack_dev" | base64
# → c2hvcHN0YWNrX2Rldg==
```

> ⚠️ The `-n` flag is critical. Without it, `echo` adds a newline. The encoded value becomes wrong. Postgres fails to authenticate with a cryptic error.

<details>
<summary>📄 Show full file — infra/k8s/db-secret.yaml</summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret              # referenced by name in Deployment envFrom
type: Opaque                   # Opaque = generic key-value Secret (default)
data:
  # base64 encoded — NOT encrypted. Do not commit to GitHub.
  # Decode: kubectl get secret db-secret -o jsonpath='{.data.DB_USER}' | base64 -d

  DB_USER: c2hvcHN0YWNr               # "shopstack"
  DB_PASS: c2hvcHN0YWNrX2Rldg==       # "shopstack_dev"
  POSTGRES_DB: c2hvcHN0YWNr           # "shopstack" — Postgres reads this on init
  POSTGRES_USER: c2hvcHN0YWNr         # "shopstack" — Postgres reads this on init
  POSTGRES_PASSWORD: c2hvcHN0YWNrX2Rldg==  # "shopstack_dev" — Postgres reads this
```

</details>

---

### 📄 infra/k8s/db-pvc.yaml

**What it does:** requests 1Gi of disk storage from k3s's local storage provisioner. Postgres mounts this at `/var/lib/postgresql/data`.

<details>
<summary>📄 Show full file — infra/k8s/db-pvc.yaml</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc                 # referenced by name in db Deployment volumes section
spec:
  accessModes:
    - ReadWriteOnce            # one node can read/write at a time
                               # correct for a single Postgres instance
  resources:
    requests:
      storage: 1Gi             # request 1 gigabyte — k3s local storage provisions this
```

</details>

---

### 📄 infra/k8s/db-deployment.yaml — Updated with Secret, ConfigMap, PVC

**What changed from Session 03:** plain env vars replaced with `envFrom`, PVC mounted at Postgres data directory.

<details>
<summary>📄 Show full updated file — infra/k8s/db-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-db
  labels:
    app: shopstack
    tier: db
spec:
  replicas: 1
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
          envFrom:
            - secretRef:
                name: db-secret      # injects: POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD
                                     # also injects: DB_USER, DB_PASS (for API compatibility)
            - configMapRef:
                name: db-config      # injects: DB_HOST, DB_PORT, DB_NAME
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data  # where Postgres stores all data
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: db-pvc        # reference the PVC by name
```

</details>

---

### 📄 infra/k8s/api-deployment.yaml — Updated with Secret and ConfigMap

<details>
<summary>📄 Show full updated file — infra/k8s/api-deployment.yaml</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-api
  labels:
    app: shopstack
    tier: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: shopstack
      tier: api
  template:
    metadata:
      labels:
        app: shopstack
        tier: api
    spec:
      containers:
        - name: api
          image: akhiltejadoosari/shopstack-api:1.0
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: db-secret      # injects: DB_USER, DB_PASS
            - configMapRef:
                name: db-config      # injects: DB_HOST, DB_PORT, DB_NAME
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Deployments and Services from Sessions 03-04 must be running
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Daily opening sequence

```bash
kubectl get nodes
kubectl get pods
kubectl get services
```

---

### Step 2 — Apply ConfigMap and Secret

```bash
kubectl apply -f ~/shopstack/infra/k8s/db-configmap.yaml
kubectl apply -f ~/shopstack/infra/k8s/db-secret.yaml

kubectl get configmaps | grep db-config
kubectl get secrets | grep db-secret
```

---

### Step 3 — Apply the PVC

```bash
kubectl apply -f ~/shopstack/infra/k8s/db-pvc.yaml
kubectl get pvc
```

Expected:
```
NAME     STATUS   VOLUME         CAPACITY   ACCESS MODES
db-pvc   Bound    pvc-xxx-xxx    1Gi        RWO
```

`Bound` = storage provisioned and ready. `Pending` = problem.

---

### Step 4 — Update Deployments to use Secret, ConfigMap, PVC

```bash
kubectl apply -f ~/shopstack/infra/k8s/db-deployment.yaml
kubectl apply -f ~/shopstack/infra/k8s/api-deployment.yaml

kubectl rollout status deployment/shopstack-db
kubectl rollout status deployment/shopstack-api
```

---

### Step 5 — Verify credentials injected correctly

```bash
DB_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')
kubectl exec $DB_POD -- env | grep -E "POSTGRES|DB_"
```

Expected: all DB environment variables present with correct values

---

### Step 6 — Verify Secret values are correct

```bash
kubectl get secret db-secret -o jsonpath='{.data.DB_PASS}' | base64 -d
```

Expected: `shopstack_dev`

---

### Step 7 — Prove data survives Pod restart

```bash
# Get the DB Pod name
DB_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')

# Insert test data
kubectl exec $DB_POD -- psql -U shopstack -d shopstack \
  -c "INSERT INTO inventory.products (name, category, price, stock) VALUES ('Test Product', 'test', 9.99, 1);"

# Verify row exists
kubectl exec $DB_POD -- psql -U shopstack -d shopstack \
  -c "SELECT name FROM inventory.products WHERE category='test';"

# Delete the Pod — Deployment recreates it, PVC keeps data
kubectl delete pod $DB_POD

# Wait for new Pod
kubectl get pods -l tier=db -w
# Ctrl+C when Running

# New Pod — data still there
NEW_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -- psql -U shopstack -d shopstack \
  -c "SELECT name FROM inventory.products WHERE category='test';"
```

Expected: `Test Product` row still exists after Pod deletion. PVC works.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. ConfigMap exists
kubectl get configmap db-config
```
Expected: `db-config` listed

---

```bash
# 2. Secret exists
kubectl get secret db-secret
```
Expected: `db-secret` listed

---

```bash
# 3. PVC is Bound
kubectl get pvc db-pvc | grep Bound
```
Expected: `Bound` in output

---

```bash
# 4. Credentials decoded correctly
kubectl get secret db-secret -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```
Expected: `shopstack_dev`

---

```bash
# 5. API health still works
curl -s http://localhost:30080/api/health | grep '"status":"ok"'
```
Expected: `"status":"ok"`

---

```bash
# 6. Data survives pod restart
DB_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')
kubectl exec $DB_POD -- psql -U shopstack -d shopstack -c "SELECT COUNT(*) FROM inventory.products;"
kubectl delete pod $DB_POD
sleep 20
NEW_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -- psql -U shopstack -d shopstack -c "SELECT COUNT(*) FROM inventory.products;"
```
Expected: same count before and after Pod deletion

**All 6 green?** → Save file → push → Session 06 — Full Clean Run.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| `base64: invalid input` | Encoded with newline — missing `-n` flag | Re-encode: `echo -n "value" \| base64` |
| Postgres auth fails after Secret | Secret keys don't match — Postgres needs `POSTGRES_*` | Check Secret has both `DB_*` and `POSTGRES_*` keys |
| PVC stuck in `Pending` | local-path-provisioner not running | `kubectl get pods -n kube-system \| grep local-path` |
| Data lost after Pod restart | PVC not mounted — missing volumeMounts | Check `volumeMounts` and `volumes` in db Deployment |
| `envFrom` not working | Secret or ConfigMap name mismatch | Check name matches exactly in `envFrom` reference |

---

## Quick Reference

| What | Command |
|---|---|
| Apply ConfigMap | `kubectl apply -f db-configmap.yaml` |
| Apply Secret | `kubectl apply -f db-secret.yaml` |
| Apply PVC | `kubectl apply -f db-pvc.yaml` |
| List ConfigMaps | `kubectl get configmaps` |
| List Secrets | `kubectl get secrets` |
| List PVCs | `kubectl get pvc` |
| Decode Secret value | `kubectl get secret NAME -o jsonpath='{.data.KEY}' \| base64 -d` |
| Encode value | `echo -n "value" \| base64` |
| Check env vars in pod | `kubectl exec POD -- env` |
