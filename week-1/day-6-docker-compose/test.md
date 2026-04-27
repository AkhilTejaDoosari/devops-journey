[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md) · [4. Dockerfiles](../day-4-dockerfiles/readme.md) · [5. Registry](../day-5-registry/readme.md) · [6. Compose](../day-6-compose/readme.md)

---

# Day 6 — Test Yourself

No notes. No checklist. Answer out loud or write it down first.
Check answers only after you have answered.

---

## Round 1 — Concepts

**Q1.** What are the three top-level blocks in a docker-compose.yml? What does each one declare?

<details><summary>Answer</summary>

`services:` — declares the containers (who runs and how).
`networks:` — declares the named network lanes (how they talk).
`volumes:` — declares named persistent volumes (what survives a `down`).

Everything else lives inside these three.
</details>

---

**Q2.** What is the difference between `image:` and `build:` in a service block?

<details><summary>Answer</summary>

`image:` — pulls a pre-built image from a registry (e.g. `postgres:15-alpine`). Used when you do not own the source code.

`build:` — builds an image from a Dockerfile at the given path (e.g. `../services/api`). Used for services you wrote — api, worker, frontend.

In ShopStack: `db` and `adminer` use `image:`. `api`, `worker`, and `frontend` use `build:`.
</details>

---

**Q3.** What is the difference between plain `depends_on` and `depends_on: condition: service_healthy`?

<details><summary>Answer</summary>

Plain `depends_on` only controls **start order** — it starts the dependency container before this one, but does not wait for it to be ready. The DB process may be starting but not yet accepting connections.

`condition: service_healthy` waits for the dependency's **healthcheck to pass** — a real probe confirming the service is ready — before starting the next container.

In ShopStack: `api` uses `condition: service_healthy` on `db` because it needs a live DB connection on boot. If it started too early it would crash.
</details>

---

**Q4.** Why does ShopStack use two networks (`web` and `backend`) instead of one?

<details><summary>Answer</summary>

Security by isolation. `frontend` is on `web` only — it can talk to `api` but has **no path to the database**. If the frontend were compromised, the attacker cannot reach `db` directly.

`api` is on both networks — it bridges the public-facing layer and the private data layer. It is the only gatekeeper to the database.

`worker`, `db`, and `adminer` are on `backend` only — never directly reachable from the web network.
</details>

---

**Q5.** What is the difference between `docker compose down` and `docker compose down -v`?

<details><summary>Answer</summary>

`docker compose down` — stops and removes containers and the network. Named volumes **survive**. Data is safe.

`docker compose down -v` — stops and removes containers, network, **and all named volumes**. The `db-data` volume is deleted. All database data is permanently gone. No recovery.

Rule: never use `-v` unless you explicitly want a clean slate.
</details>

---

**Q6.** `docker compose up --build` vs `docker compose up` — when do you use each?

<details><summary>Answer</summary>

`docker compose up` — starts services using **cached images**. If you changed a Dockerfile or source file but do not rebuild, the old image runs.

`docker compose up --build` — forces a rebuild of every `build:` service before starting. Use this any time you changed a Dockerfile, requirements file, or source code and need the change to take effect.

In ShopStack: after editing `services/api/Dockerfile`, always use `--build`.
</details>

---

**Q7.** ShopStack's `worker` service has no `ports:` mapping. Why can it still talk to the `api`?

<details><summary>Answer</summary>

`ports:` exposes a container to the **host machine** (your laptop, the internet). It is not needed for container-to-container communication.

Containers on the same Docker network talk to each other **by service name over the internal network** — no host port needed. `worker` uses `http://api:8080` as its `API_URL`. Docker DNS resolves `api` to the api container's internal IP automatically.

`ports:` is only needed when a human or external system needs to reach the container.
</details>

---

## Round 2 — Commands

> 🔧 Environment Setup — Not a DevOps Skill
> You need a clean working directory with the backup file restored before running these drills.
> ```bash
> cd ~/shopstack/infra
> # If you backed up the original file earlier:
> cp docker-compose.yml.bak docker-compose.yml
> docker compose down -v
> ```
> ↩️ Back to DevOps

---

**Q8.** Start all ShopStack services in the background after rebuilding images.

<details><summary>Answer</summary>

