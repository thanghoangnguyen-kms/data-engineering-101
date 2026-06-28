---
tags:
  - BE101
  - backend-7
  - docker
  - microservices
date: 2026-06-27
status: complete
domain: 7 of 8
track: backend
---
4
# B7 — Microservices & Containers

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll containerise a FastAPI application with Docker, orchestrate multi-service setups with Docker Compose, learn microservice architecture patterns, use gRPC for inter-service communication, and see how all B1–B7 skills come together in a production Clean Architecture codebase.

---

## 7.1 — Docker Fundamentals

> [!NOTE]
> A **container** packages your application with everything it needs to run — Python version,
> dependencies, OS libraries, config — into a portable, isolated unit. It runs identically
> on your laptop, a teammate's machine, or a cloud server.

### Container vs VM

|                     | Virtual Machine (VM)            | Container                        |
|---------------------|---------------------------------|----------------------------------|
| **Isolates**        | Full OS + kernel                | Process + filesystem             |
| **Size**            | GBs                             | MBs                              |
| **Start time**      | Minutes                         | Milliseconds                     |
| **Use case**        | Full server isolation           | Application process isolation    |

Containers share the host OS kernel — they are lighter than VMs but still fully isolated
from each other and from the host.

> [!IMPORTANT] Image vs Container
> An **image** is a blueprint — a static, read-only snapshot of the filesystem and config.
> A **container** is a running instance of that image. You can run many containers from the
> same image simultaneously (e.g., 3 replicas of your API behind a load balancer).

### Dockerfile Anatomy

A `Dockerfile` defines how to build an image, one instruction at a time:

```dockerfile
FROM python:3.11-slim          # base image — pulled from Docker Hub by default
WORKDIR /app                   # all subsequent commands run in /app

COPY pyproject.toml uv.lock .  # copy dependency files first (layer cache trick)
RUN pip install uv && uv sync --frozen --no-dev  # install dependencies
COPY . .                       # copy the rest of the source code

EXPOSE 8000                    # document the port (does not publish it)
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Each instruction creates a **layer**. Layers are cached — if `requirements.txt` hasn't
changed since the last build, Docker reuses the cached layer and skips reinstalling
dependencies. This is why you copy dependency files *before* copying source code.

### Multi-Stage Builds

A **multi-stage build** separates the heavy build environment from the lean runtime image.
The final image only contains what is needed to run the app — no build tools, no caches.

```dockerfile
# ── Stage 1: builder ─────────────────────────────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /app

# Copy uv binary from the official image
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Install dependencies into a virtual environment inside /app/.venv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

# ── Stage 2: runtime ─────────────────────────────────────────────────────────
FROM python:3.11-slim AS runtime

WORKDIR /app

# Copy only the venv and application code from the builder — no build tools
COPY --from=builder /app/.venv /app/.venv
COPY ./src ./src

# Add the venv's executables to PATH
ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> [!TIP] Why `python:3.11-slim`?
> `FROM python:3.11-slim` pulls the official `python` image at tag `3.11-slim` from
> **Docker Hub** (`hub.docker.com`) — the default public registry. Any image name without
> a hostname is fetched from Docker Hub automatically.
>
> The `-slim` variant strips documentation and test files from the base OS, giving a
> smaller image without sacrificing Python functionality. Avoid `-alpine` for Python — it
> uses `musl libc` which causes subtle compatibility issues with many compiled wheels.

### .dockerignore

Always create a `.dockerignore` so `COPY . .` doesn't pull in unnecessary files:

```
.venv/
__pycache__/
*.pyc
.git/
.env
*.egg-info/
.pytest_cache/
.mypy_cache/
```

### Essential Docker Commands

```bash
# Build an image and tag it "myapp:latest"
docker build -t myapp:latest .

# List all locally available images
docker image ls

# Run a container, mapping host port 8000 to container port 8000
docker run -p 8000:8000 myapp:latest

# Run detached (background) with a name
docker run -d --name api -p 8000:8000 myapp:latest

# List running containers
docker ps

# Follow logs from a container
docker logs -f api

# Open a shell inside a running container
docker exec -it api /bin/bash

# Stop and remove a container
docker stop api && docker rm api

# Remove an image
docker rmi myapp:latest
```

