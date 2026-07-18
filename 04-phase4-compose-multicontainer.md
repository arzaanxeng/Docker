# Phase 4 — Docker Compose: Running Multiple Containers Together

Real apps are rarely a single container. A typical ML API might need:

- Your FastAPI app
- A Postgres database (to log predictions, store users, etc.)
- A Redis cache (to cache expensive model calls)

Running each with separate `docker run` commands, manually wiring up networking and env vars, gets tedious and error-prone fast. **Docker Compose** lets you define your whole multi-container setup in one YAML file and spin it all up with one command.

## Installing Compose

Docker Desktop ships with Compose built in (`docker compose`, no hyphen, is the modern command — the old standalone `docker-compose` is legacy). Check:

```bash
docker compose version
```

## A basic `docker-compose.yml`

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/mydb
      - REDIS_URL=redis://cache:6379
    depends_on:
      - db
      - cache

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - db_data:/var/lib/postgresql/data

  cache:
    image: redis:7-alpine

volumes:
  db_data:
```

Run everything:

```bash
docker compose up
```

Run in the background (detached):

```bash
docker compose up -d
```

Stop and remove everything Compose created:

```bash
docker compose down
```

Stop but keep volumes (so your DB data survives):

```bash
docker compose down
# (volumes persist by default unless you add -v)
```

Remove volumes too (wipes DB data):

```bash
docker compose down -v
```

## What's actually happening under the hood

- Each `service` in the YAML becomes its own container.
- Compose automatically creates a private **network** shared by all services in that file.
- Inside that network, services can reach each other **by service name** as a hostname. Notice `DATABASE_URL` uses `db` (the service name), not `localhost` — `db` resolves via Docker's internal DNS to the database container's IP. This is the single most important thing to understand about Compose networking.
- `depends_on` controls **start order** only — it does *not* wait for Postgres to actually be ready to accept connections, just for the container to start. Your app should still handle "DB not ready yet" gracefully (retry logic), especially on first boot.

## `build:` vs `image:`

```yaml
api:
  build: .        # build from a local Dockerfile in this directory
db:
  image: postgres:16   # pull a prebuilt image from Docker Hub, don't build anything
```

Use `build` for your own app's code, `image` for off-the-shelf services (databases, caches, message queues) you don't need to customize.

## Volumes: making data survive container restarts

Containers are ephemeral by default — if you delete the `db` container, all data inside it is gone. A **volume** is Docker-managed persistent storage that lives outside the container's filesystem lifecycle.

```yaml
volumes:
  - db_data:/var/lib/postgresql/data
```

This maps a named volume `db_data` to the path Postgres stores its data at inside the container. Even if you `docker compose down` and `up` again, the data in `db_data` persists (unless you explicitly `down -v`).

There's also **bind mounts** — mapping a specific folder on your host machine into the container, useful during development so code changes reflect immediately without rebuilding:

```yaml
services:
  api:
    build: .
    volumes:
      - .:/app        # bind-mount your local code into the container
    ports:
      - "8000:8000"
```

Combined with `uvicorn app:app --reload`, this gives you live-reload development inside a container — edit code on your host, see changes immediately, without rebuilding the image each time.

**Rule of thumb**: bind mounts for your own source code during development; named volumes for data that needs to persist (databases, uploaded files).

## Environment variables and `.env` files with Compose

Compose automatically reads a `.env` file in the same directory and lets you reference variables with `${VAR_NAME}`:

`.env`:
```
DB_PASSWORD=supersecret
```

`docker-compose.yml`:
```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=${DB_PASSWORD}
```

Keep `.env` out of Git (`.gitignore` it) — it's for local secrets, same reasoning as Phase 3.

## Scaling a service (brief preview)

```bash
docker compose up -d --scale api=3
```

Runs 3 instances of the `api` service. You'd then need a load balancer in front of them for this to be useful in practice — this is really the entry point into orchestration tools like Kubernetes, which is out of scope here but good to know exists as "the next step up" from Compose for serious production scale.

## Useful Compose commands

```bash
docker compose ps                 # list running services in this project
docker compose logs -f api        # follow logs for just the 'api' service
docker compose exec api bash      # shell into the running 'api' container
docker compose restart api        # restart just one service
```

## You should now be able to...

- Write a `docker-compose.yml` for an app + database + cache setup
- Explain why services reach each other by service name, not `localhost`
- Distinguish named volumes (persistent data) from bind mounts (live code sync)
- Use `.env` files with Compose safely
- Use `docker compose logs -f` and `exec` to debug a multi-service setup

## Exercise

Extend your Phase 3 FastAPI + model project: add a Redis service via Compose, and modify one endpoint to cache its result in Redis (using service name `cache` as the hostname) so repeated identical requests skip recomputation. Verify with `docker compose logs -f api` that the second identical request is served from cache.

Next: [Phase 5 — ML-specific Docker: GPUs, multi-stage builds, and image size](05-phase5-ml-specific-docker.md)
