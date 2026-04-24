[Linux](./01-linux.md) · [Git](./02-git.md) · [Networking](./03-networking.md) · [Docker](./04-docker.md) · [Kubernetes](./05-kubernetes.md) · [GitHub Actions](./06-github-actions.md) · [ArgoCD](./07-argocd.md) · [Observability](./08-observability.md) · [Terraform](./09-terraform.md) · [Ansible](./10-ansible.md) · [Bash](./11-bash.md)

---

# Linux

**Hire importance at $25–40/hr:** 9/10

**One line:** Every server you will ever SSH into runs Linux. No Linux fluency — no job.

📚 Full runbook → [Linux – System Fundamentals](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/README.md)

---

## 0 — Before You Open A Terminal

### What Linux actually is

Linux is the operating system kernel that everything runs on top of — Docker, Kubernetes, Nginx, Postgres, your scripts. It is not an app. It is the ground.

As a DevOps engineer you do not write Linux. You operate it. You SSH into it, read its logs, check its ports, fix its permissions, and manage the services running on top of it.

### The developer's contract with Linux

A developer writes Python, Go, or Node on their laptop. When that code goes to production it runs on Linux — inside a Docker container on an EC2 instance. The developer does not manage that Linux server. You do.

Their code reads from the filesystem, opens network ports, writes to stdout, and crashes with error messages. All of that happens inside Linux. Your job is to see exactly what the app is doing and why it is failing — even when the developer cannot.

### The one aha that prevents the biggest blind spot

**Everything in Linux is a file.**

Processes are files in `/proc`. Network sockets are files. Devices are files. Docker's socket is a file at `/var/run/docker.sock`. Config lives in `/etc`. Logs live in `/var/log`.

When this clicks, every Linux command makes sense. You are not memorising commands. You are learning to read files — which is all Linux ever does.

### ShopStack files to read before touching Linux

```
shopstack/scripts/setup.sh               ← Docker install script for EC2
shopstack/services/frontend/nginx.conf   ← config that permissions can break
/var/lib/docker/volumes/                 ← where Docker stores named volumes
/var/log/                                ← where all logs live on EC2
```

---

## 1 — The Stop Point

### 🧠 Stop here for $25–40/hr — own every concept below cold

- Linux is the OS under every container and every cloud server — you operate it, not write it
- The filesystem is a tree from `/` — everything hangs off root
- `/etc` is config · `/var` is logs and data · `/home` is users · `/tmp` is temporary · `/run` is live state wiped on reboot
- A process is a running program with a PID
- Every file has an owner, a group, and three permission sets — `rwx` for each
- `sudo` runs as root — root can do anything including destroy everything
- SSH uses public/private key pairs — the private key never leaves your machine
- Logs live in `/var/log` — you read them live and search them
- `ss -tulnp` shows every open port and what process owns it
- A full disk silently breaks services — you must know how to find and fix it

**When you can explain all ten without hesitation — stop. Move to Git.**

### What unlocks higher pay

At $55/hr you add: systemd unit file writing, advanced log analysis, `dmesg` for kernel-level debugging, performance tools like `htop` and `vmstat`, and security hardening with `ufw` and `fail2ban`.

At $80/hr you diagnose memory leaks, OOM kills, network saturation, and filesystem corruption from first principles.

📚 Deep dive → [Linux Runbook Overview](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/README.md)

---

## 2 — The Bag

### Own These Cold

Pure concepts. No syntax needed. These are what the interviewer tests and what rings the bell at 3am. When you can explain every one of these without hesitation — you own Linux at the $40/hr floor.

Linux is the OS under every container and every cloud server. You operate it — you do not write it.

Everything in Linux is a file. Processes, sockets, devices, logs, the Docker socket — all files. This is why every Linux command makes sense once you know it.

`/etc` is config. `/var` is logs and variable data. `/run` is live state wiped on every reboot. `/proc` is live process information.

A process is a running program with a PID. `ps aux` shows all of them.

`sudo` runs a command as root. Root can do anything — including destroy everything. Never run it blindly.