> [!WARNING] Common Docker Anti-Patterns
> ❌ Running the container process as `root` — if the process is compromised, an attacker
> gets root inside the container. Add a non-root user:
> ```dockerfile
> RUN addgroup --system app && adduser --system --ingroup app app
> USER app
> ```
>
> ❌ Copying the host `.venv/` into the image — binaries compiled on macOS won't run on
> Linux. Always install dependencies inside the image via `RUN uv sync`.
>
> ❌ Hardcoding secrets in `ENV` instructions inside the `Dockerfile` — they are visible
> in `docker history`. Pass secrets at runtime via environment variables or Docker secrets.

---

## 7.2 — Docker Compose & Multi-Service Setups

> [!NOTE]
> Running one container by hand is manageable. Running three containers that need to
> communicate, share volumes, and start in the correct order is where **Docker Compose**
> excels. A single `docker-compose.yml` defines and launches your entire local stack.

### A Complete `docker-compose.yml`

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://app:${POSTGRES_PASSWORD}@postgres:5432/appdb
      REDIS_URL: redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src   # dev-only: remove in production (code is baked into the image)

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
```

### Key Concepts

**Named volumes** (`postgres_data`) persist data across container restarts. Without a
named volume, all PostgreSQL data is lost every time the container stops.

**`depends_on` with `condition: service_healthy`** ensures the API container does not
start until Postgres is actually ready to accept connections — not just the moment the
container process starts. Without health checks, FastAPI may crash on startup trying to
connect before PostgreSQL has finished initialising.

**Service networking**: Compose creates a private network for all services in the file.
Each service is reachable by its **service name** as the hostname — `postgres:5432`,
`redis:6379`. You do not use `localhost` for container-to-container calls.

### Using a `.env` File

Compose automatically loads a `.env` file in the same directory. Keep secrets out of the
YAML and out of version control:

```bash
# .env (add to .gitignore — never commit this)
POSTGRES_PASSWORD=supersecret
API_SECRET_KEY=changeme
```

Commit a `.env.example` with placeholder values so teammates know what to set:

```bash
# .env.example (safe to commit)
POSTGRES_PASSWORD=
API_SECRET_KEY=
```

### Essential Compose Commands

```bash
# Build images and start all services in the background
docker compose up --build -d

# Follow logs from all services
docker compose logs -f

# Follow logs from one service only
docker compose logs -f api

# Stop all containers (volumes are kept)
docker compose down

# Stop all containers and delete volumes (wipes the database)
docker compose down -v

# Run a one-off command in a service container (e.g., run Alembic migrations)
# --rm removes the temporary container automatically once the command finishes
docker compose run --rm api alembic upgrade head

