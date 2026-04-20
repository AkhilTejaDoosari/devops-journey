# Day 1 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Check answers in Test-Answers.md only after you have answered.

---

## Round 1 — Concepts

Q1. What is the difference between an image and a container?

Q2. You run `docker ps` after exiting a container and it is not there. Where did it go?

Q3. You created a file inside a container, stopped it, started it again. Is the file still there? Why?

Q4. What does `-d` do in `docker run -d`?

Q5. What does `0.0.0.0:8090->80/tcp` mean in `docker ps` output?

Q6. What is the correct order to clean up Docker? Why that order?

Q7. Your current directory is `/home/ubuntu/shopstack/infra`. You run `docker images`. What do you see?

Q8. What is the difference between `docker ps` and `docker ps -a`?

---

## Round 2 — Commands

Q9. Pull the ubuntu image without a tag.

Q10. Pull ubuntu version 22.04 specifically.

Q11. Run a container named `test-box` from ubuntu:22.04 and enter it interactively.

Q12. Re-enter a stopped container named `test-box`.

Q13. Run nginx:1.24 in the background, name it `web`, bind EC2 port 8090 to container port 80.

Q14. Follow the logs of a running container named `web` live.

Q15. Enter a running container named `web` with a shell.

Q16. Find the IP address of container `web`.

Q17. Find the port mapping of container `web` with 5 lines of context.

Q18. Stop, delete the container, then delete the image — correct order.

---

## Round 3 — Scenario Debug

**Scenario A**
You run `docker run -d --name api node:18` and immediately check `docker ps` — the container is not there. You check `docker ps -a` and see it with status `Exited (1)`. What happened and what is the first command you run?

**Scenario B**
You run nginx with `-p 8090:80` and confirm it shows in `docker ps` with the correct port. You open the browser and get connection refused. What are the two most likely causes and how do you check each?

**Scenario C**
You try to delete an image with `docker rmi nginx:1.24` and get:
```
Error response from daemon: conflict: unable to delete — container is using its referenced image
```
What do you do and in what order?

**Scenario D**
Your team says "the container is running fine." You open the app and it is down. What is the first thing you check and what command do you run?

---

## Round 4 — Reading Output

Q19. You run `docker images` and see two rows for ubuntu — one tagged `latest` and one tagged `22.04`. Are they the same image?

Q20. You run `docker inspect web | grep "IPAddress"` and see `172.17.0.3`. What is that IP and can you reach it from your Mac browser?

Q21. You run `docker logs web` and see:
```
173.17.106.24 - - [19/Apr/2026] "GET / HTTP/1.1" 200 615
```
What does this tell you?

---

## Redo Checklist — Do This Without Notes

1. Pull ubuntu:22.04 and nginx:1.24
2. Run ubuntu:22.04 interactively, create a file, exit
3. Confirm the container is stopped but still exists
4. Re-enter it — confirm the file is still there
5. Run nginx:1.24 in background on port 8090
6. Confirm it is running and open it in your browser
7. Check the logs and follow them live
8. Inspect the container — find IP and port mapping
9. Enter the nginx container — look at `/usr/share/nginx/html/`
10. Clean up — stop, rm containers, rmi images
11. Confirm `docker ps -a` is empty and `docker images` shows only ShopStack images

Done when: All 11 steps with zero help.
