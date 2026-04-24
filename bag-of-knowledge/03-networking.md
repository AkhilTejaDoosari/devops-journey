[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Networking

**Hire importance at $25–40/hr:** 8/10

**One line:** Docker networking, Kubernetes Services, and AWS VPCs are all networking with different wrappers. Understand the base — the wrappers stop feeling like magic.

📚 Full runbook → [Networking – Foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/README.md)

---

## 0 — Before You Open A Terminal

### What a developer does that makes networking necessary

A developer writes an API that listens on port 8080. They write a database connection that talks to `localhost:5432`. They write a frontend that calls `http://api:8080/products`. They do not think about how packets travel — they just write the address and the port and expect it to work.

Your job is to make it work. When the developer writes `localhost:5432`, you need to know that inside a container `localhost` means the container itself — not the database. When they write `http://api:8080`, you need to know that `api` resolves via Docker DNS only if both containers are on the same custom network. When the browser cannot reach the frontend, you need to know whether the problem is a port binding, a firewall rule, or a DNS resolution failure.

Networking is the layer underneath everything. You cannot debug Docker, Kubernetes, or AWS without it.

### The building and apartment analogy — the one aha that never leaves

```
IP address = the building address — gets you to the right machine
Port number = the apartment number — gets you to the right application

192.168.1.100:8080
└───────────┘ └──┘
   building   apartment

One building. Many apartments.
:22   SSH
:80   HTTP / Nginx
:443  HTTPS
:8080 ShopStack API
:5432 Postgres
```

A packet arrives at the machine. The OS reads the port number. It delivers the packet to whichever application is listening on that port. If nothing is listening — connection refused.

### The complete packet journey — simplified

```
Browser types EC2_IP:80
      ↓
DNS not needed — direct IP
      ↓
TCP handshake — SYN, SYN-ACK, ACK
      ↓
Packet hits AWS Security Group — is port 80 allowed inbound?
      ↓
Packet hits EC2 — is anything listening on port 80?
      ↓
Nginx receives the request
      ↓
Nginx proxies to API on port 8080 via Docker network
      ↓
API talks to DB on port 5432 via backend Docker network
```

Every step in that journey is a networking concept. When it breaks — you debug layer by layer.

### ShopStack files that depend on networking

```
shopstack/infra/docker-compose.yml    ← networks: web, backend defined here
shopstack/services/frontend/nginx.conf ← proxy_pass http://api:8080
AWS Security Group                     ← inbound rules for ports 80, 8080, 22
EC2 instance                           ← private IP + public IP, NAT by AWS
```

---

## 1 — The Stop Point

### Stop here for $25–40/hr — own every concept below cold

- IP address gets you to the right machine — port gets you to the right application
- Private IP vs public IP — why EC2 has both and what NAT does between them
- TCP vs UDP — TCP guarantees delivery and order, UDP does not — Postgres uses TCP
- DNS — translates a name to an IP — why container names work as hostnames on Docker networks
- What a firewall rule is — inbound vs outbound, what allows and what blocks
- What NAT does — your laptop has a private IP, the router translates it to public
- `curl`, `ping`, `ss`, `dig` — what each one tests and what the result tells you
- What "connection refused" means vs "connection timeout" — different problems, different fixes
- What a Docker custom network does — container isolation and built-in DNS
- AWS Security Group — stateful firewall — inbound rule is enough, return traffic is automatic

**When you can explain all ten without hesitation — stop. Move to Docker.**

### What unlocks higher pay

At $55/hr you add: VPC design — public vs private subnets, NAT gateway, Internet Gateway. Load balancer concepts — ALB, health checks, SSL termination. Network policies in Kubernetes — restricting pod-to-pod traffic. Deep iptables rules — understanding what Docker and Kubernetes write automatically. At $80/hr you design the network architecture, not just operate it.

📚 Deep dive → [Networking Runbook](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/README.md)

---

## 2 — The Bag

### Own These Cold

IP address gets you to the right machine. Port number gets you to the right application on that machine. You need both. A packet addressed to `192.168.1.100` with no port is like a letter with a building address and no apartment number — nobody knows where to deliver it.

TCP guarantees delivery. It does a three-way handshake before sending data — SYN, SYN-ACK, ACK — and confirms every packet arrives. Postgres, SSH, HTTP all use TCP. UDP does not guarantee delivery. It sends and does not wait for confirmation — DNS queries and video streaming use UDP because speed matters more than perfect delivery.

Private IP addresses are for internal networks — `10.x.x.x`, `172.16.x.x`, `192.168.x.x`. They are not routable on the internet. Your EC2 has a private IP — something like `172.31.x.x`. AWS assigns it a public IP and uses NAT to translate between them. When a browser connects to your EC2 public IP, AWS translates it to the private IP before it reaches the instance.

DNS translates names to IPs. When you type `shopstack.com`, your machine asks a DNS server "what IP is this?" and gets back an address. Inside a Docker custom network, Docker runs its own DNS server. When the API container talks to `db`, Docker DNS resolves `db` to the database container's IP automatically. That is why container names work as hostnames.

A firewall rule controls which traffic is allowed. Inbound rules control what can reach your machine. Outbound rules control what your machine can send. AWS Security Groups are stateful — if you allow inbound traffic on port 8080, the return traffic is automatically allowed without a separate outbound rule. AWS NACLs are stateless — you need rules in both directions.

"Connection refused" means the machine is reachable but nothing is listening on that port. The application is not running or it is bound to the wrong address. "Connection timeout" means the packet never arrived — a firewall dropped it or the host is unreachable. These two errors tell you completely different things.

📚 Deep dive → [Ports & Transport](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/06-ports-transport/README.md)

📚 Deep dive → [Firewalls](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/09-firewalls/README.md)

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Exact syntax → combat sheet or AI.

- Port not reachable from browser → check Security Group inbound rules then `ss -tulnp`
- Container cannot reach another container → check both are on the same Docker network
- Container name not resolving → not on a custom network — bridge network has no DNS
- 502 Bad Gateway from Nginx → Nginx cannot reach the API — check API container is running
- Connection refused on a port → nothing listening — check the app is running and bound to correct address
- Connection timeout → firewall blocking — check Security Group or iptables
- DNS not resolving → check `/etc/resolv.conf` or use `dig` to test directly

---

### Domain Awareness

- Subnetting math — CIDR notation beyond /24 and /16 — useful for VPC design later
- BGP and routing protocols — network engineering territory, not DevOps at this level
- OSI model layers 1 and 2 — Physical and Data Link — abstracted in cloud and containers
- IPv6 — know it exists, not tested at junior level in most interviews
- Load balancer algorithms — round robin, least connections — comes up at $55/hr level

---

### The one insight that separates copiers from understanders

Networking does not change between Docker, Kubernetes, and AWS — only the vocabulary does.

Docker bridge network is NAT. Kubernetes Service is a load balancer with DNS. AWS Security Group is a stateful firewall. AWS VPC is a private network with subnets.

Once you understand the base concepts — IP, port, NAT, DNS, firewall — every new tool becomes "which networking concept is this wrapping?" not "what magic is this doing?"

---

## 3 — Combat Sheet

The combat sheet is not a memorisation list. It is your pressure-proof reference. When something breaks — open this, find the situation, run the command.

---

### Daily Drivers

These are the commands you use every single working day debugging ShopStack and EC2.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Don't know if a port is listening | `ss -tulnp \| grep PORT` | `ss -tulnp \| grep 8080` |
| Don't know if an endpoint is alive | `curl -I http://host:PORT/path` | `curl -I http://localhost:8080/api/health` |
| Don't know if host is reachable at all | `ping host` | `ping 8.8.8.8` |
| Need to resolve a domain to IP | `dig domain` | `dig api.shopstack.com` |
| Need to check container-to-container reach | `docker exec -it CONTAINER ping OTHER` | `docker exec -it infra-frontend-1 ping api` |

---

### Situational

You recognise the situation. You open this table. You find the command. You close it.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Port reachable locally but not from browser | Check AWS Security Group inbound rules for that port | Port 80 or 8080 not open in ShopStack SG |
| Connection refused on port | `ss -tulnp \| grep PORT` — nothing listening | App not running or bound to wrong address |
| Connection timeout to port | Firewall dropping — check Security Group or iptables | NACL blocking return traffic |
| 502 Bad Gateway from Nginx | `docker compose logs frontend` then `curl http://localhost:8080` | API container down or wrong port |
| Container name not resolving | `docker network inspect NETWORK` — is container on it? | Worker not on backend network |
| Need full connection trace | `curl -v http://host:PORT` | `curl -v http://localhost:8080/api/health` |
| Need to test port reachability without curl | `nc -zv host PORT` | `nc -zv db 5432` |
| Need to trace where packet dies | `traceroute host` | `traceroute 8.8.8.8` |
| Need to see all Docker networks | `docker network ls` | Check web and backend networks exist |
| Need to inspect a Docker network | `docker network inspect NETWORK` | `docker network inspect infra_backend` |
| Need to check what DNS server EC2 uses | `cat /etc/resolv.conf` | Check after networking issues on EC2 |
| Need to check EC2 public IP | `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` | After every EC2 restart |

---

## 4 — ShopStack Map

| Situation | Command or File | Knowledge Needed |
|---|---|---|
| Frontend cannot reach API | `docker exec -it infra-frontend-1 ping api` | Container DNS, custom network |
| Port 8080 not reachable from browser | Check AWS Security Group → `ss -tulnp \| grep 8080` | Firewall + port binding |
| Worker cannot connect to API | `docker compose logs worker` — look for connection refused | Service discovery, container hostname = service name |
| Adminer cannot reach DB | Both must be on backend network — `docker network inspect infra_backend` | Docker network isolation |
| 502 Bad Gateway from Nginx | `docker compose logs frontend` → check proxy_pass in nginx.conf | Proxy, upstream unreachable |
| Check what IP the API container has | `docker inspect infra-api-1 \| grep IPAddress` | Container IP, bridge network |
| Prove port binding works for frontend | `curl -I http://EC2_IP:80` | NAT, port binding, -p flag |
| API container cannot reach DB by hostname | Both must be on same network — `db` resolves only on backend network | Docker DNS, network scope |
| EC2 has new IP after restart | `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` | AWS metadata service |
| Need to prove two containers are isolated | Try `ping` from one to other — if unreachable, networks are separate | Network isolation by design |

📚 Deep dive → [Complete Journey](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/10-complete-journey/README.md)

---

## 5 — Peace of Mind

> Answer every question out loud before opening the toggle.

> If you stumble — go to the terminal. Not back to the notes.

---

**Q1. What is the difference between an IP address and a port? Why do you need both to reach a service?**

<details>
<summary>Answer</summary>

The IP address gets you to the right machine — like the building address. The port gets you to the right application on that machine — like the apartment number. A packet addressed to an IP with no port has no way of knowing which of the dozens of processes running on that machine should receive it. You need both. `192.168.1.100:8080` means "machine at this IP, API application listening on port 8080."

</details>

---

**Q2. ShopStack's API runs on port 8080 inside the container. What does `-p 8080:8080` in docker-compose actually do?**

<details>
<summary>Answer</summary>

It tells Docker to bind port 8080 on the EC2 host to port 8080 inside the API container. When a browser sends a request to `EC2_IP:8080`, the OS receives it on host port 8080 and Docker forwards it to the container's port 8080. Without this binding, the API is only reachable inside the Docker network — not from outside the machine. The format is `HOST_PORT:CONTAINER_PORT`.

</details>

---

**Q3. The frontend cannot reach the API by hostname `api`. What are the two most likely causes?**

<details>
<summary>Answer</summary>

First: the containers are not on the same custom Docker network. Docker only provides automatic DNS resolution on custom networks — not on the default bridge. On a custom network, Docker's internal DNS resolves `api` to the API container's IP automatically. On the default bridge, there is no DNS.

Second: the API container is not running. DNS resolves the name correctly but the connection is refused because nothing is listening. `docker compose ps` confirms the container state.

</details>

---

**Q4. What is the difference between "connection refused" and "connection timeout"?**

<details>
<summary>Answer</summary>

Connection refused means the machine is reachable but nothing is listening on that port. The OS actively sends back a refusal. The application is either not running or bound to a different address — like `127.0.0.1` instead of `0.0.0.0`.

Connection timeout means the packet never came back at all. Either a firewall dropped it silently, the host is unreachable, or the route does not exist. The client waited, got no response, and gave up.

These tell you completely different things. Refused means the app problem. Timeout means the network or firewall problem.

</details>

---

**Q5. ShopStack has two networks — web and backend. Why not put everything on one network?**

<details>
<summary>Answer</summary>

Isolation by design. The frontend should never be able to talk directly to the database. Two networks enforce that boundary at the infrastructure level — not just by convention or hope. The web network connects frontend and API. The backend network connects API, DB, worker, and Adminer. The API is on both because it is the intentional bridge between tiers. If everything shared one network, a misconfigured frontend could theoretically reach the database directly. Two networks make that impossible.

</details>

---

**Q6. EC2 has a private IP of `172.31.x.x` but the browser connects to a public IP. What makes that work?**

<details>
<summary>Answer</summary>

NAT — Network Address Translation — done by AWS. The EC2 instance only knows its private IP. AWS maps a public IP to it at the infrastructure level. When a browser connects to the public IP, AWS translates the destination to the private IP before the packet reaches the instance. When the instance responds, AWS translates the source from private back to public. The instance never sees the public IP — `ip addr` on EC2 shows only the private IP.

</details>

---

**Q7. What does DNS do and why do container names work as hostnames inside ShopStack?**

<details>
<summary>Answer</summary>

DNS translates a name to an IP address. When you type `api` as a hostname, something must translate it to an actual IP before a connection can be made.

Inside a Docker custom network, Docker runs an embedded DNS server at `127.0.0.11`. Every container on the network is registered by its service name. When the frontend container tries to reach `api`, Docker DNS looks up `api` and returns the API container's current IP. This is why you use service names instead of hardcoded IPs in docker-compose — the IP changes every time a container restarts, but the name stays the same.

</details>

---

**Q8. What is the difference between an AWS Security Group and a NACL?**

<details>
<summary>Answer</summary>

A Security Group is stateful — if you allow inbound traffic on port 8080, the return traffic is automatically allowed without a separate outbound rule. Security Groups are attached to instances.

A NACL is stateless — it checks every packet independently, both directions. If you allow inbound on port 8080, you must also explicitly allow outbound for the ephemeral response ports or the return traffic is blocked. NACLs are attached to subnets and are evaluated before Security Groups.

Most ShopStack debugging starts at Security Groups because they are the more common point of failure for junior setups.

</details>

---

**Q9. How does `ping` differ from `curl` as a diagnostic tool?**

<details>
<summary>Answer</summary>

`ping` tests Layer 3 — IP connectivity. It sends ICMP packets and checks if the machine responds. If ping works, the machine is reachable at the network level. It tells you nothing about whether a specific port is open or an application is running.

`curl` tests Layer 7 — application connectivity. It makes a real HTTP request to a specific port and path. If curl works, the machine is reachable, the port is open, and the application responded. If ping works but curl fails, the problem is the application or the port — not the network path.

For ShopStack: ping the EC2 IP to confirm connectivity, then curl the specific endpoints to confirm the services are running.

</details>

---

**Q10. The worker container logs show "connection refused" trying to reach the API. What do you check first?**

<details>
<summary>Answer</summary>

First: `docker compose ps` — is the API container actually running and healthy. Connection refused to the service name means either the API is down or the worker cannot resolve `api` to reach it.

Second: `docker network inspect infra_backend` — is the worker on the backend network. The worker reaches the API via the backend network. If the worker is not on that network, Docker DNS cannot resolve `api` from the worker's perspective.

Third: `docker compose logs api` — did the API start correctly and is it actually listening on port 8080.

</details>

---

## 6 — AI Split

### Own this yourself — never outsource

- IP vs port — the building and apartment analogy — tested in every interview
- TCP vs UDP — when each is used and why — Postgres uses TCP and why that matters
- What NAT does — private IP, public IP, translation — this is AWS EC2 every day
- Connection refused vs connection timeout — different problems, different fixes
- What Docker DNS does on a custom network — why container names work as hostnames
- Stateful vs stateless firewall — Security Group vs NACL — tested in AWS interviews

### Use AI for this — guilt free

- Exact `iptables` rule syntax for a specific scenario
- CIDR subnet calculations beyond /24 — how many hosts in a /20
- Writing a specific `dig` command with trace flags
- Setting up a static IP on Ubuntu EC2
- VPC subnet design for a multi-tier application

### The right questions to ask AI for Networking

```
"ShopStack API container is getting connection refused
when trying to reach the db container.
Both are defined in docker-compose.yml.
Here is my networks config: [paste].
What are the two most likely causes and how do I verify each?"

"I have this ss -tulnp output: [paste].
The API should be listening on 8080.
Is it? What does the address column tell me?"

"My EC2 Security Group has port 80 open inbound.
Nginx is running. But the browser times out instead of refused.
What is the most likely cause and what do I check next?"

"Explain the difference between Docker bridge network
and a Docker custom network for container DNS resolution.
Use ShopStack frontend and api as the example."
```

📚 Full Networking runbook → [Networking – Foundations](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/03.%20Networking%20%E2%80%93%20Foundations/README.md)