# Open a shell in the running api container
docker compose exec api /bin/bash
```

> [!WARNING] Common Compose Anti-Patterns
> ❌ Omitting `healthcheck` — `depends_on` without a health condition only waits for the
> container process to start, not for the service inside to be ready. Your API will crash
> connecting to a database still initialising.
>
> ❌ Using Docker Compose for production deployments — Compose is a local development and
> CI tool. In production, use Kubernetes or a managed container platform.
>
> ❌ Committing `.env` to Git — always add `.env` to `.gitignore`. Commit `.env.example`
> with empty or placeholder values instead.

---

## 7.3 — Microservice Architecture Patterns

> [!NOTE]
> A **microservice** is an independently deployable service that owns a single business
> domain. Understanding *when* and *how* to decompose a system is more valuable than
> knowing the tools — premature decomposition creates more problems than it solves.

### Start with a Monolith

> [!IMPORTANT] The Monolith-First Rule
> Build a monolith first. A well-structured monolith with clear module boundaries is
> easier to build, test, and operate than a prematurely decomposed system. Split only when
> the monolith becomes genuinely painful — not because microservices sound impressive.

The warning sign is not "this codebase is large." Look for one of these three signals:

| Signal                  | Description                                                                 |
|-------------------------|-----------------------------------------------------------------------------|
| **Team coupling**       | Two teams can't deploy independently because their code is entangled        |
| **Deployment coupling** | A change to the email module forces redeploying the entire order system     |
| **Scaling mismatch**    | Image processing needs 10× more resources than every other part of the app |

### Service Boundary Rules

Good microservice boundaries follow **domain ownership**: one service owns one business
domain *and its data store*. Services expose their data through APIs — they never share
a database table.

> [!WARNING] The Distributed Monolith Anti-Pattern
> ❌ Splitting code into separate services but having them share a single database:
> ```
> order-service ─┐
>                ├─ shared_postgres_db   ← anti-pattern
> user-service  ─┘
> ```
> This gives you all the operational complexity of microservices with none of the
> independence. Services can't evolve their schema without coordinating a joint deploy.
>
> ✅ Each service owns its own data store:
> ```
> order-service ── orders_db
> user-service  ── users_db
> ```

### Communication Styles

| Style                     | Protocol              | When to Use                                                   |
|---------------------------|-----------------------|---------------------------------------------------------------|
| **Sync — REST**           | HTTP/JSON             | External-facing APIs, simple internal CRUD                    |
| **Sync — gRPC**           | HTTP/2 + Protobuf     | Internal calls needing low latency or strict typed contracts  |
| **Async — Message queue** | AMQP / Redis Streams  | Fire-and-forget, event-driven flows, decoupled processing     |

**Synchronous** calls are simpler but create coupling — if the downstream service is
unavailable, the caller fails too.

**Asynchronous** calls via a message queue decouple services — the publisher sends an
event and moves on. The consumer processes it when it can, even if it was temporarily down.

### Common Microservice Patterns

| Pattern             | What It Solves                                                                 |
|---------------------|--------------------------------------------------------------------------------|
| **API Gateway**     | Single entry point for all external clients; handles auth, routing, rate limits |
| **Saga**            | Distributed transactions across multiple services. **Choreography**: each service listens for events and reacts. **Orchestration**: a central coordinator explicitly tells each service what to do next. |
| **CQRS**            | Separate read/write models; useful when reads and writes scale differently      |
| **Event Sourcing**  | Store all changes as immutable events rather than the current state             |
| **Circuit Breaker** | Stop calling a failing downstream service, return a fallback until it recovers  |

> [!TIP] You don't need all of these at once. API Gateway is almost always useful when
> you have multiple services. The others solve specific, concrete problems — add them when
> you encounter that problem, not speculatively.

### Monolith vs Microservices

| Concern                    | Monolith                          | Microservices                               |
|----------------------------|-----------------------------------|---------------------------------------------|
| **Deployment**             | Deploy one artefact               | Deploy N services independently             |
| **Testing**                | Easier — in-process function calls | Harder — need mocks or real service instances |
| **Operational complexity** | Low                               | High — N log pipelines, health checks, service discovery |
| **Scaling**                | Scale the whole app               | Scale individual services                   |
| **Team ownership**         | Shared codebase                   | Clear per-team ownership                    |
| **Inter-call latency**     | In-process function call          | Network hop                                 |
| **Best for**               | Small teams, early-stage products | Large orgs, independently scalable components |

> [!NOTE] What Is "Service Discovery"?
> In a microservice system, services need to find each other's network address at runtime.
> **Service discovery** is the mechanism that resolves a service name (e.g., `user-service`)
> to an actual host and port. Docker Compose handles this for you automatically — every
> service is reachable by its name on the Compose network. In production environments
> (e.g., Kubernetes), a built-in DNS resolver provides the same capability.

---

## 7.4 — gRPC & Inter-Service Communication

> [!NOTE] Building on B2 §2.6
> B2 introduced gRPC as an API style — protocol buffers, HTTP/2, and the REST comparison.
> Here you will implement actual service-to-service gRPC: defining a `.proto` contract,
> generating Python stubs, building an async server, and calling it from another FastAPI
> service.

### Why gRPC for Internal Calls

When `order-service` needs to fetch user details from `user-service`, gRPC is often the
right choice for internal calls:

- **Strict contract** — both sides agree on the exact message structure (the `.proto` file)
- **Binary serialisation** — Protobuf is 3–10× smaller and faster to parse than JSON
- **HTTP/2** — multiplexed connections, no head-of-line blocking
- **Auto-generated clients** — run `protoc` once, get a typed client stub

### Step 1 — Define the `.proto` Contract

The `.proto` file is the contract between services. Both sides generate code from the
same file — mismatches are caught at compile time, not at runtime.

```protobuf
// proto/user_service.proto
syntax = "proto3";

package user;

service UserService {
  // Unary RPC: one request, one response
  rpc GetUser (GetUserRequest) returns (UserResponse);

  // Server-streaming RPC: one request, stream of responses
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
}

message UserResponse {
  string user_id = 1;
  string email   = 2;
  string full_name = 3;
}
```

### Step 2 — Generate Python Stubs

```bash
uv add grpcio grpcio-tools
```

```bash
python -m grpc_tools.protoc \
  -I./proto \
  --python_out=./src/generated \
  --grpc_python_out=./src/generated \
  ./proto/user_service.proto
