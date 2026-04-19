# Day 1 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Check answers in TEST_ANSWERS.md only after you have answered.

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

Q9. How do you pull the ubuntu image without specifying a tag?

Q10. How do you pull ubuntu version 22.04 specifically?

Q11. How do you run a container named `test-box` from ubuntu:22.04 and enter it interactively?

Q12. How do you re-enter a stopped container named `test-box`?

Q13. How do you run nginx:1.24 in the background, name it `web`, and bind EC2 port 8090 to container port 80?

Q14. How do you follow the logs of a running container named `web` live?

Q15. How do you enter a running container named `web` with a shell?

Q16. How do you find the IP address of a container named `web` using one command?

Q17. How do you find the port mapping of a container named `web` with context?

Q18. How do you stop, delete the container, then delete the image — in the correct order?

---

## Round 3 — Reading Output

Q19. You run `docker images` and see two rows for ubuntu — one tagged `latest` and one tagged `22.04`. Are they the same image?

Q20. You run `docker inspect web | grep "IPAddress"` and see `172.17.0.3`. What is that IP and can you reach it from your Mac browser?

Q21. You run `docker logs web` and see a line:
```
173.17.106.24 - - [19/Apr/2026] "GET / HTTP/1.1" 200 615
```
What does this tell you?

---

## Redo Checklist — Do This Without Notes

Go through these in order. No looking at the checklist.

1. Pull ubuntu:22.04 and nginx:1.24
2. Run ubuntu:22.04 interactively, create a file with text in it, exit
3. Confirm the container is stopped but exists
4. Re-enter it and confirm the file is still there
5. Exit and run nginx:1.24 in background on port 8090
6. Confirm it is running and open it in your browser
7. Check the logs and follow them live
8. Inspect the container — find IP and port mapping using grep
9. Enter the nginx container and look at /usr/share/nginx/html/
10. Clean up everything — stop, rm containers, rmi images
11. Confirm docker ps -a is empty and docker images shows only ShopStack images

Done when: You completed all 11 steps with zero help.
