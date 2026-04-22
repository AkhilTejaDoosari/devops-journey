# Week 1 — 2. Ports & Networking

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md)

---

# Test Yourself — No Notes. No Help.

Answer out loud or write it down. Open the toggle only after you have answered.

---

## Round 1 — Concepts

**Q1. What does `-p 80:80` mean in the ShopStack frontend service? Which side is the host?**

<details>
<summary>Answer</summary>

Left side is the host port — what the outside world connects to on EC2. Right side is the container port — what nginx listens on inside. Traffic hits EC2 port 80, Docker rewrites the destination via iptables DNAT and forwards it to port 80 inside the frontend container. They do not have to match.

</details>

---

**Q2. The `db` container has no `-p` flag. Can your browser reach it? Can `api` reach it?**

<details>
<summary>Answer</summary>

Browser: no. No `-p` flag means no NAT rule on the host. Nothing from outside Docker can connect to port 5432. This is intentional — databases are never publicly exposed.

The `api` container: yes. Both are on the `backend` network. Containers on the same network communicate directly using container ports — no port binding needed. The api connects to `db:5432` using Docker DNS.

</details>

---

**Q3. What is Docker DNS? When does it work in ShopStack and when would it not work?**

<details>
<summary>Answer</summary>

Docker's embedded DNS server at `127.0.0.11` inside every container on a custom named network. It resolves container names to internal IPs automatically. In ShopStack, `api` reaches `db` by name — Docker DNS translates `db` to its internal IP on the `backend` network.

Works on: custom named networks — `web` and `backend` in ShopStack.
Does NOT work on: the default `bridge` network — containers there can only find each other by IP, which changes every restart.

</details>

---

**Q4. Why does the `api` container have two IP addresses?**

<details>
<summary>Answer</summary>

The api is connected to both `web` and `backend` networks. Docker assigns one IP per network. One IP on `web` — used by `frontend` to reach it. One IP on `backend` — used to reach `db`, `worker`, and `adminer`. The api is the only bridge between the two networks. `frontend` cannot reach `backend` directly.

</details>

---

**Q5. Why does the `worker` have no port binding?**

<details>
<summary>Answer</summary>

The worker is a background process that pings the api every 10 seconds. Nothing ever initiates a connection to the worker — it always reaches out, never receives. Port binding is only needed when something from outside needs to connect to you. The worker never needs to be connected to.

</details>

---

## Round 2 — Commands

Every question maps directly to your Day 2 checklist. By the end of this round every checklist item has been recalled from memory.

---

**Q6. Run `nginx:1.24` in the background named `frontend` binding host port 8080 to container port 80.**

<details>
<summary>Answer</summary>

```bash
docker run -d --name frontend -p 8080:80 nginx:1.24
```

</details>

---

**Q7. Run a second `nginx:1.24` in the background named `api` binding host port 8081 to container port 80.**

<details>
<summary>Answer</summary>

```bash
docker run -d --name api -p 8081:80 nginx:1.24
```

</details>

---

**Q8. Confirm both containers are running and read the PORTS column for each.**

<details>
<summary>Answer</summary>

```bash
docker ps
```

You should see `0.0.0.0:8080->80/tcp` for frontend and `0.0.0.0:8081->80/tcp` for api.

</details>

---

**Q9. Stop and remove both containers.**

<details>
<summary>Answer</summary>

```bash
docker stop frontend api
docker rm frontend api
```

</details>

---

**Q10. Create a network named `webstore-network`.**

<details>
<summary>Answer</summary>

```bash
docker network create webstore-network
```

</details>

---

**Q11. Confirm `webstore-network` exists.**

<details>
<summary>Answer</summary>

```bash
docker network ls
```

Look for `webstore-network` in the NAME column.

</details>

---

**Q12. Run `postgres:15` named `db` on `webstore-network` with DB=webstore, user=admin, password=secret. No port binding.**

<details>
<summary>Answer</summary>

```bash
docker run -d --name db \
  --network webstore-network \
  -e POSTGRES_DB=webstore \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
```

</details>

---

**Q13. Run `adminer` on `webstore-network` binding host port 8081 to container port 8080.**

<details>
<summary>Answer</summary>

```bash
docker run -d --name adminer \
  --network webstore-network \
  -p 8081:8080 \
  adminer
```

</details>

---

**Q14. Show all containers on `webstore-network` and their internal IPs.**

<details>
<summary>Answer</summary>

```bash
docker network inspect webstore-network
```

Look for the `Containers` section — it shows every container on the network with its assigned IP.

</details>

---