```

This creates two generated files — never edit them by hand:
- `user_service_pb2.py` — message classes (`GetUserRequest`, `UserResponse`, …)
- `user_service_pb2_grpc.py` — server base class and client stub

Create an `__init__.py` so Python treats the directory as a package:

```bash
touch src/generated/__init__.py
```

> [!WARNING] Fix the Generated Import
> The generated `user_service_pb2_grpc.py` uses a bare import that breaks in Python 3:
> ```python
> import user_service_pb2          # ❌ bare import — raises ModuleNotFoundError
> from . import user_service_pb2   # ✅ relative import — use this instead
> ```
> Open `src/generated/user_service_pb2_grpc.py` and change that line by hand after each
> `protoc` run. If you find this tedious, the
> [`betterproto`](https://github.com/danielgtaylor/python-betterproto) generator plugin
> produces correct Python 3 output with type annotations automatically.

### Step 3 — Implement the gRPC Server

```python
# src/grpc_server.py
import grpc

from src.generated import user_service_pb2, user_service_pb2_grpc
from src.domain.interfaces.user_repo import UserRepository


class UserServiceServicer(user_service_pb2_grpc.UserServiceServicer):
    def __init__(self, user_repo: UserRepository) -> None:
        self._repo = user_repo

    async def GetUser(
        self,
        request: user_service_pb2.GetUserRequest,
        context: grpc.aio.ServicerContext,
    ) -> user_service_pb2.UserResponse:
        user = await self._repo.get_by_id(request.user_id)
        if user is None:
            await context.abort(grpc.StatusCode.NOT_FOUND, "User not found")
        return user_service_pb2.UserResponse(
            user_id=str(user.id),
            email=user.email,
            full_name=user.full_name,
        )

    async def ListUsers(
        self,
        request: user_service_pb2.ListUsersRequest,
        context: grpc.aio.ServicerContext,
    ):
        async for user in self._repo.stream_all(page_size=request.page_size):
            yield user_service_pb2.UserResponse(
                user_id=str(user.id),
                email=user.email,
                full_name=user.full_name,
            )


async def serve(user_repo: UserRepository) -> None:
    server = grpc.aio.server()
    user_service_pb2_grpc.add_UserServiceServicer_to_server(
        UserServiceServicer(user_repo), server
    )
    server.add_insecure_port("[::]:50051")
    await server.start()
    await server.wait_for_termination()
```

### Step 4 — Call gRPC from Another FastAPI Service

```python
# src/clients/user_grpc_client.py
import grpc
from src.generated import user_service_pb2, user_service_pb2_grpc


class UserServiceClient:
    def __init__(self, host: str = "user-service", port: int = 50051) -> None:
        self._channel = grpc.aio.insecure_channel(f"{host}:{port}")
        self._stub = user_service_pb2_grpc.UserServiceStub(self._channel)

    async def get_user(self, user_id: str) -> user_service_pb2.UserResponse:
        return await self._stub.GetUser(
            user_service_pb2.GetUserRequest(user_id=user_id)
        )

    async def close(self) -> None:
        await self._channel.close()
```

Inject it into a FastAPI route via `Depends`:

```python
# src/api/orders.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel

from src.clients.user_grpc_client import UserServiceClient

router = APIRouter()


class OrderUserResponse(BaseModel):
    email: str
    full_name: str


def get_user_client() -> UserServiceClient:
    # In production, wire this through the IoC container (see §7.6)
    return UserServiceClient(host="user-service")


@router.get("/orders/{order_id}/owner", response_model=OrderUserResponse)
async def get_order_owner(
    order_id: str,
    client: UserServiceClient = Depends(get_user_client),
) -> OrderUserResponse:
    user = await client.get_user(user_id=order_id)
    return OrderUserResponse(email=user.email, full_name=user.full_name)
