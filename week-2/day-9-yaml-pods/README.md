# Day 9 — YAML & Pods

**[1. Architecture](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/05.%20Kubernetes%20%E2%80%93%20Orchestration/01-architecture/README.md) · [2. YAML & Pods](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/05.%20Kubernetes%20%E2%80%93%20Orchestration/02-yaml-pods/README.md) · [3. Deployments](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/05.%20Kubernetes%20%E2%80%93%20Orchestration/03-deployments/README.md)**

---

## 🧠 The Mental Model First — Before Any YAML

### The analogy that makes everything click

Think about a **job posting** at a company.

The job posting doesn't *do* the work. It *describes* what the company wants:
- Job title
- Skills required
- Which team they join
- What tools they'll use

You hand it to HR. HR reads it, finds the right person, places them.

**That job posting is a Kubernetes manifest.**
**HR is Kubernetes.**
**The hired person is the running Pod.**

You never *command* Kubernetes — "start this container NOW." You *declare* — "here is what I want to exist." Kubernetes reads it, compares it to reality, and makes it happen.

This is called **declarative management** — and it's the entire philosophy of Kubernetes.

---

### The second thing to lock in: what a Pod actually IS

In Docker you ran containers directly. In Kubernetes, **you never run a naked container.**

Kubernetes always wraps your container in a **Pod** first.

Think of a Pod like a **shipping container on a cargo ship:**
- The shipping container (Pod) has its own address, its own identity
- Inside it sits your actual cargo (the container — your app)
- The ship (the node/EC2) carries many containers at once
- When the shipping container falls off the ship — it's gone. The cargo inside goes with it. A new identical one gets placed.

That last part is critical: **when a Pod dies, its IP address dies with it.** A brand new Pod gets a brand new IP. This is why you never hardcode Pod IPs — but that's Day 11's problem. Today you just need to feel what a Pod is.

---

### Declarative vs Imperative — side by side

| Imperative (Docker) | Declarative (Kubernetes) |
|---|---|
| `docker run -p 8080:8080 shopstack-api` | Write `api-pod.yaml`, then `kubectl apply -f api-pod.yaml` |
| You describe the *action* | You describe the *desired state* |
| Nothing watches it after it starts | Kubernetes watches it forever |
| Container crashes = it stays dead | Pod crashes = gap detected → fixed |

---

## 📍 Where Everything Lives Today

Before you write a single character, map the file system:

```
shopstack/
├── infra/
│   └── k8s/                  ← YOU ARE BUILDING THIS TODAY
│       ├── api-pod.yaml      ← Pod for the API service
│       ├── frontend-pod.yaml ← Pod for the frontend service
│       └── db-pod.yaml       ← Pod for the database
├── services/
│   └── api/                  ← where the actual app code lives
│       └── Dockerfile        ← what your image was built from
└── docs/

devops-journey/
└── week-2/
    └── day-9-yaml-pods/
        ├── readme.md         ← this file
        └── test.md           ← after session
```

**Why you're reading a manifest:** You are reading it to understand how Kubernetes knows *what to run, what to call it, and how to find it.* Every Pod manifest answers three questions: **Who are you? What do you run? How do others find you?**

---

## 📖 The Four Pillars — Every Manifest Has These

Every single Kubernetes object — Pod, Deployment, Service, Secret — starts with the same skeleton. Miss one pillar and the API Server rejects the file before reading anything else.

```yaml
apiVersion: v1        # Which version of the K8s API handles this object
kind: Pod             # What TYPE of object — one word changes everything
metadata:             # Identity — name, labels
  name: shopstack-api
  labels:
    app: shopstack
    tier: api
spec:                 # The blueprint — what actually runs
  containers:
    - name: api
      image: akhiltejadoosari/shopstack-api:1.0
```

### Breaking down each pillar:

**`apiVersion`** — Kubernetes has different API groups for different objects.
- Core objects (Pod, Service, ConfigMap): `v1`
- App objects (Deployment): `apps/v1`
- Wrong version = instant rejection. No error message tells you why clearly.

**`kind`** — Case sensitive. `pod` ≠ `Pod`. One word triggers an entire different controller inside Kubernetes.

**`metadata.name`** — The Pod's identity in the cluster. Must be unique. When a Pod dies and a new one is created, the new one gets a new generated name.

**`metadata.labels`** — The **badge system**. Labels are how Kubernetes objects find each other. A Service finds Pods by matching labels, not by name. This is what connects your manifest to everything else.

**`spec`** — Everything inside here is the blueprint. What image to run, what ports to open, what environment variables to inject.

---

## 📋 Full Cheatsheet

### YAML Syntax Rules [CORE]

| Rule | What it means | What breaks if ignored |
|---|---|---|
| 2-space indent | Nesting is depth — always 2 spaces, never tabs | `yaml: mapping values are not allowed here` |
| `:` must have a space after it | `key: value` not `key:value` | Silent parse failure |
| `-` means list item | `containers:` takes a list — each item starts with `- ` | Wrong nesting, object not created |
| Strings with special chars need quotes | `value: "db"` — safe. `value: db` — usually fine too | Rare edge cases with `:` or `#` in values |

---

### Pod Manifest Anatomy [CORE]

