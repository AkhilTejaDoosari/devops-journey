# Day 3 — Docker Volumes

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md)

**Date:** April 22 2026    
**Read before session:** [06-docker-volumes](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/06-docker-volumes)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Feel data die without a volume. Feel it survive with one.   

---

## Knowledge — What These Topics Are Really About

### The Core Problem
A container's filesystem is temporary. It lives and dies with the container. ShopStack's `infra-db-1` writes all its data to `/var/lib/postgresql/data` — inside the container. Run `docker rm infra-db-1` and every product, every order, every stock level is erased. The next container starts from zero.

```
No Volume:                         With Named Volume:
infra-db-1 → deleted               infra-db-1 → deleted
  inventory.products → GONE ❌       db-data volume → SURVIVES ✅
  orders.orders      → GONE ❌       New infra-db-1 mounts db-data
                                     inventory.products → STILL THERE ✅
                                     orders.orders      → STILL THERE ✅
```

**The core rule:** Data in containers = temporary. Data in volumes = permanent.

### Named Volumes
Docker creates and manages the storage location. You give it a name and mount it to a path inside the container. The volume lives independently — delete and recreate the container as many times as you want, the data does not move.

On EC2, ShopStack's `db-data` volume lives at:
```
/var/lib/docker/volumes/infra_db-data/
```
Docker put it there. You never have to think about that path.

> **The core rule:** Use named volumes for anything that must survive — databases, uploaded files, critical state.

### Bind Mounts
You specify an exact path on your host. That folder is mounted directly into the container. Changes on the host appear instantly inside the container and vice versa.

> **The core rule:** Use bind mounts during development — edit code on your laptop, see changes immediately inside the container.

### Why `/var/lib/postgresql/data`?
That is PostgreSQL's internal data directory — hardcoded by the image. Every row inserted into `inventory.products` or `orders.orders` lives in that path. Mount a named volume there and Docker writes data to the volume instead of the ephemeral container layer.

### ShopStack Connection
`infra-db-1` uses a named volume `db-data` mounted to `/var/lib/postgresql/data`. This is declared in `~/shopstack/infra/docker-compose.yml`. Without it, `docker compose down -v` wipes the entire shopstack database — every product, every order, gone. The volume is what makes the database production-safe.

```
docker compose down      → containers stop, db-data SURVIVES, data SAFE
docker compose down -v   → containers stop, db-data DELETED, data GONE
```

---

# 📟 DAY 3 FULL CHEATSHEET — Docker Volumes

## Volume Management

| What it does | Command | Example |
|---|---|---|
| Create a named volume | `docker volume create <NAME>` | `docker volume create db-data` |
| List all volumes on this host | `docker volume ls` | `docker volume ls` |
| Show driver, mount point, creation time | `docker volume inspect <NAME>` | `docker volume inspect db-data` |
| Delete a volume — container must be removed first | `docker volume rm <NAME>` | `docker volume rm db-data` |
| Delete all volumes not mounted to any container | `docker volume prune` | `docker volume prune` |

---

## Running Containers With Storage

| What it does | Command | Example |
|---|---|---|
| Mount a named volume into a container | `docker run -v <VOLUME_NAME>:<CONTAINER_PATH> <IMAGE>` | `docker run -v db-data:/var/lib/postgresql/data postgres:15` |
| Bind mount a host directory into a container | `docker run -v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH> <IMAGE>` | `docker run -v $(pwd)/host-data:/data ubuntu:22.04` |
| Show exactly what is mounted and where | `docker inspect <NAME> \| grep -A 10 Mounts` | `docker inspect infra-db-1 \| grep -A 10 Mounts` |

**Syntax breakdown — Named Volume:**
```
docker run -v db-data:/var/lib/postgresql/data postgres:15
                ↑                ↑
          volume name      path inside container
          (Docker manages) (where postgres writes ALL ShopStack data)
```

**Syntax breakdown — Bind Mount:**
```
docker run -v $(pwd)/host-data:/data ubuntu:22.04
                ↑                ↑
          absolute path      path inside container
          on your machine    (you control both sides)
```

---

## Entering a Running Container — ShopStack DB

| What it does | Command | Example |
|---|---|---|
| Open PostgreSQL shell inside infra-db-1 | `docker exec -it <NAME> psql -U <USER> -d <DATABASE>` | `docker exec -it infra-db-1 psql -U shopstack -d shopstack` |
| Run one SQL query and exit immediately | `docker exec -it <NAME> psql -U <USER> -d <DATABASE> -c "<SQL>"` | `docker exec -it infra-db-1 psql -U shopstack -d shopstack -c "SELECT * FROM inventory.products;"` |

**Syntax breakdown — psql flags:**
```
docker exec -it infra-db-1 psql -U shopstack -d shopstack
                            ↑        ↑              ↑
                        program    user          database
                    (inside container)
```

---

## Adminer Login — ShopStack Credentials

Adminer is `infra-adminer-1`. It connects to `infra-db-1` over the `backend` network using Docker DNS.

| Field    | Value         |
|----------|---------------|
| System   | PostgreSQL    |
| Server   | db            |
| Username | shopstack     |
| Password | shopstack_dev |
| Database | shopstack     |

