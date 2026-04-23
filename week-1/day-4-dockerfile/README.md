# Day 4 — Dockerfiles

**Date:** April 23, 2026   
**Read before session:** [07-docker-layers](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/07-docker-layers) · [08-docker-build-dockerfile](https://github.com/AkhilTejaDoosari/devops-runbook/tree/main/notes/04.%20Docker%20%E2%80%93%20Containerization/08-docker-build-dockerfile)   
**App reference:** [ShopStack](https://github.com/AkhilTejaDoosari/shopstack)   
**Goal:** Write all 3 ShopStack Dockerfiles from scratch. Understand every line. Build every image with zero errors. Explain why layer order and multi-stage builds exist.

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md)

---

## Knowledge — What This Day Is Really About

### The Problem Dockerfiles Solve

Before Dockerfiles, getting software to run on a new machine meant: 
* install the right OS 
* install the right language runtime version
* install the right libraries
* set the right environment variables
* copy the code, figure out the start command.    

If anything didn't match — the app crashed. "Works on my machine" was a real crisis.

A Dockerfile is a script that captures all of those steps in order.   
You run `docker build` and Docker executes them layer by layer, producing an image.    
That image runs identically on your Mac, your EC2, your colleague's laptop, and Kubernetes — forever. The environment is baked in.

**The core rule:** A Dockerfile is a reproducible, ordered build recipe. Every line is a layer. Every layer is cached. Order determines speed.

---

### What a Layer Is

Every Dockerfile instruction that writes to the filesystem creates a layer — a snapshot of what changed. Layers stack. Docker combines them into one visible filesystem when a container runs.

```
┌─────────────────────────────────────┐
│  WRITABLE CONTAINER LAYER (runtime) │  ← Created when container runs. Temporary.
├─────────────────────────────────────┤
│  COPY src/ .                        │  ← Layer 5: your app code
├─────────────────────────────────────┤
│  RUN pip install -r requirements    │  ← Layer 4: installed dependencies (heavy)
├─────────────────────────────────────┤
│  COPY requirements.txt .            │  ← Layer 3: dependency manifest
├─────────────────────────────────────┤
│  WORKDIR /app                       │  ← Layer 2: directory created
├─────────────────────────────────────┤
│  FROM python:3.12-slim              │  ← Layer 1: base OS + Python runtime
└─────────────────────────────────────┘
   All these image layers are READ-ONLY
```

`CMD` and `EXPOSE` add metadata layers — zero bytes. Only `RUN` and `COPY` add real size.

---

### The Caching Law — Why Order Is Everything

Docker hashes every instruction. If the hash matches a previous build, Docker reuses the cached layer — instantly. If the hash changes, Docker rebuilds that layer **and every layer after it**. This is the most important thing about Dockerfiles.

```
Stable first → Volatile last

1. FROM (base image)          ← almost never changes
2. RUN (install system tools) ← rarely changes
3. COPY (dependency manifest) ← changes only when you add a library
4. RUN (install dependencies) ← only reruns when manifest changes
5. COPY (source code)         ← changes on every commit
6. CMD (start command)        ← almost never changes
```

**The real cost of wrong ordering:** If you put `COPY . .` (all source code) before `RUN pip install`, then every time you change a single line of code, pip install reruns from scratch.   
On a large Python app that could be 3 minutes. With correct ordering — change source code, pip install is cached, rebuild in 4 seconds.

---

### What Each Instruction Does

| Instruction | When it runs | What it does |
|---|---|---|
| `FROM` | Build time | Sets the starting filesystem — base OS + tools |
| `WORKDIR` | Build time | Creates directory + sets it as default. All subsequent `COPY` and `RUN` operate here |
| `COPY` | Build time | Brings files from your machine into the image |
| `RUN` | Build time | Executes a shell command — installs packages, compiles code, creates files |
| `ARG` | Build time | A variable you can pass in at `docker build` time — never baked into the image |
| `EXPOSE` | Never | Documentation only — tells readers what port the app uses. Does not open anything |
| `CMD` | Run time | The default command when a container starts. Can be overridden |

---

### Multi-Stage Builds — The Worker Pattern

The Go worker is the clearest example of why multi-stage builds exist.

To compile Go code you need the Go compiler — a ~800MB image. But to **run** compiled Go code you need almost nothing — the binary plus an OS (Alpine, ~5MB).

Without multi-stage: your production image weighs 800MB, contains the compiler, contains the source code. Every pull is slow. The attack surface is huge.

**With multi-stage:** you define two `FROM` blocks.    
**Stage 1 (builder)** has the compiler and does the work.    
**Stage 2 (runner)** starts fresh and copies **only the compiled binary** from Stage 1. Stage 1 is discarded. The final image is ~10MB.

```
Stage 1 — builder (golang:1.22-alpine, ~800MB)
  → compiles the Go binary
  → this stage is NEVER shipped

Stage 2 — runner (alpine:3.19, ~5MB)
  → copies only the binary from Stage 1
  → this is what gets pushed and deployed

Final image: ~10MB
```

**The one-sentence rule:** Multi-stage builds exist so that build tools never make it into the production image.

---

### ShopStack Connection

Today you build the images that Day 6 Compose uses.    
The `docker-compose.yml` has `build:` paths for api, frontend, and worker — it points at the Dockerfiles you write today.    
No Dockerfile → no Compose → no stack.

| Service | Base Image | Why |
|---|---|---|
| shopstack-api | python:3.12-slim | FastAPI runs on Python. Slim = smaller than full, still has pip |
| shopstack-frontend | nginx:1.24-alpine | Nginx serves HTML. Alpine = minimal OS |
| shopstack-worker | golang:1.22-alpine → alpine:3.19 | Go compiles to binary. Multi-stage drops the compiler |

---

## 📟 DAY 4 FULL CHEATSHEET — Dockerfiles

## Build Commands

| What it does | Command | Example |
|---|---|---|
| Build image from Dockerfile in current directory | `docker build -t <NAME> .` | `docker build -t shopstack-api .` |
| Build image from Dockerfile in another directory | `docker build -t <NAME> <PATH>` | `docker build -t shopstack-api ./services/api` |
| Build and force rebuild all layers — no cache | `docker build --no-cache -t <NAME> .` | `docker build --no-cache -t shopstack-api .` |
| See all layers of an image | `docker history <IMAGE>` | `docker history shopstack-api` |
| List all images — see sizes | `docker images` | `docker images` |

**[CORE] Syntax breakdown — docker build:**
```
docker build -t shopstack-api ./services/api
             ↑               ↑
             -t = tag        build context path
             (name:version)  (where Dockerfile and source files live)

If no version is given:  shopstack-api  →  Docker tags it as :latest automatically
With version:            shopstack-api:1.0
```

---

## Dockerfile Instructions

### `FROM` — [CORE]
```dockerfile
FROM python:3.12-slim
```
Sets the starting point. Everything runs on top of this image.
- `python:3.12-slim` = Python runtime on Debian, with non-essential packages stripped
- `nginx:1.24-alpine` = Nginx on Alpine Linux (~5MB OS)
- `golang:1.22-alpine` = Go compiler on Alpine

---

### `WORKDIR` — [CORE]
```dockerfile
WORKDIR /app
```
Creates the directory if it doesn't exist. Sets it as the current directory for all subsequent instructions. Every `RUN`, `COPY`, and `CMD` operates relative to this path.

---

### `COPY` — [CORE]
```dockerfile
COPY requirements.txt .        # copies one file into WORKDIR
COPY src/ .                    # copies src/ folder contents into WORKDIR
COPY html/ /usr/share/nginx/html/  # copies to an absolute path
```

**Syntax breakdown:**
```
COPY <source on your machine>   <destination inside image>
COPY requirements.txt           .
     ↑                          ↑
     relative to build context  . = current WORKDIR
```

---

### `RUN` — [CORE]
```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
RUN go mod download
RUN apk --no-cache add ca-certificates
RUN rm /etc/nginx/conf.d/default.conf
```
Executes a shell command at build time. Creates a layer. Use for installs, file manipulation, compilation.

`--no-cache-dir` on pip: don't save the download cache inside the image — keeps the layer smaller.
`--no-cache` on apk: same idea for Alpine's package manager.

---

### `EXPOSE` — [WIKI]
```dockerfile
EXPOSE 8080
EXPOSE 80
```
Documentation only. Tells anyone reading the Dockerfile which port the app listens on. Does not bind anything. Port binding still happens at `docker run -p`.

---

### `CMD` — [CORE]
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
CMD ["./worker"]
```
The default command when a container starts. JSON array format (exec form) — preferred. Each element is one part of the command.

**Syntax breakdown — uvicorn CMD:**
```
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
      ↑          ↑           ↑                     ↑
   the server  module:app   bind to all           port
               (main.py,    interfaces            inside container
               app object)  (not just localhost)
```

---

### `ARG` — [WIKI]
```dockerfile
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build -o worker .
```
Build-time variable. Docker BuildKit sets `TARGETARCH` automatically based on the platform you're building on (amd64 on EC2 x86, arm64 on Mac M-series). Used by the worker to compile the correct binary — without it you'd get `exec format error` running the binary on a different architecture.

---

### `AS` — Multi-Stage Naming — [CORE]
```dockerfile
FROM golang:1.22-alpine AS builder   # ← names this stage "builder"
...
FROM alpine:3.19                     # ← second FROM starts a new stage
COPY --from=builder /build/worker .  # ← pulls only this file from "builder"
```

**[CORE] Syntax breakdown — multi-stage copy:**
```
COPY --from=builder  /build/worker  .
     ↑               ↑              ↑
  stage name      source path    destination
  (from Stage 1)  inside builder  in current stage (WORKDIR)
```

---

## The Three ShopStack Dockerfiles

### shopstack-api — Python FastAPI

```
services/api/
├── Dockerfile
├── requirements.txt
└── src/
    └── main.py
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Why this order:** requirements.txt is copied first and pip runs before any source code is touched. Change `main.py` → pip install uses cache. Change `requirements.txt` → pip reruns. That's the caching law in action.

---

### shopstack-frontend — Nginx

```
services/frontend/
├── Dockerfile
├── nginx.conf
└── html/
    └── index.html
```

```dockerfile
FROM nginx:1.24-alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY html/ /usr/share/nginx/html/
EXPOSE 80
```

**Why `RUN rm` first:** Nginx ships with a default config. That config would conflict with ShopStack's. Remove it, then copy in the correct one. No `CMD` needed — nginx:1.24-alpine already has `CMD ["nginx", "-g", "daemon off;"]` baked in from its own Dockerfile.

---

### shopstack-worker — Go Multi-Stage

```
services/worker/
├── Dockerfile
├── go.mod
└── main.go (or similar)
```

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /build
COPY go.mod .
RUN go mod download
COPY . .
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build -o worker .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /build/worker .
CMD ["./worker"]
```

**Layer order logic in Stage 1:** `go.mod` is copied and dependencies downloaded before source code is copied. Module dependencies change rarely. Source code changes often. Same caching law — keep the slow `go mod download` cached.

---

## Cache Experiment — The Core Mental Model

| What you change | Which layers rebuild |
|---|---|
| Add a comment to `main.py` | Only `COPY src/ .` — pip install cached |
| Add a library to `requirements.txt` | `COPY requirements.txt .` + `RUN pip install` + `COPY src/ .` — all three rebuild |
| Change `main.py` and `requirements.txt` | Same as above — requirements change triggers everything after it |

---

## ⚠️ WHAT BREAKS TODAY

| Symptom | Cause | First command to run |
|---|---|---|
| `failed to solve: failed to read dockerfile` | Dockerfile doesn't exist or is misnamed (`dockerfile` vs `Dockerfile`) | `ls -la services/api/` |
| `COPY failed: file not found in build context` | Path in `COPY` doesn't match actual file location | `ls services/api/` — check what exists |
| `exec format error` when running worker | Binary compiled for wrong architecture | Rebuild with `docker build --platform linux/amd64` or check `ARG TARGETARCH` |
| pip install reruns every build even when requirements.txt didn't change | `COPY . .` comes before `RUN pip install` — cache busted on every code change | Fix order: COPY requirements first, RUN pip, then COPY src |
| Container starts but crashes immediately | `CMD` is wrong — wrong module path, wrong port | `docker logs <container>` |
| `shopstack-worker` image is 800MB | Not using multi-stage — final image contains the Go compiler | Check Dockerfile has two `FROM` blocks |

---

## Checklist

- [ ] Open `services/api/src/main.py` — read it, understand what it does
- [ ] Open `services/api/requirements.txt` — read it
- [ ] Write `services/api/Dockerfile` from scratch — no copy paste:
  - [ ] `FROM python:3.12-slim`
  - [ ] `WORKDIR /app`
  - [ ] `COPY requirements.txt .`
  - [ ] `RUN pip install --no-cache-dir -r requirements.txt`
  - [ ] `COPY src/ .`
  - [ ] `EXPOSE 8080`
  - [ ] `CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]`
- [ ] `docker build -t shopstack-api ./services/api`
- [ ] `docker images` — confirm shopstack-api exists
- [ ] `docker history shopstack-api` — read every layer
- [ ] Add a comment to `main.py` — rebuild — watch pip install use cache
- [ ] Change `requirements.txt` — rebuild — watch pip install rerun
- [ ] Understand why: stable layers first, volatile layers last
- [ ] Write `services/frontend/Dockerfile` from scratch:
  - [ ] `FROM nginx:1.24-alpine`
  - [ ] `RUN rm /etc/nginx/conf.d/default.conf`
  - [ ] `COPY nginx.conf /etc/nginx/conf.d/default.conf`
  - [ ] `COPY html/ /usr/share/nginx/html/`
  - [ ] `EXPOSE 80`
- [ ] `docker build -t shopstack-frontend ./services/frontend`
- [ ] `docker images` — confirm shopstack-frontend exists
- [ ] Write `services/worker/Dockerfile` from scratch — multi-stage:
  - [ ] `FROM golang:1.22-alpine AS builder`
  - [ ] `WORKDIR /build`
  - [ ] `COPY go.mod .`
  - [ ] `RUN go mod download`
  - [ ] `COPY . .`
  - [ ] `ARG TARGETARCH`
  - [ ] `RUN CGO_ENABLED=0 GOOS=linux GOARCH=${TARGETARCH} go build -o worker .`
  - [ ] `FROM alpine:3.19`
  - [ ] `RUN apk --no-cache add ca-certificates`
  - [ ] `WORKDIR /app`
  - [ ] `COPY --from=builder /build/worker .`
  - [ ] `CMD ["./worker"]`
- [ ] `docker build -t shopstack-worker ./services/worker`
- [ ] `docker images` — compare shopstack-worker size vs golang:1.22-alpine size
- [ ] Answer out loud: what is a multi-stage build and why use it?

**Session win condition:** All 3 images build with zero errors. You can explain every line.

---

Ready to test yourself? → [Test](./test.md)
