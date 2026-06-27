---
tags:
  - DE101
  - checkpoint
  - backend
date: 2026-06-27
status: not-started
domain: "2 of 8"
track: backend
verified_by: ""
---

# CB2 — API Built & Documented

**Domain:** [[Backend/B2 - Web & API Fundamentals|B2 — Web & API Fundamentals]]

**Verified by:** *(mentor name and date)*

---

## ✅ Pass Criteria

- [ ] FastAPI app running locally with at least 3 endpoints (GET, POST, DELETE)
- [ ] All request bodies validated with Pydantic models (including at least one `Field` constraint)
- [ ] All endpoints return structured error responses with correct HTTP status codes (404, 422, 500)
- [ ] At least one route protected with a `Depends()` guard that rejects unauthenticated requests
- [ ] OpenAPI docs available at `/docs` and accurately reflect the API
- [ ] Can explain the difference between `async def` and `def` route handlers and when to use each
- [ ] Can explain what happens step-by-step when a POST request with invalid data is sent
