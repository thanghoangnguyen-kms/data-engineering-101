---
tags:
  - BE101
  - checkpoint
  - backend
date: 2026-06-27
status: not-started
domain: "3 of 8"
track: backend
verified_by: ""
---

# CB3 — DB & ORM Proficiency

**Domain:** [[Backend/B3 - Databases & ORM|B3 — Databases & ORM]]

**Verified by:** *(mentor name and date)*

---

## ✅ Pass Criteria

- [ ] SQLAlchemy models defined for at least 2 tables with a foreign key relationship
- [ ] Alembic migration successfully adds a new column to an existing table
- [ ] CRUD endpoints (create, read, update, delete) working against a real PostgreSQL database using async SQLAlchemy
- [ ] Redis cache-aside pattern implemented: first request hits DB, second request returns from cache
- [ ] Repository interface (abstract class) defined and a concrete SQLAlchemy implementation provided
- [ ] Unit of Work demonstrated: two writes wrapped in one transaction, second fails, first rolls back
- [ ] Can explain when to use the ORM vs raw SQL
