# Session 04 — Dockerfiles

**Goal:** Understand what a Dockerfile is, how layers and caching work, build all three ShopStack images and understand every line inside each Dockerfile.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Dockerfiles Exist

A Docker image doesn't appear from nowhere. A Dockerfile defines exactly what goes inside it — which OS, which runtime, which files, which startup command.

**In ShopStack terms:**
- Developer writes the app code and the Dockerfile
- 🚀 You run `docker build` to turn that Dockerfile into an image
- 🚀 You push that image to Docker Hub
- 🚀 That image is what Kubernetes pulls and runs

---

## Dockerfile Instructions

| Instruction | When it runs | What it does |
|---|---|---|
| `FROM` | Build time | Base image to start from |
| `WORKDIR` | Build time | Set working directory inside container |
| `COPY` | Build time | Copy files from host into image |
| `RUN` | Build time | Execute command, result saved as a layer |
| `ARG` | Build time | Build-time variable (e.g. architecture) |
| `EXPOSE` | Documentation only | Documents which port the app uses |
| `CMD` | Runtime | Default command when container starts |

---

## Layer Caching — The Critical Concept

Every instruction creates a layer. Docker hashes each layer. If the hash matches a previous build — layer is reused. If not — it rebuilds that layer AND everything after it.

```
# Wrong order — code change busts pip install every time
COPY src/ .              ← changes every commit
RUN pip install ...      ← rebuilds every time even if deps didn't change

# Right order — stable things first
COPY requirements.txt .  ← changes rarely
RUN pip install ...      ← cached until requirements.txt changes
COPY src/ .              ← only this layer rebuilds on code change
```

**Rule: stable instructions first, volatile instructions last.**

---

## The Files — All Three ShopStack Dockerfiles

### 📄 services/api/Dockerfile — Python FastAPI

**What it does:** packages the API into a runnable Python image.
**Key decision:** `requirements.txt` copied before `src/` — so pip install is cached across code changes.

<details>
<summary>📄 Show full file — services/api/Dockerfile</summary>

```dockerfile
FROM python:3.12-slim
# python:3.12-slim = Python 3.12 on Debian slim
# slim = no dev tools, compilers, or extras — just Python runtime
# smaller image than python:3.12 (~150MB vs ~1GB)

WORKDIR /app
# sets /app as the working directory inside the container
# all COPY, RUN, CMD instructions use this as their starting point

COPY requirements.txt .
# copies only requirements.txt first — NOT the whole src/ folder
# if requirements.txt hasn't changed, the next RUN is cached
# cache trick: separate stable files from volatile files

RUN pip install --no-cache-dir -r requirements.txt
# installs: fastapi, uvicorn, asyncpg, pydantic
# --no-cache-dir = don't store pip download cache inside the image
# this keeps the image smaller
# this layer is CACHED until requirements.txt changes

COPY src/ .
# copies the actual app code into /app
# this is LAST because code changes on every commit
# changing main.py only rebuilds this layer — pip install stays cached

EXPOSE 8080
# documentation only — tells readers which port the app listens on
# does NOT actually open the port

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
# runs when the container starts
# uvicorn = ASGI server that runs FastAPI
# main:app = file main.py, FastAPI instance called app
# 0.0.0.0 = accept connections from any IP (not just localhost)
# 8080 = port the API listens on
```

</details>

---

### 📄 services/frontend/Dockerfile — Nginx

**What it does:** packages Nginx with ShopStack's custom config and static HTML files.
**Key decision:** removes default nginx config first, then copies ShopStack's config.

<details>
<summary>📄 Show full file — services/frontend/Dockerfile</summary>

```dockerfile
FROM nginx:1.24-alpine
# nginx:1.24-alpine = Nginx 1.24 on Alpine Linux
# Alpine = minimal Linux distro (~5MB base)
# Result: tiny image (~50MB total)

RUN rm /etc/nginx/conf.d/default.conf
# removes nginx's default config
# the default config serves a generic "Welcome to nginx" page
# we replace it with our own config in the next line

COPY nginx.conf /etc/nginx/conf.d/default.conf
# copies ShopStack's nginx config into the container
# this config does two things:
#   1. serves static files (index.html) from /usr/share/nginx/html
#   2. proxies /api/* requests to the Python API container

COPY html/ /usr/share/nginx/html/
# copies the store UI (index.html) into nginx's serving directory
# when browser hits port 80, nginx serves this file

EXPOSE 80
# nginx listens on port 80 — documentation only

# No CMD needed — nginx starts automatically as the default CMD
# inherited from the nginx:1.24-alpine base image
```

