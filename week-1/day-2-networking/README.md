# Day 2 — Ports and Networking

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md)

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

# 📟 DAY 2 FULL CHEATSHEET — Ports and Networking

## Port Binding

| What it does | Command | Example |
|---|---|---|
| Bind host port to container port | `docker run -p <HOST>:<CONTAINER> <IMAGE>` | `docker run -p 8080:80 nginx:1.24` |
| See PORTS column — active bindings | `docker ps` | `docker ps` |
| Show all port mappings for one container | `docker port <CONTAINER>` | `docker port webstore-frontend` |
| Find what process owns a host port | `sudo ss -tlnp \| grep <PORT>` | `sudo ss -tlnp \| grep 8080` |
| Full port binding detail | `docker inspect <CONTAINER> \| grep -A 5 Ports` | `docker inspect webstore-frontend \| grep -A 5 Ports` |
| Show DNAT rules Docker created | `sudo iptables -t nat -L DOCKER -n` | `sudo iptables -t nat -L DOCKER -n` |

**Syntax breakdown — port binding:**
```
docker run -p 8080:80 nginx:1.24
              ↑    ↑
         host port  container port
         (browser)  (nginx inside)

They do NOT have to match.
Host port = what you type in your browser.
Container port = what the app listens on inside.
```

---

## Networking

| What it does | Command | Example |
|---|---|---|
| Create a named bridge network with DNS | `docker network create <n>` | `docker network create webstore-network` |
| List all networks on this host | `docker network ls` | `docker network ls` |
| Show all containers on a network and their IPs | `docker network inspect <n>` | `docker network inspect webstore-network` |
| Delete a network — containers must be removed first | `docker network rm <n>` | `docker network rm webstore-network` |
| Attach container to a named network at startup | `docker run --network <n> <IMAGE>` | `docker run --network webstore-network postgres:15` |

---

## DNS Verification (Inside Container)

> 🧪 **Practice Only — Not a DevOps Skill**
> As a DevOps engineer you will not shell into containers to debug DNS in production. These commands exist only to give you visual proof that Docker DNS is working during learning. In production you debug from the host using `docker inspect` and `docker network inspect`. The test will walk you through them step by step.

| What it does | Command | Example |
|---|---|---|
| Confirm DNS server is `127.0.0.11` | `cat /etc/resolv.conf` | `cat /etc/resolv.conf` |
| Test DNS resolution by name | `ping <CONTAINER_NAME>` | `ping webstore-db` |
| Full DNS lookup — shows server and resolved IP | `nslookup <CONTAINER_NAME>` | `nslookup webstore-db` |

**Syntax breakdown — what Docker DNS does:**
```
adminer container → wants to reach webstore-db
       ↓
sends DNS query to 127.0.0.11  (Docker's embedded DNS server)
       ↓
gets back 172.18.0.2           (webstore-db's private IP)
       ↓
connects directly on port 5432

You never hardcode the IP. Names are stable. IPs change every restart.
```

---

## Full webstore-db Run Command (Day 2 Reference)

```
docker run -d \
  --name webstore-db \
  --network webstore-network \
  -e POSTGRES_DB=webstore \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
        ↑              ↑                  ↑
   no -p flag     named network      env vars for postgres
  (internal only)  (DNS enabled)     (db, user, password)
```

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| `Bind for 0.0.0.0:8080 failed: port already allocated` | Another container or process already owns that host port | `sudo ss -tlnp \| grep 8080` → find it → stop it |
| Browser shows connection refused despite container running | The `-p` flag was omitted | `docker ps` — check PORTS column, if empty the flag is missing |
| `ping webstore-db` fails from inside another container | Containers are not on the same network | `docker inspect` both — check `Networks` section |
| Adminer can't connect — server field wrong | Used `localhost` instead of container name | Use `webstore-db` as the server — never `localhost` between containers |
| Port 8081 already in use | Leftover container from previous run still running | `docker ps` → `docker rm -f adminer` |

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

> 🧪 **Practice Only — Not a DevOps Skill**   
> As a DevOps engineer you will not shell into containers to debug DNS in production. These steps exist only to give you visual proof that Docker DNS is working during learning. In production you debug from the host using `docker inspect` and `docker network inspect`. The test will walk you through them step by step.

- ✅ Inside: `ping webstore-db` — resolves by name
- ✅ Inside: `cat /etc/resolv.conf` — confirmed `nameserver 127.0.0.11`
- ✅ `exit`

> ↩️ **Back to DevOps — resume normal checklist**
- ✅ `docker network inspect webstore-network` — read container IPs
- ✅ `docker stop adminer webstore-db`
- ✅ `docker rm adminer webstore-db`
- ✅ `docker network rm webstore-network`
- ✅ `docker rmi postgres:15 adminer:latest nginx:1.24`

**Session win condition:** Two containers on the same network talk by name. You can explain Docker DNS.

---

Ready to test yourself? → [Test](./test.md)
