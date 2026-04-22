# Day 3 — Docker Volumes

**Date:** April 21 2026
**Read before session:** [06-docker-volumes](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/06-docker-volumes)
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)
**Goal:** Feel data die without a volume. Feel it survive with one.

---

## Knowledge — What These Topics Are Really About

### The Core Problem
A container's filesystem is temporary. It lives and dies with the container. PostgreSQL writes its data files inside the container — so when you `docker rm webstore-db`, every table, every row, every schema is erased.

```
No Volume:                         With Named Volume:
Container → deleted                Container → deleted
Data      → GONE ❌                Volume    → SURVIVES ✅
                                   New Container mounts same volume
                                   Data      → STILL THERE ✅
```

**The core rule:** Data in containers = temporary. Data in volumes = permanent.

### Named Volumes
Docker creates and manages the storage location. You give it a name and mount it to a path inside the container. The volume lives independently — delete and recreate the container as many times as you want, the data does not move.

**The core rule:** Use named volumes for anything that must survive — databases, uploaded files, critical state.

### Bind Mounts
You specify an exact path on your host. That folder is mounted directly into the container. Changes on the host appear instantly inside the container and vice versa.

**The core rule:** Use bind mounts during development — edit code on your laptop, see changes immediately inside the container.

### Why `/var/lib/postgresql/data`?
That is PostgreSQL's internal data directory — hardcoded by the image. Every row you insert lives in that path. Mount a named volume there and Docker writes data to the volume instead of the ephemeral container layer.

### ShopStack Connection
`webstore-db` uses a named volume `webstore-db-data` mounted to `/var/lib/postgresql/data`. Without it, every `docker compose down` would wipe the entire database. The volume is what makes the database production-safe.

---

# 📟 DAY 3 FULL CHEATSHEET — Docker Volumes

## Volume Management

| What it does | Command | Example |
|---|---|---|
| Create a named volume | `docker volume create <NAME>` | `docker volume create webstore-db-data` |
| List all volumes on this host | `docker volume ls` | `docker volume ls` |
| Show driver, mount point, creation time | `docker volume inspect <NAME>` | `docker volume inspect webstore-db-data` |
| Delete a volume — container must be removed first | `docker volume rm <NAME>` | `docker volume rm webstore-db-data` |
| Delete all volumes not mounted to any container | `docker volume prune` | `docker volume prune` |

---

## Running Containers With Storage

| What it does | Command | Example |
|---|---|---|
| Mount a named volume into a container | `docker run -v <VOLUME_NAME>:<CONTAINER_PATH> <IMAGE>` | `docker run -v webstore-db-data:/var/lib/postgresql/data postgres:15` |
| Bind mount a host directory into a container | `docker run -v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH> <IMAGE>` | `docker run -v $(pwd)/host-data:/data ubuntu:22.04` |
| Show exactly what is mounted and where | `docker inspect <NAME> \| grep -A 10 Mounts` | `docker inspect webstore-db \| grep -A 10 Mounts` |

**Syntax breakdown — Named Volume:**
```
docker run -v webstore-db-data:/var/lib/postgresql/data postgres:15
                ↑                        ↑
          volume name            path inside container
          (Docker manages)       (where postgres writes data)
```

**Syntax breakdown — Bind Mount:**
```
docker run -v $(pwd)/host-data:/data ubuntu:22.04
                ↑                ↑
          absolute path      path inside container
          on your machine    (you control both sides)
```

---

## Entering a Running Container

| What it does | Command | Example |
|---|---|---|
| Open PostgreSQL shell inside a container | `docker exec -it <NAME> psql -U <USER> -d <DATABASE>` | `docker exec -it webstore-db psql -U admin -d webstore` |
| Run one SQL query and exit immediately | `docker exec -it <NAME> psql -U <USER> -d <DATABASE> -c "<SQL>"` | `docker exec -it webstore-db psql -U admin -d webstore -c "SELECT * FROM products;"` |

**Syntax breakdown — psql flags:**
```
docker exec -it webstore-db psql -U admin -d webstore
                             ↑      ↑         ↑
                         program  user     database
                      (inside container)
```

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
| Data must survive — databases, critical state | Named Volume | `-v <VOLUME_NAME>:<CONTAINER_PATH>` | `-v webstore-db-data:/var/lib/postgresql/data` |
| Editing files from host — development | Bind Mount | `-v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH>` | `-v $(pwd)/host-data:/data` |

---

## Safe Delete Order

| Step | What it does | Command | Example |
|---|---|---|---|
| 1 | Stop the container | `docker stop <NAME>` | `docker stop webstore-db` |
| 2 | Remove the container | `docker rm <NAME>` | `docker rm webstore-db` |
| 3 | Remove the volume *(only if deleting data)* | `docker volume rm <NAME>` | `docker volume rm webstore-db-data` |

> ⚠️ `docker rm` does NOT delete the volume. That is intentional. You have to explicitly pull the trigger on data.

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| `volume is in use` on `docker volume rm` | A stopped container still references the volume | `docker ps -a` → find it → `docker rm` it first |
| Adminer can't connect to `webstore-db` | Forgot `--network webstore-network` on one container | `docker rm` both, rerun with `--network` on both |
| Data still gone even with volume flag | Volume flag syntax wrong — common typo | `docker inspect webstore-db \| grep Mounts` — if empty, volume didn't attach |
| Port 8081 already in use | Leftover adminer from Day 2 still running | `docker ps` → `docker rm -f adminer` |

---

## Checklist

- [ ] `docker network create webstore-network`
- [ ] `docker run -d --name webstore-db --network webstore-network -e POSTGRES_DB=webstore -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=secret postgres:15`
- [ ] `docker run -d --name adminer --network webstore-network -p 8081:8080 adminer`

> 🧪 **Practice Only — Not a DevOps Skill**
> As a DevOps engineer you will never manually write SQL or use a database GUI to prove persistence in production. These steps exist only to give you visual proof that data died without a volume and survived with one. The test will walk you through them step by step.

- [ ] Open Adminer — login — create a table manually — insert one row
- [ ] `docker stop webstore-db adminer`
- [ ] `docker rm webstore-db adminer`
- [ ] Run fresh webstore-db with no volume — same command, no `-v` flag
- [ ] `docker run -d --name adminer --network webstore-network -p 8081:8080 adminer`
- [ ] Open Adminer — your table is GONE — no volume, data died with container

> ↩️ **Back to DevOps — resume normal checklist**
- [ ] `docker stop webstore-db adminer`
- [ ] `docker rm webstore-db adminer`
- [ ] `docker run -d --name webstore-db --network webstore-network -e POSTGRES_DB=webstore -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=secret -v webstore-db-data:/var/lib/postgresql/data postgres:15`
- [ ] `docker run -d --name adminer --network webstore-network -p 8081:8080 adminer`
- [ ] Open Adminer — insert one row
- [ ] `docker stop webstore-db adminer`
- [ ] `docker rm webstore-db adminer`
- [ ] `docker volume ls` — webstore-db-data still exists
- [ ] Rerun webstore-db with same volume attached
- [ ] `docker run -d --name adminer --network webstore-network -p 8081:8080 adminer`
- [ ] Open Adminer — your row is STILL THERE — volume worked
- [ ] `docker stop webstore-db adminer`
- [ ] `docker rm webstore-db adminer`
- [ ] `docker volume rm webstore-db-data`
- [ ] `docker network rm webstore-network`
- [ ] `docker rmi postgres:15 adminer`

**Session win condition:** Data survived container deletion. Secrets not hardcoded anywhere. You felt data die and then fixed it.
