---
tags:
  - BE101
  - backend-2
  - api
  - fastapi
date: 2026-06-27
status: complete
domain: "2 of 8"
track: backend
---

# B2 — Web & API Design

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll build REST and gRPC APIs with FastAPI. By the end you'll understand Python's async/await model, HTTP in depth, REST design principles, Pydantic validation, FastAPI's dependency injection system, gRPC as an alternative API style, and how to document APIs with OpenAPI.

---

## 2.0 — Python async/await Basics

> [!NOTE]
> FastAPI is built on Python's async system. Understanding `async def` and `await` means you won't be confused when your route handler looks different from regular Python code.

**The Event Loop Mental Model**

Python's async runtime runs in a **single thread** but can juggle many tasks at once by switching between them whenever one is waiting for I/O (network call, DB query, file read). That scheduler is called the **event loop**.

```mermaid
sequenceDiagram
    participant EL as Event Loop
    participant A as Request A
    participant B as Request B
    participant DB as Database

    A->>EL: async handler starts
    EL->>+DB: await db.query() — Request A waiting
    Note over EL: A is parked → switch to B
    B->>EL: async handler starts
    EL->>+DB: await db.query() — Request B waiting
    DB-->>-EL: Result for A ready
    EL-->>A: Resume A → return response
    DB-->>-EL: Result for B ready
    EL-->>B: Resume B → return response
```

No threads, no multiprocessing — just cooperative task switching.

**`async def` vs `def`**

```python
# Synchronous — blocks the event loop while running
def get_user_sync(user_id: int) -> dict:
    result = db.execute("SELECT * FROM users WHERE id = ?", [user_id])
    return result.fetchone()

# Asynchronous — yields control while waiting for I/O
async def get_user(user_id: int) -> dict:
    result = await db.execute("SELECT * FROM users WHERE id = ?", [user_id])
    return await result.fetchone()
```

`await` tells the event loop: *"I'm waiting — go do something else and come back when this is ready."*

**When to use which**

| Use `async def` | Use `def` |
|----------------|-----------|
| Route handlers (FastAPI default) | CPU-heavy computation |
| DB queries (async ORM) | Calling sync third-party libraries |
| Outbound HTTP calls (`httpx`) | Simple utility/helper functions |

> [!WARNING] Don't Block the Event Loop
> ❌ `time.sleep(5)` inside an `async def` — freezes the entire server for 5 seconds
> ❌ Calling a synchronous blocking DB driver from an async route
> ✅ `await asyncio.sleep(5)` — yields control, other requests are served while waiting
> ✅ Use async-compatible libraries: `asyncpg`, `httpx`, `aiofiles`

> [!TIP] Going deeper
> Deep dives into `asyncio.gather()`, task groups, and background workers are in [[Backend/B6 - Async, Queues & Background Jobs|B6]]. For now, just remember: `async def` + `await` = non-blocking I/O.

## 2.1 — HTTP & REST Fundamentals

> [!NOTE]
> REST is an architectural style — a set of conventions for designing APIs over HTTP. Understanding these conventions makes your API predictable and easy for other developers to use.

**HTTP Methods**

| Method | Meaning | Example |
|--------|---------|---------|
| `GET` | Read a resource | `GET /users/42` |
| `POST` | Create a resource | `POST /users` |
| `PUT` | Replace a resource entirely | `PUT /users/42` |
| `PATCH` | Partially update a resource | `PATCH /users/42` |
| `DELETE` | Remove a resource | `DELETE /users/42` |

**Anatomy of an HTTP Request**

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJ...

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

- **Method + path** — what to do and where
- **Headers** — metadata: content type, auth token, accepted formats
- **Body** — data payload (POST/PUT/PATCH only)

**Anatomy of an HTTP Response**

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com"
}
```

**Status Codes Cheatsheet**

| Code | Meaning | When to use |
|------|---------|------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST that creates a resource |
| 204 | No Content | Successful DELETE (empty body) |
| 400 | Bad Request | Malformed request syntax |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not allowed |
| 404 | Not Found | Resource doesn't exist |
| 422 | Unprocessable Entity | Valid syntax but failed validation |
| 500 | Internal Server Error | Bug in your code |

**REST Design Principles**

1. **Resources are nouns** — URLs name things, not actions
2. **Stateless** — each request contains everything needed; no server-side session
3. **Consistent status codes** — use the HTTP standard
4. **JSON bodies** — standard format for request/response data

> [!WARNING] REST Anti-Patterns
> ❌ Verbs in URLs: `POST /createUser`, `GET /getUserById`
> ❌ Wrong status codes: returning `200 OK` for a failed operation
> ❌ Inconsistent naming: `/Users` vs `/user` vs `/user_profile`
> ✅ Nouns only: `POST /users`, `GET /users/42`
> ✅ Correct codes: `201` for create, `404` for not found, `422` for validation failure

## 2.2 — FastAPI Basics

> [!NOTE]
> FastAPI is a Python web framework built on Starlette (HTTP) and Pydantic (validation). It generates OpenAPI docs automatically and is one of the fastest Python frameworks available.

**Install**

```bash
uv add fastapi uvicorn
```

**Minimal App**

```python
# app/main.py
from fastapi import FastAPI

