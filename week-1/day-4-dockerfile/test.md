# Day 4 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Then check the answer toggle.

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md)

---

## Round 1 — Concepts

**Q1.** What is a Dockerfile and what problem does it solve?

<details><summary>Answer</summary>

A Dockerfile is a reproducible, ordered build recipe. It captures every step needed to run an app — base OS, dependencies, source code, start command. Without it, getting software to run on a new machine means manual installs that break differently on every machine. With it, the environment is baked into an image that runs identically everywhere.

</details>

---

**Q2.** You change one line in `main.py` and rebuild `shopstack-api`. Does pip install rerun? Why or why not?

<details><summary>Answer</summary>

No. pip install is cached. `COPY requirements.txt .` and `RUN pip install` both come before `COPY src/ .` in the Dockerfile. Changing `main.py` only invalidates `COPY src/ .` and everything after it. pip install layer hash hasn't changed — Docker reuses it.

</details>

---

**Q3.** You add a new library to `requirements.txt` and rebuild. Which layers rebuild and which stay cached?

<details><summary>Answer</summary>

`COPY requirements.txt .` — rebuilds (file changed, hash changed).
`RUN pip install` — rebuilds (layer after the changed one, always invalidated).
`COPY src/ .` — rebuilds (everything after the cache break rebuilds).
`WORKDIR /app` and `FROM` — cached, they come before the change.

</details>

---

**Q4.** What does `EXPOSE 8080` actually do at runtime?

<details><summary>Answer</summary>

Nothing. It is documentation only — it tells anyone reading the Dockerfile which port the app listens on. Actual port binding happens at `docker run -p 8080:8080` or in `docker-compose.yml` under `ports:`.

</details>

---

**Q5.** What is the difference between `RUN`, `CMD`, and `COPY` in a Dockerfile?

<details><summary>Answer</summary>

`RUN` — executes a shell command at build time. Creates a layer. Used to install packages, compile code, manipulate files.
`CMD` — the default command when a container starts. Runs at runtime, not build time. Can be overridden.
`COPY` — brings files from your machine (build context) into the image at build time.

</details>

---

**Q6.** Why does the worker Dockerfile have two `FROM` lines? What happens to Stage 1?

<details><summary>Answer</summary>

Two `FROM` lines = multi-stage build. Stage 1 (builder) uses `golang:1.22-alpine` to compile the Go binary — it has the compiler, source code, go.mod. Stage 2 (runner) uses `alpine:3.19` and copies only the compiled binary from Stage 1. Stage 1 is completely discarded — it never makes it into the final image. Result: 800MB compiler image → 24MB production image.

</details>

---

**Q7.** Why does the frontend Dockerfile have `RUN rm /etc/nginx/conf.d/default.conf` before copying in `nginx.conf`?

<details><summary>Answer</summary>

Nginx ships with a default config that serves a "Welcome to Nginx" page. That config conflicts with ShopStack's config. You must remove it first, then copy ShopStack's `nginx.conf` in its place. If you skip the `rm`, Nginx would have two configs and behave unpredictably.

</details>

---

## Round 2 — Commands

**Q8.** Build `shopstack-api` from the correct path. You are currently in `~`.

<details><summary>Answer</summary>

```bash
docker build -t shopstack-api ~/shopstack/services/api
```

</details>

---

**Q9.** See every layer of `shopstack-api` — instruction, size, when created.

<details><summary>Answer</summary>

```bash
docker history shopstack-api
```

</details>

---

**Q10.** Build `shopstack-api` and force every layer to rebuild — no cache.

<details><summary>Answer</summary>

```bash
docker build --no-cache -t shopstack-api ~/shopstack/services/api
```

</details>

---

**Q11.** Confirm all three ShopStack images exist and see their sizes.

<details><summary>Answer</summary>

```bash
docker images | grep shopstack
```

</details>

---

**Q12.** Write the complete `shopstack-api` Dockerfile from memory. All 7 lines.

<details><summary>Answer</summary>

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY src/ .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

</details>

---

**Q13.** Write the complete `shopstack-frontend` Dockerfile from memory. All 5 lines.

<details><summary>Answer</summary>

```dockerfile
FROM nginx:1.24-alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY html/ /usr/share/nginx/html/
EXPOSE 80
```

</details>

---

