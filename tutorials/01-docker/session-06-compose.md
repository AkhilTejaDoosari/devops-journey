 # Session 06 — Docker Compose

**Goal:** Bring up the full ShopStack stack with one command, understand every line of every file involved, verify all five services are healthy and accessible.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Docker Compose Exists

Sessions 01-05 taught you to run containers manually — one `docker run` at a time. For a 5-service app like ShopStack, that's 5 commands with 20+ flags in the exact right order every single time.

Docker Compose replaces all of that with one file and one command.

```
Before Compose:                          After Compose:
  docker network create web
  docker network create backend
  docker volume create db-data           docker compose up -d
  docker run -d --name db ...
  docker run -d --name api ...
  docker run -d --name worker ...
  docker run -d --name frontend ...
  docker run -d --name adminer ...
```

---

## Visual Map

```
docker-compose.yml
├── services
│   ├── db        → postgres:15-alpine | backend network | db-data volume
│   ├── api       → build services/api | web + backend | port 8080
│   ├── worker    → build services/worker | backend network
│   ├── frontend  → build services/frontend | web network | port 80
│   └── adminer   → adminer:4 | backend network | port 8081
├── networks
│   ├── web       → frontend ↔ api
│   └── backend   → api ↔ db ↔ worker ↔ adminer
└── volumes
    └── db-data   → Postgres data persistence
```

---

## The Files — Every File Fully Exposed

### 📄 infra/docker-compose.yml — The Full Stack Definition

**What it does:** defines all 5 services, 2 networks, 1 volume. One file = the entire ShopStack system.

<details>
<summary>📄 Show full file — infra/docker-compose.yml</summary>

```yaml
services:

  db:
    image: postgres:15-alpine
    # pre-built image — no Dockerfile needed for db
    # postgres:15-alpine = Postgres 15 on Alpine Linux (~240MB)

    environment:
      POSTGRES_DB:       shopstack     # database name
      POSTGRES_USER:     shopstack     # username the API connects with
      POSTGRES_PASSWORD: shopstack_dev # password — matches DB_PASS in api service

    volumes:
      - db-data:/var/lib/postgresql/data
      # named volume → Postgres data directory
      # data survives container deletion
      # volume name: db-data (Compose prefixes it: infra_db-data)

      - ../db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      # bind mount → seed file
      # Postgres runs this SQL on FIRST start (when volume is empty)
      # :ro = read-only — Postgres reads it, cannot write to it

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U shopstack -d shopstack"]
      # pg_isready checks if Postgres is accepting connections
      # this is what api's depends_on waits for
      interval: 5s      # check every 5 seconds
      timeout: 5s       # fail if no response in 5 seconds
      retries: 10       # retry 10 times before marking unhealthy
      start_period: 10s # wait 10s before first check (Postgres needs time to init)

    networks:
      - backend
      # db is ONLY on backend — invisible to frontend (intentional)

  api:
    build: ../services/api
    # Compose finds Dockerfile at services/api/Dockerfile and builds it
    # result: shopstack-api image used to start this container

    environment:
      DB_HOST: db        # Docker DNS resolves "db" → db container IP
      DB_PORT: "5432"    # quoted string — prevents K8s env var collision bug
      DB_NAME: shopstack
      DB_USER: shopstack
      DB_PASS: shopstack_dev  # matches POSTGRES_PASSWORD in db service

    ports:
      - "8080:8080"
      # host port 8080 → container port 8080
      # makes API accessible from browser at YOUR_EC2_IP:8080

    depends_on:
      db:
        condition: service_healthy
        # wait for db healthcheck to PASS before starting api
        # without this: api starts before Postgres is ready → connection refused

    healthcheck:
      test: ["CMD-SHELL", "python -c \"import urllib.request; urllib.request.urlopen('http://localhost:8080/api/health')\""]
      # checks if the API is actually responding — not just that the process started
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 15s  # give API 15s to start before first check

    restart: unless-stopped
    # restart automatically if it crashes
    # stops only when you run docker compose down

    networks:
      - backend  # can reach db
      - web      # can reach frontend
      # api is on BOTH networks — the bridge between the two tiers

  worker:
    build: ../services/worker
    # multi-stage Go build — final image is ~15MB

    environment:
      API_URL: http://api:8080
      # Docker DNS resolves "api" → api container IP
      # worker pings this URL every 10 seconds

    depends_on:
      api:
        condition: service_healthy
        # wait for api healthcheck to pass before starting worker

    restart: unless-stopped

    networks:
      - backend
      # worker only needs to reach api — backend network is enough
      # no exposed ports — worker calls OUT, nothing calls IN

  frontend:
    build: ../services/frontend
    # builds from services/frontend/Dockerfile
    # nginx with custom config + store HTML

    ports:
      - "80:80"
      # host port 80 → container port 80
      # makes store UI accessible at YOUR_EC2_IP

    depends_on:
      api:
        condition: service_healthy
        # wait for api before starting frontend

    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1/nginx-health || exit 1"]
      # checks the /nginx-health endpoint defined in nginx.conf
      interval: 10s
      timeout: 5s
      retries: 3

    restart: unless-stopped

    networks:
      - web
      # frontend ONLY on web network
      # cannot reach db directly — intentional isolation

  adminer:
    image: adminer:4
    # pre-built DB browser UI — no Dockerfile needed

    ports:
      - "8081:8080"
      # host 8081 → container 8080
      # Adminer listens on 8080 internally

    depends_on:
      - db
      # just needs db to be started (no healthcheck condition)

    restart: unless-stopped

    networks:
      - backend
      # adminer connects to db by name — Docker DNS resolves "db"

networks:
  web:      # Compose creates this automatically on docker compose up
  backend:  # Compose creates this automatically on docker compose up

volumes:
  db-data:  # Compose creates this automatically on docker compose up
            # full name on host: infra_db-data
```

