---
id: command-queue
title: Command-Queue Architecture
description: Comprehensive guide to the platform's command-queue architecture with Azure Service Bus queues, CommandRouter pattern, and exponential backoff retry
sidebar_label: Command-Queue Architecture
---

# COMMAND-QUEUE ARCHITECTURE

## Table of Contents

- [Overview](#overview)
- [Commands vs Events](#commands-vs-events)
- [Architecture Principles](#architecture-principles)
- [Azure Service Bus Queue Integration](#azure-service-bus-queue-integration)
  - [Queue Hierarchy](#queue-hierarchy)
  - [Command Schema](#command-schema)
  - [Delivery Guarantees](#delivery-guarantees)
- [Tiered Worker Architecture](#tiered-worker-architecture)
  - [Tier 1: Dedicated Workers](#tier-1-dedicated-workers)
  - [Tier 2: Background Workers](#tier-2-background-workers)
  - [Worker Configuration](#worker-configuration)
- [CommandRouter Pattern](#commandrouter-pattern)
  - [Decorator-Based Registration](#decorator-based-registration)
  - [Handler Signature Enforcement](#handler-signature-enforcement)
  - [Dependency Injection](#dependency-injection)
- [Command Handler Development](#command-handler-development)
  - [Handler Implementation](#handler-implementation)
  - [Handler Registration](#handler-registration)
  - [Message Lifecycle](#message-lifecycle)
- [Exponential Backoff Retry](#exponential-backoff-retry)
- [Resilience Patterns](#resilience-patterns)
  - [Idempotency Service](#idempotency-service)
  - [Circuit Breaker](#circuit-breaker)
  - [Dead-Letter Handling](#dead-letter-handling)
- [Correlation Context](#correlation-context)
- [Testing Command Handlers](#testing-command-handlers)
- [Monitoring & Observability](#monitoring--observability)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

The platform implements a **native Azure Service Bus command-queue architecture** for point-to-point task processing. The architecture complements the event-driven system by providing:

- **Point-to-point task distribution** with single-consumer guarantees
- **Reliable retry with exponential backoff** for transient failures
- **Long-running task processing** without blocking API responses
- **Work distribution** across horizontally scaled workers
- **Developer experience** matching FastAPI-style patterns

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Azure Service Bus Queues** | Point-to-point delivery with exactly-once processing guarantees |
| **Exponential Backoff Retry** | Automatic recovery from transient failures (2s, 4s, 8s) |
| **Separate from Events** | Clear distinction between notifications (events) and tasks (commands) |
| **CommandProcessor** | Specialized processor for queue-based command handling |
| **Strict Return Contracts** | `True`/`False` semantics for explicit retry/dead-letter control |

---

## Commands vs Events

### Semantic Differences

| Aspect | **Events** (Pub/Sub) | **Commands** (Point-to-Point) |
|--------|---------------------|-------------------------------|
| **Pattern** | Publish/Subscribe | Task Queue |
| **Transport** | Azure Service Bus Topics | Azure Service Bus Queues |
| **Intent** | "Something happened" (notification) | "Do this task" (instruction) |
| **Consumers** | Multiple subscribers (0 to N) | Single consumer (exactly 1) |
| **Delivery** | Fire-and-forget with at-least-once | Guaranteed processing with retry |
| **Naming** | Past tense (`document.created`) | Imperative (`generate_thumbnail`) |
| **Failure** | Dead-letter immediately | Retry with exponential backoff |
| **Use Cases** | State change notifications, audit | Long-running tasks, async operations |

### When to Use Commands

✅ **Use Commands for**:
- Long-running background tasks (thumbnail generation, PDF processing)
- Async operations that must complete (email sending, file conversion)
- Work distribution across workers (batch processing, data imports)
- Tasks requiring retry logic (external API calls, transient failures)
- Point-to-point task assignment (single handler responsibility)

❌ **Don't Use Commands for**:
- Broadcasting state changes to multiple services (use events)
- Real-time notifications to multiple consumers (use events)
- Event sourcing or audit trails (use events)
- Loosely coupled integration (use events)

### Example Scenarios

```python
# ❌ BAD: Using events for task execution
@event_router.subscribe(event_type="document.thumbnail.requested")
async def generate_thumbnail(event_data: dict, message: ServiceBusReceivedMessage):
    # Multiple subscribers might process this (not what we want!)
    await thumbnail_service.generate(document_id)

# ✅ GOOD: Using commands for task execution
@command_router.subscribe(command_type=CommandType.GENERATE_DOCUMENT_THUMBNAIL)
async def generate_thumbnail(command_data: dict, message: ServiceBusReceivedMessage):
    # Only one worker processes this command
    await thumbnail_service.generate(command_data["document_id"])
```

---

## Architecture Principles

### Clean Architecture Integration

Command handlers follow the same Clean Architecture separation as event handlers:

```
services/{service}/{service}_service/
├── controller/
│   ├── api/v1/              # REST endpoint handlers
│   ├── events/              # Event handlers (pub/sub)
│   └── commands/            # Command handlers (point-to-point)
│       ├── __init__.py      # register_command_handlers()
│       └── v1/
│           └── {context}_commands.py  # @command_router.subscribe()
├── orchestration/           # Business logic (shared by API, events, commands)
├── domain/                  # Domain entities & logic
└── infrastructure/          # External concerns
```

**Key Principles**:
- **Controllers invoke orchestration** - Command handlers call same orchestration services as API/event handlers
- **No business logic in handlers** - Handlers only validate and delegate
- **Explicit return values** - `True`/`False` for message lifecycle control
- **Dependency injection** - `Depends()` pattern matches FastAPI

### Command Processing Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     Command Processing Flow                  │
│                                                              │
│  1. API Request (Async Task Initiation)                     │
│     └─> Orchestration Service                               │
│         └─> CommandPublisher.send_command()                 │
│             └─> Publish to Azure Service Bus Queue          │
│                                                              │
│  2. Async Processing (Worker Consumes Command)              │
│     └─> CommandProcessor receives message                   │
│         └─> CommandRouter invokes handler                   │
│             └─> Orchestration Service (with DI)             │
│                 └─> Complete/Retry/Dead-letter              │
│                                                              │
│  3. Retry Logic (On Failure)                                │
│     └─> Exponential backoff: 2s, 4s, 8s                    │
│         └─> Auto-retry up to max_retry_attempts (3)        │
│             └─> Dead-letter after exhausting retries        │
└─────────────────────────────────────────────────────────────┘
```

---

## Azure Service Bus Queue Integration

### Queue Hierarchy

**Organized by Domain and Task Type**:

```
document-commands              # Document processing tasks
├── generate-thumbnail
├── process-pdf
└── extract-metadata

integration-commands           # External system integration
├── sync-teamwork-data
├── send-notification-email
└── export-report

maintenance-commands           # System maintenance
├── cleanup-expired-files
├── recalculate-statistics
└── archive-old-data
```

**Naming Convention**: `{domain}-commands` for the queue name

### Command Schema

**Structured Command Payload** with CloudEvents-compatible metadata:

```python
from pydantic import BaseModel, Field
from datetime import datetime
from uuid import UUID
from enum import Enum

class CommandType(str, Enum):
    """Enumeration of command types."""
    GENERATE_DOCUMENT_THUMBNAIL = "generate_document_thumbnail"
    PROCESS_PDF_DOCUMENT = "process_pdf_document"
    SEND_NOTIFICATION_EMAIL = "send_notification_email"

class BaseCommand(BaseModel):
    """Base command structure following CloudEvents specification."""

    # CloudEvents metadata
    spec_version: str = "1.0"
    type: str  # Command type (e.g., "generate_document_thumbnail")
    source: str  # Service that issued command
    id: str  # Unique command ID
    time: datetime

    # Command data
    data: dict  # Command-specific payload
    correlation_id: UUID
    tenant_id: UUID  # Multi-tenancy support

class GenerateThumbnailCommand(BaseCommand):
    """Command to generate document thumbnail."""

    type: str = Field(default=CommandType.GENERATE_DOCUMENT_THUMBNAIL.value)

    class DataModel(BaseModel):
        document_id: UUID
        size: str = "medium"  # "small", "medium", "large"
        priority: int = 5  # 1-10 (10 = highest)

    data: DataModel
```

**Command Publishing**:

```python
# In orchestration service
from platform_infrastructure.messaging.command_publisher import CommandPublisher

async def request_thumbnail_generation(
    self,
    document_id: UUID,
    size: str = "medium"
) -> None:
    """Publish command to generate document thumbnail."""

    command = GenerateThumbnailCommand(
        source=f"{self._service_name}.document-service",
        id=str(uuid4()),
        time=datetime.utcnow(),
        data=GenerateThumbnailCommand.DataModel(
            document_id=document_id,
            size=size,
            priority=5
        ),
        correlation_id=get_correlation_id(),
        tenant_id=get_tenant_id()
    )

    await self._command_publisher.send_command(
        queue_name="document-commands",
        command=command.model_dump()
    )
```

### Delivery Guarantees

| Feature | Configuration | Purpose |
|---------|---------------|---------|
| **Exactly-Once Processing** | Queue + idempotency | Prevents duplicate task execution |
| **Exponential Backoff** | 2s, 4s, 8s (configurable) | Gradual retry for transient failures |
| **Max Retry Attempts** | 3 attempts (default) | Balance between recovery and speed |
| **Peek-Lock** | 5-minute lock duration | Automatic redelivery on timeout |
| **Dead-Letter** | Auto-move after max retries | Manual investigation required |
| **FIFO Ordering** | Optional session support | Strict ordering within session |

---

## Tiered Worker Architecture

### Strategy Overview

**Two-Tier Design** shared with event workers for consistency:

```
┌─────────────────────────────────────────────────────────────┐
│                      TIER 1: DEDICATED                        │
│                                                              │
│  • One queue per worker instance                            │
│  • Session-aware for strict ordering                        │
│  • KEDA auto-scaling based on queue depth                  │
│  • Reserved for critical/ordered tasks                     │
│                                                              │
│  Example: document-thumbnail-worker-dedicated               │
│    └─> Processes: document-commands queue only             │
│        with session ordering for each document              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     TIER 2: BACKGROUND                        │
│                                                              │
│  • Multiple queues pooled in one worker                     │
│  • No session support (cost optimization)                  │
│  • Lower priority, batch processing                        │
│  • Fixed replica count                                     │
│                                                              │
│  Example: document-background-worker                        │
│    └─> Processes: maintenance-commands,                     │
│                   notification-commands,                     │
│                   cleanup-commands                           │
└─────────────────────────────────────────────────────────────┘
```

### Tier 1: Dedicated Workers

**Use for**: Critical tasks requiring guaranteed processing or strict ordering

```yaml
# config/cwa_config.yaml (dedicated)
worker:
  worker_tier: dedicated  # Single queue

  queue:
    name: document_thumbnail_queue
    queue: document-commands
    session_enabled: true  # Strict FIFO ordering
    max_concurrent: 5
    prefetch_count: 10

  max_concurrent_handlers: 5
  shutdown_timeout: 30
  health_check_host: 0.0.0.0
  health_check_port: 8089
```

**KEDA Scaling**:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: document-thumbnail-worker-dedicated
spec:
  scaleTargetRef:
    name: document-thumbnail-worker
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: document-commands
        messageCount: "10"  # Scale up when > 10 messages
```

### Tier 2: Background Workers

**Use for**: Lower-priority batch tasks, maintenance operations

```yaml
# config/cwa_config.yaml (background)
worker:
  worker_tier: background  # Multiple queues

  queues:
    - name: maintenance_queue
      queue: maintenance-commands
      session_enabled: false
      max_concurrent: 2
      prefetch_count: 5

    - name: notification_queue
      queue: notification-commands
      session_enabled: false
      max_concurrent: 3
      prefetch_count: 5

    - name: cleanup_queue
      queue: cleanup-commands
      session_enabled: false
      max_concurrent: 1
      prefetch_count: 3

  max_concurrent_handlers: 10  # Total across all queues
  shutdown_timeout: 60  # Longer timeout for batch jobs
```

**Deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: document-background-worker
spec:
  replicas: 2  # Fixed replica count (no KEDA)
  template:
    spec:
      containers:
        - name: worker
          image: document-service-command-worker:latest
          args: ["--config", "config/cwa_config.yaml"]
```

### Worker Configuration

**Configuration File Structure** (`cwa_config.yaml`):

```yaml
# ============================================================================
# WORKER CONFIGURATION
# ============================================================================
worker:
  worker_tier: background  # "dedicated" or "background"

  queues:
    - name: document_command_queue
      queue: document-commands
      session_enabled: false
      max_concurrent: 5
      prefetch_count: 10

  max_concurrent_handlers: 10
  shutdown_timeout: 30
  health_check_host: 0.0.0.0
  health_check_port: 8089
  log_level: info

# ============================================================================
# PROCESSING CONFIGURATION
# ============================================================================
processing:
  message_receive_timeout: 30.0
  max_lock_renewal_duration: 300
  error_backoff_seconds: 5

  # Exponential backoff retry configuration
  max_retry_attempts: 3  # Total attempts before dead-letter
  retry_backoff_base_seconds: 2.0  # Initial delay
  retry_backoff_multiplier: 2.0  # Exponential growth
  max_retry_backoff_seconds: 60.0  # Cap on delay

  idempotency_ttl_seconds: 86400
  circuit_breaker_failure_threshold: 5
  circuit_breaker_timeout_seconds: 60

# ============================================================================
# METRICS CONFIGURATION
# ============================================================================
metrics:
  backend: noop  # "azure_monitor", "prometheus", or "noop"
  connection_string: null
```

---

## CommandRouter Pattern

### Decorator-Based Registration

**FastAPI-Style Command Registration**:

```python
from platform_event.routing import command_router
from azure.servicebus import ServiceBusReceivedMessage
from fastapi import Depends

@command_router.subscribe(command_type=CommandType.GENERATE_DOCUMENT_THUMBNAIL)
async def handle_generate_thumbnail(
    command_data: dict,                          # Required parameter
    message: ServiceBusReceivedMessage,          # Required parameter
    service: ThumbnailService = Depends(get_thumbnail_service)  # DI
) -> bool:
    """
    Process generate_document_thumbnail commands.

    Args:
        command_data: Parsed command payload
        message: Azure Service Bus message (for correlation_id, metadata)
        service: Injected orchestration service

    Returns:
        True: Successfully processed → auto-complete
        False: Handler failed → retry with exponential backoff
    """
    try:
        # Validate command schema
        payload = GenerateThumbnailCommand.model_validate(command_data)

        # Invoke orchestration service
        await service.generate_thumbnail(
            document_id=payload.data.document_id,
            size=payload.data.size
        )

        return True  # Success - complete message
    except ValidationError as e:
        logger.error(f"Invalid command schema: {e}")
        return False  # Schema error - retry won't help, but let backoff handle it
    except Exception as e:
        logger.error(f"Failed to generate thumbnail: {e}")
        return False  # Failure - trigger retry with exponential backoff
```

### Handler Signature Enforcement

**Strict Signature Validation** prevents runtime errors:

```python
class CommandRouter:
    """Command router with strict signature enforcement."""

    def subscribe(self, command_type: CommandType):
        def decorator(handler: Callable):
            # Extract signature
            sig = inspect.signature(handler)

            # Required parameters
            required = {'command_data', 'message'}
            actual = set(sig.parameters.keys())

            # Validate
            if not required.issubset(actual):
                raise ValueError(
                    f"Handler {handler.__name__} missing required parameters: "
                    f"{required - actual}"
                )

            # Store metadata
            self._handlers[command_type.value] = HandlerMetadata(
                handler=handler,
                params=sig.parameters,
                depends=self._extract_depends(sig),
                command_type=command_type
            )

            return handler
        return decorator
```

**Parameter Requirements**:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `command_data` | `dict` | ✅ Yes | Parsed command payload |
| `message` | `ServiceBusReceivedMessage` | ✅ Yes | Raw Service Bus message |
| Dependencies | Any with `Depends()` | ❌ No | Injected services |

### Dependency Injection

**ContextVar-Based Container** for async command handlers:

```python
# controller/dependencies/command_dependencies.py
from contextvars import ContextVar
from typing import Optional
from document_service.ioc import Container

_command_container_context: ContextVar[Optional["Container"]] = ContextVar(
    "command_container", default=None
)

def set_command_container(container: Container) -> None:
    """Set DI container for command handler context."""
    _command_container_context.set(container)

def get_command_container() -> Container:
    """Get DI container from command handler context."""
    container = _command_container_context.get()
    if container is None:
        raise RuntimeError(
            "No command container found in async context. "
            "Ensure set_command_container() was called before handler execution."
        )
    return container

# Dependency providers
def get_thumbnail_service() -> ThumbnailService:
    """Get ThumbnailService instance from command container."""
    container = get_command_container()
    return container.thumbnail_service()
```

**Worker Initialization** (sets up DI context):

```python
# cwa.py (Command Worker Application)
from document_service.controller.dependencies.command_dependencies import (
    set_command_container,
)

async def main():
    # Initialize container
    container = Container()
    await container.init_resources()

    # Set container in command context
    set_command_container(container)

    # Create command processor
    command_processor = CommandProcessor(
        command_router=command_router,
        settings=settings,
        container=container,
    )

    # Start processing
    await command_processor.start_processing()
```

---

## Command Handler Development

### Handler Implementation

**Complete Handler Example**:

```python
# controller/commands/v1/document_commands.py
from platform_event.routing import CommandRouter
from azure.servicebus import ServiceBusReceivedMessage
from fastapi import Depends
from pydantic import ValidationError

from document_service.controller.dependencies.command_dependencies import (
    get_thumbnail_service,
    get_pdf_processor_service,
)
from document_service.orchestration.thumbnail.thumbnail_service import ThumbnailService
from document_service.orchestration.pdf.pdf_processor_service import PDFProcessorService

# Create sub-router for document commands
command_router = CommandRouter()


@command_router.subscribe(command_type=CommandType.GENERATE_DOCUMENT_THUMBNAIL)
async def handle_generate_thumbnail(
    command_data: dict,
    message: ServiceBusReceivedMessage,
    service: ThumbnailService = Depends(get_thumbnail_service),
) -> bool:
    """
    Generate thumbnail for a document.

    Accepts thumbnail generation commands and delegates to ThumbnailService
    for actual processing. Supports configurable thumbnail sizes.

    Args:
        command_data: Command payload with document_id and size
        message: Service Bus message for metadata
        service: Injected ThumbnailService

    Returns:
        True if thumbnail generated successfully, False to trigger retry
    """
    try:
        # Parse and validate command
        command = GenerateThumbnailCommand.model_validate(command_data)

        logger.info(
            f"Processing thumbnail command for document {command.data.document_id}",
            extra={
                "document_id": str(command.data.document_id),
                "size": command.data.size,
                "correlation_id": str(command.correlation_id),
            },
        )

        # Invoke orchestration service
        await service.generate_thumbnail(
            document_id=command.data.document_id,
            size=command.data.size,
            priority=command.data.priority,
        )

        logger.info(
            f"Successfully generated thumbnail for document {command.data.document_id}"
        )
        return True

    except ValidationError as e:
        logger.error(f"Invalid command schema: {e}", exc_info=True)
        return False  # Dead-letter after retries

    except Exception as e:
        logger.error(f"Failed to generate thumbnail: {e}", exc_info=True)
        return False  # Retry with exponential backoff


@command_router.subscribe(command_type=CommandType.PROCESS_PDF_DOCUMENT)
async def handle_process_pdf(
    command_data: dict,
    message: ServiceBusReceivedMessage,
    service: PDFProcessorService = Depends(get_pdf_processor_service),
) -> bool:
    """
    Process PDF document extraction and analysis.

    Extracts text, metadata, and structure from PDF documents.
    Long-running operation suitable for async processing.

    Args:
        command_data: Command payload with document_id
        message: Service Bus message for metadata
        service: Injected PDFProcessorService

    Returns:
        True if PDF processed successfully, False to trigger retry
    """
    try:
        command = ProcessPDFCommand.model_validate(command_data)

        logger.info(
            f"Processing PDF for document {command.data.document_id}",
            extra={
                "document_id": str(command.data.document_id),
                "correlation_id": str(command.correlation_id),
            },
        )

        # Long-running PDF processing
        result = await service.process_pdf(
            document_id=command.data.document_id,
            extract_text=command.data.extract_text,
            extract_metadata=command.data.extract_metadata,
        )

        logger.info(
            f"Successfully processed PDF: {result.pages_processed} pages",
            extra={"pages": result.pages_processed},
        )
        return True

    except ValidationError as e:
        logger.error(f"Invalid command schema: {e}", exc_info=True)
        return False

    except Exception as e:
        logger.error(f"Failed to process PDF: {e}", exc_info=True)
        return False
```

### Handler Registration

**Command Handler Registration** (similar to event handler registration):

```python
# controller/commands/__init__.py
from platform_event.routing.command_router import CommandRouter
from .v1 import document_commands, notification_commands


def register_command_handlers(main_router: CommandRouter) -> None:
    """
    Register all command handlers for the document service.

    Args:
        main_router: The main CommandRouter instance to register handlers with
    """
    _include_router(main_router, document_commands.command_router)
    _include_router(main_router, notification_commands.command_router)


def _include_router(main_router: CommandRouter, sub_router: CommandRouter) -> None:
    """Include a sub-router's handlers into the main router."""
    for command_type, metadata in sub_router._handlers.items():
        if command_type in main_router._handlers:
            raise ValueError(
                f"Command type '{command_type}' already registered. "
                f"Cannot include duplicate from sub-router."
            )
        main_router._handlers[command_type] = metadata
```

### Message Lifecycle

**Command Processing States**:

```
┌──────────────┐
│  Message     │
│  Received    │
└──────┬───────┘
       │
       ▼
┌──────────────┐      ┌─────────────┐
│  Idempotency │─────►│  Skip       │
│  Check       │      │  (Duplicate)│
└──────┬───────┘      └─────────────┘
       │
       ▼
┌──────────────┐
│  Invoke      │
│  Handler     │
└──────┬───────┘
       │
       ├─────────────┐
       │             │
   True (Success)  False (Failure)
       │             │
       ▼             ▼
┌──────────────┐  ┌──────────────┐
│  Complete    │  │  Check       │
│  Message     │  │  Retry Count │
└──────────────┘  └──────┬───────┘
                         │
                         ├──────────────┐
                         │              │
                   < Max Retries    ≥ Max Retries
                         │              │
                         ▼              ▼
                  ┌──────────────┐  ┌──────────────┐
                  │  Abandon     │  │  Dead-Letter │
                  │  (Retry)     │  │  Message     │
                  └──────────────┘  └──────────────┘
```

**Return Value Semantics**:

| Return Value | Action | Next State |
|--------------|--------|------------|
| `True` | Auto-complete message | Removed from queue |
| `False` | Check retry count | Abandon (retry) or Dead-letter |
| Exception | Same as `False` | Abandon or Dead-letter |

---

## Exponential Backoff Retry

### Retry Strategy

**Exponential Backoff Formula**:

```
delay = min(base * (multiplier ^ attempt), max_backoff)
```

**Default Configuration**:
- `retry_backoff_base_seconds`: 2.0 seconds
- `retry_backoff_multiplier`: 2.0
- `max_retry_backoff_seconds`: 60.0 seconds
- `max_retry_attempts`: 3

**Retry Sequence**:

| Attempt | Calculation | Delay | Notes |
|---------|-------------|-------|-------|
| 1 | `2.0 * (2.0 ^ 0)` | 2.0s | First retry (immediate) |
| 2 | `2.0 * (2.0 ^ 1)` | 4.0s | Second retry |
| 3 | `2.0 * (2.0 ^ 2)` | 8.0s | Third retry |
| 4 | N/A | Dead-letter | Max attempts exceeded |

### Implementation

**CommandProcessor Retry Logic**:

```python
# shared/event/platform_event/core/processor/command_processor.py
class CommandProcessor(BaseMessageProcessor):
    """Processes commands with exponential backoff retry."""

    def _should_retry_on_failure(
        self,
        message: ServiceBusReceivedMessage,
        error: Exception,
        handler_result: bool,
    ) -> bool:
        """
        Determine if command should be retried with exponential backoff.

        Args:
            message: The Service Bus message that failed
            error: The exception that occurred (if any)
            handler_result: The boolean result from handler

        Returns:
            True to abandon message (retry), False to dead-letter
        """
        delivery_count = message.delivery_count or 1
        max_attempts = self._settings.processing.max_retry_attempts

        # Check if max attempts exceeded
        if delivery_count > max_attempts:
            logger.info(
                f"Max retry attempts ({max_attempts}) exceeded, dead-lettering",
                extra={
                    "message_id": message.message_id,
                    "delivery_count": delivery_count,
                },
            )
            return False  # Dead-letter

        # Calculate exponential backoff delay (for logging)
        base = self._settings.processing.retry_backoff_base_seconds
        multiplier = self._settings.processing.retry_backoff_multiplier
        max_backoff = self._settings.processing.max_retry_backoff_seconds

        backoff_delay = min(
            base * (multiplier ** (delivery_count - 1)),
            max_backoff
        )

        logger.info(
            f"Command will retry with exponential backoff "
            f"(attempt {delivery_count}/{max_attempts}, delay: {backoff_delay:.1f}s)",
            extra={
                "message_id": message.message_id,
                "delivery_count": delivery_count,
                "backoff_seconds": backoff_delay,
            },
        )

        return True  # Abandon for retry
```

**Azure Service Bus** handles the actual delay between retries automatically using message lock expiration.

### Configuring Retry Behavior

**Custom Retry Configuration**:

```yaml
# config/cwa_config.yaml
processing:
  # Aggressive retry for critical operations
  max_retry_attempts: 5
  retry_backoff_base_seconds: 1.0
  retry_backoff_multiplier: 2.0
  max_retry_backoff_seconds: 30.0

  # Results in: 1s, 2s, 4s, 8s, 16s, then dead-letter
```

```yaml
# config/cwa_config.yaml (conservative)
processing:
  # Conservative retry for external API calls
  max_retry_attempts: 4
  retry_backoff_base_seconds: 5.0
  retry_backoff_multiplier: 3.0
  max_retry_backoff_seconds: 300.0

  # Results in: 5s, 15s, 45s, 135s, then dead-letter
```

---

## Resilience Patterns

### Idempotency Service

**Redis-Backed Duplicate Detection**:

```python
from platform_infrastructure.idempotency.redis_idempotency_service import (
    RedisIdempotencyService,
)

# In CommandProcessor
async def process_message(self, message: ServiceBusReceivedMessage) -> bool:
    """Process command with idempotency check."""

    # Check if already processed (using message ID)
    if await self._idempotency_service.is_processed(message.message_id):
        logger.info(
            f"Skipping duplicate command: {message.message_id}",
            extra={"message_id": message.message_id},
        )
        return True  # Already processed, complete message

    # Process command
    result = await self._invoke_handler(message)

    # Mark as processed (24h TTL)
    if result:
        await self._idempotency_service.mark_processed(
            message.message_id,
            ttl_seconds=self._settings.processing.idempotency_ttl_seconds
        )

    return result
```

**Configuration**:

```yaml
processing:
  idempotency_ttl_seconds: 86400  # 24 hours
```

### Circuit Breaker

**Automatic Handler Suspension**:

```python
from platform_infrastructure.circuit_breaker import CircuitBreaker

# In CommandProcessor initialization
self._circuit_breaker = CircuitBreaker(
    failure_threshold=settings.processing.circuit_breaker_failure_threshold,
    timeout_seconds=settings.processing.circuit_breaker_timeout_seconds,
    half_open_max_calls=settings.processing.circuit_breaker_half_open_max_calls,
)

# Before invoking handler
if self._circuit_breaker.is_open():
    logger.warning(
        f"Circuit breaker OPEN for {command_type}, failing fast",
        extra={"command_type": command_type},
    )
    return False  # Fail fast, trigger retry
```

**States**:

```
┌─────────┐  Threshold     ┌─────────┐
│ CLOSED  │──────────────►│  OPEN   │
│(Normal) │   Exceeded     │(Failing)│
└────┬────┘                └────┬────┘
     ▲                          │
     │                          │ Timeout
     │                          │ Elapsed
     │                          ▼
     │                     ┌─────────┐
     │  Success            │  HALF-  │
     └─────────────────────┤  OPEN   │
       Calls               │(Testing)│
                           └─────────┘
```

**Configuration**:

```yaml
processing:
  circuit_breaker_failure_threshold: 5  # Failures before opening
  circuit_breaker_timeout_seconds: 60  # Time before testing recovery
  circuit_breaker_half_open_max_calls: 3  # Test calls in half-open
```

### Dead-Letter Handling

**Dead-Letter Queue Investigation**:

```python
# scripts/investigate_dlq.py
from azure.servicebus import ServiceBusClient

async def investigate_dead_letters(queue_name: str):
    """Retrieve and analyze dead-lettered commands."""

    async with ServiceBusClient.from_connection_string(
        connection_string
    ) as client:
        receiver = client.get_queue_receiver(
            queue_name=queue_name,
            sub_queue=ServiceBusSubQueue.DEAD_LETTER
        )

        async with receiver:
            messages = await receiver.receive_messages(max_message_count=10)

            for message in messages:
                print(f"Command Type: {message.application_properties.get('type')}")
                print(f"Dead-letter Reason: {message.dead_letter_reason}")
                print(f"Dead-letter Description: {message.dead_letter_error_description}")
                print(f"Delivery Count: {message.delivery_count}")
                print(f"Enqueued Time: {message.enqueued_time_utc}")
                print(f"Body: {message.body}")
                print("---")
```

**Manual Reprocessing**:

```python
# scripts/reprocess_dlq.py
async def reprocess_dead_letters(queue_name: str, message_ids: list[str]):
    """Move dead-lettered messages back to active queue for reprocessing."""

    async with ServiceBusClient.from_connection_string(
        connection_string
    ) as client:
        dlq_receiver = client.get_queue_receiver(
            queue_name=queue_name,
            sub_queue=ServiceBusSubQueue.DEAD_LETTER
        )
        sender = client.get_queue_sender(queue_name=queue_name)

        async with dlq_receiver, sender:
            messages = await dlq_receiver.receive_messages(max_message_count=100)

            for message in messages:
                if message.message_id in message_ids:
                    # Recreate message without dead-letter metadata
                    new_message = ServiceBusMessage(
                        body=message.body,
                        application_properties=message.application_properties,
                    )
                    await sender.send_messages(new_message)
                    await dlq_receiver.complete_message(message)

                    print(f"Reprocessed: {message.message_id}")
```

---

## Correlation Context

### Correlation ID Propagation

**Command-Level Correlation**:

```python
class BaseCommand(BaseModel):
    """Base command with correlation context."""

    correlation_id: UUID
    causation_id: Optional[UUID] = None  # Command that caused this command
    tenant_id: UUID
    user_id: Optional[UUID] = None

# In handler
@command_router.subscribe(command_type=CommandType.GENERATE_DOCUMENT_THUMBNAIL)
async def handle_generate_thumbnail(
    command_data: dict,
    message: ServiceBusReceivedMessage,
) -> bool:
    command = GenerateThumbnailCommand.model_validate(command_data)

    # Set correlation context for downstream operations
    set_correlation_id(command.correlation_id)
    set_tenant_id(command.tenant_id)

    # All logs/events will include correlation_id
    logger.info(
        "Processing thumbnail command",
        extra={"correlation_id": str(command.correlation_id)}
    )
```

**Logging Integration**:

```python
# All logs automatically include correlation context
logger.info(
    "Thumbnail generated successfully",
    extra={
        "correlation_id": str(get_correlation_id()),
        "tenant_id": str(get_tenant_id()),
        "command_type": "generate_document_thumbnail",
    }
)
```

---

## Testing Command Handlers

### Unit Testing Handlers

```python
# tests/unit/controller/commands/test_document_commands.py
import pytest
from unittest.mock import AsyncMock, MagicMock
from azure.servicebus import ServiceBusReceivedMessage

from document_service.controller.commands.v1.document_commands import (
    handle_generate_thumbnail,
)


@pytest.fixture
def mock_message():
    """Create mock Service Bus message."""
    message = MagicMock(spec=ServiceBusReceivedMessage)
    message.message_id = "test-message-id"
    message.correlation_id = "test-correlation-id"
    message.delivery_count = 1
    return message


@pytest.fixture
def mock_thumbnail_service():
    """Create mock ThumbnailService."""
    service = AsyncMock()
    service.generate_thumbnail = AsyncMock(return_value=None)
    return service


async def test_handle_generate_thumbnail_success(
    mock_message,
    mock_thumbnail_service,
):
    """Test successful thumbnail generation command."""
    # Arrange
    command_data = {
        "spec_version": "1.0",
        "type": "generate_document_thumbnail",
        "source": "test-service",
        "id": "command-id",
        "time": "2025-01-01T00:00:00Z",
        "data": {
            "document_id": "550e8400-e29b-41d4-a716-446655440000",
            "size": "medium",
            "priority": 5,
        },
        "correlation_id": "correlation-id",
        "tenant_id": "tenant-id",
    }

    # Act
    result = await handle_generate_thumbnail(
        command_data=command_data,
        message=mock_message,
        service=mock_thumbnail_service,
    )

    # Assert
    assert result is True
    mock_thumbnail_service.generate_thumbnail.assert_called_once()


async def test_handle_generate_thumbnail_validation_error(
    mock_message,
    mock_thumbnail_service,
):
    """Test command with invalid schema."""
    # Arrange
    command_data = {
        "data": {}  # Missing required fields
    }

    # Act
    result = await handle_generate_thumbnail(
        command_data=command_data,
        message=mock_message,
        service=mock_thumbnail_service,
    )

    # Assert
    assert result is False  # Validation failed
    mock_thumbnail_service.generate_thumbnail.assert_not_called()


async def test_handle_generate_thumbnail_service_error(
    mock_message,
    mock_thumbnail_service,
):
    """Test command with service error (should retry)."""
    # Arrange
    command_data = {
        "spec_version": "1.0",
        "type": "generate_document_thumbnail",
        "source": "test-service",
        "id": "command-id",
        "time": "2025-01-01T00:00:00Z",
        "data": {
            "document_id": "550e8400-e29b-41d4-a716-446655440000",
            "size": "medium",
            "priority": 5,
        },
        "correlation_id": "correlation-id",
        "tenant_id": "tenant-id",
    }

    # Simulate service error
    mock_thumbnail_service.generate_thumbnail.side_effect = Exception("Service error")

    # Act
    result = await handle_generate_thumbnail(
        command_data=command_data,
        message=mock_message,
        service=mock_thumbnail_service,
    )

    # Assert
    assert result is False  # Failure should trigger retry
```

### Integration Testing

```python
# tests/integration/commands/test_command_processing.py
import pytest
from azure.servicebus import ServiceBusMessage

from document_service.cwa import main as command_worker_main


@pytest.mark.integration
async def test_command_processing_end_to_end(
    service_bus_client,
    redis_client,
):
    """Test complete command processing flow."""
    # Arrange
    queue_name = "test-document-commands"
    command = {
        "spec_version": "1.0",
        "type": "generate_document_thumbnail",
        "source": "test",
        "id": "test-command-id",
        "time": "2025-01-01T00:00:00Z",
        "data": {
            "document_id": "550e8400-e29b-41d4-a716-446655440000",
            "size": "medium",
            "priority": 5,
        },
        "correlation_id": "test-correlation-id",
        "tenant_id": "test-tenant-id",
    }

    # Act - Send command to queue
    sender = service_bus_client.get_queue_sender(queue_name)
    async with sender:
        message = ServiceBusMessage(
            body=json.dumps(command),
            application_properties={"type": "generate_document_thumbnail"},
        )
        await sender.send_messages(message)

    # Wait for processing
    await asyncio.sleep(5)

    # Assert - Check idempotency key was set
    idempotency_key = f"command:test-command-id"
    is_processed = await redis_client.exists(idempotency_key)
    assert is_processed == 1
```

---

## Monitoring & Observability

### Key Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `commands_processed_total` | Counter | Total commands processed |
| `commands_success_total` | Counter | Successfully processed commands |
| `commands_failed_total` | Counter | Failed commands (before dead-letter) |
| `commands_deadlettered_total` | Counter | Commands sent to dead-letter queue |
| `commands_processing_duration_seconds` | Histogram | Command processing duration |
| `commands_retry_count` | Histogram | Distribution of retry attempts |
| `queue_depth` | Gauge | Current queue depth |
| `circuit_breaker_state` | Gauge | Circuit breaker state (0=closed, 1=open, 2=half-open) |

### Logging Standards

```python
# Structured logging with command context
logger.info(
    "Processing command",
    extra={
        "command_type": "generate_document_thumbnail",
        "command_id": command.id,
        "correlation_id": str(command.correlation_id),
        "tenant_id": str(command.tenant_id),
        "delivery_count": message.delivery_count,
    }
)

# Log retry attempts
logger.warning(
    "Command retry",
    extra={
        "command_type": "generate_document_thumbnail",
        "retry_attempt": message.delivery_count,
        "max_retries": max_retry_attempts,
        "backoff_seconds": backoff_delay,
    }
)

# Log dead-letter events
logger.error(
    "Command dead-lettered",
    extra={
        "command_type": "generate_document_thumbnail",
        "command_id": message.message_id,
        "delivery_count": message.delivery_count,
        "reason": "max_retries_exceeded",
    }
)
```

### Health Checks

```bash
# Command worker health endpoint
curl http://localhost:8089/health

# Response
{
  "status": "healthy",
  "service": "document-command-worker-application",
  "version": "1.0.0",
  "processor": {
    "is_running": true,
    "queues": ["document-commands", "maintenance-commands"],
    "handlers_registered": 5
  }
}
```

---

## Troubleshooting Guide

### Common Issues

#### Commands Not Processing

**Symptoms**: Messages stay in queue, not consumed

**Diagnosis**:
```bash
# Check worker pods
kubectl get pods -l app=document-command-worker

# Check worker logs
kubectl logs -l app=document-command-worker --tail=100

# Check queue depth
az servicebus queue show \
  --resource-group platform-rg \
  --namespace-name platform-bus \
  --name document-commands \
  --query "messageCount"
```

**Causes**:
- Worker pods not running
- Handler registration errors
- DI container initialization failures

#### Commands Stuck in Retry Loop

**Symptoms**: Same messages retrying repeatedly

**Diagnosis**:
```bash
# Check delivery counts
az servicebus queue show \
  --resource-group platform-rg \
  --namespace-name platform-bus \
  --name document-commands \
  --query "deliveryCount"

# Check dead-letter queue
az servicebus queue show \
  --resource-group platform-rg \
  --namespace-name platform-bus \
  --name document-commands/$deadletterqueue \
  --query "messageCount"
```

**Causes**:
- Transient failures not resolving
- Handler always returning `False`
- External service unavailable

**Resolution**:
- Increase `max_retry_attempts` if transient
- Fix handler logic if persistent
- Implement circuit breaker for external services

#### High Dead-Letter Queue Depth

**Symptoms**: Many messages in DLQ

**Diagnosis**:
```python
# Investigate dead-letter messages
python scripts/investigate_dlq.py --queue document-commands

# Check common failure reasons
for msg in dead_letter_messages:
    print(f"Reason: {msg.dead_letter_reason}")
    print(f"Description: {msg.dead_letter_error_description}")
```

**Causes**:
- Schema validation errors
- Business logic errors
- External service failures

**Resolution**:
1. Fix root cause in handler/schema
2. Reprocess valid messages: `python scripts/reprocess_dlq.py`
3. Manually handle invalid messages

---

## Best Practices

### Command Design

✅ **Do**:
- Use commands for tasks requiring exactly-once processing
- Include all necessary data in command payload (avoid lookups)
- Design commands to be idempotent
- Use clear, imperative command names (`generate_thumbnail`, `send_email`)
- Include correlation context for traceability

❌ **Don't**:
- Use commands for broadcasting state changes (use events)
- Include sensitive data without encryption
- Make commands dependent on external state
- Create command chains (use sagas or orchestration)

### Handler Best Practices

✅ **Do**:
- Keep handlers thin, delegate to orchestration services
- Return `True` only on successful completion
- Log important processing steps with correlation context
- Handle validation errors gracefully
- Use structured exception handling

❌ **Don't**:
- Put business logic directly in handlers
- Suppress exceptions without logging
- Return `True` on partial failures
- Make synchronous blocking calls
- Access database directly from handlers

### Configuration Best Practices

✅ **Do**:
- Use dedicated workers for critical, ordered tasks
- Use background workers for batch/maintenance tasks
- Configure retry settings based on failure patterns
- Monitor queue depth and processing times
- Set appropriate `max_concurrent_handlers`

❌ **Don't**:
- Over-provision dedicated workers (cost implications)
- Set retry limits too high (delayed failure detection)
- Mix high/low priority commands in same queue
- Ignore circuit breaker warnings
- Use session-enabled queues for background workers

---

## Summary

The command-queue architecture provides:

- **Point-to-point task distribution** with guaranteed delivery
- **Exponential backoff retry** for transient failure recovery
- **Clean separation** from event-driven pub/sub patterns
- **Developer ergonomics** matching FastAPI style
- **Production resilience** with idempotency, circuit breakers, and dead-letter handling

This pattern complements event-driven architecture by handling tasks that require:
- Exactly-once processing guarantees
- Automatic retry with backoff
- Single consumer semantics
- Long-running operations

For real-time state change notifications and loosely coupled integration, use [Event-Driven Architecture](./event-driven.md) instead.
