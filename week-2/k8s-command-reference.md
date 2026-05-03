# Kubernetes Week 2 — Full Command Reference
### Days 8 through 14 — ShopStack Anchored

---

## How to use this file

Every command is labeled one of two ways:

**🧠 BRAIN** — Type it blind. No looking. If you hesitate on any of these in the final test, go back and drill it.

**📖 LOOKUP** — Know *when* to reach for it and *why* it exists. Exact flags → come back here.

---

## The 4 Verbs That Run Everything

```bash
kubectl apply -f <file>          # create or update any object — Pod, Deployment, Service, Secret, PVC
kubectl get <noun>               # list — pods, deployments, rs, services, pvc, secrets, configmaps
kubectl describe <noun> <name>   # full detail + Events — use this when something is wrong
kubectl delete <noun> <name>     # remove it — Deployment deletion cascades to RS and Pods
```

---

## The Universal Debug Sequence — Always In This Order

```bash
kubectl get pods                       # step 1 — what is the status? is restart count climbing?
kubectl describe pod <name>            # step 2 — what did the cluster observe? read Events section
kubectl logs <name>                    # step 3 — what did the app print? connection errors live here
kubectl logs <name> --previous         # step 4 — what did it print before it crashed? for CrashLoopBackOff
kubectl exec -it <name> -- /bin/sh     # step 5 — go inside, test DNS, check env vars, curl other services
```

Stop when you find the answer. Most problems resolve at step 2 or 3.

---

---

# DAY 8 — Cluster Setup

**The soul of Day 8:** Prove that kubectl on your Mac talks to k3s on EC2. Node must be Ready before you touch a manifest.

---

### 🧠 BRAIN

```bash
kubectl get nodes                      # first command every session — node must show Ready
kubectl get pods -A                    # full cluster scan — all namespaces — kube-system must be Running
kubectl get pods                       # your ShopStack pods in default namespace only
```

---

### 📖 LOOKUP

```bash
kubectl cluster-info                   # confirm API Server address — use after EC2 restart
kubectl get pods -n kube-system        # check control plane components specifically — CoreDNS, provisioner
kubectl describe node <name>           # when Pods stuck in Pending — check CPU and RAM capacity on the node

# EC2 IP changed after restart — fix kubeconfig on your Mac
sed -i '' 's/OLD_IP/NEW_IP/g' ~/.kube/config
kubectl get nodes                      # verify connection restored
```

---

### Pod Status Decoder

| Status | Meaning | First Move |
|---|---|---|
| `Running` | Alive and healthy | Check logs if app misbehaves |
| `Pending` | Scheduler can't place it | `kubectl describe pod` → Events → `Insufficient cpu` |
| `CrashLoopBackOff` | Keeps crashing, K8s keeps retrying | `kubectl logs --previous` |
| `ImagePullBackOff` | Can't pull the image | `kubectl describe pod` → Events → image name or tag wrong |
| `OOMKilled` | Container exceeded memory limit | `kubectl describe pod` → Events |
| `ContainerCreating` | Normal during startup | Wait 30s, then check if stuck |
| `Terminating` | Being deleted | Normal — just wait |

---

---

# DAY 9 — Pods

**The soul of Day 9:** Write Pod manifests. Apply them. Delete one. Watch it stay dead. That feeling is why Deployments exist.

---

### 🧠 BRAIN

```bash
kubectl apply -f infra/k8s/api-pod.yaml        # send manifest to API Server — creates if new, updates if exists
kubectl apply -f infra/k8s/                    # apply every manifest in the folder at once
kubectl get pods                               # status, restart count, age — read every column
kubectl describe pod shopstack-api             # read Events at the bottom — this is where errors live
kubectl logs shopstack-api                     # what the app printed to stdout — application errors live here
kubectl logs shopstack-api --previous          # logs from the LAST crash — use this for CrashLoopBackOff
kubectl delete pod shopstack-api               # bare Pod: gone forever — Deployment-owned Pod: comes back
```

---

### 📖 LOOKUP

```bash
kubectl get pods -o wide                       # when you need Pod IPs and which node they landed on
kubectl get pods -l tier=api                   # when filtering — show only pods with this label
kubectl get pods -w                            # watch Pod changes live — Ctrl+C to stop
kubectl logs -f shopstack-api                  # follow logs live as the app runs — Ctrl+C to stop
kubectl logs shopstack-api --tail=50           # last 50 lines only — when history is too long to read
kubectl exec -it shopstack-api -- /bin/sh      # enter the running container — when logs aren't enough
kubectl exec shopstack-api -- env              # check env vars container actually sees — no need to fully enter
kubectl delete -f infra/k8s/api-pod.yaml       # delete by file instead of by name — uses the manifest definition
```

