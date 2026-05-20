# Session 03 — Volumes

**Goal:** Prove data dies without a volume, prove data survives with one, understand named volumes vs bind mounts.

---

## Who Does What

| Label | Meaning |
|---|---|
| 🧪 **Lab Setup** | One-time setup. Not your job in a real company. Copy paste without guilt. |
| 🚀 **DevOps Work** | This is your job. Understand every line. Goes on your resume. |
| 🧑‍💻 **Dev Work** | The developer owns this. You consume it, you don't build it. |
| ⚙️ **Ops Work** | Traditional sysadmin territory. Worth knowing, not your primary skill. |

---

## Why Volumes Exist

Containers are ephemeral. When deleted, everything inside them is gone — including database rows.

```
docker run postgres:15-alpine   → database starts, stores data inside container
docker rm shopstack-db          → container deleted — ALL DATA GONE
docker run postgres:15-alpine   → fresh container — empty database
```

Volumes store data outside the container. The container is replaceable. The data is not.

```
Container (code runs here)  ──►  Volume (data lives here)
    ↓                                    ↓
  Dies when deleted               Survives forever
```

---

## Visual Map

```
EC2 Host
├── Docker Volumes (managed by Docker)
│   └── infra_db-data
│       └── /var/lib/docker/volumes/infra_db-data/_data  ← actual data on disk
└── Containers
    └── infra-db-1
        └── /var/lib/postgresql/data  ← mounted FROM infra_db-data volume
                                         Postgres writes here
                                         Data actually lives in the volume
```

Delete `infra-db-1` → recreate it → remounts `infra_db-data` → all data still there.

---

## Named Volume vs Bind Mount

| Type | Who controls path | Where data lives | Use for |
|---|---|---|---|
| **Named Volume** | Docker | Docker-managed location | Database data, critical state |
| **Bind Mount** | You | Exact host path you specify | Development — edit code, see changes instantly |

---

## The Files — ShopStack Volume Design

### 📄 infra/docker-compose.yml — Volumes Section

**What it does:** defines `db-data` as a named volume. Compose creates it automatically. Postgres mounts it at `/var/lib/postgresql/data`.

<details>
<summary>📄 Show volumes section — infra/docker-compose.yml</summary>

```yaml
volumes:
  db-data:   # Compose creates and manages this volume
             # Name on host: infra_db-data (folder name prefix added by Compose)
             # Location on host: /var/lib/docker/volumes/infra_db-data/_data

# How db service uses it:
  db:
    volumes:
      - db-data:/var/lib/postgresql/data
      # ↑            ↑
      # volume name  path inside the container where Postgres stores its data
      # Docker maps the volume to this path
      # Postgres writes to /var/lib/postgresql/data
      # Data actually goes to the named volume on the host

      - ../db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      # ↑                ↑                                    ↑
      # host file path   container path Postgres reads on     read-only
      #                  first start to seed the database
      # This is a BIND MOUNT — used for the seed SQL file only
      # :ro = read-only — Postgres can read it but not write to it
```

</details>

---

### 📄 db/init.sql — The Seed File

**What it does:** runs automatically the first time Postgres starts (when the volume is empty). Creates two schemas and seeds 6 products.

<details>
<summary>📄 Show full file — db/init.sql</summary>

```sql
-- Two schemas: one per service domain
-- inventory schema → products (owned by API)
-- orders schema    → orders (owned by API)
-- Same Postgres instance, logically separated
-- When you move to microservices, each schema becomes its own database

CREATE SCHEMA IF NOT EXISTS inventory;
CREATE SCHEMA IF NOT EXISTS orders;

-- Products table
CREATE TABLE IF NOT EXISTS inventory.products (
    id          SERIAL PRIMARY KEY,      -- auto-incrementing ID
    name        VARCHAR(200)   NOT NULL,
    category    VARCHAR(100)   NOT NULL, -- book, course, gear
    price       NUMERIC(10,2)  NOT NULL,
    stock       INTEGER        NOT NULL DEFAULT 0,
    description TEXT,
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

-- Orders table
CREATE TABLE IF NOT EXISTS orders.orders (
    id          SERIAL PRIMARY KEY,
    product_id  INTEGER        NOT NULL, -- references inventory.products.id
    quantity    INTEGER        NOT NULL DEFAULT 1,
    total       NUMERIC(10,2)  NOT NULL,
    status      VARCHAR(50)    NOT NULL DEFAULT 'confirmed',
    created_at  TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

-- Seed 6 products — DevOps tooling domain
INSERT INTO inventory.products (name, category, price, stock, description) VALUES
('The Linux Command Line',              'book',   39.99, 150, 'William Shotts. The definitive guide.'),
('AWS Solutions Architect Course',      'course', 89.99, 999, 'Hands-on AWS — EC2, VPC, EKS, RDS.'),
('Mechanical Keyboard TKL',             'gear',  129.99,  34, 'Brown switches. Productivity justified.'),
('Kubernetes in Action',                'book',   49.99,  85, 'Marko Luksa. Makes K8s click.'),
('Docker & Kubernetes Bootcamp',        'course', 74.99, 999, 'Build, ship, run containers.'),
('USB-C Hub — 7 Port',                  'gear',   59.99,  60, 'HDMI, USB-A, SD, PD, Ethernet.');

-- One seed order so /api/orders returns real data on first boot
INSERT INTO orders.orders (product_id, quantity, total, status) VALUES
(1, 1, 39.99, 'confirmed');
```

