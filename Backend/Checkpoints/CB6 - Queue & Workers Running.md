---
tags:
  - BE101
  - checkpoint
  - backend
date: 2026-06-27
status: not-started
domain: "6 of 8"
track: backend
verified_by: ""
---

# CB6 — Queue & Workers Running

**Domain:** [[Backend/B6 - Async, Queues & Background Jobs|B6 — Async, Queues & Background Jobs]]

**Verified by:** *(mentor name and date)*

---

## ✅ Pass Criteria

- [ ] A slow operation (e.g., sending an email, resizing an image) runs as a background task and returns `202 Accepted` or `201 Created` immediately while the task runs in the background
- [ ] A Celery (or ARQ) worker successfully processes at least one job type from a queue
- [ ] Architecture diagram drawn showing producer → queue → consumer for a real use case
- [ ] Periodic scheduled job configured and running (e.g., runs every minute, logs a message)
- [ ] Can explain the difference between RabbitMQ and Kafka at a conceptual level