</details>

---

### 📄 services/frontend/nginx.conf — The Proxy Config

**What it does:** tells Nginx to serve static files AND proxy `/api/*` requests to the Python API. This is the glue between frontend and API.

<details>
<summary>📄 Show full file — services/frontend/nginx.conf</summary>

```nginx
server {
    listen 80;
    server_name _;
    # listen on port 80, accept any hostname

    access_log /var/log/nginx/access.log combined;
    error_log  /var/log/nginx/error.log warn;
    # log every request — visible with: docker compose logs frontend

    root /usr/share/nginx/html;
    index index.html;
    # serve files from this directory
    # index.html is the store UI

    location / {
        try_files $uri $uri/ /index.html;
        # serve static files
        # if file not found, fall back to index.html (SPA behaviour)
    }

    location /api/ {
        proxy_pass         http://api:8080;
        # forward all /api/* requests to the Python API container
        # "api" = Docker DNS — resolves to the api container's IP
        # if you rename the service in docker-compose.yml, change it here too

        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        # pass real client IP to the API so it appears in logs

        proxy_connect_timeout 5s;
        proxy_read_timeout    10s;
        proxy_send_timeout    10s;
        # if API doesn't respond in 10s → nginx returns 504 Gateway Timeout
        # hit /api/stress (3s delay) — you WON'T see 504 (10s is enough)
        # change to 2s and hit /api/stress → you WILL see 504
    }

    location /nginx-health {
        access_log off;
        return 200 "ok\n";
        add_header Content-Type text/plain;
        # health check endpoint for Docker healthcheck and K8s liveness probe
    }
}
```

</details>

---

### 📄 services/worker/Dockerfile — Go Multi-Stage

**What it does:** compiles the Go worker binary in a full Go environment (Stage 1), then copies only the compiled binary into a tiny Alpine image (Stage 2). The build tools never make it into the final image.

**This is the most asked-about Dockerfile pattern in DevOps interviews.**

<details>
<summary>📄 Show full file — services/worker/Dockerfile</summary>

```dockerfile
# ── Stage 1: builder ─────────────────────────────────────────────────────────
FROM golang:1.22-alpine AS builder
# golang:1.22-alpine = full Go compiler (~250MB)
# AS builder = name this stage so Stage 2 can reference it
# This entire stage is DISCARDED from the final image

WORKDIR /build

COPY go.mod .
# copy dependency manifest first (cache trick — same as Python)

RUN go mod download
# download Go dependencies
# cached until go.mod changes

COPY . .
# copy all worker source files

ARG TARGETARCH
# build argument — Docker BuildKit sets this automatically
# amd64 on AWS x86 instances, arm64 on Apple Silicon

RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build -o worker .
# compile the Go binary
# CGO_ENABLED=0 = static binary — no C library dependency
# GOOS=linux    = build for Linux (even if building on Mac)
# GOARCH        = target CPU architecture (set by ARG TARGETARCH)
# -o worker     = output binary named "worker"
# Result: a single static binary (~8MB)

# ── Stage 2: runner ──────────────────────────────────────────────────────────
FROM alpine:3.19
# tiny Alpine base (~5MB) — NO Go compiler, NO source code
# this is the FINAL image that gets pushed to Docker Hub

RUN apk --no-cache add ca-certificates
# add SSL certificates — needed for HTTPS requests
# the /api/health ping uses HTTP but this is future-proofing

WORKDIR /app

COPY --from=builder /build/worker .
# copy ONLY the compiled binary from Stage 1
# nothing else from Stage 1 comes with it
# no Go compiler, no source code, no build tools

CMD ["./worker"]
# start the worker binary when the container runs
```

</details>

**Size comparison:**
```
golang:1.22-alpine (builder stage)  → ~250MB  ← DISCARDED
alpine:3.19 + binary (final image)  → ~15MB   ← what gets pushed
```

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing to set up
## All Dockerfiles already exist in the shopstack repo.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Read all three Dockerfiles

