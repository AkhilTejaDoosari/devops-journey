# Day 9 Test — YAML & Pods

> Anchor: ShopStack on k3s EC2. No notes. No hints. All four rounds.

---

## Round 1 — Concepts (30 seconds each, answer out loud)

**Q1. What is a Pod? One sentence.**

<details><summary>Answer</summary>

The smallest deployable unit in Kubernetes — a wrapper around one or more containers that gives them a shared identity, IP address, and network namespace.

</details>

---

**Q2. What is declarative management? How is it different from what you did in Docker?**

<details><summary>Answer</summary>

Declarative = you write what you want to exist. Kubernetes compares it to reality and closes any gap. Docker was imperative — you gave direct commands like `docker run`. In Kubernetes you write a manifest and `kubectl apply` it. Kubernetes figures out the rest.

</details>

---

**Q3. The ShopStack API Pod kept crashing with `CrashLoopBackOff`. What was the root cause you saw in the logs today?**

<details><summary>Answer</summary>

The API tried to connect to host `db` — 12 times — and failed every time with `Name or service not known`. There is no Kubernetes Service named `db` yet. DNS cannot resolve it. The app exhausted its retries and crashed. This is expected — the fix is a Service object, not the Pod.

</details>

---

**Q4. You delete a bare Pod. Does it come back? Why or why not?**

<details><summary>Answer</summary>

No. A bare Pod has no controller watching it. When it dies, Kubernetes detects no gap — there is no desired state declaring it should exist. It stays dead. A Deployment would watch it and recreate it — that is Day 10.

</details>

---

**Q5. What are the four pillars every Kubernetes manifest must have?**

<details><summary>Answer</summary>

1. `apiVersion` — which API group handles this object (`v1` for Pods)
2. `kind` — what type of object (`Pod`, `Deployment`, `Service`)
3. `metadata` — identity: name and labels
4. `spec` — the blueprint: what containers to run, what ports, what env vars

Missing any one = API Server rejects the file immediately.

</details>

---

## Round 2 — Commands (recall cold, no notes)

