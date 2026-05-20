# Session 06 — Full ShopStack Clean Run

**Goal:** Tear down everything, apply all manifests from scratch in the correct order, verify the full stack is healthy and accessible from the browser.

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

Sessions 01-05 built ShopStack incrementally — one object at a time, sometimes out of order, sometimes with debugging detours. This session proves you can deploy the full stack cleanly from a blank cluster, in the correct order, with zero manual fixes needed. That is the real deliverable.

This is the session you describe in your interview when they ask: "Have you deployed a multi-tier application to Kubernetes?"

---

## Visual Map — Full ShopStack on k3s

```
infra/k8s/
├── db-configmap.yaml          → ConfigMap: db-config
├── db-secret.yaml             → Secret: db-secret
├── db-pvc.yaml                → PVC: db-pvc (1Gi, Bound)
├── db-deployment.yaml         → Deployment: shopstack-db (1 replica)
├── db-service.yaml            → Service: db (ClusterIP :5432)
├── api-deployment.yaml        → Deployment: shopstack-api (2 replicas)
├── api-service.yaml           → Service: api (ClusterIP :8080)
├── worker-deployment.yaml     → Deployment: shopstack-worker (1 replica)
├── worker-service.yaml        → Service: worker (ClusterIP :8080)
├── frontend-deployment.yaml   → Deployment: shopstack-frontend (2 replicas)
└── frontend-service.yaml      → Service: frontend (NodePort :30080)

Deploy order:
1. ConfigMap + Secret + PVC  (no dependencies)
2. db Deployment + Service   (needs PVC + Secret + ConfigMap)
3. api Deployment + Service  (needs db Service to exist for DNS)
4. worker Deployment         (needs api Service)
5. frontend Deployment + Service (needs api Service for nginx proxy)
```

---

## The Files — Final State of Every Manifest

All files shown here are the final versions from Session 05 — with Secrets, ConfigMaps, and PVC wired in.

### 📄 infra/k8s/db-configmap.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: "db"
  DB_PORT: "5432"
  DB_NAME: "shopstack"
```

</details>

---

### 📄 infra/k8s/db-secret.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_USER: c2hvcHN0YWNr
  DB_PASS: c2hvcHN0YWNrX2Rldg==
  POSTGRES_DB: c2hvcHN0YWNr
  POSTGRES_USER: c2hvcHN0YWNr
  POSTGRES_PASSWORD: c2hvcHN0YWNrX2Rldg==
```

</details>

---

### 📄 infra/k8s/db-pvc.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

</details>

---

### 📄 infra/k8s/db-deployment.yaml

<details>
<summary>📄 Show full file</summary>

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
                name: db-secret
            - configMapRef:
                name: db-config
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: db-pvc
```

</details>

---

### 📄 infra/k8s/db-service.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ClusterIP
  selector:
    app: shopstack
    tier: db
  ports:
    - port: 5432
      targetPort: 5432
```

</details>

---

### 📄 infra/k8s/api-deployment.yaml

<details>
<summary>📄 Show full file</summary>

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
                name: db-secret
            - configMapRef:
                name: db-config
```

</details>

---

### 📄 infra/k8s/api-service.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  type: ClusterIP
  selector:
    app: shopstack
    tier: api
  ports:
    - port: 8080
      targetPort: 8080
```

</details>

---

### 📄 infra/k8s/worker-deployment.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopstack-worker
  labels:
    app: shopstack
    tier: worker
spec:
  replicas: 1
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
              value: "http://api:8080"
```

</details>

---

### 📄 infra/k8s/worker-service.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: worker
spec:
  type: ClusterIP
  selector:
    app: shopstack
    tier: worker
  ports:
    - port: 8080
      targetPort: 8080
```

</details>

---

### 📄 infra/k8s/frontend-deployment.yaml

<details>
<summary>📄 Show full file</summary>

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

### 📄 infra/k8s/frontend-service.yaml

<details>
<summary>📄 Show full file</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: shopstack
    tier: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing. Start from a clean cluster.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Tear down everything

```bash
kubectl delete deployments --all
kubectl delete services --all --field-selector metadata.name!=kubernetes
kubectl delete configmaps --all --field-selector metadata.name!=kube-root-ca.crt
kubectl delete secrets --all --field-selector type=Opaque
kubectl delete pvc --all

kubectl get pods
kubectl get services
kubectl get pvc
```

Expected: no ShopStack pods, no services except `kubernetes`, no PVCs.

---

### Step 2 — Apply in correct order

