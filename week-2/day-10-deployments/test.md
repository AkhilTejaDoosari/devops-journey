[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md) · [7. Interview](../day-7-interview/readme.md) · [8. K8s Architecture](../day-8-k8s-architecture/readme.md) · [9. Pods](../day-9-pods/readme.md) · **10. Deployments**

---

# Day 10 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Open the toggles only after you have answered.

---

## Round 1 — Concepts

**Q1. You delete a Pod that belongs to a Deployment. What happens next and who makes it happen? Trace the full chain.**

<details>
<summary>Answer</summary>

The ReplicaSet controller inside the Controller Manager notices the Pod count dropped below the desired number. It reports the gap to the API Server. The Scheduler picks the best node. The Kubelet on that node spins up a new container. The Pod comes back automatically — nobody woke up, nobody typed a command.

The chain: Controller Manager (ReplicaSet controller) → API Server → Scheduler → Kubelet → new Pod running.

</details>

---

**Q2. What is the difference between a bare Pod and a Deployment? Give a real scenario where the difference matters.**

<details>
<summary>Answer</summary>

A bare Pod has no guardian — if it crashes it stays dead forever until someone manually recreates it. A Deployment wraps the Pod in a ReplicaSet that enforces the desired count 24/7.

Real scenario: The ShopStack API crashes at 3 AM due to a memory spike. With a bare Pod — downtime until morning. With a Deployment — the ReplicaSet creates a new Pod in seconds. Nobody gets paged.

</details>

---

**Q3. What is a ReplicaSet? Do you ever create one directly?**

<details>
<summary>Answer</summary>

A ReplicaSet has one job — ensure a specific number of identical Pod replicas are running at all times. It watches constantly. If the count drops it creates. If the count is too high it terminates.

You almost never create a ReplicaSet directly. You create a Deployment, which creates and manages the ReplicaSet for you. The RS is the mechanism. The Deployment is the manager you interact with.

</details>

---

**Q4. What does `selector.matchLabels` do in a Deployment manifest? What happens if it does not match the Pod template labels?**

<details>
<summary>Answer</summary>

`selector.matchLabels` is how the Deployment finds and claims its Pods. Kubernetes uses these labels to know which Pods belong to which Deployment.

If it does not match the Pod template labels — the Deployment creates Pods it cannot manage. No self-healing. No rolling updates. Nothing works. The Pods exist but the Deployment is blind to them.

One typo here is one of the most common silent failures in Kubernetes.

</details>

---

**Q5. Today the node had a `disk-pressure` taint. What does a taint do and what caused it?**

<details>
<summary>Answer</summary>

A taint tells the Kubernetes Scheduler — do not place any new Pods on this node. Pods without a matching toleration cannot be scheduled there.

The `disk-pressure` taint was added automatically by Kubernetes when the node's disk hit 97% full. Even after disk space was recovered, the taint remained until it was manually removed. This is Kubernetes protecting the node from further evictions.

</details>

---

**Q6. After a rolling update you run `kubectl get rs` and see two ReplicaSets — one at 2, one at 0. Why does the old one still exist?**

<details>
<summary>Answer</summary>

The old ReplicaSet is kept as your rollback parachute. If the new version has a bug, `kubectl rollout undo` scales the old RS back up to 2 and scales the new RS down to 0 — instantly, no new objects created.

If Kubernetes deleted the old RS after the update, rollback would be impossible or much slower.

</details>

---

**Q7. What is the difference between `apiVersion: v1` and `apiVersion: apps/v1`? When do you use each?**

<details>
<summary>Answer</summary>

`v1` is the core API group — used for Pods, Services, ConfigMaps, Secrets, PVCs.

`apps/v1` is the apps API group — used for Deployments, ReplicaSets, StatefulSets, DaemonSets.

Deployments use `apps/v1`. If you write `v1` for a Deployment the API Server rejects it immediately. This is the most common first mistake when writing Deployment manifests.

</details>

---

## Round 2 — Commands

**Q8. Apply all manifests in the `infra/k8s/` folder at once.**

<details>
<summary>Answer</summary>

```bash
kubectl apply -f infra/k8s/
```

</details>

---

**Q9. Check all Deployments — read READY, UP-TO-DATE, AVAILABLE.**

<details>
<summary>Answer</summary>

```bash
kubectl get deployments
```

</details>

---

**Q10. See the ReplicaSets behind your Deployments.**

<details>
<summary>Answer</summary>

```bash
kubectl get rs
```

</details>

---

**Q11. Watch Pods change live — you just deleted one and want to see it come back.**

<details>
<summary>Answer</summary>

```bash
kubectl get pods -w
```

Ctrl+C to stop.

</details>

---

**Q12. The API Deployment has no Pods appearing after apply. What command tells you why?**

<details>
<summary>Answer</summary>

```bash
kubectl describe deployment shopstack-api
```

Read the Events section at the bottom.

</details>

---

**Q13. Trigger a rolling update — change the API image from `1.0` to `1.1` without editing YAML.**

<details>
<summary>Answer</summary>

```bash
kubectl set image deployment/shopstack-api api=akhiltejadoosari/shopstack-api:1.1
```

Syntax: `deployment/<name>` then `<container-name>=<image>:<tag>`. The container name must match what is in the manifest.

</details>

---

**Q14. Watch the rolling update happen in real time.**

<details>
<summary>Answer</summary>

```bash
kubectl rollout status deployment/shopstack-api
```

</details>

---

**Q15. See all previous revisions of the API Deployment.**

<details>
<summary>Answer</summary>

```bash
kubectl rollout history deployment/shopstack-api
```

