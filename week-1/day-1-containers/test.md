# Week 1 — 1. Containers

[1. Containers](../day-1-containers/test.md) · [2. Ports & Networking](../day-2-networking/test.md) · [3. Volumes](../day-3-volumes/test.md)

---

# Test Yourself — No Notes. No Help.

Answer out loud or write it down. Open the toggle only after you have answered.

---

## Round 1 — Concepts

**Q1. What is the difference between an image and a container?**

<details>
<summary>Answer</summary>

An image is frozen and read-only — the blueprint. It never runs on its own. A container is a running instance of that image. One image can run as many containers as you want. Each container is completely independent. Stopping a container does not delete it — only `docker rm` does.

</details>

---

**Q2. You exit the `frontend` container. You run `docker ps` and it is not there. Where did it go?**

<details>
<summary>Answer</summary>

It is still there — just stopped. `docker ps` shows only running containers. `docker ps -a` shows all containers including stopped ones. Exiting stops the container but does not delete it. The writable layer and everything inside is still intact until you run `docker rm`.

</details>

---

**Q3. What does `-d` do? Why does ShopStack use it for every service?**

<details>
<summary>Answer</summary>

Detached mode. The container runs in the background and you get your terminal back immediately. ShopStack uses it for every service because `frontend`, `api`, `db`, `worker`, and `adminer` are all long-running services — they need to keep running after you start them without blocking the terminal.

</details>

---

**Q4. What is the correct cleanup order and why does the order matter?**

<details>
<summary>Answer</summary>

Stop → rm → rmi. Docker blocks image deletion if any container still references it — even a stopped one. You must remove the container first then the image. Skipping stop before rm also fails if the container is still running.

</details>

---

**Q5. What is the difference between `docker ps` and `docker ps -a`?**

<details>
<summary>Answer</summary>

`docker ps` shows only currently running containers. `docker ps -a` shows all containers — running, stopped, and exited. Always run `docker ps -a` before assuming things are clean. Stopped containers still consume disk space.

</details>

---

## Round 2 — Commands

Every question maps directly to your Day 1 checklist. By the end of this round every checklist item has been recalled from memory.

---

**Q6. Confirm Docker is installed and show the version.**

<details>
<summary>Answer</summary>

```bash
docker -v
```

</details>

---

**Q7. Pull `nginx:1.24`.**

<details>
<summary>Answer</summary>

```bash
docker pull nginx:1.24
```

</details>

---

**Q8. List all images on this machine.**

<details>
<summary>Answer</summary>

```bash
docker images
```

Read every column out loud — REPOSITORY, TAG, IMAGE ID, CREATED, SIZE.

</details>

---

**Q9. Run `nginx:1.24` in the background, name it `frontend`, bind EC2 port 8090 to container port 80.**

<details>
<summary>Answer</summary>

```bash
docker run -d --name frontend -p 8090:80 nginx:1.24
```

</details>

---

**Q10. Confirm `frontend` is running and read the PORTS column.**

<details>
<summary>Answer</summary>

```bash
docker ps
```

Look for `0.0.0.0:8090->80/tcp` in the PORTS column.

</details>

---

**Q11. See all logs from `frontend`.**

<details>
<summary>Answer</summary>

```bash
docker logs frontend
```

</details>

---

**Q12. Follow the live logs of `frontend`. Refresh your browser and watch a new line appear.**

<details>
<summary>Answer</summary>

```bash
docker logs -f frontend
```

Ctrl+C to stop following.

</details>

---

**Q13. Find the port mapping of `frontend` with 5 lines of context.**

<details>
<summary>Answer</summary>

```bash
docker inspect frontend | grep -A 5 "Ports"
```

</details>

---

**Q14. Find the internal IP address of `frontend`.**

<details>
<summary>Answer</summary>

```bash
docker inspect frontend | grep "IPAddress"
```

</details>

---

**Q15. Find which image `frontend` is running.**

<details>
<summary>Answer</summary>

```bash
docker inspect frontend | grep "Image"
```

</details>

---

**Q16. Enter the running `frontend` container with a shell.**

<details>
<summary>Answer</summary>

```bash
docker exec -it frontend /bin/sh
```

