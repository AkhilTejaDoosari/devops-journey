# Session 04 — Kubernetes Services

**Folder:** `~/shopstack/infra/k8s/`  
**Cluster:** k3s on EC2 t3.micro  
**Goal:** Wire all ShopStack services together. Open frontend to browser.

---

## Why Services Exist

Pods get a new IP every time they restart. You can't hardcode a Pod IP — it'll be different tomorrow.

A Service is a **stable endpoint** that sits in front of Pods. Pods die and restart. The Service never moves. Other Pods talk to the Service name, not the Pod IP.

```
Without Service:
API Pod → 10.42.0.14:5432 (hardcoded) → Pod restarts → IP changes → connection breaks

With Service:
API Pod → db:5432 (Service name) → Service finds current Pod → always works
```

---

## How a Service Finds Its Pods

A Service uses **label selectors**. Your Deployment puts labels on every Pod it creates. Your Service has a `selector` that matches those labels. That's the only connection.

```yaml
# Pod has this label (set in Deployment)
labels:
  app: shopstack
  tier: db

# Service selects by this
selector:
  app: shopstack
  tier: db        # Must match exactly. One typo → no traffic.
```

---

## Three Service Types

| Type | Who Can Reach It | ShopStack Use |
|---|---|---|
| ClusterIP | Inside cluster only | db, api, worker |
| NodePort | Outside via EC2 IP + port | frontend (browser) |
| LoadBalancer | Outside via AWS DNS | frontend on EKS (Week 5) |

---

## The 3 Port Fields

NodePort Services have three port fields. They are independent.

```yaml
ports:
  - port: 80          # Port OTHER pods use to reach this Service inside the cluster
    targetPort: 80    # Port the container is actually listening on
    nodePort: 30080   # Port punched open on the EC2 machine — browser uses this
```

**Rule:** NodePort must be between 30000–32767. You can't use 80 directly.

---

## Full Traffic Map

```
Your Browser
  │
  │  http://YOUR_EC2_IP:30080
  ▼
EC2 Machine ← AWS Security Group must allow port 30080
  │
  │  nodePort: 30080
  ▼
frontend Service (NodePort)
  │  port: 80 → targetPort: 80
  ▼
frontend Pods ×2 (nginx :80)
  │
  │  JS calls /api/* endpoints
  ▼
api Service (ClusterIP :8080)
  │
  ▼
api Pods ×2 (fastapi :8080)
  │
  │  DB_HOST=db → resolves to db Service
  ▼
db Service (ClusterIP :5432)
  │
  ▼
db Pod ×1 (postgres :5432)

🔒 db has NO external port. Internet cannot reach it directly.
```

---

## Step 1 — Write db-service.yaml

```bash
nano ~/shopstack/infra/k8s/db-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db              # THIS NAME is the DNS hostname. DB_HOST=db works because of this.
spec:
  selector:
    app: shopstack
    tier: db
  ports:
    - port: 5432
      targetPort: 5432
  type: ClusterIP       # Internal only — no external access
```

---

## Step 2 — Write api-service.yaml

```bash
nano ~/shopstack/infra/k8s/api-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: shopstack
    tier: api
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
```

---

## Step 3 — Write frontend-service.yaml

```bash
nano ~/shopstack/infra/k8s/frontend-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: shopstack
    tier: frontend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080   # Port punched on EC2 — browser uses this
  type: NodePort
```

---

## Step 4 — Apply & Verify

```bash
# Apply all three services
kubectl apply -f ~/shopstack/infra/k8s/db-service.yaml
kubectl apply -f ~/shopstack/infra/k8s/api-service.yaml
kubectl apply -f ~/shopstack/infra/k8s/frontend-service.yaml

# Confirm all exist
kubectl get svc
```

Expected output:
```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
api          ClusterIP   10.43.x.x       <none>        8080/TCP       Xs
db           ClusterIP   10.43.x.x       <none>        5432/TCP       Xs
frontend     NodePort    10.43.x.x       <none>        80:30080/TCP   Xs
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP        Xd
```

```bash
# Apply all deployments
kubectl apply -f ~/shopstack/infra/k8s/

# Check pods
kubectl get pods
```

All pods should show `1/1 Running`.

---

## Bug — DB_PORT CrashLoopBackOff

**Symptom:** API pods show `CrashLoopBackOff` after Services are created.

**Error in logs:**
```
ValueError: invalid literal for int() with base 10: 'tcp://10.43.187.201:5432'
```

**Why:** When you create a Service named `db`, Kubernetes auto-injects an env var called `DB_PORT` into every Pod with value `tcp://10.43.x.x:5432`. Your API code does `int(os.getenv("DB_PORT"))` — it explodes.

**Fix:** Explicitly set `DB_PORT` in `api-deployment.yaml`. Your value overrides K8s's auto-inject.

```yaml
# In api-deployment.yaml — add DB_PORT to env:
env:
  - name: DB_HOST
    value: "db"
  - name: DB_NAME
    value: "shopstack"
  - name: DB_USER
    value: "shopstack"
  - name: DB_PASSWORD
    value: "shopstack_dev"
  - name: DB_PORT
    value: "5432"        # ADD THIS LINE
```

```bash
kubectl apply -f ~/shopstack/infra/k8s/api-deployment.yaml
kubectl get pods   # new api pods — RESTARTS: 0
```

---

## Bug — Ghost Pods (ContainerStatusUnknown)

Dead pods from previous sessions. Harmless but messy. Clean them:

```bash
kubectl get pods | grep ContainerStatusUnknown | awk '{print $1}' | xargs kubectl delete pod
```

---

## Step 5 — Open Port 30080 in AWS

`curl localhost:30080` works on EC2 but browser times out → AWS Security Group is blocking the port.

1. AWS Console → EC2 → Instances → click your instance
2. **Security** tab → click the Security Group link
3. **Edit inbound rules** → **Add rule**
   - Type: `Custom TCP`
   - Port: `30080`
   - Source: `0.0.0.0/0`
4. Save rules

```bash
# Get your EC2 public IP
curl -s http://169.254.169.254/latest/meta-data/public-ipv4
```

Open browser: `http://YOUR_EC2_IP:30080`

**Done when:** Status bar shows `api ✓  postgres ✓  uptime: Xs`

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| API `CrashLoopBackOff` — `ValueError: invalid literal for int()` | K8s auto-injects `DB_PORT` as URL string | Add `DB_PORT: "5432"` explicitly to api-deployment.yaml |
| Browser times out on `:30080` | AWS Security Group blocking port | Add inbound rule: Custom TCP, 30080, 0.0.0.0/0 |
| `kubectl get endpoints` shows `<none>` | Service selector doesn't match Pod labels | Compare `selector:` in Service to `labels:` in Deployment Pod template |
| API can't reach DB | `DB_HOST` doesn't match Service name | Service must be named `db`, api-deployment must have `DB_HOST: "db"` |
| `curl localhost:30080` works but browser doesn't | Security Group not saved yet | Wait 10s after saving rule, retry |

---

## Quick Reference

| What | Command |
|---|---|
| List all Services | `kubectl get svc` |
| See which Pods a Service routes to | `kubectl get endpoints` |
| Full Service details | `kubectl describe service db` |
| Test DNS from inside a Pod | `kubectl exec -it <pod> -- nslookup db` |
| Read crash logs | `kubectl logs <pod-name>` |
| Get EC2 public IP | `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` |
| Apply everything in folder | `kubectl apply -f ~/shopstack/infra/k8s/` |

---

**Next:** `session-05-state-secrets.md` — PVC, Secrets, ConfigMaps. Products load in browser.
