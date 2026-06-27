---
tags:
  - DE101
  - backend-6
  - queues
  - async
date: 2026-06-27
status: not-started
domain: "6 of 8"
track: backend
---

# B6 — Async, Queues & Background Jobs

**Back to:** [[Backend/00 - Backend Track Roadmap|Backend Track Roadmap]]

> [!NOTE] Domain Overview
> You'll learn async Python, implement background task workers, and understand message queue concepts. RabbitMQ, NATS, and Kafka are covered conceptually — you'll do hands-on work with Python task queues.

---

## 6.1 — Async Python Deep Dive

> [!NOTE] Scope
> This section goes beyond the brief intro in [[Backend/B2 - Web & API Fundamentals|B2 §2.0]]. Here you'll learn how the event loop works, coroutine chaining, `asyncio.gather`, `asyncio.TaskGroup`, and the difference between concurrency and parallelism.

*Content coming soon.*

## 6.2 — Background Tasks in FastAPI

*Content coming soon.*

## 6.3 — Task Queues & Workers

*Content coming soon.*

## 6.4 — Message Queues: RabbitMQ, NATS, Kafka (Conceptual)

*Content coming soon.*

## 6.5 — Scheduled Jobs

*Content coming soon.*

---

## ✅ Practice Checklist

- [ ] Refactor a slow endpoint to run its work as a background task
- [ ] Implement a Celery (or ARQ) worker that processes a job queue
- [ ] Draw a message queue architecture diagram for a notification system
- [ ] Schedule a periodic job that runs every minute

## 📚 Domain References

| Resource | Use For |
|----------|---------|
| https://docs.python.org/3/library/asyncio.html | asyncio reference |
| https://docs.celeryq.dev | Celery task queue |
| https://arq-docs.helpmanual.io | ARQ async job queue |
| https://www.rabbitmq.com/documentation.html | RabbitMQ docs |
| https://nats.io/documentation/ | NATS docs |

## 🃏 Quick-Reference Flash Cards

*Coming soon.*

*Checkpoint: [[Backend/Checkpoints/CB6 - Queue & Workers Running|CB6]]*

*Previous: [[Backend/B5 - Testing & Code Quality|B5]] | Next: [[Backend/B7 - Microservices & Containers|B7]]*
