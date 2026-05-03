# Docker Week 1 — Command Reference (Test Prep)
### Days 1 through 7 — ShopStack Anchored

---

## How to use this file

**🧠 BRAIN** — Type it blind. No looking. These are what the Day 7 test asks you to recall cold.

**📖 LOOKUP** — Know *when* to reach for it and *why* it exists. Exact flags → come back here.

> Your combat sheet in `bag-of-knowledge/docker.md Section 3` is your daily driver.
> This file is for test prep only — cover the right column in the Master Table and produce commands cold.

---

## The 4 Verbs That Run Everything

```bash
docker pull <image>:<tag>        # download image from registry — never runs anything
docker run <flags> <image>       # create and start a container from an image
docker exec <flags> <name> <cmd> # run a command inside an already-running container
docker build -t <name>:<tag> .   # build an image from a Dockerfile in current directory
```

---

---

# DAY 1 — Containers

**The soul of Day 1:** An image is frozen. A container is running. Stopping is not deleting. One image → many containers.

---

### 🧠 BRAIN

```bash
docker pull ubuntu:22.04                       # download image — nothing runs yet
docker images                                  # list every image on this machine — read all columns
docker run -it --name ubuntu-test ubuntu:22.04 # enter container interactively — -i keeps stdin open, -t gives terminal
docker run -d --name webstore-frontend -p 8090:80 nginx:1.24
                                               # run as background service — -d detached, -p host:container port binding
docker ps                                      # running containers only
docker ps -a                                   # all containers including stopped — stopped ≠ deleted
docker start -i ubuntu-test                    # restart a stopped container and enter it — file you wrote is still there
docker logs webstore-frontend                  # everything the container printed to stdout
docker logs -f webstore-frontend               # follow logs live — Ctrl+C to stop
docker exec -it webstore-frontend /bin/sh      # enter a running container — use /bin/sh not /bin/bash on alpine images
docker stop webstore-frontend                  # stop gracefully — container still exists, use docker ps -a to see it
docker rm webstore-frontend                    # delete a stopped container — cannot delete running containers
docker rmi nginx:1.24                          # delete an image — must remove all containers referencing it first
```

---

### 📖 LOOKUP

```bash
docker inspect webstore-frontend               # full JSON — ports, IP, env vars, image SHA — when you need any config detail
docker inspect webstore-frontend | grep Image  # when you need just the image version running
docker system prune                            # when Docker is eating disk — removes stopped containers, unused images, dangling volumes
docker --version                               # confirm Docker is installed — first thing on a new machine
```

---

### The Safe Delete Order — Burn This In

```bash
# Always in this order — Docker blocks image deletion if any container still references it
docker stop <name>    # 1 — stop the container
docker rm <name>      # 2 — delete the container
docker rmi <image>    # 3 — delete the image
```

---

---

# DAY 2 — Port Binding & Networking

**The soul of Day 2:** `-p` maps host port to container port. `localhost` inside a container means itself — not the host. Containers on the same custom network find each other by name.

---

### 🧠 BRAIN

```bash
docker run -d --name webstore-api -p 8080:8080 shopstack-api:1.0
                                               # bind host port 8080 to container port 8080
                                               # browser hits EC2:8080 → forwards to container:8080
docker network create webstore-network         # create a custom bridge network — Docker DNS only works on custom networks
docker run -d --name webstore-db \
  --network webstore-network \
  -e POSTGRES_DB=webstore \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  postgres:15                                  # run Postgres on custom network with required env vars
docker network ls                              # list all Docker networks — confirm web and backend exist
```

---

### 📖 LOOKUP

```bash
docker network inspect infra_backend           # when a container can't reach another — see who is on the network
docker network rm webstore-network             # clean up a network after removing all containers from it
docker run -d --name adminer \
  --network webstore-network \
  -p 8081:8080 adminer                         # adminer needs same network as db to resolve 'webstore-db' by name
```

---

### Port Binding Rule

```bash
# -p HOST:CONTAINER
# HOST    = port on your EC2 machine — what the browser hits
# CONTAINER = port the app listens on inside the container
# Example: -p 8080:8080 — browser → EC2:8080 → container:8080

# localhost INSIDE a container = the container itself — not the host
# DB_HOST=localhost  ❌ — api cannot reach db this way
# DB_HOST=webstore-db ✅ — Docker DNS on custom network resolves by service name
```