SSH uses a public/private key pair. The private key never leaves your machine. `ssh -i key.pem user@host` is how you get into EC2.

Logs live in `/var/log`. `tail -f` follows them live. `grep` searches them.

`ss -tulnp` shows every open port and what process owns it. This is your first move when a port is unreachable.

`df -h` shows disk usage. A full disk silently kills services — containers stop writing, Docker stops pulling images.

Permissions are `rwx` for owner, group, and others. `chmod` changes them. `ls -la` reads them.

📚 Deep dive → [File Ownership & Permissions](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/09-file-ownership-and-permissions/README.md)

---

### Know The Category

Concept lives in your brain. Situation triggers the right tool. Flags and exact syntax → combat sheet or AI.

- Disk full → `df` to confirm, `du` to find the culprit
- Port unreachable → `ss -tulnp` to check what is listening
- Process not responding → `ps aux` to find it, `kill` to stop it
- Log file needs watching → `tail -f`
- Permission denied → `ls -la` to read it, `chmod` to fix it
- Service not starting → `systemctl status` then `journalctl`
- Search text across files → `grep -r`

---

### Domain Awareness

These exist. You know they exist. You find them when needed — no shame, no rush.

- Log rotation — `logrotate`, compressed logs with `gzip`
- Advanced text parsing — `awk`, `sed` one-liners
- Archive and compress — `tar` with various flags
- Kernel messages — `dmesg` for hardware and boot issues
- Firewall rules — `iptables`, though AWS Security Groups handle this at your level

---

### The one insight that separates copiers from understanders

Linux does not hide information. It exposes everything through files.

When something breaks, a senior does not google first. They run `ls -la /var/run/docker.sock`, `systemctl status docker`, `journalctl -u docker -n 50`. Not special knowledge — just the habit of reading what Linux is already telling you.

---

## 3 — Combat Sheet

The combat sheet is not a memorisation list. It is your pressure-proof reference. When something breaks and your brain goes blank — open this, find the situation, run the command.

---

### Daily Drivers

These are the commands you will type every single working day. Drill these until your hands type them without thinking. After enough repetition they leave this table and live in your hands.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| Need to get into EC2 | `ssh -i key.pem user@host` | `ssh -i shopstack.pem ubuntu@EC2_IP` |
| Don't know what is listening on a port | `ss -tulnp \| grep PORT` | `ss -tulnp \| grep 8080` |
| Need to watch logs live | `tail -f /path/to/file` | `tail -f /var/log/nginx/access.log` |
| Need to search logs for a string | `grep -i "string" /path/to/file` | `grep -i "error" /var/log/syslog` |
| Need EC2 health at a glance | `uptime && df -h && free -h` | Run this every time you SSH in |

---

### Situational

You recognise the situation. Your brain reaches for the right category. You open this table and find the exact command. No shame — this is how seniors work too.

| What Breaks | Command | ShopStack Example |
|---|---|---|
| EC2 restarts — need new public IP | `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` | Run after every EC2 restart |
| Port reachable locally but not from browser | Check AWS Security Group inbound rules | Port 8080 not open in ShopStack SG |
| Need system-level Docker logs | `journalctl -u docker -f` | When container logs show nothing |
| Disk full — containers failing | `df -h` then `du -sh /var/lib/docker` | Docker images eating the disk |
| Process running or not | `ps aux \| grep PROCESS` | `ps aux \| grep nginx` |
| Find a string across all ShopStack files | `grep -r "STRING" ~/shopstack` | `grep -r "POSTGRES_PASSWORD" ~/shopstack` |
| File permission blocking a service | `ls -la /path` then `chmod +x /path` | `ls -la /var/run/docker.sock` |
| Script won't execute | `chmod +x script.sh` | `chmod +x ~/shopstack/scripts/setup.sh` |
| Force kill a stuck process | `kill -9 PID` | `ps aux \| grep nginx` → `kill -9 PID` |
| Service running but not responding | `journalctl -u SERVICE -n 50` | `journalctl -u docker -n 50` |
| Install a package on EC2 | `sudo apt update && sudo apt install -y pkg` | `sudo apt install -y docker-ce` |
| Copy a file to EC2 | `scp -i key.pem file user@host:/path` | `scp -i shopstack.pem setup.sh ubuntu@EC2:~` |

