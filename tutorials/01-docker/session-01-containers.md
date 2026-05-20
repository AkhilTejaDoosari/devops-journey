# Session 01 — Containers

**Goal:** Understand what a container is, pull images, run ShopStack containers manually, and understand the difference between an image and a container.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Containers Exist

Before containers: "it works on my machine" was a real problem. The developer's laptop had Python 3.12. The server had Python 3.9. The app crashed in production but worked locally.

A container packages the app AND its entire environment — the runtime, dependencies, config — into one unit. Same container runs on your laptop, on EC2, on any server. Environment differences disappear.

**In ShopStack terms:**
- The API needs Python 3.12, FastAPI, asyncpg
- The worker needs Go 1.22
- The DB needs Postgres 15
- Each runs in its own container with exactly what it needs — nothing more, nothing less

---

## Visual Map

```
Docker Hub (registry)
├── postgres:15-alpine     ← pre-built image, pulled on demand
├── python:3.12-slim       ← base image the API Dockerfile starts from
├── nginx:1.24-alpine      ← base image the frontend Dockerfile starts from
└── golang:1.22-alpine     ← base image the worker Dockerfile starts from

EC2 (your machine)
├── Images (templates — nothing running yet)
│   └── postgres:15-alpine
└── Containers (running instances of images)
    └── shopstack-db       ← postgres:15-alpine running as a container
```

---

## Image vs Container — The Most Important Distinction

```
Image     = recipe / blueprint / template
            → exists on disk
            → nothing is running
            → created with: docker build or docker pull

Container = running instance of an image
            → a process is running
            → has its own filesystem, network, memory
            → created with: docker run
```

One image → many containers. Like one recipe → many cakes.

---

## Container Lifecycle

```
docker pull    → download image from Docker Hub
docker run     → create container + start it
docker stop    → stop the container gracefully
docker start   → restart a stopped container
docker rm      → delete the container (not the image)
docker rmi     → delete the image

Order for cleanup (always this order):
stop → rm → rmi
You cannot delete an image while a container (even stopped) references it.
```

---

## The Files — What the Developer Built

These files exist in the ShopStack repo. The developer wrote them. You don't modify them. You read them to understand what you're running.

### 📄 services/api/Dockerfile

**What it does:** packages the Python FastAPI app into a runnable image.
**Why this layer order:** `requirements.txt` changes rarely. Source code changes constantly. By copying deps first, pip install is cached — code changes only rebuild the last layer.

<details>
<summary>📄 Show full file — services/api/Dockerfile</summary>

```dockerfile
FROM python:3.12-slim
# base image — slim = no dev tools, smaller final image size

WORKDIR /app
# all subsequent commands run from /app inside the container

COPY requirements.txt .
# copy dependency manifest FIRST — stable, changes rarely
# if this file hasn't changed, the next RUN layer is cached

RUN pip install --no-cache-dir -r requirements.txt
# install all Python dependencies
# --no-cache-dir = don't cache pip downloads inside the image (smaller image)
# cached until requirements.txt changes

COPY src/ .
# copy app source code LAST — changes on every commit
# only this layer rebuilds when you change main.py

EXPOSE 8080
# documentation only — tells readers the app listens on 8080
# does NOT actually open the port (docker run -p does that)

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
# start the FastAPI app when the container runs
# uvicorn = ASGI server for Python async apps
# 0.0.0.0 = accept connections from any IP (not just localhost)
```

</details>

---

### 📄 services/api/requirements.txt

**What it does:** lists every Python library the API needs. Docker installs all of these during `docker build`.

<details>
<summary>📄 Show full file — services/api/requirements.txt</summary>

