# Day 2 — Ports and Networking

**Date:** April 20 2026
**Read before session:** [04-docker-port-binding](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/04-docker-port-binding) · [05-docker-networking](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/05-docker-networking)
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)
**Goal:** Bind ports, create networks, prove Docker DNS resolves container names automatically.

---

## Knowledge — What These Topics Are Really About

### Port Binding
Containers are isolated. An app running inside a container on port 80 is unreachable from the outside world by default. Port binding creates a NAT rule on the host — it maps a host port to a container port. When traffic arrives on the host port, iptables rewrites the destination and forwards it into the container.

```
Browser → EC2:8080 → iptables DNAT rule → container:80 → nginx
```

`-p 8080:80` means: host port 8080 maps to container port 80. They do not have to match.

**The core rule:** Port binding is only for traffic coming FROM outside Docker. Containers on the same network talk directly — no port binding needed between them.

### Docker Networking
Every container gets its own network namespace — its own IP stack, its own localhost. By default containers are isolated from each other. A named network is how you wire containers together.

The default `bridge` network has no DNS — containers can only find each other by IP, which changes every restart. A named network (`docker network create`) has DNS built in — containers resolve each other by name automatically.

**The core rule:** Put containers that need to talk on the same named network. Use names not IPs in your app config.

### Docker DNS
Every container on a custom named network gets `127.0.0.11` as its DNS server automatically. That server is managed by Docker and maps every container name to its internal IP. When adminer connects to `webstore-db`, it sends a DNS query to `127.0.0.11`, gets back `172.18.0.2`, and connects.

**The core rule:** Never hardcode container IPs. IPs change every restart. Names do not.

### ShopStack Implementation
ShopStack uses two networks — `infra_frontend` and `infra_backend`. The API sits on both. The frontend talks to the API. The API talks to the database. The frontend has no route to the database — different network, complete isolation. The database has no `-p` flag — unreachable from outside Docker by design.

```
Browser → frontend (infra_frontend) → api (infra_frontend + infra_backend) → db (infra_backend)
```

---

## Full Cheatsheet

### Port Binding

| Command | What it does |
|---|---|
| `docker run -p 8080:80 IMAGE` | Bind host port 8080 to container port 80 |
| `docker ps` | See PORTS column — active bindings |
| `docker port CONTAINER` | Show all port mappings for one container |
| `sudo ss -tlnp \| grep 8080` | Find what process owns host port 8080 |
| `docker inspect CONTAINER \| grep -A 5 Ports` | Full port binding detail |
| `sudo iptables -t nat -L DOCKER -n` | Show DNAT rules Docker created — one per `-p` flag |

### Networking

| Command | What it does |
|---|---|
| `docker network create NAME` | Create a named bridge network with DNS |
| `docker network ls` | List all networks on this host |
| `docker network inspect NAME` | Show all containers on network and their IPs |
| `docker network rm NAME` | Delete a network — containers must be removed first |
| `docker run --network NAME IMAGE` | Attach container to a named network at startup |

### DNS Verification (inside container)

| Command | What it does |
|---|---|
| `cat /etc/resolv.conf` | Confirm DNS server is `127.0.0.11` |
| `ping CONTAINER_NAME` | Test DNS resolution by name |
| `nslookup CONTAINER_NAME` | Full DNS lookup — shows server and resolved IP |

---

## Power Commands — Debugging Combos

These are sequences you run when something is broken. Not individual commands — flows.

### App not loading in browser
```bash
# 1 — is the container running?
docker ps

# 2 — is the port binding active?
docker ps --format "table {{.Names}}\t{{.Ports}}"

# 3 — is something else already on that host port?
sudo ss -tlnp | grep 8080

# 4 — is the app inside the container actually listening?
docker exec -it CONTAINER ss -tlnp

# 5 — is the EC2 security group open on that port?
# AWS Console → EC2 → Security Groups → Inbound rules
```

