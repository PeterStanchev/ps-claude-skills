---
name: fastapi-skeleton
description: Scaffold or standardize a Python FastAPI service following the official "Bigger Applications" structure (incl. GPU/ML model-serving). Use when creating a new FastAPI app, adding routers/endpoints, wiring auth/CORS/config/health, building an inference or streaming API, or normalizing an existing FastAPI service's structure.
---

# FastAPI service skeleton

Default to the official [FastAPI "Bigger Applications"](https://fastapi.tiangolo.com/tutorial/bigger-applications/) structure — simple, official, scales to most services.

## Layout
```
app/
  __init__.py
  main.py            # create FastAPI app; include routers, middleware, logging, lifespan
  dependencies.py    # shared DI (auth, common params)
  routers/
    __init__.py
    <domain>.py      # one APIRouter per domain (users, items, …)
  internal/
    admin.py         # internal/admin routers
```

## Router pattern
- Each `routers/<domain>.py`: `router = APIRouter(prefix="/<domain>", tags=["<domain>"], dependencies=[Depends(...)])`; endpoints only.
- `main.py`: `app.include_router(<domain>.router)`; apply extra `prefix`/`dependencies` at include time when a router is reused.
- Import the module (`from .routers import users`) and use `users.router` to avoid name clashes.

## Conventions (best-practices)
- **Async-first**: an `async def` route does only non-blocking I/O; offload blocking/CPU work to a threadpool/executor — never block the event loop.
- **Pydantic v2**: a custom base model for app-wide config (datetime serialization, etc.); validate hard at the edge (constraints, enums, regex). Response models from ORM rows use `ConfigDict(from_attributes=True)`.
- **Dependencies for validation**: small, decoupled, composable deps (cached per request) — for auth AND request/constraint checks; enforce at the router, never trust the frontend.
- **Config**: `pydantic-settings BaseSettings`, read from env; never hardcode secrets; encrypt sensitive stored values at rest.
- **Middleware / logging**: `CORSMiddleware` (origins from config) + a request-id middleware; `logging.config.dictConfig` once; `log = logging.getLogger(__name__)` per module.
- **Lifespan**: load heavy resources (models, DB pools) in a `lifespan` handler, not at import time.
- **Health**: `GET /health` (liveness) + a deeper readiness check wired to the orchestrator probe.
- **Run**: `gunicorn -k uvicorn.workers.UvicornWorker app.main:app`, Dockerized, non-root.

## Model-serving / streaming addendum
- Load each model **once**; cache keyed by `(model_name, device, compute_type)` behind a lock. Long jobs: async job store + `progress_callback` (stage/percent/ETA). SSE (`StreamingResponse`) for token streaming with heartbeats + disconnect handling. Keep GPU/CPU work off the event loop.

## Scaling up
For a large multi-domain service, graduate to the domain-driven `src/<domain>/{router,service,schemas,models,dependencies,config,constants,exceptions,utils}.py` layout from [zhanymkanov/fastapi-best-practices](https://github.com/zhanymkanov/fastapi-best-practices) (business logic isolated in `service.py`, per-domain `BaseSettings`, snake_case `_at`/`_date` DB naming, SQL-first).

Sources: FastAPI official "Bigger Applications"; zhanymkanov/fastapi-best-practices.
