# 🐳 DOCKER_CORE — Combat Card
> **80/20 Rule:** Flags → Run → Manage → Debug → Clean. In that order.   
> **Pre-shift scan:** The 5 flags → Compose loop → Cleanup order.   
> **Source:** [04. Docker – Containerization](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/README.md)

---

## 1. THE 5 FLAGS — Memorize These First
> **Source:** [04-docker-port-binding](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/04-docker-port-binding/README.md)   
> Every `docker run` command is built from combinations of these.

| Flag | What it does | Example |
|---|---|---|
| `-d` | Detached — run in background, return terminal | `docker run -d nginx:1.24` |
| `-p HOST:CONTAINER` | Port binding — expose container port to host | `-p 8080:8080` |
| `-v HOST:CONTAINER` | Volume mount — persist data outside container | `-v webstore-db-data:/var/lib/postgresql/data` |
| `-e KEY=VALUE` | Environment variable — configure the app at startup | `-e POSTGRES_PASSWORD=secret` |
| `--name NAME` | Give the container a memorable name | `--name webstore-db` |
| `--network NAME` | Attach to a Docker network — enables DNS by name | `--network webstore-network` |
| `--rm` | Auto-delete container when it stops | `docker run --rm alpine echo "hello"` |

**The full ShopStack manual run:**
```bash
docker run -d \
  --name webstore-db \
  --network webstore-network \
  -v webstore-db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=webstore \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
```

**Port binding mental model:**
```
-p 8080:8080
   ↑     ↑
   │     └── Container port (what the app listens on INSIDE)
   └──────── Host port (what YOUR browser connects to)

webstore-frontend  →  -p 80:80      (browser hits port 80)
webstore-api       →  -p 8080:8080  (browser hits port 8080)
webstore-db        →  NO -p flag    (databases are NEVER public)
```

---

## 2. IMAGES — Pull and Inspect
> **Source:** [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers/README.md)   

| Command | What it does |
|---|---|
| `docker pull IMAGE:TAG` | Download a specific image version |
| `docker images` | List all images on this machine |
| `docker rmi IMAGE:TAG` | Delete an image |
| `docker image prune` | Delete all dangling (untagged) images |

---

## 3. CONTAINERS — Run and Manage
> **Source:** [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers/README.md)   

| Command | What it does |
|---|---|
| `docker run -d --name NAME -p H:C IMAGE` | Full background service run |
| `docker run -it IMAGE /bin/sh` | Interactive — enter the shell |
| `docker ps` | Show running containers only |
| `docker ps -a` | Show ALL containers including stopped |
| `docker stop NAME` | Stop gracefully |
| `docker start NAME` | Start a stopped container |
| `docker restart NAME` | Stop and start without rebuilding |
| `docker rm NAME` | Delete a stopped container |

**ShopStack reference:**
```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
# Expected when healthy:
# webstore-frontend   Up 2 hours   0.0.0.0:80->80/tcp
# webstore-api        Up 2 hours   0.0.0.0:8080->8080/tcp
# webstore-db         Up 2 hours   (no port — internal only)
```

---

## 4. DEBUG — Observe Before You Rebuild
> **Source:** [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers/README.md)   
> **Operator mindset:** observe → inspect → intervene → restart.
> Never rebuild first. Rebuilding hides the root cause.

| Command | What it does |
|---|---|
| `docker logs NAME` | All logs from this container |
| `docker logs -f NAME` | Follow logs live — Ctrl+C to stop |
| `docker logs --tail 50 NAME` | Last 50 lines only |
| `docker inspect NAME` | Full container truth — config, env, ports, IPs |
| `docker inspect NAME \| grep -A 5 "Ports"` | Just the port binding detail |
| `docker inspect NAME \| grep "IPAddress"` | Container's internal IP |
| `docker exec -it NAME /bin/sh` | Enter a running container — always works |
| `docker exec -it NAME /bin/bash` | Enter — only for ubuntu/debian images |
| `docker exec NAME COMMAND` | Run one command inside without entering |

