---
id: event-driven
title: Event-Driven Architecture
description: Comprehensive guide to the platform's event-driven architecture with Azure Service Bus, EventRouter pattern, and resilience patterns
sidebar_label: Event-Driven Architecture
---

# EVENT-DRIVEN ARCHITECTURE

## Table of Contents

- [Overview](#overview)
- [Architecture Principles](#architecture-principles)
- [Azure Service Bus Integration](#azure-service-bus-integration)
  - [Topic Hierarchy](#topic-hierarchy)
  - [Event Schema](#event-schema)
  - [Delivery Guarantees](#delivery-guarantees)
- [Tiered Worker Architecture](#tiered-worker-architecture)
  - [Tier 1: Dedicated Workers](#tier-1-dedicated-workers)
  - [Tier 2: Background Workers](#tier-2-background-workers)
  - [Worker Configuration](#worker-configuration)
- [EventRouter Pattern](#eventrouter-pattern)
  - [Decorator-Based Subscription](#decorator-based-subscription)
  - [Handler Signature Enforcement](#handler-signature-enforcement)
  - [Dependency Injection](#dependency-injection)
- [Event Handler Development](#event-handler-development)
  - [Handler Implementation](#handler-implementation)
  - [Handler Registration](#handler-registration)
  - [Message Lifecycle](#message-lifecycle)
- [Transactional Outbox Pattern](#transactional-outbox-pattern)
- [Resilience Patterns](#resilience-patterns)
  - [Idempotency Service](#idempotency-service)
  - [Circuit Breaker](#circuit-breaker)
  - [Dead-Letter Handling](#dead-letter-handling)
- [Correlation Context](#correlation-context)
- [Testing Event Handlers](#testing-event-handlers)
- [Monitoring & Observability](#monitoring--observability)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

The platform implements a **native Azure Service Bus event-driven architecture** inspired by FastStream but built without sidecars (vs Dapr) for maximum performance and control. The architecture supports:

- **Asynchronous communication** between microservices
- **Eventual consistency** with transactional outbox pattern
- **Horizontal scaling** with KEDA-based auto-scaling
- **Resilient processing** with idempotency, circuit breakers, and dead-letter handling
- **Developer experience** with FastAPI-style decorator patterns

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Native Azure SDK** | Direct AMQP over TCP (no HTTP sidecar overhead) |
| **FastStream-Inspired API** | Familiar decorator pattern for Python developers |
| **Tiered Workers** | Balance complexity vs cost (dedicated vs background) |
| **Protocol-Based Abstractions** | Polymorphism without `isinstance()` checks |
| **Strict Return Contracts** | `True`/`False` semantics eliminate ambiguity |

---

## Architecture Principles

### Clean Architecture Integration

Event handlers follow Clean Architecture separation:

```
services/{service}/{service}_service/
├── controller/
│   ├── api/v1/              # REST endpoint handlers
│   └── events/              # Event handlers
│       ├── __init__.py      # register_event_handlers()
│       └── v1/
│           └── {context}_events.py  # @event_router.subscribe()
├── orchestration/           # Business logic (shared by API & events)
├── domain/                  # Domain entities & logic
└── infrastructure/          # External concerns
```

**Key Principles**:
- **Controllers invoke orchestration** - Event handlers call same orchestration services as API controllers
- **No business logic in handlers** - Handlers only validate and delegate
- **Explicit return values** - `True`/`False` for message lifecycle control
- **Dependency injection** - `Depends()` pattern matches FastAPI

### Event-Driven Communication Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      Transactional Outbox                    │
│                                                              │
│  1. API Request                                              │
│     └─> Orchestration Service                               │
│         └─> UnitOfWork Transaction                          │
│             ├─> Persist Entity (DB write)                   │
│             └─> Add Domain Event (in-memory)                │
│                                                              │
│  2. Transaction Commit                                       │
│     └─> EventPublisher processes outbox                     │
│         └─> Publish to Azure Service Bus                    │
│                                                              │
│  3. Async Processing                                         │
│     └─> EventProcessor consumes message                     │
│         └─> EventRouter invokes handler                     │
│             └─> Orchestration Service (with DI)             │
│                 └─> Complete/Dead-letter message            │
└─────────────────────────────────────────────────────────────┘
```

---

## Azure Service Bus Integration

### Topic Hierarchy

**Hierarchical Organization**: Events organized by domain and aggregate

```
clinical.documents.events     # Document lifecycle
├── document.created
├── document.revisioned
└── document.deleted

clinical.comments.events      # Collaboration
├── comment.added
├── thread.updated
└── mention.created

identity.auth.events          # Authentication
├── user.created
├── role.assigned
└── permission.changed

system.audit.events           # Compliance
└── audit.recorded
```

**Naming Convention**: `{domain}.{aggregate}.{category}`

### Event Schema

**CloudEvents 1.0 Specification** with Pydantic validation:

```python
from pydantic import BaseModel
from datetime import datetime
from uuid import UUID

class DocumentCreatedEvent(BaseModel):
    """Event emitted when a new document is created."""

    # CloudEvents metadata
    spec_version: str = "1.0"
    type: str = "document.created"
    source: str  # Service that emitted event
    id: str      # Unique event ID
    time: datetime

    # Domain data
    data: DocumentCreatedEventData
    correlation_id: UUID

class DocumentCreatedEventData(BaseModel):
    """Domain data for document.created event."""
    document_id: UUID
    title: str
    reporting_event_id: UUID
    deliverable_set_id: UUID
    created_by: UUID
    company_id: UUID
```

**Versioning Strategy**:
- Schema version in event type: `document.created.v1`, `document.created.v2`
- Backward-compatible changes within major version
- Separate handlers for different versions if needed

### Delivery Guarantees

| Feature | Configuration | Purpose |
|---------|---------------|---------|
| **At-Least-Once** | Default Service Bus behavior | Messages processed minimum once |
| **Session Ordering** | `session_enabled: true` | Strict FIFO within session |
| **Idempotency** | Redis-backed with 24h TTL | Prevent duplicate processing |
| **Peek-Lock** | 5-minute lock duration | Automatic retry on failure |
| **Dead-Letter** | Auto-move after max retries | Manual intervention required |

---

## Tiered Worker Architecture

### Strategy Overview

**Two-Tier Design** balances operational complexity with resource efficiency:

```
┌─────────────────────────────────────────────────────────────┐
│  TIER 1: DEDICATED WORKERS (Session-Critical & High-Volume)  │
├─────────────────────────────────────────────────────────────┤
│  • One worker per subscription                              │
│  • Session support enabled                                  │
│  • KEDA scaling: 1-10 replicas                             │
│  • Use cases: Document lifecycle, critical notifications    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  TIER 2: BACKGROUND WORKERS (Audit & Eventual Consistency)  │
├─────────────────────────────────────────────────────────────┤
│  • Up to 5 subscriptions per worker                         │
│  • Shared ServiceBusClient connection                       │
│  • KEDA scaling: 1-5 replicas (OR logic)                   │
│  • Use cases: Audit, cleanup, analytics                     │
└─────────────────────────────────────────────────────────────┘
```

### Tier 1: Dedicated Workers

**Characteristics**:
- **One subscription per worker process** - Exclusive resource allocation
- **Session support** - Ordered message processing within session
- **High concurrency** - Optimized for throughput (10+ concurrent messages)
- **KEDA auto-scaling** - Based on queue depth (1-10 replicas)

**Use Cases**:
- Document lifecycle events (strict ordering required)
- Critical notifications (low latency required)
- Payment processing (transactional integrity)

**Configuration Example**:

```yaml
# config/event-worker-config.yaml
worker:
  worker_tier: dedicated
  subscriptions:
    - name: documents_lifecycle
      topic: clinical.documents.events
      subscription: ordered-subscription
      session_enabled: true
      max_concurrent: 10
      prefetch_count: 5
```

### Tier 2: Background Workers

**Characteristics**:
- **Up to 5 subscriptions per worker** - Resource sharing for efficiency
- **Shared ServiceBusClient** - Single connection reduces overhead
- **Lower concurrency** - 3-5 messages per subscription
- **Error isolation** - Failed subscriber doesn't crash worker

**Use Cases**:
- Audit logging (eventual consistency acceptable)
- Cleanup jobs (non-critical timing)
- Analytics updates (batch processing)

**Configuration Example**:

```yaml
worker:
  worker_tier: background
  subscriptions:
    - name: documents_audit
      topic: system.audit.events
      subscription: audit-sub
      session_enabled: false
      max_concurrent: 3
      prefetch_count: 5

    - name: comments_audit
      topic: clinical.comments.events
      subscription: comments-audit-sub
      session_enabled: false
      max_concurrent: 3
      prefetch_count: 5
```

### Worker Configuration

**Architectural Constraints**:

| Constraint | Rationale |
|------------|-----------|
| Max 5 subscriptions per background worker | Prevent resource exhaustion |
| No session-enabled in background workers | Ordering incompatible with pooling |
| No mixing high/low volume topics | Prevent starvation |
| Dedicated workers for critical paths | Guarantee SLA compliance |

**KEDA Scaling Configuration**:

```yaml
# k8s/{service}/scaledobject-eventworker.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: document-worker-dedicated
spec:
  scaleTargetRef:
    name: document-worker-dedicated
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: clinical.documents.events
        subscriptionName: ordered-subscription
        messageCount: "10"  # Scale up when > 10 messages
```

---

## EventRouter Pattern

### Decorator-Based Subscription

**FastStream-Inspired API** for type-safe event registration:

```python
from platform_event.routing import event_router
from azure.servicebus import ServiceBusReceivedMessage
from fastapi import Depends

@event_router.subscribe(event_type=EventType.DOCUMENT_CREATED.value)
async def handle_document_created(
    event_data: dict,                          # Required parameter
    message: ServiceBusReceivedMessage,         # Required parameter
    service: DocumentService = Depends(get_document_service)  # DI
) -> bool:
    """
    Process document.created events.

    Args:
        event_data: Parsed CloudEvents payload
        message: Azure Service Bus message (for correlation_id, dead_letter)
        service: Injected orchestration service

    Returns:
        True: Successfully processed → auto-complete
        False: Handler failed → auto-dead-letter
    """
    try:
        # Validate event schema
        payload = DocumentCreatedEvent.model_validate(event_data.get("data"))

        # Invoke orchestration service
        await service.process_created_event(payload)

        return True  # Success
    except ValidationError:
        return False  # Failure
```

### Handler Signature Enforcement

**Strict Signature Validation** at decoration time prevents runtime errors:

```python
# EventRouter validates required parameters
class EventRouter:
    def subscribe(self, event_type: str):
        def decorator(handler: Callable):
            # Extract signature
            sig = inspect.signature(handler)

            # Required parameters
            required = {'event_data', 'message'}
            actual = set(sig.parameters.keys())

            # Validate
            if not required.issubset(actual):
                raise ValueError(
                    f"Handler {handler.__name__} missing required parameters: "
                    f"{required - actual}"
                )

            # Store metadata
            self._handlers[event_type] = HandlerMetadata(
                handler=handler,
                params=sig.parameters,
                depends=self._extract_depends(sig)
            )

            return handler
        return decorator
```

### Dependency Injection

**ContextVar-Based Container** for async event handlers:

```python
# controller/dependencies/event_dependencies.py
from contextvars import ContextVar

_event_container_context: ContextVar[Container] = ContextVar('event_container')

def set_event_container(container: Container) -> None:
    """Store container during worker startup."""
    _event_container_context.set(container)

def get_event_container() -> Container:
    """Retrieve container in handler context."""
    container = _event_container_context.get()
    if container is None:
        raise RuntimeError("Event container not initialized")
    return container

# Dependency providers
def get_document_service(
    container: Container = Depends(get_event_container)
) -> DocumentService:
    """Provide DocumentService with DI."""
    return container.document_service()

def get_unit_of_work(
    container: Container = Depends(get_event_container)
) -> UnitOfWork:
    """Provide fresh UnitOfWork per handler invocation."""
    return container.unit_of_work_factory()
```

**Dependency Resolution Flow**:

1. Worker startup: `set_event_container(container)`
2. Handler decoration: Extract `Depends()` metadata via `inspect.signature()`
3. Message processing: `EventRouter._resolve_dependencies()` invokes callables
4. Handler invocation: `await handler(**resolved_deps)`

---

## Event Handler Development

### Handler Implementation

**Step-by-Step Guide** for adding new event handlers:

#### 1. Define Event Schema

```python
# shared/dto/platform_dto/events/document_events.py

from pydantic import BaseModel, Field
from uuid import UUID
from datetime import datetime

class DocumentCreatedEventData(BaseModel):
    """Domain data for document.created event."""
    document_id: UUID = Field(..., description="Unique document identifier")
    title: str = Field(..., description="Document title")
    reporting_event_id: UUID
    deliverable_set_id: UUID
    created_by: UUID
    company_id: UUID

class DocumentCreatedEvent(BaseModel):
    """CloudEvents-compliant document.created event."""
    spec_version: str = "1.0"
    type: str = "document.created"
    source: str
    id: str
    time: datetime
    data: DocumentCreatedEventData
    correlation_id: UUID
```

#### 2. Create Event Handler

```python
# services/document-service/document_service/controller/events/v1/document_events.py

from platform_event.routing import event_router
from azure.servicebus import ServiceBusReceivedMessage
from fastapi import Depends
from pydantic import ValidationError

@event_router.subscribe(event_type=EventType.DOCUMENT_CREATED.value)
async def handle_document_created(
    event_data: dict,
    message: ServiceBusReceivedMessage,
    service: DocumentService = Depends(get_document_service)
) -> bool:
    """
    Process document.created events.

    Business Logic:
    - Update search index
    - Trigger AI analysis
    - Send notifications

    Returns:
        True: Successfully processed
        False: Validation or business logic failure
    """
    try:
        # 1. Validate schema
        payload = DocumentCreatedEvent.model_validate(event_data.get("data"))

        # 2. Log processing start
        logger.info(
            f"Processing document.created event",
            extra={
                "document_id": str(payload.data.document_id),
                "correlation_id": str(message.correlation_id),
                "event_type": EventType.DOCUMENT_CREATED.value
            }
        )

        # 3. Invoke orchestration service
        await service.process_created_event(payload.data)

        # 4. Success
        logger.info(
            f"Document created event processed successfully",
            extra={"document_id": str(payload.data.document_id)}
        )
        return True

    except ValidationError as e:
        # Schema validation failure
        logger.warning(
            f"Invalid event schema: {e}",
            extra={
                "event_type": EventType.DOCUMENT_CREATED.value,
                "correlation_id": str(message.correlation_id)
            }
        )
        return False

    except Exception as e:
        # Unexpected error
        logger.error(
            f"Handler exception: {e}",
            exc_info=True,
            extra={
                "event_type": EventType.DOCUMENT_CREATED.value,
                "correlation_id": str(message.correlation_id)
            }
        )
        return False
```

#### 3. Register Handler

```python
# services/document-service/document_service/controller/events/__init__.py

from platform_event.routing import EventRouter

def register_event_handlers(event_router: EventRouter) -> None:
    """
    Auto-discover and register all event handlers.

    Importing v1 module triggers decorator execution.
    """
    # Import triggers @event_router.subscribe() decorators
    from . import v1  # noqa: F401

    # Validate expected handlers registered
    registered_types = set(event_router.get_handlers().keys())
    expected_types = {
        EventType.DOCUMENT_CREATED.value,
        EventType.DOCUMENT_REVISIONED.value,
        EventType.DOCUMENT_DELETED.value,
    }

    missing = expected_types - registered_types
    if missing:
        raise ValueError(f"Missing event handlers: {missing}")

    logger.info(f"Registered {len(registered_types)} event handlers")
```

### Handler Registration

**Auto-Discovery Pattern** via module imports:

```python
# services/document-service/document_service/eventapp.py

from document_service.controller.events import register_event_handlers

# Module-level singleton
event_router = EventRouter()

async def startup():
    # ... initialize container ...

    # Register handlers (imports trigger decorators)
    register_event_handlers(event_router)

    # Create EventProcessor
    processor = EventProcessor(
        event_router=event_router,
        # ... config ...
    )
```

### Message Lifecycle

**Return Value Semantics** control message settlement:

| Return Value | Behavior | Use Case |
|--------------|----------|----------|
| `True` | Auto-complete message | Handler succeeded |
| `False` | Auto-dead-letter message | Handler validation failed |
| `None` | Raise `ValueError` | Programming error |

**Message Settlement Flow**:

```python
# EventProcessor._execute_handler()

async def _execute_handler(handler, event_data, message, ...):
    # 1. Resolve dependencies
    resolved_deps = event_router._resolve_dependencies(
        handler=handler,
        container=container,
        event_data=event_data,
        message=message
    )

    # 2. Invoke handler with correlation context
    async with correlation_context(message.correlation_id):
        result = await handler(**resolved_deps)

    # 3. Settle message based on return value
    if result is True:
        # Success: complete message
        await message_settlement_service.complete_message(message)

    elif result is False:
        # Failure: dead-letter with metadata
        await message_settlement_service.dead_letter_message(
            message=message,
            reason="Handler explicitly failed",
            application_properties={
                "handler_name": handler.__name__,
                "failure_reason": "Handler returned False",
                "correlation_id": str(message.correlation_id),
                "event_type": event_type,
            }
        )

    elif result is None:
        # Programming error
        raise ValueError(
            f"Handler {handler.__name__} returned None. "
            "Must return True or False."
        )
```

---

## Transactional Outbox Pattern

### Problem Statement

**Dual-Write Problem**: Atomically updating database and publishing events to message broker is impossible without distributed transactions.

**Failure Scenarios**:
- Database commit succeeds, message publish fails → Lost event
- Message publish succeeds, database commit fails → Ghost event
- Partial failure → Inconsistent state

### Solution: Transactional Outbox

**Write events and data to same database atomically**, then publish asynchronously:

```python
# orchestration/document/document_upload_service.py

from platform_infrastructure.repositories.unit_of_work import UnitOfWork

class DocumentUploadService:
    async def upload_document(self, request: UploadDocumentDTO) -> DocumentDTO:
        """Upload document with transactional outbox."""

        async with UnitOfWork(self._session_factory) as uow:
            # 1. Persist entity
            document = Document(
                id=uuid4(),
                title=request.title,
                # ... other fields ...
            )
            await self._document_repository.save(document, session=uow.session)

            # 2. Add domain event to transaction
            event = DocumentCreatedEvent(
                spec_version="1.0",
                type=EventType.DOCUMENT_CREATED.value,
                source=f"document-service/{document.id}",
                id=str(uuid4()),
                time=datetime.utcnow(),
                data=DocumentCreatedEventData(
                    document_id=document.id,
                    title=document.title,
                    # ... event data ...
                ),
                correlation_id=correlation_id
            )
            uow.add_event(event)

            # 3. Atomic commit (entity + event)
            # UnitOfWork.__aexit__ commits transaction

        # 4. Event published after successful commit
        # EventPublisher processes outbox asynchronously

        return DocumentMapper.to_dto(document)
```

**UnitOfWork Implementation**:

```python
class UnitOfWork:
    def __init__(self, session_factory):
        self._session_factory = session_factory
        self._session = None
        self._events: List[BaseModel] = []

    async def __aenter__(self):
        self._session = self._session_factory()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is not None:
            # Rollback on exception
            await self._session.rollback()
            return False

        try:
            # Commit transaction
            await self._session.commit()

            # Publish events after successful commit
            await self._publish_events()
        except Exception:
            await self._session.rollback()
            raise
        finally:
            await self._session.close()

    def add_event(self, event: BaseModel):
        """Add event to outbox."""
        self._events.append(event)

    async def _publish_events(self):
        """Publish events to Service Bus."""
        publisher = get_event_publisher()
        for event in self._events:
            await publisher.publish(event)
```

### Benefits

- ✅ **Atomic guarantee** - Events and data committed together
- ✅ **No lost events** - Database failure prevents event publishing
- ✅ **No ghost events** - Events only published after successful commit
- ✅ **Eventual consistency** - Events processed asynchronously

---

## Resilience Patterns

### Idempotency Service

**Redis-Backed Deduplication** with atomic Lua scripts:

```python
# shared/infrastructure/platform_infrastructure/services/idempotency_service.py

class IdempotencyService:
    """Redis-backed idempotency with 24-hour TTL."""

    async def is_processed(self, event_id: str) -> bool:
        """
        Check if event already processed.

        Returns:
            True: Event already processed (duplicate)
            False: Event not seen before
        """
        # Atomic check-and-set with Lua script
        result = await self._redis.eval(
            """
            if redis.call("EXISTS", KEYS[1]) == 1 then
                return 1
            else
                redis.call("SETEX", KEYS[1], ARGV[1], ARGV[2])
                return 0
            end
            """,
            keys=[f"event:processed:{event_id}"],
            args=[86400, "true"]  # 24-hour TTL
        )
        return bool(result)
```

**Usage in EventProcessor**:

```python
async def _process_message(self, message: ServiceBusReceivedMessage):
    # Extract event ID
    event_id = message.message_id

    # Idempotency check
    if await self._idempotency_service.is_processed(event_id):
        logger.info(f"Duplicate event {event_id}, skipping")
        await self._message_settlement.complete_message(message)
        return

    # Process handler
    await self._execute_handler(...)
```

### Circuit Breaker

**Distributed State Coordination** with Redis:

```python
# shared/infrastructure/platform_infrastructure/services/circuit_breaker.py

class CircuitBreaker:
    """
    Distributed circuit breaker with states:
    - CLOSED: Normal operation
    - OPEN: Fail-fast mode
    - HALF_OPEN: Recovery testing
    """

    async def call(self, func: Callable, *args, **kwargs):
        """Execute function with circuit breaker protection."""

        state = await self._get_state()

        if state == CircuitState.OPEN:
            # Check if recovery timeout elapsed
            if await self._should_attempt_reset():
                await self._set_state(CircuitState.HALF_OPEN)
            else:
                raise CircuitBreakerError("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)

            # Success in HALF_OPEN → transition to CLOSED
            if state == CircuitState.HALF_OPEN:
                await self._set_state(CircuitState.CLOSED)
                await self._reset_failure_count()

            return result

        except Exception as e:
            # Increment failure count
            failures = await self._increment_failure_count()

            # Transition to OPEN if threshold exceeded
            if failures >= self._failure_threshold:
                await self._set_state(CircuitState.OPEN)
                await self._set_open_timestamp()

            raise

    async def _set_state(self, state: CircuitState):
        """Atomically set circuit breaker state in Redis."""
        await self._redis.set(
            f"circuit:{self._name}:state",
            state.value,
            ex=3600  # 1-hour TTL
        )
```

**Usage in Event Handlers**:

```python
@event_router.subscribe(event_type="document.created")
async def handle_document_created(
    event_data: dict,
    message: ServiceBusReceivedMessage,
    circuit_breaker: CircuitBreaker = Depends(get_circuit_breaker)
) -> bool:
    try:
        # External API call protected by circuit breaker
        await circuit_breaker.call(
            external_api_client.notify_created,
            document_id=event_data["document_id"]
        )
        return True
    except CircuitBreakerError:
        # Circuit open - don't retry
        logger.warning("Circuit breaker OPEN, skipping external notification")
        return True  # Complete message anyway
```

### Dead-Letter Handling

**Enriched Metadata** for troubleshooting:

```python
# Dead-lettered messages include application_properties

application_properties = {
    "handler_name": "handle_document_created",
    "failure_reason": "Handler returned False",
    "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
    "event_type": "document.created",
    "timestamp": "2025-12-03T10:30:45Z",
    "worker_id": "doc-worker-1",
    "event_version": "1",
    "original_arrival_time": "2025-12-03T10:30:40Z",
}
```

**Dead-Letter Queue Management**:

```bash
# Azure CLI - View dead-letter messages
az servicebus topic subscription show \
  --resource-group platform-rg \
  --namespace-name platform-sb \
  --topic-name clinical.documents.events \
  --name ordered-subscription \
  --query "countDetails.deadLetterMessageCount"

# Peek dead-letter messages
az servicebus topic subscription dead-letter peek \
  --resource-group platform-rg \
  --namespace-name platform-sb \
  --topic-name clinical.documents.events \
  --name ordered-subscription \
  --max-message-count 10
```

---

## Correlation Context

### Correlation ID Propagation

**End-to-End Tracing** with `correlation_id`:

```
User Request → API → Orchestration → Database → Event → Handler → Logs
     ↓           ↓         ↓             ↓         ↓        ↓        ↓
correlation_id flows automatically through entire system
```

### Implementation

**Async Context Manager**:

```python
# shared/common/platform_common/correlation.py

from contextvars import ContextVar

_correlation_id_context: ContextVar[Optional[str]] = ContextVar(
    'correlation_id',
    default=None
)

@asynccontextmanager
async def correlation_context(correlation_id: str):
    """Set correlation_id in async context."""
    token = _correlation_id_context.set(correlation_id)
    try:
        yield
    finally:
        _correlation_id_context.reset(token)

def get_correlation_id() -> Optional[str]:
    """Retrieve correlation_id from async context."""
    return _correlation_id_context.get()
```

**Usage in EventProcessor**:

```python
async def _execute_handler(self, handler, message, ...):
    # Extract correlation_id from message
    correlation_id = message.correlation_id or str(uuid4())

    # Set in async context
    async with correlation_context(correlation_id):
        # Handler and all downstream calls inherit correlation_id
        result = await handler(...)
```

**Structured Logging**:

```python
import structlog

logger = structlog.get_logger()

# Automatically includes correlation_id in logs
logger.info(
    "Processing event",
    event_type="document.created",
    document_id=str(document_id)
)
# Output: {"timestamp": "...", "correlation_id": "550e8400-...", "event_type": "document.created", ...}
```

---

## Testing Event Handlers

### Unit Testing Handlers

**Mock Service Dependencies**:

```python
# tests/unit/controller/events/test_document_events.py

import pytest
from unittest.mock import AsyncMock
from azure.servicebus import ServiceBusReceivedMessage

@pytest.fixture
def mock_document_service():
    """Mock orchestration service."""
    service = AsyncMock(spec=DocumentService)
    service.process_created_event.return_value = None
    return service

@pytest.fixture
def mock_message():
    """Mock Service Bus message."""
    message = AsyncMock(spec=ServiceBusReceivedMessage)
    message.correlation_id = "550e8400-e29b-41d4-a716-446655440000"
    message.message_id = "msg-123"
    return message

async def test_handle_document_created_success(
    mock_document_service,
    mock_message
):
    """Test successful document.created event processing."""
    # Arrange
    event_data = {
        "data": {
            "document_id": "550e8400-e29b-41d4-a716-446655440001",
            "title": "Test Document",
            "reporting_event_id": "550e8400-e29b-41d4-a716-446655440002",
            "deliverable_set_id": "550e8400-e29b-41d4-a716-446655440003",
            "created_by": "550e8400-e29b-41d4-a716-446655440004",
            "company_id": "550e8400-e29b-41d4-a716-446655440005",
        }
    }

    # Act
    result = await handle_document_created(
        event_data=event_data,
        message=mock_message,
        service=mock_document_service
    )

    # Assert
    assert result is True
    mock_document_service.process_created_event.assert_called_once()

async def test_handle_document_created_validation_failure(
    mock_document_service,
    mock_message
):
    """Test document.created with invalid schema."""
    # Arrange - missing required fields
    event_data = {"data": {}}

    # Act
    result = await handle_document_created(
        event_data=event_data,
        message=mock_message,
        service=mock_document_service
    )

    # Assert
    assert result is False  # Validation failure
    mock_document_service.process_created_event.assert_not_called()
```

### Integration Testing with Service Bus

**Testcontainers Pattern** (deferred to separate story):

```python
# tests/integration/events/test_event_processor.py

import pytest
from testcontainers.azure import AzureServiceBusContainer

@pytest.fixture
async def service_bus_container():
    """Start Azure Service Bus emulator."""
    container = AzureServiceBusContainer()
    container.start()

    # Configure connection
    connection_string = container.get_connection_string()

    yield connection_string

    container.stop()

async def test_event_processor_end_to_end(service_bus_container):
    """Test complete event processing flow."""
    # Publish test event
    publisher = EventPublisher(service_bus_container)
    event = DocumentCreatedEvent(...)
    await publisher.publish(event)

    # Start processor
    processor = EventProcessor(...)
    await processor.start()

    # Wait for processing
    await asyncio.sleep(2)

    # Verify handler invoked
    assert mock_handler.call_count == 1
```

---

## Monitoring & Observability

### Key Metrics

| Metric | Type | Target | Alert Threshold |
|--------|------|--------|-----------------|
| `event_processing_duration_seconds` | Histogram | p99 &lt; 30s | p99 > 60s |
| `event_processing_total` | Counter | Monitor trends | N/A |
| `event_handler_errors_total` | Counter | &lt; 1% error rate | > 10% in 5min |
| `event_dead_letter_queue_depth` | Gauge | &lt; 50 | > 100 |
| `circuit_breaker_state` | Gauge | 0 (closed) | 1 (open) for 5min |

### Health Checks

**Event Worker Health Endpoint** (`GET /health`):

```json
{
  "status": "healthy",
  "timestamp": "2025-12-03T10:30:45Z",
  "components": {
    "event_processor": {
      "status": "healthy",
      "handlers_registered": 12,
      "subscriptions_active": 3,
      "messages_processed_total": 150000
    },
    "service_bus": {
      "status": "healthy",
      "connected": true
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 2.5
    }
  }
}
```

### Distributed Tracing

**Azure Application Insights KQL**:

```kusto
// Trace all operations for a correlation_id
customEvents
| where customDimensions.correlation_id == "550e8400-e29b-41d4-a716-446655440000"
| project timestamp, operation_Name, duration, customDimensions
| order by timestamp asc

// Find slow event handlers
customEvents
| where name == "event_processing_complete"
| where customDimensions.duration_ms > 30000
| summarize count() by tostring(customDimensions.handler_name)
```

---

## Troubleshooting Guide

### Common Issues

#### Dead-Letter Queue Growing

**Symptoms**: Dead-letter message count increasing

**Diagnosis**:
```bash
# Check dead-letter messages
az servicebus topic subscription dead-letter peek \
  --topic-name clinical.documents.events \
  --name ordered-subscription \
  --max-message-count 10
```

**Solutions**:
1. Check `application_properties.failure_reason` for error pattern
2. Review handler logs with `correlation_id`
3. Fix validation issues or business logic bugs
4. Resubmit messages after fix

#### Circuit Breaker Open

**Symptoms**: `CircuitBreakerError` in logs

**Diagnosis**:
```bash
# Check circuit breaker state in Redis
redis-cli GET circuit:external_api:state
```

**Solutions**:
1. Verify external service availability
2. Check network connectivity
3. Manually reset circuit breaker if service recovered:
   ```bash
   redis-cli DEL circuit:external_api:state
   redis-cli DEL circuit:external_api:failures
   ```

#### Slow Event Processing

**Symptoms**: High `event_processing_duration_seconds` p99

**Diagnosis**:
1. Check handler execution time in Application Insights
2. Review database query performance
3. Check external API latency

**Solutions**:
1. Optimize database queries
2. Add caching for repeated lookups
3. Increase worker concurrency
4. Scale up KEDA replicas

---

## Further Reading

- **Worker Operations**: `docs/event-driven-workers.md` - KEDA scaling, operational runbooks
- **Story 6.1.3**: `bmad_docs/stories/story-6.1.3.md` - SubscriberPool and tiered workers
- **Story 6.1.4**: `bmad_docs/stories/story-6.1.4.md` - EventRouter decorator pattern
- **Story 6.1.5**: `bmad_docs/stories/story-6.1.5.md` - Resilience patterns (Protocol, exceptions)
- **Clean Architecture**: `docs/architecture/clean-architecture.md` - Layer separation principles
