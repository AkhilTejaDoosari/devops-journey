# Day 3 — Test Yourself: Volumes

[1. Containers](../day-1-containers/readme.md) · [2. Ports & Networking](../day-2-networking/readme.md) · [3. Volumes](../day-3-volumes/readme.md)

No notes. No checklist. Answer out loud or write it down first.
Check the answer toggle only after you have committed to an answer.

---

## Round 1 — Concepts
*30 seconds per question. Why and what — no commands.*

**Q1.** What happens to data written inside `infra-db-1` when you run `docker rm infra-db-1`?

<details><summary>Answer</summary>

It is gone permanently. The container's writable layer is deleted with the container. Every table, every row, every schema written inside the container filesystem is erased. This is why `infra-db-1` needs a named volume.

</details>

---

**Q2.** What is the difference between a named volume and a bind mount? Give one use case for each.

<details><summary>Answer</summary>

Named volume — Docker manages the storage location. You give it a name and mount it to a path inside the container. Use it for databases and critical state that must survive container deletion. Example: `db-data` for `infra-db-1`.

Bind mount — You specify an exact path on your host. That folder is mounted directly into the container. Use it during development so you can edit code on your laptop and see changes instantly inside the container.

</details>

---

**Q3.** Why does ShopStack mount `db-data` to `/var/lib/postgresql/data` specifically? Why not some other path?

<details><summary>Answer</summary>

`/var/lib/postgresql/data` is PostgreSQL's internal data directory — hardcoded by the image. Every row inserted into `inventory.products` or `orders.orders` is written there. Mounting the volume to that exact path means Docker writes data to the volume instead of the ephemeral container layer. Any other path would not capture Postgres's data.

</details>

---

**Q4.** You run `docker compose down` on ShopStack. Then you run `docker compose down -v`. What is the difference?

<details><summary>Answer</summary>

`docker compose down` — stops and removes containers. The `db-data` volume survives. All ShopStack data is safe.

`docker compose down -v` — stops and removes containers AND deletes all volumes including `db-data`. Every product, every order, every stock level is gone. The next `docker compose up` starts from a fresh database.

</details>

---

**Q5.** You run `docker rm infra-db-1`. Then you run `docker volume ls` and see `db-data` is still there. Is that a bug?

<details><summary>Answer</summary>

No — it is intentional. `docker rm` never touches volumes. Docker forces you to explicitly pull the trigger on data with `docker volume rm db-data`. This protects you from accidentally wiping a production database just because you removed a container.

</details>

---

**Q6.** Adminer tried to connect to server `db` and failed with "could not translate host name". What caused it and how do you fix it?

<details><summary>Answer</summary>

Docker DNS resolves containers by their container name, not a service alias. The container was named `infra-db-1`, not `db`. Adminer tried to find a container called `db` on the network — nothing with that name existed. Fix: change the Server field in Adminer from `db` to `infra-db-1`, or rename the container to `db` when running it.

</details>

---

## Round 2 — Commands
*Recall cold. ShopStack names only. No flags looked up.*

**Q7.** Create a network called `backend`.

<details><summary>Answer</summary>

```bash
docker network create backend
```

</details>

---

**Q8.** Run `infra-db-1` on the `backend` network with the ShopStack credentials. No volume.

<details><summary>Answer</summary>

```bash
docker run -d --name infra-db-1 --network backend \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  postgres:15
```

</details>

---

**Q9.** Run `infra-adminer-1` on the `backend` network. EC2 port 8081, container port 8080.

<details><summary>Answer</summary>

```bash
docker run -d --name infra-adminer-1 --network backend -p 8081:8080 adminer:4
```

Port mapping is `8081:8080` — NOT `8081:8081`. Adminer listens on 8080 internally.

</details>

---

**Q10.** Run `infra-db-1` again — this time attach the `db-data` volume to the correct Postgres path.

<details><summary>Answer</summary>

```bash
docker run -d --name infra-db-1 --network backend \
  -e POSTGRES_DB=shopstack \
  -e POSTGRES_USER=shopstack \
  -e POSTGRES_PASSWORD=shopstack_dev \
  -v db-data:/var/lib/postgresql/data \
  postgres:15
```

</details>

---

**Q11.** Check what is mounted on `infra-db-1` — confirm the volume attached correctly.

<details><summary>Answer</summary>

```bash
docker inspect infra-db-1 | grep -A 10 Mounts
```

If the Mounts section is empty, the volume flag was wrong and did not attach.

</details>

---

**Q12.** Open a PostgreSQL shell inside `infra-db-1` using the ShopStack credentials.

<details><summary>Answer</summary>

```bash
docker exec -it infra-db-1 psql -U shopstack -d shopstack
```

</details>

---

**Q13.** List all volumes on this host.

<details><summary>Answer</summary>

```bash
docker volume ls
```

</details>

---

**Q14.** Inspect the `db-data` volume — see where Docker put it on the EC2 disk.

<details><summary>Answer</summary>

```bash
docker volume inspect db-data
```

Look for the `Mountpoint` field — it shows the actual path on the EC2 host where Docker stores the data, typically `/var/lib/docker/volumes/db-data/_data`.

</details>

---

**Q15.** Delete all anonymous volumes not attached to any container in one command.

<details><summary>Answer</summary>

```bash
docker volume prune
```

Note: `prune` only removes anonymous volumes. Named volumes like `db-data` and `infra_db-data` are NOT removed — you must `docker volume rm` those explicitly.

</details>

---