---

---

# DAY 10 — Deployments

**The soul of Day 10:** A Deployment is a guardian. Delete a Pod — it comes back. Change an image — rolls out safely. Something breaks — one command undoes it.

---

### 🧠 BRAIN

```bash
kubectl get deployments                                # READY / UP-TO-DATE / AVAILABLE — read every column
kubectl get rs                                         # ReplicaSets behind your Deployments — two RS after a rolling update
kubectl get pods -w                                    # watch Pods appear and disappear live during update or scale
kubectl rollout status deployment/shopstack-api        # watch rolling update in real time — blocks until done or stuck
kubectl rollout undo deployment/shopstack-api          # emergency rollback to previous version — one command, instant
kubectl describe deployment shopstack-api              # when no Pods appear after apply — read Events section
```

---

### 📖 LOOKUP

```bash
kubectl rollout history deployment/shopstack-api                   # see all revisions — check before rolling back
kubectl rollout undo deployment/shopstack-api --to-revision=1      # rollback to a specific revision, not just previous
kubectl set image deployment/shopstack-api api=akhiltejadoosari/shopstack-api:1.1
                                                                   # trigger rolling update without editing YAML
                                                                   # syntax: deployment/<name> <container>=<image>:<tag>
kubectl scale deployment/shopstack-api --replicas=4                # manually scale up to handle load, or down to save
kubectl delete deployment shopstack-api                            # removes Deployment, RS, and all Pods — careful
```

---

### Two Rules That Burn People Every Time

```bash
# RULE 1 — selector must match template labels exactly
# selector.matchLabels  ←→  template.metadata.labels
# One typo = Deployment creates Pods it cannot manage — no self-healing, no rolling updates

# RULE 2 — apiVersion is different for Pods vs Deployments
# Pods use:        apiVersion: v1
# Deployments use: apiVersion: apps/v1
# Wrong version = API Server rejects immediately
```

---

---

# DAY 11 — Services & Networking

**The soul of Day 11:** Pod IPs die on every restart. A Service has a stable DNS name that never changes. `DB_HOST=db` works because K8s DNS resolves `db` to the db Service — not to a Pod IP directly.

---

### 🧠 BRAIN

```bash
kubectl get services                           # list all Services — TYPE, CLUSTER-IP, PORT(S)
kubectl get endpoints                          # which Pod IPs each Service is routing to right now
                                               # if <none> — selector doesn't match any Pod labels
kubectl describe service api                   # full Service detail — check Selector and Endpoints fields
                                               # use when: Service exists but traffic isn't reaching Pods
```

---

### 📖 LOOKUP

```bash
kubectl get services -o wide                           # see selector alongside service details
kubectl get endpoints api                              # check one specific Service's routing
kubectl delete service api                             # remove a Service — Pods keep running, just unreachable
kubectl exec -it shopstack-api-xxx -- nslookup db      # from inside cluster — does 'db' resolve to a Service IP?
kubectl exec -it shopstack-api-xxx -- wget -qO- http://api:8080/api/health
                                                       # test HTTP connectivity between services from inside a Pod
```

---

### Service Types — Know Which Is Which

```bash
# ClusterIP    — inside cluster only         — use for: api, db, worker
# NodePort     — browser via EC2 IP + port   — use for: frontend :30080, adminer :30081
# LoadBalancer — public internet via cloud LB — use for: production on EKS Week 5
```

### The 3 Port Fields — One Typo Kills Traffic

```yaml
ports:
  - port: 80          # port OTHER Pods use to reach this Service
    targetPort: 80    # port on the POD the Service actually forwards traffic to
    nodePort: 30080   # NodePort only — port exposed on EC2 — must be 30000-32767
```

### DNS Rule

```bash
# Same namespace:      DB_HOST=db                             ✅ resolves
# Different namespace: DB_HOST=db                             ❌ fails
# Different namespace: DB_HOST=db.default.svc.cluster.local   ✅ resolves
# ShopStack is all in default — short names always work
```

---

---

# DAY 12 — State (ConfigMap, Secret, PVC)

**The soul of Day 12:** Containers are ephemeral — data dies with them. Secrets keep passwords out of plain YAML. ConfigMaps centralize non-sensitive config. PVCs give Postgres a disk that outlives any Pod.

---

### 🧠 BRAIN

```bash
kubectl get pvc                                # STATUS must be Bound — Pending means provisioner not running
kubectl get secrets                            # confirm Secret exists — values hidden by default
kubectl get configmaps                         # confirm ConfigMap exists
echo -n "shopstack_dev" | base64              # encode a value for a Secret manifest
                                               # -n is critical — without it echo adds newline, value is wrong
```

