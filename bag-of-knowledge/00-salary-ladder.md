[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# The Salary Ladder

**Open this once a week. Ask yourself: where am I on this ladder right now?**
**Then close it. Go back to the terminal.**

This file answers one question permanently:
*How much do I need to know, in which tools, to earn which salary?*

It does not change. The market has spoken on these tiers.
Every tier builds on the one before it — you do not skip.

---

## How To Read This File

Each tier shows:
- What the market is actually paying for at that level
- Which tools matter most — ranked by impact on your salary at that tier
- The exact knowledge floor per tool — concepts only, not commands
- The proof that gets you hired — what your portfolio must show

The tiers are not years of experience. They are depth of understanding
combined with proof of real work. You can reach $55/hr in 18 months
if you build real things and understand what you built.

---

## The Hire Importance Rating

Every tool has a rating out of 10 per tier.
This tells you where to spend your hours at each level.

```
10 — Non-negotiable. Interviewers test this every time.
8  — Expected. Gaps here cost you offers.
6  — Good to have. Differentiates you from equal candidates.
4  — Bonus. Rarely tested at this tier but impressive.
2  — Ignore for now. Not worth your time at this level.
```

---

# TIER 1 — $25 to $40/hr
## The Floor. Getting In The Door.

> This is not the finish line. This is the entry point.
> The goal here is simple: prove you can operate, not just recite.
> One real project understood deeply beats ten tools studied shallowly.

### What The Market Is Paying For

An engineer who can:
- Run and operate a containerised application without hand-holding
- Read logs, find the problem, and fix it without panicking
- SSH into a Linux server and navigate it confidently
- Write and push code using Git without breaking things
- Explain what they built and why every decision was made

They are not paying for expertise. They are paying for operational confidence
and the ability to learn fast on the job.

### Tool Priority At This Tier

| Tool | Importance | Why It Matters Here |
|---|---|---|
| Docker | 10/10 | Everything runs in containers. No Docker = no job. |
| Linux | 9/10 | Every server is Linux. You must navigate it cold. |
| Git | 8/10 | Every pipeline starts with a push. Must be fluent. |
| Networking | 8/10 | Ports, DNS, firewalls — Docker and AWS build on this. |
| Kubernetes | 6/10 | Basics only. Know what a Pod and Deployment are. |
| Bash | 6/10 | Health checks, deploy scripts, glue between tools. |
| GitHub Actions | 5/10 | Basic pipeline. Build, test, push. |
| Observability | 4/10 | Know the three pillars. Have one working dashboard. |
| Terraform | 3/10 | Know what it is. Have seen a plan output. |
| Ansible | 2/10 | Know it exists. Know what idempotency means. |
| ArgoCD | 2/10 | Know what GitOps means conceptually. |

### Knowledge Floor Per Tool At This Tier

**Docker — must own all of this**
- Image vs container — cold, no hesitation, every time
- Lifecycle: pull → run → stop → rm → rmi and why the order matters
- Flags: `-d`, `-it`, `--name`, `-p`, `-e`, `-v` — what each does
- Logs and debugging: `docker logs`, `docker exec`, `docker inspect`
- Networking: why localhost breaks, what a custom network does
- Volumes: named volume vs bind mount, when data survives
- Dockerfile: `FROM`, `WORKDIR`, `COPY`, `RUN`, `CMD`, layer cache
- Compose: `up -d`, `down`, `logs`, `ps`, `exec`

**Linux — must own all of this**
- Navigate filesystem without thinking: `cd`, `ls`, `pwd`, `cat`
- Read logs: `tail -f`, `grep`, `grep -A/-B/-C`
- Check ports: `ss -tulnp`
- Check disk: `df -h`, `du -sh`
- Manage processes: `ps aux`, `kill -9`
- SSH into EC2 without help
- File permissions: `chmod`, what `rwx` means
- Curl an endpoint and read the response

**Git — must own all of this**
- The three areas: working directory, staging, repository
- `add`, `commit`, `push`, `pull`, `clone`, `status`, `diff`
- Branch: create, switch, merge
- Stash: save and restore mid-work changes
- Resolve a merge conflict manually
- Read `git log --oneline` and understand what you are seeing

**Networking — must own all of this**
- IP address vs port — the building and apartment analogy
- Private vs public IP — why EC2 has both
- What DNS does — name to IP translation
- TCP vs UDP — when each is used
- What a firewall rule is — inbound vs outbound
- What NAT does — why your laptop has a private IP
- `curl`, `ping`, `ss`, `dig` — what each one tests

**Kubernetes — floor only**
- What a Pod is — smallest deployable unit
- What a Deployment does — manages Pods, handles restarts
- What a Service does — stable endpoint for a set of Pods
- `kubectl get pods`, `kubectl describe pod`, `kubectl logs`
- What CrashLoopBackOff means and first two debug steps

**Bash — floor only**
- Write a 30-line script without googling `if` syntax
- Variables, conditionals, loops — the three building blocks
- Exit codes — `$?`, what 0 means, what non-zero means
- `&&` and `||` — chain commands based on success or failure
- `set -e` — why it belongs at the top of every deploy script

**GitHub Actions — floor only**
- What a workflow file is and where it lives
- What a trigger event is — `push`, `pull_request`
- What a runner is — disposable Linux VM, resets every run
- Wire a `docker build` → `docker push` pipeline end to end
- Store a secret and reference it in a workflow

**Observability — awareness only**
- The three pillars: metrics, logs, traces — what each answers
- What Prometheus is — pull-based metrics scraper
- What Grafana is — dashboard on top of Prometheus
- ShopStack exposes `/api/metrics` — that is the scrape target

**Terraform — awareness only**
- What infrastructure as code means and why it exists
- What `terraform plan` does vs `terraform apply`
- What state is — the map between your code and real cloud resources

**Ansible — awareness only**
- What configuration management is
- What idempotency means — run it twice, same result

**ArgoCD — awareness only**
- What GitOps means — Git is the source of truth
- ArgoCD watches Git and syncs the cluster — it does not build

### The Proof That Gets You Hired At This Tier

```
ShopStack running on EC2
  ✅ All 5 containers healthy
  ✅ Store UI accessible in browser
  ✅ You can explain every container, every network, every volume
  ✅ You can break it and fix it in front of an interviewer
  ✅ GitHub repo with clean commit history
  ✅ Docker images on Docker Hub
```

One project. Understood completely. That is the $40/hr proof.

---

# TIER 2 — $40 to $55/hr
## The First Jump. Proving You Can Do More Than Operate.

> You are still junior to mid. But now you have production experience
> or a portfolio that simulates it convincingly.
> The jump from Tier 1 to Tier 2 happens when you stop asking
> "how do I run this" and start asking "why is this designed this way."

### What The Market Is Paying For

An engineer who can:
- Deploy to Kubernetes with confidence — not just run commands
- Build and maintain a CI/CD pipeline end to end
- Observe a system and find problems before users report them
- Write infrastructure as code that a teammate can read and extend
- Automate repetitive work so humans do not have to do it

### What You Add On Top Of Tier 1

| Tool | Importance | What You Add |
|---|---|---|
| Kubernetes | 9/10 | Full deployment cycle, probes, PVCs, Secrets, rolling updates |
| GitHub Actions | 8/10 | Multi-job pipelines, caching, environment promotion |
| ArgoCD | 7/10 | Full GitOps workflow — app definition, sync, rollback |
| Observability | 8/10 | Working dashboards, alert rules, structured log querying |
| Terraform | 7/10 | Write real resources, manage state in S3, plan before apply |
| Bash | 7/10 | Health check scripts, backup scripts, deploy automation |
| Ansible | 5/10 | Playbook that configures a real server end to end |

### Knowledge Added Per Tool At This Tier

**Kubernetes — full operational depth**
- Write Deployment, Service, ConfigMap, Secret manifests from scratch
- Liveness vs readiness probes — what each does when it fails
- PersistentVolumeClaim — how stateful apps get disk storage
- Rolling updates — how Kubernetes deploys without downtime
- `kubectl rollout status`, `kubectl rollout undo`
- Namespaces — logical separation within a cluster
- Read and understand `kubectl describe` output fully
- Debug a CrashLoopBackOff from scratch without help

**GitHub Actions — pipeline depth**
- Multi-job pipelines with dependencies between jobs
- Caching dependencies — why it matters and how to implement it
- Environment secrets per environment — dev vs prod
- Build → test → push → deploy as one connected pipeline
- Trigger on tag push for release automation

**ArgoCD — full GitOps workflow**
- Create and configure an Application manifest
- Understand Synced vs OutOfSync vs Degraded vs Healthy
- Automated sync vs manual sync — when to use each
- Rollback from UI and CLI
- Understand why image tag change in manifest triggers deployment

**Observability — working system**
- Prometheus scrape config — how to add ShopStack as a target
- Write basic PromQL: `rate()`, `sum()`, `avg()`
- Build a Grafana dashboard with at least 4 panels
- Write one alert rule — what expr, for, labels, annotations mean
- Understand structured JSON logs and why they beat plain text
- Query logs with Loki in Grafana

**Terraform — real infrastructure**
- Write `aws_instance`, `aws_security_group`, `aws_vpc` resources
- Remote state in S3 with DynamoDB locking
- Variables and outputs — parameterise your modules
- `terraform plan` output — read it and catch mistakes before apply
- Destroy and rebuild the same infrastructure reproducibly

**Bash — automation scripts**
- Health check script that checks all 5 ShopStack services
- Deploy script that builds, tags, pushes, and restarts
- Error handling with `set -e` and meaningful exit codes
- Logging with timestamps to a file

**Ansible — real playbook**
- Playbook that installs Docker on a fresh EC2
- Variables and templates — Jinja2 basics
- Idempotency — run twice, second run changes nothing
- Tags — run only the tasks you need

### The Proof That Gets You Hired At This Tier

```
ShopStack on Kubernetes
  ✅ Full K8s manifests written from scratch
  ✅ GitHub Actions pipeline builds and pushes on every commit
  ✅ ArgoCD syncs cluster from Git automatically
  ✅ Grafana dashboard showing real ShopStack metrics
  ✅ Terraform provisions the EC2 or EKS cluster
  ✅ Ansible configures the server
  ✅ Health check and deploy scripts in scripts/
```

---

# TIER 3 — $55 to $80/hr
## Mid-Level. You Own Systems, Not Just Tasks.

> This tier is earned on the job, not in a course.
> The knowledge here comes from having seen things break in production,
> having been responsible for fixing them, and having built systems
> other engineers depend on.

### What The Market Is Paying For

An engineer who can:
- Design the infrastructure, not just implement the spec
- Own the reliability of a system — SLOs, error budgets, runbooks
- Lead a junior engineer through a problem
- Make architectural decisions and defend them
- Reduce toil — automate anything that happens more than once

### What You Add On Top Of Tier 2

| Tool | Importance | What You Add |
|---|---|---|
| Kubernetes | 10/10 | Multi-service, multi-namespace, Ingress, HPA, resource limits |
| Observability | 9/10 | SLOs, error budgets, on-call runbooks, distributed tracing |
| Terraform | 8/10 | Reusable modules, workspace strategy, drift detection |
| AWS | 9/10 | EKS, RDS, ALB, VPC design, IAM least privilege, cost awareness |
| Security | 8/10 | Secrets management, RBAC, network policies, scanning |
| Bash | 7/10 | Complex automation, argument parsing, safe production scripts |

### Knowledge Added Per Tool At This Tier

**Kubernetes — architect level**
- Ingress controllers — route external traffic to services
- Horizontal Pod Autoscaler — scale on CPU and custom metrics
- Resource requests and limits — why they matter for scheduling
- Network policies — restrict pod-to-pod communication
- RBAC — who can do what in the cluster
- Multi-namespace design — isolating environments

**Observability — reliability engineering**
- Define SLOs — what good looks like for ShopStack
- Error budget — how much failure is acceptable
- On-call runbook — what to do when each alert fires
- Distributed tracing concepts — following a request across services
- Alert fatigue — why bad alerts are worse than no alerts

**Terraform — team-scale**
- Write reusable modules other engineers can use
- Workspace strategy — how to manage dev/staging/prod
- Detect and handle drift — when reality diverges from state
- Import existing resources into state

**AWS — production cloud**
- EKS — managed Kubernetes, node groups, IAM roles for pods
- RDS — managed Postgres, connection pooling, backups
- ALB — application load balancer, health checks, SSL termination
- VPC design — public vs private subnets, NAT gateway
- IAM — least privilege, roles vs users, service accounts
- Cost awareness — what costs money and how to control it

### The Proof That Gets You Hired At This Tier

```
This tier is proven by experience, not portfolio alone.
But the portfolio must show:
  ✅ Full AWS infrastructure provisioned by Terraform
  ✅ EKS cluster with real workloads
  ✅ RDS replacing containerised database
  ✅ Monitoring with SLO definitions and alert runbooks
  ✅ Evidence of having debugged real production problems
```

---

# TIER 4 — $80 to $120/hr
## Senior. You Design Systems Other People Build.

> This is where judgment becomes the product.
> You are not hired for your hands. You are hired for your decisions.
> The tools are the same. The difference is you understand the tradeoffs
> deeply enough to know which tool to use, when not to use it,
> and how to explain the decision to a team.

### What The Market Is Paying For

An engineer who can:
- Design the entire platform — not just implement it
- Make build vs buy decisions with confidence
- Set standards the team follows
- Mentor junior and mid engineers
- Own the incident process — not just respond to alerts
- Reduce complexity, not add it

### What Separates This Tier From Tier 3

At Tier 3 you implement what you are told to implement well.
At Tier 4 you decide what should be implemented and what should not.

The knowledge is similar. The judgment is different.
Judgment only comes from having been wrong in production,
having owned the consequences, and having learned from it.

You cannot study your way to Tier 4.
You earn it by doing Tiers 1, 2, and 3 with full attention.

---

# The Weekly Check-In

Open this file once a week. Answer these three questions:

**1. Which tier am I operating at right now?**
Not which tier I have studied. Which tier I can prove with working systems.

**2. What is the single highest-leverage thing I can add this week?**
Look at the tool importance ratings for your current tier.
The highest number you have not yet proven is your answer.

**3. What is the one proof item I am still missing?**
Look at the proof section for your current tier.
The unchecked box is your target for this week.

---

*The ladder is clear. The path is real. The only variable is the work.*
