# Day 5 — Registry

**Date:** April 25 2026   
**Read before session:** [09-docker-registry](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/09-docker-registry)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Tag the three ShopStack images you built yesterday, push them to DockerHub, delete them locally, and prove the registry works by pulling and running from scratch.   

**Nav:** [1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md)

---

## 3-Minute Foundation Warm-up

Before we start — one question from a previous topic:

> **Yesterday you wrote a multi-stage Dockerfile for the worker.** What is the *one reason* you need two FROM statements instead of one? Answer out loud or in writing. Then start the session.

---

## Knowledge — What This Topic Is Really About

### The Problem

Yesterday you built three images: shopstack-api, shopstack-frontend, shopstack-worker. They live on your EC2. Only your EC2.   
A teammate's machine — no images. A CI server — no images. A fresh EC2 in production — no images. Any machine that isn't yours has to rebuild from scratch, and there's no guarantee it builds the same way.   
This is the problem a registry solves. Push once. Any machine pulls. Same image, everywhere, no rebuild drift.   

### What a Registry Actually Is

Think of it like GitHub, but for images instead of code.

GitHub stores your source code so any machine can clone it. DockerHub stores your images so any machine can pull them. The workflow is the same:

```
GitHub:  git commit + git push  →  repo is available everywhere
DockerHub: docker build + docker push  →  image is available everywhere
```

You push once. Anyone with access can pull. No rebuild, no drift, no "it worked on my machine."

**The core rule:** A registry is passive storage. It does not run anything. Developers push to it. CI pulls from it. Production pulls from it. The registry just holds the image.

### The Only Flow That Matters

```
You (EC2)  →  docker push  →  DockerHub  →  docker pull  →  Any Machine
```

Today you walk this entire loop for all three ShopStack services.

### Tags — The Version System

A tag is a label on an image. Without tags, you cannot distinguish `shopstack-api` built today from `shopstack-api` built three weeks ago after someone broke something.

The naming rule for DockerHub is strict:

```
USERNAME/IMAGE:TAG
akhiltejadoosari/shopstack-api:1.0
      ↑                ↑         ↑
  your account     image name   version
```

The `docker tag` command does not copy or rename the image. It creates a second label pointing to the exact same image data. One image, two names. Pushing by either name uploads the same bytes.

### Why Not Just Use `latest`?

`latest` is the default tag when you do not specify one. It sounds useful — it sounds like "the newest stable version." It is not. `latest` means "whatever was pushed most recently." There is no stability guarantee, no way to roll back to a specific version, and no way to know what is actually running in production.

In real teams: **`latest` is banned in CI/CD pipelines.** Every image gets a version tag — either semantic (`1.0`, `1.1.2`) or a Git SHA (`a3f92c1`). Today you use `1.0` — the simplest stable tag.

### ShopStack Connection

This is Step 5 of the DevOps engineer flow from your 60-day plan: "Push images to DockerHub." The images you push today are what Kubernetes will pull in Week 2. K8s does not run `docker run` — it pulls from a registry. What you do today is the exact pattern production uses.

---

# 📟 DAY 5 FULL CHEATSHEET — Registry

## Authentication

| Command | What it does | Example | Label |
|---|---|---|---|
| `docker login` | Authenticate to DockerHub — stores creds in OS keychain | `docker login` | [CORE] |
| `docker logout` | Remove stored credentials | `docker logout` | [CORE] |
| `docker info \| grep -i username` | Confirm which account you are logged into | `docker info \| grep -i username` | [CORE] |

**Syntax breakdown — why grep `-i`:**
```
docker info | grep -i username
                        ↑
               -i = case-insensitive match
               catches "Username", "username", "USERNAME" — all the same
```

---

## Tagging

| Command | What it does | Example | Label |
|---|---|---|---|
| `docker tag SOURCE TARGET` | Create a new name pointing to the same image | `docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0` | [CORE] |
| `docker images` | List all local images and tags — confirm both names exist | `docker images` | [CORE] |

**Syntax breakdown — docker tag:**
```
docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0
           ↑              ↑               ↑              ↑
        SOURCE          USERNAME       REPO NAME       TAG
     (local name)    (DockerHub)    (image name)   (version)
```

> ⚠️ `docker tag` does not copy or move the image. It adds a second label. Both names point to the same image ID. You can confirm with `docker images` — the IMAGE ID column will be identical.

---

## Push and Pull

| Command | What it does | Example | Label |
|---|---|---|---|
| `docker push USERNAME/IMAGE:TAG` | Upload image layers to registry | `docker push akhiltejadoosari/shopstack-api:1.0` | [CORE] |
| `docker pull USERNAME/IMAGE:TAG` | Download image from registry | `docker pull akhiltejadoosari/shopstack-api:1.0` | [CORE] |

**What push actually does — layer deduplication:**
```
Pushing akhiltejadoosari/shopstack-api:1.0
  Layer 1 (alpine base)  → Already exists in DockerHub — SKIPPED
  Layer 2 (pip install)  → Already exists — SKIPPED
  Layer 3 (COPY src)     → New — UPLOADING...
  Layer 4 (CMD)          → New — UPLOADING...

Result: Only changed layers upload. Shared base layers are reused.
```

This is why the first push is slower. Every subsequent push after a small code change is fast — only the changed layers move.

---

## Delete Local Images

| Command | What it does | Example | Label |
|---|---|---|---|
| `docker rmi IMAGE:TAG` | Delete one local image tag | `docker rmi shopstack-api` | [CORE] |
| `docker rmi IMAGE1 IMAGE2 IMAGE3` | Delete multiple images in one command | `docker rmi shopstack-api shopstack-frontend shopstack-worker` | [CORE] |

