# Day 3 - Docker Volumes

**Session:** 3 of 7 | **Week:** 1 — Docker   
**Read before starting:** [06-docker-volumes](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/04.%20Docker%20%E2%80%93%20Containerization/06-docker-volumes/README.md)   
**The Mission:** Feel data die without a volume. Feel it survive with one.   

---

## THE NARRATIVE

You already know containers can talk to each other (Day 2).    
Today you discover their fatal flaw: **they are amnesiac**. Delete a container, lose everything inside it.    
Today you will prove this failure with your own hands — then fix it permanently with named volumes.   

By end of session, `webstore-db` will outlive any container that holds it. Data written today persists through deletion and recreation. That's the Day 3 win condition.

---

### The Core Problem
A container's filesystem is temporary. It lives and dies with the container. PostgreSQL writes its data files inside the container — so when you `docker rm webstore-db`, every table, every row, every schema is erased.

```
No Volume:                         With Named Volume:
Container → deleted                Container → deleted
Data      → GONE ❌                Volume    → SURVIVES ✅
                                   New Container mounts same volume
                                   Data      → STILL THERE ✅
```

### The Three Acts of Day 3
| Act | What You Prove | Outcome |
|---|---|---|
| **Act 1** | Run postgres with no volume, insert data, delete container | Data dies ❌ |
| **Act 2** | Run postgres WITH named volume, insert data, delete container | Data lives ✅ |
| **Act 3** | Inspect, manage, and clean up volumes properly | Full control |

### Named Volume — Mental Model
```
Container (dies)     Named Volume (lives forever)
     │                        │
     └──── mounts to ────────►│ /var/lib/postgresql/data
                              │
                              └── data written here survives
                                   any number of container deletions
```

### Why `/var/lib/postgresql/data`?
That's PostgreSQL's internal data directory — hardcoded by the image. Every row you've ever inserted lives in that path.    
Mount a volume there, and Docker writes data to the volume instead of the ephemeral container layer.   

### Adminer — Your Visual Proof Tool
Adminer is a lightweight database GUI. You'll use it today to insert rows visually and **see with your own eyes** when data disappears and when it doesn't. It connects to `webstore-db` by container name (DNS, same as Day 2).

---

# 📟 DAY 3 FULL CHEATSHEET — Docker Volumes

## Volume Management

| What it does | Command |
|---|---|
| Create a named volume | `docker volume create <NAME>` |
| List all volumes on this host | `docker volume ls` |
| Show driver, mount point, creation time | `docker volume inspect <NAME>` |
| Delete a volume — container must be removed first | `docker volume rm <NAME>` |
| Delete all volumes not mounted to any container | `docker volume prune` |

---

## Running Containers With Storage

| What it does | Command |
|---|---|
| Mount a named volume into a container | `docker run -v <VOLUME_NAME>:<CONTAINER_PATH> <IMAGE>` |
| Bind mount a host directory into a container | `docker run -v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH> <IMAGE>` |
| Show exactly what is mounted and where | `docker inspect <NAME> \| grep -A 10 Mounts` |

---

## Entering a Running Container

| What it does | Command |
|---|---|
| Open an interactive shell inside a container | `docker exec -it <NAME> <SHELL>` |
| Run any binary inside the container directly | `docker exec -it <NAME> <PROGRAM> <PROGRAM_FLAGS>` |
| Open PostgreSQL shell inside a container | `docker exec -it <NAME> psql -U <USER> -d <DATABASE>` |
| Run one SQL query and exit immediately | `docker exec -it <NAME> psql -U <USER> -d <DATABASE> -c "<SQL>"` |

---

## Inside psql — PostgreSQL Shell

| What it does | Command |
|---|---|
| Create a table | `CREATE TABLE <NAME> (id SERIAL, name TEXT);` |
| Insert one row | `INSERT INTO <NAME> (name) VALUES ('<VALUE>');` |
| Read all rows | `SELECT * FROM <NAME>;` |
| Quit psql — return to host terminal | `\q` |

---

## Cleanup

| What it does | Command |
|---|---|
| Stop one or more containers | `docker stop <NAME> [<NAME2>...]` |
| Remove one or more stopped containers | `docker rm <NAME> [<NAME2>...]` |
| Remove one or more images | `docker rmi <IMAGE> [<IMAGE2>...]` |
| Delete a network | `docker network rm <NAME>` |

---

## Safe Delete Order

| Step | What it does | Command |
|---|---|---|
| 1 | Stop the container | `docker stop <NAME>` |
| 2 | Remove the container | `docker rm <NAME>` |
| 3 | Remove the volume (only if deleting data) | `docker volume rm <NAME>` |

---

## Named Volume vs Bind Mount

| When to use | Type | Syntax |
|---|---|---|
| Data must survive — databases, critical state | Named Volume | `-v <VOLUME_NAME>:<CONTAINER_PATH>` |
| Editing files from host — development | Bind Mount | `-v <HOST_ABSOLUTE_PATH>:<CONTAINER_PATH>` |

---

## ⚠️ WHAT BREAKS TODAY (Know Before You Hit It)

| Symptom | Cause | Fix |
|---|---|---|
| `volume is in use` on `docker volume rm` | A stopped container still references the volume | `docker rm` the container first, then retry |
| Adminer can't connect to `webstore-db` | Forgot `--network webstore-network` on one of the containers | `docker rm` both, rerun with `--network` flag on both |
| Data still gone even with volume flag | Volume flag syntax wrong — common typo | `docker inspect webstore-db \| grep Mounts` — if empty, volume didn't attach |
| Port 8081 already in use | Leftover adminer from Day 2 still running | `docker ps` — find and `docker rm -f` it |

---
