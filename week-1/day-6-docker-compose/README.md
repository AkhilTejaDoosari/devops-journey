[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md)

---

# Day 6 — Docker Compose

**Date:** April 25 2026   
**Read before session:** [10-docker-compose](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/10-docker-compose)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Write the full ShopStack docker-compose.yml from scratch. Bring all 5 services up in correct dependency order. Hit every endpoint. Break it 8 times. Fix it 8 times. Walk away knowing exactly what each line does and why order matters.   

---

## Knowledge — What This Topic Is Really About

### The Problem Compose Solves

Days 1 through 5 you ran containers manually — one `docker run` command per container, one network create, one volume create, flags typed out every time. That works for one container. ShopStack has five. Running them manually means five commands, in the right order, with the right flags, every single time. Miss one flag and the stack is broken. Change a port and you hunt through five commands to find it.

Compose collapses all of that into **one file and one command.**

The analogy: imagine you are opening a restaurant. You have a chef, a waiter, a cashier, a dishwasher, and a manager. Opening manually means you call each one individually, tell them where to stand, and hope they show up in the right order. Compose is the **opening checklist pinned to the back door** — one document that defines who does what, in what order, and how they communicate. The manager (Docker) reads the checklist and handles the rest.

### The Mental Model — Three Sections

Every `docker-compose.yml` has exactly three top-level blocks. Memorise these three words:

```
services:   ← the containers (who runs)
networks:   ← the communication lanes (how they talk)
volumes:    ← the persistent storage (what survives)
```

Everything else — ports, environment variables, healthchecks, depends_on — lives *inside* a service block. The three top-level blocks never nest inside each other.

### depends_on and Why `condition: service_healthy` Matters

`depends_on` with no condition just says: *start the DB container before the API container.* But "started" does not mean "ready to accept connections." PostgreSQL takes a few seconds to initialise after the process starts. If the API connects immediately, it hits a closed socket and crashes.

`condition: service_healthy` solves this by waiting for the **healthcheck** to pass — a real probe that pings the database and confirms it is accepting queries — before starting the next service.

The analogy: you would not send a customer to the kitchen before the chef has finished prepping. `service_healthy` is the chef texting "ready" before the waiter opens the door.

### Networks — Two Lanes, One Reason

ShopStack uses two networks: `web` and `backend`.

- `web` — connects `frontend` ↔ `api`. The browser talks to frontend, frontend talks to api.
- `backend` — connects `api` ↔ `db` ↔ `worker` ↔ `adminer`.

The `frontend` container has **no access to the database.** It is not on the `backend` network. This is intentional security design — a compromised frontend cannot reach the database directly. The API is the only gatekeeper.

The analogy: in a restaurant, waiters go to the pass (frontend ↔ api lane). Waiters do not walk into the cold storage room (backend). Only the kitchen staff do.

### ShopStack Service Dependency Chain

```
db
 └── api (waits for db: service_healthy)
      ├── worker (waits for api: service_healthy)
      └── frontend (waits for api: service_healthy)
           └── adminer (waits for db — simple depends_on, no healthcheck)
```

This is why the startup sequence in the logs always reads: db → api → worker + frontend → adminer.

---

# 📟 DAY 6 FULL CHEATSHEET — Docker Compose

## Core Compose Commands

| Command | What it does | Example |
|---|---|---|
| `docker compose up` | Start all services — foreground, logs visible | `docker compose up` |
| `docker compose up -d` | Start all services — background, terminal free | `docker compose up -d` |
| `docker compose up --build` | Rebuild images then start — use after Dockerfile changes | `docker compose up --build` |
| `docker compose down` | Stop + remove containers and network — **volumes survive** | `docker compose down` |
| `docker compose down -v` | Stop + remove everything including volumes — **data gone** | `docker compose down -v` |
| `docker compose ps` | List all services in this stack and their status | `docker compose ps` |
| `docker compose logs <service>` | See all logs for one service | `docker compose logs api` |
| `docker compose logs -f <service>` | Follow live logs for one service — Ctrl+C to stop | `docker compose logs -f api` |
| `docker compose exec <service> <cmd>` | Run a command inside a running service | `docker compose exec db psql -U shopstack` |
| `docker compose build` | Build/rebuild images only — don't start containers | `docker compose build` |
| `docker compose restart <service>` | Restart one service without touching others | `docker compose restart api` |

**[CORE]** — All commands above. These are your daily driver. Write them in the notebook.

---

## YAML File Structure

**[CORE]** — The three top-level blocks. Burn this shape into memory.

```yaml
services:         ← containers live here
  <name>:
    image:        ← use a pre-built image (db, adminer)
    build:        ← build from Dockerfile (api, worker, frontend)
    environment:  ← env vars passed into the container
    ports:        ← HOST:CONTAINER port binding
    volumes:      ← mount named volumes or host paths
    depends_on:   ← startup order + health gating
    healthcheck:  ← how Docker knows this service is ready
    networks:     ← which network lanes this service joins
    restart:      ← what to do if the container crashes

networks:         ← declare named networks here
  web:
  backend:

volumes:          ← declare named volumes here
  db-data:
```

---

## depends_on Syntax — Two Forms

**[CORE]** — Know both forms. Know when to use each.

```yaml
# Form 1 — Simple: just controls start order, no health check
depends_on:
  - db

# Form 2 — Health-gated: waits for the healthcheck to pass before starting
depends_on:
  db:
    condition: service_healthy
```

**Syntax breakdown:**
```
depends_on:
  db:                          ← the service this one waits for
    condition: service_healthy ← only start ME after db's healthcheck passes
```

Rule: use `condition: service_healthy` whenever the downstream service needs a real connection (api waiting for db). Use the simple form when order matters but readiness does not (adminer waiting for db is fine without it since adminer connects lazily).

