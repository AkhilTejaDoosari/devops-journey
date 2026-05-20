# Session 05 — Registry

**Goal:** Push all three ShopStack images to Docker Hub, understand tagging strategy, pull back and verify. What you push here is what Kubernetes pulls later.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Registries Exist

An image that only lives on your laptop is useless in a pipeline.

```
Without registry:
  Image lives on your laptop only
  CI cannot pull it
  Kubernetes cannot pull it
  Teammate cannot pull it

With registry:
  Image pushed once → pulled anywhere
  Same image: dev laptop → CI → staging → production
  No rebuild drift — everyone runs the same binary
```

**Docker Hub** is the default public registry. In production companies use private registries — AWS ECR, GitHub Container Registry, Google GCR. The workflow is identical — only the URL changes.

---

## Visual Map

```
EC2 (your machine)
    │
    │ docker build
    ▼
Local Image
    │
    │ docker tag
    ▼
akhiltejadoosari/shopstack-api:1.0
    │
    │ docker push
    ▼
Docker Hub (registry)
    │
    │ docker pull (from anywhere)
    ▼
Kubernetes node / CI runner / teammate's machine
```

---

## Tagging Strategy

Tags are how CI/CD knows what to deploy. Wrong tags = production disasters.

| Context | Tag | Example |
|---|---|---|
| Every CI build | Git commit SHA | `shopstack-api:a3f92c1` |
| Versioned release | Semantic version | `shopstack-api:v1.0.0` |
| Local dev only | `latest` | Never deploy this to production |
| Production | Specific SHA or semver | Never `latest` |

**Why never `latest` in production:**
`latest` changes on every push. You cannot know what version is running. You cannot roll back to a specific version. Always use an immutable tag.

**ShopStack uses commit SHA tagging in CI:**
```bash
GIT_SHA=$(git rev-parse --short HEAD)
docker build -t akhiltejadoosari/shopstack-api:${GIT_SHA} .
docker push akhiltejadoosari/shopstack-api:${GIT_SHA}
```

When something breaks in production: check the deployed SHA → `git show SHA` → see exactly what changed.

---

## 🧪 Lab Setup — Docker Hub Login

> Not a DevOps skill. Authentication setup done once on each machine.

```bash
# Login to Docker Hub
docker login
# Enter your Docker Hub username and password/token when prompted

# Verify logged in
docker info | grep -i username
# expected: Username: akhiltejadoosari
```

---

## 🚀 DevOps Work — Build All Three ShopStack Images

```bash
# Build API
docker build -t shopstack-api:1.0 ~/shopstack/services/api

# Build frontend
docker build -t shopstack-frontend:1.0 ~/shopstack/services/frontend

# Build worker
docker build -t shopstack-worker:1.0 ~/shopstack/services/worker

# Verify all three exist
docker images | grep shopstack
```

Expected:
```
shopstack-api        1.0   xxxxxxxxxxxx   ~288MB
shopstack-frontend   1.0   xxxxxxxxxxxx   ~63MB
shopstack-worker     1.0   xxxxxxxxxxxx   ~25MB
```

---

## 🚀 DevOps Work — Tag for Docker Hub

Docker Hub requires the format: `USERNAME/IMAGE:TAG`

```bash
# Tag all three images for Docker Hub
docker tag shopstack-api:1.0 akhiltejadoosari/shopstack-api:1.0
docker tag shopstack-frontend:1.0 akhiltejadoosari/shopstack-frontend:1.0
docker tag shopstack-worker:1.0 akhiltejadoosari/shopstack-worker:1.0

# Verify tags exist
docker images | grep akhiltejadoosari
```

Expected — six rows total (local tag + Docker Hub tag for each image).

---

## 🚀 DevOps Work — Push to Docker Hub

```bash
# Push all three
docker push akhiltejadoosari/shopstack-api:1.0
docker push akhiltejadoosari/shopstack-frontend:1.0
docker push akhiltejadoosari/shopstack-worker:1.0
```

What happens during push:
- Docker checks which layers already exist in Docker Hub
- Only missing layers upload — shared base layers are skipped
- This is why second pushes are fast

---

## 🚀 DevOps Work — Verify Push Worked

```bash
# Delete local images to prove pull works
docker rmi shopstack-api:1.0 akhiltejadoosari/shopstack-api:1.0

# Pull from Docker Hub
docker pull akhiltejadoosari/shopstack-api:1.0

# Verify it pulled
docker images | grep shopstack-api
```