</details>

---

### 📄 services/api/Dockerfile — Python API

**What it does:** packages FastAPI into a runnable image. Layer order matters — deps before code.

<details>
<summary>📄 Show full file — services/api/Dockerfile</summary>

```dockerfile
FROM python:3.12-slim
# slim = no dev tools, smaller image

WORKDIR /app
# all commands run from /app

COPY requirements.txt .
# copy deps manifest FIRST — stable, cached until this file changes

RUN pip install --no-cache-dir -r requirements.txt
# installs: fastapi, uvicorn, asyncpg, pydantic
# cached until requirements.txt changes

COPY src/ .
# copy app code LAST — changes most often, only busts this layer

EXPOSE 8080
# documentation only

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
# start the FastAPI app
```

</details>

---

### 📄 services/frontend/Dockerfile — Nginx

**What it does:** packages Nginx with ShopStack config and static store UI.

<details>
<summary>📄 Show full file — services/frontend/Dockerfile</summary>

```dockerfile
FROM nginx:1.24-alpine
# tiny Nginx on Alpine (~50MB total)

RUN rm /etc/nginx/conf.d/default.conf
# remove generic nginx welcome page config

COPY nginx.conf /etc/nginx/conf.d/default.conf
# replace with ShopStack config (proxy + static files)

COPY html/ /usr/share/nginx/html/
# copy the store UI into nginx's serving directory

EXPOSE 80
# nginx listens on port 80
```

</details>

---

### 📄 services/frontend/nginx.conf — Proxy Config

**What it does:** serves static files + proxies `/api/*` to the Python API. This is the glue between frontend and backend.

<details>
<summary>📄 Show full file — services/frontend/nginx.conf</summary>

```nginx
server {
    listen 80;
    server_name _;

    access_log /var/log/nginx/access.log combined;
    error_log  /var/log/nginx/error.log warn;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
        # serve static files — index.html is the store UI
    }

    location /api/ {
        proxy_pass         http://api:8080;
        # "api" = Docker DNS → resolves to api container IP
        # ALL /api/* requests forwarded to Python API

        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;

        proxy_connect_timeout 5s;
        proxy_read_timeout    10s;  # 504 if API doesn't respond in 10s
        proxy_send_timeout    10s;
    }

    location /nginx-health {
        access_log off;
        return 200 "ok\n";
        add_header Content-Type text/plain;
        # used by Docker healthcheck and K8s liveness probe
    }
}
```

</details>

---

### 📄 services/worker/Dockerfile — Go Multi-Stage

**What it does:** compiles Go binary in Stage 1 (250MB), copies only the binary to Stage 2 (~15MB). Build tools never make it into the final image.

<details>
<summary>📄 Show full file — services/worker/Dockerfile</summary>

```dockerfile
# Stage 1: builder — full Go toolchain
FROM golang:1.22-alpine AS builder
WORKDIR /build
COPY go.mod .
RUN go mod download     # cached until go.mod changes
COPY . .
ARG TARGETARCH          # set automatically by Docker BuildKit
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build -o worker .
# compiles to a single static binary

# Stage 2: runner — tiny Alpine image
FROM alpine:3.19
RUN apk --no-cache add ca-certificates   # SSL support
WORKDIR /app
COPY --from=builder /build/worker .      # ONLY the binary — nothing else
CMD ["./worker"]
```

