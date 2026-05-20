# Session 02 — Networking

**Goal:** Understand Docker networking, wire containers together using custom networks and Docker DNS, verify ShopStack network isolation works correctly.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Docker Networking Exists

Containers are isolated by default — they cannot talk to each other unless you explicitly wire them together. Without networking:
- The API cannot reach the database
- The frontend cannot reach the API
- Nothing works

Docker networking lets you wire containers together safely — with full control over who can talk to who.

---

## The Localhost Rule — Most Common Mistake

```
localhost always means "the machine I am currently running inside"
```

| Where you are | What localhost means |
|---|---|
| Your laptop | Your laptop |
| api container | api container only |
| db container | db container only |

```bash
DB_HOST=localhost   # WRONG — api looks for db inside itself
DB_HOST=db          # RIGHT  — Docker DNS resolves "db" to the db container
```

---

## Visual Map — ShopStack Two-Network Design

```
Browser
    │ port 80
    ▼
┌─────────────┐    web network     ┌─────────────┐
│  frontend   │ ←────────────────► │     api     │
│  Nginx :80  │                    │  Python     │
└─────────────┘                    │  :8080      │
                                   └──────┬──────┘
                                          │ backend network
                            ┌─────────────┼─────────────┐
                            ▼             ▼             ▼
                      ┌──────────┐ ┌──────────┐ ┌──────────┐
                      │    db    │ │  worker  │ │ adminer  │
                      │ Postgres │ │    Go    │ │  :8081   │
                      │  :5432   │ │    —     │ │          │
                      └──────────┘ └──────────┘ └──────────┘

frontend CANNOT reach db — different networks (intentional isolation)
api is on BOTH networks — the only bridge between the two tiers
```

This is the same concept as AWS VPC subnets:
- `web` network = public subnet
- `backend` network = private subnet

---

## Docker DNS — How It Works

When you create a custom Docker network, Docker starts an embedded DNS server at `127.0.0.11`. Every container on that network is registered by its service name.

```
api container asks: "who is db?"
        ↓
Docker DNS at 127.0.0.11
        ↓
Returns: "db = 172.18.0.3"
        ↓
api connects to 172.18.0.3:5432
```

> ⚠️ Docker DNS only works on **custom named networks**. The default `bridge` network has NO DNS. Containers on default bridge cannot find each other by name.

---

## The Files — ShopStack Network Design

The developer designed the two-network architecture. It lives in `docker-compose.yml`. You implement and maintain it.

### 📄 infra/docker-compose.yml — Networks Section

**What it does:** defines two isolated networks. Compose creates them automatically on `docker compose up`.

<details>
<summary>📄 Show networks section — infra/docker-compose.yml</summary>

```yaml
networks:
  web:      # Compose creates this — frontend and api join it
  backend:  # Compose creates this — api, db, worker, adminer join it

# Each service declares which networks it joins:

# frontend — web only
  frontend:
    networks:
      - web          # can reach api, cannot reach db

# api — both networks (the bridge between tiers)
  api:
    networks:
      - backend      # can reach db
      - web          # can reach frontend

# db — backend only
  db:
    networks:
      - backend      # invisible to frontend — intentional

# worker — backend only
  worker:
    networks:
      - backend      # pings api by name — Docker DNS resolves "api"

# adminer — backend only
  adminer:
    networks:
      - backend      # connects to db by name
```

</details>

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing to set up
## Docker is already installed. Networks are created fresh each step.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Create a custom network

```bash
docker network create shopstack-web
docker network ls | grep shopstack
```

Expected: `shopstack-web` appears with driver `bridge`

---

### Step 2 — Run two containers on the same network

```bash
docker run -d --name container-a --network shopstack-web nginx:alpine
docker run -d --name container-b --network shopstack-web nginx:alpine
```

---

### Step 3 — Prove DNS works on custom network

```bash
docker exec container-a ping -c 2 container-b
```

Expected: ping succeeds — Docker DNS resolved `container-b` to its IP

```bash
docker exec container-a cat /etc/resolv.conf
```

Expected: `nameserver 127.0.0.11` — that's Docker's embedded DNS server

---

### Step 4 — Prove default bridge has NO DNS

```bash
docker run -d --name no-dns-test nginx:alpine
# this container is on the default bridge — no custom network

docker exec container-a ping -c 1 no-dns-test 2>&1 | head -2
```

Expected: `ping: bad address 'no-dns-test'` — default bridge has no DNS

---

### Step 5 — Prove network isolation

```bash
# container-a is on shopstack-web
# no-dns-test is on default bridge
# They cannot reach each other
docker exec container-a ping -c 1 no-dns-test 2>&1
```

Expected: failure — different networks = isolated

---

### Step 6 — Connect a container to a second network

```bash
# Connect container-a to a second network
docker network create shopstack-backend
docker network connect shopstack-backend container-a

# Verify container-a is on both networks
docker inspect container-a | grep -A 5 '"Networks"'
```

Expected: `shopstack-web` and `shopstack-backend` both appear

---

### Step 7 — Inspect ShopStack network design

```bash
cat ~/shopstack/infra/docker-compose.yml | grep -A 3 "networks:"
```

Expected: `web` and `backend` networks defined

---

### Step 8 — Clean up

```bash
docker stop container-a container-b no-dns-test
docker rm container-a container-b no-dns-test
docker network rm shopstack-web shopstack-backend
```

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Custom network creation works
docker network create test-net
docker network ls | grep test-net

# 2. DNS works on custom network
docker run -d --name net-test --network test-net nginx:alpine
docker run -d --name dns-test --network test-net nginx:alpine
docker exec net-test ping -c 2 dns-test

# 3. Default bridge has NO DNS
docker run -d --name no-dns-test nginx:alpine
docker exec net-test ping -c 1 no-dns-test 2>&1 | head -2

# 4. ShopStack networks defined correctly
cat ~/shopstack/infra/docker-compose.yml | grep -A 2 "^networks:"

# 5. Clean up
docker stop net-test dns-test no-dns-test
docker rm net-test dns-test no-dns-test
docker network rm test-net
docker ps -a | grep -E "net-test|dns-test|no-dns-test"
```

Expected for #2: ping succeeds
Expected for #3: `ping: bad address 'no-dns-test'`
Expected for #5: no output — all cleaned up

**All 5 green?** → Save file → push → Session 03.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ping: bad address 'NAME'` | On default bridge or different network | Recreate with `--network YOUR_NETWORK` |
| Frontend reaches db (it shouldn't) | Both on same network | Check network design — frontend should be web only |
| `network rm` fails | Containers still connected | Stop and rm all containers first |
| Connection refused on correct port | Service not ready yet | Check logs — wait for healthcheck |

---

## Quick Reference

| What | Command |
|---|---|
| Create network | `docker network create NAME` |
| List networks | `docker network ls` |
| Inspect network | `docker network inspect NAME` |
| Connect container | `docker network connect NETWORK CONTAINER` |
| Remove network | `docker network rm NAME` |
| Test DNS | `docker exec CONTAINER ping -c 2 TARGET` |
| Check container network | `docker inspect CONTAINER \| grep -A 5 Networks` |
| Check DNS config | `docker exec CONTAINER cat /etc/resolv.conf` |