app = FastAPI(title="My API", version="1.0.0")

@app.get("/")
async def root() -> dict:
    return {"message": "Hello, world"}
```

**Run**

```bash
uvicorn app.main:app --reload
```

- `app.main` — module path (`app/main.py`)
- `app` — the `FastAPI()` instance variable name
- `--reload` — restart on file changes (dev only)

**How FastAPI Handles a Request**

Every incoming request passes through the same pipeline before your handler runs:

```mermaid
flowchart TD
    Request([Incoming HTTP Request]) --> Match{Route matched?}
    Match -->|No| R404[Return 404 Not Found]
    Match -->|Yes| Deps[Resolve Depends chains]
    Deps --> Valid{Pydantic validation passes?}
    Valid -->|No| R422[Return 422 Unprocessable Entity]
    Valid -->|Yes| Handler[Run route handler]
    Handler --> Err{Exception raised?}
    Err -->|HTTPException| ErrResp[Return error response]
    Err -->|No| Serialize[Serialize via response_model]
    Serialize --> Response([Return HTTP Response])
```

**Path Parameters**

Variables embedded in the URL path. FastAPI extracts and validates them automatically.

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int) -> dict:
    return {"user_id": user_id}
```

`GET /users/42` → `user_id = 42`. Sending `GET /users/abc` returns `422` automatically.

**Query Parameters**

Parameters after the `?` in the URL. Declared as plain function arguments with defaults.

```python
@app.get("/users")
async def list_users(skip: int = 0, limit: int = 10) -> dict:
    return {"skip": skip, "limit": limit}
```

`GET /users?skip=20&limit=5` → `skip = 20`, `limit = 5`.

**`status_code` on the Route Decorator**

By default all FastAPI routes return `200 OK`. Override with `status_code`:

```python
@app.post("/users", status_code=201)
async def create_user(name: str) -> dict:
    return {"name": name}  # returns 201 Created

@app.delete("/users/{user_id}", status_code=204)
async def delete_user(user_id: int) -> None:
    pass  # returns 204 No Content, empty body
```

> [!IMPORTANT] Match status codes to HTTP semantics
> POST that creates → `201`. DELETE with no body → `204`. Returning `200` for everything is a common REST mistake.

> [!EXAMPLE] Full Working App
> ```python
> from fastapi import FastAPI, HTTPException
> from pydantic import BaseModel, EmailStr
>
> app = FastAPI()
> next_id = 1
>
> class UserCreate(BaseModel):
>     name: str
>     email: EmailStr
>
> class UserResponse(BaseModel):
>     id: int
>     name: str
>     email: str
>
> users: dict[int, UserResponse] = {}
>
> @app.get("/users")
> async def list_users() -> list[UserResponse]:
>     return list(users.values())
>
> @app.get("/users/{user_id}")
> async def get_user(user_id: int) -> UserResponse:
>     if user_id not in users:
>         raise HTTPException(status_code=404, detail="User not found")
>     return users[user_id]
>
> @app.post("/users", status_code=201)
> async def create_user(data: UserCreate) -> UserResponse:
>     global next_id
>     user = UserResponse(id=next_id, name=data.name, email=data.email)
>     users[next_id] = user
>     next_id += 1
>     return user
>
> @app.delete("/users/{user_id}", status_code=204)
> async def delete_user(user_id: int) -> None:
>     if user_id not in users:
>         raise HTTPException(status_code=404, detail="User not found")
>     del users[user_id]
> ```
> *Note: `users` is an in-memory dict — for demo only. Real data persistence is covered in [[Backend/B3 - Databases & ORM|B3]].*

---

## 2.3 — Project Structure & APIRouter

