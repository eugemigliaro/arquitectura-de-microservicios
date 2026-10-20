# Monorepo practices

This repository follows [specs/00-monorepo-gitlab.md](../specs/00-monorepo-gitlab.md). It hosts multiple **Python** services under `./services/`, with shared norms in `./docs` and normative specs in `./specs`.

---

## 1. How to create a new service

1. **Choose a service name** and create `./services/<service-name>/`. The folder name is the canonical service name and MUST match `[a-z][a-z0-9-]+`, at most 40 characters, unique among `./services/*`.
2. **Scaffold a Python project** (Python **3.12+**):
   - Add a dependency manifest (`pyproject.toml`). Document how to install for local dev and for images.
   - Lay out application code under a clear package or module tree (import names cannot use hyphens; map `my-service` → `my_service` if needed).
   - Add **pytest**-discoverable tests (for example `tests/test_*.py`).
   - In **`pyproject.toml`**, define pytest only under **`[tool.pytest.ini_options]`** (test paths, markers, default CLI options). Do **not** use `pytest.ini` or `tox.ini` as the primary pytest configuration file.
3. **Pin Python** per service (for example `.python-version` and/or the base image tag) and mention it in the service README.
4. **Add a `Dockerfile`** and **`docker-compose.yaml`** in the service directory with Compose services named **`builder`** (build the runtime image) and **`test`** (run pytest with dev/test dependencies). See sections 2–3 below.
5. **Verify locally** from the service directory:

   ```bash
   docker compose run --rm builder
   docker compose run --rm test
   ```

---

## 2. How to set up the building process

- **`Dockerfile`** describes how the service image is built (base image, dependencies, copied code). The Compose **`builder`** service points at that file (and usually a **build target** that corresponds to the runtime image).
- **Build one service only:** `cd services/<service-name>` and run `docker compose build builder` (or `docker compose run --rm builder` if your `builder` command completes a build check). You do not need to build other services first; each service is **independent**.
- **Folder name → service name:** The directory under `./services/` is what this repo uses everywhere for identity. Image names SHOULD include that string (for example `my-service:local`) so multiple stacks on one machine stay distinguishable.
- **Independent build** means the service’s `Dockerfile` and `docker-compose.yaml` are self-contained under that folder: no step assumes another service’s image was built beforehand.

---

## 3. How to run the test process

- The Compose **`test`** service runs **pytest** in an environment with application code and test/dev dependencies (installed in the image or mounted—your service README should say which).
- **Pytest configuration** for each service lives only in that service’s **`pyproject.toml`** under **`[tool.pytest.ini_options]`**. That controls discovery (`testpaths`), markers, and default options—one place to look when tests are not found or flags differ from the CLI.
- **Extra pytest arguments:** with Compose, append after the service name, for example:

  ```bash
  docker compose run --rm test -- tests/test_foo.py -k bar
  ```

  (Exact behavior depends on whether the image’s `ENTRYPOINT`/`CMD` forwards arguments; adjust per service documentation.)

- **Tests that need Redis, databases, etc.:** add more services to the **same** `docker-compose.yaml` and document how to run them (for example `docker compose run --rm test` with dependencies started via `docker compose up -d redis`). This spec does not require those extras for every service.

---

## 4. Foundational concepts

- **Monorepo boundaries:** Runnable or deployable Python code lives under **`./services/<service-name>/`**. **`./docs`** holds human-facing guidance like this page. **`./specs`** holds normative specifications. Deployment, release pipelines, and artifact publishing are **out of scope** for the base spec; teams may add GitLab CI jobs later that `cd` into each service and run the same `docker compose` commands as locally.
- **pytest** is the standard test runner; configuration MUST be in each service’s **`pyproject.toml`** (`[tool.pytest.ini_options]`).
- **Compliance:** the repository root includes a **structural** pytest suite under `./tests/` that checks layout and config files. It does **not** build containers or run service tests; it keeps the compliance gate fast and dependency-free.

For future topics (GitLab CI YAML, shared internal packages, production monitoring), see the “Future work” section of the specification and any additional docs the project adds later.

---

## 5. CI (GitLab)

Normative detail: [specs/01-gitlab-service-ci.md](../specs/01-gitlab-service-ci.md). The repository root file `.gitlab-ci.yml` drives pipelines on GitLab **15+** with shared Linux runners that support **Docker** and **Docker Compose v2** (`docker compose`).

### Stages and jobs

1. **`compliance`** (stage `validate`) — from the repo root, runs `pytest tests/`. This is the **structural** suite only: it checks layout and required files. It does **not** build images or run per-service pytest. It is intended to fail fast before Docker jobs when the monorepo contract breaks.
2. **`prepare-child-pipeline`** (stage `prepare`) — runs `ci/prepare_child_pipeline.py`, which compares the current commit to **`origin/$CI_DEFAULT_BRANCH`** (typically `main`) and writes **`child-pipeline.yml`**. That file lists only the services that need Docker jobs for this pipeline (see “Path rules” below). This job can run in parallel with `compliance`.
3. **`trigger-service-pipelines`** (stage `services`) — starts a **child pipeline** from the generated `child-pipeline.yml` after `compliance` succeeds (`strategy: depend` propagates child success/failure to the parent). Inside the child:
   - **`build:<service>`** — `cd services/<service>` then `docker compose run --rm builder`, then `docker compose build test`, then `docker save` of all images present on the daemon (see the service CI spec).
   - **`test:<service>`** — `docker load` the artifact, then `docker compose run --rm test`. Each `test:<service>` job `needs:` its matching **`build:<service>`** with artifacts.

### Interpreting failures

| Failing job | Likely cause |
|-------------|----------------|
| `compliance` | Missing or invalid service layout (naming, `docker-compose.yaml`, `pyproject.toml` / `[tool.pytest.ini_options]`, etc.). Read the pytest output for the specific assertion. |
| `build:<service>` | Image build or the `builder` command failed; logs show whether Compose, Docker build, or the builder entrypoint failed. |
| `test:<service>` | Pytest failed inside the `test` service, or the image could not be loaded from artifacts. |

### Path rules and selection vs `main`

Selection is implemented in **`ci/prepare_child_pipeline.py`** (not GitLab `rules:changes` on a static matrix):

- On the **default branch** (push where the branch name equals `CI_DEFAULT_BRANCH`), or on the **first commit** bootstrap (`CI_COMMIT_BEFORE_SHA` all zeros), the child pipeline includes **every** directory under **`services/`**.
- Otherwise, `git diff --name-only origin/$CI_DEFAULT_BRANCH...HEAD` drives the set: paths under **`services/<name>/`** select that service; changes under **`.gitlab-ci.yml`**, **`ci/`**, **`tests/`**, or **`specs/`** force **all** services (shared impact). Changes under **`docs/`** alone do not add services to the child matrix; you may still get a child pipeline whose only job is **`no-service-changes`**.

Adding **`services/<new-name>/`** does **not** require editing a job matrix: the next pipeline that includes that tree in the diff (or any shared-path / default-branch run) will pick it up automatically once the folder exists on the branch.

### Caching and cost

This project uses **no persistent Docker layer cache** on shared runners: each job may pull base images and rebuild as needed. That keeps the configuration simple and correct; see Section 8 of the service CI spec for alternatives (dependency proxy, pip/npm caches, registry-based layer cache) if build times become a problem.
