# Day 5 — Test Yourself

No notes. No checklist. Answer out loud or write it down first. Then check.

**Nav:** [1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md)

---

## Round 1 — Concepts
*30 seconds each. Why and what — no commands yet.*

**Q1.** What is a container registry and what problem does it solve?

<details><summary>Answer</summary>

A container registry is a remote storage system for Docker images. Without it, images only exist on one machine — no CI system, no production server, and no teammate can access them. With a registry, you push once and any machine with access can pull. Docker Hub is the public default. ECR, GCR, and ACR are cloud-managed private registries.

</details>

---

**Q2.** You have two image names on your machine: `infra-api` and `shopstack-api`. Both were built from the same Dockerfile. Which one do you push to DockerHub and why?

<details><summary>Answer</summary>

`shopstack-api` — it's the image you deliberately named and own. `infra-api` is the name Compose auto-generated when it built the same Dockerfile. They may contain the same code but `shopstack-api` is the one you control and version. You push what you own, not what Compose auto-named. Always verify IMAGE IDs match after tagging to confirm you tagged the right source.

</details>

---

**Q3.** You run `docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0`. Does this copy the image? What does `docker images` show after?

<details><summary>Answer</summary>

No copy. `docker tag` creates a second label pointing to the exact same image data. `docker images` shows two rows — `shopstack-api` and `akhiltejadoosari/shopstack-api:1.0` — with identical IMAGE IDs. One image, two names.

</details>

---

**Q4.** You delete `akhiltejadoosari/shopstack-api:1.0` first, then `shopstack-api:latest`. The first `docker rmi` says `Untagged` only. The second says `Untagged` + `Deleted`. Why the difference?

<details><summary>Answer</summary>

Docker only deletes image data when the last tag pointing to it is removed. When you deleted `akhiltejadoosari/shopstack-api:1.0`, the tag `shopstack-api:latest` still pointed to the same IMAGE ID — so Docker removed the label but kept the data. When you deleted `shopstack-api:latest`, that was the last tag. No tags left = no reason to keep the data. Docker deleted the layers. The order doesn't matter — the last tag always triggers the delete.

</details>

---

**Q5.** Why is using `latest` in production a bad practice?

<details><summary>Answer</summary>

`latest` means "whatever was pushed most recently" — not "the most stable version." It is mutable. If something breaks in production you cannot roll back because you don't know exactly which image `latest` was pointing to when it was deployed. Real pipelines use Git SHA tags (`a3f92c1`) for CI builds and semantic version tags (`v1.0.0`) for releases.

</details>

---

**Q6.** You pull `akhiltejadoosari/shopstack-api:1.0` and run it standalone. The logs show it tried to reach `db` 12 times and failed with `Name or service not known`. Is this a registry problem? What actually went wrong?

<details><summary>Answer</summary>

Not a registry problem. The image pulled and ran correctly — the registry did its job. The failure is a networking problem. The container was started with plain `docker run` and landed on Docker's default bridge network. `infra-db-1` is on the Compose-created `backend` network. They are on different networks so the DNS name `db` doesn't resolve. The fix is `docker compose up` — which puts all services on the correct networks automatically. This is what Day 6 solves.

</details>

---

## Round 2 — Commands
*Recall cold. ShopStack names only. No looking.*

**Q7.** Authenticate to DockerHub from the terminal.

<details><summary>Answer</summary>

```bash
docker login
```

</details>

---

**Q8.** Confirm which DockerHub account you are currently logged into.

<details><summary>Answer</summary>

```bash
docker info | grep -i username
```

</details>

---

**Q9.** Tag `shopstack-api` for DockerHub under your account as version 1.0. Write the exact command — pay attention to the source image name.

<details><summary>Answer</summary>

```bash
docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0
```

Source is `shopstack-api` — not `infra-api`. Always run `docker images` after and confirm IMAGE IDs match.

</details>

---

**Q10.** After tagging, how do you confirm the original and the DockerHub tag point to the same image?

<details><summary>Answer</summary>

```bash
docker images
```

Check the IMAGE ID column — `shopstack-api` and `akhiltejadoosari/shopstack-api:1.0` must show identical IDs.

</details>

---