> [!NOTE]
> Every real FastAPI project splits routes across multiple files. FastAPI's `APIRouter` is the tool for this — it works like a mini-app that you mount onto the main `FastAPI()` instance.

**Why Not Everything in `main.py`?**

A project with users, products, and orders in one file becomes unmanageable fast. The convention is one router file per resource.

**Recommended Project Layout**

```
app/
├── main.py           ← creates FastAPI() instance, registers routers
├── routers/
│   ├── users.py      ← user endpoints
│   └── items.py      ← item endpoints
├── models/           ← Pydantic schemas (B2) and ORM models (B3)
└── dependencies.py   ← shared Depends() functions
```

**Defining a Router**

```python
# app/routers/users.py
from fastapi import APIRouter, HTTPException

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/")
async def list_users() -> list:
    return []

@router.get("/{user_id}")
async def get_user(user_id: int) -> dict:
    raise HTTPException(status_code=404, detail="User not found")

@router.post("/", status_code=201)
async def create_user() -> dict:
    return {}
```

- `prefix="/users"` — all routes in this file start with `/users`
- `tags=["users"]` — groups routes under "users" in `/docs`

**Registering Routers in `main.py`**

```python
# app/main.py
from fastapi import FastAPI
from app.routers import users, items

app = FastAPI()

app.include_router(users.router)
app.include_router(items.router)
```

**CORS Middleware**

When a browser calls your API from a different origin (e.g., your frontend at `localhost:3000` calling your API at `localhost:8000`), the browser blocks the request unless your API explicitly permits it. This is CORS — Cross-Origin Resource Sharing.

```python
# app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # your frontend URL
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> [!WARNING] CORS in Production
> ❌ `allow_origins=["*"]` in production — allows any website to call your API
> ✅ List specific origins explicitly: `["https://yourapp.com"]`
> Dev environments can use `["*"]` for convenience, but restrict before deploying.

> [!TIP] CORS errors appear in the browser, not in FastAPI logs
> If your browser console shows `Access-Control-Allow-Origin` errors but your API logs look fine — it's a CORS issue. Add `CORSMiddleware` as shown above.

## 2.4 — Pydantic Schemas & Validation

> [!NOTE]
> Pydantic models define the shape of your request and response data. FastAPI uses them to automatically validate incoming data, serialize outgoing data, and generate OpenAPI docs.

**Why Pydantic Over Plain `dict`**

```python
# ❌ dict — no validation, no type safety, easy to pass garbage
def create_user(data: dict):
    name = data["name"]       # KeyError if missing
    age = data.get("age", 0)  # silently defaults, no range check

# ✅ Pydantic — validated, typed, self-documenting
class UserCreate(BaseModel):
    name: str
    age: int
```

**Defining a `BaseModel`**

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(gt=0, lt=150)
    role: str = Field(default="viewer")
```

> [!TIP] Use `EmailStr` instead of manual regex
> `EmailStr` (from `pydantic`) validates the format and lowercases the value automatically — no manual regex needed.

FastAPI returns `422` automatically if any constraint is violated.

**Common `Field()` Options**

| Constraint | Type | Example |
|-----------|------|---------|
| `min_length` / `max_length` | `str` | `Field(min_length=1)` |
| `gt` / `ge` / `lt` / `le` | `int`, `float` | `Field(gt=0)` — greater than 0 |
| `pattern` | `str` | `Field(pattern=r"^\d{4}$")` |
| `default` | any | `Field(default="viewer")` |
| `description` | any | Shown in OpenAPI docs |

**`@field_validator` — Custom Validation Logic**

When `Field()` constraints aren't enough (normalize a value, enforce business rules):

```python
from pydantic import BaseModel, field_validator

class UserCreate(BaseModel):
    name: str
    email: str

    @field_validator("email")
    @classmethod
    def email_must_be_lowercase(cls, value: str) -> str:
        if value != value.lower():
            raise ValueError("Email must be lowercase")
        return value.lower()

    @field_validator("name")
    @classmethod
    def name_no_numbers(cls, value: str) -> str:
        if any(c.isdigit() for c in value):
            raise ValueError("Name cannot contain numbers")
        return value.strip()
```

`@field_validator` runs after type coercion. Raise `ValueError` to fail validation — Pydantic wraps it into a `422` response automatically.

**Request Model vs Response Model**

Always use **separate models** for input and output:

```python
class UserCreate(BaseModel):    # input — what the client sends
    name: str
    email: str
    password: str               # needed for creation

class UserResponse(BaseModel):  # output — what the client receives
    id: int
    name: str
    email: str
    # password intentionally absent

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(data: UserCreate) -> UserResponse:
    ...
```