```bash
cd ~/shopstack/infra
docker compose up --build -d
```
</details>

---

**Q9.** Check that all 5 ShopStack services are running and healthy.

<details><summary>Answer</summary>

```bash
docker compose ps
```

All 5 services should show `healthy` or `running` under STATUS.
</details>

---

**Q10.** Follow the live logs of the `api` service only.

<details><summary>Answer</summary>

```bash
docker compose logs -f api
```

Ctrl+C to stop following.
</details>

---

**Q11.** Run a command inside the running `db` container to open a psql shell as the shopstack user.

<details><summary>Answer</summary>

```bash
docker compose exec db psql -U shopstack -d shopstack
```
</details>

---

**Q12.** Check all four ShopStack API endpoints from the terminal.

<details><summary>Answer</summary>

```bash
curl http://localhost/api/health
curl http://localhost/api/products
curl http://localhost/api/orders
curl http://localhost/api/metrics
```
</details>

---

**Q13.** Stop the stack and remove containers and network — keep the database volume.

<details><summary>Answer</summary>

```bash
docker compose down
```

No `-v` flag. The `db-data` volume survives.
</details>

---

**Q14.** Stop the stack and wipe everything including the database volume.

<details><summary>Answer</summary>

```bash
docker compose down -v
```

Warning: all database data is permanently gone.
</details>

---

**Q15.** Restart only the `worker` service without touching the rest of the stack.

<details><summary>Answer</summary>

```bash
docker compose restart worker
```
</details>

---

**Q16.** Rebuild only the `api` image without starting any containers.

<details><summary>Answer</summary>

```bash
docker compose build api
```
</details>

---

## Round 3 — Scenario Debug

**Scenario A — Race Condition**

You remove `depends_on` from the `api` service in `docker-compose.yml` and run `docker compose up --build`. The `api` container starts, immediately logs `connection refused` errors, and exits. `docker compose ps` shows `api` as `exited`.

What happened? What do you fix and how do you verify the fix?

<details><summary>Answer</summary>

**What happened:** Without `depends_on`, Compose started `api` at the same time as `db`. PostgreSQL was still initialising and not accepting connections. The api tried to connect, got `connection refused`, and crashed.

**Fix:** Restore `depends_on` with `condition: service_healthy` in the api service block:
```yaml
depends_on:
  db:
    condition: service_healthy
```

**Verify:** `docker compose down`, then `docker compose up --build`. Watch the logs — `api` should only start after db logs `database system is ready to accept connections`.
</details>

---

**Scenario B — DNS Failure**

You change `DB_HOST: db` to `DB_HOST: database` in the api service environment block. You bring the stack up. The api logs show: `could not translate host name "database" to address: Name or service not known`.

What is the cause? How do you fix it?

<details><summary>Answer</summary>

**Cause:** Docker DNS resolves service names — not arbitrary hostnames. The db service is named `db` in the compose file. That is its hostname on the network. `database` does not exist as a name on any Docker network, so DNS resolution fails.

**Fix:** Change `DB_HOST` back to `db` — the exact service name from the compose file.

Rule: `DB_HOST` must always match the service name declared under `services:`, character for character.
</details>

---

**Scenario C — 502 Bad Gateway**

ShopStack is up. You open `http://localhost` and see the frontend. But when the page tries to load products, the browser shows `502 Bad Gateway`. `docker compose ps` shows all 5 services as healthy.

What is the most likely cause and where do you look first?

<details><summary>Answer</summary>

**Most likely cause:** The nginx `proxy_pass` directive in the frontend config is pointing to the wrong upstream — wrong service name, wrong port, or wrong path.

**Where to look:**
1. `docker compose logs frontend` — look for upstream connection errors
2. Check `services/frontend/nginx.conf` — find the `proxy_pass` line. It should point to `http://api:8080`

If it says `http://api:8081` or `http://infra-api-1:8080` or any other variation, that is your bug. Fix the hostname or port, rebuild: `docker compose up --build frontend`.
</details>

---

**Scenario D — Port Conflict**

You run `docker compose up --build` and immediately see:

```
Error response from daemon: Bind for 0.0.0.0:80 failed: port is already allocated
```

The stack does not start. What do you do?

<details><summary>Answer</summary>

Another container or process already owns port 80 on the host.

