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

# 📟 DAY 1 FULL CHEATSHEET — Containers

## Images

| What it does | Command | Example |
|---|---|---|
| Pull with latest tag | `docker pull <IMAGE>` | `docker pull ubuntu` |
| Pull specific version | `docker pull <IMAGE>:<TAG>` | `docker pull ubuntu:22.04` |
| List all images on this machine | `docker images` | `docker images` |
| Delete one image | `docker rmi <IMAGE>:<TAG>` | `docker rmi nginx:1.24` |
| Delete multiple images at once | `docker rmi <IMAGE1> <IMAGE2> <IMAGE3>` | `docker rmi nginx:1.24 ubuntu:22.04 ubuntu` |

---

## Running Containers

| What it does | Command | Example |
|---|---|---|
| Run foreground — blocks terminal | `docker run <IMAGE>` | `docker run ubuntu` |
| Run interactively — enter the shell | `docker run -it <IMAGE>` | `docker run -it ubuntu:22.04` |
| Run in background — detached | `docker run -d <IMAGE>` | `docker run -d nginx:1.24` |
| Give the container a name | `docker run --name <NAME> <IMAGE>` | `docker run --name ubuntu-test ubuntu:22.04` |
| Bind host port to container port | `docker run -p <HOST>:<CONTAINER> <IMAGE>` | `docker run -p 8090:80 nginx:1.24` |
| Pass environment variable at startup | `docker run -e <KEY>=<VALUE> <IMAGE>` | `docker run -e POSTGRES_DB=webstore postgres:15` |
| Full background service run | `docker run -d --name <NAME> -p <HOST>:<CONTAINER> <IMAGE>` | `docker run -d --name webstore-frontend -p 8090:80 nginx:1.24` |

**Syntax breakdown — port binding:**
```
docker run -d --name webstore-frontend -p 8090:80 nginx:1.24
                                           ↑    ↑
                                      host port  container port
                                      (browser)  (app inside)
```

---

## Managing Containers

| What it does | Command | Example |
|---|---|---|
| Show running containers only | `docker ps` | `docker ps` |
| Show all containers including stopped | `docker ps -a` | `docker ps -a` |
| Start a stopped container | `docker start <NAME>` | `docker start ubuntu-test` |
| Start a stopped container and enter it | `docker start -i <NAME>` | `docker start -i ubuntu-test` |
| Stop a running container | `docker stop <NAME>` | `docker stop webstore-frontend` |
| Stop multiple at once | `docker stop <NAME1> <NAME2>` | `docker stop webstore-frontend webstore-api` |
| Delete a stopped container | `docker rm <NAME>` | `docker rm ubuntu-test` |
| Delete multiple at once | `docker rm <NAME1> <NAME2>` | `docker rm webstore-frontend webstore-api` |

---

## Logs

| What it does | Command | Example |
|---|---|---|
| See all logs from the container | `docker logs <NAME>` | `docker logs webstore-frontend` |
| Follow logs live — Ctrl+C to stop | `docker logs -f <NAME>` | `docker logs -f webstore-frontend` |
| See last 50 lines only | `docker logs --tail 50 <NAME>` | `docker logs --tail 50 webstore-frontend` |

---

## Inspect

| What it does | Command | Example |
|---|---|---|
| Full container config — everything | `docker inspect <NAME>` | `docker inspect webstore-frontend` |
| Find port mapping | `docker inspect <NAME> \| grep -A 5 "Ports"` | `docker inspect webstore-frontend \| grep -A 5 "Ports"` |
| Find container internal IP | `docker inspect <NAME> \| grep "IPAddress"` | `docker inspect webstore-frontend \| grep "IPAddress"` |
| Find which image is running | `docker inspect <NAME> \| grep "Image"` | `docker inspect webstore-frontend \| grep "Image"` |

**Syntax breakdown — grep flags:**
```
docker inspect webstore-frontend | grep -A 5 "Ports"
                                          ↑
                                   -A = lines After match
                                   -B = lines Before match
                                   -C = lines on both sides (Context)
```

---

## Exec — Enter Running Container

