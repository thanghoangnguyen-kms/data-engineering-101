---
tags:
  - DE101
  - backend-3
  - database
  - orm
date: 2026-06-27
status: not-started
domain: "3 of 8"
track: backend
---

# B3 — Databases & ORM

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll connect a FastAPI application to PostgreSQL using SQLAlchemy ORM (async), manage schema changes with Alembic migrations, add Redis caching, and learn two key data access patterns: Repository Pattern and Unit of Work.

---

## 3.1 — PostgreSQL Fundamentals

*Content coming soon.*

## 3.2 — SQLAlchemy ORM

*Content coming soon.*

## 3.3 — Alembic Migrations

*Content coming soon.*

## 3.4 — Redis & Caching Patterns

*Content coming soon.*

## 3.5 — Connection Pooling & Performance

*Content coming soon.*

## 3.6 — Repository Pattern

> [!IMPORTANT] Why this pattern matters
> The Repository Pattern separates your data access logic from your business logic. You define an abstract interface in the domain layer and a concrete SQLAlchemy implementation in the infrastructure layer — so your business logic never depends on the database directly. This is the foundation of Clean Architecture in [[Backend/B7 - Microservices & Containers|B7]].

*Content coming soon.*

## 3.7 — Unit of Work Pattern

> [!IMPORTANT] Transactions as a unit
> The Unit of Work pattern groups multiple repository operations into a single database transaction. If any step fails, the whole unit rolls back. In SQLAlchemy 2.0 async, this maps directly to the `AsyncSession` lifecycle.

*Content coming soon.*

---

## ✅ Practice Checklist

- [ ] Define SQLAlchemy models for at least 2 related tables
- [ ] Run an Alembic migration to add a column to an existing table
- [ ] Implement a cache-aside pattern with Redis for a GET endpoint
- [ ] Demonstrate connection pooling configuration
- [ ] Implement a repository interface (abstract class) and a concrete SQLAlchemy implementation
- [ ] Wrap two repository calls in a Unit of Work transaction and verify rollback on failure

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.sqlalchemy.org | SQLAlchemy ORM & Core (async) |
| https://alembic.sqlalchemy.org | Alembic migrations |
| https://redis.io/docs | Redis documentation |
| https://www.postgresql.org/docs/ | PostgreSQL reference |

## 🃏 Quick-Reference Flash Cards

*Coming soon.*

*Checkpoint: [[Backend/Checkpoints/CB3 - DB & ORM Proficiency|CB3]]*

*Previous: [[Backend/B2 - Web & API Fundamentals|B2]] | Next: [[Backend/B4 - Authentication & Security|B4]]*