**Steps:**
1. `docker ps` — find the container using port 80
2. `docker stop <container-name>` — stop it
3. `docker rm <container-name>` — remove it
4. `docker compose up --build` — try again

If no container shows port 80, a host process (like a local nginx or Apache) owns it. Find it with `sudo lsof -i :80` and stop that service.
</details>

---

**Scenario E — Data Wipe**

Your stack was running. You ran `docker compose down -v` to "restart fresh." Now you run `docker compose up` and the api logs show: `relation "products" does not exist`. Adminer shows an empty database.

What happened and how do you get the data back?

<details><summary>Answer</summary>

**What happened:** `docker compose down -v` deleted the `db-data` named volume. All PostgreSQL data was permanently destroyed. The `init.sql` script runs on first startup of a **new, empty** volume — but if the volume was recreated, the schema and seed data need to re-run.

**Recovery:** The `db-data` volume is gone. Bring the stack up fresh — `docker compose up --build`. Because the volume is new, PostgreSQL will run `init.sql` automatically (it is mounted at `/docker-entrypoint-initdb.d/`). The schema and seed data are restored from that file.

**Lesson:** Never use `-v` unless you want to wipe the database. For a normal restart, use `docker compose down` (no `-v`).
</details>

---

## Round 4 — Reading Output

**Q17.** You run `docker compose ps` and see:

```
NAME                IMAGE               STATUS              PORTS
infra-adminer-1     adminer:4           running             0.0.0.0:8081->8080/tcp
infra-api-1         infra-api           healthy             0.0.0.0:8080->8080/tcp
infra-db-1          postgres:15-alpine  healthy             5432/tcp
infra-frontend-1    infra-frontend      healthy             0.0.0.0:80->80/tcp
infra-worker-1      infra-worker        running             
```

Three questions:
1. Why does `db` show `5432/tcp` but no `0.0.0.0` prefix?
2. Why does `worker` show `running` instead of `healthy`?
3. What does `infra-api-1` tell you about the project folder name?

<details><summary>Answer</summary>

1. `db` has no `ports:` mapping in the compose file. `5432/tcp` is the container's internal port — not bound to the host. It is only reachable by other containers on the `backend` network. No browser or external tool can hit it directly.

2. `worker` has no `healthcheck:` block defined. Docker can only report `running` (process is alive) vs `healthy` (healthcheck passed). Without a healthcheck, it never reaches `healthy` status — this is expected, not a bug.

3. Container names follow the pattern `<project>-<service>-<instance>`. The project name is derived from the folder containing `docker-compose.yml`. `infra-api-1` means the compose file lives in a folder named `infra` — which is `~/shopstack/infra`. Correct.
</details>

---

**Q18.** You run `docker compose logs api` and see near the top:

```
api-1  | waiting for db...
api-1  | waiting for db...
api-1  | waiting for db...
api-1  | connected to shopstack database
api-1  | Uvicorn running on http://0.0.0.0:8080
```

What does this sequence tell you about the startup logic? Is this a problem?

<details><summary>Answer</summary>

This is **not a problem** — it is the retry logic working correctly. The api container started (Compose released it after the db healthcheck passed) and its own internal connection loop retried a few times before the DB was fully accepting connections at the application level.

The sequence tells you:
- `depends_on: condition: service_healthy` released the api after `pg_isready` passed
- But `pg_isready` checks socket-level readiness — the DB may still need a moment to load the schema
- The api's internal retry logic handled the gap gracefully
- Final line confirms a successful connection

This is correct behaviour. The retry loop exists precisely for this window.
</details>

---

**Q19.** You run `docker compose logs db` and see:

```
db-1  | PostgreSQL init process complete; ready for start up.
db-1  | LOG:  database system is ready to accept connections
```

Then 3 seconds later, the `api` container starts logging. Why the 3-second gap?

<details><summary>Answer</summary>

The `api` service has `depends_on: db: condition: service_healthy`. The healthcheck on `db` is configured with:

```
interval: 5s
start_period: 10s
```

Docker waited for `pg_isready` to return success. Once the healthcheck passed, Docker released the `api` container to start. The gap between "ready for start up" and the api's first log line is the healthcheck probe firing and succeeding, plus Docker's own container start time.

This is the entire point of `condition: service_healthy` — the gap proves it worked.
</details>

---