**When to use what:**
```
Container exited?           → docker logs NAME
Container misbehaving?      → docker logs -f NAME (watch live)
Forgot how it was started?  → docker inspect NAME
Need to look inside?        → docker exec -it NAME /bin/sh
Config changed, app stuck?  → docker restart NAME
```

---

## 5. DOCKER COMPOSE — The Daily Driver
> **Source:** [10-docker-compose](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/10-docker-compose/README.md)   
> Compose replaces multiple `docker run` commands with one file.
> Everything you know about flags still applies — Compose just automates them.

| Command | What it does |
|---|---|
| `docker compose up -d` | Start all services in background |
| `docker compose down` | Stop + remove containers and network (volumes survive) |
| `docker compose down -v` | ⚠️ Stop + remove EVERYTHING including volumes — data gone |
| `docker compose ps` | List all containers for this Compose file |
| `docker compose logs <svc>` | Logs for one specific service |
| `docker compose logs -f <svc>` | Follow live logs for a service |
| `docker compose exec <svc> <cmd>` | Run a command inside a running service |
| `docker compose build` | Rebuild images for services using `build:` |
| `docker compose restart <svc>` | Restart one service without touching others |

**ShopStack daily loop:**
```bash
cd ~/shopstack/infra/

docker compose up -d                    # start everything
docker compose ps                       # confirm all running
docker compose logs -f webstore-api     # watch API logs

# After making changes to docker-compose.yml:
docker compose down
docker compose up -d

# End of session:
docker compose down                     # stop everything (data survives)
```

**`down` vs `down -v` — NEVER mix these up:**
```
docker compose down      → containers gone, volumes SAFE  ✅
docker compose down -v   → containers gone, volumes GONE  ⚠️ data wiped
```

---

## 6. NETWORKS AND VOLUMES — The Two Persistence Tools
> **Source:** [05-docker-networking](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/05-docker-networking/README.md)   

| Command | What it does |
|---|---|
| `docker network create <name>` | Create a custom network |
| `docker network ls` | List all networks |
| `docker network inspect <name>` | See all containers on a network + their IPs |
| `docker volume create <name>` | Create a named volume |
| `docker volume ls` | List all volumes |
| `docker volume inspect <name>` | See where Docker stores the data on the host |
| `docker volume rm <name>` | Delete a volume (must have no containers using it) |

**ShopStack network rule:** All three services must be on `webstore-network` so they can reach each other by container name. `webstore-db` has no `-p` flag — only reachable from inside the network.

---

## 7. CLEANUP — The Safe Delete Order
> **Source:** [06-docker-volumes](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/06-docker-volumes/README.md)   
> Docker blocks image deletion if any container still references it — even stopped ones.
> **Always: stop → rm → rmi**

```bash
docker stop webstore-db          # 1. stop the container
docker rm webstore-db            # 2. delete the container
docker rmi postgres:15           # 3. delete the image

# Nuclear option — wipe everything unused
docker system prune -a           # removes stopped containers, unused images, networks
docker system prune -a --volumes # same + volumes (ALL DATA GONE)
```

| Command | What it does |
|---|---|
| `docker stop NAME` | Graceful stop |
| `docker rm NAME` | Delete stopped container |
| `docker rmi IMAGE:TAG` | Delete image |
| `docker container prune` | Delete all stopped containers |
| `docker image prune` | Delete all dangling images |
| `docker system prune -a` | Delete everything unused (keep running containers) |

---

## ⚡ MASTER ONE-LINERS

```bash
# ShopStack — start everything
cd ~/shopstack/infra && docker compose up -d && docker compose ps

# Service not starting — read the crash
docker compose logs webstore-api

# Enter running container to debug
docker exec -it webstore-db psql -U admin -d webstore

# Port already in use?
sudo ss -tlnp | grep :8080

# Full nuclear cleanup (keep images)
docker compose down && docker container prune -f

# Full nuclear cleanup (remove everything)
docker compose down -v && docker system prune -a
```

---

> **Own this page. The rest is details.**