---

---

# DAY 3 — Volumes

**The soul of Day 3:** Without a volume Postgres data dies when the container is deleted. With a named volume data survives every container restart and replacement.

---

### 🧠 BRAIN

```bash
docker run -d --name webstore-db \
  --network webstore-network \
  -e POSTGRES_DB=webstore \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  -v webstore-db-data:/var/lib/postgresql/data \
  postgres:15                                  # -v name:path — named volume mounted at Postgres data path
docker volume ls                               # list all volumes — confirm webstore-db-data exists after container deleted
docker volume rm webstore-db-data              # delete a volume — all data inside is gone permanently
```

---

### 📖 LOOKUP

```bash
docker volume inspect webstore-db-data         # when you need to find where Docker actually stores the volume on the host
# Bind mount syntax — development only
docker run -v /host/path:/container/path image  # maps exact host directory into container — use for live code editing not data
```

---

### Volume Rule

```bash
# Named volume:  -v volume-name:/container/path
#   → Docker manages the storage location
#   → data survives docker stop, docker rm
#   → data dies ONLY on docker volume rm or docker compose down -v
#   → use for: Postgres data, anything critical

# Bind mount:   -v /host/absolute/path:/container/path
#   → you control the exact host path
#   → use for: development — edit code on Mac, see it instantly in container
#   → not for production data
```

---

---

# DAY 4 — Dockerfiles & Layers

**The soul of Day 4:** Layer order determines build speed. Stable things first, volatile things last. Source code changes every commit — it goes last so expensive install steps stay cached.

---

### 🧠 BRAIN

```bash
docker build -t shopstack-api:1.0 .            # build image from Dockerfile in current directory — . is build context
docker build --no-cache -t shopstack-api:1.0 . # force every layer to rebuild — use when cache is stale or wrong
docker history shopstack-api:1.0               # inspect every layer — size, command that created it — use to diagnose bloat
docker run --rm shopstack-api:1.0              # run and auto-delete container when it exits — use for one-off testing
```

---

### 📖 LOOKUP

```bash
docker build -f services/api/Dockerfile -t shopstack-api:1.0 .
                                               # when Dockerfile is not in current directory — -f specifies the path
docker image inspect shopstack-api:1.0         # full metadata — entrypoint, CMD, env vars, all layers
```

---

### Layer Order Rule — The One That Matters Most

```bash
# WRONG — source code before dependencies
COPY . .                    # ← changes every commit
RUN pip install -r req.txt  # ← cache busted every time — slow build

# CORRECT — stable things first
COPY requirements.txt .     # ← only changes when deps change
RUN pip install -r req.txt  # ← cached until requirements.txt changes
COPY . .                    # ← source code last — only this layer rebuilds on code change
```

### EXPOSE Rule

```bash
# EXPOSE does nothing at runtime
# It is documentation only — tells readers which port the app uses
# Actual port binding happens with -p on docker run
EXPOSE 8080   # ← tells readers the app listens on 8080 — does NOT expose it
docker run -p 8080:8080 ...  # ← THIS is what actually opens the port
```

---

---

# DAY 5 — Registry

**The soul of Day 5:** Build once, push once, pull anywhere. The tag format is `USERNAME/IMAGE:TAG`. What you push here is exactly what Kubernetes pulls in Week 2.

---

### 🧠 BRAIN

```bash
docker login                                           # authenticate to Docker Hub — required before push
docker tag shopstack-api:1.0 akhiltejadoosari/shopstack-api:1.0
                                                       # create a new tag pointing to same image — does not copy or duplicate
docker push akhiltejadoosari/shopstack-api:1.0         # upload image to Docker Hub — pushes only new layers
docker pull akhiltejadoosari/shopstack-api:1.0         # download from Docker Hub — what Kubernetes does automatically
docker logout                                          # remove stored credentials — use on shared machines
```

---

### 📖 LOOKUP

```bash
docker images                                          # verify tag exists locally before pushing
docker rmi akhiltejadoosari/shopstack-api:1.0          # delete local tag — does not affect what is on Docker Hub
```

---