| Field | What it does | ShopStack example |
|---|---|---|
| `apiVersion: v1` | Tells API Server which object group handles this | All Pods use `v1` |
| `kind: Pod` | The object type — triggers the Pod controller | `Pod` — capital P always |
| `metadata.name` | Unique name in the cluster | `shopstack-api` |
| `metadata.labels` | Badges — how Services and Deployments find this Pod | `app: shopstack`, `tier: api` |
| `spec.containers[].name` | Name of the container inside the Pod | `api` |
| `spec.containers[].image` | Docker image to pull and run | `akhiltejadoosari/shopstack-api:1.0` |
| `spec.containers[].ports[].containerPort` | Documents what port the container listens on — does NOT open or block anything | `8080` for api, `80` for frontend |
| `spec.containers[].env[]` | Environment variables injected at runtime | `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` |

---

### Core kubectl Commands [CORE]

| Command | What it does | ShopStack example |
|---|---|---|
| `kubectl apply -f <file>` | Declare desired state — create or update the object | `kubectl apply -f infra/k8s/api-pod.yaml` |
| `kubectl get pods` | List all Pods and their status | `kubectl get pods` |
| `kubectl get pods -o wide` | Same + node IP and which node it's on | `kubectl get pods -o wide` |
| `kubectl get pods -w` | Watch Pod changes live as they happen | `kubectl get pods -w` |
| `kubectl describe pod <name>` | Full Pod profile — events, probes, env vars, errors | `kubectl describe pod shopstack-api` |
| `kubectl logs <name>` | Container stdout — what the app printed | `kubectl logs shopstack-api` |
| `kubectl logs -f <name>` | Follow logs live | `kubectl logs -f shopstack-api` |
| `kubectl delete pod <name>` | Delete the Pod — no controller means it stays dead | `kubectl delete pod shopstack-api` |
| `kubectl exec -it <name> -- /bin/sh` | Shell into a running container | `kubectl exec -it shopstack-api -- /bin/sh` |

---

### Reading `kubectl get pods` Output [CORE]

```
NAME             READY   STATUS    RESTARTS   AGE
shopstack-api    1/1     Running   0          2m
shopstack-db     0/1     Pending   0          10s
```

| Column | What it means |
|---|---|
| `NAME` | Pod name — auto-generated suffix if created by Deployment |
| `READY` | `1/1` = all containers in this Pod are ready. `0/1` = container exists but not ready |
| `STATUS` | `Running`, `Pending`, `CrashLoopBackOff`, `Error`, `Completed` |
| `RESTARTS` | How many times the container has crashed and been restarted |
| `AGE` | How long this Pod has existed |

---

### Status Meanings [CORE]

| Status | What it means | First command to run |
|---|---|---|
| `Running` | Container is up and passing health checks | Nothing — it's healthy |
| `Pending` | Kubernetes accepted it but hasn't placed it on a node yet | `kubectl describe pod <name>` → read Events |
| `CrashLoopBackOff` | Container starts, crashes, K8s retries, crash again — loop | `kubectl logs <name> --previous` |
| `Error` | Container exited with non-zero code | `kubectl logs <name>` |
| `ContainerCreating` | Image is being pulled | Wait, then check again |

---

### [WIKI] — Lower Frequency, Document Only

| Command | What it does |
|---|---|
| `kubectl get pods -l tier=api` | Filter Pods by label |
| `kubectl logs <name> --tail=50` | Last 50 lines only |
| `kubectl logs <name> --previous` | Logs from the crashed container before restart |
| `kubectl exec <name> -- env` | Print all env vars in container without shelling in |
| `kubectl get pods -n kube-system` | See Kubernetes' own internal Pods |
| `kubectl get pods -A` | Every Pod in every namespace — full cluster scan |

---

## ⚠️ What Breaks Today

| Symptom | Cause | Fix |
|---|---|---|
| `error: no kind "pod" is registered` | `kind: pod` instead of `kind: Pod` | Capitalise |
| `mapping values are not allowed here` | Tab instead of spaces in YAML | Replace all tabs with 2 spaces |
| Pod stuck in `Pending` forever | k3s node doesn't have enough resources, or image name wrong | `kubectl describe pod <name>` → Events section |
| Pod stuck in `CrashLoopBackOff` | Container crashes on start — env var wrong, missing DB connection | `kubectl logs shopstack-api --previous` |
| Deleted Pod, it didn't come back | Correct — bare Pods don't self-heal | This is the lesson. Deployments fix this on Day 10. |
| `ImagePullBackOff` | Image name or tag wrong — Docker Hub can't find it | Check image name matches exactly what's on Docker Hub |

---

## ✅ Day 9 Checklist

```
[ ] mkdir -p ~/shopstack/infra/k8s
[ ] Write infra/k8s/api-pod.yaml from scratch
[ ] Write infra/k8s/frontend-pod.yaml from scratch
[ ] Write infra/k8s/db-pod.yaml from scratch
[ ] kubectl apply -f infra/k8s/api-pod.yaml
[ ] kubectl apply -f infra/k8s/frontend-pod.yaml
[ ] kubectl apply -f infra/k8s/db-pod.yaml
[ ] kubectl get pods — read every column out loud
[ ] kubectl describe pod shopstack-api — read Events section
[ ] kubectl logs shopstack-api — read startup output
[ ] kubectl delete pod shopstack-api
[ ] kubectl get pods — confirm it is gone and does NOT come back
[ ] Answer out loud: what is a Pod and why is it not enough alone?
```

---

**Ready to test yourself? → [Test](./test.md)**