```

### Streaming Types

| Type                        | Pattern                            | When to Use                              |
|-----------------------------|------------------------------------|------------------------------------------|
| **Unary**                   | one request → one response         | Most RPCs: get, create, update           |
| **Server streaming**        | one request → stream of responses  | Large result sets, paginated exports, live feeds |
| **Client streaming**        | stream of requests → one response  | File uploads, batch ingestion            |
| **Bidirectional streaming** | stream ↔ stream                    | Real-time chat, live telemetry           |

### Inter-Service Communication Decision Table

| Requirement                                          | Use                            |
|------------------------------------------------------|--------------------------------|
| External API (browser, mobile app, third-party)      | REST (HTTP/JSON)               |
| Internal call — low latency, strict typed contract   | gRPC                           |
| Internal call — decouple services, fire-and-forget   | Message queue (B6 — Redis Streams / RabbitMQ) |
| Internal call — simple, no performance constraints   | REST                           |

> [!TIP] Keep your gRPC port separate from your HTTP port. Convention: HTTP on `8000`,
> gRPC on `50051`. Expose both in `docker-compose.yml` if other services need to reach
> the gRPC server.

---

## 7.5 — System Design Basics

> [!NOTE]
> This section covers the vocabulary and concepts behind large-scale backend systems.
> You won't build all of these at intern level — but you will encounter them in design
> discussions, technical reviews, and incident post-mortems from your first week.

### Scalability: Vertical vs Horizontal

|                     | Vertical Scaling                      | Horizontal Scaling                        |
|---------------------|---------------------------------------|-------------------------------------------|
| **What**            | Add more CPU/RAM to one machine        | Add more machines (replicas)              |
| **Limit**           | Single machine ceiling                 | Theoretically unlimited                   |
| **Complexity**      | Simple — no code changes              | Requires stateless services + load balancer |
| **Cost**            | Exponentially expensive at high end    | Roughly linear                            |

> [!IMPORTANT] Stateless = Horizontally Scalable
> A service that stores state in memory (e.g., user sessions in a Python dict) cannot
> scale horizontally — replica 2 won't have the session that replica 1 created. This is
> why storing session data in Redis (external, shared) instead of in-process memory is
> the correct pattern. Any replica can serve any request.

### Reliability Vocabulary

| Term     | Definition                                                                        |
|----------|-----------------------------------------------------------------------------------|
| **SLA**  | Service Level Agreement — the contractual uptime guarantee to customers           |
| **SLO**  | Service Level Objective — the internal target (usually stricter than SLA)         |
| **SLI**  | Service Level Indicator — the metric you actually measure (e.g., p99 latency, error rate) |
| **MTTR** | Mean Time To Recovery — how quickly you restore service after a failure           |
| **MTBF** | Mean Time Between Failures — how often failures occur                            |

*Example*: SLA says 99.9% uptime (≈ 8.7 hours downtime/year). Your SLO is 99.95% to
give yourself a buffer. Your SLI is the measured error rate from your monitoring system.

### Key Architectural Patterns

**Load Balancing** distributes incoming requests across multiple service replicas. Your
service must be stateless for this to work correctly — requests may hit different replicas.

**Caching Layers** store expensive query results in Redis (covered in B3). Cache what is
read frequently and changes rarely. Do not cache everything — stale data causes subtle bugs.

**Read Replicas** route read-heavy queries to a replica of the primary database, reducing
load on the primary. The trade-off is **replication lag** — replicas may return data that
is slightly behind the primary.

**CDN (Content Delivery Network)** caches static assets at edge nodes geographically
close to users. Relevant for static files and images, not for API responses.

### CAP Theorem

In a distributed system, you can guarantee at most two of three properties:

| Property                    | Meaning                                                       |
|-----------------------------|---------------------------------------------------------------|
| **Consistency (C)**         | Every read gets the most recent write                         |
| **Availability (A)**        | Every request receives a (non-error) response                 |
| **Partition Tolerance (P)** | The system continues operating when network partitions occur  |

Network partitions are inevitable in distributed systems — so the practical choice is
**CP vs AP**:

- **CP** (PostgreSQL, HBase): Prefer consistency. During a partition, some requests fail
  rather than return stale data.
- **AP** (Cassandra, DynamoDB): Prefer availability. During a partition, requests succeed
  but may return slightly stale data.

> [!TIP] Most web applications using a single-region PostgreSQL are not "distributed" in
> the CAP sense. CAP matters most when choosing a multi-region database or designing
> systems where the network between components can split.

### Designing for Failure

Real systems fail. Design with that assumption:

| Pattern                    | How It Helps                                                                |
|----------------------------|-----------------------------------------------------------------------------|
| **Timeouts**               | Don't wait forever for a downstream call — set a maximum deadline           |
| **Retries with backoff**   | Retry transient failures with exponential backoff + jitter, not immediately |
| **Circuit Breaker**        | After N failures, stop calling the failing service for a cooldown period, then retry |
| **Bulkhead**               | Isolate thread pools or connection pools per feature so one failure can't exhaust all resources |
| **Health checks**          | Expose `/health` so load balancers and Compose detect unhealthy instances   |

```python
# Minimal health check endpoint — add to every FastAPI service
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

