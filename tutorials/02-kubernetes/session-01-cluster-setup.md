# Session 01 — Kubernetes Cluster Setup

**Goal:** Install k3s on EC2, connect kubectl from your Mac to the cluster, verify the cluster is healthy and ready for ShopStack.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Kubernetes Exists

Docker Compose runs ShopStack on one machine. That works until:
- The machine goes down → everything is down
- Traffic spikes → you need more API containers → manual work on every restart
- A container crashes at 3 AM → nobody restarts it until morning

Kubernetes solves all three: it runs containers across multiple machines, scales them automatically, and restarts crashed containers without human intervention.

**One line:** Docker runs containers. Kubernetes manages them at scale so you don't have to.

---

## Visual Map

```
Your Mac (terminal)
    │
    │  kubectl command
    │  ~/.kube/config → points to EC2:6443
    ▼
EC2 (c7i-flex.large — Ubuntu 26.04)
    │
    ├── k3s control plane  ← the brain
    │   ├── API Server     ← single entry point for all kubectl commands
    │   ├── etcd           ← cluster database — stores desired state
    │   ├── Scheduler      ← decides which node runs which Pod
    │   └── Controller     ← watchdog — closes gap between desired and actual
    │
    └── k3s worker         ← same machine — where Pods actually run
        ├── kubelet        ← receives orders from API Server
        └── containerd     ← pulls images and runs containers
```

k3s = lightweight Kubernetes. Control plane + worker on the same EC2 instance. Same API as EKS — only the infrastructure underneath differs.

---

## How kubectl Talks to k3s

```
kubectl get pods
    │
    ▼
~/.kube/config          ← tells kubectl where the cluster is
    │                      server: https://YOUR_EC2_IP:6443
    ▼
k3s API Server on EC2   ← receives the request
    │
    ▼
Returns pod list        ← kubectl prints it
```

The kubeconfig file is the key. It contains:
- Where the cluster is (`server:`)
- How to authenticate (`certificate-authority-data:`, `client-certificate-data:`)

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Install k3s on EC2
> Not a DevOps skill. In a real company the cluster already exists. You just get a kubeconfig.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

### Step 1 — Install k3s (one command)

```bash
# On EC2
curl -sfL https://get.k3s.io | sh -
```

Takes ~60 seconds. k3s downloads, installs, and starts as a systemd service automatically.

---

### Step 2 — Verify cluster is running

```bash
# On EC2
sudo k3s kubectl get nodes
```

Expected:
```
NAME               STATUS   ROLES                  AGE   VERSION
ip-172-31-xx-xx   Ready    control-plane,master   30s   v1.x.x+k3s1
```

`Ready` = cluster is healthy. If `NotReady` — wait 30 seconds and run again.

---

## 🚀 The Work Begins

### Step 3 — Wire kubectl on EC2 to k3s

k3s creates its own kubeconfig at `/etc/rancher/k3s/k3s.yaml`. Make it accessible to the ubuntu user:

```bash
# On EC2
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config

# Verify kubectl works without sudo
kubectl get nodes
```

Expected: same node shown as `Ready`

---

### Step 4 — Verify cluster internals

```bash
# See all system pods — k3s components
kubectl get pods -n kube-system
```

Expected — all pods Running or Completed:
```
NAME                                     READY   STATUS
coredns-xxx                              1/1     Running   ← internal DNS
local-path-provisioner-xxx               1/1     Running   ← PVC storage
metrics-server-xxx                       1/1     Running
svclb-traefik-xxx                        1/1     Running
traefik-xxx                              1/1     Running   ← ingress controller
```

`coredns` is critical — it handles all Kubernetes DNS (`DB_HOST=db` resolves via CoreDNS).
`local-path-provisioner` is critical — it handles PVCs for Postgres storage.

---

### Step 5 — Get your EC2 IP

```bash
# On EC2
curl -s http://checkip.amazonaws.com
```

> ⚠️ EC2 public IP changes on every restart. When it changes:
> 1. Get new IP with this command
> 2. Update kubeconfig: `sed -i 's/OLD_IP/NEW_IP/g' ~/.kube/config`
> 3. Verify: `kubectl get nodes`

---

### Step 6 — Understand the daily opening sequence

Run this at the start of every Kubernetes session before touching any manifests:

```bash
# Step 1 — is cluster responding?
kubectl get nodes

# Step 2 — are system pods healthy?
kubectl get pods -n kube-system

# Step 3 — what's deployed in default namespace?
kubectl get pods
```

| What you see | Meaning | Fix |
|---|---|---|
| Node `Ready` | Cluster healthy | Continue |
| `connection refused` | EC2 rebooted — IP changed | Update kubeconfig server IP |
| kube-system pod in `CrashLoopBackOff` | Cluster unhealthy | `sudo systemctl restart k3s` |

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. k3s is running
sudo systemctl status k3s | grep Active
```
Expected: `Active: active (running)`

---

```bash
# 2. kubectl works without sudo
kubectl get nodes
```
Expected: node shows `Ready`

---

```bash
# 3. All system pods healthy
kubectl get pods -n kube-system | grep -v "Running\|Completed" | grep -v NAME
```
Expected: no output — all pods are Running or Completed

---

```bash
# 4. CoreDNS is running (critical for K8s DNS)
kubectl get pods -n kube-system | grep coredns
```
Expected: `coredns-xxx   1/1   Running`

---

```bash
# 5. Local path provisioner running (critical for PVCs)
kubectl get pods -n kube-system | grep local-path
```
Expected: `local-path-provisioner-xxx   1/1   Running`

---

```bash
# 6. kubectl version matches cluster
kubectl version --short 2>/dev/null || kubectl version
```
Expected: client and server versions both shown

---

**All 6 green?** → Save file → push → Session 02 — Pods.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl: command not found` | k3s installed but kubectl not linked | Use `sudo k3s kubectl` or copy kubeconfig |
| Node shows `NotReady` | k3s still starting | Wait 30s and retry |
| `connection refused` on port 6443 | EC2 IP changed or SG missing port 6443 | Update kubeconfig IP, check security group |
| kube-system pods crashing | k3s install issue | `sudo systemctl restart k3s` |
| `permission denied` on kubeconfig | Wrong file ownership | `sudo chown ubuntu:ubuntu ~/.kube/config` |

---

## Quick Reference

| What | Command |
|---|---|
| Install k3s | `curl -sfL https://get.k3s.io \| sh -` |
| k3s status | `sudo systemctl status k3s` |
| Restart k3s | `sudo systemctl restart k3s` |
| Get nodes | `kubectl get nodes` |
| All system pods | `kubectl get pods -n kube-system` |
| All pods all namespaces | `kubectl get pods -A` |
| EC2 IP | `curl -s http://checkip.amazonaws.com` |
| Update kubeconfig IP | `sed -i 's/OLD_IP/NEW_IP/g' ~/.kube/config` |
