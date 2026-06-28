---
tags:
  - BE101
  - backend-8
  - capstone
date: 2026-06-28
status: in-progress
domain: 8 of 8
track: backend
---

# B8 — Capstone Project

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll build a production-structured, two-service backend from scratch — integrating
> everything from B1–B7: REST API with FastAPI, Clean Architecture layers, JWT auth,
> PostgreSQL with async SQLAlchemy, inter-service messaging via Azure Service Bus,
> Docker-based event workers, and unit tests. There is no single "correct" project —
> pick any domain you care about. This is your portfolio piece.

---

## 8.1 — Project Specification

### Choosing Your Project

Pick any CRUD project from [roadmap.sh/backend/projects](https://roadmap.sh/backend/projects)
that naturally splits into **two services that trigger each other**. Good pairings:

| Service A          | Service B              | Trigger                              |
|--------------------|------------------------|--------------------------------------|
| Orders Service     | Inventory Service      | Order placed → reserve stock         |
| Tasks Service      | Audit Log Service      | Task completed → log activity        |
| Blog Service       | Notification Service   | Post published → notify subscribers  |

The key question: *"When something happens in Service A, does Service B need to react?"*
If yes, it's a good pairing.

### Core Constraints

| Requirement        | Detail                                                                |
|--------------------|-----------------------------------------------------------------------|
| **2 services**     | Naturally coupled via an async event                                  |
| **CRUD coverage**  | ≥ 3 endpoints per service (Create, Read, Update or Delete)            |
| **JWT auth**       | All write endpoints require a valid bearer token                      |
| **Event worker**   | Each service has a pub/sub worker via Azure Service Bus               |
| **Tests**          | Unit tests only — mock all external dependencies                      |
| **Docker**         | `docker compose up -d` starts infra + workers; APIs run via `uv run`  |

### Project Structure

Every service follows the same monorepo layout rooted from the workspace:

```
my-platform/
├── pyproject.toml                   ← uv workspace root
├── uv.lock
├── docker-compose.yml
├── .env
├── .env.example
├── local/
│   └── servicebus/
│       └── Config.json              ← topic/subscription definitions for the emulator
├── docker/
│   └── postgres-init/               ← optional SQL init scripts
├── services/
│   ├── service-a/
│   │   ├── pyproject.toml           ← service-a workspace member
│   │   ├── docker/
│   │   │   └── Dockerfile.eventworker
│   │   ├── service_a/               ← Python package (underscores, not hyphens)
│   │   │   ├── main.py
│   │   │   ├── worker.py
│   │   │   ├── ioc.py
│   │   │   ├── settings.py
│   │   │   ├── controller/
│   │   │   │   ├── api/
│   │   │   │   │   └── v1/
│   │   │   │   ├── events/
│   │   │   │   │   └── v1/
│   │   │   │   ├── middleware/
│   │   │   │   ├── dependencies/
│   │   │   │   └── responses/
│   │   │   ├── orchestration/
│   │   │   │   └── {context}/
│   │   │   │       ├── {context}_service.py
│   │   │   │       ├── {context}_mapper.py
│   │   │   │       └── dto/
│   │   │   ├── domain/
│   │   │   │   └── {context}/
│   │   │   │       ├── entities/
│   │   │   │       ├── interfaces/
│   │   │   │       ├── logic/
│   │   │   │       └── types/
│   │   │   └── infrastructure/
│   │   │       ├── repositories/
│   │   │       ├── messaging/
│   │   │       ├── external/
│   │   │       └── database.py
│   │   ├── tests/
│   │   │   └── unit/
│   │   │       ├── orchestration/
│   │   │       └── domain/
│   │   └── alembic/
│   └── service-b/                   ← identical layout
│       └── ...
└── shared/                          ← only if a DTO genuinely crosses both services
    └── shared_schemas/
        └── pyproject.toml
```

> [!TIP] When to use `shared/`
> Only move code to `shared/` when the *exact same* entity or DTO is consumed by both
> services. If Service B only needs 2 fields from Service A's event payload, define a
> lightweight local DTO in Service B — don't move Service A's full `ResponseOrderDTO`
> into `shared/`. Premature sharing creates tight coupling between independent services.

### Naming Conventions

Follow these exactly — they mirror the production codebase interns will join:

| Component                   | Pattern                        | Example                        |
|-----------------------------|--------------------------------|--------------------------------|
| Repository interface        | `I{Entity}Repository`          | `IOrderRepository`             |
| Repository implementation   | `SQLAlchemy{Entity}Repository` | `SQLAlchemyOrderRepository`    |
| Mapper                      | `{Entity}Mapper`               | `OrderMapper`                  |
| Domain logic class          | `{Entity}Logic`                | `OrderLogic`                   |
| Request DTO                 | `Request{Action}{Resource}DTO` | `RequestCreateOrderDTO`        |
| Response DTO                | `Response{Resource}DTO`        | `ResponseOrderDTO`             |

> [!IMPORTANT] `uv sync` after every dependency change
> Run `uv sync` from the workspace root whenever you:
> - Set up the project for the first time
> - Add or remove a package with `uv add` / `uv remove`
> - Pull changes from the repo that modified `uv.lock`
>
> Without this step, your virtual environment will be out of sync with the lockfile and
> you'll get confusing `ModuleNotFoundError` messages.

### docker-compose.yml Reference

Copy this to your project root and replace service names, DB names, and topic names
with your own:

```yaml
version: '3.8'

services:
  # ─── PostgreSQL ──────────────────────────────────────────────────────────────
  postgres:
    image: postgres:17-alpine
    container_name: platform-postgres
    ports:
      - "${DB_PORT:-5433}:5433"              # custom port — not default 5432
    environment:
      - POSTGRES_USER=${DB_USER:-postgres}
      - POSTGRES_PASSWORD=${DB_PASSWORD:-password}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres-init:/docker-entrypoint-initdb.d:ro
    command: ["postgres", "-c", "port=5433"]  # override port inside container
    networks:
      - platform-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${DB_USER:-postgres}", "-p", "5433"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  # ─── Redis ───────────────────────────────────────────────────────────────────
  redis:
    image: redis:7.2-alpine
    container_name: platform-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis_data:/data
    networks:
      - platform-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

  # ─── Redis Insight — optional debug UI ───────────────────────────────────────
  redisinsight:
    image: redislabs/redisinsight:latest
    container_name: platform-redisinsight
    ports:
      - "8082:5540"
    depends_on:
      - redis
    profiles:
      - debug                  # docker compose --profile debug up

  # ─── Azure Service Bus Emulator ──────────────────────────────────────────────
  sqledge:                     # required sidecar — Service Bus emulator needs SQL storage
    container_name: sqledge
    image: mcr.microsoft.com/azure-sql-edge:latest
    networks:
      - platform-network
    environment:
      ACCEPT_EULA: ${ACCEPT_EULA:-Y}
      MSSQL_SA_PASSWORD: "${SQL_PASSWORD:-StrongPassword1!23}"

  servicebus:
    container_name: servicebus-emulator
    image: mcr.microsoft.com/azure-messaging/servicebus-emulator:latest
    volumes:
      - "./local/servicebus/Config.json:/ServiceBus_Emulator/ConfigFiles/Config.json"
    ports:
      - "5672:5672"
      - "5300:5300"
    environment:
      SQL_SERVER: sqledge
      MSSQL_SA_PASSWORD: "${SQL_PASSWORD:-StrongPassword1!23}"
      ACCEPT_EULA: ${ACCEPT_EULA:-Y}
    depends_on:
      - sqledge
    networks:
      - platform-network

  # ─── Service A Event Worker ───────────────────────────────────────────────────
  service-a-event-worker:
    build:
      context: .                                            # workspace root
      dockerfile: services/service-a/docker/Dockerfile.eventworker
    container_name: platform-service-a-worker
    ports:
      - "8088:8088"
    environment:
      - ENVIRONMENT=${ENVIRONMENT:-development}
      - DB_HOST=postgres
      - DB_PORT=5433
      - DB_NAME=${SERVICE_A_DB_NAME:-service_a_db}
      - DB_USER=${DB_USER:-postgres}
      - DB_PASSWORD=${DB_PASSWORD:-password}
      - REDIS_URL=redis://redis:6379/0
      - SERVICE_BUS_CONNECTION_STRING=${SERVICE_BUS_CONNECTION_STRING:-Endpoint=sb://servicebus:5672;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true}
    depends_on:
      redis:
        condition: service_healthy
      servicebus:
        condition: service_started
    networks:
      - platform-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8088/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # ─── Service B Event Worker ───────────────────────────────────────────────────
  service-b-event-worker:
    build:
      context: .
      dockerfile: services/service-b/docker/Dockerfile.eventworker
    container_name: platform-service-b-worker
    ports:
      - "8089:8088"
    environment:
      - ENVIRONMENT=${ENVIRONMENT:-development}
      - DB_HOST=postgres
      - DB_PORT=5433
      - DB_NAME=${SERVICE_B_DB_NAME:-service_b_db}
      - DB_USER=${DB_USER:-postgres}
      - DB_PASSWORD=${DB_PASSWORD:-password}
      - REDIS_URL=redis://redis:6379/0
      - SERVICE_BUS_CONNECTION_STRING=${SERVICE_BUS_CONNECTION_STRING:-Endpoint=sb://servicebus:5672;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=SAS_KEY_VALUE;UseDevelopmentEmulator=true}
    depends_on:
      redis:
        condition: service_healthy
      servicebus:
        condition: service_started
    networks:
      - platform-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8088/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

networks:
  platform-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

> [!TIP] `sqledge` is a required sidecar
> The Azure Service Bus Emulator uses SQL Edge as its internal persistence store.
> It will fail to start without `sqledge` running first — hence `depends_on: [sqledge]`
> in docker-compose. You cannot remove this container.

> [!NOTE] `local/servicebus/Config.json` must exist before `docker compose up`
> The emulator reads this file at startup to create topics and subscriptions. Create
> this file before running `docker compose up`. See §8.2 for the exact format.

### Dev Workflow

```bash
# 1. Start all infrastructure + workers
docker compose up -d

# 2. Verify everything is healthy
docker compose ps

# 3. Run Service A API locally (Terminal 2)
cd services/service-a
uv run uvicorn service_a.main:app --reload --port 8000

# 4. Run Service B API locally (Terminal 3)
cd services/service-b
uv run uvicorn service_b.main:app --reload --port 8001
```

> [!IMPORTANT] APIs are NOT in docker-compose
> `service_a.main:app` and `service_b.main:app` run locally with `uv run` — not inside
> Docker. Only infrastructure and event workers are in docker-compose. This gives you
> fast hot-reload during development without rebuilding Docker images on every code change.

---

## 8.2 — Architecture & Event Flow

> [!NOTE]
> Read this section before writing any event-handling code. Understanding the runtime
> flow makes the implementation decisions obvious — where publishers live, what
> `True`/`False` means, why idempotency matters.

### System Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  Local (uv run)                                                  │
│                                                                  │
│  service-a :8000  ──── REST call ────▶  service-b :8001          │
│       │                                      │                   │
│       │ publish event                        │ publish event     │
└───────┼──────────────────────────────────────┼───────────────────┘
        │                                      │
        ▼                                      ▼
┌──────────────────────────────────────────────────────────────────┐
│  Docker Compose                                                  │
│                                                                  │
│  postgres  │  redis  │  sqledge + servicebus-emulator            │
│                                                                  │
│  service-a-worker  ◀──  service-b-events / service-a-subscription│
│  service-b-worker  ◀──  service-a-events / service-b-subscription│
└──────────────────────────────────────────────────────────────────┘
```

Each API publishes events to its own Service Bus topic. Each worker subscribes to the
*other* service's topic — so events cross service boundaries asynchronously, without
direct HTTP coupling.

### Layer Responsibilities

| Layer              | Package           | Owns                                                    |
|--------------------|-------------------|---------------------------------------------------------|
| **Controller**     | `controller/`     | HTTP routes, event handlers, middleware, dependencies   |
| **Orchestration**  | `orchestration/`  | Use-case coordination, DTO mapping, transactions        |
| **Domain**         | `domain/`         | Entities, business rules (`Logic`), repo interfaces     |
| **Infrastructure** | `infrastructure/` | DB sessions, SQLAlchemy repos, Service Bus publisher    |

No layer imports from a layer above it. Domain never imports from Infrastructure.
See [[docs/clean-architecture|Clean Architecture Guide]] for the full dependency rule.

### Patterns in Play

These four patterns appear throughout the codebase. They were introduced in
[[Backend/B3 - Databases & ORM|B3]] and [[Backend/B7 - Microservices & Containers|B7]]
— this is where you apply them all together for the first time.

| Pattern                 | Lives in                                          | What it does                                                          |
|-------------------------|---------------------------------------------------|-----------------------------------------------------------------------|
| **Repository**          | `domain/interfaces/` + `infrastructure/repositories/` | Abstracts DB access; domain depends on the interface, not SQLAlchemy |
| **Unit of Work**        | `orchestration/`                                  | Wraps a transaction; collects domain events to publish on commit      |
| **Mapper**              | `orchestration/{context}/`                        | Converts entities ↔ DTOs; keeps SQLAlchemy models out of routes       |
| **Dependency Injection**| `ioc.py`                                          | Wires all dependencies; routes receive services, not raw DB sessions  |

The **Unit of Work + Transactional Outbox** combination is the critical one for Milestone 5:
the UoW commits the DB write *and* publishes the event atomically — so you never get
a DB write without an event, or an event without a DB write.

```python
async with UnitOfWork(self._session_factory) as uow:
    await self._order_repo.save(order, session=uow.session)
    uow.add_event("order.created", {"order_id": str(order.id)})
    # On exit: commits to DB + publishes to Service Bus atomically
```

### Event Flow — Step by Step

```
1. HTTP request hits service-a route  (controller/api/v1/)
         │
2. Route calls {Context}Service       (orchestration/{context}/)
         │
3. Service persists entity via UnitOfWork → registers domain event
         │
4. On commit → infrastructure/messaging/ → publishes to Service Bus topic
         │
         └─── [async, decoupled] ──▶  service-b-worker receives message
                                              │
                                      5. controller/events/v1/ handler fires
                                              │
                                      6. Handler calls its own {Context}Service
                                              │
                                      7. Returns True  → acknowledge (message removed)
                                             Returns False → abandon (retry / dead-letter)
```

> [!IMPORTANT] Event handlers must be idempotent
> Azure Service Bus guarantees *at-least-once* delivery — a message may arrive more
> than once if the consumer crashes before acknowledging. Design handlers so that
> processing the same event twice produces the same outcome:
>
> ```python
> # ✅ Safe: INSERT ... ON CONFLICT DO NOTHING
> # ✅ Safe: check if record exists before inserting
> # ❌ Unsafe: always INSERT without checking — creates duplicates
> ```

### `local/servicebus/Config.json`

Create this file before running `docker compose up`. It defines all topics and
subscriptions for the emulator:

```json
{
  "UserConfig": {
    "Namespaces": [
      {
        "Name": "my-platform",
        "Topics": [
          {
            "Name": "service-a-events",
            "Subscriptions": [{ "Name": "service-b-subscription" }]
          },
          {
            "Name": "service-b-events",
            "Subscriptions": [{ "Name": "service-a-subscription" }]
          }
        ]
      }
    ]
  }
}
```

> [!TIP] Name topics after the producer, not the event
> `service-a-events` carries all events published by Service A. The handler inspects
> a `type` field in the message body (`"order.created"`, `"order.cancelled"`, etc.)
> to decide which function to call. One topic per service keeps the subscription model
> simple.

### `Dockerfile.eventworker`

The worker is a long-lived Python process — no Uvicorn, no HTTP request handling.
The build context is the **workspace root** so the Dockerfile can access both the
service source and any `shared/` packages:

```dockerfile
# services/service-a/docker/Dockerfile.eventworker
FROM python:3.11-slim AS builder
WORKDIR /build

RUN pip install --no-cache-dir uv

# Copy workspace manifests first — maximises Docker layer cache
COPY pyproject.toml uv.lock ./
COPY services/service-a/pyproject.toml ./services/service-a/

# Install only this service's production dependencies
RUN uv sync --frozen --no-dev --package service-a

# ─── Runtime stage ────────────────────────────────────────────────────────────
FROM python:3.11-slim
WORKDIR /app

# curl is needed for the healthcheck command in docker-compose
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/.venv /app/.venv
COPY services/service-a/service_a ./service_a

ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8088
CMD ["python", "-m", "service_a.worker"]
```

> [!WARNING] `CMD` is `python -m`, not `uvicorn`
> `Dockerfile.eventworker` starts `service_a.worker` — a plain async Python module
> that connects to Service Bus and listens for messages. It does NOT start Uvicorn.
> Keep `Dockerfile` (API) and `Dockerfile.eventworker` (worker) clearly separate.

---

## 8.3 — Implementation Milestones

> [!NOTE]
> Work through these milestones in order. Each has a clear done-state. Don't move to
> the next milestone until the current one is fully working.

> [!IMPORTANT] Run `uv sync` after every dependency change
> After `uv add`, `uv remove`, or pulling changes that touch `uv.lock`, always run
> `uv sync` from the workspace root. Skipping it causes `ModuleNotFoundError` for
> packages that exist in `pyproject.toml` but haven't been installed yet.

---

### Milestone 1 — Project Setup & Scaffold

**Goal:** Workspace initialised, all containers healthy, both `/health` endpoints responding.

```bash
# Create workspace root
uv init my-platform
cd my-platform

# Create each service as a workspace member
uv init services/service-a --package
uv init services/service-b --package

# Optional: shared schemas package (only if needed later)
# uv init shared/shared_schemas --package
```

Add `[tool.uv.workspace]` to the root `pyproject.toml`:

```toml
[tool.uv.workspace]
members = [
    "services/service-a",
    "services/service-b",
    # "shared/shared_schemas",   # uncomment only if needed
]
```

Install all dependencies for each service:

```bash
# Service A — production dependencies
uv add --package service-a \
    fastapi "uvicorn[standard]" \
    sqlalchemy asyncpg alembic \
    pydantic-settings \
    dependency-injector \
    PyJWT "passlib[bcrypt]" \
    azure-servicebus

# Service A — dev dependencies
uv add --package service-a --dev \
    pytest pytest-asyncio pytest-cov \
    ruff mypy

# Service B — same set
uv add --package service-b \
    fastapi "uvicorn[standard]" \
    sqlalchemy asyncpg alembic \
    pydantic-settings \
    dependency-injector \
    PyJWT "passlib[bcrypt]" \
    azure-servicebus

uv add --package service-b --dev \
    pytest pytest-asyncio pytest-cov \
    ruff mypy

# Sync the full workspace
uv sync
```

Add a skeleton `main.py` with a health endpoint to both services:

```python
# services/service-a/service_a/main.py
from fastapi import FastAPI

app = FastAPI(title="Service A")

@app.get("/health")
async def health() -> dict:
    return {"status": "ok", "service": "service-a"}
```

Create `local/servicebus/Config.json`, `.env.example`, and `docker-compose.yml` (see §8.1).

**Done when:**
- [ ] `docker compose up -d` → all containers show healthy in `docker compose ps`
- [ ] `curl http://localhost:8000/health` → `{"status":"ok","service":"service-a"}`
- [ ] `curl http://localhost:8001/health` → `{"status":"ok","service":"service-b"}`
- [ ] `uv sync` exits with no errors

---

### Milestone 2 — Service A: Domain, DB, CRUD

**Goal:** Service A has a working domain layer and ≥ 3 CRUD endpoints backed by PostgreSQL.

**Build in this order** — it keeps dependencies flowing in the right direction:

```
entity → repository interface → domain logic
→ SQLAlchemy model → repository implementation
→ request/response DTOs → mapper → orchestration service
→ IoC wiring (ioc.py) → FastAPI routes → Alembic migration
```

**Entity** — pure Python, no framework imports:

```python
# service_a/domain/order/entities/order.py
from dataclasses import dataclass, field
from uuid import UUID, uuid4
from datetime import datetime

@dataclass
class Order:
    customer_id: UUID
    total: float
    status: str = "pending"
    id: UUID = field(default_factory=uuid4)
    created_at: datetime = field(default_factory=datetime.utcnow)
```

**Repository interface** — domain defines the contract, infrastructure fulfils it:

```python
# service_a/domain/order/interfaces/i_order_repository.py
from abc import ABC, abstractmethod
from uuid import UUID
from ..entities.order import Order

class IOrderRepository(ABC):
    @abstractmethod
    async def get_by_id(self, id: UUID) -> Order | None: ...

    @abstractmethod
    async def list_all(self) -> list[Order]: ...

    @abstractmethod
    async def save(self, order: Order) -> Order: ...

    @abstractmethod
    async def delete(self, id: UUID) -> None: ...
```

Run migrations once the SQLAlchemy model and routes are in place:

```bash
cd services/service-a
uv run alembic revision --autogenerate -m "create orders table"
uv run alembic upgrade head
```

**Done when:**
- [ ] `POST /orders` creates a record persisted to PostgreSQL
- [ ] `GET /orders/{id}` returns the record
- [ ] `DELETE /orders/{id}` removes it
- [ ] Alembic migration applied cleanly (`uv run alembic current` shows the latest revision)

---

### Milestone 3 — Service B: Domain, DB, CRUD

**Goal:** Service B has its own domain and ≥ 3 CRUD endpoints, fully independent from Service A.

Follow the same build order from Milestone 2.

> [!TIP] The `shared/` decision
> Before writing DTOs in Service B, ask: *"Does Service B need the exact same DTO that
> Service A already defines?"* If Service B only needs `order_id` and `total` from an
> incoming event, define a lightweight local DTO — don't move Service A's full
> `ResponseOrderDTO` into `shared/`. Only reach for `shared/` when both services
> genuinely need an identical schema.

**Done when:**
- [ ] Service B has ≥ 3 CRUD endpoints backed by its own PostgreSQL schema
- [ ] Service B's Alembic migration applied cleanly
- [ ] Both services start and respond independently, without importing from each other

---

### Milestone 4 — JWT Authentication

**Goal:** Both services issue JWT tokens and all write endpoints reject unauthenticated requests.

**Password utilities:**

```python
# service_a/infrastructure/auth.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)
```

**Token creation and validation with `PyJWT`:**

```python
# service_a/infrastructure/jwt.py
import jwt
from datetime import datetime, timedelta, timezone
from service_a.settings import settings

def create_access_token(user_id: str) -> str:
    payload = {
        "sub": user_id,
        "exp": datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")

def decode_access_token(token: str) -> dict:
    # raises jwt.ExpiredSignatureError or jwt.InvalidTokenError on failure
    return jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
```

> [!WARNING] Use `PyJWT`, not `python-jose`
> `python-jose` is unmaintained and has known security issues. The correct package is
> `PyJWT` — it installs as `PyJWT` in `pyproject.toml` but imports as `import jwt`.
> Double-check you haven't accidentally installed `python-jose` from a tutorial.

**Login endpoint:**

```python
# service_a/controller/api/v1/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from service_a.infrastructure.auth import verify_password
from service_a.infrastructure.jwt import create_access_token

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/token")
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    user_repo: IUserRepository = Depends(get_user_repo),
) -> dict:
    user = await user_repo.get_by_username(form_data.username)
    if not user or not verify_password(form_data.password, user.hashed_password):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
        )
    return {"access_token": create_access_token(str(user.id)), "token_type": "bearer"}
```

**`get_current_user` dependency** — inject into any write route:

```python
# service_a/controller/dependencies/auth.py
import jwt
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from service_a.infrastructure.jwt import decode_access_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> str:
    try:
        payload = decode_access_token(token)
        return payload["sub"]
    except (jwt.InvalidTokenError, KeyError):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"},
        )
```

Add `Depends(get_current_user)` to every `POST`, `PUT`, `PATCH`, and `DELETE` route.

**Done when:**
- [ ] `POST /auth/token` with valid credentials returns a bearer token
- [ ] `POST /orders` without a token returns `401 Unauthorized`
- [ ] `POST /orders` with a valid token returns `201 Created`

---

### Milestone 5 — Inter-Service Event Flow

**Goal:** A write in Service A publishes an event → Service B's worker receives and persists it.

**Step 1 — Publisher in Service A (`infrastructure/messaging/`):**

```python
# service_a/infrastructure/messaging/publisher.py
import json
from azure.servicebus.aio import ServiceBusClient
from azure.servicebus import ServiceBusMessage

class ServiceBusPublisher:
    def __init__(self, connection_string: str, topic_name: str) -> None:
        self._client = ServiceBusClient.from_connection_string(connection_string)
        self._topic_name = topic_name

    async def publish(self, event_type: str, payload: dict) -> None:
        async with self._client.get_topic_sender(self._topic_name) as sender:
            message = ServiceBusMessage(
                json.dumps({"type": event_type, "data": payload})
            )
            await sender.send_messages(message)
```

**Step 2 — Event handler in Service B (`controller/events/v1/`):**

```python
# service_b/controller/events/v1/order_events.py
import json
import logging
from azure.servicebus import ServiceBusReceivedMessage

logger = logging.getLogger(__name__)

async def handle_order_created(
    message: ServiceBusReceivedMessage,
    notification_svc: NotificationService,
) -> bool:
    try:
        body = json.loads(str(message))
        order_id = body["data"]["order_id"]
        await notification_svc.register_order_event(order_id)
        return True    # ✅ acknowledge — message removed from queue
    except Exception as exc:
        logger.error("Failed to handle order.created: %s", exc)
        return False   # ❌ abandon — message retried or dead-lettered
```

> [!WARNING] Return value matters
> `True` → `complete_message()` — the message is permanently removed from the queue.
> `False` → `abandon_message()` — the message becomes visible again and will be
> retried up to the configured max delivery count, then moved to the dead-letter queue.
>
> Only return `False` for transient failures you expect to self-resolve (e.g., a
> downstream service is temporarily unavailable). For permanent failures like a bad
> payload format, log the error and return `True` — returning `False` on a malformed
> message causes an infinite retry loop.

**Step 3 — Worker entrypoint (`service_b/worker.py`):**

```python
# service_b/worker.py
import asyncio
import logging
from azure.servicebus.aio import ServiceBusClient
from service_b.settings import settings
from service_b.controller.events.v1.order_events import handle_order_created

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def main() -> None:
    logger.info("Service B worker starting...")
    async with ServiceBusClient.from_connection_string(
        settings.service_bus_connection_string
    ) as client:
        async with client.get_subscription_receiver(
            topic_name="service-a-events",
            subscription_name="service-b-subscription",
        ) as receiver:
            async for message in receiver:
                success = await handle_order_created(message, ...)
                if success:
                    await receiver.complete_message(message)
                else:
                    await receiver.abandon_message(message)

if __name__ == "__main__":
    asyncio.run(main())
```

**Done when:**
- [ ] `POST /orders` in Service A → Service B worker logs receipt within a few seconds
- [ ] The event record appears in Service B's database
- [ ] Sending the same event twice produces the same outcome (idempotency verified)

---

### Milestone 6 — Tests, Quality & README

**Goal:** Unit tests pass, quality gates green, README covers setup end-to-end.

**Unit test pattern** — mock all dependencies, test one unit at a time:

```python
# tests/unit/orchestration/test_order_service.py
from unittest.mock import AsyncMock
import pytest
from service_a.orchestration.order.order_service import OrderService
from service_a.orchestration.order.dto.request_create_order_dto import RequestCreateOrderDTO

@pytest.fixture
def mock_order_repo():
    repo = AsyncMock()
    repo.save.return_value = ...  # realistic Order instance
    return repo

@pytest.fixture
def order_service(mock_order_repo):
    return OrderService(order_repo=mock_order_repo)

@pytest.mark.asyncio
async def test_create_order_success(order_service, mock_order_repo):
    dto = RequestCreateOrderDTO(customer_id="...", total=99.99)
    result = await order_service.create(dto)
    mock_order_repo.save.assert_called_once()
    assert result.status == "pending"

@pytest.mark.asyncio
async def test_create_order_rejects_negative_total(order_service):
    with pytest.raises(ValueError, match="total must be positive"):
        await order_service.create(RequestCreateOrderDTO(customer_id="...", total=-1))
```

**Quality gates** — run these before every commit:

```bash
# From workspace root
ruff check services/
ruff format --check services/
mypy services/service-a/service_a
mypy services/service-b/service_b

# Per service
cd services/service-a
uv run pytest tests/unit --cov=service_a --cov-report=term-missing
```

**README sections to include:**

```markdown
## Setup

1. `cp .env.example .env` — fill in any required secrets
2. `uv sync`
3. Create `local/servicebus/Config.json` (copy template from docs)
4. `docker compose up -d` — wait for all containers to show healthy
5. Terminal 2: `cd services/service-a && uv run uvicorn service_a.main:app --reload --port 8000`
6. Terminal 3: `cd services/service-b && uv run uvicorn service_b.main:app --reload --port 8001`

## Architecture

[ASCII diagram or image showing Service A, Service B, Service Bus, workers]

## API Docs

- Service A: http://localhost:8000/docs
- Service B: http://localhost:8001/docs
```

**Done when:**
- [ ] `ruff check services/` exits clean
- [ ] `mypy` exits clean on both services
- [ ] `pytest tests/unit` passes on both services
- [ ] README covers setup, architecture, and API docs links

---

## ✅ Project Deliverables

Present this checklist to your mentor when you're ready for CB8 sign-off:

- [ ] GitHub repository with conventional commits and clean commit history
- [ ] 2 services, each with ≥ 3 CRUD endpoints
- [ ] JWT auth — all write endpoints return `401 Unauthorized` without a valid token
- [ ] PostgreSQL schema per service with Alembic migrations applied
- [ ] Event published on write in Service A → consumed and persisted in Service B
- [ ] Event handlers are idempotent (re-running the same event produces the same result)
- [ ] Unit tests pass for both services (`pytest tests/unit`)
- [ ] `ruff` and `mypy` exit clean on both services
- [ ] `docker compose up -d` starts all infra + workers with no errors
- [ ] APIs start cleanly with `uv run uvicorn`
- [ ] README covers setup, architecture diagram, and API docs links
- [ ] Intern can explain their architecture decisions in a 10-minute walkthrough

---

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| [roadmap.sh/backend/projects](https://roadmap.sh/backend/projects) | Project ideas and inspiration |
| [Azure Service Bus Emulator docs](https://learn.microsoft.com/azure/service-bus-messaging/overview-emulator) | Local Service Bus setup |
| [azure-servicebus Python SDK](https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/servicebus/azure-servicebus) | Publisher and receiver API |
| [PyJWT docs](https://pyjwt.readthedocs.io) | JWT encoding and decoding |
| [passlib docs](https://passlib.readthedocs.io) | Password hashing with bcrypt |
| [dependency-injector docs](https://python-dependency-injector.ets-labs.org) | IoC container wiring |
| [uv workspaces](https://docs.astral.sh/uv/concepts/workspaces/) | Multi-package project setup |
| [[docs/clean-architecture\|Clean Architecture Guide]] | Layer dependency rules |

---

## 🃏 Quick-Reference Flash Cards

**Q: What is a uv workspace?**
**A:** A single repository containing multiple Python packages, each with its own `pyproject.toml`. A root `pyproject.toml` with `[tool.uv.workspace]` declares all members. `uv sync` at the root installs and links all of them into one shared virtual environment.

---

**Q: Why does the Service Bus emulator require `sqledge`?**
**A:** The Azure Service Bus Emulator uses Microsoft SQL Edge as its internal persistence store. `sqledge` must be running before `servicebus` starts — hence `depends_on: [sqledge]` in docker-compose. You cannot remove this container.

---

**Q: Why do the API services run with `uv run`, not in docker-compose?**
**A:** For hot-reload during development. Running APIs locally means code changes take effect instantly. Only infrastructure (postgres, redis, sqledge, servicebus) and event workers are in docker-compose — workers have no benefit from hot-reload since they run as long-lived listeners.

---

**Q: What does returning `True` from an event handler mean?**
**A:** Success — the Service Bus client calls `complete_message()`, permanently removing the message from the queue. `False` calls `abandon_message()` — the message becomes visible again for re-delivery up to the max delivery count, then moves to the dead-letter queue.

---

**Q: Why must event handlers be idempotent?**
**A:** Azure Service Bus guarantees *at-least-once* delivery — a message may arrive more than once if the consumer crashes before acknowledging. Idempotent handlers (e.g., `INSERT ... ON CONFLICT DO NOTHING`) produce the same result whether called once or ten times.

---

**Q: What does the `I` prefix mean in `IOrderRepository`?**
**A:** Interface — a naming convention from the production codebase. `IOrderRepository` is the abstract contract (domain layer). `SQLAlchemyOrderRepository` is the concrete implementation (infrastructure layer). The orchestration layer depends only on the interface — never the SQLAlchemy class directly.

---

**Q: What goes in `shared/`?**
**A:** Only entities or DTOs that *both* services need in identical form. If Service B only needs 2 fields from an event payload, define a small local DTO in Service B. Premature sharing creates tight coupling between services that should remain independent.

---

**Q: How do you import `PyJWT`?**
**A:** `import jwt` — the package name in `pyproject.toml` is `PyJWT` but the Python module is `jwt`. Never use `python-jose` — it is unmaintained and has known vulnerabilities.

---

**Q: What is `Dockerfile.eventworker` for?**
**A:** It builds the event worker image — a long-lived Python process that listens to Service Bus messages. Its `CMD` is `python -m service_a.worker`, not `uvicorn`. It is separate from the regular `Dockerfile` which builds the API service.

---

**Q: When should you run `uv sync`?**
**A:** After `uv init`, after every `uv add` or `uv remove`, and after pulling changes that modify `uv.lock`. Always run it from the workspace root. Skipping it leaves your virtual environment out of sync with the lockfile.

*Checkpoint: [[Backend/Checkpoints/CB8 - Capstone Complete|CB8]]*

*Previous: [[Backend/B7 - Microservices & Containers|B7]]*