---

## 4 — ShopStack Map

| Situation | Command or File | Knowledge Needed |
|---|---|---|
| EC2 restarts — get new public IP | `curl -s http://169.254.169.254/latest/meta-data/public-ipv4` | AWS metadata service |
| Docker won't start after EC2 restart | `systemctl status docker` → `journalctl -u docker -n 50` | Service management |
| EC2 disk full — containers fail | `df -h` → `du -sh /var/lib/docker` → `docker system prune` | Disk usage commands |
| Port 8080 unreachable from browser | `ss -tulnp \| grep 8080` → check AWS Security Group | Port checking + firewall |
| Nginx cannot read its config | `ls -la ~/shopstack/services/frontend/nginx.conf` | File permissions |
| setup.sh fails to run | `chmod +x ~/shopstack/scripts/setup.sh` | Execute permission |
| Find where Postgres data lives on disk | `ls -la /var/lib/docker/volumes/infra_db-data/` | Docker volume filesystem path |
| Worker logs missing | `journalctl -u docker -f` then `docker compose logs worker` | System vs container logs |
| Need to edit nginx.conf on EC2 | `vim ~/shopstack/services/frontend/nginx.conf` | vim — open, edit, save, quit |
| Confirm Docker socket exists | `ls -la /var/run/docker.sock` | Everything is a file |

📚 Deep dive → [Linux Labs](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/README.md)

---

## 5 — Peace of Mind

> Answer every question out loud before opening the toggle.

> If you stumble — go to the terminal. Not back to the notes.

---

**Q1. You SSH into EC2 and want to know the health of the machine in 10 seconds. What three commands and what are you looking for in each?**

<details>
<summary>Answer</summary>

`uptime` — load average. If the three numbers are below the number of CPU cores, the machine is healthy. A spike means something is hammering the CPU.

`df -h` — disk usage. Above 90% means services will start failing to write logs and Docker will fail to pull images.

`free -h` — available memory. Near zero means the OOM killer may start terminating processes silently.

These three tell me if the machine itself is the problem before I look at any service.

</details>

---

**Q2. Port 8080 is not reachable from the browser but ShopStack is running. Walk me through your diagnosis.**

<details>
<summary>Answer</summary>

First: `ss -tulnp | grep 8080` — is anything actually listening on 8080. If nothing shows, the API container failed to bind the port.

Second: `docker compose ps` — is the API container healthy.

Third: `curl -I http://localhost:8080/api/health` — can the EC2 machine itself reach it. If yes locally but not from browser, the problem is the AWS Security Group — port 8080 is not open for inbound traffic.

The sequence: is it listening → is the container running → reachable locally → firewall blocking externally.

</details>

---

**Q3. What does `sudo` do and why would a command fail without it even though you are logged in as `ubuntu`?**

<details>
<summary>Answer</summary>

`sudo` runs a command as root — the superuser who can do anything. The `ubuntu` user is a normal user. Many system operations require root — installing packages, managing services, reading protected files. Without `sudo`, Linux denies the operation with "Permission denied" because the `ubuntu` user does not have the privilege. `sudo` temporarily elevates privilege for that one command only.

</details>

---

**Q4. The EC2 disk is full. Services are failing. Walk me through finding and fixing it.**

<details>
<summary>Answer</summary>

`df -h` — confirm which filesystem is full. On EC2 it is almost always `/`.

`du -sh /var/lib/docker` — Docker images, containers, and volumes are the most common culprit.

`du -sh /var/log/*` — if Docker is not it, unrotated log files are the next suspect.

Fix: `docker system prune` removes stopped containers, unused images, and dangling volumes. For logs: compress old ones with `find /var/log -name "*.log" -mtime +7 -exec gzip {} \;`.

Prevent it: run `docker system prune` weekly and configure log rotation.

</details>

---

**Q5. What is the difference between `/var/log` and `/run`?**

<details>
<summary>Answer</summary>