> [!WARNING] Never Expose Internal Fields in Response Models
> ❌ Returning the DB model object directly — may include `password_hash`, `deleted_at`, internal flags
> ✅ Always define a `UserResponse` listing only what clients should see
> FastAPI's `response_model=` filters the output automatically

## 2.5 — Error Handling & Status Codes

> [!NOTE]
> Consistent, structured error responses make your API predictable. Clients shouldn't receive raw Python exceptions or HTML error pages.

**`HTTPException` — Raising Errors from Routes**

```python
from fastapi import HTTPException

@app.get("/users/{user_id}")
async def get_user(user_id: int) -> dict:
    user = await fetch_user(user_id)
    if user is None:
        raise HTTPException(status_code=404, detail="User not found")
    return user
```

FastAPI returns:
```json
{"detail": "User not found"}
```

`detail` can be any JSON-serializable value:
```python
raise HTTPException(
    status_code=422,
    detail={"field": "email", "message": "Already in use"},
)
```

**Global Exception Handlers**

For catching domain exceptions and mapping them to HTTP responses:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

class UserNotFoundError(Exception):
    def __init__(self, user_id: int):
        self.user_id = user_id

@app.exception_handler(UserNotFoundError)
async def user_not_found_handler(request: Request, exc: UserNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"detail": f"User {exc.user_id} not found"},
    )
```

Now you can `raise UserNotFoundError(42)` anywhere in your service layer — it maps to `404` automatically without cluttering routes with `try/except` blocks.

> [!WARNING] Never Expose Stack Traces in Production
> ❌ Returning Python exception messages directly to the client — reveals your code structure
> ✅ Use `@app.exception_handler(Exception)` to catch all unhandled errors
> ✅ Log the full error internally; return `{"detail": "Internal server error"}` to the client

## 2.6 — Dependency Injection with `Depends()`

> [!NOTE]
> FastAPI's `Depends()` solves a common problem: how do route handlers get access to shared resources (DB sessions, config, current user) without relying on global state?

**What Problem DI Solves**

```python
# ❌ Global DB session — shared state, not testable, not thread-safe
db = create_engine("sqlite:///app.db")

@app.get("/users")
async def list_users():
    return db.execute("SELECT * FROM users")

# ✅ Session created per request, injected in, closed after response
@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    return await db.execute(...)
```

**Simple `Depends()` Example**

```python
from fastapi import Depends

def get_settings() -> dict:
    return {"env": "production", "debug": False}

@app.get("/info")
async def app_info(settings: dict = Depends(get_settings)) -> dict:
    return settings
```

FastAPI calls `get_settings()` automatically and passes the result as `settings`.

**The `yield` Pattern — Resource Cleanup**

For resources that need to be closed after the request (DB sessions, file handles), use `yield` instead of `return`:

```python
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker

engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with SessionLocal() as session:
        yield session  # route handler runs here
        # session is automatically closed when the request ends
```

```python
@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)) -> list:
    result = await db.execute(select(User))  # User model and select() covered in B3
    return result.scalars().all()
```

> [!IMPORTANT] Always Use `yield` for DB Sessions
> Without `yield`, the session is never closed — connections leak. With `yield`, FastAPI guarantees cleanup even if the route raises an exception.

**Chaining Dependencies**

One `Depends` can depend on another:

```python
def get_db() -> AsyncGenerator[AsyncSession, None]:
    ...  # yields DB session

def get_current_user(db: AsyncSession = Depends(get_db)) -> User:
    ...  # uses DB session to look up current user

@app.get("/profile")
async def profile(user: User = Depends(get_current_user)) -> dict:
    return {"name": user.name}  # FastAPI resolves the full chain automatically
```

## 2.7 — gRPC as an API Style

> [!NOTE]
> gRPC is an alternative to REST for API communication. Where REST uses JSON over HTTP/1.1, gRPC uses **Protocol Buffers** (binary format) over **HTTP/2**. It's faster and stricter, but requires more setup.

**REST vs gRPC**

| | REST | gRPC |
|--|------|------|
| **Format** | JSON (text) | Protocol Buffers (binary) |
| **Protocol** | HTTP/1.1 | HTTP/2 |
| **Contract** | Optional (OpenAPI) | Required (`.proto` file) |
| **Browser support** | Native | Needs proxy (`grpc-web`) |
| **Streaming** | Limited | First-class support |
| **Use case** | Public APIs, frontends | Internal service-to-service |

**How gRPC Works**

You define your API in a `.proto` file — this is the enforced contract both sides must follow:

```protobuf
// user_service.proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
}

