# Databases in Docker — General Notes (MongoDB & Postgres examples)

## 1. Direct install vs Docker (same app code either way)

Your app driver/ORM doesn't care *how* the database is running — it just connects to a host and port. Docker doesn't change your connection code, only where the database process physically lives.

```js
// Mongoose — identical whether mongod is installed directly or running in a container
mongoose.connect('mongodb://localhost:27017/mydb');
```

```js
// node-postgres — identical whether postgres is installed directly or running in a container
const client = new Client({ connectionString: 'postgresql://user:pass@localhost:5432/mydb' });
```

| | Direct install | Docker |
|---|---|---|
| Where the DB process runs | Natively on your OS | Inside an isolated container |
| Cleanup | Manual (services, config files scattered) | `docker rm` — nothing left behind |
| Switching versions | Painful (uninstall/reinstall) | Trivial — different image tags, can run side by side |
| Team consistency | Drifts over time per machine | Same image = same version/config everywhere |
| Running app + db together | Manual setup | `docker-compose up` — one command |

**Running each database as a container:**

```bash
# MongoDB
docker run -d -p 27017:27017 -v mongo-data:/data/db mongo

# Postgres
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=secret -v pg-data:/var/lib/postgresql/data postgres
```

**docker-compose.yml (app + db together, containers talk by service name, not localhost):**

```yaml
services:
  app:
    build: .
    ports: ["3000:3000"]
    depends_on: [db]
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb   # or mongodb://db:27017/mydb
  db:
    image: postgres          # or: image: mongo
    ports: ["5432:5432"]     # or: ["27017:27017"]
    environment:
      - POSTGRES_PASSWORD=secret
    volumes: ["pg-data:/var/lib/postgresql/data"]   # or: ["mongo-data:/data/db"]
volumes:
  pg-data:
```

---

## 2. Where does the data actually live?

| Setup | Data location | Notes |
|---|---|---|
| **Managed cloud service** (Atlas, RDS, Cloud SQL, etc.) | Provider's own servers | Fully managed, nothing local |
| **Direct install** | A local data directory (e.g. `/data/db` for Mongo, `/var/lib/postgresql/data` for Postgres) | Plain folder, survives service restarts |
| **Docker (no volume)** | Container's writable layer | **Deleted when container is removed** |
| **Docker (with volume)** | Named volume → `/var/lib/docker/volumes/<name>/_data` on host disk | Survives container removal |

Every DBMS image has its own internal data path that needs a volume pointed at it:

```bash
-v mongo-data:/data/db                        # MongoDB
-v pg-data:/var/lib/postgresql/data           # Postgres
```

> **A GUI client (Compass, pgAdmin, TablePlus, etc.) is not a storage location.** It's just a window into whatever URI you give it — cloud, direct install, or a container all look identical to the GUI.

**If using a managed cloud DB, skip running that DB in Docker locally** — a local container still consumes local disk via its volume. The cloud service replaces the need for a local database process altogether.

---

## 3. Image vs Volume — why teammates get empty databases

This applies to any DBMS.

A **Docker image** = the database software only (the recipe). Sharing/pulling an image never includes data.
A **Volume** = created fresh, at run time, on whichever machine runs the container.

So: Dev A's image + Dev A's volume (their real local data) → pushed/pulled by teammate → teammate's container creates a **brand new, empty** volume with the same *name*, on their own disk. This is expected Docker behavior, not a bug — and it's true whether the image is `mongo`, `postgres`, `mysql`, or anything else.

**How teams actually share data:**
1. **Shared cloud database (usual real fix)** — everyone connects to the same managed instance; no sync problem exists.
2. **Export/import a dump** — e.g. `mongodump`/`mongorestore` or `pg_dump`/`pg_restore`, share the archive file, each dev imports it into their own local volume.
3. **Seed script on first run** — official images support an init-script folder that auto-runs the first time a fresh volume is created (`docker-entrypoint-initdb.d/` for both Mongo and Postgres images), giving every teammate identical starter data.

---

## 4. What Docker actually solves (and what it deliberately doesn't)

- **Docker's job:** identical software environment — same DB engine version, same config, same behavior, on every machine, for any DBMS.
- **Not Docker's job:** making data identical/shared across machines — that's handled separately, on purpose. (You wouldn't want production data auto-shipping into every dev's laptop via an image.)

### Dev/test data vs real/shared data

| | Dev/test data | Real/shared (production) data |
|---|---|---|
| Storage | Local, disposable Docker volume | Managed database service |
| Populated via | Seed/init script | Real application traffic |
| Safe to wipe? | Yes, anytime (`docker compose down -v`) | No — needs backups, replication |
| Shared across team? | No — each dev has their own copy of the same seed data | Yes — one instance everyone connects to |

---

## 5. Do professional teams use Docker for databases?

**Yes, mainly at these stages, regardless of which DBMS:**
- **Local development** — very common for Postgres, MongoDB, MySQL, Redis, etc. Spin up disposable DB containers, no manual install needed.
- **CI/testing pipelines** — fresh empty DB container per test run, discarded after.

**Less common in production**, because a plain `docker run -v` volume lacks backups, replication, and failover. When containers *are* used for databases in production, it's typically via an orchestrator like **Kubernetes**, backed by real persistent cloud storage — but this means the team owns all operational complexity (backups, scaling, patching, disaster recovery) themselves.

### Production options compared

| | Managed cloud service (Atlas, RDS, Cloud SQL...) | Self-managed (Kubernetes + containers) |
|---|---|---|
| Who runs the servers | The provider | Your team |
| Backups / scaling / failover | Built-in, automatic | You configure and maintain it |
| Effort | Low — just connect | High — full operational ownership |
| When teams choose this | Default choice for most teams | Teams with strong infra/compliance needs, or already deep in Kubernetes for everything |

---

## Bottom line

- Docker locally = disposable dev convenience, ensuring everyone runs the **same DB engine and version**, not the same data — true for Postgres, MongoDB, or any DBMS.
- In production, most teams hand the database off to a **managed cloud service** rather than containerizing it themselves.
- Self-managed containerized databases (via Kubernetes) are the exception — used by teams that need tighter control, not the default even among professional teams.