> [!TIP] Observability: Logs, Metrics, Traces
> A health check tells you whether a service is alive. **Observability** tells you *why*
> it isn't. Three pillars: **structured logs** (JSON lines you can query — `structlog`),
> **metrics** (counters and histograms like request rate and p99 latency —
> `prometheus-client`), and **traces** (the full path of a request across all services —
> `opentelemetry-sdk`). You won't set these up at intern level, but expect to find all
> three in every production service you work on.

---

## 7.6 — Production Codebase Architecture

> [!IMPORTANT] How Real Backends Are Structured
> This section maps everything from B1–B6 to how a production Python service is actually
> organised. The goal is **Clean Architecture**: business logic that is independent of
> FastAPI, SQLAlchemy, Redis, and every other framework. This makes it testable, portable,
> and maintainable as the codebase grows.
>
> For the full platform specification with annotated production examples, see:
> [[docs/clean-architecture|Clean Architecture reference]]

### The 4-Layer Model

```
┌──────────────────────────────────────────────────────────────────────┐
│                           CONTROLLER LAYER                           │
│  FastAPI routes, middleware, event handlers                          │
│  HTTP in / HTTP out — no business logic lives here                   │
├──────────────────────────────────────────────────────────────────────┤
│                         ORCHESTRATION LAYER                          │
│  Use cases, DTO ↔ Entity mapping, transaction scope                  │
│  One use case class per user-facing action                           │
├──────────────────────────────────────────────────────────────────────┤
│                             DOMAIN LAYER                             │
│  Entities, value objects, business rules                             │
│  Repository interfaces — pure Python, no framework imports           │
├──────────────────────────────────────────────────────────────────────┤
│                         INFRASTRUCTURE LAYER                         │
│  SQLAlchemy repos, Redis client, gRPC clients, Alembic               │
│  Implements domain repository interfaces                             │
└──────────────────────────────────────────────────────────────────────┘
         Dependency direction: outer → inner  (never reversed)
```

**Dependency direction rule**: outer layers depend on inner layers. The domain layer
**never** imports from FastAPI, SQLAlchemy, Redis, or any framework. Infrastructure
implements interfaces that the domain defines.

> [!WARNING] Infrastructure Leaking into Domain
> ❌ Domain entity importing SQLAlchemy directly:
> ```python
> # src/domain/user.py — WRONG
> from sqlalchemy.orm import DeclarativeBase
>
> class User(DeclarativeBase):   # SQLAlchemy leaks into the domain
>     ...
> ```
> ✅ Domain entity is pure Python. The ORM model lives in infrastructure:
> ```python
> # src/domain/entities/user.py — correct
> from dataclasses import dataclass
>
> @dataclass
> class User:
>     id: str
>     email: str
>     full_name: str
> ```

### Production Directory Tree

```
my-service/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── alembic.ini
├── proto/
│   └── user_service.proto
└── src/
    ├── main.py                      # FastAPI app factory
    ├── container.py                 # IoC container (dependency-injector)
    ├── settings.py                  # Pydantic Settings
    │
    ├── api/                         # ── Controller layer ──────────────────
    │   ├── deps.py                  # FastAPI Depends() providers
    │   ├── users.py                 # /users routes
    │   └── orders.py                # /orders routes
    │
    ├── use_cases/                   # ── Orchestration layer ───────────────
    │   ├── create_user.py
    │   └── place_order.py
    │
    ├── domain/                      # ── Domain layer (no framework imports)
    │   ├── entities/
    │   │   ├── user.py              # User dataclass / value objects
    │   │   └── order.py
    │   └── interfaces/
    │       ├── user_repo.py         # Abstract repository (Protocol or ABC)
    │       └── order_repo.py
    │
    ├── infrastructure/              # ── Infrastructure layer ──────────────
    │   ├── database.py              # SQLAlchemy engine + session factory
    │   ├── models/                  # ORM models (separate from domain entities)
    │   │   ├── user_model.py
    │   │   └── order_model.py
    │   ├── repositories/            # Implement domain repository interfaces
    │   │   ├── user_repo.py
    │   │   └── order_repo.py
    │   └── clients/
    │       └── user_grpc_client.py
    │
    ├── generated/                   # gRPC stubs — never edit by hand
    │   ├── user_service_pb2.py
    │   └── user_service_pb2_grpc.py
    │
    └── migrations/                  # Alembic migrations
        ├── env.py
        └── versions/
```

### IoC Container with `dependency-injector`