**Q15. Show the DNAT rules Docker created for your port bindings.**

<details>
<summary>Answer</summary>

```bash
sudo iptables -t nat -L DOCKER -n
```

One DNAT rule per `-p` flag. This proves port binding is NAT.

</details>

---

**Q16. Find what process is holding port 8081 on the host.**

<details>
<summary>Answer</summary>

```bash
sudo ss -tlnp | grep 8081
```

</details>

---

> 🔧 **Environment Setup — Not a DevOps Skill**
> The following steps put data into the lab so you can prove DNS resolution works. Commands are provided — copy and paste.
>
> ```bash
> docker exec -it adminer /bin/sh
> ping db               # proves DNS resolves db by name
> cat /etc/resolv.conf  # confirms nameserver 127.0.0.11
> exit
> ```
>
> ↩️ **Back to DevOps**

---

**Q17. Stop and remove both `db` and `adminer`.**

<details>
<summary>Answer</summary>

```bash
docker stop db adminer
docker rm db adminer
```

</details>

---

**Q18. Delete `webstore-network`.**

<details>
<summary>Answer</summary>

```bash
docker network rm webstore-network
```

</details>

---

**Q19. Delete `postgres:15`, `adminer:latest`, and `nginx:1.24` images.**

<details>
<summary>Answer</summary>

```bash
docker rmi postgres:15 adminer:latest nginx:1.24
```

</details>

---

## Round 3 — Scenario Debug

**Scenario A — DNS resolution fails**

The ShopStack `api` logs show `bad address 'db'`. The db container is running. What is the most likely cause and how do you confirm it?

<details>
<summary>Answer</summary>

Most likely cause: `api` is not on the `backend` network. Docker DNS only resolves names within the same network.

```bash
docker inspect infra-api-1 | grep -A 10 Networks
docker inspect infra-db-1 | grep -A 10 Networks
```

Both must show `backend`. If `api` only shows `web` — it was never attached to `backend`.

</details>

---

**Scenario B — Port already allocated**

You try to start ShopStack and get:
```
Bind for 0.0.0.0:80 failed: port is already allocated
```
What does this mean and what do you run first?

<details>
<summary>Answer</summary>

Another process on the EC2 host already owns port 80. Either a previous container is still running or nginx is installed directly on the host.

```bash
sudo ss -tlnp | grep :80
```

This shows exactly what process is holding port 80. Either stop it or change the host port in `docker-compose.yml`.

</details>

---

**Scenario C — Adminer works but Mac browser cannot reach db**

`adminer` can reach `db` by name. You try to connect to `db:5432` from your Mac directly and it fails. Is something broken?

<details>
<summary>Answer</summary>

Nothing is broken. This is correct and intentional. `db` has no `-p` flag so there is no NAT rule on the EC2 host. Your Mac is outside Docker — Docker never created a path for external traffic on port 5432. `adminer` works because it is inside Docker on the `backend` network. Databases are never publicly exposed.

</details>

---

## Round 4 — Reading Output

**Q20. You run `docker network inspect webstore-network` and see `db` at `172.18.0.2`. You restart `db`. What happens to its IP?**

<details>
<summary>Answer</summary>

The IP changes. Docker assigns IPs dynamically at container start. After a restart `db` may get a different IP. This is exactly why you never hardcode container IPs — you always use the container name `db` and Docker DNS resolves it to whatever the current IP is automatically.

</details>

---

**Q21. You run `docker ps` and the PORTS column for `db` is empty. What does this mean?**

<details>
<summary>Answer</summary>

No port binding is active on `db`. This is correct and intentional — `db` has no `-p` flag because databases are internal only. The empty PORTS column confirms `db` is unreachable from outside Docker. Only containers on the `backend` network can reach it by name on port 5432.

</details>

---

**Q22. You run `docker compose logs frontend` and see:**

```
172.20.0.3 - - [22/Apr/2026] "GET /api/products HTTP/1.1" 502
172.20.0.3 - - [22/Apr/2026] "GET /api/products HTTP/1.1" 502
172.20.0.3 - - [22/Apr/2026] "GET /api/products HTTP/1.1" 200
```

What happened across these three lines?

<details>
<summary>Answer</summary>

Lines 1 and 2: nginx tried to proxy `/api/products` to the `api` container and got 502 — Bad Gateway. This means nginx is up but the upstream api was not responding. Either api was still starting up or crashed temporarily.

Line 3: The request succeeded — HTTP 200. The api came back up and nginx successfully proxied the request. This pattern is typical during startup — `frontend` starts before `api` is fully healthy and the first few requests fail until api is ready.

</details>