---

## Healthcheck Block

**[CORE]** — This is what `condition: service_healthy` actually checks.

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U shopstack -d shopstack"]
  interval: 5s       ← run the check every 5 seconds
  timeout: 5s        ← if no response in 5s, count as failure
  retries: 10        ← fail 10 times before marking unhealthy
  start_period: 10s  ← give the service 10s to boot before checks begin
```

**Syntax breakdown:**
```
test: ["CMD-SHELL", "pg_isready -U shopstack -d shopstack"]
       ↑              ↑
  run in shell    the actual probe command
```

For the API, the healthcheck is a Python one-liner hitting `http://localhost:8080/api/health`.
For the frontend, it is a `wget` hitting `/nginx-health`.
For the db, it is `pg_isready` — the official PostgreSQL readiness probe.

**[WIKI]** — `pg_isready` exits 0 if postgres accepts connections, non-zero if not. Docker watches the exit code.

---

## Networks Block — Two-Lane Design

**[CORE]** — Which service lives on which lane, and why.

| Service | web | backend | Reason |
|---|---|---|---|
| frontend | ✅ | ❌ | Talks to api over web. No DB access by design. |
| api | ✅ | ✅ | Bridge — receives from frontend, reads/writes db |
| worker | ❌ | ✅ | Internal only. Talks to api and db. No public exposure. |
| db | ❌ | ✅ | Never exposed to web. Backend only. |
| adminer | ❌ | ✅ | DB UI — backend only. Port 8081 exposed to host, not to web network. |

```yaml
networks:
  web:       ← Compose creates this bridge network automatically
  backend:   ← Compose creates this bridge network automatically
```

No extra config needed. Declaring the name is enough. Compose handles the rest.

---

## Volumes Block

**[CORE]**

```yaml
volumes:
  db-data:   ← declare the named volume at the top level
```

Used inside the db service:
```yaml
volumes:
  - db-data:/var/lib/postgresql/data   ← mount it into the container
```

**Syntax breakdown:**
```
- db-data:/var/lib/postgresql/data
  ↑         ↑
named      where postgres stores all data inside the container
volume
```

Rule: `docker compose down` — volume survives. `docker compose down -v` — volume is deleted. Data gone. No recovery.

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | First command to run |
|---|---|---|
| API exits immediately on startup | DB not ready — `depends_on` missing or wrong condition | `docker compose logs api` |
| `connection refused` or `could not translate host name "X"` | `DB_HOST` set to wrong service name | Check `environment: DB_HOST:` in compose file |
| Frontend returns `502 Bad Gateway` | nginx `proxy_pass` points to wrong upstream address | `docker compose logs frontend` — check proxy config |
| `Bind for 0.0.0.0:80 failed: port is already allocated` | Another container or process owns that port | `docker ps` — find and stop the conflict |
| Data disappeared after restart | `docker compose down -v` was used — volume was deleted | No recovery. Recreate with init.sql. |
| Worker crashes immediately | API not healthy yet — `depends_on` missing `condition: service_healthy` | `docker compose logs worker` |
| Build takes forever, ignores cached layers | Dockerfile layer order changed — cache busted | Check Dockerfile, restore `requirements.txt` copy before `COPY . .` |
| `504 Gateway Timeout` on API endpoint | nginx `proxy_read_timeout` too short for slow queries | Check nginx config — increase timeout |

---

## Checklist

> 🧪 Practice Only — Not a DevOps Skill
> The docker-compose.yml already exists in the repo. You are deleting it and writing it from scratch to prove you understand every line. In a real job, you would write this once and commit it — not delete and redo it daily.
> ```bash
> cd ~/shopstack/infra
> mv docker-compose.yml docker-compose.yml.bak
> ```
> ↩️ Back to DevOps — now write it from scratch.

- [ ] Write `infra/docker-compose.yml` from scratch — no copy paste:
  - [ ] `services:` block
  - [ ] `db:` — `image: postgres:15-alpine`, `environment`, `volumes`, `healthcheck`, `networks`
  - [ ] `api:` — `build` path, `environment`, `ports`, `depends_on: condition: service_healthy`, `healthcheck`, `networks`
  - [ ] `worker:` — `build` path, `environment`, `depends_on`, `networks`
  - [ ] `frontend:` — `build` path, `ports`, `depends_on`, `healthcheck`, `networks`
  - [ ] `adminer:` — `image`, `ports`, `depends_on`, `networks`
  - [ ] `networks:` block — `web` and `backend`
  - [ ] `volumes:` block — `db-data`
- [ ] `docker compose up --build` — watch every service start in order
- [ ] `docker compose ps` — all 5 healthy
- [ ] `curl http://localhost` — green banner (or open browser)
- [ ] `curl http://localhost/api/health`
- [ ] `curl http://localhost/api/products`
- [ ] `curl http://localhost/api/orders`
- [ ] `curl http://localhost/api/metrics`
- [ ] **Break 1** — remove `depends_on` from api and worker — startup race condition
- [ ] **Break 2** — wrong `DB_HOST` — DNS failure
- [ ] **Break 3** — wrong nginx `proxy_pass` — 502
- [ ] **Break 4** — reorder Dockerfile layers — cache miss
- [ ] **Break 5** — remove DB volume mapping — data wipe
- [ ] **Break 6** — delete `asyncpg` from requirements — Python crash
- [ ] **Break 7** — nginx timeout too short — 504
- [ ] **Break 8** — port conflict — container won't start
- [ ] `docker compose down`
- [ ] Answer out loud: what does `depends_on: condition: service_healthy` do that plain `depends_on` does not?

**Session win condition:** All 8 breaks done and fixed. Stack healthy. You can explain every line of the compose file without looking.

---

Ready to test yourself? → [Test](./test.md)
