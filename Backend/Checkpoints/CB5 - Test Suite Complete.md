---
tags:
  - DE101
  - checkpoint
  - backend
date: 2026-06-27
status: not-started
domain: "5 of 8"
track: backend
verified_by: ""
---

# CB5 — Test Suite Complete

**Domain:** [[Backend/B5 - Testing & Code Quality|B5 — Testing & Code Quality]]

**Verified by:** *(mentor name and date)*

---

## ✅ Pass Criteria

- [ ] Unit tests written for at least 3 service-layer functions with all external dependencies mocked
- [ ] Integration test runs against a real test database (not the production database)
- [ ] End-to-end test uses FastAPI `TestClient` to test at least one full request/response cycle
- [ ] `pytest --cov` report shows ≥80% line coverage
- [ ] `mypy` runs with zero errors on the entire project
- [ ] Can explain the difference between a unit test and an integration test