```bash
cd ~/shopstack/infra/k8s

# 1. State objects first — no dependencies
kubectl apply -f db-configmap.yaml
kubectl apply -f db-secret.yaml
kubectl apply -f db-pvc.yaml

# Verify PVC is Bound before continuing
kubectl get pvc
```

Wait for `db-pvc` to show `Bound` before proceeding.

```bash
# 2. Database tier
kubectl apply -f db-deployment.yaml
kubectl apply -f db-service.yaml

# Wait for db to be ready
kubectl rollout status deployment/shopstack-db
```

```bash
# 3. API tier
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml

kubectl rollout status deployment/shopstack-api
```

```bash
# 4. Worker
kubectl apply -f worker-deployment.yaml
kubectl apply -f worker-service.yaml
```

```bash
# 5. Frontend
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml

kubectl rollout status deployment/shopstack-frontend
```

---

### Step 3 — Full stack health check

```bash
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get pvc
```

Expected pods:
```
NAME                                  READY   STATUS    RESTARTS
shopstack-api-xxxxx                   1/1     Running   0
shopstack-api-xxxxx                   1/1     Running   0
shopstack-db-xxxxx                    1/1     Running   0
shopstack-frontend-xxxxx              1/1     Running   0
shopstack-frontend-xxxxx              1/1     Running   0
shopstack-worker-xxxxx                1/1     Running   0
```

---

### Step 4 — Verify all endpoints

```bash
EC2_IP=$(curl -s http://checkip.amazonaws.com)

curl -s http://localhost:30080/api/health
curl -s http://localhost:30080/api/products | head -c 100
curl -s http://localhost:30080/api/orders | head -c 100
curl -s http://localhost:30080/api/metrics | head -c 100
```

All four should return valid responses.

---

### Step 5 — Open in browser

```
http://YOUR_EC2_IP:30080        ← ShopStack store
```

Expected: store UI loads, green health banner, products visible, can click buy.

---

### Step 6 — Verify worker health pings

```bash
WORKER=$(kubectl get pods -l tier=worker -o jsonpath='{.items[0].metadata.name}')
kubectl logs $WORKER | tail -5
```

Expected: `health_ping_ok` JSON lines every 10 seconds.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. All 7 pods running
kubectl get pods | grep -c "Running"
```
Expected: `7` (2 api + 1 db + 2 frontend + 1 worker + adminer if deployed)

---

```bash
# 2. All deployments healthy
kubectl get deployments | grep -v "^NAME" | awk '{print $1, $2, $4}' | grep -v "^$"
```
Expected: all deployments show READY = AVAILABLE

---

```bash
# 3. PVC Bound
kubectl get pvc | grep Bound
```
Expected: `db-pvc` with status `Bound`

---

```bash
# 4. API health
curl -s http://localhost:30080/api/health | grep '"status":"ok"'
```
Expected: `"status":"ok"`

---

```bash
# 5. Products return data
curl -s http://localhost:30080/api/products | python3 -c "import sys,json; data=json.load(sys.stdin); print(f'{len(data)} products')"
```
Expected: `6 products`

---

```bash
# 6. Data persists across db pod restart
DB_POD=$(kubectl get pods -l tier=db -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $DB_POD
sleep 25
curl -s http://localhost:30080/api/products | python3 -c "import sys,json; data=json.load(sys.stdin); print(f'{len(data)} products')"
```
Expected: `6 products` — data survived pod deletion

**All 6 green?** → Save file → push → Kubernetes complete → CI/CD Session 01.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| API pods in CrashLoopBackOff | DB not ready when API started | Wait for db rollout then apply api |
| PVC stuck Pending | local-path-provisioner issue | `kubectl get pods -n kube-system \| grep local-path` |
| 502 on frontend | API service endpoints empty | `kubectl get endpoints api` — check labels |
| Products empty after restart | PVC not mounted | Check `volumeMounts` in db-deployment.yaml |
| Worker logs show ping failed | API service not ready yet | Wait 30s after api deployment |

---

## Quick Reference — Full Stack Commands

| What | Command |
|---|---|
| Apply all at once | `kubectl apply -f ~/shopstack/infra/k8s/` |
| Delete all ShopStack | `kubectl delete -f ~/shopstack/infra/k8s/` |
| Watch all pods | `kubectl get pods -w` |
| Full health scan | `kubectl get pods,services,pvc,deployments` |
| API health | `curl http://localhost:30080/api/health` |
| EC2 IP | `curl -s http://checkip.amazonaws.com` |
