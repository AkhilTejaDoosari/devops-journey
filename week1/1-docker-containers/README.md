# Day 1 — Containers

**Date:** April 19 2026   
**Read before session:** [03-docker-containers](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/03-docker-containers)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Pull images, run containers, enter them, inspect them, log them, stop them, delete them. Feel the difference between an image and a container with your own hands.   

---

## Knowledge — What These Topics Are Really About

### Images
An image is a frozen, read-only package — the app plus everything it needs to run. It never runs on its own. It is the blueprint. You download it once with `docker pull` and it lives in `/var/lib/docker/` on the host.

**The core rule:** Image = what you ship. Container = what runs. Never confuse the two.

### Containers
A container is a running instance of an image. It has its own isolated filesystem, its own process space, its own network namespace. When you stop a container it does not disappear — it is still there, frozen, with its writable layer intact. It only disappears when you `docker rm` it.

**The core rule:** Stopping ≠ deleting. One image → many containers. Each container is independent.

### Lifecycle
Every container goes through: created → running → stopped → deleted. You control each transition. Docker will not auto-delete unless you use `--rm`. Stopped containers still consume disk space. That is why cleanup discipline matters.

**The core rule:** Stop → rm → rmi. Always in that order. Docker blocks image deletion if any container still references it, even stopped.

### ShopStack Connection
ShopStack runs 5 containers from a `docker compose up` command. Every one of those containers is an image running as a container — the same concept you practiced today manually. When you ran nginx on port 8090 and checked `docker ps`, you were doing exactly what ShopStack does on ports 80, 8080, and 8081.

---

## Full Cheatsheet

### Images

| Command | What it does |
|---|---|
| `docker pull IMAGE` | Pull with latest tag |
| `docker pull IMAGE:TAG` | Pull specific version |
| `docker images` | List all images on this machine |
| `docker rmi IMAGE:TAG` | Delete one image |
| `docker rmi IMAGE1 IMAGE2 IMAGE3` | Delete multiple images at once |

### Running Containers

| Command | What it does |
|---|---|
| `docker run IMAGE` | Run foreground — blocks terminal |
| `docker run -it IMAGE` | Run interactively — enter the shell |
| `docker run -d IMAGE` | Run in background — detached |
| `docker run --name NAME IMAGE` | Give the container a name |
| `docker run -p HOST:CONTAINER IMAGE` | Bind EC2 port to container port |
| `docker run -d --name NAME -p HOST:CONTAINER IMAGE` | Full background service run |
| `docker run -e KEY=VALUE IMAGE` | Pass environment variable at startup |

### Managing Containers

| Command | What it does |
|---|---|
| `docker ps` | Show running containers only |
| `docker ps -a` | Show all containers including stopped |
| `docker start NAME` | Start a stopped container |
| `docker start -i NAME` | Start a stopped container and enter it |
| `docker stop NAME` | Stop a running container |
| `docker stop NAME1 NAME2` | Stop multiple at once |
| `docker rm NAME` | Delete a stopped container |
| `docker rm NAME1 NAME2` | Delete multiple at once |
| `docker restart NAME` | Stop and start without rebuilding |

### Logs

| Command | What it does |
|---|---|
| `docker logs NAME` | See all logs from the container |
| `docker logs -f NAME` | Follow logs live — Ctrl+C to stop |
| `docker logs --tail 50 NAME` | See last 50 lines only |
| `docker logs --previous NAME` | Logs from before the last restart |

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
| `docker exec NAME COMMAND` | Run one command inside without entering |

### Grep Flags

| Flag | What it does |
|---|---|
| `-A 5` | Show 5 lines After the match |
| `-B 5` | Show 5 lines Before the match |
| `-C 5` | Show 5 lines on both sides (Context) |

### Cleanup Order — Memorize This

```
docker stop NAME      ← 1. stop first
docker rm NAME        ← 2. delete container
docker rmi IMAGE:TAG  ← 3. delete image last
```

Docker blocks image deletion if any container references it — even stopped ones.

---

## Power Commands — Debugging Combos

These are sequences you run when something is broken. Not individual commands — flows.

### Container won't stay up — keeps restarting or exiting immediately
```bash
# 1 — what is the exit status?
docker ps -a
# Look at STATUS column — "Exited (1)" means app crashed

# 2 — what did the app say before it died?
docker logs CONTAINER

# 3 — what was the last restart like?
docker logs --previous CONTAINER
```

### Container is running but app is not responding
```bash
# 1 — is the port binding active?
docker ps
# Check PORTS column — if empty, -p flag was missing

# 2 — is the app inside actually listening on the right port?
docker exec -it CONTAINER ss -tlnp

# 3 — check the app logs for errors
docker logs -f CONTAINER
```

