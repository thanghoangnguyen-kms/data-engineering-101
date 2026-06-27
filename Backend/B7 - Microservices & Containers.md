---
tags:
  - DE101
  - backend-7
  - docker
  - microservices
date: 2026-06-27
status: not-started
domain: "7 of 8"
track: backend
---

# B7 — Microservices & Containers

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll containerise a FastAPI application with Docker, orchestrate multi-service setups with Docker Compose, learn microservice architecture patterns, use gRPC for inter-service communication, and see how all B1–B7 skills come together in a production Clean Architecture codebase.

---

## 7.1 — Docker Fundamentals

*Content coming soon.*

## 7.2 — Docker Compose & Multi-Service Setups

*Content coming soon.*

## 7.3 — Microservice Architecture Patterns

*Content coming soon.*

## 7.4 — gRPC & Inter-Service Communication

> [!NOTE] Scope
> B2 §2.6 introduced gRPC as an API style. Here you'll apply it specifically for service-to-service communication — defining `.proto` contracts between two services, handling streaming, and comparing gRPC vs REST for internal calls.

*Content coming soon.*

## 7.5 — System Design Basics

*Content coming soon.*

## 7.6 — Production Codebase Architecture

> [!IMPORTANT] How real backends are structured
> This section maps everything you've learned to how a production Python backend is actually organised. You'll see Clean Architecture's 4-layer model in action, full `dependency-injector` IoC container wiring, and how a `uv` workspace monorepo ties multiple services and shared packages together.

**The 4-layer model:**
- **Controller** → FastAPI routes, middleware, event handlers (HTTP concerns only)
- **Orchestration** → Use case coordinators, DTO ↔ Entity mapping, transaction coordination
- **Domain** → Pure business logic, entities, repository interfaces (no external dependencies)
- **Infrastructure** → SQLAlchemy repositories, external clients, messaging, configuration

*Content coming soon.*

---

## ✅ Practice Checklist

- [ ] Write a `Dockerfile` for a FastAPI app and run it with `docker run`
- [ ] Write a `docker-compose.yml` that spins up the API, PostgreSQL, and Redis together
- [ ] Define a `.proto` file and use gRPC to call one service from another
- [ ] Draw a microservice architecture diagram for a simple e-commerce system
- [ ] Identify which layer of the Clean Architecture each component in a sample repo belongs to

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.docker.com | Docker documentation |
| https://docs.docker.com/compose/ | Docker Compose |
| https://grpc.io/docs/languages/python/ | gRPC Python |
| https://python-dependency-injector.ets-labs.org | dependency-injector (IoC container) |
| https://docs.astral.sh/uv/concepts/workspaces/ | uv workspaces — monorepo setup |

## 🃏 Quick-Reference Flash Cards

*Coming soon.*

*Checkpoint: [[Backend/Checkpoints/CB7 - Service Containerised|CB7]]*

*Previous: [[Backend/B6 - Async, Queues & Background Jobs|B6]] | Next: [[Backend/B8 - Capstone Project|B8]]*