**Q11.** Push all three ShopStack images to DockerHub.

<details><summary>Answer</summary>

```bash
docker push akhiltejadoosari/shopstack-api:1.0
docker push akhiltejadoosari/shopstack-frontend:1.0
docker push akhiltejadoosari/shopstack-worker:1.0
```

`docker push` takes one image at a time — no batching.

</details>

---

**Q12.** Delete all six local ShopStack tags in two commands.

<details><summary>Answer</summary>

```bash
docker rmi akhiltejadoosari/shopstack-api:1.0 akhiltejadoosari/shopstack-frontend:1.0 akhiltejadoosari/shopstack-worker:1.0
docker rmi shopstack-api shopstack-frontend shopstack-worker
```

</details>

---

**Q13.** Pull `shopstack-api:1.0` back from DockerHub.

<details><summary>Answer</summary>

```bash
docker pull akhiltejadoosari/shopstack-api:1.0
```

</details>

---

**Q14.** Run the pulled api image detached, name it `api-test`, host port 8090, container port 8080.

<details><summary>Answer</summary>

```bash
docker run -d --name api-test -p 8090:8080 akhiltejadoosari/shopstack-api:1.0
```

Host port is your choice. Container port is fixed — the app listens on 8080 inside.

</details>

---

**Q15.** You try to run `api-test` but get: `The container name "/api-test" is already in use`. What happened and what do you run?

<details><summary>Answer</summary>

A previous `docker run` attempt created the container but it failed before starting — it exists in `Created` state. Docker won't reuse the name.

Fix:
```bash
docker rm api-test
docker run -d --name api-test -p 8090:8080 akhiltejadoosari/shopstack-api:1.0
```

</details>

---

**Q16.** Check whether `api-test` started, then stop and remove it in one line.

<details><summary>Answer</summary>

```bash
docker logs api-test
docker stop api-test && docker rm api-test
```

</details>

---

> 🔧 Environment Setup — Not a DevOps Skill
> If the three ShopStack images do not exist from Day 4, rebuild them before attempting Round 2 commands.
> Commands are provided — copy and paste.
> ```bash
> cd ~/shopstack
> docker build -t shopstack-api ./services/api
> docker build -t shopstack-frontend ./services/frontend
> docker build -t shopstack-worker ./services/worker
> ```
> ↩️ Back to DevOps

---

## Round 3 — Scenario Debug
*ShopStack is broken. Diagnose and fix.*

**Scenario A**

You run:
```
docker push akhiltejadoosari/shopstack-api:1.0
```
And get:
```
denied: requested access to the resource is denied
```
You are sure you're logged in. What else could cause this and what is your exact fix sequence?

<details><summary>Answer</summary>

The tag prefix doesn't match your logged-in account. Even if you're logged in, DockerHub rejects pushes where the username in the tag doesn't match the authenticated account.

Fix sequence:
```bash
docker info | grep -i username
docker logout
docker login
docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0
docker push akhiltejadoosari/shopstack-api:1.0
```

</details>

---

**Scenario B**

You run `docker run -d --name api-test -p 8080:8080 akhiltejadoosari/shopstack-api:1.0` and get:
```
Bind for 0.0.0.0:8080 failed: port is already allocated
```
What owns port 8080 and what is the correct fix?

<details><summary>Answer</summary>

`infra-api-1` — the ShopStack Compose service — already binds host port 8080. You can't bind two containers to the same host port.

Fix — use a different host port, keep container port fixed at 8080:
```bash
docker run -d --name api-test -p 8090:8080 akhiltejadoosari/shopstack-api:1.0
```

Host port is yours to choose. Container port 8080 is fixed — that's what uvicorn listens on inside.

</details>

---

**Scenario C**

You run `docker rmi shopstack-api` and get:
```
Error response from daemon: conflict: unable to delete — image is referenced in multiple repositories
```
What is blocking you and what do you do?

<details><summary>Answer</summary>

Another tag still points to the same IMAGE ID — likely `akhiltejadoosari/shopstack-api:1.0`. Docker won't delete image data while any tag references it.

Fix — delete the other tag first:
```bash
docker rmi akhiltejadoosari/shopstack-api:1.0
docker rmi shopstack-api
```