**Q14.** Write the complete `shopstack-worker` Dockerfile from memory. All 12 lines.

<details><summary>Answer</summary>

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

</details>

---

> 🔧 Environment Setup — Not a DevOps Skill
> Commands are provided — copy and paste.
> ```bash
> cd ~/shopstack/infra && docker compose up -d
> docker compose ps
> ```
> ↩️ Back to DevOps

---

## Round 3 — Scenario Debug

**Scenario A**

You run `docker build -t shopstack-api .` from `~/shopstack/services/api` and get:

```
COPY failed: file not found in build context: stat src/: file not found
```

What went wrong and what is the first thing you check?

<details><summary>Answer</summary>

The `COPY src/ .` instruction can't find the `src/` folder. Either you're running `docker build` from the wrong directory, or `src/` doesn't exist where the Dockerfile expects it. First command: `ls ~/shopstack/services/api/` — confirm `src/` exists. Second: confirm you're running the build from the correct path.

</details>

---

**Scenario B**

You rebuild `shopstack-api` after changing `main.py`. You expect pip install to be cached but it reruns every single time. What is the most likely cause?

<details><summary>Answer</summary>

`COPY . .` comes before `RUN pip install` in the Dockerfile. Every time any file changes — including `main.py` — `COPY . .` invalidates, which forces pip install to rerun. Fix: put `COPY requirements.txt .` and `RUN pip install` before `COPY src/ .`.

</details>

---

**Scenario C**

You build `shopstack-worker` and the container crashes immediately with:

```
exec ./worker: exec format error
```

What happened and how do you fix it?

<details><summary>Answer</summary>

The binary was compiled for the wrong CPU architecture. Built for `amd64` but running on `arm64` (or vice versa). Fix: confirm `ARG TARGETARCH` is present and `GOARCH=${TARGETARCH}` is used in the build command. On Mac M-series building for EC2 x86, add `--platform linux/amd64` to the build command.

</details>

---

**Scenario D**

You build `shopstack-frontend` and run it. The browser shows "Welcome to nginx" instead of the ShopStack store. What happened?

<details><summary>Answer</summary>

The `RUN rm /etc/nginx/conf.d/default.conf` line is missing from the Dockerfile. Nginx's default config is still in place. The default config serves the welcome page instead of ShopStack's `html/index.html`. Fix: add `RUN rm /etc/nginx/conf.d/default.conf` before the `COPY nginx.conf` line.

</details>

---

## Round 4 — Reading Output

**Q15.** You run `docker history shopstack-api` and see:

```
CREATED BY                                          SIZE
CMD ["uvicorn" "main:app" "--host" "0.0.0.0"…      0B
EXPOSE [8080/tcp]                                   0B
COPY src/ . # buildkit                              20.5kB
RUN pip install --no-cache-dir -r requirements…     86.3MB
COPY requirements.txt . # buildkit                  12.3kB
WORKDIR /app                                        8.19kB
```

Why do CMD and EXPOSE show 0B?

<details><summary>Answer</summary>

CMD and EXPOSE are metadata instructions — they don't write any files to the filesystem. Only `RUN` and `COPY` instructions that actually add or modify files create layers with real size.

</details>

---

**Q16.** You run `docker build` and see:

```
=> CACHED [3/5] COPY requirements.txt .       0.0s
=> CACHED [4/5] RUN pip install ...           0.0s
=> [5/5] COPY src/ .                          0.1s
```

What changed between this build and the last one?

<details><summary>Answer</summary>

Only source code changed — something inside `src/`. `requirements.txt` hash is unchanged so layers 3 and 4 are cached. Layer 5 (`COPY src/ .`) rebuilt because a file inside `src/` changed. This is the caching law working correctly.

</details>

---

**Q17.** You run `docker images | grep shopstack` and see:

```
shopstack-api        288MB
shopstack-frontend   63.3MB
shopstack-worker     24.8MB
```

The Go compiler image is ~800MB. Why is `shopstack-worker` only 24.8MB?

<details><summary>Answer</summary>

Multi-stage build. The Go compiler lives only in Stage 1 (builder). Stage 2 starts fresh from `alpine:3.19` (~5MB) and copies only the compiled binary from Stage 1. The compiler, source code, and go.mod never make it into the final image. Final image = alpine base + ca-certificates + binary = ~24.8MB.

</details>

---
