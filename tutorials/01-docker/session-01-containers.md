# Session 01 — Containers

**Goal:** Understand what a container is, pull images from Docker Hub, run ShopStack containers manually one by one, and understand the difference between an image and a container.

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

Before containers: "it works on my machine" was a real problem. The developer's laptop had Node 16. Production had Node 14. The app crashed in production but worked locally. Debugging took days.

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
├── python:3.12-slim       ← base image for API
├── nginx:1.24-alpine      ← base image for frontend
└── golang:1.22-alpine     ← base image for worker

EC2 (your machine)
├── Images (templates — nothing running)
│   ├── postgres:15-alpine
│   └── nginx:1.24-alpine
└── Containers (running instances of images)
    ├── shopstack-db       ← postgres:15-alpine running
    └── shopstack-frontend ← nginx:1.24-alpine running
```

---

## Image vs Container

The most important distinction in Docker. Get this wrong and nothing makes sense.

```
Image    = recipe / blueprint / template
           → exists on disk
           → nothing is running
           → created with: docker build or docker pull

Container = running instance of an image
           → process is running
           → has its own filesystem, network, memory
           → created with: docker run
```

One image → many containers. Like one recipe → many cakes.

```
postgres:15-alpine (image)
    ├── shopstack-db (container 1 — running)
    ├── test-db (container 2 — running)
    └── old-db (container 3 — stopped)
```

---

## Container Lifecycle

```
docker pull    → download image from Docker Hub
docker run     → create container from image + start it
docker stop    → send SIGTERM → container stops gracefully
docker start   → restart a stopped container
docker rm      → delete the container (not the image)
docker rmi     → delete the image
```

Order matters for cleanup:
```
stop container → rm container → rmi image
```
You cannot delete an image if a container (even stopped) is using it.

---

## 🚀 DevOps Work — Pull and Inspect Images

> Understanding images is core DevOps work. You pull, inspect, and manage images constantly.

```bash
# Pull the Postgres image ShopStack uses
docker pull postgres:15-alpine

# See all images on this machine
docker images

# Inspect the image — see its layers, config, entrypoint
docker inspect postgres:15-alpine
```

Expected from `docker images`:
```
REPOSITORY         TAG          IMAGE ID       SIZE
postgres           15-alpine    xxxxxxxxxxxx   ~240MB
```

---

## 🚀 DevOps Work — Run Your First Container

> Running containers manually is how you debug. You'll do this constantly in production.

```bash
# Run postgres as a container
# -d          → detached (background)
# --name      → give it a name you choose
# -e          → environment variable
docker run -d \
  --name shopstack-db \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine

# Verify it's running
docker ps
```

Expected from `docker ps`:
```
CONTAINER ID   IMAGE              COMMAND                  STATUS          NAMES
xxxxxxxxxxxx   postgres:15-alpine "docker-entrypoint.s…"   Up X seconds    shopstack-db
```

---

## 🚀 DevOps Work — Inspect a Running Container

> These are your debugging tools. Know them cold.

```bash
# See logs — what the container is printing
docker logs shopstack-db

# Follow logs live (Ctrl+C to stop)
docker logs -f shopstack-db

# See resource usage — CPU, RAM, network
docker stats shopstack-db

# Get a shell inside the container
docker exec -it shopstack-db sh

# Inside the container — verify postgres is running
psql -U shopstack -d shopstack -c "\l"

# Exit the container shell
exit
```

---

## 🚀 DevOps Work — Container Lifecycle Commands

> You manage container state constantly. Know every command and what it does.

```bash
# Stop the container gracefully
docker stop shopstack-db

# Verify it stopped
docker ps
# expected: empty — no running containers

# See ALL containers including stopped ones
docker ps -a
# expected: shopstack-db shows STATUS = Exited

# Restart the stopped container
docker start shopstack-db
docker ps
# expected: shopstack-db running again

# Stop and delete the container
docker stop shopstack-db
docker rm shopstack-db

# Verify container is gone
docker ps -a
# expected: no shopstack-db

# Delete the image
docker rmi postgres:15-alpine
# expected: Untagged and Deleted lines

# Verify image is gone
docker images
# expected: no postgres image
```

---

## 🧑‍💻 Dev Work — ShopStack Dockerfiles

> The developer wrote these. You don't modify them. You need to know they exist and what they produce.

```bash
# See the API Dockerfile the developer wrote
cat ~/shopstack/services/api/Dockerfile

# See the frontend Dockerfile
cat ~/shopstack/services/frontend/Dockerfile

