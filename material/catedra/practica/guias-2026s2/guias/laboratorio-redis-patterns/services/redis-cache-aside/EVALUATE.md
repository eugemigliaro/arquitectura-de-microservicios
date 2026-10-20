# Running evaluations (visual client + demo script)

These checks are **not pytest**. They run **inside Docker** against real **Valkey** and **PostgreSQL**, using the same images as the API.

**Specs:** [specs/02-cli-eval.md](specs/02-cli-eval.md) (visual eval), [specs/01-redis-cache-aside.md](specs/01-redis-cache-aside.md) (cache-aside behavior and metrics).

## Prerequisites

From `services/redis-cache-aside`:

- Docker and Docker Compose v2 (`docker compose`).

## Run everything (recommended)

```bash
./scripts/run-evaluations.sh
```

The script brings up **Valkey** and **Postgres**, then runs:

1. **Visual evaluation** — GraphQL-heavy consumer, table of metrics for cache off vs on.
2. **Catalog demo** — parent spec §5: direct `CatalogService` loop (same counters, no HTTP).

## Visual evaluation only

Start infra (if not already up):

```bash
docker compose up -d valkey postgres
```

Run the eval container (one-shot):

```bash
docker compose run --rm eval
```

### Environment variables (`eval` service)

| Variable | Default | Meaning |
|----------|---------|---------|
| `VISUAL_EVAL_REQUESTS` | `1000` | Number of GraphQL `product(id)` calls per phase. |
| `VISUAL_EVAL_PRODUCT_ID` | `demo-1` | Hot key; must exist in seeded `products` (see `db/init.sql`). |
| `VISUAL_EVAL_CONCURRENT` | `1` | `1` = sequential requests; larger values allow many in-flight requests (cold-cache race). |

Example: more requests, concurrent burst:

```bash
docker compose run --rm \
  -e VISUAL_EVAL_REQUESTS=5000 \
  -e VISUAL_EVAL_CONCURRENT=20 \
  eval
```

CLI flags override env for ad-hoc runs:

```bash
docker compose run --rm eval \
  python -m redis_cache_aside.visual_eval --iterations 200 --concurrent 1 --product-id hot-sku
```

### What you should see

The client prints a **comparison table** with **wall time**, **DB reads**, **cache hits**, **cache misses**, and **hit ratio**.

**Phase 1 — `CACHE_ENABLED=false` (no cache path):**

- Every read resolves through PostgreSQL (the service skips Valkey for this demo mode).
- Expect **`db_reads`** equal to **`VISUAL_EVAL_REQUESTS`** (one loader call per iteration).
- The reference implementation increments **`cache_misses`** once per read on this bypass path; **`cache_hits`** stays **0**. Use **`db_reads`** and the phase 2 **`hit_ratio`** as the main comparison.

**Phase 2 — `CACHE_ENABLED=true` after Valkey flush:**

- **Sequential** (`VISUAL_EVAL_CONCURRENT=1`): first request **misses** and loads the row; the rest should **hit** the cache for that product id.
  - Expect **`cache_misses` ≈ 1**, **`cache_hits` ≈ iterations − 1**, **`db_reads` ≈ 1**.
- **Concurrent** (many in-flight requests): several coroutines can **miss** before the cache is populated, so **`db_reads`** may be **greater than 1** but should still be **far below** the cache-off run for large iteration counts — compare the two rows side by side.

**Wall time:** On a laptop, the cache-on phase is often **faster** than cache-off for the same iteration count, but timing is **not** a formal benchmark (CPU, Docker, disk). Treat **DB read count** and **hit ratio** as the primary teaching signals; time is supporting evidence.

**Narrative:** The lab’s **~80% reduction in DB read load** for **skewed, hot-key** traffic is a **design goal** (see spec §2.1 / §5.3). This eval proves the **mechanism** on one hot key, not a production SLA.

## Demo script (§5, in-process catalog)

Same database and Valkey URLs; exercises `CatalogService` directly (prints counters without GraphQL):

```bash
docker compose run --rm eval python -m redis_cache_aside.demo
```

Expect the same qualitative story: **high `db_reads`** with cache off vs **one miss + many hits** (and **`db_reads` ≈ 1**) with cache on for repeated reads of **`demo-1`**.

## Pytest (for contrast)

Automated tests are separate:

```bash
docker compose run --rm test
```

See [README.md](README.md) for details.
