---
tags:
  - DE101
  - checkpoint
  - backend
date: 2026-06-27
status: not-started
domain: "8 of 8"
track: backend
verified_by: ""
---

# CB8 — Capstone Complete

**Domain:** [[Backend/B8 - Capstone Project|B8 — Capstone Project]]

**Verified by:** *(mentor name and date)*

---

## ✅ Pass Criteria

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