message GetUserRequest  { int32 user_id = 1; }
message CreateUserRequest { string name = 1; string email = 2; }
message UserResponse    { int32 id = 1; string name = 2; string email = 3; }
```

Run `protoc` to generate Python client and server stubs from this file. Both sides use the generated code — no manual JSON parsing.

**When gRPC Wins**

- **Internal service-to-service** calls (not exposed to browsers)
- **High-throughput, low-latency** requirements (binary is smaller and faster than JSON)
- **Bi-directional streaming** (real-time data feeds between services)
- Teams that want **strict contracts** enforced at compile time

> [!NOTE] Scope for this track
> REST is the primary focus. gRPC is here for awareness — you'll encounter it in production microservice systems. Service communication patterns are explored further in [[Backend/B7 - Microservices & Containers|B7]].

## 2.8 — Protecting Routes with Dependencies

> [!NOTE] Scope
> This section covers using `Depends()` to guard routes (e.g., require a logged-in user). JWT internals and OAuth 2.0 flows are covered in [[Backend/B4 - Authentication & Security|B4]].

**The Auth Guard Pattern**

```python
from fastapi import Depends, HTTPException, Header

def require_auth(authorization: str = Header(default=None)) -> str:
    """Checks for a Bearer token in the Authorization header."""
    if authorization is None or not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing or invalid token")
    token = authorization.removeprefix("Bearer ")
    # In B4 you'll validate the JWT here; for now, just check it exists
    return token
```

Apply to a single route:

```python
@app.get("/me")
async def get_profile(token: str = Depends(require_auth)) -> dict:
    return {"token_received": token}
```

Apply to an entire router (all routes protected at once):

```python
router = APIRouter(
    prefix="/admin",
    tags=["admin"],
    dependencies=[Depends(require_auth)],  # all routes in this router are protected
)
```

**401 vs 403**

| Code | Meaning | When |
|------|---------|------|
| `401 Unauthorized` | No valid identity | Missing token, expired token, bad signature |
| `403 Forbidden` | Valid identity, wrong permissions | Logged in but doesn't have required role |

```python
def require_admin(token: str = Depends(require_auth)) -> None:
    user = decode_token(token)   # covered in B4
    if user.role != "admin":
        raise HTTPException(status_code=403, detail="Admin access required")
```

```mermaid
sequenceDiagram
    actor Client
    participant FastAPI
    participant Guard as require_auth()
    participant Handler as Route Handler

    Client->>FastAPI: GET /me\nAuthorization: Bearer token
    FastAPI->>Guard: Depends(require_auth)
    alt No token or malformed header
        Guard-->>FastAPI: raise HTTPException(401)
        FastAPI-->>Client: 401 Unauthorized
    else Valid token, wrong role
        Guard-->>FastAPI: raise HTTPException(403)
        FastAPI-->>Client: 403 Forbidden
    else Valid token, correct role
        Guard-->>FastAPI: return token
        FastAPI->>Handler: execute handler
        Handler-->>FastAPI: return data
        FastAPI-->>Client: 200 OK
    end
```

> [!TIP] Order matters
> Check authentication before authorization. A common mistake is returning `403` when the token is simply missing — the correct code is `401`.

## 2.9 — OpenAPI & API Documentation

> [!NOTE]
> FastAPI generates interactive API documentation automatically from your route decorators, Pydantic models, and type hints. No extra configuration needed.

**What FastAPI Generates**

| URL | UI | Use For |
|-----|----|---------|
| `/docs` | Swagger UI | Interactive — test endpoints in the browser |
| `/redoc` | ReDoc | Read-only — clean reference documentation |
| `/openapi.json` | Raw JSON | Machine-readable spec for code generation |

**Enriching the Documentation**

```python
@app.get(
    "/users/{user_id}",
    summary="Get a user by ID",
    description="Returns a single user. Raises 404 if the user does not exist.",
    response_model=UserResponse,
    tags=["users"],
)
async def get_user(user_id: int) -> UserResponse:
    ...
```

- `summary` — short title shown in the docs list
- `description` — longer description in the expanded view
- `response_model` — documents and filters the response shape
- `tags` — groups routes into collapsible sections

**Adding Example Responses**

```python
@app.get(
    "/users/{user_id}",
    responses={
        200: {"description": "User found"},
        404: {
            "description": "User not found",
            "content": {"application/json": {"example": {"detail": "User not found"}}},
        },
    },
)
async def get_user(user_id: int) -> UserResponse:
    ...