### Container cannot reach another container by name
```bash
# 1 — are they on the same network?
docker inspect CONTAINER_A | grep -A 5 Networks
docker inspect CONTAINER_B | grep -A 5 Networks

# 2 — confirm DNS is configured inside the container
docker exec -it CONTAINER_A cat /etc/resolv.conf

# 3 — test DNS resolution by name
docker exec -it CONTAINER_A ping CONTAINER_B

# 4 — test actual TCP connection
docker exec -it CONTAINER_A nc -zv CONTAINER_B PORT
```

### See everything about a container's network
```bash
docker inspect CONTAINER | grep -A 20 NetworkSettings
```

### See the real iptables rules Docker created
```bash
sudo iptables -t nat -L DOCKER -n
# One DNAT rule per -p flag — if missing, port binding failed silently
```

### Check all port bindings across all containers at once
```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

---

## Best Practices

**Never expose the database.** No `-p` flag on postgres. Databases are internal only — reached by container name, never by browser. `webstore-db` is only reachable inside the Docker network.

**Never use the default bridge network for multi-container apps.** Default bridge has no DNS. Your app hardcodes IPs that change every restart. Always `docker network create` a named network.

**Never hardcode container IPs.** Use `webstore-db:5432` not `172.18.0.2:5432`. IPs are assigned fresh every container start. Names are stable.

**Port binding is for external traffic only.** Container-to-container on the same network uses the container port directly — not the host port. Adminer reaches postgres on `5432`, not `8081`. `8081` is only for your browser.

**Both security group AND port binding must be open for external access.** Security group closed = packet blocked before it hits Docker. Port binding missing = packet blocked inside Docker. Two separate layers, both required.

**Use the same port on both sides when there is no conflict.** `-p 8080:8080` is easier to reason about than `-p 3000:8080`. Only use different host ports when you have a conflict.

---

## What You Proved Today

1. Same image, two containers, different host ports — they are completely independent instances
2. Port binding is NAT — host port → iptables DNAT rule → container port
3. Containers on the same named network resolve each other by name automatically
4. Docker DNS is `127.0.0.11` — configured automatically, no manual setup needed
5. Containers on different networks cannot see each other — ShopStack uses this for security isolation
6. Database never needs `-p` — internal only, other containers reach it by name on the shared network

---

## Checklist

- ✅ `docker run -d --name webstore-frontend -p 8080:80 nginx:1.24`
- ✅ Open `http://EC2_IP:8080` — nginx welcome page loads
- ✅ `docker run -d --name webstore-api -p 8081:80 nginx:1.24`
- ✅ `docker ps` — read PORTS column for both containers
- ✅ `docker stop webstore-frontend webstore-api`
- ✅ `docker rm webstore-frontend webstore-api`
- ✅ `docker network create webstore-network`
- ✅ `docker network ls` — confirm webstore-network exists
- ✅ `docker run -d --name webstore-db --network webstore-network -e POSTGRES_DB=webstore -e POSTGRES_USER=admin -e POSTGRES_PASSWORD=secret postgres:15`
- ✅ `docker run -d --name adminer --network webstore-network -p 8081:8080 adminer`
- ✅ Open `http://EC2_IP:8081` — Adminer UI loads
- ✅ Login — system: PostgreSQL, server: `webstore-db`, user: admin, pass: secret, db: webstore
- ✅ `docker exec -it adminer /bin/sh`
- ✅ Inside: `ping webstore-db` — resolves by name
- ✅ Inside: `cat /etc/resolv.conf` — confirmed `nameserver 127.0.0.11`
- ✅ `exit`
- ✅ `docker network inspect webstore-network` — read container IPs
- ✅ `docker stop adminer webstore-db`
- ✅ `docker rm adminer webstore-db`
- ✅ `docker network rm webstore-network`
- ✅ `docker rmi postgres:15 adminer:latest nginx:1.24`
