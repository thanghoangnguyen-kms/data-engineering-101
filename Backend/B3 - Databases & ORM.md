---
tags:
  - BE101
  - backend-3
  - database
  - orm
date: 2026-06-27
status: complete
domain: "3 of 8"
track: backend
---

# B3 — Databases & ORM

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll connect a FastAPI application to PostgreSQL using SQLAlchemy ORM (async), manage schema changes with Alembic migrations, add Redis caching, and learn three key data access patterns: Repository Pattern, Unit of Work, and Mapper Pattern.

---

## 3.0 — Project Setup & Configuration

> [!NOTE]
> Before connecting to a database, you need a safe way to manage configuration. Hard-coding database URLs in source code is a security risk — they end up in Git and get exposed. `pydantic-settings` gives you typed, validated config loaded from environment variables.

**Install dependencies**

```bash
uv add sqlalchemy asyncpg alembic redis pydantic-settings
```

| Package | Purpose |
|---------|---------|
| `sqlalchemy` | ORM + async engine |
| `asyncpg` | Async PostgreSQL driver (used by SQLAlchemy under the hood) |
| `alembic` | Versioned DB schema migrations |
| `redis` | Redis client — `redis.asyncio` for async access |
| `pydantic-settings` | Typed config loaded from `.env` and environment variables |

**Settings with `BaseSettings`**

```python
# app/core/config.py
from pydantic import ConfigDict
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    database_url: str
    redis_url: str = "redis://localhost:6379"

    model_config = ConfigDict(env_file=".env", env_file_encoding="utf-8")


settings = Settings()
```

Create a `.env` file at the project root:

```dotenv
# .env
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/mydb
REDIS_URL=redis://localhost:6379
```

> [!WARNING] Never commit `.env` to Git
> ❌ Adding `.env` to a repository — passwords and secrets end up in version history forever
> ✅ Add `.env` to `.gitignore`. Commit a `.env.example` with placeholder values so teammates know what variables to set.

> [!TIP] `.env` vs production
> On local dev you use a `.env` file. In production (Docker, cloud), you inject the same variables as real environment variables — `pydantic-settings` reads both identically. No code changes needed between environments.

**Updated project layout**

```
app/
├── main.py                  ← FastAPI instance, register routers
├── core/
│   ├── __init__.py
│   └── config.py            ← Settings class
├── database.py              ← engine, SessionLocal, get_db dependency
├── models/
│   ├── __init__.py
│   └── user.py              ← SQLAlchemy ORM models
├── schemas/
│   ├── __init__.py
│   └── user.py              ← Pydantic request/response schemas
├── repositories/
│   ├── __init__.py
│   └── user_repository.py   ← Repository pattern (section 3.6)
├── services/
│   ├── __init__.py
│   └── user_service.py      ← Business logic + caching (section 3.4)
└── routers/
    ├── __init__.py
    └── users.py
```

> [!IMPORTANT] `models/` vs `schemas/`
> Keep these separate. `models/` contains SQLAlchemy ORM classes — the database shape. `schemas/` contains Pydantic classes — the API shape. They serve different purposes. Mixing them into the same file creates tight coupling between your database structure and your API contract.

---

## 3.1 — PostgreSQL Fundamentals

> [!NOTE]
> PostgreSQL is the most widely used open-source relational database for backend applications. It's ACID-compliant, battle-tested, and the database you'll encounter most often in production backends.

**Run PostgreSQL with Docker**

No native installation needed. One command starts a local PostgreSQL instance:

```bash
docker run --name mydb \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  -d postgres:16
```

- `-p 5432:5432` — maps the container port to your machine
- `-d` — runs the container in the background

Stop and start again later:

```bash
docker stop mydb
docker start mydb
```

**Core SQL concepts**

| Concept | What it is | Example |
|---------|-----------|---------|
| **DDL** | Schema changes (structure) | `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE` |
| **DML** | Data manipulation | `INSERT`, `UPDATE`, `DELETE`, `SELECT` |
| **Primary Key** | Unique row identifier | `id SERIAL PRIMARY KEY` |
| **Foreign Key** | Links a row to a row in another table | `user_id INT REFERENCES users(id)` |
| **Index** | Lookup structure for fast queries | `CREATE INDEX idx_users_email ON users(email)` |

**SQL vs NoSQL**

| | PostgreSQL (SQL) | MongoDB / Redis (NoSQL) |
|--|-----------------|------------------------|
| **Data shape** | Fixed schema, typed columns | Flexible, schema-less documents |
| **Relationships** | First-class (`JOIN`, foreign keys) | Manual (embed or reference) |
| **Transactions** | Full ACID | Varies by product |
| **Use case** | User accounts, orders, billing | Logs, catalogs, caches, unstructured data |
| **When to choose** | Default for most applications | Specific performance or flexibility needs |

> [!TIP] Default to PostgreSQL
> If you're unsure which database to use, pick PostgreSQL. It handles the vast majority of use cases well, has excellent tooling, and you'll find it on almost every production backend you join.

**`psql` CLI basics**

Connect to your running instance:

```bash
psql -h localhost -U postgres -d mydb
```

Useful commands inside `psql`:

```
\dt              — list all tables
\d users         — describe the 'users' table (columns, types, constraints)
\l               — list all databases
\q               — quit
```