</details>

---

**Q17. While inside — list the contents of `/usr/share/nginx/html/`.**

<details>
<summary>Answer</summary>

```bash
ls /usr/share/nginx/html/
```

You should see `index.html` — this is the file nginx serves to the browser.

</details>

---

**Q18. Stop `frontend`. Confirm it is gone from running. Confirm it still exists as stopped.**

<details>
<summary>Answer</summary>

```bash
docker stop frontend
docker ps       # gone from running
docker ps -a    # still exists, status Exited
```

</details>

---

**Q19. Delete the `frontend` container, then delete the `nginx:1.24` image. Confirm both are gone.**

<details>
<summary>Answer</summary>

```bash
docker rm frontend
docker rmi nginx:1.24
docker ps -a    # confirm clean
docker images   # confirm clean
```

</details>

---

## Round 3 — Scenario Debug

**Scenario A — Container exits immediately**

You run `docker run -d --name api python:3.12` and check `docker ps` — the container is not there. You check `docker ps -a` and see `Exited (1)`. What happened and what is the first command you run?

<details>
<summary>Answer</summary>

Exit code 1 means the process crashed at startup. Python has no default foreground command — it starts and immediately exits because there is nothing to keep it alive.

First command:
```bash
docker logs api
```

Never restart first. Never rebuild first. Read the crash.

</details>

---

**Scenario B — Port binding shows but browser fails**

You run `frontend` with `-p 8090:80` and confirm it shows in `docker ps`. You open the browser and get connection refused. What are the two most likely causes?

<details>
<summary>Answer</summary>

**Cause 1 — EC2 security group does not have port 8090 open.**
Check: AWS Console → EC2 → Security Groups → Inbound rules.

**Cause 2 — The app inside is listening on `127.0.0.1` not `0.0.0.0`.**
```bash
docker exec -it frontend ss -tlnp
```
If it shows `127.0.0.1:80` — the app is bound to localhost only and cannot be reached from outside.

</details>

---

**Scenario C — Image deletion blocked**

You run `docker rmi nginx:1.24` and get:
```
Error response from daemon: conflict: unable to delete — container is using its referenced image
```
What do you do and in what order?

<details>
<summary>Answer</summary>

A stopped container still references the image. Find it and remove it first.

```bash
docker ps -a
# Find the container using nginx:1.24 in the IMAGE column
docker rm CONTAINER_NAME
docker rmi nginx:1.24
```

</details>

---

**Scenario D — Container running but app is down**

Your team says `frontend` is running fine. You open the store and it is down. What is the first command you run?

<details>
<summary>Answer</summary>

```bash
docker logs frontend
```

Running does not mean healthy. The container process can be alive while nginx inside has a config error and is serving nothing. Read the logs before doing anything else.

</details>

---

## Round 4 — Reading Output

**Q20. You run `docker images` and see two rows for `nginx` — tagged `latest` and `1.24`. Are they the same image?**

<details>
<summary>Answer</summary>

No. Different tags, different image IDs, potentially different content. `nginx:latest` changes silently whenever a new version is released. `nginx:1.24` is pinned to that specific version. Always use specific tags in ShopStack — never `latest` in any config that needs to be reproducible.

</details>

---

**Q21. You run `docker inspect frontend | grep "IPAddress"` and see `172.17.0.3`. Can you reach this IP from your Mac browser?**

<details>
<summary>Answer</summary>

No. That is the container's internal IP on the Docker bridge network inside the EC2 instance. It is a private IP that only exists inside the EC2 host. You reach the container from your Mac via the port binding on the EC2 public IP: `http://YOUR_EC2_IP:8090`. The internal IP is only useful for container-to-container communication.

</details>

---

**Q22. You run `docker logs frontend` and see:**

```
172.17.0.1 - - [19/Apr/2026:22:14:03] "GET / HTTP/1.1" 200 612
```

What does this tell you?

<details>
<summary>Answer</summary>

Someone made a GET request to `/` — the homepage. The server responded HTTP 200 — success. Response body was 612 bytes. The IP `172.17.0.1` is the Docker bridge gateway — this is your browser's request coming through the port binding. This confirms nginx is running and serving correctly.

</details>
