[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Docker

**Hire importance at $25–40/hr:** 10/10

**One line:** Everything in DevOps runs in containers. No Docker — no job. This is the most important file in this folder.

📚 Full runbook → [Docker – Containerization](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/README.md)

---

## 0 — Before You Open A Terminal

### What a developer does that makes Docker necessary

A developer writes a Python FastAPI application. It depends on Python 3.11, specific library versions in `requirements.txt`, and environment variables for database credentials. It works perfectly on their MacBook. It breaks on the Linux EC2 because Python 3.9 is installed there and one library version is different.

This is the environment problem. Docker solves it by packaging the application and everything it needs to run — the runtime, the libraries, the config — into one portable unit called an image. Wherever that image runs, the environment is identical.

### The files a developer writes that you will read

A developer writes a `Dockerfile` — instructions for building the image. They write a `requirements.txt` or `go.mod` or `package.json` — the list of dependencies the application needs. They write environment variable names in the code — `os.getenv("POSTGRES_DB")` — and expect you to supply the values at runtime.

```
shopstack/services/api/Dockerfile          ← you read this to understand the image
shopstack/services/api/requirements.txt    ← dependencies Docker must install
shopstack/services/api/src/main.py         ← where env vars are read from
shopstack/services/worker/go.mod           ← Go dependencies
shopstack/services/frontend/nginx.conf     ← Nginx proxy rules
shopstack/infra/docker-compose.yml         ← the whole system in one file
```

### The contract between developer and infrastructure

The developer writes code that reads `os.getenv("POSTGRES_DB")`. They expect someone — you — to inject that value when the container starts. If you forget it, the app crashes. Not a code bug. An infrastructure gap.

This is why `-e POSTGRES_DB=shopstack` in docker-compose exists. You are fulfilling the contract the developer wrote in the code.

### The multi-stage build aha

A developer's `requirements.txt` lists 20 libraries. Installing them requires `pip` and build tools — heavy, only needed during installation. The running app only needs the installed libraries themselves — not pip, not build tools.

Stage 1 — builder: bring in pip and build tools, install everything from `requirements.txt`.
Stage 2 — runtime: copy only the installed libraries. Leave everything else behind.

Result: image drops from 800MB to 80MB. Not Docker magic. Developer declared dependencies → Stage 1 fulfilled them → Stage 2 took only what the app needs to run.

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- Image vs container — the single most important distinction in Docker
- A container is just an isolated Linux process — not a VM, not magic
- The lifecycle — pull → run → stop → rm → rmi — and what each step does
- `-d` `-it` `--name` `-p` `-e` `-v` — what each flag does and when to use it
- `docker logs` and `docker exec` — your two primary debugging tools
- Why `localhost` inside a container does not mean the host machine
- Named volume vs bind mount — when data survives and when it dies
- `docker compose down` vs `docker compose down -v` — one is safe, one destroys data
- Dockerfile layer order — stable things first, changing things last — why it matters
- `latest` is dangerous in production — always use specific tags

**When you can explain all ten without hesitation — stop. Move to Kubernetes.**

### What unlocks higher pay

At $55/hr you add: multi-stage builds for image size optimisation, `.dockerignore` to protect secrets, image scanning for vulnerabilities, private registries — ECR and GCR, health checks in Compose and Kubernetes probes, Docker layer cache strategies in CI pipelines, and debugging containers that have no shell.

📚 Deep dive → [Docker Runbook](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/README.md)

---

## 2 — The Bag

### Own These Cold

An image is a read-only package — the app, its runtime, its dependencies, its config — frozen at build time. A container is a running instance of that image. One image can run many containers simultaneously. Stopping a container does not delete it. Deleting a container does not delete the image. This distinction must be cold and instant.

A container is not a VM. It is a Linux process with its own isolated filesystem and network, created using kernel features called namespaces and cgroups. It shares the host kernel. It starts in milliseconds. It uses megabytes of memory. When you understand this, every strange container behaviour makes sense — a crashed process means the container exits, not that a machine went down.

`localhost` inside a container means the container itself — not the EC2 host, not the database, not the frontend. This is the single most common Docker mistake. Every container has its own network namespace. Its own localhost. When the API container needs to reach the database, it uses `db` — the service name — not `localhost`. Docker DNS on a custom network resolves `db` to the database container's IP.

A named volume lives independently of any container. `docker compose down` removes the containers but the volume survives. `docker compose down -v` removes the volume. All Postgres data is gone permanently. This distinction is a production disaster waiting to happen. Know it cold.

Dockerfile layer cache: every instruction creates a layer. If the instruction's input has not changed since the last build, Docker reuses the cached layer instead of rebuilding it. `COPY requirements.txt .` then `RUN pip install` must come before `COPY . .` — source code changes on every commit. If `COPY . .` comes before the install, every code change invalidates the install cache. Every build reinstalls everything. A 3-minute build becomes a 30-second build with correct layer order.

Never use `latest` in production. `latest` is mutable — it points to whatever was pushed most recently. You cannot roll back to `latest`. You cannot know which version is running. Use Git SHA tags for CI builds, semantic version tags for releases.

📚 Deep dive → [Docker Volumes](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/06-docker-volumes/README.md)

📚 Deep dive → [Dockerfile & Build](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/08-docker-build-dockerfile/README.md)

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI.

- Container exits immediately → `docker logs` to read the exit reason
- Container running but not responding → `docker exec` to get inside and investigate
- Port not reachable from browser → check `-p` binding and AWS Security Group
- Container cannot reach database by hostname → both must be on the same custom network
- Data gone after container recreate → volume was not attached or `-v` was used on down
- Build is slow every time → layer order wrong — changing files are invalidating install cache
- Image is 800MB → no multi-stage build — build tools are baked into the runtime image
- Compose not picking up new code → `--build` flag missing — old image still running

---

### Domain Awareness

- Docker Swarm — Docker's own orchestration — Kubernetes won, Swarm is legacy
- Multi-platform builds — ARM and AMD cross-compilation — useful for Apple Silicon
- Image signing and verification — supply chain security — comes up at $80/hr
- BuildKit advanced features — cache mounts, secret mounts — production CI territory
- Container security scanning — Trivy, Snyk — important, comes up at $55/hr level

---

### The one insight that separates copiers from understanders

A container is just a Linux process.

Not a virtual machine. Not a separate computer. A process. With its own filesystem view and network namespace, created by the Linux kernel using namespaces and cgroups. When you know this, you understand why a container exits when its main process exits, why `localhost` means the container itself, why containers on different networks cannot reach each other, and why Docker needs the host kernel to be Linux.

Everything about Docker becomes mechanical once you know the container is a process.

---

## 3 — Combat Sheet

The combat sheet is not a memorisation list. It is your pressure-proof reference. When something breaks — open this, find the situation, run the command.

---

### Daily Drivers

These are the commands you type every single working day operating ShopStack.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Don't know which containers are running | `docker compose ps` | `cd ~/shopstack/infra && docker compose ps` |
| Need to see what a container is outputting | `docker compose logs -f SERVICE` | `docker compose logs -f api` |
| Need to get inside a running container | `docker exec -it CONTAINER /bin/sh` | `docker exec -it infra-api-1 /bin/sh` |
| Need to start the whole stack | `docker compose up -d` | `cd ~/shopstack/infra && docker compose up -d` |
| Need to stop the stack safely | `docker compose down` | `docker compose down` — volumes survive |

---

### Situational

You recognise the situation. You open this table. You find the command. You close it.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Container exits immediately | `docker compose logs SERVICE` | `docker compose logs api` — read the exit reason |
| Need last 50 lines of logs only | `docker compose logs --tail 50 SERVICE` | `docker compose logs --tail 50 db` |
| Need full container config — ports, env, image | `docker inspect CONTAINER` | `docker inspect infra-db-1` |
| Need CPU and memory usage all containers | `docker stats` | `docker stats` |
| Code changed — need rebuild | `docker compose up --build -d SERVICE` | `docker compose up --build -d api` |
| Need to destroy everything including data | `docker compose down -v` | ⚠️ Deletes db-data volume — all Postgres data gone |
| Need to see all images on this machine | `docker images` | `docker images` |
| Need to see all containers including stopped | `docker ps -a` | `docker ps -a` |
| Need to remove a stopped container | `docker rm CONTAINER` | `docker rm infra-api-1` |
| Need to remove an image | `docker rmi IMAGE` | `docker rmi shopstack-api:1.0` |
| Need to push image to Docker Hub | `docker tag IMAGE USERNAME/IMAGE:TAG` then `docker push` | `docker push akhiltejadoosari/shopstack-api:1.0` |
| Need to pull image from Docker Hub | `docker pull USERNAME/IMAGE:TAG` | `docker pull akhiltejadoosari/shopstack-api:1.0` |
| Need to see all Docker networks | `docker network ls` | Confirm web and backend exist |
| Need disk space — Docker using too much | `docker system prune` | Removes stopped containers, unused images, dangling volumes |
| Need to check what volumes exist | `docker volume ls` | Confirm infra_db-data exists |

---

## 4 — ShopStack Map

| Situation | Command or File | Knowledge Needed |
|---|---|---|
| Start ShopStack | `cd ~/shopstack/infra && docker compose up -d` | Compose, detached mode |
| API crashes on startup | `docker compose logs api` | Container logs, exit reason |
| db-data volume deleted by accident | `docker compose down -v` was run — data is gone, must reseed | Volume lifecycle, `-v` flag danger |
| Check all 5 services are healthy | `docker compose ps` | Compose status, health checks |
| Enter API container to test DB connection | `docker exec -it infra-api-1 /bin/sh` | exec, interactive shell |
| Rebuild API after code change | `docker compose up --build -d api` | `--build` flag, layer cache |
| Frontend image is 800MB | Read `services/frontend/Dockerfile` — missing multi-stage build | Multi-stage, layer optimisation |
| Check which image version is running | `docker inspect infra-api-1 \| grep Image` | inspect, image SHA |
| Worker cannot reach API | `docker network inspect infra_backend` — is worker on backend network | Network isolation, Docker DNS |
| Push API image after build | `docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0` → `docker push` | Tagging, registry workflow |
| Postgres env var missing — API crashes | Check `-e` vars in docker-compose.yml match what `main.py` reads | Developer contract, env vars |
| Port 8080 not reachable from browser | Check `-p 8080:8080` in compose and AWS Security Group | Port binding, host-to-container |

📚 Deep dive → [Docker Compose](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/10-docker-compose/README.md)

---

## 5 — Peace of Mind

> Answer every question out loud before opening the toggle.

> If you stumble — go to the terminal. Not back to the notes.

---

**Q1. What is the difference between an image and a container? Can one image run multiple containers simultaneously?**

<details>
<summary>Answer</summary>

An image is a read-only frozen package — the app, runtime, dependencies, and config — built from a Dockerfile. It never changes when a container runs from it. A container is a running instance of that image — an isolated Linux process with its own filesystem and network.

Yes — one image can run multiple containers simultaneously. They all start from the same frozen image but each has its own writable layer and its own isolated state. Stopping or deleting one container has no effect on the others or on the image.

</details>

---

**Q2. You run `docker compose down`. Is the container deleted? Is the `db-data` volume deleted? What must you run to delete the volume?**

<details>
<summary>Answer</summary>

`docker compose down` removes the containers and the Docker network. The `db-data` volume is not deleted — it survives intentionally. Postgres data is still there.

To delete the volume you must run `docker compose down -v`. The `-v` flag removes all named volumes defined in the compose file. All Postgres data is gone permanently. This is the one flag you never run in production without knowing exactly what you are deleting and confirming there is a backup.

</details>

---

**Q3. Why does `localhost` inside the API container not mean the EC2 host machine?**

<details>
<summary>Answer</summary>

Every container has its own network namespace — its own isolated network stack with its own localhost. When the API container says `localhost`, it means itself. Not the EC2 machine. Not the database. Not the frontend.

This is why `DB_HOST=localhost` breaks the database connection. The database is not inside the API container — it is in its own container, reachable by the service name `db` on the backend Docker network. Docker DNS on a custom network resolves `db` to the database container's current IP automatically.

</details>

---

**Q4. What is the difference between a named volume and a bind mount? When do you use each?**

<details>
<summary>Answer</summary>

A named volume is managed by Docker. Docker controls where it lives on the host. You give it a name and mount it to a path inside the container. Use it for database data — anything critical that must survive container deletion. `db-data` in ShopStack is a named volume because Postgres data must survive every container restart and replacement.

A bind mount maps a specific host directory into the container. You control the exact path. Use it in development — edit code on your Mac, see changes instantly inside the container without rebuilding the image. Not for production data because you are tying the container to a specific host path.

</details>

---

**Q5. The API container starts and immediately exits with no output. Walk me through your diagnosis.**

<details>
<summary>Answer</summary>

First: `docker compose logs api` — the container printed something before it crashed and that output is the diagnosis. Look for: missing environment variables, failed database connection, port already in use, import error.

Second: `docker compose ps` — confirm the container status is Exited and check the exit code. Exit code 1 means the app crashed with an error. Exit code 0 means the main process finished — CMD had nothing to keep it running.

Third: if logs show nothing — `docker inspect infra-api-1` — check the environment variables were injected correctly and the image is the right one.

The answer is always in the logs. Always check them first.

</details>

---

**Q6. Why does Dockerfile layer order matter? What is the correct order for a Python app?**

<details>
<summary>Answer</summary>

Each Dockerfile instruction creates a layer. If the instruction's input has not changed since the last build, Docker reuses the cached layer. If it has changed, that layer and every layer after it rebuilds from scratch.

Source code changes on every commit. If `COPY . .` comes before `RUN pip install -r requirements.txt`, every code change invalidates the install layer. Every build reinstalls all dependencies. A 3-minute build.

Correct order: `COPY requirements.txt .` first, then `RUN pip install -r requirements.txt`, then `COPY . .`. Now code changes only rebuild from `COPY . .` onward. Dependencies stay cached. A 30-second build.

Stable things first. Volatile things last.

</details>

---

**Q7. What is a multi-stage build and why did the ShopStack worker use one?**

<details>
<summary>Answer</summary>

A multi-stage build uses multiple `FROM` instructions in one Dockerfile. The first stage — builder — installs build tools and compiles or prepares the app. The second stage — runtime — copies only the compiled output from the builder. No build tools, no source code, no intermediate files in the final image.

The ShopStack Go worker uses one because Go compilation requires the Go toolchain — hundreds of megabytes. The compiled binary is a single small file. Stage 1 uses `golang:1.22-alpine` to compile the worker binary. Stage 2 uses `alpine:3.19` — tiny — and copies only the compiled binary. Image drops from 400MB to under 20MB.

</details>

---

**Q8. What does `docker exec -it infra-api-1 /bin/sh` do and when do you use it instead of `docker logs`?**

<details>
<summary>Answer</summary>

It opens an interactive shell inside the running API container. You are inside the container's filesystem and network. You can run commands, check environment variables with `env`, test database connectivity with `psql`, inspect files, and make live network calls.

Use `docker logs` first — it is faster and shows the app's output without entering the container. Use `docker exec` when logs do not tell you enough — when you need to actively investigate the container's internal state, test a connection from inside, or check if a file exists with the right content.

</details>

---

**Q9. Why should you never use the `latest` tag in production?**

<details>
<summary>Answer</summary>

`latest` is mutable — it points to whatever image was pushed most recently. In production you need to know exactly which version is running and be able to roll back to a specific version. If you deployed `latest` and something breaks, `latest` now points to the broken version. You cannot roll back because `latest` moved.

Use Git SHA tags for CI builds — `shopstack-api:a3f92c1` — so you always know which commit is running. Use semantic version tags for releases — `shopstack-api:v1.2.0` — so rollback means pulling the previous tag. Both are immutable — they always point to the same image.

</details>

---

**Q10. `docker compose down` vs `docker compose down -v` — what is the difference and what does each one delete?**

<details>
<summary>Answer</summary>

`docker compose down` stops and removes containers, and removes the Docker network created by Compose. Named volumes are left untouched. The `db-data` volume with all Postgres data survives. This is the safe command — you can bring everything back up with `docker compose up -d` and the data is still there.

`docker compose down -v` does everything above plus removes all named volumes defined in the compose file. `db-data` is deleted. All Postgres rows, all order history, all product data — gone permanently. No undo. This command exists for complete resets only — development cleanup, not production.

</details>

---

## 6 — AI Split

### Own this yourself — never outsource

- Image vs container — this is Q1 in every Docker interview, owns it cold
- Why localhost breaks in containers — tested constantly, must come from your brain
- Volume lifecycle — named volume survives `down`, dies on `down -v` — production-critical
- Dockerfile layer order — stable first, volatile last — why and what happens if reversed
- `docker compose down` vs `docker compose down -v` — one is safe, one destroys data
- What a multi-stage build does and why the worker uses one — ShopStack-specific

### Use AI for this — guilt free

- Writing a Dockerfile for a new language or framework you have not used before
- `.dockerignore` file for a specific project type
- Multi-stage build for a specific runtime — Node, Go, Python
- Exact `docker inspect` filter syntax to extract a specific field
- Health check syntax for a specific service in docker-compose.yml

### The right questions to ask AI for Docker

```
"Write a production Dockerfile for a Python FastAPI app.
The app uses requirements.txt.
It must use a multi-stage build.
Final image must use python:3.11-slim.
Port is 8080."

"I have this docker inspect output: [paste].
What environment variables are set on the container
and what network is it connected to?"

"The ShopStack API container exits immediately.
docker compose logs shows: [paste error].
What are the two most likely causes and how do I verify each?"

"Write a .dockerignore for the ShopStack API service.
It is a Python FastAPI app with a .env file,
a requirements.txt, and a src/ directory."
```

📚 Full Docker runbook → [Docker – Containerization](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/README.md)