The **IoC (Inversion of Control) container** wires your entire dependency graph in one
place. Instead of constructing repositories and use cases inside every route, you declare
them once and the framework injects them.

```bash
uv add dependency-injector
```

Two provider types used most often:

- **`providers.Singleton`** — creates the object **once** and reuses it for the lifetime of the app. Use for the DB engine and session factory — they are expensive to initialise and safe to share across requests.
- **`providers.Factory`** — creates a **fresh instance** on every injection. Use for repositories and use cases — each request should get its own instance to avoid shared mutable state.

```python
# src/container.py
from dependency_injector import containers, providers

from src.infrastructure.database import get_async_engine, get_async_session_factory
from src.infrastructure.repositories.user_repo import SQLAlchemyUserRepo
from src.use_cases.create_user import CreateUserUseCase


class Container(containers.DeclarativeContainer):
    wiring_config = containers.WiringConfiguration(modules=["src.api.users"])

    config = providers.Configuration()

    # Infrastructure
    db_engine = providers.Singleton(
        get_async_engine,
        url=config.database_url,
    )
    db_session_factory = providers.Singleton(
        get_async_session_factory,
        engine=db_engine,
    )

    # Repositories
    user_repo = providers.Factory(
        SQLAlchemyUserRepo,
        session_factory=db_session_factory,
    )

    # Use cases
    create_user_use_case = providers.Factory(
        CreateUserUseCase,
        user_repo=user_repo,
    )
```

```python
# src/main.py
from fastapi import FastAPI

from src.container import Container
from src.settings import Settings
from src.api import users


def create_app() -> FastAPI:
    settings = Settings()

    container = Container()
    container.config.from_pydantic(settings)
    container.wire(modules=[users])

    app = FastAPI()
    app.container = container  # type: ignore[attr-defined]
    app.include_router(users.router, prefix="/users")
    return app


app = create_app()
```

```python
# src/api/users.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from dependency_injector.wiring import inject, Provide

from src.container import Container
from src.use_cases.create_user import CreateUserUseCase


class CreateUserRequest(BaseModel):
    email: str
    full_name: str


class UserResponse(BaseModel):
    id: str
    email: str
    full_name: str


router = APIRouter()


@router.post("/", status_code=201, response_model=UserResponse)
@inject
async def create_user(
    body: CreateUserRequest,
    use_case: CreateUserUseCase = Depends(Provide[Container.create_user_use_case]),
) -> UserResponse:
    user = await use_case.execute(email=body.email, full_name=body.full_name)
    return UserResponse(id=user.id, email=user.email, full_name=user.full_name)
```

### uv Workspace Monorepo

When multiple services need to share common code (Pydantic schemas, auth utilities,
shared clients), use a **uv workspace** to manage them in a single repository:

```
my-platform/
├── pyproject.toml               # workspace root — lists all members
├── services/
│   ├── user-service/
│   │   └── pyproject.toml
│   └── order-service/
│       └── pyproject.toml
└── packages/
    └── shared-schemas/          # shared Pydantic models and utilities
        └── pyproject.toml
```

```toml
# Root pyproject.toml
[tool.uv.workspace]
members = [
    "services/*",
    "packages/*",
]
```

Each service declares the shared package as a dependency — resolved from the workspace,
not from PyPI:

```toml
# services/order-service/pyproject.toml
[project]
dependencies = [
    "shared-schemas",
]
```

```bash
# Run tests for one service
uv run --package user-service pytest

# Add a dependency to one service only
uv add --package order-service httpx
```

> [!TIP] Start with a single-package repository. Add a shared package only when two
> services genuinely need the same code — don't create `shared-schemas` speculatively.
> Premature sharing creates an invisible coupling between services.

---

## ✅ Practice Checklist