</details>

> **Key behaviour:** `init.sql` only runs when the volume is empty (first start). On every restart after that, Postgres reconnects to existing data. Run `docker compose down -v` → volume deleted → next `up` reruns `init.sql` → only seed data comes back, not your orders.

---

## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
## 🧪 Lab Setup — Nothing to set up
## Docker and the shopstack repo are already on this machine.
## ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

---

## 🚀 The Work Begins

### Step 1 — Prove data dies without a volume

```bash
# Start Postgres WITHOUT a volume
docker run -d \
  --name no-vol-db \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine

sleep 5

# Insert a row
docker exec no-vol-db psql -U shopstack -d shopstack \
  -c "CREATE TABLE test (id SERIAL, name TEXT);"
docker exec no-vol-db psql -U shopstack -d shopstack \
  -c "INSERT INTO test (name) VALUES ('dies without volume');"

# Verify row exists
docker exec no-vol-db psql -U shopstack -d shopstack \
  -c "SELECT * FROM test;"
```

Expected: row appears

```bash
# Delete the container
docker stop no-vol-db && docker rm no-vol-db

# Start fresh — same image, no volume
docker run -d \
  --name no-vol-db \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine

sleep 5

# Try to find the data
docker exec no-vol-db psql -U shopstack -d shopstack \
  -c "SELECT * FROM test;"
```

Expected: `ERROR: relation "test" does not exist` — data is gone. This is why volumes exist.

```bash
docker stop no-vol-db && docker rm no-vol-db
```

---

### Step 2 — Prove data survives with a volume

```bash
# Create a named volume
docker volume create db-data
docker volume ls | grep db-data

# Start Postgres WITH the volume
docker run -d \
  --name vol-db \
  -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine

sleep 5

# Insert data
docker exec vol-db psql -U shopstack -d shopstack \
  -c "CREATE TABLE test (id SERIAL, name TEXT);"
docker exec vol-db psql -U shopstack -d shopstack \
  -c "INSERT INTO test (name) VALUES ('survives with volume');"

# Verify
docker exec vol-db psql -U shopstack -d shopstack \
  -c "SELECT * FROM test;"
```

Expected: row appears

```bash
# Delete the container — volume survives
docker stop vol-db && docker rm vol-db
docker volume ls | grep db-data  # still there

# Start fresh with SAME volume
docker run -d \
  --name vol-db \
  -v db-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15-alpine

sleep 5

# Data is still there
docker exec vol-db psql -U shopstack -d shopstack \
  -c "SELECT * FROM test;"
```

Expected: `survives with volume` appears — volume worked.

```bash
docker stop vol-db && docker rm vol-db
docker volume rm db-data
```

---

### Step 3 — See where the volume lives on the host

```bash
docker volume create inspect-test
docker volume inspect inspect-test | grep Mountpoint
```

Expected: `"Mountpoint": "/var/lib/docker/volumes/inspect-test/_data"`

```bash
docker volume rm inspect-test
```

---

### Step 4 — Confirm ShopStack volume design

```bash
grep -A 2 "^volumes:" ~/shopstack/infra/docker-compose.yml
```

Expected: `db-data:` defined

---

## ✅ Session Checkpoint — Verify Before Moving On

```bash
# 1. Volume creation works
docker volume create checkpoint-vol
docker volume ls | grep checkpoint-vol

# 2. Data survives container deletion
docker run -d --name vol-test \
  -v checkpoint-vol:/data \
  alpine sh -c "echo 'volume works' > /data/test.txt && sleep 60"
docker exec vol-test cat /data/test.txt
docker stop vol-test && docker rm vol-test
docker run --rm -v checkpoint-vol:/data alpine cat /data/test.txt

# 3. Host path confirmed
docker volume inspect checkpoint-vol | grep Mountpoint

# 4. ShopStack volume defined
grep -A 2 "^volumes:" ~/shopstack/infra/docker-compose.yml

# 5. Clean up
docker volume rm checkpoint-vol
docker volume ls | grep checkpoint-vol
```

Expected for #2: `volume works` after container deletion
Expected for #5: no output — volume deleted

**All 5 green?** → Save file → push → Session 04.

---

## ⚠️ The Dangerous Command

```bash
docker compose down      # ✅ safe — volumes survive, data safe
docker compose down -v   # ❌ dangerous — ALL volumes deleted, ALL data gone permanently
```

Never run `docker compose down -v` in production without a backup.

---

## What Breaks

| Symptom | Cause | Fix |
|---|---|---|
| Data gone after container delete | No volume attached | Always use `-v VOLUME:/path` |
| `volume is in use` on rm | Stopped container references it | `docker rm CONTAINER` first |
| Data missing after `compose down` | Used `-v` flag | Restore from backup or reseed |

---

## Quick Reference

| What | Command |
|---|---|
| Create volume | `docker volume create NAME` |
| List volumes | `docker volume ls` |
| Inspect volume | `docker volume inspect NAME` |
| Remove volume | `docker volume rm NAME` |
| Mount named volume | `docker run -v NAME:/path IMAGE` |
| Safe compose stop | `docker compose down` |
| Dangerous compose stop | `docker compose down -v` ⚠️ |
