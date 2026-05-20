# Session 04 — Services

**Goal:** Write Service manifests for all ShopStack services, wire all tiers together via Kubernetes DNS, expose the frontend via NodePort and hit it from the browser.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Services Exist

Pods are ephemeral. Every time a Pod restarts it gets a new IP. If the API connected to the DB by IP, it would lose the connection on every DB restart.

A Service gives Pods a stable name and IP that never changes. The Service finds Pods by their labels — not by IP. Pods restart, IPs change, labels stay the same.

```
Without Service:
  api Pod connects to db Pod IP 10.42.0.14
  db Pod restarts → new IP 10.42.0.17
  api cannot find db anymore

With Service:
  api connects to "db" (Service name)
  Kubernetes DNS resolves "db" → db Service ClusterIP (stable, never changes)
  db Service routes to whatever db Pod is running right now
```

---

## Visual Map — ShopStack Services

```
Browser
    │
    │ http://EC2_IP:30080
    ▼
frontend-service (NodePort :30080)
    │
    ▼
frontend Pods (:80)
    │ nginx proxy_pass http://api:8080
    ▼
api-service (ClusterIP :8080)
    │
    ▼
api Pods (:8080)
    │ DB_HOST=db
    ▼
db-service (ClusterIP :5432)
    │
    ▼
db Pod (:5432)

worker Pod → http://api:8080/api/health (every 10s)
```

---

## Service Types

| Type | Who can reach it | Use for |
|---|---|---|
| `ClusterIP` | Only Pods inside the cluster | api, db, worker — internal services |
| `NodePort` | Anyone with EC2 IP + port | frontend, adminer — browser access |
| `LoadBalancer` | Public internet via AWS ALB | Production EKS — not k3s |

---

## The Three Port Fields

```yaml
ports:
  - port: 8080        # port other Pods use to reach this Service
    targetPort: 8080  # port on the Pod the Service forwards traffic to
    nodePort: 30080   # (NodePort only) port exposed on the EC2 instance
```

Most common mistake: `targetPort` doesn't match the port the container actually listens on → traffic reaches the Service but never reaches the Pod.

---

## The Files — All ShopStack Service Manifests

### 📄 infra/k8s/frontend-service.yaml

<details>
<summary>📄 Show full file — infra/k8s/frontend-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend            # name used internally — not for DNS routing
spec:
  type: NodePort            # exposes on EC2 host at nodePort
  selector:
    app: shopstack          # finds Pods with this label
    tier: frontend          # must match labels on frontend Deployment pods
  ports:
    - port: 80              # Service listens on port 80 inside cluster
      targetPort: 80        # forwards to port 80 on the frontend Pod (nginx)
      nodePort: 30080       # accessible at http://EC2_IP:30080 from browser
```

</details>

---

### 📄 infra/k8s/api-service.yaml

<details>
<summary>📄 Show full file — infra/k8s/api-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api                 # THIS NAME is what nginx uses in proxy_pass http://api:8080
                            # Kubernetes DNS resolves "api" → this Service's ClusterIP
spec:
  type: ClusterIP           # only reachable inside the cluster
  selector:
    app: shopstack
    tier: api
  ports:
    - port: 8080
      targetPort: 8080      # API container listens on 8080
```

</details>

---

### 📄 infra/k8s/db-service.yaml

<details>
<summary>📄 Show full file — infra/k8s/db-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db                  # THIS NAME is what DB_HOST=db resolves to
                            # If you rename this → api cannot find the database
spec:
  type: ClusterIP
  selector:
    app: shopstack
    tier: db
  ports:
    - port: 5432
      targetPort: 5432      # Postgres listens on 5432
```

</details>

---

### 📄 infra/k8s/worker-service.yaml

<details>
<summary>📄 Show full file — infra/k8s/worker-service.yaml</summary>

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

### 📄 infra/k8s/adminer-service.yaml

<details>
<summary>📄 Show full file — infra/k8s/adminer-service.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: adminer
spec:
  type: NodePort
  selector:
    app: shopstack
    tier: adminer
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30081       # http://EC2_IP:30081 → Adminer UI
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Deployments from Session 03 must be running
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Daily opening sequence

```bash
kubectl get nodes
kubectl get pods
kubectl get deployments
```

All Deployments should be running from Session 03.

---

### Step 2 — Apply all Services

```bash
kubectl apply -f ~/shopstack/infra/k8s/frontend-service.yaml
kubectl apply -f ~/shopstack/infra/k8s/api-service.yaml
kubectl apply -f ~/shopstack/infra/k8s/db-service.yaml
kubectl apply -f ~/shopstack/infra/k8s/worker-service.yaml

