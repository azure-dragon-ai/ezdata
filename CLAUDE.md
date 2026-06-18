# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`ezdata` is a data processing, analysis, and task-scheduling system. It has a Python/Flask backend (`api/`) and a Vue 3 frontend (`web/`) derived from JeecgBoot-Vue3. The backend handles multi-source data connectors, a unified data model, LLM/RAG chat, low-code ETL pipelines, single/DAG task scheduling via Celery, and a sandboxed code-execution service.

## Repository Layout

- `api/` — Python Flask backend, Celery workers, scheduler, and sandbox.
- `web/` — Vue 3 + Vite + Ant Design Vue frontend.
- `deploy/` — Docker, Kubernetes, and local deployment assets.
- `docs/` — Project documentation.

## Backend (`api/`)

### Stack

- Flask + Flask-SQLAlchemy + Flask-CORS
- Celery + Redis (broker/backend) with `celery_once` for idempotency
- APScheduler for cron-style scheduling
- MySQL (default) for persistent state
- Elasticsearch for logs (when `LOGGER_TYPE=es`)
- MindsDB adapters for ETL source connectors

### Entry Points

| Service | File | Default Port | Purpose |
|---------|------|--------------|---------|
| Web API | `web_api.py` | 8001 | Main REST API; registers all blueprints from `blueprints.py`. |
| Scheduler API | `scheduler_api.py` | 8002 | APScheduler-based task scheduling service. |
| Sandbox API | `sandbox_api.py` | 8003 | Restricted Python/shell/data execution sandbox. |
| Celery Worker | `tasks/__init__.py` | — | Dispatches `normal_task`, `dag_task`, `dag_node_task`, etc. |
| Flower | `celery_app.py` | 5555 | Celery monitoring UI (`celery -A tasks flower`). |

### Configuration

Backend config is loaded from an environment file by `config.py`:

```python
dotenv_path = os.environ.get('ENV', 'prod.env')   # defaults to prod.env
load_dotenv(dotenv_path=dotenv_path)
```

Set `ENV=dev.env` (or any file in `api/`) and `read_env=1` to override. `dev.env` is provided as a development template. Key groups: `DB_*`, `REDIS_*`, `ES_HOSTS`, `OSS_*`/`S3_*`, `SANDBOX_*`, `LLM_*`, `RAG_*`.

### Common Backend Commands

Run from `api/`:

```bash
# Install dependencies
pip install -r requirements.txt -i https://pypi.doubanio.com/simple
pip install -r etl/requirements.txt -i https://pypi.doubanio.com/simple

# Run web API (port 8001)
python web_api.py

# Run scheduler API (port 8002)
python scheduler_api.py

# Run sandbox API (port 8003)
python sandbox_api.py

# Celery worker (Linux)
celery -A tasks worker

# Celery worker (Windows)
celery -A tasks worker -P eventlet

# Celery Flower monitoring
celery -A tasks flower
```

### Backend Architecture

- `web_apps/__init__.py` creates the Flask app and `db` (SQLAlchemy) instance.
- `blueprints.py` is the single registry of all API modules. Each entry maps a module path to a URL prefix (e.g., `web_apps.datasource.views.datasource_bp` → `/api/datasource`). Add a new domain by creating a `views.py` with a blueprint and registering it here.
- `models.py` defines the shared `BaseModel` (soft-delete, tenant, audit fields) and system tables (`User`, `Role`, `Depart`, `Permission`, etc.).
- `web_apps/<domain>/` typically contains:
  - `views/*.py` — Flask blueprints and route handlers.
  - `services/` — business logic.
- `tasks/` — Celery task definitions. `tasks/__init__.py` exposes the `task_dict` used by the scheduler and worker.
- `etl/` — Data-integration engine: `registry.py` resolves readers/writers, `transform_algs.py` contains built-in transform algorithms, and `etl_task.py` orchestrates extract/process/load batches.
- `utils/` — Shared utilities: auth, cache, DAG helpers, query builders, storage, sandbox helpers, logging, validation.
- `mindsdb/` — MindsDB handler integration for additional database connectors.

### Security Notes

- `sandbox_api.py` executes user-provided Python and shell commands. It uses `RestrictedPython` (when configured), AST-based dangerous-pattern filtering, and a configurable allow-list (`SANDBOX_ALLOWED_MODULES`).
- `SAFE_MODE` in `dev.env` controls whether dynamic code runs in the sandbox service instead of the main API process.

## Frontend (`web/`)

### Stack

- Vue 3, Vite 6, TypeScript 4.9
- Ant Design Vue 4, Vxe Table, Pinia, Vue Router 4
- JeecgBoot-Vue3 derived project structure (`src/api`, `src/views`, `src/router`, etc.)
- Jest for unit tests (only a smoke test exists currently)

### Configuration

- `.env.development` — dev server proxy targets the backend at `http://127.0.0.1:8001/api`.
- `vite.config.ts` — aliases `/@/` and `@/` to `src/`, and `/#/` and `#/` to `types/`.
- `tsconfig.json` — strict enabled, supports `tsx`/`vue`, paths match Vite aliases.

### Common Frontend Commands

Run from `web/`:

```bash
# Install dependencies (project uses pnpm in docs, npm --legacy-peer-deps also works)
pnpm install

# Dev server
pnpm dev

# Production build
pnpm build

# Build with bundle analyzer
pnpm build:report

# Preview production build
pnpm preview

# Format with Prettier
pnpm batch:prettier

# Run Jest smoke test
npx jest
```

### Frontend Architecture

- `src/main.ts` bootstraps the app, sets up Pinia, i18n, router guards, global components, and registers the LLM chat route.
- `src/api/` — HTTP API clients, grouped by domain (`sys`, `model`, `demo`, `common`).
- `src/views/` — Page components aligned with backend domains (`dataManage`, `task`, `llm`, `rag`, `alert`, `algorithm`, etc.).
- `src/store/modules/` — Pinia stores (app, user, permission, etc.).
- `src/router/` — Route definitions, guards, menu generation, and helper utilities.
- `src/components/` — Shared and globally registered components.
- `src/utils/http/` — Axios setup and interceptors.
- The project is also wired to run as a Qiankun micro-app when `VITE_GLOB_QIANKUN_MICRO_APP_NAME` is set.

## Database Initialization

SQL schema is in `api/sql/ezdata.sql`. Import it into MySQL before starting the services. Backend models in `api/models.py` can recreate tables with:

```bash
cd api
python -c "from web_apps import db, app; from models import *; app.app_context().push(); db.create_all()"
```

## Deployment

- `deploy/docker/` and `deploy/kubernetes/` contain orchestration assets.
- `api/Dockerfile` builds a Conda-based image, installs backend dependencies, and optionally installs MindsDB handlers via `install_handlers.py`.
- `api/docker-compose.yml` starts the full backend stack (web API, scheduler, Celery worker, Flower) using `api/init.sh` and `api/supervisord.ini`.

## Working Conventions

- Backend modules use Chinese comments and docstrings; preserve this style.
- Backend route blueprints are imported by string path via `utils.common_utils.import_class`.
- Frontend path aliases: `@/foo` and `/@/foo` → `src/foo`; `#/foo` and `/#/foo` → `types/foo`.
- Frontend build drops `console`/`debugger` in production (`esbuild.drop`).