> ⚠️ `docker rmi` deletes the local copy only. The registry is not affected. This is how you prove the registry works — delete locally, pull back, confirm it runs.

---

## Run a Pulled Image

| Command | What it does | Example | Label |
|---|---|---|---|
| `docker run -d -p HOST:CONTAINER --name NAME IMAGE:TAG` | Run a pulled image as a named detached container | `docker run -d -p 8080:8080 --name api-test akhiltejadoosari/shopstack-api:1.0` | [CORE] |
| `docker logs NAME` | Confirm the container started correctly | `docker logs api-test` | [CORE] |
| `docker stop NAME && docker rm NAME` | Stop and remove the test container | `docker stop api-test && docker rm api-test` | [CORE] |

---

## The Full Day 5 Workflow (Mental Map)

```
Step 1: docker login          ← prove you own the account
Step 2: docker tag            ← rename image to USERNAME/NAME:VERSION
Step 3: docker images         ← confirm both tags exist, same IMAGE ID
Step 4: docker push           ← upload to DockerHub
Step 5: hub.docker.com        ← verify in browser, tag 1.0 visible
Step 6: docker rmi            ← delete local copies (all 6 tags)
Step 7: docker images         ← confirm clean
Step 8: docker pull           ← download back from registry
Step 9: docker run            ← prove it runs
Step 10: docker logs          ← confirm it started
Step 11: docker stop && rm    ← clean up
```

---

## Common Failure Modes

| Symptom | Cause | Fix | Label |
|---|---|---|---|
| `denied: requested access to the resource is denied` | Wrong DockerHub username in tag, or not logged in | `docker logout` → `docker login` → retag with correct username | [CORE] |
| `unauthorized: authentication required` | Credentials expired or never set | `docker login` — re-authenticate | [CORE] |
| `tag does not exist` on pull | Tag was never pushed, or wrong tag name used | `docker images` to see local tags → check DockerHub UI for what was pushed | [CORE] |
| Push succeeds but Kubernetes can't pull | Image is private, no pull secret configured | Set repo to Public on DockerHub for now | [WIKI] |
| Every push re-uploads all layers | Base image tag changed (`node:20` resolved to new digest) | Pin base image to a specific version like `node:20.11.0-alpine` | [WIKI] |

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | Fix |
|---|---|---|
| Push fails with `denied` | You are logged in as the wrong account — tag prefix must match your DockerHub username exactly | `docker info \| grep -i username` → confirm account → `docker logout` → `docker login` → `docker tag` again with correct username |
| `docker rmi` fails — image still has dependents | Stopped containers still reference the old tag | `docker ps -a` → `docker rm` the containers → then `docker rmi` |
| Pulled image runs but crashes immediately | The image was pushed broken, or the env vars it needs are missing | `docker logs api-test` — read the error — it likely needs `DATABASE_URL` or similar env var to start |
| `docker images` shows 6 tags but IMAGE IDs do not match | You tagged the wrong source image | `docker images` — compare IMAGE ID column — original and tagged version must be identical |

---

## Checklist

> 🧪 Practice Only — Not a DevOps Skill
> The three ShopStack images (shopstack-api, shopstack-frontend, shopstack-worker) must exist locally from Day 4.
> If they are missing, rebuild them now:
> ```bash
> cd ~/shopstack
> docker build -t shopstack-api ./services/api
> docker build -t shopstack-frontend ./services/frontend
> docker build -t shopstack-worker ./services/worker
> ```
> ↩️ Back to DevOps — proceed with the checklist below once all 3 images exist.

- ✅ `docker login` — authenticate with akhiltejadoosari credentials
- ✅ `docker info | grep -i username` — confirm correct account
- ✅ `docker tag shopstack-api akhiltejadoosari/shopstack-api:1.0`
- ✅ `docker tag shopstack-frontend akhiltejadoosari/shopstack-frontend:1.0`
- ✅ `docker tag shopstack-worker akhiltejadoosari/shopstack-worker:1.0`
- ✅ `docker images` — confirm all 6 tags exist — read IMAGE ID column: original and tagged must match
- ✅ `docker push akhiltejadoosari/shopstack-api:1.0`
- ✅ `docker push akhiltejadoosari/shopstack-frontend:1.0`
- ✅ `docker push akhiltejadoosari/shopstack-worker:1.0`
- ✅ Open hub.docker.com — confirm all 3 repos exist with tag 1.0
- ✅ `docker rmi shopstack-api shopstack-frontend shopstack-worker`
- ✅ `docker rmi akhiltejadoosari/shopstack-api:1.0 akhiltejadoosari/shopstack-frontend:1.0 akhiltejadoosari/shopstack-worker:1.0`
- ✅ `docker images` — confirm all 6 tags are gone
- ✅ `docker pull akhiltejadoosari/shopstack-api:1.0`
- ✅ `docker run -d -p 8080:8080 --name api-test akhiltejadoosari/shopstack-api:1.0`
- ✅ `docker logs api-test` — confirm it started (errors are OK — the registry pull worked, that's the win)
- ✅ `docker stop api-test && docker rm api-test`
- ✅ Answer out loud: why do we never use the `latest` tag in production?

**Session win condition:** All 3 ShopStack images on DockerHub tagged 1.0. Pull and run proves the registry works. You can explain every step without notes.

---

Ready to test yourself? → [Test](./test.md)