**Q16.** Stop and remove both `infra-db-1` and `infra-adminer-1` in two commands.

<details><summary>Answer</summary>

```bash
docker stop infra-db-1 infra-adminer-1
docker rm infra-db-1 infra-adminer-1
```

These are separate commands. `docker stop rm infra-db-1` does not work — `rm` is not a container name, it is a separate subcommand.

</details>

---

> 🔧 **Environment Setup — Not a DevOps Skill**
> Commands are provided — copy and paste.
> ```sql
> CREATE TABLE products (id SERIAL, name TEXT);
> INSERT INTO products (name) VALUES ('Widget');
> SELECT * FROM products;
> ```
> Run these inside Adminer SQL command to seed data for the persistence proof.
> ↩️ Back to DevOps

---

## Round 3 — Scenario Debug
*ShopStack is broken. Diagnose and fix.*

**Scenario A**
You run `infra-db-1` with `-v db-data:/var/lib/postgresql/data`. You insert a row in Adminer. You stop and remove the container. You recreate it with the same volume flag. You open Adminer — no tables. What are the two most likely causes?

<details><summary>Answer</summary>

1. Volume flag syntax was wrong on the second run — you forgot `-v db-data:/var/lib/postgresql/data` or used a different volume name. Run `docker inspect infra-db-1 | grep -A 10 Mounts` — if the Mounts section is empty, the volume did not attach.

2. You used a different volume name — `db-data` vs `shopstack-db-data` for example. Docker created a new empty volume instead of reusing the old one. Run `docker volume ls` to see what volumes exist.

</details>

---

**Scenario B**
You run `infra-adminer-1` with `-p 8081:8081`. You open `http://YOUR_EC2_IP:8081` and the page loads but Adminer immediately crashes on every login attempt. What is wrong?

<details><summary>Answer</summary>

The port mapping is wrong. Adminer listens on port `8080` internally — not `8081`. The correct mapping is `-p 8081:8080`. With `-p 8081:8081`, traffic hits the container on port 8081 where nothing is listening, so the connection drops. Remove the container and rerun with `-p 8081:8080`.

</details>

---

**Scenario C**
You run `docker volume rm db-data` and get:
```
Error response from daemon: remove db-data: volume is in use
```
What do you do?

<details><summary>Answer</summary>

A container — even a stopped one — still references the volume. You cannot delete a volume while any container holds a reference to it.

1. `docker ps -a` — find the container referencing `db-data`
2. `docker rm <container-name>` — remove it
3. `docker volume rm db-data` — now it works

</details>

---

**Scenario D**
A colleague says "I redeployed `infra-db-1` and now all the product data is gone." They show you the run command — it looks correct. What is the first thing you check?

<details><summary>Answer</summary>

Check if the volume flag is present and correct:
```bash
docker inspect infra-db-1 | grep -A 10 Mounts
```
If Mounts is empty — they forgot `-v db-data:/var/lib/postgresql/data`. The container ran with no volume, wrote data to the ephemeral container layer, and when it was replaced the data was gone.

Also run `docker volume ls` — if `db-data` doesn't exist, they may have run `docker compose down -v` which wiped the volume entirely.

</details>

---

## Round 4 — Reading Output
*Real terminal output. Tell me what it means.*

**Q17.** You run `docker volume ls` and see:
```
DRIVER    VOLUME NAME
local     4ad187aec23c84d3db2e16d25607e844fe7279971b104d30285e920ab48c71a0
local     db-data
local     infra_db-data
```
What are each of these and which ones are safe to prune?

<details><summary>Answer</summary>

- `4ad187aec23c...` — anonymous volume. Created automatically by Postgres when you ran a container without a `-v` flag. Safe to prune with `docker volume prune`.
- `db-data` — named volume you created manually during today's session. Contains your Widget row. Do NOT prune — requires explicit `docker volume rm db-data`.
- `infra_db-data` — named volume created by Docker Compose for ShopStack. Contains all ShopStack product and order data. Do NOT prune — requires explicit `docker volume rm infra_db-data`.

`docker volume prune` only removes anonymous volumes — the hash-named ones. Named volumes are always safe from prune.

</details>

---

**Q18.** You run `docker inspect infra-db-1 | grep -A 10 Mounts` and see:
```json
"Mounts": [],
```
What does this tell you and what do you do?

<details><summary>Answer</summary>

The volume did not attach. `infra-db-1` is running with no volume — all data is being written to the ephemeral container layer. When this container is removed, all data is gone.

Fix: stop and remove the container, rerun with the correct volume flag:
```bash
docker stop infra-db-1 && docker rm infra-db-1
docker run -d --name infra-db-1 --network backend \
  -e POSTGRES_DB=shopstack -e POSTGRES_USER=shopstack -e POSTGRES_PASSWORD=shopstack_dev \
  -v db-data:/var/lib/postgresql/data postgres:15
```
Then verify: `docker inspect infra-db-1 | grep -A 10 Mounts` — should now show Source and Destination fields.

</details>

---

**Q19.** You run `docker compose down` on ShopStack and then `docker volume ls`. You see `infra_db-data` still listed. Your teammate says "the volume got deleted." Who is right?

<details><summary>Answer</summary>

You are right. `docker compose down` stops and removes containers but intentionally leaves volumes intact. `infra_db-data` surviving is correct behaviour — it protects the database from accidental data loss. If your teammate wanted to delete the volume they needed `docker compose down -v`.

</details>

---

**Session win condition:** You can explain why data died, why it survived, and what the `-v` flag actually does — without looking at notes.
