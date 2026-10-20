# Flash Sale Inventory Guard (distributed lock lab)

Educational demo: **distributed locking** with **Redlock** over **three independent Valkey masters** so concurrent consumers do not **oversell** a small in-memory stock during a **flash sale**. Inventory counts live **only in the inventory server process**; Valkey holds **lock keys** (`lock:inventory:{product_id}`), not stock.

This folder follows the monorepo layout in [docs/monorepo-practices.md](../../docs/monorepo-practices.md). The normative specification is [specs/01-distributed-lock.md](./specs/01-distributed-lock.md).

## Problem

Many worker processes handle orders at once. If each one **reads** the remaining count and **writes** a new count without coordination, races let more than `INITIAL_STOCK` sales succeed (**overselling**). The lab shows **acquire → critical section → release** around the HTTP call that decrements stock.

## Solution sketch

- One logical lock per `product_id`: key `lock:inventory:{product_id}` (default `product_id` = `flash-sale-001`, overridable with `PRODUCT_ID`).
- **Redlock** with **N = 3** unrelated Valkey **masters** (`valkey-1` … `valkey-3`) and **quorum = 2**: the client must succeed with `SET key token NX PX ttl` on **at least two** nodes for the same attempt.
- Each acquire uses a fresh **UUID v7** string (RFC 9562) as the **lock value** (`token`) on every node in that attempt.
- **TTL** (`LOCK_TTL_MS`, default **10 000 ms**) bounds how long a crashed holder can block others; the client computes **validity time** as in the **original Redlock** paper (TTL minus elapsed time minus a **clock drift** margin: \(0.01 \times \text{ttl} + 2\) ms). If validity is not positive before the critical section, the consumer **does not** call `POST /consume`, releases best-effort, and retries (subject to `MAX_ITERATIONS`).
- **Release** is **not** a blind `DEL`: a **Lua** script deletes the key **only if** the stored value equals the holder’s `token`, so a stale client cannot remove another holder’s lock on that node.

### Why not `DEL` alone?

If the key simply disappeared with `DEL`, any client could clear the lock—even after TTL expiry when another node already took over—so **compare-and-delete** with a **proof-of-ownership token** is required per node.

### Implementation note

The Redlock **acquire**, **partial unlock on failed quorum**, **validity** check, and **per-node Lua release** are implemented in code (`distributed_lock/redlock.py`) for transparency. **redis-py** is used for `SET … NX PX` and `EVAL` only; algorithmic policy is in this package.

## How to run

**Prerequisites:** Docker and Compose v2, Python **3.12+** for local installs (the **Dockerfile** uses `python:3.12-slim`).

### Full simulation (Compose)

From `services/distributed-lock`:

```bash
docker compose up --build
```

This starts three Valkey services, the **inventory** HTTP server on port **8000**, and one **consumer**. Scale consumers (each replica gets a distinct `CONSUMER_ID` from the container hostname when unset):

```bash
docker compose up --build --scale consumer=5
```

Environment defaults are listed in the spec (e.g. `INITIAL_STOCK=100`, `LOCK_TTL_MS=10000`, `MAX_ITERATIONS=1000`). Override with Compose or shell env vars.

### Local development

```bash
python3.12 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
pytest
```

Integration tests start **real Valkey** containers via **testcontainers**; **Docker** must be available (they are skipped if the daemon is not reachable).

### CI-style checks (same image as GitLab child pipeline)

```bash
docker compose run --rm builder
docker compose run --rm test
```

The **`test`** service mounts the Docker socket so testcontainers can spawn Valkey.

## Where to look in logs

- **Inventory server:** lines describing each successful decrement (`consumed` / remaining), and a clear **`exhausted`** line when stock hits zero (total units sold).
- **Consumers:** every line is prefixed with `[consumer_id=…]` — look for **`acquired`**, **`consumed`**, and **`exhausted`** when the API returns **409**.

## Redlock in this lab

- This repository implements **full Redlock** with **N = 3** independent Valkey **masters** and **quorum = 2**, following the **original Redlock** specification, including **validity time** and a **clock drift** margin on acquire.
- **Acquire:** the client issues **SET … NX PX** on **all three** nodes with the **same** `token`, counts successes, and grants the lock only if **≥ quorum** nodes accepted; otherwise it runs the **Lua release** on every node where `SET` succeeded so no partial lock remains.
- **Release:** the client runs the **Lua compare-and-delete** on **all three** nodes with the **same** `token` used for the successful acquire (best-effort per node).

### Production caveats (pedagogic)

Redlock’s assumptions include bounded **clock drift** and sensible **network latency**. In **multi-datacenter** deployments, RTT and failure patterns differ from a single-site lab. **Three single-node Valkey instances in Compose** illustrate the algorithm; they **do not** replace production **high availability** (e.g. Valkey **Cluster** / **Sentinel**, managed services, persistence, and operational runbooks). Treat this repo as a **learning sandbox**, not a production locking service.

## Layout

- **`distributed_lock/`** — inventory FastAPI app, Redlock client, consumer loop, CLI (`python -m distributed_lock inventory-server` / `inventory-consumer`).
- **`tests/`** — pytest (`T-01`–`T-04` per spec); **fakeredis** is not used for Lua tests.

## Console entry point

```bash
distributed-lock --help
```
