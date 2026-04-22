# Day 2 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Check answers in Test-Answers.md only after you have answered.

---

## Round 1 — Concepts

Q1. What does `-p 8080:80` mean? Which side is the host and which is the container?

Q2. Two containers are running from the same nginx:1.24 image — one on port 8080, one on 8081. Are they the same container?

Q3. You run a postgres container with no `-p` flag. Can you reach it from your browser? Can another container on the same network reach it?

Q4. What is Docker DNS? When does it work and when does it not work?

Q5. You have two containers on different networks. Can they ping each other by name? Why?

Q6. What is `127.0.0.11` inside a container?

Q7. In ShopStack, why does the API container have two IP addresses?

Q8. In ShopStack, the worker has no port binding at all. Why not?

---

## Round 2 — Scenario Debug

**Scenario A**
Your app container is trying to connect to `webstore-db` but gets `bad address 'webstore-db'`. The database container is running. What is the most likely cause and how do you confirm it?

**Scenario B**
You run `docker run -p 8080:80 nginx` and get:
```
Bind for 0.0.0.0:8080 failed: port is already allocated
```
What does this mean and what command do you run first to debug it?

**Scenario C**
Adminer can reach `webstore-db` by name. You try to reach `webstore-db` from your browser directly and it fails. Is something broken?

---

## Round 3 — Commands

Q9. Create a network named `app-network`.

Q10. Run postgres:15 named `app-db` on `app-network` with DB=myapp, user=admin, password=secret.

Q11. Show all containers on `app-network` and their IPs with one command.

Q12. From inside a container, how do you confirm which DNS server Docker assigned?

Q13. Clean up — stop and remove containers, remove network, remove images. What is the correct order?

---

## Redo Checklist — Do This Without Notes

1. Pull nginx:1.24 and postgres:15
2. Run two nginx containers on different host ports — confirm both load in browser
3. Stop and remove both
4. Create a network
5. Run postgres on the network — no port binding
6. Run adminer on the network — port 8081
7. Login to adminer using the db container name as server
8. Enter adminer container — ping the db by name — confirm DNS resolves
9. Check `/etc/resolv.conf` — confirm `127.0.0.11`
10. Inspect the network — read both container IPs
11. Clean everything — containers, network, images

Done when: You complete all 11 steps with zero help.