# See the worker Dockerfile — multi-stage Go build
cat ~/shopstack/services/worker/Dockerfile
```

**What to notice in each:**
- `FROM` — what base image it starts from
- `WORKDIR` — where files live inside the container
- `COPY` — what files go in
- `RUN` — what gets installed
- `CMD` — what starts when the container runs

---

## 🚀 DevOps Work — Build a ShopStack Image

> Building images from Dockerfiles is your job. You run this in pipelines.

```bash
# Build the API image from the developer's Dockerfile
docker build -t shopstack-api:local ./shopstack/services/api

# Verify the image was created
docker images | grep shopstack-api
```

Expected:
```
shopstack-api   local   xxxxxxxxxxxx   ~200MB
```

---

## 🚀 DevOps Work — Run the Built Image

```bash
# Run the API image as a container
docker run -d \
  --name shopstack-api-test \
  -p 8080:8080 \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_NAME=shopstack \
  -e DB_USER=shopstack \
  -e DB_PASS=shopstack_dev \
  shopstack-api:local

# Check it started
docker ps

# Check logs — it will fail to connect to DB (no DB running yet)
docker logs shopstack-api-test
# expected: connection refused errors — normal, no DB is running

# Clean up
docker stop shopstack-api-test
docker rm shopstack-api-test
docker rmi shopstack-api:local
```

> The API fails to connect to DB — that is expected and correct. Each container runs in isolation. They cannot reach each other without networking. That is Docker Session 02.

---

## 🚀 DevOps Work — Key Flags Reference

| Flag | What it does | Example |
|---|---|---|
| `-d` | Run in background (detached) | `docker run -d` |
| `--name` | Give container a name | `--name shopstack-db` |
| `-p HOST:CONTAINER` | Publish port to host | `-p 8080:8080` |
| `-e KEY=VALUE` | Set environment variable | `-e POSTGRES_DB=shopstack` |
| `-v HOST:CONTAINER` | Mount volume | `-v ./data:/var/lib/postgresql/data` |
| `-it` | Interactive terminal | `docker exec -it NAME sh` |
| `--rm` | Delete container when it stops | `docker run --rm` |

---

## ✅ Session Checkpoint — Verify Before Moving On

Run every command. Output must match expected. If not — fix before starting Session 02.

```bash
# 1. Docker is running
sudo systemctl status docker | grep Active
```
Expected: `Active: active (running)`

---

```bash
# 2. Pull an image
docker pull nginx:alpine
docker images | grep nginx
```
Expected: `nginx   alpine   xxxxxxxxxxxx   ~45MB`

---

```bash
# 3. Run a container
docker run -d --name checkpoint-test nginx:alpine
docker ps | grep checkpoint-test
```
Expected: `checkpoint-test` appears with STATUS `Up`

---

```bash
# 4. Logs work
docker logs checkpoint-test
```
Expected: nginx startup logs — no errors

---

```bash
# 5. Exec works
docker exec checkpoint-test nginx -v
```
Expected: `nginx version: nginx/1.x.x`

---

```bash
# 6. Stop and remove
docker stop checkpoint-test
docker rm checkpoint-test
docker rmi nginx:alpine
docker ps -a | grep checkpoint-test
```
Expected: no output — container is gone

---

```bash
# 7. ShopStack API Dockerfile exists
cat ~/shopstack/services/api/Dockerfile | head -3
```
Expected: first line starts with `FROM python:3.12-slim`

---

**All 7 green?** → Save this file to `devops-journey/tutorials/01-docker/session-01-containers.md` → push to GitHub → start Session 02.

**Any red?** → Fix before moving on.

---

## Bugs You Will Hit

**`docker: permission denied`**
```
Got permission denied while trying to connect to the Docker daemon socket
```
Fix:
```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

---

**`docker rm` fails — container still running**
```
Error response from daemon: You cannot remove a running container
```
Fix: stop first, then remove
```bash
docker stop CONTAINER_NAME
docker rm CONTAINER_NAME
```

---

**`docker rmi` fails — container using image**
```
Error response from daemon: conflict: unable to delete — image is being used by stopped container
```
Fix: remove the container first
```bash
docker ps -a  # find the container using it
docker rm CONTAINER_NAME
docker rmi IMAGE_NAME
```

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Permission denied on docker commands | ubuntu not in docker group | `sudo usermod -aG docker ubuntu` then `newgrp docker` |
| Container exits immediately | App crashed on start — check logs | `docker logs CONTAINER_NAME` |
| Cannot remove image | Stopped container still references it | `docker ps -a` → `docker rm` → `docker rmi` |
| Port already in use | Another container or process owns the port | `docker ps` to find it → stop it first |
| Container not found | Wrong name or already removed | `docker ps -a` to see all including stopped |

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
| Stop container | `docker stop NAME` |
| Start container | `docker start NAME` |
| Remove container | `docker rm NAME` |
| Remove image | `docker rmi IMAGE:TAG` |
| Resource usage | `docker stats NAME` |
| Inspect image | `docker inspect IMAGE` |