</details>

---

**Q16. Emergency rollback — something is wrong with the new version.**

<details>
<summary>Answer</summary>

```bash
kubectl rollout undo deployment/shopstack-api
```

</details>

---

**Q17. Scale the API Deployment to 4 replicas.**

<details>
<summary>Answer</summary>

```bash
kubectl scale deployment/shopstack-api --replicas=4
```

</details>

---

> 🔧 Environment Setup — Not a DevOps Skill
> If you need to fix kubectl permissions on a fresh EC2 session:
> ```bash
> mkdir -p ~/.kube
> sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
> sudo chown $(whoami):$(whoami) ~/.kube/config
> export KUBECONFIG=~/.kube/config
> echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
> source ~/.bashrc
> ```
> ↩️ Back to DevOps

---

## Round 3 — Scenario Debug

**Scenario A**

You run `kubectl apply -f infra/k8s/` and all 4 Deployments are created. But `kubectl get pods` shows no Pods at all — not even Pending. What is the most likely cause and what command tells you?

<details>
<summary>Answer</summary>

Most likely cause: `selector.matchLabels` does not match the Pod template labels. The Deployment was created but it cannot find or own any Pods.

Command:
```bash
kubectl describe deployment shopstack-api
```

Read the Events section — it will show a selector mismatch error.

</details>

---

**Scenario B**

You run `kubectl get pods -w` and see all Pods stuck in `Pending` for 3 minutes. No errors visible. What do you check and what are the two most likely causes?

<details>
<summary>Answer</summary>

Command:
```bash
kubectl describe pod <name>
```

Read Events at the bottom.

Two most likely causes on a t3.micro:
1. `Insufficient cpu` or `Insufficient memory` — node is out of resources
2. Node has a taint — `0/1 nodes are available: 1 node(s) had untolerated taint(s)` — check with `kubectl describe node <name> | grep Taint`

</details>

---

**Scenario C**

You trigger a rolling update on the API. `kubectl rollout status` hangs at "1 out of 2 new replicas have been updated." The old Pods are still running. What is happening and what do you do?

<details>
<summary>Answer</summary>

The new Pod is failing to start — bad image tag, wrong env var, or the container crashes immediately. The rolling update is blocked because Kubernetes will not terminate old Pods until new ones are healthy.

The old Pods keep serving traffic — no downtime.

Debug steps:
```bash
kubectl get pods          # find the new Pod — look for ImagePullBackOff or CrashLoopBackOff
kubectl describe pod <new-pod-name>   # read Events — find the exact error
kubectl logs <new-pod-name>           # read what the app printed
```

Fix the issue, then `kubectl apply` the corrected manifest. Or rollback:
```bash
kubectl rollout undo deployment/shopstack-api
```

</details>

---

**Scenario D**

This is exactly what happened today. Your Pods were evicted and then stuck in Pending after you cleared disk space. Walk through what caused it and how you fixed it.

<details>
<summary>Answer</summary>

Cause: Disk hit 97% full. Kubernetes automatically evicted Pods to protect the node and added a `disk-pressure:NoSchedule` taint. After clearing disk space with `docker system prune -a`, the taint remained — blocking new Pod scheduling.

Fix:
```bash
# Confirm the taint
kubectl describe node <name> | grep Taint

# Remove the taint — the - at the end means remove
kubectl taint node <name> node.kubernetes.io/disk-pressure:NoSchedule-

# Pods scheduled immediately after taint removed
kubectl get pods -w
```

</details>

---

## Round 4 — Reading Output

**Q18. You run `kubectl get deployments` and see:**

```
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
shopstack-api        1/2     2            1           5m
```

What does this tell you?

<details>
<summary>Answer</summary>

- `READY 1/2` — 2 Pods desired, only 1 is running and healthy
- `UP-TO-DATE 2` — both Pods are on the latest template version
- `AVAILABLE 1` — only 1 Pod is passing health checks and ready for traffic

One Pod is either still starting, crashing, or being evicted. Run `kubectl get pods` to find which one and `kubectl describe pod <name>` to find why.

</details>

---

**Q19. You run `kubectl get rs` after a rolling update and see:**

```
NAME                       DESIRED   CURRENT   READY   AGE
shopstack-db-8bb56d5d5     1         1         1       2m
shopstack-db-84bbd5455     0         0         0       15m
```

What does this tell you?

<details>
<summary>Answer</summary>

The rolling update completed successfully. The new ReplicaSet `8bb56d5d5` is active with 1 Pod running. The old ReplicaSet `84bbd5455` is scaled to 0 but kept alive as the rollback parachute.

If you run `kubectl rollout undo`, the old RS scales back to 1 and the new RS scales to 0 — instant rollback.

</details>

---

**Q20. You run `kubectl get pods` and see:**

```
NAME                             READY   STATUS             RESTARTS   AGE
shopstack-api-d4b4c45d9-gx2hx   0/1     CrashLoopBackOff   6          11m
```

What does `CrashLoopBackOff` mean and what are your first two commands?

<details>
<summary>Answer</summary>

`CrashLoopBackOff` means the container starts, crashes, Kubernetes tries to restart it, it crashes again. The backoff means Kubernetes is increasing the wait time between restart attempts to avoid hammering a broken system. Restart count 6 means it has crashed and restarted 6 times.

First two commands:
```bash
kubectl logs shopstack-api-d4b4c45d9-gx2hx --previous   # what did it print before the last crash?
kubectl describe pod shopstack-api-d4b4c45d9-gx2hx       # read Events — what did the cluster observe?
```

</details>
