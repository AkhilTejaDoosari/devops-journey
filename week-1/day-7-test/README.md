# Day 7 — Test Yourself

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md) · [7. Git + Interview](../day-7-interview/readme.md)

No notes. No checklist. Answer out loud or write it down first. Check the toggles only after you have answered.

---

## Round 1 — Concepts

*Why and what. 30 seconds each. No notes.*

---

**Q1. A teammate says "I deleted the container so the image is gone too." Are they right? Explain why or why not.**

<details>
<summary>Answer</summary>

Wrong. Deleting a container has no effect on the image. The container is a running instance — deleting it is like closing a tab in a browser. The website still exists. The image still lives in Docker's local storage. You need `docker rmi` to delete the image — and even that is blocked if any container, even a stopped one, still references it.

</details>

---

**Q2. Docker builds your image in 45 seconds the first time. You change one line in `main.py` and rebuild. It finishes in 3 seconds. What happened?**

<details>
<summary>Answer</summary>

Docker's layer cache did the work. Every layer that didn't change was served from cache — the base image pull, the dependency install, all of it. Only the `COPY . .` layer changed because `main.py` changed, so only that layer and everything after it rebuilt. That is why stable slow steps go first and volatile fast-changing steps go last — so the cache protects the expensive work.

</details>

---

**Q3. Same scenario — but now it takes 45 seconds again even though you only changed one line of code. What went wrong in the Dockerfile?**

<details>
<summary>Answer</summary>

Layer order is wrong. `COPY . .` is placed before `RUN pip install -r requirements.txt`. Because `main.py` changed, the `COPY . .` layer's hash changed, which invalidated every layer after it — including the pip install. Docker rebuilt the entire pip install from scratch even though `requirements.txt` didn't change.

Fix: put `COPY requirements.txt .` and `RUN pip install` before `COPY . .`. The pip install layer is now cached unless dependencies actually change.

</details>

---

**Q4. What is the difference between `docker compose down` and `docker compose down -v`? Which one would you never run on a production database and why?**

<details>
<summary>Answer</summary>

`docker compose down` stops and removes containers and networks. Named volumes are untouched. The `db-data` volume survives — all Postgres data is still there. You can bring everything back up with `docker compose up` and nothing is lost.

`docker compose down -v` does everything above plus permanently deletes all named volumes. `db-data` is gone. Every row in every table — orders, products, users — deleted with no undo.

Never run `-v` on a production database. It is a complete data wipe. In ShopStack, this is the difference between a normal shutdown and destroying the entire dataset.

</details>

---

**Q5. A junior dev asks: "Why can't I just use `docker run` five times instead of writing a compose file?" What do you tell them?**

<details>
<summary>Answer</summary>

You can — but you will spend 10 minutes typing flags, get one wrong, and spend another 10 debugging. For every restart. And the next person who takes over has no idea what flags you used.

Compose puts the entire stack — all 5 services, all networks, all volumes, all env vars, all health checks, all startup order — in one file. Written once, version controlled, repeatable by anyone. `docker compose up` and the whole thing starts correctly every time. That is the difference between a one-person script and a system other people can trust.

In ShopStack: without compose you'd type 5 `docker run` commands with port flags, network flags, env vars, and depends_on logic by hand. One typo in DB_HOST and the API silently fails. The compose file caught and version-controlled all of that.

</details>

---

## Round 2 — Commands

*Recall every checklist item cold. ShopStack specific. No notes.*

---

> 🔧 Environment Setup — Not a DevOps Skill
> ShopStack must be running before Docker commands work.
> ```bash
> cd ~/shopstack/infra && docker compose up -d
> docker compose ps
> ```
> All 5 services must show healthy before continuing.
> ↩️ Back to DevOps

---

**Q6. How do you check that all 5 ShopStack services are running and healthy?**

<details>
<summary>Answer</summary>

```bash
docker compose ps
```

Run from `~/shopstack/infra`. Shows all 5 services — name, state, and ports. All must show `running` or `healthy`.

</details>

---

**Q7. The API is behaving strangely. How do you read its logs? How do you follow them live?**

<details>
<summary>Answer</summary>

```bash
# Read all logs
docker compose logs api

# Follow live — Ctrl+C to stop
docker compose logs -f api
```

</details>

---

**Q8. You need to enter the running API container and check which environment variables are set inside it. What do you run?**

<details>
<summary>Answer</summary>

```bash
docker exec -it infra-api-1 /bin/sh
```

Then inside the container:
```bash
env
```

This shows all environment variables — DB_HOST, DB_PORT, DB_NAME — exactly as the running process sees them. If DB_HOST is wrong, this is where you catch it.

</details>

---

**Q9. You updated the API source code. How do you rebuild only the API image and restart just that service without touching db, worker, or frontend?**

<details>
<summary>Answer</summary>

```bash
docker compose up --build -d api
```

`--build` forces Docker to rebuild the image from the Dockerfile. `-d` runs it detached. `api` targets only that service — the other 4 are untouched.

</details>

---

**Q10. What is the exact volume name ShopStack uses for Postgres? What path inside the container does it map to?**

<details>
<summary>Answer</summary>

Volume name: `db-data`

Maps to: `/var/lib/postgresql/data` inside `infra-db-1`

This is where Postgres writes all its data files. The volume keeps this data alive even when the container is deleted.

</details>

---

**Q11. How do you verify that the `db-data` volume exists on this machine?**

<details>
<summary>Answer</summary>

```bash
docker volume ls
```

Look for `infra_db-data` in the output. Docker prefixes volumes with the compose project name — `infra` is the folder name, so the full volume name becomes `infra_db-data`.

