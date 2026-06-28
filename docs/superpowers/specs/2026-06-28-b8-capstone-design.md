# B8 Capstone Project — Design Spec

**Date:** 2026-06-28
**Track:** Backend (B8 of 8)
**Status:** Approved

---

## Overview

B8 is a free-choice capstone where the intern builds a production-structured two-service backend from scratch, integrating all skills from B1–B7. The note is restructured from 6 sections to 3, plus a deliverables checklist.

**Final section structure:**
- `## 8.1 — Project Specification` — constraints, directory tree, naming conventions, docker-compose reference, dev workflow
- `## 8.2 — Architecture & Event Flow` — system diagram, event flow, `Config.json`, `Dockerfile.eventworker`
- `## 8.3 — Implementation Milestones` — 6 milestones (setup → service A CRUD → service B CRUD → auth → events → tests + README)
- `## ✅ Project Deliverables` — updated observable checklist
- `## 📚 Domain References` — updated references
- `## 🃏 Quick-Reference Flash Cards` — new Q&A pairs
- Drop: §8.4, §8.5, §8.6 (content folded into above)

---

## Key Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| Monorepo structure | 2 self-contained services; `shared/` only when genuinely needed | Matches ARCHITECTURE_REFERENCE.md simplified for interns |
| Message bus | Azure Service Bus Emulator (Docker) | Matches production stack exactly |
| Worker pattern | Each service has its own `Dockerfile.eventworker`; workers in Docker, APIs via `uv run` | Mirrors production docker-compose pattern |
| Auth library | `PyJWT` | `python-jose` is deprecated |
| Tests | Unit only — mock all dependencies | Integration tests out of scope at intern level |
| SQL emulator dependency | `sqledge` container required alongside `servicebus-emulator` | Service Bus emulator uses SQL Edge as internal storage |

---

## §8.1 — Project Specification

### Constraints
- Pick any CRUD project from roadmap.sh/backend/projects
- Must have 2 services that naturally trigger each other
- ≥ 3 CRUD endpoints per service
- JWT auth on write endpoints
- Each service: pub/sub worker via Azure Service Bus
- `docker compose up -d` starts infra + workers only; APIs via `uv run`
- Unit tests only; mock all dependencies

### Directory Tree
```
my-platform/
├── pyproject.toml
├── uv.lock
├── docker-compose.yml
├── .env.example
├── local/
│   └── servicebus/
│       └── Config.json
├── docker/
│   └── postgres-init/
├── services/
│   ├── service-a/
│   │   ├── pyproject.toml
│   │   ├── docker/
│   │   │   └── Dockerfile.eventworker
│   │   ├── {service_a}/
│   │   │   ├── main.py
│   │   │   ├── ioc.py
│   │   │   ├── settings.py
│   │   │   ├── controller/
│   │   │   │   ├── api/v1/
│   │   │   │   ├── events/v1/
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
│   └── service-b/
└── shared/
    └── shared_schemas/
        └── pyproject.toml
```

### Naming Conventions
| Component | Convention | Example |
|-----------|-----------|---------|
| Repository interface | `I{Entity}Repository` | `IOrderRepository` |
| Repository implementation | `SQLAlchemy{Entity}Repository` | `SQLAlchemyOrderRepository` |
| Mapper | `{Entity}Mapper` | `OrderMapper` |
| Domain logic | `{Entity}Logic` | `OrderLogic` |
| Request DTO | `Request{Action}{Resource}DTO` | `RequestCreateOrderDTO` |
| Response DTO | `Response{Resource}DTO` | `ResponseOrderDTO` |

### Dev Workflow
```bash
docker compose up -d
# Terminal 2: cd services/service-a && uv run uvicorn service_a.main:app --reload --port 8000
# Terminal 3: cd services/service-b && uv run uvicorn service_b.main:app --reload --port 8001
```

---

## §8.2 — Architecture & Event Flow

### System Diagram
```
service-a (uv run :8000) ──REST──▶ service-b (uv run :8001)
     │ pub                                │ pub
     ▼                                    ▼
[docker compose]
  postgres | redis | sqledge + servicebus-emulator
  service-a-event-worker ◀── topic subscriptions
  service-b-event-worker ◀── topic subscriptions
```

### Event Flow
```
service-a route → {Context}Service → UnitOfWork.add_event("resource.created")
  → on commit → infrastructure/messaging/ → Service Bus topic
                    → service-b-event-worker → controller/events/v1/
                        → @event_router.subscribe("resource.created")
                            → True = acknowledge | False = retry
```

### Config.json format
```json
{
  "UserConfig": {
    "Namespaces": [{
      "Name": "my-platform",
      "Topics": [
        { "Name": "service-a-events", "Subscriptions": [{ "Name": "service-b-subscription" }] },
        { "Name": "service-b-events", "Subscriptions": [{ "Name": "service-a-subscription" }] }
      ]
    }]
  }
}
```

### Dockerfile.eventworker pattern
Multi-stage build. CMD runs `python -m service_a.worker` — no uvicorn. Exposes port 8088 for healthcheck.

---

## §8.3 — Implementation Milestones

### Milestone 1 — Project Setup & Scaffold
- uv workspace init, `uv sync`
- Initial `uv add` for all packages (fastapi, sqlalchemy, asyncpg, alembic, pydantic-settings, dependency-injector, PyJWT, passlib[bcrypt], azure-servicebus)
- Dev deps: pytest, pytest-asyncio, pytest-cov, ruff, mypy
- `docker compose up -d` all healthy
- Both `/health` endpoints respond

### Milestone 2 — Service A: Domain, DB, CRUD
Build order: entity → interface → logic → repo impl → DTOs → mapper → orchestration service → ioc wiring → routes → alembic migration.

### Milestone 3 — Service B: Domain, DB, CRUD
Same build order. Check for shared entities/DTOs → move to `shared/` if found.

### Milestone 4 — JWT Authentication
`POST /auth/token` in both services. `get_current_user` dependency. All write endpoints 401 without token. Uses `PyJWT`.

### Milestone 5 — Inter-Service Event Flow
Config.json → messaging publisher in service A → event handler in service B → worker.py entrypoint → Dockerfile.eventworker → docker-compose entry → end-to-end test (write in A → observe in B DB). Handlers must be idempotent.

### Milestone 6 — Tests, Quality & README
Unit tests: `{Context}Service` (happy path + validation failures) + `{Entity}Logic` rules. Quality gates: ruff, mypy, pytest --cov. README: setup steps, architecture diagram, API docs links.

---

## Deliverables Checklist

- [ ] GitHub repo with conventional commits and clean history
- [ ] 2 services, each with ≥ 3 CRUD endpoints
- [ ] JWT auth — write endpoints return 401 without token
- [ ] PostgreSQL schema per service with Alembic migrations
- [ ] Event published on write in Service A → consumed and persisted in Service B
- [ ] Event handlers are idempotent
- [ ] Unit tests pass for both services; ruff + mypy clean
- [ ] `docker compose up -d` starts infra + workers with no errors
- [ ] APIs start cleanly with `uv run uvicorn`
- [ ] README: setup, architecture diagram, API docs links

---

## References to Add

- https://roadmap.sh/backend/projects
- https://learn.microsoft.com/azure/service-bus-messaging/
- https://github.com/Azure/azure-sdk-for-python/tree/main/sdk/servicebus/azure-servicebus
- https://python-dependency-injector.ets-labs.org
- https://docs.astral.sh/uv/concepts/workspaces/
- docs/clean-architecture.md (internal reference)