kubectl get services
```

Expected:
```
NAME         TYPE        CLUSTER-IP      PORT(S)
api          ClusterIP   10.96.xx.xx     8080/TCP
db           ClusterIP   10.96.xx.xx     5432/TCP
frontend     NodePort    10.96.xx.xx     80:30080/TCP
kubernetes   ClusterIP   10.96.0.1       443/TCP
worker       ClusterIP   10.96.xx.xx     8080/TCP
```

---

### Step 3 — Verify Services have Endpoints (critical)

```bash
kubectl get endpoints
```

Expected — all Services show Pod IPs, not `<none>`:
```
NAME       ENDPOINTS
api        10.42.x.x:8080,10.42.x.x:8080
db         10.42.x.x:5432
frontend   10.42.x.x:80,10.42.x.x:80
worker     10.42.x.x:8080
```

If a Service shows `<none>` → labels mismatch between Service selector and Pod labels.

---

### Step 4 — Test Kubernetes DNS

```bash
# From inside an API pod, can it find the DB by name?
API_POD=$(kubectl get pods -l tier=api -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $API_POD -- sh

# Inside the pod
nslookup db
# Expected: returns ClusterIP of the db Service

wget -qO- http://api:8080/api/health
# Expected: {"status":"ok",...}

exit
```

---

### Step 5 — Hit ShopStack from the browser

```bash
EC2_IP=$(curl -s http://checkip.amazonaws.com)
echo "Open: http://$EC2_IP:30080"

# Test from EC2
curl http://localhost:30080
curl http://localhost:30080/api/health
curl http://localhost:30080/api/products
```

Open `http://YOUR_EC2_IP:30080` in browser → ShopStack store UI should load.

---

### Step 6 — Verify worker is connecting to API via Service

```bash
WORKER_POD=$(kubectl get pods -l tier=worker -o jsonpath='{.items[0].metadata.name}')
kubectl logs $WORKER_POD | tail -5
```

Expected: JSON log lines with `health_ping_ok` → worker found the API via DNS.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. All 5 services exist
kubectl get services | grep -v kubernetes | grep -c ClusterIP\|NodePort
```
Expected: `5`

---

```bash
# 2. All services have endpoints (not <none>)
kubectl get endpoints | grep -v "^NAME" | grep "<none>"
```
Expected: no output — all services have endpoints

---

```bash
# 3. Kubernetes DNS works
API_POD=$(kubectl get pods -l tier=api -o jsonpath='{.items[0].metadata.name}')
kubectl exec $API_POD -- nslookup db 2>/dev/null | grep Address
```
Expected: IP address returned for "db"

---

```bash
# 4. API health via NodePort
curl -s http://localhost:30080/api/health | grep '"status":"ok"'
```
Expected: `"status":"ok"` in response

---

```bash
# 5. Products endpoint returns data
curl -s http://localhost:30080/api/products | head -c 100
```
Expected: JSON array with products

---

```bash
# 6. Worker logs show successful pings
WORKER_POD=$(kubectl get pods -l tier=worker -o jsonpath='{.items[0].metadata.name}')
kubectl logs $WORKER_POD | grep "health_ping_ok" | tail -2
```
Expected: two lines with `health_ping_ok`

**All 6 green?** → Save file → push → Session 05 — Secrets, ConfigMaps, PVCs.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Service has `<none>` endpoints | Label mismatch — selector doesn't match Pod labels | `kubectl describe service NAME` → check selector |
| `nslookup db` fails from inside pod | Service name wrong or not applied | `kubectl get services` → verify `db` exists |
| 502 Bad Gateway on frontend | API service not routing — check endpoints | `kubectl get endpoints api` |
| NodePort not accessible | Security group missing port 30080 | AWS Console → SG → add 30080 inbound |
| `DB_HOST=db` fails | Service named differently or not applied | Service name must be exactly `db` |

---

## Quick Reference

| What | Command |
|---|---|
| List services | `kubectl get services` |
| Check endpoints | `kubectl get endpoints` |
| Describe service | `kubectl describe service NAME` |
| Test DNS from pod | `kubectl exec POD -- nslookup SERVICE` |
| Delete service | `kubectl delete service NAME` |
| EC2 IP | `curl -s http://checkip.amazonaws.com` |