**Q1. Create the k8s folder inside shopstack/infra/**

<details><summary>Answer</summary>

```bash
mkdir -p ~/shopstack/infra/k8s
```

</details>

---

**Q2. Apply a Pod manifest file.**

<details><summary>Answer</summary>

```bash
kubectl apply -f infra/k8s/api-pod.yaml
```

</details>

---

**Q3. List all running Pods.**

<details><summary>Answer</summary>

```bash
kubectl get pods
```

</details>

---

**Q4. List all Pods and show which node they are on.**

<details><summary>Answer</summary>

```bash
kubectl get pods -o wide
```

</details>

---

**Q5. Watch Pod status changes live.**

<details><summary>Answer</summary>

```bash
kubectl get pods -w
```

</details>

---

**Q6. Get the full event log for the shopstack-api Pod.**

<details><summary>Answer</summary>

```bash
kubectl describe pod shopstack-api
```

</details>

---

**Q7. Read the logs from the shopstack-api container.**

<details><summary>Answer</summary>

```bash
kubectl logs shopstack-api
```

</details>

---

**Q8. Read the logs from the PREVIOUS crash of shopstack-api.**

<details><summary>Answer</summary>

```bash
kubectl logs shopstack-api --previous
```

</details>

---

**Q9. Follow the shopstack-api logs live.**

<details><summary>Answer</summary>

```bash
kubectl logs -f shopstack-api
```

</details>

---

**Q10. Delete the shopstack-api Pod.**

<details><summary>Answer</summary>

```bash
kubectl delete pod shopstack-api
```

</details>

---

**Q11. Shell into the running shopstack-api container.**

<details><summary>Answer</summary>

```bash
kubectl exec -it shopstack-api -- /bin/sh
```

</details>

---

**Q12. Filter Pods by label — show only api tier Pods.**

<details><summary>Answer</summary>

```bash
kubectl get pods -l tier=api
```

</details>

---

> 🔧 Environment Setup — Not a DevOps Skill
> These commands create the lab files needed to run Round 3. Copy and paste.
>
> ```bash
> mkdir -p ~/shopstack/infra/k8s
> cat > ~/shopstack/infra/k8s/api-pod.yaml << 'EOF'
> apiVersion: v1
> kind: Pod
> metadata:
>   name: shopstack-api
>   labels:
>     app: shopstack
>     tier: api
> spec:
>   containers:
>     - name: api
>       image: akhiltejadoosari/shopstack-api:1.0
>       ports:
>         - containerPort: 8080
> EOF
> kubectl apply -f ~/shopstack/infra/k8s/api-pod.yaml
> ```
>
> ↩️ Back to DevOps

---

## Round 3 — Scenario Debug (ShopStack is broken — diagnose and fix)

**Scenario 1.**
You run `kubectl apply -f infra/k8s/api-pod.yaml` and get:
```
error: error validating "api-pod.yaml": unknown field "label"
```
What is wrong and what do you fix?

<details><summary>Answer</summary>

`label` is a typo — the correct field is `labels` (with an `s`). Open the manifest, fix `label:` to `labels:`, and re-apply.

</details>

---

**Scenario 2.**
`kubectl get pods` shows:
```
NAME            READY   STATUS             RESTARTS   AGE
shopstack-api   0/1     CrashLoopBackOff   5          3m
```
What are the first two commands you run?

<details><summary>Answer</summary>

```bash
kubectl logs shopstack-api --previous
kubectl describe pod shopstack-api
```

`--previous` shows what the container printed before the last crash. `describe` shows what Kubernetes observed — exit codes, events, error messages.

</details>

---

**Scenario 3.**
You run `kubectl get pods` after deleting the shopstack-api Pod. Output:
```
No resources found in default namespace.
```
Is this a problem? Explain.

<details><summary>Answer</summary>

No. This is correct. A bare Pod has no controller. When it dies it stays dead — Kubernetes has no record of a desired state asking for it. If you want self-healing, you need a Deployment. That is Day 10.

</details>

---

**Scenario 4.**
You apply the api-pod.yaml but the Pod stays in `Pending` for 2 minutes. What do you run and what do you look for?

<details><summary>Answer</summary>

```bash
kubectl describe pod shopstack-api
```

Scroll to the Events section. Look for `Insufficient cpu` or `Insufficient memory` — the scheduler cannot find a node with enough resources. On a t3.micro this can happen if too many Pods are already running.

</details>

---

**Scenario 5.**
The logs show:
```
db_connect_failed: [Errno -2] Name or service not known
db_connect_exhausted: max_attempts 12
```
What is wrong? What is the fix — and is today the day to fix it?

<details><summary>Answer</summary>

The API cannot resolve the hostname `db`. There is no Kubernetes Service named `db` — DNS has nothing to resolve. This is expected on Day 9. The fix is creating a `db` Service object — that is Day 11 (Networking). Today is not the day to fix it.

</details>

---

## Round 4 — Reading Output (tell me what it means)

**Output 1.**
```
NAME            READY   STATUS    RESTARTS   AGE
shopstack-api   1/1     Running   1          4m
```
Read every column. What does `1 restart` tell you?

<details><summary>Answer</summary>

- `NAME` — the Pod is named `shopstack-api`
- `READY` — `1/1` — 1 container defined, 1 running and healthy
- `STATUS` — `Running` — Pod is alive
- `RESTARTS` — `1` — the container crashed once and Kubernetes restarted it automatically
- `AGE` — Pod has existed for 4 minutes

The restart is not necessarily a problem — it is common on first start when a dependency like the database is not yet available. Watch it — if restarts keep climbing, run `kubectl logs --previous`.

</details>

---

**Output 2.**
```
Events:
  Normal   Scheduled  5s    default-scheduler  Successfully assigned default/shopstack-api to ip-172-31-33-164
  Normal   Pulling    4s    kubelet            Pulling image "akhiltejadoosari/shopstack-api:1.0"
  Normal   Pulled     2s    kubelet            Successfully pulled image in 4.5s
  Normal   Created    2s    kubelet            Created container api
  Normal   Started    1s    kubelet            Started container api
  Warning  BackOff    0s    kubelet            Back-off restarting failed container api
```
Tell me the story — what happened step by step?

<details><summary>Answer</summary>

1. Scheduler assigned the Pod to the EC2 node
2. kubelet pulled the image from Docker Hub — took 4.5 seconds
3. kubelet created the container inside the Pod
4. kubelet started the container
5. The container crashed — kubelet is now backing off before retrying

The `Warning BackOff` at the end means the container started, then immediately exited with an error. Next step: `kubectl logs shopstack-api --previous` to read what the app printed before dying.

</details>

---

**Output 3.**
```
NAME            READY   STATUS             RESTARTS   AGE
shopstack-api   0/1     ImagePullBackOff   0          30s
```
What is wrong? What is the first command you run?

<details><summary>Answer</summary>

Kubernetes cannot pull the image. The image name or tag is wrong, or Docker Hub is unreachable. `RESTARTS: 0` means the container never even started — it failed before that.

First command:
```bash
kubectl describe pod shopstack-api
```
Read the Events section — it will show the exact pull error message including what image name it tried.

</details>

---

**Output 4 — from `kubectl logs shopstack-api`:**
```
{"event": "db_connect_attempt", "attempt": 1, "host": "db"}
{"event": "db_connect_failed", "attempt": 1, "error": "[Errno -2] Name or service not known"}
...
{"event": "db_connect_exhausted", "max_attempts": 12}
ERROR: Application startup failed. Exiting.
```
Three questions:
1. What host is the API trying to reach?
2. Why can't it find it?
3. What Kubernetes object would fix this?

<details><summary>Answer</summary>

1. The API is trying to reach host `db` on port 5432 (Postgres default)
2. There is no Kubernetes Service named `db` — DNS cannot resolve it. In Docker Compose this worked automatically because Compose created DNS for every service name. In Kubernetes you must create a Service object explicitly.
3. A `kind: Service` named `db` with a selector pointing at the db Pod — covered in Day 11.

</details>

---