```

**App-Level Metadata**

```python
app = FastAPI(
    title="My Backend API",
    description="Backend API for the intern project.",
    version="0.1.0",
    contact={"name": "Backend Team", "email": "backend@example.com"},
)
```

> [!TIP] Use Tags to Organize Large APIs
> As routes grow, `tags=["users"]` in each router keeps `/docs` navigable. Routes without tags pile into a generic "default" group and become hard to find.

---

## 🎯 What You Learned

You can now:

- **Build REST APIs with FastAPI** — route handlers, path/query parameters, request bodies via Pydantic, status codes, and `HTTPException` for error responses
- **Validate request data automatically** — Pydantic `BaseModel` schemas with `EmailStr`, `Field` constraints, and `field_validator` catch bad input before your logic runs
- **Organise a real FastAPI project** — `APIRouter` splits routes by resource, `Depends()` injects shared logic (DB sessions, auth checks), and the yield pattern handles teardown cleanly
- **Design a clean API contract** — explicit `response_model` schemas define exactly what gets serialised, preventing internal fields from leaking to callers
- **Work with OpenAPI docs** — FastAPI generates `/docs` and `/redoc` automatically; every schema and example you add shows up there

---

## ✅ Practice Checklist

- [ ] Build a FastAPI app with at least 3 endpoints (GET, POST, DELETE)
- [ ] Write both `async def` and `def` handlers and explain when each is appropriate
- [ ] Define a Pydantic model with validation constraints
- [ ] Return structured error responses with correct HTTP status codes
- [ ] Use `Depends()` to inject a dependency (e.g., a shared config or DB session) into a route
- [ ] Add a route guard that rejects unauthenticated requests using `Depends()`
- [ ] View and navigate the auto-generated `/docs` OpenAPI UI

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://fastapi.tiangolo.com | FastAPI documentation |
| https://docs.pydantic.dev | Pydantic v2 documentation |
| https://developer.mozilla.org/en-US/docs/Web/HTTP | HTTP fundamentals |
| https://docs.python.org/3/library/asyncio.html | asyncio reference |
| https://grpc.io/docs/languages/python/ | gRPC Python |

## 🃏 Quick-Reference Flash Cards

**Q:** What is the event loop in Python async?
**A:** A single-threaded scheduler that switches between tasks whenever one is waiting for I/O. `await` hands control back to it while waiting.

**Q:** When should you use `async def` vs `def` in FastAPI?
**A:** `async def` for I/O-bound work (DB queries, HTTP calls). `def` for CPU-bound work or when using sync-only libraries.

**Q:** What HTTP status code should a successful POST that creates a resource return?
**A:** `201 Created` — set with `@app.post("/users", status_code=201)`.

**Q:** What is `APIRouter` and why use it?
**A:** A mini-app that groups related routes. Keeps routes in separate files instead of all in `main.py`. Mounted with `app.include_router()`.

**Q:** What does the `yield` pattern in a `Depends()` do?
**A:** Code before `yield` runs before the route (setup). Code after `yield` runs after (cleanup). Guarantees DB sessions are closed even if the route raises an exception.

**Q:** What is CORS and how do you enable it in FastAPI?
**A:** Cross-Origin Resource Sharing — browsers block requests from a different origin by default. Add `CORSMiddleware` with allowed origins to permit cross-origin calls.

**Q:** What's the difference between a request model and a response model?
**A:** Request model defines what the client sends (may include `password`). Response model defines what the client receives (excludes sensitive fields). Keep them separate.

**Q:** When should you use `@field_validator` instead of `Field()`?
**A:** When validation needs custom logic — e.g., normalizing a value, cross-field checks, or enforcing business rules beyond simple constraints.

**Q:** What's the difference between 401 and 403?
**A:** `401` — no valid identity (missing/invalid token). `403` — valid identity but insufficient permissions. Always check authentication before authorization.

**Q:** How does FastAPI generate `/docs`?
**A:** Automatically from route decorators, `response_model`, Pydantic schemas, and `summary`/`description`/`tags` parameters.

*Checkpoint: [[Backend/Checkpoints/CB2 - API Built & Documented|CB2]]*

*Previous: [[Backend/B1 - Foundations & Dev Setup|B1]] | Next: [[Backend/B3 - Databases & ORM|B3]]*