| What it does | Command | Example |
|---|---|---|
| Enter container — always works | `docker exec -it <NAME> /bin/sh` | `docker exec -it webstore-frontend /bin/sh` |
| Enter container — ubuntu/debian only | `docker exec -it <NAME> /bin/bash` | `docker exec -it ubuntu-test /bin/bash` |
| Run one command inside without entering | `docker exec <NAME> <COMMAND>` | `docker exec webstore-frontend ls /usr/share/nginx/html` |

---

## Safe Delete Order

| Step | What it does | Command | Example |
|---|---|---|---|
| 1 | Stop the container | `docker stop <NAME>` | `docker stop webstore-frontend` |
| 2 | Remove the container | `docker rm <NAME>` | `docker rm webstore-frontend` |
| 3 | Remove the image | `docker rmi <IMAGE>:<TAG>` | `docker rmi nginx:1.24` |

> ⚠️ Docker blocks image deletion if any container still references it — even stopped ones.

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| Container exits immediately with no error | Main process has no foreground task to keep it alive | `docker logs <NAME>` — look for the exit reason |
| `docker rm` fails — container still running | Can't delete a running container | `docker stop <NAME>` first, then `docker rm <NAME>` |
| `docker rmi` fails — image in use | A stopped container still references the image | `docker ps -a` → find it → `docker rm <NAME>` → then `docker rmi` |
| `Error: No such container` | Wrong name or container already deleted | `docker ps -a` to see what actually exists |

---

## Checklist

- ✅ `docker -v` — confirm Docker is installed
- ✅ `docker pull ubuntu` — pull without tag
- ✅ `docker pull ubuntu:22.04` — pull with specific tag
- ✅ `docker pull nginx:1.24` — pull the frontend image
- ✅ `docker images` — read every column out loud: REPOSITORY, TAG, IMAGE ID, SIZE
- ✅ `docker run --name ubuntu-test -it ubuntu:22.04` — enter container interactively

> 🧪 **Practice Only — Not a DevOps Skill**
> As a DevOps engineer you will never shell into a container to run basic Linux commands in production. These steps exist only to prove that a container has its own isolated filesystem and process space. The test will walk you through them step by step.

- ✅ Inside container: `whoami` `hostname` `ls /` `cat /etc/os-release`
- ✅ Inside container: `echo "I was here" > /tmp/test.txt`
- ✅ `exit` — leave the container

> ↩️ **Back to DevOps — resume normal checklist**
- ✅ `docker ps` — notice ubuntu-test is NOT here
- ✅ `docker ps -a` — it is here, status Exited
- ✅ `docker start -i ubuntu-test` — re-enter same container
- ✅ `cat /tmp/test.txt` — file survived stop and start
- ✅ `exit`
- ✅ `docker run -d --name webstore-frontend -p 8090:80 nginx:1.24` — run as service
- ✅ `docker ps` — confirm running, read PORTS column
- ✅ Open browser — `http://YOUR_EC2_IP:8090` — nginx welcome page
- ✅ `docker logs webstore-frontend` — see nginx startup logs
- ✅ `docker logs -f webstore-frontend` — follow live, refresh browser, watch new line appear
- ✅ `Ctrl+C` — stop following logs
- ✅ `docker inspect webstore-frontend | grep -A 5 "Ports"` — find port mapping
- ✅ `docker inspect webstore-frontend | grep "IPAddress"` — find container IP
- ✅ `docker inspect webstore-frontend | grep "Image"` — find image name
- ✅ `docker exec -it webstore-frontend /bin/sh` — enter running container
- ✅ Inside: `ls /usr/share/nginx/html/` — see the default index.html
- ✅ `exit`
- ✅ `docker stop webstore-frontend ubuntu-test`
- ✅ `docker ps` — gone from running
- ✅ `docker ps -a` — still exists, status Exited
- ✅ `docker rm webstore-frontend ubuntu-test`
- ✅ `docker rmi nginx:1.24 ubuntu:22.04 ubuntu`
- ✅ `docker images` — confirm clean
- ✅ `docker ps -a` — confirm clean

**Session win condition:** You can pull, run, enter, inspect, log, stop, and remove without looking at notes.