```bash
cat ~/shopstack/services/api/Dockerfile
cat ~/shopstack/services/frontend/Dockerfile
cat ~/shopstack/services/worker/Dockerfile
```

Match each line against the toggle above. Don't move on until every line makes sense.

---

### Step 2 — Build the API image

```bash
docker build -t shopstack-api:local ~/shopstack/services/api
```

Watch the output. Each step is a layer. Notice which ones are cached on rebuild.

```bash
docker images | grep shopstack-api
docker history shopstack-api:local
```

---

### Step 3 — Prove layer caching works

```bash
# Build again — nothing changed
docker build -t shopstack-api:local ~/shopstack/services/api 2>&1 | grep -c CACHED
```

Expected: 4 or more layers cached — instant build.

```bash
# Simulate a code change
touch ~/shopstack/services/api/src/main.py

# Build again
docker build -t shopstack-api:local ~/shopstack/services/api
```

Expected:
- `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install` → CACHED
- `COPY src/` → rebuilt (file timestamp changed)
- pip install was NOT re-run — only the last layer rebuilt

---

### Step 4 — Build the frontend image

```bash
docker build -t shopstack-frontend:local ~/shopstack/services/frontend
docker images | grep shopstack-frontend
docker history shopstack-frontend:local
```

Expected: ~63MB image. Much smaller than the API — Nginx Alpine base is tiny.

---

### Step 5 — Build the worker image (multi-stage)

```bash
docker build -t shopstack-worker:local ~/shopstack/services/worker
docker images | grep shopstack-worker
```

Expected: ~15-25MB — final image contains only the compiled binary.

```bash
# Compare sizes
docker images | grep -E "shopstack-worker|golang"
```

Expected: worker image is ~15MB, golang base would be ~250MB.

---

### Step 6 — Verify all three images exist

```bash
docker images | grep shopstack
```

Expected:
```
shopstack-api        local   xxxxxxxxxxxx   ~288MB
shopstack-frontend   local   xxxxxxxxxxxx   ~63MB
shopstack-worker     local   xxxxxxxxxxxx   ~25MB
```

---

### Step 7 — Clean up

```bash
docker rmi shopstack-api:local shopstack-frontend:local shopstack-worker:local
```

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. API base image correct
head -1 ~/shopstack/services/api/Dockerfile

# 2. Build API
docker build -t checkpoint-api ~/shopstack/services/api
docker images | grep checkpoint-api

# 3. Caching works — second build
docker build -t checkpoint-api ~/shopstack/services/api 2>&1 | grep -c "CACHED"

# 4. Frontend base image correct
head -1 ~/shopstack/services/frontend/Dockerfile

# 5. Build frontend
docker build -t checkpoint-frontend ~/shopstack/services/frontend
docker images | grep checkpoint-frontend

# 6. Worker is multi-stage
grep -c "^FROM" ~/shopstack/services/worker/Dockerfile

# 7. Build worker — verify small size
docker build -t checkpoint-worker ~/shopstack/services/worker
docker images | grep checkpoint-worker

# 8. Clean up
docker rmi checkpoint-api checkpoint-frontend checkpoint-worker
docker images | grep checkpoint
```

Expected for #1: `FROM python:3.12-slim`
Expected for #3: number > 0
Expected for #4: `FROM nginx:1.24-alpine` (or comment line before it)
Expected for #6: `2` — two FROM = multi-stage
Expected for #7: under 30MB

**All 8 green?** → Save file → push → Session 05.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Every build slow | `COPY src/` before `RUN pip install` | Move dep install before code copy |
| `COPY failed: file not found` | Wrong build context path | Check path passed to `docker build` |
| Multi-stage binary missing | Missing `COPY --from=builder` | Add explicit copy from builder stage |
| `exec format error` | Wrong architecture binary | Use `ARG TARGETARCH` in Go builds |

---

## Quick Reference

| What | Command |
|---|---|
| Build image | `docker build -t NAME:TAG PATH` |
| Build no cache | `docker build --no-cache -t NAME:TAG PATH` |
| See layers | `docker history IMAGE` |
| List images | `docker images` |
| Remove image | `docker rmi IMAGE` |