> ⚠️ Server is `db` — the Docker service name, not `localhost`. Docker DNS resolves it to `infra-db-1` automatically.


---

## Inside psql — PostgreSQL Shell

> 🧪 **Practice Only — Not a DevOps Skill**   
> As a DevOps engineer you will never manually write SQL in production. These commands exist only to give you visual proof that data persisted — or didn't — inside the volume. The test will walk you through them step by step.

| What it does | Command | Example |
|---|---|---|
| Create a table | `CREATE TABLE <NAME> (id SERIAL, name TEXT);` | `CREATE TABLE products (id SERIAL, name TEXT);` |
| Insert one row | `INSERT INTO <NAME> (name) VALUES ('<VALUE>');` | `INSERT INTO products (name) VALUES ('Widget');` |
| Read all rows | `SELECT * FROM <NAME>;` | `SELECT * FROM products;` |
| Quit psql — return to host terminal | `\q` | `\q` |

> ⚠️ You are not in Docker once inside psql. `\q` is a psql command, not a shell command.

---

## Named Volume vs Bind Mount

| When to use | Type | Syntax | Example |
|---|---|---|---|
| Data must survive — databases, critical state | Named Volume | `-v <VOLUME_NAME>:<CONTAINER_PATH>` | `-v db-data:/var/lib/postgresql/data` |
| Editing files from host — development | Bind Mount | `-v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH>` | `-v $(pwd)/host-data:/data` |

---

## Safe Delete Order

| Step | What it does | Command | Example |
|---|---|---|---|
| 1 | Stop the container | `docker stop <NAME>` | `docker stop infra-db-1` |
| 2 | Remove the container | `docker rm <NAME>` | `docker rm infra-db-1` |
| 3 | Remove the volume *(only if deleting data)* | `docker volume rm <NAME>` | `docker volume rm db-data` |

> ⚠️ `docker rm` does NOT delete the volume. That is intentional. You have to explicitly pull the trigger on data.

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| `volume is in use` on `docker volume rm db-data` | A stopped container still references the volume | `docker ps -a` → find it → `docker rm` it first |
| Adminer can't connect to `infra-db-1` | Forgot `--network backend` on one or both containers | `docker rm` both, rerun with `--network backend` on both |
| Data still gone even with `-v db-data:/var/lib/postgresql/data` | Volume flag syntax wrong — typo or relative path used | `docker inspect infra-db-1 \| grep -A 10 Mounts` — if empty, volume didn't attach |
| Port 8081 already in use | Leftover adminer from Day 2 still running | `docker ps` → `docker rm -f infra-adminer-1` |

---

## Checklist

- ✅ `docker network create backend`
- ✅ `docker run -d --name infra-db-1 --network backend -e POSTGRES_DB=shopstack -e POSTGRES_USER=shopstack -e POSTGRES_PASSWORD=shopstack_dev postgres:15`
- ✅ `docker run -d --name infra-adminer-1 --network backend -p 8081:8080 adminer`

> 🧪 **Practice Only — Not a DevOps Skill**
> As a DevOps engineer you will never manually write SQL or use a database GUI to prove persistence in production. These steps exist only to give you visual proof that data died without a volume and survived with one.

- ✅ Open Adminer — `http://YOUR_EC2_IP:8081` — login with ShopStack credentials above — create a table manually — insert one row
- ✅ `docker stop infra-db-1 infra-adminer-1`
- ✅ `docker rm infra-db-1 infra-adminer-1`
- ✅ Run fresh `infra-db-1` with **no volume** — same command, no `-v` flag
- ✅ `docker run -d --name infra-adminer-1 --network backend -p 8081:8080 adminer`
- ✅ Open Adminer — your table is **GONE** — no volume, data died with the container

> ↩️ **Back to DevOps — resume normal checklist**

- ✅ `docker stop infra-db-1 infra-adminer-1`
- ✅ `docker rm infra-db-1 infra-adminer-1`
- ✅ `docker run -d --name infra-db-1 --network backend -e POSTGRES_DB=shopstack -e POSTGRES_USER=shopstack -e POSTGRES_PASSWORD=shopstack_dev -v db-data:/var/lib/postgresql/data postgres:15`
- ✅ `docker run -d --name infra-adminer-1 --network backend -p 8081:8080 adminer`
- ✅ Open Adminer — insert one row
- ✅ `docker stop infra-db-1 infra-adminer-1`
- ✅ `docker rm infra-db-1 infra-adminer-1`
- ✅ `docker volume ls` — `db-data` still exists
- ✅ Rerun `infra-db-1` with the same `-v db-data:/var/lib/postgresql/data` flag
- ✅ `docker run -d --name infra-adminer-1 --network backend -p 8081:8080 adminer`
- ✅ Open Adminer — your row is **STILL THERE** — volume worked
- ✅ `docker stop infra-db-1 infra-adminer-1`
- ✅ `docker rm infra-db-1 infra-adminer-1`
- ✅ `docker volume inspect db-data` — note the `Mountpoint` path on EC2
- ✅ `docker volume rm db-data`
- ✅ `docker network rm backend`
- ✅ `docker rmi postgres:15 adminer`

**Session win condition:** Data survived container deletion. You felt data die and then fixed it. You can explain why ShopStack uses `db-data`.

---

Ready to test yourself? → [Test](./test.md)
