---
tags:
  - BE101
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
- [ ] Can explain the difference between unit tests (no I/O, use fakes) and integration tests (real DB/network) and describe how Docker Compose enables integration testing (covered hands-on in B7)
- [ ] End-to-end test uses FastAPI `TestClient` to test at least one full request/response cycle
- [ ] `pytest --cov` report shows ≥80% line coverage
- [ ] `mypy` runs with zero errors on the entire project
