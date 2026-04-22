# Day 2 — Test Answers

Only open this after you have answered Test.md on your own.

---

## Round 1 — Concepts

**A1. What does `-p 8080:80` mean?**

`8080` is the host port — what the outside world connects to on the EC2.
`80` is the container port — what the app listens on inside.
Traffic hits EC2 port 8080, Docker rewrites the destination and forwards it to port 80 inside the container. This is NAT.

---

**A2. Two containers from the same image on different ports — same container?**

No. Same image, two completely separate containers. Each has its own filesystem, its own network namespace, its own process. The image is just the blueprint. The containers are independent instances.

---

**A3. Postgres with no `-p` flag — browser vs container?**

Browser: cannot reach it. No `-p` flag means no NAT rule on the host. Nothing from outside Docker can connect.
Another container on the same network: yes. Containers on the same network communicate directly using container ports — no port binding needed. Port binding is only for traffic coming from outside the Docker network.

---

**A4. What is Docker DNS?**

Docker's embedded DNS server running at `127.0.0.11` inside every container on a custom named network. It resolves container names to their internal IPs automatically. You use a container name like `webstore-db` instead of an IP like `172.18.0.2`.

Works on: custom named networks (`docker network create`).
Does NOT work on: the default `bridge` network — that one has no DNS.

---

**A5. Two containers on different networks — can they ping each other by name?**

No. Docker DNS only resolves names within the same network. Different networks have no route between them. The frontend cannot reach the database in ShopStack because they are on different networks — that isolation is intentional.

---

**A6. What is `127.0.0.11`?**

Docker's embedded DNS server. Automatically configured inside every container on a custom network. When a container does a DNS lookup for another container name, this server answers with the correct internal IP.

---

**A7. Why does the ShopStack API have two IP addresses?**

The API is connected to both `infra_frontend` and `infra_backend` networks. One IP per network. It is the bridge — it receives requests from the frontend on one network and talks to the database on the other. The frontend has no route to the backend network so it can never reach the database directly.

---

**A8. Why does the ShopStack worker have no port binding?**

The worker processes orders in the background. Nothing initiates a connection to it — it reaches out to the database, not the other way around. Port binding is only needed when something from outside needs to connect to you. The worker never needs to be connected to.

---

## Round 2 — Scenario Debug

**Scenario A — `bad address 'webstore-db'`**

Most likely cause: the containers are on different networks, or one of them is on the default `bridge` network which has no DNS.

How to confirm:
```bash
docker inspect YOUR_APP_CONTAINER | grep -A 5 Networks
docker inspect webstore-db | grep -A 5 Networks
```
Both must show the same network name. If one shows `bridge` and the other shows `webstore-network` — that is the problem. They are isolated from each other.

---

**Scenario B — `port is already allocated`**

Means another container or process on the EC2 already owns host port 8080. Host ports are unique — only one process can bind a port at a time.

First command to run:
```bash
sudo ss -tlnp | grep 8080
```
This shows what process is holding port 8080. Kill it or use a different host port.

---

**Scenario C — Adminer works, browser cannot reach db directly**

Nothing is broken. This is correct behavior. `webstore-db` has no `-p` flag so there is no NAT rule on the host. The browser is outside Docker — it cannot reach the container. Adminer is inside Docker on the same network — it can reach the container by name. Database should never be publicly exposed.

---

## Round 3 — Commands

**A9. Create a network:**
```bash
docker network create app-network
```

**A10. Run postgres on the network:**
```bash
docker run -d --name app-db --network app-network -e POSTGRES_DB=myapp -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=secret postgres:15
```

**A11. Show all containers on a network with IPs:**
```bash
docker network inspect app-network
```
Look for the `Containers` section.

**A12. Confirm DNS server inside a container:**
```bash
cat /etc/resolv.conf
```
You should see `nameserver 127.0.0.11`.

**A13. Correct cleanup order:**
```bash
docker stop CONTAINERS
docker rm CONTAINERS
docker network rm NETWORK
docker rmi IMAGES
```
Always stop before rm. Always rm containers before removing the network. Always remove containers before removing images.
