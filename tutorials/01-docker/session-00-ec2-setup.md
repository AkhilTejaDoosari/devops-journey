# Session 00 — EC2 Setup

**Goal:** Launch a properly sized EC2 instance, install Git and Docker, clone ShopStack. Every session that follows assumes this session is complete and verified.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why This Session Exists

Every tool in this curriculum runs on EC2. If the EC2 is wrong — wrong size, wrong ports, wrong disk — every session after this breaks. Do this once, do it right, never touch it again.

---

## Visual Map

```
Your Mac (terminal)
        │ SSH
        ▼
EC2 (c7i-flex.large — Ubuntu 26.04)
├── 4GB RAM      ← enough for Jenkins + k3s + Prometheus simultaneously
├── 30GB disk    ← enough for Docker images + k8s + Jenkins workspace
├── Git          ← pull ShopStack code
├── Docker       ← run containers
└── shopstack/   ← cloned repo, all work happens here
```

---

## EC2 Decision Log

These decisions were made for specific reasons. If you rebuild, use the same values.

| Setting | Value | Why |
|---|---|---|
| Instance type | c7i-flex.large | 4GB RAM — minimum for Jenkins + k3s + Prometheus together |
| AMI | Ubuntu 26.04 LTS | Latest LTS — long support, all packages available |
| Storage | 30GB gp3 | Docker images + k8s + Jenkins workspace — 20GB runs out |
| Key pair | shopshack | Your existing key — reuse it |
| Security group | launch-wizard-2 | Pre-configured with all required ports |

---

## Security Group — Required Ports

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | My IP | SSH access |
| 80 | TCP | Anywhere | ShopStack frontend |
| 8080 | TCP | Anywhere | Jenkins UI + ShopStack API |
| 8081 | TCP | My IP | Adminer DB browser |
| 30080 | TCP | Anywhere | Kubernetes NodePort frontend |
| 6443 | TCP | My IP | Kubernetes API server |

---

## 🧪 Lab Setup — Launch EC2

> Not a DevOps skill. In a real company EC2 instances are provisioned by infra team or Terraform.
> You are doing this manually because it is a learning lab.

1. AWS Console → EC2 → Launch Instance
2. Name: `shopstack`
3. AMI: Ubuntu Server 26.04 LTS
4. Instance type: `c7i-flex.large`
5. Key pair: select existing `shopshack`
6. Security group: select existing `launch-wizard-2`
7. Storage: change 8 GiB → **30 GiB** gp3
8. Launch instance
9. Wait 60 seconds → Connect via SSH

```bash
# SSH from your Mac
ssh -i ~/.ssh/shopshack.pem ubuntu@YOUR_EC2_IP
```

---

## 🧪 Lab Setup — Update system packages

> Not a DevOps skill. Standard server hygiene done on first boot.

```bash
sudo apt-get update -y
```

---

## 🧪 Lab Setup — Install Git

> Not a DevOps skill. Git is a developer tool. Ubuntu 26.04 ships with it pre-installed.
> Run this to confirm — if already installed, nothing changes.

```bash
sudo apt-get install -y git
git --version
# expected: git version 2.x.x
```

---

## 🧪 Lab Setup — Configure Git identity

> Not a DevOps skill. Required so git commits have an author.

```bash
git config --global user.name "AkhilTejaDoosari"
git config --global user.email "doosariakhilteja@gmail.com"
git config --list
# expected: user.name and user.email appear
```

---

## 🧪 Lab Setup — Install Docker

> Not a DevOps skill. Docker is the runtime — it needs to exist before any container work.
> In a real company Docker is pre-installed on build servers.

```bash
# One command install — official Docker script
curl -fsSL https://get.docker.com | sudo sh

# Add ubuntu user to docker group — so you can run docker without sudo
sudo usermod -aG docker ubuntu

# Apply group change without logout
newgrp docker

# Verify
docker --version
# expected: Docker version 29.x.x
```

---

## 🧪 Lab Setup — Clone ShopStack

> Not a DevOps skill. Cloning the app repo is something any engineer does on a new machine.

```bash
git clone https://github.com/AkhilTejaDoosari/shopstack.git
cd shopstack
ls
# expected: Jenkinsfile  README.md  db  docs  infra  scripts  services
```

---

## ✅ Session Checkpoint — Verify Before Moving On

Run every command. Your output must match expected. If it doesn't — fix it before starting Docker Session 01.

```bash
# 1. OS
cat /etc/os-release | grep PRETTY
```
Expected: `PRETTY_NAME="Ubuntu 26.04 LTS"`

---

```bash
# 2. RAM
free -h | grep Mem
```
Expected: `Mem: 3.7Gi` (total) — available should be above 3G on a fresh instance

---

```bash
# 3. Disk
df -h /
```
Expected: Size ~28G, Used ~2G, Use% under 15%

---

```bash
# 4. Git
git --version
```
Expected: `git version 2.53.x` or higher

---

```bash
# 5. Git identity
git config --list
```
Expected: `user.name=AkhilTejaDoosari` and `user.email=doosariakhilteja@gmail.com`

---

```bash
# 6. Docker
docker --version
```
Expected: `Docker version 29.x.x`

---

```bash
# 7. Docker runs without sudo
docker ps
```
Expected: empty table with headers — no permission error

---

```bash
# 8. ShopStack cloned
ls ~/shopstack
```
Expected: `Jenkinsfile  README.md  db  docs  infra  scripts  services`

---

```bash
# 9. Docker Compose available
docker compose version
```
Expected: `Docker Compose version v2.x.x`

---

**All 9 green?** → Push this file to GitHub → Start Docker Session 01.

**Any red?** → Fix that step before moving on. Do not carry broken state into the next session.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| `docker ps` gives permission denied | ubuntu not in docker group | `sudo usermod -aG docker ubuntu` then `newgrp docker` |
| `git config --list` shows nothing | Git identity not configured | Run the `git config --global` commands above |
| Disk shows 8G not 28G | Storage not changed before launch | Terminate instance, relaunch with 30GB storage |
| SSH connection refused | Security group missing port 22 | AWS Console → SG → add inbound TCP 22 My IP |
| `ls ~/shopstack` fails | Clone failed or wrong directory | `cd ~` then `git clone` again |

---

## Quick Reference

| What | Command |
|---|---|
| Get EC2 public IP | `curl -s http://checkip.amazonaws.com` |
| SSH into EC2 | `ssh -i ~/.ssh/shopshack.pem ubuntu@YOUR_IP` |
| Check OS | `cat /etc/os-release` |
| Check RAM | `free -h` |
| Check disk | `df -h /` |
| Check Docker | `docker --version` |
| Check Docker running | `sudo systemctl status docker` |
| ShopStack location | `~/shopstack` |
| Compose file location | `~/shopstack/infra/docker-compose.yml` |
