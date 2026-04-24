[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Observability

**Hire importance at $25–40/hr:** 4/10 — know the three pillars, have one dashboard. Grows to 8/10 at $55/hr.

**One line:** You cannot fix what you cannot see. Observability is how you see inside a running system without stopping it.

> ⏳ Runbook notes not generated yet.
> Come back Day 21 evening. Use the prompt in `00-notes-pending.md` to generate them.
> Sections 3, 4, 5, and 6 will be completed after notes are generated.

---

## 0 — Before You Open A Terminal

### What a developer does that makes observability necessary

A developer writes a FastAPI endpoint that processes orders. It works in testing. In production, some requests take 10 seconds and users complain. The developer cannot reproduce it locally. You cannot stop the server to investigate — orders are still coming in.

Observability is the answer. The developer instruments the app — adds code that records metrics, writes structured logs, and emits traces. You collect that data, store it, and make it queryable. When something breaks in production, you open a dashboard, read the metrics, query the logs, and find the problem without touching the running system.

### What the developer writes that you collect

A developer writes code that exposes a `/metrics` endpoint. ShopStack's Python API exposes this at `/api/metrics`. When Prometheus scrapes that endpoint, it reads the metrics the developer exposed — request count, error count, request duration. The developer chose what to measure. You collect and display it.

A developer writes log statements — `print(json.dumps({"level": "error", "message": "db timeout", "duration_ms": 5000}))`. Structured JSON logs are queryable. Plain text logs are not. The developer chose the format. You collect and query it.

### The three pillars — one sentence each

**Metrics** answer: how many, how fast, how long. Numbers over time. Request rate, error rate, response latency. Prometheus collects them. Grafana displays them.

**Logs** answer: what exactly happened at this specific moment. Text records of events. Container stdout, application errors, request details. Loki collects them. Grafana queries them.

**Traces** answer: where did this specific request spend its time across multiple services. A single request touches frontend, API, database — a trace shows the full path and where the slowness is. Jaeger or Tempo collects them. Not required at $40/hr.

### ShopStack observability endpoints

```
http://EC2_IP:8080/api/health    ← is the API alive — binary yes/no
http://EC2_IP:8080/api/metrics   ← Prometheus format metrics the API exposes
docker compose logs -f api       ← live structured JSON logs from the API
docker compose logs -f worker    ← worker health pings and processing logs
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- Metrics answer how many/fast/long — logs answer what exactly happened — know the difference
- Prometheus is pull-based — it scrapes `/api/metrics` on a schedule — the app does not push
- What Prometheus format looks like — counter, gauge, histogram — and what each type tracks
- `rate()` in PromQL — why you use rate on a counter instead of the raw counter value
- Grafana connects to Prometheus as a data source — it does not collect metrics itself
- A dashboard panel is a query result visualised — you write the query, Grafana draws it
- An alert rule is a PromQL expression that fires when a threshold is crossed
- Structured JSON logs are queryable — plain text logs are not — why the developer's format matters
- `docker compose logs -f SERVICE` — your first look at what the app is actually doing
- The health endpoint — `/api/health` — is the simplest observability signal you have

**When you can explain all ten without hesitation — stop. Move to Terraform.**

### What unlocks higher pay

At $55/hr you add: writing PromQL queries for real dashboards, alerting with Alertmanager and routing to Slack, Loki for log aggregation and LogQL for querying, SLO definition and error budget tracking, structured logging standards for teams, and distributed tracing concepts with OpenTelemetry. At $80/hr you design the entire observability stack and define what good looks like for a production system.

📚 Deep dive → Coming Day 21 evening — see `00-notes-pending.md`

---

## 2 — The Bag

### Own These Cold

Metrics and logs are not the same thing and they answer different questions. Metrics tell you the shape of what is happening — request rate is 500/second, error rate is 2%, p99 latency is 800ms. Logs tell you what specifically happened — at 14:23:07 this specific request to `/orders` failed with this specific database timeout error. Use metrics to detect a problem. Use logs to diagnose it.

Prometheus is pull-based. It does not wait for the app to send data. It goes to the app on a schedule — every 15 seconds by default — and reads the `/metrics` endpoint. The developer exposes the endpoint. Prometheus reads it. This means the app must be running and reachable for Prometheus to collect data.

A counter only goes up — total requests served, total errors, total bytes sent. You never graph a counter directly — it just keeps climbing and tells you nothing about the rate. `rate(counter[5m])` converts it to per-second rate over the last 5 minutes. That is meaningful — requests per second, errors per second.

A gauge goes up and down — current memory usage, current active connections, current queue depth. You can graph a gauge directly because the current value is already meaningful.

Grafana is a dashboard tool. It does not collect metrics. It connects to data sources — Prometheus, Loki, databases — queries them, and visualises the results. One Grafana instance can show metrics from Prometheus and logs from Loki side by side. The developer cannot log into Prometheus directly and get a useful picture. Grafana is the interface that makes the data usable.

Structured logging means every log line is a JSON object — `{"level": "info", "msg": "order processed", "order_id": "abc123", "duration_ms": 45}`. Every field is queryable. You can ask "show me all logs where duration_ms > 1000." Plain text logs — `INFO: order processed abc123 in 45ms` — require regex to extract values. Queryable fields versus text parsing — structured logging wins every time in production.

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI after notes are generated.

- API is slow but no errors → metrics question — look at latency histogram
- Specific request failed at 14:23 → logs question — query logs for that timestamp
- Error rate spiked 10 minutes ago → metrics — `rate(errors[5m])` in Prometheus
- What exactly broke for this specific user → logs — filter by user_id or request_id
- Is the API currently alive → health endpoint — `curl http://EC2_IP:8080/api/health`
- Dashboard shows nothing → Prometheus cannot reach the scrape target — check `/api/metrics`

---

### Domain Awareness

- OpenTelemetry — vendor-neutral standard for metrics, logs, and traces instrumentation
- Distributed tracing — following a request across multiple services — Jaeger, Tempo
- Alertmanager — routes Prometheus alerts to Slack, email, PagerDuty
- Loki — log aggregation — think Prometheus but for logs
- Grafana OnCall — on-call scheduling integrated with alerting
- SLOs and error budgets — defining what reliability means in measurable terms

---

### The one insight that separates copiers from understanders

The developer instruments the app. You collect what they instrumented.

If the developer did not add metrics, Prometheus has nothing to scrape. If the developer wrote plain text logs, you cannot query them meaningfully. If the developer did not write a health endpoint, your probe has nothing to call. Observability is a collaboration — the developer instruments, you collect and surface. When observability is missing, the question is usually not "how do I add monitoring" but "why did the developer not instrument this and how do I work with them to add it."

---

## 3 — Combat Sheet

> ⏳ To be completed after runbook notes are generated on Day 21 evening.
> Use the prompt in `00-notes-pending.md` → Observability section.
> Pull content from ShopStack-specific scenarios and the metrics endpoint.

---

## 4 — ShopStack Map

> ⏳ To be completed after runbook notes are generated on Day 21 evening.
> Pull content from ShopStack-specific scenarios in the generated notes.

---

## 5 — Peace of Mind

> ⏳ To be completed after runbook notes are generated on Day 21 evening.
> Pull content from the generated notes section: `99-interview-prep`.
> Format: 10 questions, toggle answers, written the way you say it in an interview.

---

## 6 — AI Split

> ⏳ To be completed after runbook notes are generated on Day 21 evening.
> Fill in based on your own experience during the Observability week.

### Preview — Learn this yourself no matter what

- The three pillars — metrics vs logs vs traces — what each answers — tested in every interview
- Why `rate()` exists — never graph a counter directly — interviewers test this
- Prometheus is pull-based — the app exposes, Prometheus scrapes — know the direction
- Structured logs vs plain text — why JSON matters in production
- The difference between a health check and a metric — binary vs dimensional

### Preview — Use AI for this

- PromQL queries for specific dashboard panels — request rate, error rate, latency
- Prometheus scrape config for a new service
- Alert rule syntax — `expr`, `for`, `labels`, `annotations` — first draft, you verify
- Grafana dashboard JSON export and import — syntax only

📚 Deep dive → Coming Day 21 evening — see `00-notes-pending.md`