</details>

---

### 📄 db/init.sql — Database Seed

**What it does:** runs on first Postgres start (empty volume). Creates two schemas and seeds 6 products.

<details>
<summary>📄 Show full file — db/init.sql</summary>

```sql
-- Two schemas — microservice data ownership pattern
CREATE SCHEMA IF NOT EXISTS inventory;  -- products live here
CREATE SCHEMA IF NOT EXISTS orders;     -- orders live here

CREATE TABLE IF NOT EXISTS inventory.products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(200)   NOT NULL,
    category    VARCHAR(100)   NOT NULL,   -- book, course, gear
    price       NUMERIC(10,2)  NOT NULL,
    stock       INTEGER        NOT NULL DEFAULT 0,
    description TEXT,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders.orders (
    id          SERIAL PRIMARY KEY,
    product_id  INTEGER        NOT NULL,
    quantity    INTEGER        NOT NULL DEFAULT 1,
    total       NUMERIC(10,2)  NOT NULL,
    status      VARCHAR(50)    NOT NULL DEFAULT 'confirmed',
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

-- 6 seed products
INSERT INTO inventory.products (name, category, price, stock, description) VALUES
('The Linux Command Line',         'book',   39.99, 150, 'William Shotts.'),
('AWS Solutions Architect Course', 'course', 89.99, 999, 'Hands-on AWS.'),
('Mechanical Keyboard TKL',        'gear',  129.99,  34, 'Brown switches.'),
('Kubernetes in Action',           'book',   49.99,  85, 'Marko Luksa.'),
('Docker & Kubernetes Bootcamp',   'course', 74.99, 999, 'Build and ship.'),
('USB-C Hub — 7 Port',             'gear',   59.99,  60, 'HDMI + Ethernet.');

-- One seed order so /api/orders returns data on first boot
INSERT INTO orders.orders (product_id, quantity, total, status) VALUES
(1, 1, 39.99, 'confirmed');
```

</details>

---

### 📄 services/worker/main.go — The Worker

**What it does:** pings `/api/health` every 10 seconds and writes structured JSON logs. Demonstrates microservice background processing pattern.

<details>
<summary>📄 Show full file — services/worker/main.go</summary>

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "time"
)

// LogEntry — structured JSON log format
// Same shape as Python API logs — both feed into Loki in Observability session
type LogEntry struct {
    TS      string `json:"ts"`
    Level   string `json:"level"`
    Service string `json:"service"`
    Event   string `json:"event"`
    Detail  string `json:"detail,omitempty"`
    Status  int    `json:"status,omitempty"`
    LatMS   int64  `json:"latency_ms,omitempty"`
}

func emit(level, event, detail string, status int, latMS int64) {
    entry := LogEntry{
        TS:      time.Now().UTC().Format(time.RFC3339),
        Level:   level,
        Service: "worker",  // identifies this service in log aggregation
        Event:   event,
        Detail:  detail,
        Status:  status,
        LatMS:   latMS,
    }
    line, _ := json.Marshal(entry)
    fmt.Println(string(line))  // stdout → Docker collects this as container logs
}

func ping(apiURL string) {
    start := time.Now()
    resp, err := http.Get(apiURL + "/api/health")
    latMS := time.Since(start).Milliseconds()

    if err != nil {
        emit("warn", "health_ping_failed", err.Error(), 0, latMS)
        return
    }
    defer resp.Body.Close()

    if resp.StatusCode == 200 {
        emit("info", "health_ping_ok", "api is healthy", resp.StatusCode, latMS)
    } else {
        emit("warn", "health_ping_degraded", "api returned non-200", resp.StatusCode, latMS)
    }
}

func main() {
    apiURL := os.Getenv("API_URL")  // set to http://api:8080 in docker-compose.yml
    if apiURL == "" {
        apiURL = "http://api:8080"  // default fallback
    }

    interval := 10 * time.Second
    emit("info", "worker_started", fmt.Sprintf("pinging %s every %s", apiURL, interval), 0, 0)

    ping(apiURL)  // ping immediately on startup

    ticker := time.NewTicker(interval)
    defer ticker.Stop()

    for range ticker.C {
        ping(apiURL)  // then ping every 10 seconds forever
    }
}
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing to set up
## All files already exist in the shopstack repo.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Start the full ShopStack stack