`/var/log` is persistent — survives reboots. It contains historical logs: Nginx access logs, Docker daemon logs, system logs. It grows over time and needs rotation.

`/run` is a RAM filesystem — wiped completely on every reboot. It contains live state that only makes sense while the system is running: PID files, socket files. Docker's socket lives at `/var/run/docker.sock` and is recreated fresh after every reboot.

Debug something from yesterday — `/var/log`. Check live state of a running service — `/run`.

</details>

---

**Q6. You need to find every file in ShopStack that contains `POSTGRES_PASSWORD`. What command?**

<details>
<summary>Answer</summary>

`grep -r "POSTGRES_PASSWORD" ~/shopstack`

`-r` means recursive — searches all files in all subdirectories. This finds every place that string appears: docker-compose.yml, env files, scripts. This is how you audit where secrets are referenced before changing them.

</details>

---

**Q7. What does `chmod +x setup.sh` do and when do you need it?**

<details>
<summary>Answer</summary>

It adds the execute bit — making the file runnable as a program. Without it, Linux refuses to run the script even if you know it is a shell script. Files created by `touch`, `echo`, or downloaded with `curl` do not have the execute bit by default. `ls -la` shows `x` when it is set: `-rwxr-xr-x` means executable, `-rw-r--r--` means not.

</details>

---

**Q8. A service is `active (running)` in `systemctl status` but not responding. What next?**

<details>
<summary>Answer</summary>

`journalctl -u SERVICE_NAME -n 50` — read the last 50 lines of that service's logs.

Running at the process level does not mean healthy. The process could be alive but stuck — deadlock, waiting for a dependency, or catching errors internally. The logs show what it is actually doing. The most recent 50 lines almost always contain the answer.

</details>

---

**Q9. What is the difference between `kill PID` and `kill -9 PID`?**

<details>
<summary>Answer</summary>

`kill PID` sends SIGTERM — a polite shutdown signal. The process can clean up gracefully before exiting: close file handles, finish in-flight requests, write final logs.

`kill -9 PID` sends SIGKILL — immediate forced termination. The kernel kills it instantly. No cleanup. Use this only when SIGTERM has been tried and the process is not responding. Last resort, not first.

</details>

---

**Q10. EC2 restarts and gets a new IP. What single command gives you the new public IP from inside EC2?**

<details>
<summary>Answer</summary>

`curl -s http://169.254.169.254/latest/meta-data/public-ipv4`

`169.254.169.254` is the AWS instance metadata service — a special IP that only exists inside an EC2 instance and always returns information about that instance. `-s` suppresses curl's progress output so only the IP prints. More reliable than `ip addr` which shows only the private IP.

</details>

---

## 6 — AI Split

### 🧠 Learn this yourself — never outsource

- The filesystem hierarchy — `/etc`, `/var`, `/run`, `/proc` — interviewers test this
- How permissions work — reading `ls -la` output cold, `rwx` and octal
- What `sudo` does and why — tested in every junior interview
- Reading `ss -tulnp` output — LISTEN state, process column
- SSH key auth — how it works, where keys live, why more secure than passwords
- The disk full diagnosis sequence — this happens in real production

### 📖 Use AI for this — guilt free

- One-liners to extract specific fields from log files
- Exact flags for `tar`, `find`, `grep` when you know what you want but not the syntax
- `sed` or `awk` commands for specific text transformations
- `systemd` unit file templates for new services
- Exact `chmod` octal numbers for specific permission combinations

### The right questions to ask AI for Linux

```
"I am on Ubuntu 22.04 EC2. I need to find all files in /var/log
modified in the last 24 hours larger than 100MB.
Give me the exact find command and explain each flag."

"I have this ss -tulnp output: [paste output].
Which process is listening on port 8080 and
what does the LISTEN state mean?"

"I need to add the ubuntu user to the docker group on EC2
so I can run docker without sudo.
Exact command — and do I need to log out and back in?"

"I see this error in journalctl: [paste error].
What does this mean and what are the two most likely causes
on an Ubuntu EC2 running Docker?"
```

📚 Full Linux runbook → [Linux – System Fundamentals](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/README.md)
