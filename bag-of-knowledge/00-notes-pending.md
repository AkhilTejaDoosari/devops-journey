[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Notes Pending Generation

**Open this file the evening before each tool week starts.**
**Trigger note generation. Flip the status. Close it.**

This file exists because 7 tools have no runbook notes yet.
The tool files in this folder have Sections 0, 1, and 2 complete.
Sections 3, 4, 5, and 6 need your actual runbook notes to be full.

The flow every time:
```
Evening before tool week starts
  → Open this file
  → Find the tool
  → Copy the exact prompt below
  → Paste it to me
  → I generate the full runbook notes
  → You add them to devops-runbook/
  → Flip status from ❌ to ✅
  → Open the tool file — complete Sections 3, 4, 5, 6
  → Start the week
```

---

## Status Board

| Tool | Runbook Notes | Generate Before | Status |
|---|---|---|---|
| Linux | ✅ exists | — | ✅ Done |
| Git | ✅ exists | — | ✅ Done |
| Networking | ✅ exists | — | ✅ Done |
| Docker | ✅ exists | — | ✅ Done |
| Kubernetes | ❌ empty | Day 7 evening | ⬜ Pending |
| GitHub Actions | ❌ empty | Day 14 evening | ⬜ Pending |
| ArgoCD | ❌ empty | Day 14 evening | ⬜ Pending |
| Observability | ❌ empty | Day 21 evening | ⬜ Pending |
| Terraform | ❌ empty | Day 35 evening | ⬜ Pending |
| Ansible | ❌ empty | Day 42 evening | ⬜ Pending |
| Bash | ❌ empty | Day 42 evening | ⬜ Pending |

---

## Generation Prompts — Copy and Paste Exactly

Each prompt below is ready to use. Copy it. Paste it to me.
I generate the full runbook notes for that tool.
You save them to `devops-runbook/notes/[tool]/`.

---

### Kubernetes — Use Day 7 Evening

```
Generate full Kubernetes runbook notes for devops-runbook.
ShopStack context: 5-service 3-tier app moving from Docker Compose to K8s.
Format: match the existing Linux, Git, Networking, Docker notes in devops-runbook.
Cover these topics in this order:

01 — What Kubernetes Is And Why It Exists
     Problem Docker Compose cannot solve at scale
     Desired state machine mental model
     Control plane vs worker nodes
     How this maps to ShopStack moving from Compose to K8s

02 — Pods
     What a Pod is — one or more containers, shared network
     Why you never run a bare Pod in production
     Pod lifecycle: Pending → Running → Succeeded/Failed
     ShopStack: what each service looks like as a Pod

03 — Deployments
     What a Deployment does — manages Pods, handles restarts
     ReplicaSet — what it is and why you do not touch it directly
     Rolling update — how Kubernetes deploys without downtime
     Rollback — kubectl rollout undo
     ShopStack: api-deployment.yaml written from scratch

04 — Services
     What a Service does — stable endpoint for changing Pods
     ClusterIP vs NodePort vs LoadBalancer — when to use each
     Kubernetes DNS — how api reaches db by name
     ShopStack: api-service.yaml, db-service.yaml, frontend-service.yaml

05 — State — ConfigMap, Secret, PVC
     ConfigMap — non-sensitive configuration
     Secret — sensitive data, base64 encoded
     PersistentVolumeClaim — how Postgres gets disk storage
     What happens to data when a Pod restarts with and without PVC
     ShopStack: db-secret.yaml, db-configmap.yaml, db-pvc.yaml

06 — Probes
     Liveness probe — restart if dead
     Readiness probe — remove from load balancer if not ready
     What happens when each one fails
     ShopStack: liveness and readiness on the API container

07 — Debugging
     CrashLoopBackOff — what it means, first two commands
     kubectl describe — what to read in the Events section
     kubectl logs — what it shows and what it does not
     kubectl exec — when to use it over logs
     kubectl get events — reading the cluster timeline

08 — kubectl Command Reference
     Full command reference matching the combat sheet format
     Every command with ShopStack example

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### GitHub Actions — Use Day 14 Evening

```
Generate full GitHub Actions runbook notes for devops-runbook.
ShopStack context: CI pipeline that builds Docker images and pushes to Docker Hub
on every push to main. CD handled by ArgoCD separately.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — What CI/CD Is And Why It Exists
     The problem of manual builds and deployments
     CI vs CD — not the same thing
     Where GitHub Actions fits — CI only, not CD
     Where ArgoCD fits — CD only, not CI
     ShopStack pipeline: push to main → build → push to Docker Hub → ArgoCD picks up

02 — GitHub Actions Fundamentals
     Workflow file — where it lives, what it is
     Trigger events — push, pull_request, workflow_dispatch
     Runner — what it is, why it resets every run, why checkout is required
     Jobs and steps — the difference and the dependency model
     actions/checkout — what it does and what happens without it

03 — Writing The ShopStack Pipeline
     Trigger on push to main
     Job: build-and-push
     Step by step: checkout, login to Docker Hub, build, push
     docker/login-action — what it does
     docker/build-push-action — what it does
     Secrets — where they live, how to reference them
     Full working workflow file for ShopStack API image

04 — Secrets and Environment Variables
     Where to store Docker Hub password in GitHub
     How to reference secrets in workflow — ${{ secrets.NAME }}
     Environment variables — env: block vs $GITHUB_ENV
     What never to hardcode in a workflow file

05 — Caching
     Why builds are slow without caching
     How Docker layer cache works in GitHub Actions
     actions/cache — what it does and when to use it
     The performance difference with and without cache

06 — Debugging Failed Pipelines
     Where to find the error in GitHub UI
     Reading step logs — what to look for
     Common failures and their causes
     Re-running a failed job

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### ArgoCD — Use Day 14 Evening

```
Generate full ArgoCD runbook notes for devops-runbook.
ShopStack context: ArgoCD watches the shopstack GitHub repo k8s/ folder.
When a manifest changes — ArgoCD syncs the k3s cluster automatically.
GitHub Actions builds the image. ArgoCD deploys it. They are separate.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — What GitOps Is And Why It Exists
     The problem of manual kubectl apply in production
     Git as source of truth — what this means in practice
     What ArgoCD does — watches Git, syncs cluster
     What ArgoCD does NOT do — it does not build images
     ShopStack flow: commit → GitHub Actions builds → ArgoCD deploys

02 — ArgoCD Architecture
     Application manifest — what it is, what is in it
     Source: the Git repo and path ArgoCD watches
     Destination: the cluster and namespace to sync to
     Sync policy: manual vs automated

03 — Sync Status and Health
     Synced — Git and cluster match
     OutOfSync — Git changed, cluster not updated yet
     Healthy — all resources running correctly
     Degraded — something is broken
     Progressing — update in flight

04 — The ShopStack GitOps Workflow
     Install ArgoCD on k3s cluster
     Connect ArgoCD to shopstack GitHub repo
     Create Application pointing to infra/k8s/
     Push a manifest change — watch ArgoCD sync automatically
     Push a new image tag — why the manifest must change too

05 — Rollback
     How to roll back from ArgoCD UI
     How to roll back from CLI — argocd app rollback
     Why rollback in ArgoCD is not the same as kubectl rollout undo
     When to use each

06 — argocd CLI Reference
     Full command reference matching the combat sheet format
     Every command with ShopStack example

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### Observability — Use Day 21 Evening

```
Generate full Observability runbook notes for devops-runbook.
ShopStack context: Python API exposes /api/metrics in Prometheus format.
Go worker writes structured JSON logs. Prometheus scrapes the API.
Grafana reads Prometheus. Loki collects logs. Alertmanager fires alerts.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — The Three Pillars
     Metrics — what they answer, what they do not answer
     Logs — what they answer, what they do not answer
     Traces — what they answer, when you need them
     Which pillar answers which question in ShopStack

02 — Prometheus
     Pull-based model — Prometheus scrapes, app does not push
     What a scrape target is — ShopStack /api/metrics
     What Prometheus format looks like — counters, gauges, histograms
     scrape_config — how to add ShopStack as a target
     Basic PromQL: rate(), sum(), avg(), histogram_quantile()

03 — Grafana
     What Grafana is — dashboard on top of data sources
     Connecting Prometheus as a data source
     Building a dashboard: panels, queries, visualisations
     ShopStack dashboard: request rate, error rate, latency, uptime

04 — Loki and Structured Logs
     What Loki is — log aggregation, not full-text search
     Why structured JSON logs matter — queryable fields
     ShopStack worker logs — what they contain, how to query them
     LogQL basics — filter by field, count errors

05 — Alertmanager and Alert Rules
     What an alert rule is — expr, for, labels, annotations
     What Alertmanager does — routes and deduplicates alerts
     Writing one alert rule for ShopStack API downtime
     Alert fatigue — why bad alerts are worse than no alerts

06 — The /api/metrics Endpoint
     What ShopStack exposes at /api/metrics
     How to read Prometheus format output
     Which metrics matter for a junior DevOps interview

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### Terraform — Use Day 35 Evening

```
Generate full Terraform runbook notes for devops-runbook.
ShopStack context: Terraform provisions EC2, VPC, Security Groups,
EKS cluster, RDS database, and ALB on AWS.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — What Infrastructure As Code Is And Why It Exists
     The problem of manual cloud console clicking
     Reproducibility — same terraform apply = same infrastructure
     Version control for infrastructure
     ShopStack without Terraform vs with Terraform

02 — Terraform Fundamentals
     HCL — what it looks like, how to read it
     Provider — what it is, AWS provider config
     Resource — the building block, what a resource block looks like
     Data source — read existing infrastructure, do not create
     Variables and outputs — parameterise and expose values

03 — The Core Workflow
     terraform init — download providers, set up backend
     terraform plan — what would change, always run first
     terraform apply — make it real
     terraform destroy — delete everything this config created
     State — what it is, why it is critical, never edit manually

04 — Remote State
     Why local state is dangerous on a team
     S3 backend — how to configure it
     DynamoDB locking — prevent concurrent applies
     ShopStack state stored in S3

05 — Writing ShopStack Infrastructure
     aws_vpc — the network
     aws_security_group — firewall rules
     aws_instance — EC2 for ShopStack
     aws_db_instance — RDS for Postgres
     ShopStack terraform apply output — what gets created

06 — terraform CLI Reference
     Full command reference matching the combat sheet format
     Every command with ShopStack example

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### Ansible — Use Day 42 Evening

```
Generate full Ansible runbook notes for devops-runbook.
ShopStack context: Ansible configures fresh EC2 instances —
installs Docker, clones ShopStack, sets environment variables,
starts the stack. Agentless — uses SSH.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — What Configuration Management Is And Why It Exists
     The problem with bash scripts at scale
     Idempotency — the most important concept in Ansible
     Agentless model — how Ansible uses SSH, no agent needed
     ShopStack without Ansible vs with Ansible

02 — Inventory
     What an inventory file is — list of hosts
     Static inventory — INI format
     Groups — web, db, all
     ShopStack inventory — EC2 instance

03 — Playbooks
     Playbook structure — play, tasks, modules
     Task structure — name, module, parameters
     Core modules: apt, copy, template, service, user, git, command
     ShopStack playbook: install Docker on fresh EC2

04 — Variables and Templates
     vars: block in a playbook
     Referencing variables — {{ variable_name }}
     Jinja2 basics — what it is, how templates work
     ShopStack: nginx.conf as a template with variables

05 — Tags and Handlers
     Tags — run only specific tasks
     Handlers — run only when notified, once at the end
     ShopStack: tag docker tasks separately from nginx tasks

06 — ansible and ansible-playbook CLI Reference
     Full command reference matching the combat sheet format
     Every command with ShopStack example

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

### Bash — Use Day 42 Evening

```
Generate full Bash runbook notes for devops-runbook.
ShopStack context: Bash scripts in shopstack/scripts/ —
health-check.sh checks all 5 services, deploy.sh builds
and pushes images, setup.sh installs Docker on EC2.
Format: match existing notes in devops-runbook.
Cover these topics in this order:

01 — What Bash Is And Why It Exists In DevOps
     Bash as the glue between tools
     When to use Bash vs Python vs Ansible
     ShopStack scripts — what each one does

02 — Script Foundations
     Script header — #!/bin/bash and set -e
     Variables — assignment, referencing, quoting rules
     Command substitution — VAR=$(command)
     Exit codes — $?, 0 = success, non-zero = failure

03 — Conditionals and Loops
     if/then/fi — the structure
     Test operators — -f file exists, -z empty string, -eq equal
     for loop — iterate over list
     while loop — repeat until condition
     ShopStack: loop over all 5 container names

04 — Functions and Error Handling
     Function declaration and calling
     Passing arguments — $1, $2, $@
     set -e — exit on first error
     trap — cleanup on exit or error
     Meaningful error messages

05 — Working With Commands
     Pipelines — | chaining
     Redirection — >, >>, 2>&1
     grep -q — silent match for conditionals
     curl -sf — silent curl that fails on HTTP error
     ShopStack: curl the health endpoint and check exit code

06 — The ShopStack Scripts
     health-check.sh — full walkthrough line by line
     deploy.sh — full walkthrough line by line
     What makes a production-safe script vs a fragile one

99 — Interview Prep
     10 questions written the way an interviewer asks them
     Answers written the way you say them out loud
```

---

## After Notes Are Generated — Complete The Tool File

When runbook notes exist for a tool, open the tool file
and complete the placeholder sections:

```
Section 3 — Combat Sheet
  Fill in: What breaks | Command | ShopStack example
  Pull from: 08-command-reference in the runbook notes

Section 4 — ShopStack Map
  Fill in: Situation | File or Command | Knowledge needed
  Pull from: ShopStack-specific scenarios in the notes

Section 5 — Peace of Mind
  Fill in: 10 questions with toggle answers
  Pull from: 99-interview-prep in the runbook notes

Section 6 — AI Split
  Fill in: learn yourself, use AI, right questions
  Pull from: your own experience during the tool week
```

---

*Every ❌ becomes ✅ before the week starts. No exceptions.*