### Tagging Rule

```bash
# Never push :latest to production
# latest is mutable — points to whatever was pushed last
# You cannot roll back latest to latest

# Use semantic version tags for releases
docker tag shopstack-api:1.0 akhiltejadoosari/shopstack-api:1.0   # ✅ pinned — rollback is possible

# Use Git SHA tags for CI builds
docker tag shopstack-api akhiltejadoosari/shopstack-api:a3f92c1   # ✅ traceable to exact commit
```

---

---

# DAY 6 — Compose

**The soul of Day 6:** Compose replaces 5 `docker run` commands with one file. It does not add new concepts — networks, volumes, env vars, port binding are the same. It just does it consistently from a single source of truth.

---

### 🧠 BRAIN

```bash
docker compose up -d                           # start all services in background — reads docker-compose.yml in current dir
docker compose up --build -d                   # rebuild images before starting — use after code changes
docker compose up --build -d api               # rebuild and restart one service only — other services unaffected
docker compose down                            # stop and remove containers and network — volumes survive
docker compose down -v                         # ⚠️ stop and remove everything including volumes — all Postgres data gone
docker compose ps                              # status of all services — shows health check state
docker compose logs -f api                     # follow logs for one service live — Ctrl+C to stop
docker compose logs --tail 50 db              # last 50 lines from db — when you don't want full history
```

---

### 📖 LOOKUP

```bash
docker compose exec api /bin/sh                # enter a running service container — same as docker exec but uses service name
docker compose build                           # rebuild images only — does not start containers
docker compose config                          # validate and print the merged compose file — catch YAML errors before running
```

---

### The depends_on Rule

```bash
# depends_on: controls startup ORDER — not readiness
# db starts before api — but Postgres may not be ready to accept connections yet
# The app still needs retry logic for database connections

# depends_on with condition (correct):
depends_on:
  db:
    condition: service_healthy   # ← waits until db healthcheck passes, not just process starts
```

### down vs down -v

```bash
docker compose down      # containers gone, network gone, volumes SURVIVE — use this 99% of the time
docker compose down -v   # containers gone, network gone, volumes GONE — use only for full reset
                         # ⚠️ all Postgres data deleted permanently — no undo
```

---

---

# The Master Table — Day 7 Test Prep

Cover the right column. Produce the command from the situation alone.

| Situation | Command |
|---|---|
| Download an image | `docker pull <image>:<tag>` |
| List all images on machine | `docker images` |
| Run container interactively | `docker run -it --name <name> <image>` |
| Run container as background service | `docker run -d --name <name> -p host:container <image>` |
| List running containers | `docker ps` |
| List all containers including stopped | `docker ps -a` |
| What the container printed | `docker logs <name>` |
| Follow logs live | `docker logs -f <name>` |
| Enter a running container | `docker exec -it <name> /bin/sh` |
| Stop a container | `docker stop <name>` |
| Delete a stopped container | `docker rm <name>` |
| Delete an image | `docker rmi <image>:<tag>` |
| Full container config | `docker inspect <name>` |
| Create a custom network | `docker network create <name>` |
| List all networks | `docker network ls` |
| See who is on a network | `docker network inspect <name>` |
| Run with a named volume | `docker run -v <vol-name>:/path <image>` |
| List all volumes | `docker volume ls` |
| Delete a volume | `docker volume rm <name>` |
| Build an image | `docker build -t <name>:<tag> .` |
| Build with no cache | `docker build --no-cache -t <name>:<tag> .` |
| Inspect image layers | `docker history <name>:<tag>` |
| Login to Docker Hub | `docker login` |
| Tag image for Docker Hub | `docker tag <image>:<tag> <username>/<image>:<tag>` |
| Push to Docker Hub | `docker push <username>/<image>:<tag>` |
| Pull from Docker Hub | `docker pull <username>/<image>:<tag>` |
| Start full stack | `docker compose up -d` |
| Rebuild one service | `docker compose up --build -d <service>` |
| Stop stack — keep data | `docker compose down` |
| Stop stack — wipe data | `docker compose down -v` |
| Stack service status | `docker compose ps` |
| Follow one service logs | `docker compose logs -f <service>` |
| Free up Docker disk space | `docker system prune` |