Expected: image appears — it came from Docker Hub, not local build.

---

## 🚀 DevOps Work — Also Tag as Latest

In development, `latest` is convenient for local testing. Just never use it in production deploys.

```bash
# Tag as latest for local convenience
docker tag akhiltejadoosari/shopstack-api:1.0 akhiltejadoosari/shopstack-api:latest
docker push akhiltejadoosari/shopstack-api:latest

docker tag akhiltejadoosari/shopstack-frontend:1.0 akhiltejadoosari/shopstack-frontend:latest
docker push akhiltejadoosari/shopstack-frontend:latest

docker tag akhiltejadoosari/shopstack-worker:1.0 akhiltejadoosari/shopstack-worker:latest
docker push akhiltejadoosari/shopstack-worker:latest
```

---

## 🚀 DevOps Work — What Kubernetes Will Pull

When you get to Kubernetes, your manifests will reference these exact image names:

```yaml
# In k8s deployment manifest
image: akhiltejadoosari/shopstack-api:1.0
image: akhiltejadoosari/shopstack-frontend:1.0
image: akhiltejadoosari/shopstack-worker:1.0
```

Kubernetes pulls from Docker Hub using these tags. What you pushed here is exactly what the cluster runs. This is the contract between CI and deployment.

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Docker Hub login confirmed
docker info | grep -i username
```
Expected: `Username: akhiltejadoosari`

---

```bash
# 2. All three images exist locally
docker images | grep -E "shopstack-api|shopstack-frontend|shopstack-worker" | grep "1.0"
```
Expected: three rows with tag `1.0`

---

```bash
# 3. Tags exist for Docker Hub format
docker images | grep "akhiltejadoosari/shopstack" | grep "1.0"
```
Expected: three rows with `akhiltejadoosari/` prefix

---

```bash
# 4. Pull works — delete and re-pull API image
docker rmi akhiltejadoosari/shopstack-api:1.0 2>/dev/null || true
docker pull akhiltejadoosari/shopstack-api:1.0
docker images | grep "shopstack-api" | grep "1.0"
```
Expected: image pulled from Docker Hub successfully

---

```bash
# 5. Latest tag exists on Docker Hub
docker pull akhiltejadoosari/shopstack-api:latest
docker images | grep "shopstack-api" | grep "latest"
```
Expected: latest tag pulled successfully

---

```bash
# 6. Verify all images on Docker Hub (check via pull)
docker pull akhiltejadoosari/shopstack-frontend:1.0
docker pull akhiltejadoosari/shopstack-worker:1.0
docker images | grep "akhiltejadoosari/shopstack"
```
Expected: all three images with both `1.0` and `latest` tags

---

**All 6 green?** → Save to `devops-journey/tutorials/01-docker/session-05-registry.md` → push → Session 06.

---

## Bugs You Will Hit

**`denied: requested access to the resource is denied`**
Cause: wrong Docker Hub username in tag, or not logged in.
Fix:
```bash
docker logout
docker login
# retag with correct username
docker tag IMAGE:TAG CORRECT_USERNAME/IMAGE:TAG
```

**`unauthorized: authentication required`**
Cause: credentials expired.
Fix:
```bash
docker logout
docker login
```

**Layers upload on every push — nothing reused**
Cause: base image tag changed, pulling a new digest.
Fix: pin base images to specific versions in Dockerfiles.

**`tag does not exist` when pulling**
Cause: tag was never pushed, or wrong tag name.
Fix:
```bash
docker images  # check what tags exist locally
# check Docker Hub UI for what was pushed
```

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Push denied | Wrong username in tag or not logged in | `docker logout` → `docker login` → retag |
| Pull fails on Kubernetes | Image is private, no pull secret | Set repo to public on Docker Hub for now |
| Wrong version in production | Used `latest` tag | Always use specific SHA or semver tag |
| Push slow every time | Base image changed digest | Pin base image to exact version |

---

## Quick Reference

| What | Command |
|---|---|
| Login | `docker login` |
| Logout | `docker logout` |
| Check logged in user | `docker info \| grep -i username` |
| Tag image | `docker tag SOURCE USERNAME/IMAGE:TAG` |
| Push image | `docker push USERNAME/IMAGE:TAG` |
| Pull image | `docker pull USERNAME/IMAGE:TAG` |
| List local images | `docker images` |
| Delete local image | `docker rmi IMAGE:TAG` |