---

### 📖 LOOKUP

```bash
kubectl describe configmap db-config           # verify what keys and values are stored in the ConfigMap
kubectl describe pvc db-pvc                    # when PVC stuck in Pending — read Events for provisioner errors
kubectl get pv                                 # see actual PersistentVolumes provisioned behind your claims
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
                                               # decode a specific secret field to verify the value is correct
kubectl exec -it shopstack-db-xxx -- env       # verify env vars were actually injected into the container
kubectl rollout restart deployment/shopstack-api
                                               # after updating a Secret or ConfigMap — Pods don't auto-reload on changes
```

---

### Three Objects — One Decision Rule

```bash
# Would you commit this to a public GitHub repo?
#   Yes → ConfigMap    (DB_HOST, DB_PORT, DB_NAME — not sensitive)
#   No  → Secret       (DB_PASSWORD — never in plain text)

# Does data need to survive Pod restarts?
#   Yes → PVC          (Postgres data lives at /var/lib/postgresql/data)
```

---

---

# DAY 13 — Troubleshooting

**The soul of Day 13:** You break ShopStack on purpose and fix it using only 5 commands. By the end you have muscle memory for diagnosis — not guessing.

---

### 🧠 BRAIN

```bash
kubectl get pods                               # step 1 — status and restart count — climbing restarts = problem
kubectl describe pod <name>                    # step 2 — Events section — cluster errors live here, not in logs
kubectl logs <name>                            # step 3 — app errors — wrong DB_HOST, connection refused, missing table
kubectl logs <name> --previous                 # step 4 — what it printed before crash — only useful for CrashLoopBackOff
kubectl exec -it <name> -- /bin/sh             # step 5 — enter and test manually when logs alone aren't enough
kubectl rollout restart deployment/shopstack-api
                                               # after fixing a Secret or ConfigMap — forces Pods to reload the change
```

---

### 📖 LOOKUP

```bash
kubectl get events --sort-by=.lastTimestamp    # full cluster timeline — what happened and in what order
kubectl get events -w                          # watch events stream live during a deploy or deliberate break
kubectl describe service api                   # when traffic not reaching Pods — check Endpoints field
                                               # Endpoints: <none> means selector doesn't match Pod labels
kubectl get endpoints                          # quick check — are any Pods actually behind this Service?
```

---

### Break Scenario Decoder

```bash
# CrashLoopBackOff + restart count climbing  → kubectl logs --previous
# ImagePullBackOff                            → kubectl describe pod → Events → image name or tag wrong
# Pending, never starts                       → kubectl describe pod → Events → Insufficient cpu or ram
# Running but browser returns 502             → kubectl get endpoints → is it <none>?
# Running but wrong data returned             → kubectl exec -- env → are the env vars correct?
# Service exists but Pods not in endpoints    → kubectl describe service → compare Selector to Pod labels
```

---

---

# The Master Table — Final Test Prep

Cover the right column. Produce the command from the situation alone. If you can do this cold — you are ready for Day 14.

| Situation | Command |
|---|---|
| Is my cluster alive | `kubectl get nodes` |
| Full health scan, all namespaces | `kubectl get pods -A` |
| Are my ShopStack pods running | `kubectl get pods` |
| Create or update from a file | `kubectl apply -f <file>` |
| Apply everything in a folder | `kubectl apply -f infra/k8s/` |
| Full Pod detail — read Events | `kubectl describe pod <name>` |
| What the app printed | `kubectl logs <name>` |
| What it printed before crash | `kubectl logs <name> --previous` |
| Enter the container | `kubectl exec -it <name> -- /bin/sh` |
| Check env vars inside container | `kubectl exec <name> -- env` |
| Are Deployments healthy | `kubectl get deployments` |
| ReplicaSets behind Deployments | `kubectl get rs` |
| Watch rolling update live | `kubectl rollout status deployment/<name>` |
| Roll back to previous version | `kubectl rollout undo deployment/<name>` |
| See all update revisions | `kubectl rollout history deployment/<name>` |
| Watch Pods change live | `kubectl get pods -w` |
| Are Services routing to Pods | `kubectl get endpoints` |
| Full Service detail | `kubectl describe service <name>` |
| Is PVC bound | `kubectl get pvc` |
| Does Secret exist | `kubectl get secrets` |
| Encode a value for a Secret | `echo -n "value" \| base64` |
| Force Pods to reload config | `kubectl rollout restart deployment/<name>` |
| Full cluster event timeline | `kubectl get events --sort-by=.lastTimestamp` |
