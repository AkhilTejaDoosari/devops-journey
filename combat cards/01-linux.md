# 🐧 LINUX_CORE — Combat Card
> **80/20 Rule:** Own every command on this page. Everything else is in the runbook.   
> **Pre-shift scan:** 60 seconds. Navigation → Permissions → Services → Troubleshoot.   
> **Source:** [01. Linux](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/README.md)   
---

## 1. NAVIGATION — Land and Orient
> **Source:** [02-basics](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/02-basics/README.md)   
> First 30 seconds after every SSH login. Run these before you touch anything.   

| Command | What it does |
|---|---|
| `pwd` | Where am I? Run this first. Always. |
| `ls -lahtr` | All files, human-readable sizes, newest at bottom |
| `cd <path>` | Move into a directory |
| `cd ~` | Jump to home — escape hatch when lost |
| `cd -` | Jump back to previous directory |
| `mkdir -p <path>` | Create nested directories in one shot |
| `whoami` | What user am I running as? |
| `id` | My UID, GID, and every group I belong to |
| `uptime` | Is this machine alive? What's the load? |

**ShopStack context:** After SSH into EC2, run `cd ~/shopstack/infra/` to get to the Compose file.

---

## 2. FILE OPERATIONS — Safe Moves
> **Source:** [03-working-with-files](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/03-working-with-files/README.md)

| Command | What it does |
|---|---|
| `cat <file>` | Print full file to terminal |
| `less <file>` | Scroll through large files (q to quit) |
| `tail -f <file>` | Follow a file live — new lines appear as written |
| `head -n 20 <file>` | Show first 20 lines |
| `cp -riv <src> <dest>` | Copy directory — verbose + confirm |
| `mv -iv <src> <dest>` | Move/rename — verbose + confirm |
| `rm -i <file>` | Delete with confirmation prompt |
| `touch <file>` | Create empty file or update timestamp |

---

## 3. PERMISSIONS — Fix Access Denials Fast
> **Source:** [09-file-ownership-and-permissions](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/09-file-ownership-and-permissions/README.md)   
> nginx can't read config? Script won't execute? This section.

| Command | What it does |
|---|---|
| `ls -lah <dir>` | See permissions, owner, group — all files including hidden |
| `chmod 640 <file>` | Config files: owner=rw, group=r, others=nothing |
| `chmod 755 <dir>` | Directories: owner=rwx, group=rx, others=rx |
| `chmod u+x <file>` | Add execute for owner only — use on scripts |
| `chmod -R 755 <dir>` | Apply permissions recursively |
| `sudo chown user:group <file>` | Change who owns a file |
| `sudo chown -R user:group <dir>` | Change ownership recursively |

**The octal cheat:**
```
r=4  w=2  x=1   →   add them per group
640 = Owner:rw(6)  Group:r(4)  Others:none(0)   ← config files
755 = Owner:rwx(7) Group:rx(5) Others:rx(5)     ← directories
700 = Owner:rwx(7) Group:none  Others:none       ← private scripts
```

**ShopStack context:** `webstore.conf` should be `640`. Deploy scripts should be `700`. `infra/` directory should be `755`.

---

## 4. SERVICES — Manage nginx, Docker, Everything
> **Source:** [12-service-management](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/12-service-management/README.md)   
> The service lifecycle. Know this order cold.

| Command | What it does |
|---|---|
| `sudo systemctl status <svc>` | Is it running? Last start time? Recent log lines? |
| `sudo systemctl start <svc>` | Start now |
| `sudo systemctl stop <svc>` | Stop now |
| `sudo systemctl restart <svc>` | Kill and start fresh — drops active connections |
| `sudo systemctl reload <svc>` | Apply new config — zero downtime (nginx supports this) |
| `sudo systemctl enable --now <svc>` | Start NOW + survive reboots |
| `systemctl is-enabled <svc>` | Will it start on reboot? |
| `systemctl list-units --state=failed` | Show every broken service — run this after incidents |

**The nginx rule:** Always `sudo nginx -t` before `reload`. Never `restart` on prod unless `reload` fails.

---

## 5. TROUBLESHOOTING — The 2AM Drill
> **Source:** [14-logs-and-debug](https://github.com/AkhilTejaDoosari/devops-runbook/blob/main/notes/01.%20Linux%20%E2%80%93%20System%20Fundamentals/14-logs-and-debug/README.md)   
> Run in this exact order when something breaks.

```
Step 1: uptime                              → Is the machine alive? Load normal?
Step 2: df -h                               → Is a disk full? (full disk = silent failures)
Step 3: systemctl list-units --state=failed → Did a service die?
Step 4: journalctl -u <svc> -n 50          → What did it say when it failed?
Step 5: sudo ss -tlnp | grep <port>         → Is anything even listening?
```

| Command | What it does |
|---|---|
| `journalctl -u <svc> -n 50` | Last 50 log lines — first stop in every debug session |
| `journalctl -u <svc> -f` | Follow live logs — watch it fail in real time |
| `journalctl -b -p err` | All errors since last boot |
| `journalctl -b -1` | All logs from the previous boot — use after a crash |
| `df -h` | Disk usage on all filesystems |
| `du -sh /var/log/* \| sort -rh \| head -10` | Find the log file eating all your space |
| `tail -f /var/log/nginx/error.log` | Watch nginx errors live |
| `grep "$(date +'%b %e')" /var/log/syslog \| grep -i error` | Today's errors in syslog |

**ShopStack context:** `journalctl -u docker -n 50` when Compose services won't start. `df -h` when containers exit with no error message.

---

## ⚡ MASTER ONE-LINERS

```bash
# System health in 3 commands
uptime && df -h && systemctl list-units --state=failed

# Service broken — read the evidence
journalctl -u <svc> -n 50

# Permission denied on a file
ls -lah <file>   # read the rwx bits, then chmod/chown to fix

# Disk full — find the culprit
du -sh /var/log/* | sort -rh | head -10
```

---

> **Own this page. The rest is details.**
