# Day 1 — Test Answers

Only open this after you have answered TEST.md on your own.

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

Port binding. Any traffic hitting port 8090 on the EC2 machine gets forwarded to port 80 inside the container. `0.0.0.0` means all network interfaces. That is how your Mac browser reaches nginx running inside the container.

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

**A9. Pull ubuntu without a tag:**
```bash
docker pull ubuntu
```
Pulls `ubuntu:latest` by default.

---

**A10. Pull ubuntu 22.04 specifically:**
```bash
docker pull ubuntu:22.04
```

---

**A11. Run ubuntu:22.04 interactively named test-box:**
```bash
docker run --name test-box -it ubuntu:22.04
```

---

**A12. Re-enter a stopped container named test-box:**
```bash
docker start -i test-box
```

---

**A13. Run nginx:1.24 in background named web on port 8090:**
```bash
docker run -d --name web -p 8090:80 nginx:1.24
```

---

**A14. Follow logs of web live:**
```bash
docker logs -f web
```
Ctrl+C to stop following.

---

**A15. Enter running container web with a shell:**
```bash
docker exec -it web /bin/sh
```
Use `/bin/bash` if available. `/bin/sh` always works.

---

**A16. Find IP address of container web:**
```bash
docker inspect web | grep "IPAddress"
```

---

**A17. Find port mapping of container web with context:**
```bash
docker inspect web | grep -A 5 "Ports"
```
`-A 5` shows 5 lines after the match so you see the full mapping.

---

**A18. Stop, delete container, delete image — correct order:**
```bash
docker stop web
docker rm web
docker rmi nginx:1.24
```
Always stop before rm. Always rm container before rmi image.

---

## Round 3 — Reading Output

**A19. Two ubuntu rows — latest and 22.04 — same image?**

No. Different tags, different image IDs, different content. `ubuntu:latest` is whatever the latest version is at pull time. `ubuntu:22.04` is pinned to that specific version. They may share some layers but they are separate images.

---

**A20. `docker inspect web | grep "IPAddress"` shows `172.17.0.3`. What is it and can you reach it from Mac?**

It is the container's internal IP on the Docker bridge network. You cannot reach it directly from your Mac — it is a private network inside the EC2 instance. You can only reach the container via the port binding on the EC2 public IP (`18.219.143.12:8090`).

---

**A21. Log line: `"GET / HTTP/1.1" 200 615` — what does it tell you?**

Someone made a GET request to `/` (the homepage). The server responded with HTTP 200 (success). The response was 615 bytes. The IP `173.17.106.24` is the client — your Mac. This confirms the browser successfully reached nginx and got a response.

---

## Redo Checklist — Answers

If you got stuck on any of the 11 steps go back to the README.md checklist for that specific step. Do not read the whole checklist — only look up the step you are stuck on.