```
fastapi        # web framework — handles HTTP routes and request/response
uvicorn        # ASGI server — runs the FastAPI app
asyncpg        # async Postgres driver — talks to the database
pydantic       # data validation — validates request bodies
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing to set up
## This session uses Docker which is already installed from Session 00.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Pull an image from Docker Hub

```bash
docker pull postgres:15-alpine
```

Expected:
```
15-alpine: Pulling from library/postgres
...
Status: Downloaded newer image for postgres:15-alpine
```

---

### Step 2 — See what images you have locally

```bash
docker images
```

Expected:
```
REPOSITORY   TAG          IMAGE ID       SIZE
postgres     15-alpine    xxxxxxxxxxxx   ~240MB
```

---

### Step 3 — Run your first container

```bash
docker run -d \
  --name shopstack-db \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine
```

What each flag does:
- `-d` → detached, runs in background
- `--name shopstack-db` → give it a name you choose
- `-e KEY=VALUE` → environment variable injected at startup

---

### Step 4 — Verify it's running

```bash
docker ps
```

Expected:
```
CONTAINER ID   IMAGE              STATUS        NAMES
xxxxxxxxxxxx   postgres:15-alpine Up X seconds  shopstack-db
```

---

### Step 5 — Read the container logs

```bash
docker logs shopstack-db
```

Expected: Postgres startup messages ending with `database system is ready to accept connections`

---

### Step 6 — Get a shell inside the running container

```bash
docker exec -it shopstack-db sh
```

You are now inside the container. Run:
```bash
psql -U shopstack -d shopstack -c "\l"
# lists all databases
exit
```

---

### Step 7 — Stop, remove, clean up

```bash
docker stop shopstack-db
docker ps          # empty — container stopped
docker ps -a       # shows stopped container

docker rm shopstack-db
docker ps -a       # empty — container deleted

docker rmi postgres:15-alpine
docker images      # postgres gone
```

---

### Step 8 — See the API Dockerfile

```bash
cat ~/shopstack/services/api/Dockerfile
```

Expected: matches the file shown in the toggle above.

---

### Step 9 — Build the API image from that Dockerfile

```bash
docker build -t shopstack-api:local ~/shopstack/services/api
docker images | grep shopstack-api
```

Expected: `shopstack-api   local   xxxxxxxxxxxx   ~288MB`

---

### Step 10 — Clean up built image

```bash
docker rmi shopstack-api:local
```

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Docker running
sudo systemctl status docker | grep Active

# 2. Pull works
docker pull nginx:alpine
docker images | grep nginx

# 3. Run container
docker run -d --name checkpoint-test nginx:alpine
docker ps | grep checkpoint-test

# 4. Logs work
docker logs checkpoint-test

# 5. Exec works
docker exec checkpoint-test nginx -v

# 6. Stop and remove
docker stop checkpoint-test
docker rm checkpoint-test
docker rmi nginx:alpine
docker ps -a | grep checkpoint-test

# 7. API Dockerfile exists with correct base
head -1 ~/shopstack/services/api/Dockerfile
```

Expected for #7: `FROM python:3.12-slim`

**All 7 green?** → Save file → push → Session 02.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Permission denied on docker | ubuntu not in docker group | `sudo usermod -aG docker ubuntu && newgrp docker` |
| Container exits immediately | App crashed — check logs | `docker logs CONTAINER` |
| Cannot remove image | Stopped container still references it | `docker ps -a` → `docker rm` → `docker rmi` |
| Port already in use | Another container owns the port | `docker ps` → stop it first |

---

## Quick Reference

| What | Command |
|---|---|
| Pull image | `docker pull IMAGE:TAG` |
| List images | `docker images` |
| Build image | `docker build -t NAME:TAG PATH` |
| Run container | `docker run -d --name NAME IMAGE` |
| List running | `docker ps` |
| List all | `docker ps -a` |
| View logs | `docker logs NAME` |
| Follow logs | `docker logs -f NAME` |
| Shell inside | `docker exec -it NAME sh` |
| Stop | `docker stop NAME` |
| Remove container | `docker rm NAME` |
| Remove image | `docker rmi IMAGE:TAG` |