</details>

---

**Q12. How do you check which Docker networks exist and confirm `web` and `backend` are there?**

<details>
<summary>Answer</summary>

```bash
docker network ls
```

Look for `infra_web` and `infra_backend`. Same prefix rule — the folder name `infra` is prepended by Compose.

</details>

---

## Round 3 — Scenario Debug

*ShopStack is broken. Diagnose and fix. No notes.*

---

**Scenario A**

You run `docker compose up -d` and everything starts. You open `http://localhost` and get a blank page. `docker compose ps` shows all 5 services as running. What do you check and in what order?

<details>
<summary>Answer</summary>

Running does not mean healthy. Work outward from the database:

1. `docker compose logs db` — is Postgres actually ready?
2. `docker compose logs api` — is the API connecting to the DB? Look for connection errors.
3. `docker compose logs frontend` — is nginx getting a response from the API? Look for 502 errors.

The most common cause: API started before DB was ready — the `depends_on: condition: service_healthy` either failed silently or the health check is misconfigured. Reading logs in dependency order finds the break in the chain.

</details>

---

**Scenario B**

You rebuild the API after a code change using `docker compose up --build -d api`. The new code is not showing up. The old behaviour persists. What is the most likely cause?

<details>
<summary>Answer</summary>

Docker used a cached layer. The `COPY . .` layer did not invalidate because Docker did not detect a change — possibly because the file timestamp or hash matched the cache. 

Force a full rebuild with no cache:
```bash
docker compose build --no-cache api
docker compose up -d api
```

`--no-cache` forces every layer to rebuild from scratch — no cache used, guaranteed fresh image.

</details>

---

**Scenario C**

A teammate ran `docker compose down -v` by mistake on the production ShopStack. The stack is back up but the storefront shows no products and no orders. What happened and what is the recovery path?

<details>
<summary>Answer</summary>

`down -v` deleted the `db-data` named volume. When the stack came back up, Postgres initialised a fresh empty database — all tables exist but all rows are gone. Products, orders, users — everything wiped.

Recovery path:
1. Check if a database backup exists — `pg_dump` file, snapshot, anything
2. If backup exists — restore it into the running db container
3. If no backup exists — data is permanently gone, must be re-seeded from scratch

This is why `down -v` is never run in production and why automated backups are non-negotiable for stateful services.

</details>

---

**Scenario D**

You push a fix and run `docker compose up --build -d`. The API container starts, runs for 3 seconds, then shows `Exited (1)`. What is your first command and what are you looking for?

<details>
<summary>Answer</summary>

```bash
docker compose logs api
```

Exit code 1 means the process crashed at startup. The logs will show the exact reason — wrong environment variable, missing DB connection, syntax error in the app, missing dependency. Never restart first. Never guess. Read the logs. The error is always there.

</details>

---

## Round 4 — Reading Output

*Real terminal output. Tell me what it means.*

---

**Q13. You run `docker compose ps` and see this:**

```
NAME              IMAGE              COMMAND      SERVICE    CREATED        STATUS
infra-db-1        postgres:15-alpine              db         2 minutes ago  Up 2 minutes (healthy)
infra-api-1       infra-api          ...          api        2 minutes ago  Up 2 minutes (healthy)
infra-worker-1    infra-worker       ...          worker     2 minutes ago  Up 2 minutes
infra-frontend-1  infra-frontend     ...          frontend   2 minutes ago  Up 2 minutes
infra-adminer-1   adminer            ...          adminer    2 minutes ago  Up 2 minutes
```

Worker and frontend show `Up` but not `healthy`. Is this a problem? Why or why not?

<details>
<summary>Answer</summary>

Not necessarily a problem. `healthy` only appears if a `healthcheck` is defined in the compose file for that service. Worker and frontend likely have no healthcheck configured — so Docker has no way to report health, only that the process is running.

If the app is functioning correctly — frontend loads, worker is processing — then `Up` without `healthy` is fine. If you want `healthy` status, you need to add a `healthcheck` block to those services in `docker-compose.yml`.

</details>

---

**Q14. You run `docker compose logs api` and see this repeating every 2 seconds:**

```
api_1  | sqlalchemy.exc.OperationalError: could not connect to server: Connection refused
api_1  | Is the server running on host "db" and accepting TCP/IP connections on port 5432?
```

What does this tell you and what do you check first?

<details>
<summary>Answer</summary>

The API cannot reach the database. Two possible causes:

1. **DB is not ready yet** — Postgres takes a few seconds to initialise. If `depends_on: condition: service_healthy` is not configured, the API can start before the DB is accepting connections. Check `docker compose logs db` — is it still initialising?

2. **DB_HOST is wrong** — The API is looking for a host called `db`. If the service name in `docker-compose.yml` is different, Docker DNS cannot resolve it. Check the `db:` service name in compose and the `DB_HOST` env var on the API — they must match exactly.

First command: `docker compose logs db` — confirm the database is actually up and ready before debugging the API further.

</details>

---

**Q15. You run `docker volume ls` and see:**

```
DRIVER    VOLUME NAME
local     infra_db-data
```

You then run `docker compose down`. You run `docker volume ls` again. What do you expect to see and why?

<details>
<summary>Answer</summary>

`infra_db-data` is still there. `docker compose down` stops and removes containers and networks — it does not touch named volumes. The volume and all Postgres data inside it survive. 

Only `docker compose down -v` would remove it. Since that flag was not used, the volume is safe.

</details>

---

**Session nav:**
[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md) · [7. Git + Interview](../day-7-interview/readme.md)
