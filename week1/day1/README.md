# Day 1 — Containers

**Date:** April 19 2026   
**Read before session:** [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Pull images, run containers, enter them, inspect them, log them, stop them, delete them. Feel the difference between an image and a container with your own hands.   

---

## Checklist

- [x] `docker -v` — confirm Docker is installed
- [x] `docker pull ubuntu` — pull without tag
- [x] `docker pull ubuntu:22.04` — pull with specific tag
- [x] `docker pull nginx:1.24` — pull the frontend image
- [x] `docker images` — read every column out loud: REPOSITORY, TAG, IMAGE ID, SIZE
- [x] `docker run --name ubuntu-test -it ubuntu:22.04` — enter container interactively
- [x] Inside container: `whoami` `hostname` `ls /` `cat /etc/os-release`
- [x] Inside container: `echo "I was here" > /tmp/test.txt`
- [x] `exit` — leave the container
- [x] `docker ps` — notice ubuntu-test is NOT here
- [x] `docker ps -a` — it is here, status Exited
- [x] `docker start -i ubuntu-test` — re-enter same container
- [x] `cat /tmp/test.txt` — file survived stop and start
- [x] `exit`
- [x] `docker run -d --name nginx -p 8090:80 nginx:1.24` — run as service
- [x] `docker ps` — confirm running, read PORTS column
- [x] Open browser — `http://18.219.143.12:8090` — nginx welcome page
- [x] `docker logs nginx` — see nginx startup logs
- [x] `docker logs -f nginx` — follow live, refresh browser, watch new line appear
- [x] `Ctrl+C` — stop following logs
- [x] `docker inspect nginx | grep -A 5 "Ports"` — find port mapping
- [x] `docker inspect nginx | grep "IPAddress"` — find container IP
- [x] `docker inspect nginx | grep "Image"` — find image name
- [x] `docker exec -it nginx /bin/sh` — enter running container
- [x] Inside: `ls /usr/share/nginx/html/` — see the default index.html
- [x] `exit`
- [x] `docker stop nginx` — stop the container
- [x] `docker stop ubuntu-test`
- [x] `docker ps` — gone from running
- [x] `docker ps -a` — still exists, status Exited
- [x] `docker rm nginx` — delete container
- [x] `docker rm ubuntu-test`
- [x] `docker rmi nginx:1.24 ubuntu:22.04 ubuntu:latest` — delete images
- [x] `docker images` — confirm clean
- [x] `docker ps -a` — confirm clean

---

## Full Cheatsheet — 03-docker-containers

Full notes: [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers)

### Pulling Images

| Command | What it does |
|---|---|
| `docker pull IMAGE` | Pull image with latest tag |
| `docker pull IMAGE:TAG` | Pull image with specific tag |
| `docker images` | List all images on this machine |
| `docker images \| head` | List first few images only |
| `docker rmi IMAGE:TAG` | Delete an image |
| `docker rmi IMAGE1 IMAGE2 IMAGE3` | Delete multiple images at once |

### Running Containers

| Command | What it does |
|---|---|
| `docker run IMAGE` | Run a container — foreground, deleted on exit |
| `docker run -it IMAGE` | Run interactively — enter the shell |
| `docker run -d IMAGE` | Run in background — detached |
| `docker run --name NAME IMAGE` | Give the container a name |
| `docker run -p HOST:CONTAINER IMAGE` | Bind EC2 port to container port |
| `docker run -it --name NAME IMAGE` | Interactive with a name |
| `docker run -d --name NAME -p HOST:CONTAINER IMAGE` | Full background service run |

### Managing Containers

| Command | What it does |
|---|---|
| `docker ps` | Show running containers only |
| `docker ps -a` | Show all containers including stopped |
| `docker start NAME` | Start a stopped container |
| `docker start -i NAME` | Start a stopped container and enter it |
| `docker stop NAME` | Stop a running container |
| `docker stop NAME1 NAME2` | Stop multiple containers at once |
| `docker rm NAME` | Delete a stopped container |
| `docker rm NAME1 NAME2` | Delete multiple containers at once |

### Logs

| Command | What it does |
|---|---|
| `docker logs NAME` | See all logs from container |
| `docker logs -f NAME` | Follow logs live — Ctrl+C to stop |
| `docker logs --tail 50 NAME` | See last 50 lines only |
| `docker logs --previous NAME` | See logs from before last restart |

### Inspect

| Command | What it does |
|---|---|
| `docker inspect NAME` | Full container config — everything |
| `docker inspect NAME \| grep -A 5 "Ports"` | Find port mapping |
| `docker inspect NAME \| grep "IPAddress"` | Find container internal IP |
| `docker inspect NAME \| grep "Image"` | Find which image is running |

### Exec — Enter Running Container

| Command | What it does |
|---|---|
| `docker exec -it NAME /bin/sh` | Enter container — always works |
| `docker exec -it NAME /bin/bash` | Enter container — ubuntu/debian only |
| `docker exec NAME COMMAND` | Run one command inside container without entering |

### Grep Flags

| Flag | What it does |
|---|---|
| `-A 5` | Show 5 lines After the match |
| `-B 5` | Show 5 lines Before the match |
| `-C 5` | Show 5 lines Context — both sides |

### Cleanup Order — Always This Order

```
docker stop NAME      ← 1. stop first
docker rm NAME        ← 2. delete container
docker rmi IMAGE:TAG  ← 3. delete image last
```

Docker blocks image deletion if any container references it — even stopped ones.

### Shell Options Inside Containers

```
/bin/sh    → always available — works in every container
/bin/bash  → available in ubuntu and debian containers only
```

Use `/bin/sh` as the safe default. Use `/bin/bash` when you want tab completion and better history.

### Where Docker Stores Everything

```
/var/lib/docker/    ← Docker's internal storage
```

Your current directory does NOT matter for `docker pull`, `docker run`, `docker ps`, `docker logs`, `docker exec`. It only matters for `docker build` (needs Dockerfile) and `docker compose` (needs docker-compose.yml).

---

## Key Patterns

**The one pattern you use constantly on the job:**
```bash
docker inspect CONTAINER | grep -A 5 "Ports"   # port mapping
docker inspect CONTAINER | grep "IPAddress"     # container IP
docker inspect CONTAINER | grep "Image"         # which image
```

**Port binding reads like this:**
```
0.0.0.0:8090->80/tcp
         ↑       ↑
    EC2 port   container port
```

**Image vs Container in one line:**
```
Image   = frozen blueprint — like a class in code
Container = running instance — like an object created from that class
One image → many containers
Stopping ≠ deleting
```

---

## What I Did Differently From the Checklist

- Used `nginx` as the container name instead of `webstore-frontend` — avoid conflicts with ShopStack
- Used port `8090` instead of `8080` — port 8080 is already used by ShopStack API
- Entered the stopped container with `docker exec` then `/bin/sh` — learned two ways to get inside
- Used vim first — not installed — solved it with `echo` — good instinct

---

## Done When

You can pull, run, enter, inspect, log, stop, and remove containers without looking at notes.

**Status: ✅ Achieved — April 19 2026**
