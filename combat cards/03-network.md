# 🌐 NETWORKING_CORE — Combat Card
> **80/20 Rule:** Debug in layer order. Discovery → Reachability → Port → App.   
> **Pre-shift scan:** The 5-question drill → Port commands → Common ports.   
> **Source:** [03. Networking – Foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/README.md)   

---

## 1. THE MENTAL MODEL — How Packets Move
> **Source:** [01-foundation-and-the-big-picture](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/01-foundation-and-the-big-picture/README.md)

```
Browser types webstore.com
  └─→ DNS: name → IP             (Layer 7 — dig, nslookup)
  └─→ TCP Handshake              (Layer 4 — nc, curl)
  └─→ Routing: IP hops           (Layer 3 — ping, traceroute)
  └─→ NAT: private → public IP   (your router / AWS IGW)
  └─→ Firewall: allowed?         (Security Group / iptables)
  └─→ Port: which service?       (Layer 4 — ss, nc)
  └─→ App responds               (Layer 7 — curl, logs)
```

**MAC changes every hop. IP never changes. Port never changes.**

---

## 2. THE 5-QUESTION DEBUG DRILL
> **Source:** [10-complete-journey](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/10-complete-journey/README.md)   
> Connection fails? Ask these in order. Stop when you find the break.

```
1. DNS working?      dig +short <hostname>        → Returns an IP?
2. Host reachable?   ping -c 4 <ip>               → Gets replies?
3. Port open?        nc -zv <host> <port>          → "succeeded"?
4. Firewall OK?      sudo ss -tlnp | grep <port>   → Listening?
5. App responding?   curl -v http://<host>:<port>  → HTTP 200?
```

**Read the failure:**
- `unknown host` → DNS broke (Step 1)
- `Request timeout` → host unreachable or firewall (Step 2)
- `Connection refused` → port closed, service not running (Step 3)
- `Connection timed out` → firewall is silently dropping (Step 3/4)
- `502 Bad Gateway` → nginx up, upstream API down (Step 5)

---

## 3. PORT DISCOVERY — See What's Listening
> **Source:** [06-ports-transport](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/06-ports-transport/README.md)   

| Command | What it does |
|---|---|
| `sudo ss -tlnp` | All listening TCP ports + which process owns each one |
| `sudo ss -tlnp \| grep :80` | Is port 80 in use? By what process? |
| `sudo ss -tunp` | All TCP and UDP connections with process names |
| `ss -t state established` | Show only active/established connections |
| `sudo ss -tlnp \| grep :8080` | Is the ShopStack API port in use? |

**Reading ss output:**
```
LISTEN  0  511  0.0.0.0:80   users:(("nginx",pid=1235))
              ↑              ↑
         backlog          0.0.0.0 = accessible from outside
                          127.0.0.1 = local only (good for DB)
```

**ShopStack ports to verify:**
```
:80    → webstore-frontend (nginx)
:8080  → webstore-api
:5432  → webstore-db (should be 127.0.0.1 only — never 0.0.0.0)
```

---

## 4. CONNECTIVITY — Test the Path
> **Source:** [02-addressing-fundamentals](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/02-addressing-fundamentals/README.md)   

| Command | What it does |
|---|---|
| `ping -c 4 <host>` | Is the host alive? Round-trip time? |
| `nc -zv <host> <port>` | Is that specific port open and accepting connections? |
| `nc -zv -w 3 <host> <port>` | Same, but fail after 3 seconds |
| `curl -I <url>` | HTTP status code + headers only |
| `curl -v <url>` | Full verbose HTTP request and response |
| `traceroute <host>` | Where does the packet die? Hop-by-hop |
| `ip addr show` | My interfaces and IP addresses |
| `ip route show` | My routing table and default gateway |

**ShopStack context:**
```bash
# Can the API reach the database?
nc -zv webstore-db 5432        # Docker: use container name
nc -zv localhost 5432          # Host: use localhost

# Is the frontend serving?
curl -I http://localhost:80

# Is the API responding?
curl -v http://localhost:8080/api/products
```

---

## 5. DNS — Name Resolution
> **Source:** [08-dns](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/08-dns/README.md).  

| Command | What it does |
|---|---|
| `dig +short <hostname>` | Just the IP — fastest DNS check |
| `dig <hostname>` | Full DNS answer with TTL and server info |
| `dig @8.8.8.8 <hostname>` | Query Google DNS directly — bypass local cache |
| `nslookup <hostname>` | Simpler DNS lookup — works everywhere |

**When to use:** After a DNS change, run `dig +short` to confirm the new IP is propagating. If wrong, `dig @8.8.8.8` to check what the authoritative server has.

---

## 6. COMMON PORTS — Own These
> **Source:** [06-ports-transport](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/06-ports-transport/README.md)   

```
22    → SSH
53    → DNS (UDP)
80    → HTTP
443   → HTTPS
3306  → MySQL
5432  → PostgreSQL   ← webstore-db
6379  → Redis
8080  → webstore-api (ShopStack)
8081  → adminer (ShopStack dev tool)
27017 → MongoDB
```

---

## 7. WHAT BREAKS — Fast Reference
> **Source:** [10-complete-journey](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/10-complete-journey/README.md)   

| Symptom | Meaning | First command |
|---|---|---|
| `curl: (7) Failed to connect` | Service not running or wrong port | `sudo ss -tlnp` |
| `502 Bad Gateway` | nginx up, upstream API down | `nc -zv api-host 8080` |
| `ping` succeeds but `nc` fails | Firewall blocking the specific port | `sudo ss -tlnp \| grep <port>` |
| `ss` shows `127.0.0.1` not `0.0.0.0` | Service bound to localhost — not accessible externally | Edit service config bind address |
| DNS returns old IP | TTL not expired, cached answer | `dig @8.8.8.8 <hostname>` to bypass cache |
| `Connection refused` vs `timed out` | Refused = port closed. Timed out = firewall silently dropping | `nc -zv -w 3 <host> <port>` |

---

## ⚡ MASTER ONE-LINERS

```bash
# Full connectivity chain — run these top to bottom
dig +short <hostname>              # Step 1: DNS
ping -c 4 <ip>                     # Step 2: Reachable?
nc -zv <host> <port>               # Step 3: Port open?
curl -I http://<host>:<port>       # Step 4: App responding?

# What's using port 8080?
sudo ss -tlnp | grep :8080

# ShopStack full stack check
sudo ss -tlnp | grep -E ':80|:8080|:5432'
```

---

> **Own this page. The rest is details.**
