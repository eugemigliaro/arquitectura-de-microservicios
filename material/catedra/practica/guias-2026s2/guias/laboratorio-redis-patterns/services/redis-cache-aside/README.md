# redis-cache-aside

Educational service for the **cache-aside** pattern: a read-heavy **product** catalog backed by **PostgreSQL**, with **Valkey** (Redis-compatible) as cache and a **Strawberry GraphQL** API. Cache-aside for `product(id)` is implemented in a **Strawberry field extension** plus `CatalogService` (see `redis_cache_aside/graphql_schema.py` and `redis_cache_aside/catalog_service.py`).

**Normative spec:** [specs/01-redis-cache-aside.md](specs/01-redis-cache-aside.md).

## Prerequisites

- **Docker** and **Docker Compose v2** (`docker compose`).
- **Python 3.11+** for local installs (the **Dockerfile** uses **Python 3.12**; `.python-version` is **3.12**).

## Valkey image pin

Compose uses **`valkey/valkey:8.0.1`** (see `docker-compose.yaml`). This tag is the lab default so local runs and CI stay aligned.

## Environment variables

| Variable | Required | Default | Purpose |
|----------|----------|---------|---------|
| `DATABASE_URL` | yes | — | PostgreSQL DSN (asyncpg), e.g. `postgresql://catalog:catalog@postgres:5432/catalog` |
| `REDIS_URL` | no | `redis://127.0.0.1:6379/0` | Valkey / Redis URL |
| `CACHE_TTL_SECONDS` | no | `1800` | TTL for keys `catalog:product:{id}` (30 minutes in production demos) |
| `CACHE_ENABLED` | no | `true` | When `false`, every read loads from PostgreSQL (for comparisons / demos) |
| `HOST` | no | `0.0.0.0` | HTTP bind (uvicorn) |
| `PORT` | no | `8000` | HTTP port (uvicorn) |

## Start the stack

From `services/redis-cache-aside`:

```bash
docker compose up -d valkey postgres
docker compose up app
```

- **GraphQL HTTP:** `http://localhost:8000/graphql` (GraphiQL enabled).
- **Metrics JSON:** `GET http://localhost:8000/metrics` (cache hits/misses, DB reads, cache write failures).

Default Compose URLs inside the `app` service:

- `DATABASE_URL=postgresql://catalog:catalog@postgres:5432/catalog`
- `REDIS_URL=redis://valkey:6379/0`

## Seed data

On first start, PostgreSQL loads **`db/init.sql`** (mounted into `docker-entrypoint-initdb.d`). It creates `products` and inserts sample rows (`demo-1`, `hot-sku`). To re-seed, remove the Postgres volume and `docker compose up` again.

## Example GraphQL query

Minimum schema (spec section 4.1):

```graphql
query Product($id: ID!) {
  product(id: $id) {
    id
    title
    description
    price
  }
}
```

Example with curl:

```bash
curl -sS -X POST http://localhost:8000/graphql \
  -H 'content-type: application/json' \
  -d '{"query":"query P($id:ID!){ product(id:$id){ id title description price } }","variables":{"id":"demo-1"}}'
```

## Cache key and negative caching

- **Cache key (normative):** `catalog:product:{id}` where `{id}` is the same string as the GraphQL argument (spec section 3.2).
- **Unknown product:** `product(id)` returns **`null`**. The service **SHOULD** store a **negative cache** entry at the **same key** with TTL **`CACHE_TTL_SECONDS`**, using the JSON sentinel **`{"_absent":true}`** (compact encoding). Documented in code as `NEGATIVE_CACHE_JSON` / `NEGATIVE_CACHE_SENTINEL` in `redis_cache_aside/product_model.py`.

## Degraded Valkey / cache errors

**Recommended behavior (spec section 3.5):** if Valkey is unreachable or cache operations fail, **`product(id)`** still returns data from PostgreSQL when the row exists (cache is **best-effort**). The service does **not** fail the request solely because the cache failed. Failed cache writes are counted in **`cache_write_failures`** (see `/metrics`).

## Demo script (with vs without cache)

The **`redis-cache-aside-demo`** console script runs two in-process phases: `CACHE_ENABLED=false` (every read hits the DB) vs `CACHE_ENABLED=true` (cache-aside). It prints hits, misses, and DB read counts.

**Important:** the script needs the same env as the API (`DATABASE_URL`, `REDIS_URL`, etc.). With the default `app` service environment in `docker-compose.yaml`:

```bash
docker compose run --rm app python -m redis_cache_aside.demo
```

The narrative in **spec section 5.3** applies: a large reduction in DB reads for **hot keys** under skewed traffic is a **design goal**; the printed numbers illustrate the **mechanism**, not a production SLA. Informally, warm cache hits on localhost are often on the order of tens of milliseconds or less server-side, while cold PostgreSQL paths are slower (spec section 5.4: not a formal SLA).

## Run tests

**CI / full integration (Valkey + Postgres):** from this directory:

```bash
docker compose run --rm test
```

The `test` service sets `CACHE_TTL_SECONDS=2` so the **TTL expiry** integration test can use a short wait (spec section 6.1, mechanism A).

**Local (unit tests only):** without `DATABASE_URL`, integration tests are skipped:

```bash
cd services/redis-cache-aside
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
pytest
```

## Docker Compose services

| Service | Role |
|---------|------|
| `valkey` | Cache server |
| `postgres` | Catalog database + seed |
| `app` | ASGI API (`python -m redis_cache_aside`) |
| `builder` | Build smoke check |
| `test` | Pytest with dev deps against Valkey + Postgres |

## Package layout

- **`redis_cache_aside/catalog_service.py`** — cache-aside read path, metrics, Valkey fallback.
- **`redis_cache_aside/graphql_schema.py`** — Strawberry types, `product` query, `CacheAsideProductExtension`.
- **`redis_cache_aside/asgi.py`** — Starlette app, `/graphql`, `/metrics`, lifespan wiring.