Run `docker images` first to see all tags sharing that IMAGE ID before deciding the order.

</details>

---

**Scenario D**

A teammate says "I pulled `akhiltejadoosari/shopstack-api:1.0` and ran it but it keeps retrying the database and then crashes." DockerHub shows the image is there. What do you tell them?

<details><summary>Answer</summary>

The registry is fine. The image pulled and ran — registry did its job. The problem is networking. They ran the container with plain `docker run` so it landed on Docker's default bridge network. `infra-db-1` is on the Compose `backend` network. Different networks = DNS name `db` doesn't resolve.

The fix is `docker compose up` — Compose puts all services on the correct networks and wires DNS automatically. You cannot fix this with a `docker run` flag alone.

</details>

---

## Round 4 — Reading Output
*Real terminal output. Tell me what it means.*

**Q17.** You run `docker images` and see:

```
REPOSITORY                           TAG      IMAGE ID       SIZE
infra-api                            latest   2d4e9b8d42a2   288MB
akhiltejadoosari/shopstack-api       1.0      cb1c4718b4bc   288MB
shopstack-api                        latest   cb1c4718b4bc   288MB
```

Three questions: (a) Which image should you push and why? (b) What does the IMAGE ID column tell you about rows 2 and 3? (c) What is wrong with row 1?

<details><summary>Answer</summary>

(a) `akhiltejadoosari/shopstack-api:1.0` — already correctly formatted for DockerHub with your username and version tag.

(b) Rows 2 and 3 share IMAGE ID `cb1c4718b4bc` — they point to the same image data. `docker tag` created the second label, no copy was made.

(c) `infra-api` has a different IMAGE ID `2d4e9b8d42a2` — it's a different image, auto-built by Compose. Tagging and pushing `infra-api` would push the wrong source. Always verify IMAGE IDs match before pushing.

</details>

---

**Q18.** You run `docker push akhiltejadoosari/shopstack-api:1.0` and see:

```
67ff0b34993a: Layer already exists
ed6e65697b39: Layer already exists
3fc9d9ab5045: Pushed
1.0: digest: sha256:cb1c47... size: 856
```

What does `Layer already exists` mean and why does it matter?

<details><summary>Answer</summary>

DockerHub already has those layers from a previous push — likely the Alpine base or pip dependencies shared with another image. Docker skips re-uploading them. Only the changed layer transferred. This is layer deduplication — the same system that makes Dockerfile layer ordering matter. Shared layers are never uploaded twice, making pushes faster and storage more efficient.

</details>

---

**Q19.** You run `docker rmi akhiltejadoosari/shopstack-api:1.0` and see:

```
Untagged: akhiltejadoosari/shopstack-api:1.0
```

No `Deleted` line. What does that tell you and what do you check next?

<details><summary>Answer</summary>

Another tag still points to the same IMAGE ID — most likely `shopstack-api:latest`. Docker removed the label but kept the image data because it's still referenced. Run `docker images` to see all tags sharing that IMAGE ID. The `Deleted` line only appears when the last tag is removed.

</details>

---

**Q20.** You run `docker logs api-test` and see:

```
INFO:     Started server process [1]
{"event": "db_connect_attempt", "attempt": 1, "host": "db"}
{"event": "db_connect_failed", "attempt": 1, "error": "[Errno -2] Name or service not known"}
...
{"event": "db_connect_exhausted", "max_attempts": 12}
ERROR:    Application startup failed. Exiting.
```

Four questions: (a) Did the container start? (b) Did the registry work? (c) What failed and why? (d) What is the fix?

<details><summary>Answer</summary>

(a) Yes — `Started server process [1]` confirms the container started and the app launched correctly.

(b) Yes — the image pulled and the container ran. The registry worked perfectly.

(c) DNS resolution for `db` failed. `api-test` is on Docker's default bridge network. `infra-db-1` is on the Compose `backend` network. Different networks = no DNS. The container retried 12 times and gave up.

(d) `docker compose up` — Compose puts all services on the correct networks and wires DNS automatically. Day 6.

</details>

---

**Session win condition:** All 3 ShopStack images on DockerHub at tag 1.0. Pull and run loop completed. You can explain every command, every error, and the network isolation problem without notes.