### You forgot how a container was configured
```bash
# Source of truth — everything Docker knows about this container
docker inspect CONTAINER

# Targeted extracts
docker inspect CONTAINER | grep -A 5 "Ports"     # port binding
docker inspect CONTAINER | grep "IPAddress"       # internal IP
docker inspect CONTAINER | grep "Image"           # which image
docker inspect CONTAINER | grep -A 10 "Env"       # env vars passed at startup
```

### Clean up everything fast
```bash
# Stop all running containers at once
docker stop $(docker ps -q)

# Remove all stopped containers at once
docker rm $(docker ps -aq)

# Then remove images one by one
docker rmi IMAGE1 IMAGE2
```

### Confirm truly clean state
```bash
docker ps -a      # should be empty
docker images     # should show only ShopStack images
docker network ls # should show only bridge, host, none
```

---

## Best Practices

**Always name your containers.** `docker run --name nginx` not `docker run nginx`. Random names like `sleepy_morse` are impossible to manage. You cannot stop, log, or exec into something you cannot remember the name of.

**Always use specific image tags.** `nginx:1.24` not `nginx`. The `latest` tag changes silently — your container today may behave differently than tomorrow. Pin the version.

**`-d` for services, `-it` for exploration.** Services run in the background with `-d`. You never enter them at startup. Interactive shells use `-it`. Mixing them up is the most common beginner mistake.

**Stopping is not deleting.** Stopped containers still consume disk. Always `docker ps -a` before assuming things are clean. Clean up with `stop → rm → rmi` in that order.

**`docker logs` before anything else.** When a container breaks, check logs first. Do not restart, do not rebuild, do not delete. Logs tell you the root cause in 30 seconds. Rebuilding hides it.

**Never store important data inside a container.** The writable layer dies with `docker rm`. Data that must survive lives in volumes — that is Day 3.

---

## What You Proved Today

1. `docker ps` vs `docker ps -a` — running vs all
2. Stopping a container does not delete it — filesystem survives
3. One image runs as multiple containers — each independent
4. `-d` runs in background — terminal is free, container keeps running
5. `docker logs -f` shows live traffic — browser request appeared in real time
6. `docker inspect` is the source of truth — ports, IP, image, env vars all there
7. `docker exec -it` enters a running container — no restart needed
8. Cleanup order is non-negotiable — container before image

---

## Checklist

- ✅ `docker -v` — confirm Docker is installed
- ✅ `docker pull ubuntu` — pull without tag
- ✅ `docker pull ubuntu:22.04` — pull with specific tag
- ✅ `docker pull nginx:1.24` — pull the frontend image
- ✅ `docker images` — read every column out loud: REPOSITORY, TAG, IMAGE ID, SIZE
- ✅ `docker run --name ubuntu-test -it ubuntu:22.04` — enter container interactively
- ✅ Inside container: `whoami` `hostname` `ls /` `cat /etc/os-release`
- ✅ Inside container: `echo "I was here" > /tmp/test.txt`
- ✅ `exit` — leave the container
- ✅ `docker ps` — notice ubuntu-test is NOT here
- ✅ `docker ps -a` — it is here, status Exited
- ✅ `docker start -i ubuntu-test` — re-enter same container
- ✅ `cat /tmp/test.txt` — file survived stop and start
- ✅ `exit`
- ✅ `docker run -d --name nginx -p 8090:80 nginx:1.24` — run as service
- ✅ `docker ps` — confirm running, read PORTS column
- ✅ Open browser — `http://18.219.143.12:8090` — nginx welcome page
- ✅ `docker logs nginx` — see nginx startup logs
- ✅ `docker logs -f nginx` — follow live, refresh browser, watch new line appear
- ✅ `Ctrl+C` — stop following logs
- ✅ `docker inspect nginx | grep -A 5 "Ports"` — find port mapping
- ✅ `docker inspect nginx | grep "IPAddress"` — find container IP
- ✅ `docker inspect nginx | grep "Image"` — find image name
- ✅ `docker exec -it nginx /bin/sh` — enter running container
- ✅ Inside: `ls /usr/share/nginx/html/` — see the default index.html
- ✅ `exit`
- ✅ `docker stop nginx`
- ✅ `docker stop ubuntu-test`
- ✅ `docker ps` — gone from running
- ✅ `docker ps -a` — still exists, status Exited
- ✅ `docker rm nginx`
- ✅ `docker rm ubuntu-test`
- ✅ `docker rmi nginx:1.24 ubuntu:22.04 ubuntu:latest`
- ✅ `docker images` — confirm clean
- ✅ `docker ps -a` — confirm clean