- [ ] Write a multi-stage `Dockerfile` for the FastAPI app using `python:3.11-slim` and `uv sync --frozen --no-dev`
- [ ] Add a `.dockerignore` that excludes `.venv/`, `__pycache__/`, `.git/`, and `.env`
- [ ] Build the image with `docker build` and confirm `docker run -p 8000:8000` serves the health endpoint
- [ ] Write a `docker-compose.yml` that starts the API, PostgreSQL, and Redis with `healthcheck` on both DB services
- [ ] Confirm `docker compose up --build -d` brings up the full stack and `docker compose logs -f` shows no errors
- [ ] Define a `.proto` file with at least one unary `rpc` method and generate Python stubs with `grpc_tools.protoc`
- [ ] Implement a gRPC servicer and call it successfully from a Python gRPC stub
- [ ] Draw a service architecture diagram for a system with at least 3 services, labelling sync (REST/gRPC) vs async (queue) calls
- [ ] Explain in one sentence each: when to use REST vs gRPC vs message queue for internal communication
- [ ] Map a sample FastAPI codebase to the 4-layer Clean Architecture model (controller / orchestration / domain / infrastructure)
- [ ] Wire one use case through the `dependency-injector` container and inject it into a FastAPI route with `Depends(Provide[...])`

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.docker.com | Docker — images, containers, CLI reference |
| https://docs.docker.com/compose/ | Docker Compose reference |
| https://docs.docker.com/develop/dev-best-practices/ | Docker best practices (non-root, multi-stage, .dockerignore) |
| https://grpc.io/docs/languages/python/ | gRPC Python — async server, stubs, streaming |
| https://python-dependency-injector.ets-labs.org | dependency-injector — IoC container wiring |
| [[docs/clean-architecture\|Clean Architecture reference]] | Platform's full Clean Architecture spec with annotated production code examples |
| https://docs.astral.sh/uv/concepts/workspaces/ | uv workspaces — monorepo setup |
| https://12factor.net | 12-Factor App — stateless services, config, logs |
| https://martinfowler.com/articles/microservices.html | Martin Fowler — canonical microservices introduction |
| https://microservices.io/patterns | Microservice patterns catalogue (Saga, CQRS, API Gateway, etc.) |

## 🃏 Quick-Reference Flash Cards

**Q:** What is the difference between a Docker image and a container?
**A:** An image is a static, read-only blueprint. A container is a running instance of that image. Multiple containers can run from the same image simultaneously.

**Q:** Why use a multi-stage Dockerfile?
**A:** To keep the final runtime image small — the build stage installs compilers and dev tools, and only the compiled artefacts and the virtual environment are copied into the lean runtime stage.

**Q:** Why does `depends_on` in Docker Compose need a `healthcheck`?
**A:** Without a health condition, `depends_on` only waits for the container process to start — not for the service inside to be ready. PostgreSQL may still be initialising when FastAPI tries to connect, causing a startup crash.

**Q:** What is the Distributed Monolith anti-pattern?
**A:** Splitting code into separate services while having them share a single database. This gives all the operational complexity of microservices with none of the independence benefits — services can't evolve their schema without a joint deployment.

**Q:** What are the three signals it's time to split a monolith into microservices?
**A:** (1) Team coupling — teams can't deploy independently. (2) Deployment coupling — changing one module forces redeploying everything. (3) Scaling mismatch — one component needs drastically more resources than the rest.

**Q:** When should you choose gRPC over REST for an internal service call?
**A:** When you need low latency, a strict typed contract between services, or you're in a polyglot environment where both sides generate clients from the same `.proto` file.

**Q:** When should you use a message queue instead of gRPC or REST?
**A:** When you want to decouple services so the caller doesn't wait for a response, or when the callee may be temporarily unavailable (fire-and-forget, event-driven processing).

**Q:** What is the dependency direction rule in Clean Architecture?
**A:** Outer layers (infrastructure, controllers) depend on inner layers (domain). The domain layer never imports from FastAPI, SQLAlchemy, or any other framework.

**Q:** What does an IoC container do?
**A:** It wires the entire dependency graph in one place. You declare how to construct repositories, use cases, and clients once in the container; `Depends(Provide[...])` injects them into routes automatically.

**Q:** What does CAP theorem say, and what is the practical trade-off?
**A:** A distributed system can guarantee at most two of: Consistency, Availability, Partition Tolerance. Since network partitions are inevitable, the real choice is CP (fail during a partition rather than return stale data) vs AP (return possibly stale data but stay available).

**Q:** What is a Circuit Breaker pattern?
**A:** After a downstream service fails N times in a row, the circuit "opens" — subsequent calls immediately return a fallback without waiting. After a cooldown period, a single probe request goes through to test recovery.

**Q:** What is a uv workspace used for?
**A:** Managing multiple Python packages (services + shared libraries) in a single repository. Shared packages are resolved from the workspace rather than PyPI, giving all services a consistent version without publishing.

*Checkpoint: [[Backend/Checkpoints/CB7 - Service Containerised|CB7]]*

*Previous: [[Backend/B6 - Async, Queues & Background Jobs|B6]] | Next: [[Backend/B8 - Capstone Project|B8]]*
