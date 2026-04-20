# Day 1 — Test Answers

Only open this after you have answered Test.md on your own.

---

## Round 1 — Concepts

**A1. What is the difference between an image and a container?**

An image is frozen — like Captain America in ice. A container is alive — like Captain America after being thawed. The image is the blueprint. The container is the running instance. One image can run many containers. Stopping a container does not delete it. Only `docker rm` deletes it.

---

**A2. You run `docker ps` after exiting a container and it is not there. Where did it go?**

It is still there — just stopped. `docker ps` shows only running containers. `docker ps -a` shows all containers including stopped ones. Exiting stops the container but does not delete it.

---

**A3. You created a file inside a container, stopped it, started it again. Is the file still there? Why?**

Yes. Stopping a container does not delete it. The container's filesystem survives stop and restart. The file lives in the container's writable layer. It only disappears when you `docker rm` the container.

---

**A4. What does `-d` do in `docker run -d`?**

Detached mode. The container runs in the background. You get your terminal back immediately. The container keeps running even if you close the terminal or disconnect SSH.

---

**A5. What does `0.0.0.0:8090->80/tcp` mean in `docker ps` output?**

Port binding. Any traffic hitting port 8090 on the EC2 machine gets forwarded to port 80 inside the container. `0.0.0.0` means all network interfaces on the host. That is how your Mac browser reaches nginx running inside the container.

---

**A6. What is the correct order to clean up Docker? Why that order?**

Stop → rm → rmi. Container first, image second. Docker will block image deletion if any container still references it — even a stopped one. Always remove the container first then the image.

---

**A7. Your current directory is `/home/ubuntu/shopstack/infra`. You run `docker images`. What do you see?**

All images on the machine. Current directory does not matter for `docker images`. Docker Engine stores images in `/var/lib/docker/` and `docker images` shows everything Docker knows about regardless of where you are.

---

**A8. What is the difference between `docker ps` and `docker ps -a`?**

`docker ps` — running containers only.
`docker ps -a` — all containers including stopped and exited ones.

---

## Round 2 — Commands

**A9.**
```bash
docker pull ubuntu
```

**A10.**
```bash
docker pull ubuntu:22.04
```

**A11.**
```bash
docker run --name test-box -it ubuntu:22.04
```

**A12.**
```bash
docker start -i test-box
```

**A13.**
```bash
docker run -d --name web -p 8090:80 nginx:1.24
```

**A14.**
```bash
docker logs -f web
```

**A15.**
```bash
docker exec -it web /bin/sh
```

**A16.**
```bash
docker inspect web | grep "IPAddress"
```

**A17.**
```bash
docker inspect web | grep -A 5 "Ports"
```

**A18.**
```bash
docker stop web
docker rm web
docker rmi nginx:1.24
```

---

## Round 3 — Scenario Debug

**Scenario A — container exits immediately with status Exited (1)**

The app crashed at startup. Exit code 1 means the process errored out.

First command:
```bash
docker logs api
```
The logs will show the exact error — wrong env var, missing config, port already in use, anything. Never guess. Never restart first. Read the logs.

---

**Scenario B — port binding shows in `docker ps` but browser shows connection refused**

Two most likely causes:

1. EC2 security group does not have port 8090 open to your IP.
   Check: AWS Console → EC2 → Security Groups → Inbound rules. Look for port 8090.

2. The app inside the container is listening on `127.0.0.1` instead of `0.0.0.0`.
   Check:
   ```bash
   docker exec -it web ss -tlnp
   ```
   If it shows `127.0.0.1:80` instead of `0.0.0.0:80` — the app is not reachable from outside even with port binding active.

---

**Scenario C — `docker rmi` blocked by container reference**

Docker is telling you a container — running or stopped — still references the image. You must remove the container first.

```bash
# Find which container is using it
docker ps -a
# Look for the image name in the IMAGE column

# Remove the container
docker stop CONTAINER_NAME
docker rm CONTAINER_NAME

# Now delete the image
docker rmi nginx:1.24
```

---

**Scenario D — team says container is running but app is down**

`docker ps` showing a container as running only means the process inside is alive. It does not mean the app is healthy or responding correctly.

First command:
```bash
docker logs CONTAINER_NAME
```
Look for errors, panics, crash messages, or repeated failure lines. If it is a web app, look for connection refused to a dependency like a database. The container can be "running" while the app inside is completely broken.

---

## Round 4 — Reading Output

**A19. Two ubuntu rows — latest and 22.04 — same image?**

No. Different tags, different image IDs, different content. `ubuntu:latest` is whatever the latest version is at pull time. `ubuntu:22.04` is pinned to that specific version. They may share some layers but they are separate images.

---

**A20. `172.17.0.3` — what is it and can you reach it from Mac?**

It is the container's internal IP on the Docker bridge network. You cannot reach it directly from your Mac — it is a private network inside the EC2 instance. You can only reach the container via the port binding on the EC2 public IP (`18.219.143.12:8090`).

---

**A21. Log line reading:**

Someone made a GET request to `/` (the homepage). The server responded with HTTP 200 (success). The response body was 615 bytes. The IP `173.17.106.24` is the client — your Mac. This confirms the browser successfully reached nginx and got a valid response.