```bash
cd ~/shopstack/infra
docker compose up --build -d
```

`--build` = rebuild images from Dockerfiles before starting
`-d` = detached, runs in background

---

### Step 2 — Watch startup order

```bash
docker compose logs -f
```

Expected startup sequence:
1. `db` starts → healthcheck runs every 5s → passes after ~10s
2. `api` starts → healthcheck runs → passes after ~15s
3. `worker`, `frontend`, `adminer` start

Press `Ctrl+C` to stop following logs. Containers keep running.

---

### Step 3 — Verify all 5 services are running

```bash
docker compose ps
```

Expected — all 5 with STATUS `running` or `healthy`:
```
NAME              STATUS
infra-db-1        running (healthy)
infra-api-1       running (healthy)
infra-worker-1    running
infra-frontend-1  running (healthy)
infra-adminer-1   running
```

---

### Step 4 — Get your EC2 IP and test

```bash
EC2_IP=$(curl -s http://checkip.amazonaws.com)
echo $EC2_IP
```

```bash
# Test API health
curl http://localhost:8080/api/health

# Test products (should return 6 products from init.sql)
curl http://localhost:8080/api/products

# Test orders (should return 1 seed order)
curl http://localhost:8080/api/orders

# Test metrics (Prometheus format)
curl http://localhost:8080/api/metrics
```

Open in browser:
- `http://YOUR_EC2_IP` → ShopStack store UI
- `http://YOUR_EC2_IP:8081` → Adminer DB browser

Adminer login:
| Field | Value |
|---|---|
| System | PostgreSQL |
| Server | db |
| Username | shopstack |
| Password | shopstack_dev |
| Database | shopstack |

---

### Step 5 — Verify worker is pinging

```bash
docker compose logs worker | tail -10
```

Expected: JSON lines like:
```json
{"ts":"...","level":"info","service":"worker","event":"health_ping_ok","status":200,"latency_ms":2}
```

---

### Step 6 — Verify network isolation

```bash
# frontend should NOT reach db (different networks)
docker compose exec frontend ping -c 1 db 2>&1 | head -2

# api should reach db (same backend network)
docker compose exec api ping -c 2 db
```

Expected: frontend → db fails. api → db succeeds.

---

### Step 7 — Safe stop

```bash
docker compose down
docker volume ls | grep infra_db-data
```

Expected: volume still exists — data is safe.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Compose file exists
ls ~/shopstack/infra/docker-compose.yml

# 2. Start ShopStack
cd ~/shopstack/infra && docker compose up --build -d
sleep 30
docker compose ps

# 3. API health responds
curl -s http://localhost:8080/api/health | grep '"status":"ok"'

# 4. Products endpoint works
curl -s http://localhost:8080/api/products | head -c 100

# 5. All 5 containers running
docker compose ps | grep -c "running\|healthy"

# 6. Networks created
docker network ls | grep -E "infra_web|infra_backend"

# 7. Volume created
docker volume ls | grep infra_db-data

# 8. Worker pinging
docker compose logs worker | tail -3

# 9. Safe stop — volume survives
docker compose down
docker volume ls | grep infra_db-data
```

Expected for #3: `"status":"ok"` in response
Expected for #5: `5`
Expected for #9: volume still listed

**All 9 green?** → Save file → push → Docker complete → Kubernetes Session 01.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| API crashes — DB connection refused | `depends_on` missing or DB not healthy | Add `condition: service_healthy` |
| Port already allocated | Another container owns port 8080 | `docker ps` → stop it → retry |
| Changes not taking effect | Old containers running with old config | `docker compose down` then `up` |
| 502 Bad Gateway | API not running or nginx proxy_pass wrong | `docker compose logs api` |
| Data gone | Used `down -v` | Never use `-v` in production |
| Frontend can reach db | Both on same network | Check network design — frontend → web only |

---

## Quick Reference

| What | Command |
|---|---|
| Start stack | `cd ~/shopstack/infra && docker compose up -d` |
| Start + rebuild | `docker compose up --build -d` |
| Stop — keep data | `docker compose down` |
| Stop — wipe data ⚠️ | `docker compose down -v` |
| Check status | `docker compose ps` |
| All logs | `docker compose logs -f` |
| One service logs | `docker compose logs -f SERVICE` |
| Shell in service | `docker compose exec SERVICE sh` |
| Restart service | `docker compose restart SERVICE` |
| EC2 IP | `curl -s http://checkip.amazonaws.com` |