> [!WARNING] Production safety
> ❌ Running `DROP TABLE users` on a production database without a backup — irreversible data loss
> ✅ Always test destructive operations on dev/staging first. Schema changes go through Alembic migrations (section 3.3), not manual SQL commands.

> [!TIP] GUI tools
> `psql` is useful for quick inspections, but GUI tools make schema exploration much faster. [TablePlus](https://tableplus.com) (free tier) and [DBeaver](https://dbeaver.io) (free, open-source) are both excellent options.

---

## 3.2 — SQLAlchemy ORM (async)

> [!NOTE]
> An ORM (Object-Relational Mapper) lets you work with database rows as Python objects instead of writing raw SQL strings. SQLAlchemy 2.0 introduced a fully typed, async-first API — this is what you'll use throughout the backend track.

**What an ORM does**

```python
# ❌ Raw SQL — no type safety, fragile string construction
result = await conn.execute(
    "SELECT id, name, email FROM users WHERE id = $1", [user_id]
)
row = result.fetchone()
user = {"id": row[0], "name": row[1], "email": row[2]}

# ✅ ORM — typed, readable, refactor-safe
user = await session.get(User, user_id)  # returns a User object or None
```

**Engine and session factory**

```python
# app/database.py
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import (
    AsyncSession,
    async_sessionmaker,
    create_async_engine,
)

from app.core.config import settings

engine = create_async_engine(
    settings.database_url,
    echo=False,          # set True in dev to log every SQL statement
    pool_pre_ping=True,  # test connection health before use (see 3.5 for full pool config)
)

SessionLocal = async_sessionmaker(
    engine,
    expire_on_commit=False,  # keep attributes accessible after commit
)


async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        yield session
```

> [!IMPORTANT] `expire_on_commit=False`
> By default, SQLAlchemy marks all ORM attributes as "expired" after a `commit()`. In async code, accessing an expired attribute triggers a lazy load — which raises a `MissingGreenlet` error because the session may already be closed. `expire_on_commit=False` keeps attribute values accessible after commit.

**Defining ORM models**

```python
# app/models/user.py
import datetime

from sqlalchemy import ForeignKey, String, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True)
    credits: Mapped[int] = mapped_column(default=0)
    created_at: Mapped[datetime.datetime] = mapped_column(
        server_default=func.now()
    )

    posts: Mapped[list["Post"]] = relationship(
        "Post", back_populates="author", lazy="selectin"
    )


class Post(Base):
    __tablename__ = "posts"

    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str] = mapped_column(String(200))
    content: Mapped[str] = mapped_column(String)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"))

    author: Mapped["User"] = relationship("User", back_populates="posts")
```

`Mapped[T]` declares the Python type. SQLAlchemy infers the SQL type automatically: `Mapped[int]` → `INTEGER`, `Mapped[str]` → `VARCHAR`.

> [!IMPORTANT] `lazy="selectin"` is the async-safe default for relationships
> The default `lazy="select"` triggers a synchronous DB call when you access a relationship attribute, which raises `MissingGreenlet` in async code. Use `lazy="selectin"` to load related records automatically using a single `SELECT ... IN (...)` query — safe, efficient, and no extra effort in your route handlers.

**Async CRUD**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User


# Create
async def create_user(session: AsyncSession, name: str, email: str) -> User:
    user = User(name=name, email=email)
    session.add(user)
    await session.commit()
    return user  # id is populated after commit


# Read one
async def get_user(session: AsyncSession, user_id: int) -> User | None:
    return await session.get(User, user_id)


# Read many
async def list_users(session: AsyncSession) -> list[User]:
    result = await session.execute(select(User))
    return list(result.scalars().all())


# Update
async def update_user_name(
    session: AsyncSession, user_id: int, new_name: str
) -> User | None:
    user = await session.get(User, user_id)
    if user is None:
        return None
    user.name = new_name
    await session.commit()
    return user


# Delete
async def delete_user(session: AsyncSession, user_id: int) -> bool:
    user = await session.get(User, user_id)
    if user is None:
        return False
    await session.delete(user)
    await session.commit()
    return True
```

**N+1 problem and `selectinload`**

> [!WARNING] The N+1 Query Problem
> N+1 occurs when you load N records and then accidentally trigger a separate query for each one's related data — N+1 queries total instead of 2.
>
> ❌ Loading posts without eager loading — triggers one query per user:
> ```python
> users = (await session.execute(select(User))).scalars().all()
> for user in users:
>     # With lazy="select" (the default), this fires a new DB query per user!
>     titles = [p.title for p in user.posts]
> ```
>
> ✅ With `selectinload()` in the query — always 2 queries total:
> ```python
> from sqlalchemy.orm import selectinload
>
> result = await session.execute(
>     select(User).options(selectinload(User.posts))
> )
> users = result.scalars().all()
> for user in users:
>     titles = [p.title for p in user.posts]  # already loaded — no extra queries
> ```
>
> Setting `lazy="selectin"` on the relationship (as in the model above) applies this strategy globally. Use `selectinload()` explicitly in queries when you want control over which routes load posts and which don't.

**ORM → Pydantic response bridge**

After a DB query you have an ORM object. Your routes return Pydantic schemas. Use `ConfigDict(from_attributes=True)` to let Pydantic read ORM attributes directly:

```python
# app/schemas/user.py
from pydantic import BaseModel, ConfigDict


class PostResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    title: str
    content: str


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    name: str
    email: str
    posts: list[PostResponse] = []
```

Use in a route:

```python
user_orm = await session.get(User, user_id)
user_response = UserResponse.model_validate(user_orm)  # ORM object → Pydantic
```

> [!IMPORTANT] `from_attributes=True` is required
> Without it, `UserResponse.model_validate(user_orm)` raises a `ValidationError` — Pydantic v2 does not read object attributes by default. It expects a dict unless `from_attributes=True` is set.

**ORM vs raw SQL**

| Situation | Recommendation |
|-----------|----------------|
| CRUD on individual records | ORM — cleaner, safer, less code |
| Simple filtering and listing | ORM — `select(User).where(User.email == email)` |
| Complex analytics / aggregations | Raw SQL via `text()` — ORM can be awkward |
| Bulk inserts (thousands of rows) | `session.execute(insert(User), [...])` — much faster than ORM objects |
| Debugging a slow query | Set `echo=True` to see ORM-generated SQL, then optimise |

---

## 3.3 — Alembic Migrations

> [!NOTE]
> Alembic is to your database schema what Git is to your code. Every schema change — add a column, create a table, add an index — is a versioned migration file. Your whole team applies migrations in order so dev, staging, and production all stay in sync.

**Initialise Alembic**

```bash
alembic init alembic
```

This creates:

```
alembic/
├── env.py           ← migration runner — you will edit this
├── script.py.mako   ← template for new migration files
└── versions/        ← generated migration files live here
alembic.ini          ← top-level Alembic config
```

**Configure `alembic.ini`**

Leave the URL blank — you'll inject it from `settings` in `env.py`:

```ini
# alembic.ini
sqlalchemy.url =
```

**Rewrite `env.py` for async**

The default `env.py` is synchronous. Async SQLAlchemy requires rewriting `run_migrations_online()` to use `asyncio.run()`:

```python
# alembic/env.py
import asyncio
from logging.config import fileConfig

from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config

from app.core.config import settings
from app.models.user import Base  # import every file containing ORM models

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def do_run_migrations(connection) -> None:
    context.configure(connection=connection, target_metadata=target_metadata)
    with context.begin_transaction():
        context.run_migrations()


async def run_async_migrations() -> None:
    connectable = async_engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    async with connectable.connect() as connection:
        await connection.run_sync(do_run_migrations)
    await connectable.dispose()


def run_migrations_online() -> None:
    asyncio.run(run_async_migrations())


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

> [!IMPORTANT] Import every model file in `env.py`
> Alembic compares `Base.metadata` against the live database schema. If a model file isn't imported here, its tables won't be in `target_metadata` — autogenerate won't detect them and won't create or diff those tables.

**Create and run migrations**

```bash
# Generate a migration by comparing your models to the live DB schema
alembic revision --autogenerate -m "add users and posts tables"

# Apply all pending migrations
alembic upgrade head

# Roll back one step
alembic downgrade -1

# Roll back everything (empty schema)
alembic downgrade base

# See what version the DB is currently at
alembic current

# See full migration history
alembic history --verbose
```

> [!WARNING] Always review autogenerated migrations before running
> Alembic's autogenerate is smart but not perfect — it can miss some index changes or constraint modifications. Open the generated file in `alembic/versions/` and read it before running `upgrade head`.

> [!WARNING] Never edit an already-applied migration
> ❌ Editing a migration that teammates have already run — their DB will diverge from yours permanently
> ✅ Create a new migration: `alembic revision --autogenerate -m "fix column length"`

> [!TIP] Migration naming convention
> Use descriptive, action-oriented names: `add_users_table`, `add_index_on_email`, `add_credits_column_to_users`. Avoid: `fix`, `update`, `changes`. Future you will thank present you when reading `alembic history`.

---

## 3.4 — Redis & Caching Patterns

> [!NOTE]
> Redis is an in-memory key-value store. Reads are sub-millisecond — orders of magnitude faster than a PostgreSQL query. It's used for caching, session storage, rate limiting, and as a message queue backend (covered in [[Backend/B6 - Async, Queues & Background Jobs|B6]]).

**Run Redis with Docker**

```bash
docker run --name myredis -p 6379:6379 -d redis:7
```

**Connect with `redis.asyncio`**

```python
# app/database.py  (add to existing file)
import redis.asyncio as redis

from app.core.config import settings

redis_client = redis.from_url(settings.redis_url, decode_responses=True)
```

`decode_responses=True` returns Python strings instead of raw bytes.

**When to cache**

| Cache it ✅ | Don't cache it ❌ |
|------------|------------------|
| Expensive reads that change rarely | Data that must be real-time accurate |
| Public profiles, product listings | User-specific financial balances |
| Computed aggregations (stats, counts) | Data that changes on every write |
| Slow external API call results | Write-heavy resources |

**The cache-aside pattern**

Check cache first → on miss query the DB → populate the cache → return the result. The application controls caching explicitly.

```mermaid
sequenceDiagram
    actor Client
    participant Route as FastAPI Route
    participant Service as UserService
    participant Redis
    participant DB as PostgreSQL

    Client->>Route: GET /users/42
    Route->>Service: get_user(42)
    Service->>Redis: GET user:42
    alt Cache hit
        Redis-->>Service: JSON string
        Service-->>Route: UserResponse
        Route-->>Client: 200 OK (from cache)
    else Cache miss
        Redis-->>Service: nil
        Service->>DB: session.get(User, 42)
        DB-->>Service: User row
        Service->>Redis: SET user:42 <json> EX 300
        Service-->>Route: UserResponse
        Route-->>Client: 200 OK (from DB)
    end
```

> [!IMPORTANT] Cache in the service layer, not the route
> Route handlers are responsible for HTTP concerns: status codes, request parsing, error responses. Caching is a data access concern. Putting cache logic in a `UserService` means every consumer — API routes, background jobs, CLI commands — benefits from caching automatically.

**`UserService` with cache-aside**

```python
# app/services/user_service.py
import logging

from sqlalchemy.ext.asyncio import AsyncSession

from app.database import redis_client
from app.models.user import User
from app.schemas.user import UserResponse

logger = logging.getLogger(__name__)


class UserService:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_user(self, user_id: int) -> UserResponse | None:
        cache_key = f"user:{user_id}"

        # 1. Check cache — gracefully skip if Redis is down
        try:
            cached = await redis_client.get(cache_key)
        except Exception as e:
            logger.warning("Redis unavailable, falling back to DB: %s", e)
            cached = None

        if cached:
            return UserResponse.model_validate_json(cached)

        # 2. Cache miss — query DB
        user = await self.session.get(User, user_id)
        if user is None:
            return None

        # 3. Populate cache with TTL
        response = UserResponse.model_validate(user)
        try:
            await redis_client.set(cache_key, response.model_dump_json(), ex=300)
        except Exception:
            pass  # Cache write failure is non-fatal — request still succeeds

        return response

    async def update_user_name(self, user_id: int, new_name: str) -> UserResponse | None:
        user = await self.session.get(User, user_id)
        if user is None:
            return None
        user.name = new_name
        await self.session.commit()
        # Invalidate after successful DB commit
        try:
            await redis_client.delete(f"user:{user_id}")
        except Exception:
            pass  # Cache invalidation failure is non-fatal — DB is the source of truth
        return UserResponse.model_validate(user)
```

The route stays clean — no caching logic:

```python
# app/routers/users.py
@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
) -> UserResponse:
    service = UserService(db)
    user = await service.get_user(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

> [!TIP] Cache is not a critical dependency
> If Redis goes down, requests should still succeed — just slower. Always wrap Redis calls in `try/except` and fall through to the DB on failure. Production caching libraries like [Cashews](https://github.com/Krukov/cashews) handle this automatically via `suppress=True`.

**Cache key naming convention**

Use `resource:id` — consistent and easy to pattern-match for bulk deletion:

| Pattern | Example |
|---------|---------|
| Single record | `user:42` |
| Collection / page | `users:list:page:1` |
| Computed value | `user:42:post_count` |

**TTL guidelines**

| Data type | Recommended TTL | Reasoning |
|-----------|----------------|-----------|
| User profile | `1h` – `2h` | Changes infrequently |
| Project / entity metadata | `30m` – `1h` | Updated occasionally |
| List endpoints | `15m` – `30m` | Moderate write frequency |
| Real-time stats | `1m` – `5m` | Frequently updated |
| Config / system data | `1d` | Rarely changes |

**Invalidating the cache on writes**

> [!WARNING] Stale cache after updates
> If you update a user's name in PostgreSQL but forget to delete `user:42` from Redis, clients see the old name until the TTL expires.
>
> ❌ Updating the DB but ignoring the cache:
> ```python
> user.name = new_name
> await session.commit()
> # Redis still has the old name for up to 1 hour
> ```
>
> ✅ Delete the cached key **after** a successful commit:
> ```python
> user.name = new_name
> await session.commit()                            # DB write succeeds first
> await redis_client.delete(f"user:{user.id}")     # then invalidate cache
> ```
>
> Invalidating before commit risks a race: another request could repopulate the cache with stale data between the delete and the commit.

> [!WARNING] The Thundering Herd Problem
> When a popular cache key expires, every concurrent request misses simultaneously and all hit the DB at once — potentially hundreds of queries for the same data.
> ✅ Keep TTLs short on volatile data so fewer keys expire simultaneously
> ✅ Use background refresh jobs to repopulate before expiry (covered in [[Backend/B6 - Async, Queues & Background Jobs|B6]])
> ✅ Production caching libraries like [Cashews](https://github.com/Krukov/cashews) provide `lock=True` — coalescing concurrent misses into a single DB query

> [!TIP] Going further
> The production caching strategy builds on this foundation and includes: tag-based invalidation (no Redis `SCAN`), thundering herd protection with request coalescing, distributed locking, and idempotency keys. See [[docs/caching|docs/caching.md]] for the full implementation reference. The patterns covered in this section are the prerequisite — revisit this after completing B6 and B7.

---

## 3.5 — Connection Pooling & Performance

> [!NOTE]
> Establishing a new database connection takes 20–100ms and allocates server-side resources. Connection pooling maintains a set of open connections and reuses them across requests — essential under any real load.

**How pooling works**

```mermaid
flowchart LR
    subgraph App["FastAPI App (multiple requests)"]
        R1[Request 1]
        R2[Request 2]
        R3[Request 3]
    end
    subgraph Pool["Connection Pool (pool_size=3)"]
        C1[Conn 1]
        C2[Conn 2]
        C3[Conn 3]
    end
    PG[(PostgreSQL)]

    R1 --> C1
    R2 --> C2
    R3 --> C3
    C1 & C2 & C3 --> PG
```

Without pooling, each request opens and closes its own connection. With pooling, connections are borrowed from the pool and returned when the request finishes.

**SQLAlchemy pool parameters**

```python
# app/database.py
engine = create_async_engine(
    settings.database_url,
    pool_size=10,         # connections kept open at all times
    max_overflow=20,      # extra connections allowed during spikes (max total: 30)
    pool_timeout=30,      # seconds to wait for a free connection before raising
    pool_recycle=1800,    # recycle connections after 30 min to avoid stale ones
    pool_pre_ping=True,   # send SELECT 1 before use; recreate if connection is dead
    echo=False,
)
```

| Parameter | What it controls |
|-----------|-----------------|
| `pool_size` | Base connections kept open at all times |
| `max_overflow` | Extra connections allowed under load; closed when demand drops |
| `pool_timeout` | How long a request waits if all connections are in use |
| `pool_recycle` | Forces connection refresh — prevents "server closed the connection" after idle |
| `pool_pre_ping` | Checks connection health before use; fixes stale connections after Docker restarts |

> [!IMPORTANT] Create the engine once — never per request
> SQLAlchemy's `engine` IS the pool. Creating a new engine inside a route handler or dependency destroys pooling entirely and will exhaust your database connections under load.
>
> ❌ `engine = create_async_engine(...)` inside a route handler or `Depends()` function
> ✅ `engine = create_async_engine(...)` at module level in `database.py` — created once at startup

> [!WARNING] Diagnosing pool exhaustion
> Symptom: requests hang or time out under load. The error: `TimeoutError: QueuePool limit of size X overflow Y reached`.
>
> First steps:
> 1. Check `SHOW max_connections;` in PostgreSQL — your total pool must not exceed this
> 2. Confirm you're not creating engines per-request
> 3. Check for sessions never closed (missing `async with` in `get_db`)
> 4. Increase `pool_size` / `max_overflow` if load is genuinely high

**Finding slow queries with `EXPLAIN ANALYZE`**

When a query is slow, run `EXPLAIN ANALYZE` in `psql` to see the execution plan:

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'alice@example.com';
```

If the plan shows `Seq Scan` (full table scan) on a column you filter frequently, add an index:

```sql
CREATE INDEX idx_users_email ON users (email);
```

In Alembic, the same operation goes in a migration:

```python
op.create_index("idx_users_email", "users", ["email"])
```

---

> [!TIP] Advanced — Skip on First Pass
> Sections 3.6–3.8 cover the Repository Pattern, Unit of Work, and the Mapper/DTO pattern. These are production-grade architectural patterns. **On your first pass through B3, skip these and go straight to the Practice Checklist.** Return here before starting B7 — Clean Architecture in B7 builds directly on these concepts.

## 3.6 — Repository Pattern

> [!IMPORTANT] Why this pattern matters
> The Repository Pattern separates your data access logic from your business logic. You define an abstract interface in the domain layer and a concrete SQLAlchemy implementation in the infrastructure layer — so your business logic never depends on the database directly. This is the foundation of Clean Architecture in [[Backend/B7 - Microservices & Containers|B7]].

**The problem without it**

```python
# ❌ Business logic tangled with DB access — hard to test, hard to swap
@router.post("/users", status_code=201)
async def create_user(data: UserCreate, db: AsyncSession = Depends(get_db)):
    existing = await db.execute(select(User).where(User.email == data.email))
    if existing.scalar_one_or_none():
        raise HTTPException(status_code=409, detail="Email already in use")
    user = User(name=data.name, email=data.email)
    db.add(user)
    await db.commit()
    return UserResponse.model_validate(user)
```

This route knows about `select()`, `scalar_one_or_none()`, `db.add()`, and `db.commit()`. Testing it requires a real database.

**The abstract interface**

```python
# app/repositories/user_repository.py
from abc import ABC, abstractmethod

from app.models.user import User


class AbstractUserRepository(ABC):
    @abstractmethod
    async def get_by_id(self, user_id: int) -> User | None: ...

    @abstractmethod
    async def get_by_email(self, email: str) -> User | None: ...

    @abstractmethod
    async def create(self, name: str, email: str) -> User: ...

    @abstractmethod
    async def list_all(self) -> list[User]: ...

    @abstractmethod
    async def delete(self, user_id: int) -> None: ...
```

**The concrete SQLAlchemy implementation**

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from app.models.user import User
from app.repositories.user_repository import AbstractUserRepository


class SQLAlchemyUserRepository(AbstractUserRepository):
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    async def get_by_id(self, user_id: int) -> User | None:
        return await self.session.get(User, user_id)

    async def get_by_email(self, email: str) -> User | None:
        result = await self.session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()

    async def create(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self.session.add(user)
        await self.session.flush()  # assigns user.id without committing
        return user

    async def list_all(self) -> list[User]:
        result = await self.session.execute(select(User))
        return list(result.scalars().all())

    async def delete(self, user_id: int) -> None:
        user = await self.get_by_id(user_id)
        if user is not None:
            await self.session.delete(user)
```

> [!IMPORTANT] `flush()` vs `commit()` in the repository
> `flush()` sends the SQL to the database and populates generated IDs, but **does not commit** — the transaction stays open. `commit()` is the Unit of Work's responsibility (section 3.7). The repository operates within a transaction; it does not own one.

**How this enables testing**

```python
# In tests — no database required
class FakeUserRepository(AbstractUserRepository):
    def __init__(self) -> None:
        self._store: dict[int, User] = {}
        self._next_id = 1

    async def get_by_id(self, user_id: int) -> User | None:
        return self._store.get(user_id)

    async def get_by_email(self, email: str) -> User | None:
        return next((u for u in self._store.values() if u.email == email), None)

    async def create(self, name: str, email: str) -> User:
        user = User(id=self._next_id, name=name, email=email)
        self._store[self._next_id] = user
        self._next_id += 1
        return user

    async def list_all(self) -> list[User]:
        return list(self._store.values())

    async def delete(self, user_id: int) -> None:
        self._store.pop(user_id, None)

# Your service receives AbstractUserRepository — works with Fake or Real
```

> [!TIP] One repository per aggregate root
> In Domain-Driven Design, an **aggregate root** is the main entity that owns a cluster of related objects. `UserRepository` handles `User` and, through it, `Post`. Don't create a separate `PostRepository` if posts are always accessed through a user — this keeps your data access boundaries clean and your transaction logic simple.

---

## 3.7 — Unit of Work Pattern

> [!IMPORTANT] Transactions as a unit
> The Unit of Work pattern groups multiple repository operations into a single database transaction. If any step fails, the whole unit rolls back. In SQLAlchemy 2.0 async, this maps directly to the `AsyncSession` lifecycle.

**Why it matters**

```python
# ❌ Without Unit of Work — partial writes are possible
from_user.credits -= 100  # deducted
recipient.credits += 100  # what if this line raises?
# One account loses credits; the other never gains them — data corruption
```

**`AsyncSession` is the Unit of Work**

The `AsyncSession` holds all pending changes in memory. Nothing reaches the database until `commit()` is called. If an exception is raised, `async with SessionLocal()` triggers an automatic rollback.

```python
# app/database.py — the get_db dependency (already defined in 3.2)
async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        yield session  # one transaction scope per request
        # auto-rollback on unhandled exception; auto-close on exit
```

**Rollback demonstration**

```python
# app/services/user_service.py  (extending UserService from 3.4)
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import redis_client
from app.models.user import User
from app.repositories.user_repository import SQLAlchemyUserRepository
from app.schemas.user import UserResponse


class UserService:
    def __init__(self, session: AsyncSession) -> None:
        self.session = session

    # ... get_user, update_user_name from 3.4 ...

    async def transfer_credits(
        self,
        from_user_id: int,
        to_user_id: int,
        amount: int,
    ) -> None:
        repo = SQLAlchemyUserRepository(self.session)

        from_user = await repo.get_by_id(from_user_id)
        to_user = await repo.get_by_id(to_user_id)

        if from_user is None or to_user is None:
            raise ValueError("User not found")
        if from_user.credits < amount:
            raise ValueError("Insufficient credits")

        from_user.credits -= amount   # write 1 — pending in session
        to_user.credits += amount     # write 2 — pending in session

        await self.session.commit()   # both succeed atomically
        # If commit() raises — BOTH writes are rolled back automatically
```

> [!WARNING] Race Condition Without Locking
> The `transfer_credits` example above is correct for a single concurrent request. Under concurrent load, two transfers could both read `from_user.credits = 100`, both deduct 50, and both commit — resulting in a final balance of 50 instead of 0. This is called a **lost update**.
>
> Fix with pessimistic locking (`SELECT ... FOR UPDATE`), which locks the rows for the duration of the transaction:
>
> ```python
> from sqlalchemy import select
>
> async def transfer_credits(self, from_user_id: int, to_user_id: int, amount: int) -> None:
>     # Lock both rows for the duration of this transaction
>     from_user = await self.session.scalar(
>         select(User).where(User.id == from_user_id).with_for_update()
>     )
>     to_user = await self.session.scalar(
>         select(User).where(User.id == to_user_id).with_for_update()
>     )
>
>     if from_user is None or to_user is None:
>         raise ValueError("User not found")
>     if from_user.credits < amount:
>         raise ValueError("Insufficient credits")
>
>     from_user.credits -= amount
>     to_user.credits += amount
>     await self.session.commit()  # locks released on commit
> ```
>
> `with_for_update()` translates to `SELECT ... FOR UPDATE` in PostgreSQL. Any other transaction trying to read these rows will wait until this commit completes. Use for any operation where correctness depends on the value you just read (financial transfers, inventory decrement, seat reservation).

**Wiring into FastAPI**

```python
# app/routers/users.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from app.database import get_db
from app.services.user_service import UserService

router = APIRouter(prefix="/users", tags=["users"])


@router.post("/{from_id}/transfer/{to_id}", status_code=200)
async def transfer(
    from_id: int,
    to_id: int,
    amount: int,
    db: AsyncSession = Depends(get_db),
) -> dict:
    service = UserService(db)
    try:
        await service.transfer_credits(from_id, to_id, amount)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    return {"status": "ok"}
```

**Layer responsibilities**

```
Route handler  →  calls Service     →  calls Repository  →  uses Session (UoW)
    ↑                  ↑                      ↑                    ↑
HTTP layer       business logic           data access        transaction boundary
```

> [!WARNING] Never share a session across requests
> ❌ A module-level `session = SessionLocal()` used by all requests — concurrent writes corrupt each other's transactions
> ✅ `get_db()` yields a fresh session per request — every request has its own isolated transaction scope

> [!TIP] Commit in the service layer, not the repository
> Repositories only `flush()` to get generated IDs. The service layer calls `commit()` when a complete unit of work is done. This keeps the transaction boundary at the right level — the service decides when a unit of work is complete, not the repository.

---

## 3.8 — Mapper Pattern

> [!NOTE]
> The **Mapper Pattern** converts data between layers so each layer speaks its own language.
> SQLAlchemy models are a database concern — they shouldn't leak into your routes.
> Pydantic request/response DTOs are an HTTP concern — they shouldn't reach your domain.
> A mapper sits between them and translates cleanly in both directions.

### Why Mappers Matter

Without mappers, you end up with SQLAlchemy models passed directly into Pydantic schemas
or routes importing database models. This couples your HTTP layer to your DB schema:

```python
# ❌ Anti-pattern — route directly returns the ORM model
@router.get("/users/{id}")
async def get_user(id: UUID, session: AsyncSession = Depends(get_db)):
    user = await session.get(UserModel, id)
    return user          # leaks DB model into the HTTP response
```

```python
# ✅ Correct — route returns a response DTO; mapper handles the conversion
@router.get("/users/{id}")
async def get_user(id: UUID, user_svc: UserService = Depends(get_user_service)):
    return await user_svc.get_by_id(id)   # returns ResponseUserDTO — no SQLAlchemy
```

### Mapper Implementation

A mapper is a plain class with static methods — no state, no dependencies:

```python
# app/orchestration/user/user_mapper.py
from app.domain.user.entities.user import User
from app.orchestration.user.dto.request_create_user_dto import RequestCreateUserDTO
from app.orchestration.user.dto.response_user_dto import ResponseUserDTO

class UserMapper:
    @staticmethod
    def to_entity(dto: RequestCreateUserDTO) -> User:
        """DTO → Domain Entity (used before saving)"""
        return User(
            email=dto.email,
            full_name=dto.full_name,
        )

    @staticmethod
    def to_response_dto(entity: User) -> ResponseUserDTO:
        """Domain Entity → Response DTO (used before returning from route)"""
        return ResponseUserDTO(
            id=entity.id,
            email=entity.email,
            full_name=entity.full_name,
            created_at=entity.created_at,
        )
```

The orchestration service calls the mapper — it never exposes raw entities or ORM models
to callers:

```python
# app/orchestration/user/user_service.py
class UserService:
    def __init__(self, user_repo: IUserRepository, session_factory) -> None:
        self._user_repo = user_repo
        self._session_factory = session_factory

    async def create(self, dto: RequestCreateUserDTO) -> ResponseUserDTO:
        entity = UserMapper.to_entity(dto)
        async with UnitOfWork(self._session_factory) as uow:
            saved = await self._user_repo.save(entity, session=uow.session)
        return UserMapper.to_response_dto(saved)

    async def get_by_id(self, id: UUID) -> ResponseUserDTO:
        entity = await self._user_repo.get_by_id(id)
        if entity is None:
            raise NotFoundError(f"User {id} not found")
        return UserMapper.to_response_dto(entity)
```

### DTO Naming Conventions

Follow these naming patterns — they match the production codebase you'll work in:

| DTO Type       | Pattern                        | Example                   |
|----------------|--------------------------------|---------------------------|
| Create request | `Request{Action}{Resource}DTO` | `RequestCreateUserDTO`    |
| Update request | `Request{Action}{Resource}DTO` | `RequestUpdateUserDTO`    |
| Response       | `Response{Resource}DTO`        | `ResponseUserDTO`         |

DTOs are plain Pydantic models — they carry data across layer boundaries, nothing more:

```python
# app/orchestration/user/dto/request_create_user_dto.py
from pydantic import BaseModel, EmailStr

class RequestCreateUserDTO(BaseModel):
    email: EmailStr
    full_name: str

# app/orchestration/user/dto/response_user_dto.py
from pydantic import BaseModel
from uuid import UUID
from datetime import datetime

class ResponseUserDTO(BaseModel):
    id: UUID
    email: str
    full_name: str
    created_at: datetime
```

> [!IMPORTANT] Data flows one way through mappers
> The direction is always:
> ```
> HTTP request body → RequestDTO → Mapper.to_entity() → Domain Entity → Repository
> Repository → Domain Entity → Mapper.to_response_dto() → ResponseDTO → HTTP response
> ```
> SQLAlchemy models stay in `infrastructure/`. Domain entities stay in `domain/`.
> DTOs stay in `orchestration/`. Nothing crosses these boundaries without a mapper.

> [!TIP] One mapper per domain context
> `UserMapper` handles `User` entities. `OrderMapper` handles `Order` entities. Keep
> them small and focused — if a mapper is getting complex, that's usually a sign the
> domain boundary needs rethinking.

---

## 🎯 What You Learned

You can now:

- **Connect FastAPI to PostgreSQL** — async SQLAlchemy engine, session factory, `Depends(get_db)` for per-request sessions, and `asyncpg` as the high-performance driver
- **Define ORM models and run migrations** — `DeclarativeBase`, `Mapped[T]` typed columns, and Alembic `autogenerate` to version schema changes safely
- **Cache with Redis** — cache-aside pattern, TTL-based invalidation, write-through vs write-behind trade-offs, and the thundering herd problem
- **Apply the Repository + Unit of Work pattern** — abstract DB access behind interfaces so services never import SQLAlchemy directly, enabling unit testing with fakes
- **Prevent data races in concurrent writes** — `SELECT ... FOR UPDATE` for pessimistic locking, and why atomic transactions are not enough under concurrent load

---

## ✅ Practice Checklist

- [ ] Configure `pydantic-settings` to load `DATABASE_URL` and `REDIS_URL` from a `.env` file
- [ ] Define SQLAlchemy ORM models for at least 2 related tables with a foreign key relationship
- [ ] Rewrite Alembic's `env.py` for async and run `alembic upgrade head` successfully
- [ ] Run an Alembic migration to add a new column to an existing table (`alembic revision --autogenerate`)
- [ ] Implement working CRUD endpoints (create, read, update, delete) against PostgreSQL using async SQLAlchemy
- [ ] Add `ConfigDict(from_attributes=True)` to a Pydantic response schema and convert an ORM object using `model_validate()`
- [ ] Implement cache-aside in a `UserService`: first `GET` hits the DB and caches the result; second `GET` returns from Redis without a DB query
- [ ] Delete the Redis cache key after an update operation to prevent stale data
- [ ] Configure `pool_size`, `max_overflow`, and `pool_pre_ping` on the engine
- [ ] Implement `AbstractUserRepository` and `SQLAlchemyUserRepository` with fully typed method signatures
- [ ] Wrap two repository operations in a single `AsyncSession` transaction and verify that a failure in the second rolls back the first
- [ ] Create a `UserMapper` with `to_entity()` and `to_response_dto()` static methods and use it in a service so no SQLAlchemy model is returned from a route

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html | SQLAlchemy 2.0 async ORM |
| https://alembic.sqlalchemy.org/en/latest/ | Alembic migrations |
| https://redis.io/docs | Redis documentation |
| https://www.postgresql.org/docs/ | PostgreSQL reference |
| https://docs.pydantic.dev/latest/concepts/pydantic_settings/ | `pydantic-settings` |

## 🃏 Quick-Reference Flash Cards

**Q:** What is an ORM?
**A:** A library that maps database rows to Python objects. You query and manipulate data through typed objects instead of writing raw SQL strings.

**Q:** Why use `expire_on_commit=False` in `async_sessionmaker`?
**A:** Without it, SQLAlchemy marks all attributes as expired after `commit()`. Accessing them in async code triggers a lazy load that raises `MissingGreenlet` if the session is already closed.

**Q:** What is the N+1 problem?
**A:** Loading N records and then triggering a separate DB query for each one's related data — N+1 queries instead of 2. Fix with `selectinload()` or `lazy="selectin"` on the relationship.

**Q:** What does `alembic revision --autogenerate` do?
**A:** Compares `Base.metadata` (your models) to the live DB schema and generates a migration file with the differences. Always review the generated file before running `upgrade head`.

**Q:** What is the cache-aside pattern?
**A:** Check cache → hit: return cached value. Miss: query DB → store in cache with TTL → return value. The application manages caching explicitly — neither the DB nor the cache does it automatically.

**Q:** What does `pool_pre_ping=True` do?
**A:** Before using a connection from the pool, SQLAlchemy sends `SELECT 1`. If the connection is dead (e.g., after a Docker restart), it discards it and creates a fresh one.

**Q:** What is the Repository Pattern?
**A:** An abstraction between business logic and the database. An abstract class defines the operations; a concrete class implements them with SQLAlchemy. Business logic calls the interface — never the DB directly.

**Q:** What is the Mapper Pattern?
**A:** A class with static methods that converts between layers — `to_entity()` turns a request DTO into a domain entity before saving; `to_response_dto()` turns a domain entity into a response DTO before returning from a route. It keeps SQLAlchemy models out of routes and Pydantic DTOs out of the domain.

---

**Q:** What is the Unit of Work pattern?
**A:** Grouping multiple write operations into one transaction so they all succeed or all fail together. In SQLAlchemy async, the `AsyncSession` is the Unit of Work — nothing commits until you call `await session.commit()`.

**Q:** What is `flush()` vs `commit()` in a repository?
**A:** `flush()` sends SQL to the DB and populates generated IDs, but keeps the transaction open. `commit()` makes changes permanent. Repositories `flush()`; services `commit()`.

**Q:** Why use `pydantic-settings` instead of hardcoding DB URLs?
**A:** Hardcoded secrets end up in Git history. `BaseSettings` reads from `.env` locally and from real environment variables in production — the code never changes, only the environment does.

**Q:** What does `ConfigDict(from_attributes=True)` do?
**A:** Allows `model_validate(orm_object)` to read attributes from an ORM instance instead of requiring a dict. Required when converting SQLAlchemy model objects into Pydantic response schemas.

**Q:** What happens if you don't import a model file in Alembic's `env.py`?
**A:** Its tables won't appear in `Base.metadata` — autogenerate won't detect them, and their tables won't be created or diff'd against the live schema.

**Q:** What is the thundering herd problem in caching?
**A:** When a popular cache key expires, all concurrent requests miss simultaneously and hit the DB at once. Mitigations: short TTLs on volatile data, background refresh jobs, or request coalescing with a distributed lock (see [[docs/caching|docs/caching.md]]).

*Checkpoint: [[Backend/Checkpoints/CB3 - DB & ORM Proficiency|CB3]]*

*Previous: [[Backend/B2 - Web & API Fundamentals|B2]] | Next: [[Backend/B4 - Authentication & Security|B4]]*
