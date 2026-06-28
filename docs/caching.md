---
id: caching
title: Caching Strategy
description: Enterprise-grade caching with Cashews - patterns, best practices, and implementation guide
sidebar_label: Caching
---

# CACHING STRATEGY

## Table of Contents

- [Why Cashews?](#why-cashews)
  - [The Enterprise Caching Challenges](#the-enterprise-caching-challenges)
  - [How Cashews Solves Critical Issues](#how-cashews-solves-critical-issues)
- [Core Concepts](#core-concepts)
  - [Tag-Based Invalidation](#tag-based-invalidation)
  - [Request Coalescing (Lock)](#request-coalescing-lock)
  - [Type Safety (Serialization)](#type-safety-serialization)
- [Usage Patterns](#usage-patterns)
  - [Basic Caching](#basic-caching)
  - [Tag-Based Invalidation](#tag-based-invalidation-pattern)
  - [Conditional Caching](#conditional-caching)
  - [Distributed Locking](#distributed-locking)
  - [Idempotency Keys](#idempotency-keys)
  - [Bloom Filters](#bloom-filters)
- [Implementation Guide](#implementation-guide)
  - [Cache Setup](#cache-setup)
  - [Service Integration](#service-integration)
  - [Best Practices](#best-practices)
- [Performance Considerations](#performance-considerations)

---

## Why Cashews?

The platform uses [Cashews](https://github.com/Krukov/cashews) as our caching library instead of implementing custom Redis wrappers. This decision addresses critical enterprise-grade operational risks that typical "home-grown" caching solutions often miss.

### The Enterprise Caching Challenges

When building production-ready caching, teams commonly encounter three critical issues:

#### 1. **The SCAN Anti-Pattern** (O(N) Latency)

**Problem**: Using Redis `SCAN` to find and invalidate cache keys causes "stop the world" performance degradation.

```python
# ❌ DANGEROUS: Scans entire Redis database
async def invalidate_project_cache(project_id: str):
    cursor = 0
    while True:
        cursor, keys = await redis.scan(cursor, match=f"project:{project_id}:*")
        if keys:
            await redis.delete(*keys)
        if cursor == 0:
            break
```

**Impact**:
- O(Total Keys in Redis) complexity
- Blocks Redis server during iteration
- Can cause cascading timeouts across services

#### 2. **The Serialization Trap** (Type Loss)

**Problem**: Using `json.dumps()` causes Pydantic models and domain entities to lose their type information when cached.

```python
# ❌ BROKEN: Returns dict instead of UserEntity
@cache(key="user:{id}")
async def get_user(id: str) -> UserEntity:
    user = await repository.find_by_id(id)
    return user  # Serialized as JSON, retrieved as dict

# Runtime error: dict has no attribute 'is_active'
user = await get_user("123")
user.is_active()  # ❌ AttributeError
```

**Impact**:
- Violates Liskov Substitution Principle
- Forces defensive type checks throughout codebase
- Hidden bugs that only appear with cache hits

#### 3. **Thundering Herd** (Cache Stampede)

**Problem**: When a popular cache key expires, hundreds of requests simultaneously hit the database.

```python
# ❌ STAMPEDE: 500 concurrent requests all query DB
@cache(key="project:{id}", ttl="1h")
async def get_project(id: str) -> Project:
    return await db.query(Project).filter_by(id=id).first()
```

**Impact**:
- Database connection pool exhaustion
- 500× redundant queries for identical data
- Cascading failures during traffic spikes

---

### How Cashews Solves Critical Issues

Cashews was architected specifically for these Level 2 enterprise concerns. Here's the alignment:

| Challenge | Naive Approach | Cashews Solution | Complexity |
|-----------|---------------|------------------|------------|
| **Invalidation** | `SCAN` all keys | Redis Sets (index lookup) | O(Keys per Tag) vs O(Total Keys) |
| **Type Safety** | `json.dumps` (loses types) | Pickle (preserves objects) | Same types in/out |
| **Thundering Herd** | All requests query DB | Request coalescing with lock | 1 DB query for N requests |

#### Solution 1: Tag-Based Invalidation (No SCAN)

Cashews uses **Redis Sets as indexes** to track which keys belong to each tag:

```python
# When caching with tags
@cache(ttl="1h", key="project:{id}", tags=["project:{id}", "projects:all"])
async def get_project(id: str) -> Project:
    return await repository.find_by_id(id)

# Internally, Cashews stores:
# Key: "project:123" → <cached data>
# Set: "tag:project:123" → {"project:123"}
# Set: "tag:projects:all" → {"project:123", "project:456", ...}
```

**Invalidation Flow** (O(Keys per Tag)):
1. `await cache.delete_tags("project:123")` fetches Set members (O(1))
2. Pipelines deletion for those specific keys (O(Keys in Tag))
3. Deletes the Set itself (O(1))

**Result**: No SCAN. Targeted O(N) where N = keys with that tag (typically 10-100), not total Redis keys.

#### Solution 2: Pickle Serialization (Type Preservation)

Cashews defaults to **Pickle** instead of JSON:

```python
@cache(ttl="1h", key="user:{id}")
async def get_user(id: str) -> UserEntity:
    user = await repository.find_by_id(id)
    return user  # Pickled as Python object

# Retrieved as exact UserEntity instance
user = await get_user("123")
user.is_active()  # ✅ Works! Same type as DB entity
```

**Result**: Cache hits and cache misses return identical types. No defensive `isinstance()` checks needed.

#### Solution 3: Request Coalescing (Thundering Herd Protection)

Cashews provides native `lock=True` parameter:

```python
@cache(ttl="1h", key="project:{id}", lock=True)
async def get_project(id: str) -> Project:
    return await expensive_db_query(id)

# On cache miss with 500 concurrent requests:
# Request 1: Acquires lock, queries DB, caches result, releases lock
# Requests 2-500: Wait for lock, then immediately read cached value (no DB hit)
```

**Result**: 1 DB query instead of 500. The lock is key-specific—other project IDs aren't blocked.

---

## Core Concepts

### Tag-Based Invalidation

Tags create logical groups of cache keys for efficient bulk invalidation.

**Mental Model**: Tags are like "database indexes" for your cache. When you update a project, invalidate all caches related to that project without scanning the entire Redis database.

```python
# Read Operation: Tag cached data
@cache(
    ttl="1h",
    key="deliverable:{id}",
    tags=["deliverable:{id}", "project:{project_id}", "deliverables:all"]
)
async def get_deliverable(id: str, project_id: str) -> DeliverableEntity:
    return await self.repository.find_by_id(id)

# Write Operation: Invalidate related tags
@cache.invalidate_tags("deliverable:{id}", "project:{project_id}", "deliverables:all")
async def update_deliverable(id: str, data: UpdateDeliverableDTO) -> DeliverableEntity:
    deliverable = await self.repository.update(id, data)
    return deliverable
```

**Best Practice**: Use hierarchical tags—from specific to general:
1. Single entity: `deliverable:123`
2. Parent scope: `project:456`
3. Global list: `deliverables:all`

### Request Coalescing (Lock)

The `lock=True` parameter ensures only one request executes the expensive operation when a cache key is missing.

```python
@cache(ttl="1h", key="report:{id}", lock=True)
async def generate_report(id: str) -> ReportEntity:
    # Only one request computes this, even with 1000 concurrent calls
    return await self.expensive_report_generation(id)
```

**When to use**:
- ✅ Expensive DB queries (joins across multiple tables)
- ✅ External API calls with rate limits
- ✅ Heavy computations (aggregations, reports)
- ❌ Simple key-value lookups (overhead not worth it)

### Type Safety (Serialization)

Cashews preserves Python object types across cache round-trips.

```python
# Domain entity with methods
class ProjectEntity:
    id: str
    name: str

    def is_active(self) -> bool:
        return self.status == "active"

# Caching preserves the entity class
@cache(ttl="1h", key="project:{id}")
async def get_project(id: str) -> ProjectEntity:
    return await self.repository.find_by_id(id)

# Both cache hit and miss return ProjectEntity
project = await get_project("123")
project.is_active()  # ✅ Method available
```

**Result**: Cached objects behave identically to fresh database objects. No surprise `dict` instead of entity.

---

## Usage Patterns

### Basic Caching

Cache read operations with simple TTL:

```python
from platform_infrastructure.caching import cache
from platform_infrastructure.caching.cache_keys import UserCacheKeys
from platform_infrastructure.caching.cache_ttl import CacheTTL

class UserService:
    @cache(ttl=CacheTTL.USER_PROFILE, key=UserCacheKeys.USER_BY_ID)
    async def get_user(self, id: str) -> UserEntity:
        """Cache user for 1 hour."""
        return await self.repository.find_by_id(id)
```

**Parameters**:
- `ttl`: Time-to-live constant (e.g., `CacheTTL.USER_PROFILE`, `CacheTTL.PROJECT_METADATA`)
- `key`: Cache key constant (use `{param}` for dynamic values)

**Why Constants?**: Centralized key and TTL management prevents typos, enables refactoring, and documents caching strategy.

### Tag-Based Invalidation Pattern

**Step 1**: Add tags to read operations

```python
from platform_infrastructure.caching.cache_keys import ProjectCacheKeys
from platform_infrastructure.caching.cache_tags import ProjectCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

class ProjectService:
    @cache(
        ttl=CacheTTL.PROJECT_METADATA,
        key=ProjectCacheKeys.PROJECT_BY_ID,
        tags=[ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL]
    )
    async def get_project(self, project_id: str) -> ProjectEntity:
        return await self.repository.find_by_id(project_id)

    @cache(
        ttl=CacheTTL.PROJECT_LIST,
        key=ProjectCacheKeys.PROJECTS_BY_COMPANY,
        tags=[ProjectCacheTags.COMPANY_PROJECTS, ProjectCacheTags.PROJECTS_ALL]
    )
    async def list_projects(self, company_id: str) -> list[ProjectEntity]:
        return await self.repository.find_by_company(company_id)
```

**Step 2**: Invalidate tags on write operations

```python
    @cache.invalidate_tags(ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL)
    async def update_project(
        self, project_id: str, data: UpdateProjectDTO
    ) -> ProjectEntity:
        project = await self.repository.update(project_id, data)
        return ProjectMapper.to_dto(project)

    @cache.invalidate_tags(ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL)
    async def delete_project(self, project_id: str) -> None:
        await self.repository.delete(project_id)
```

**Result**: Updating `project:123` automatically invalidates:
- Single project cache: `project:123`
- All projects list: `projects:all`

### Conditional Caching

Only cache when specific conditions are met:

```python
from platform_infrastructure.caching.cache_condition import CacheCondition
from platform_infrastructure.caching.cache_keys import DocumentCacheKeys
from platform_infrastructure.caching.cache_tags import DocumentCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

class DocumentService:
    @cache(
        ttl=CacheTTL.DOCUMENT_LIST,
        key=DocumentCacheKeys.DOCUMENTS_BY_DELIVERABLE,
        tags=[DocumentCacheTags.DELIVERABLE_DOCUMENTS],
        condition=CacheCondition.for_paginated  # Only cache default queries
    )
    async def list_documents(
        self,
        deliverable_id: str,
        q: Optional[str] = None,
        filter_by: Optional[dict] = None,
        sort_by: Optional[str] = None,
    ) -> list[DocumentEntity]:
        return await self.repository.find_all(deliverable_id, q, filter_by, sort_by)
```

**Condition Logic**: `CacheCondition.for_paginated` returns:
- `True` if `q=None`, `filter_by=None`, `sort_by=None` (cache default list)
- `False` otherwise (skip caching search/filter queries)

**Why?**: Filtered queries have low cache hit rates. Only cache high-traffic default queries.

### Conditional Invalidation

Skip invalidation when certain conditions aren't met:

```python
from platform_infrastructure.caching.cache_tags import NotificationCacheTags

class NotificationService:
    @cache.invalidate_tags(
        NotificationCacheTags.USER_NOTIFICATIONS,
        condition="not_none"  # Only invalidate if result is not None
    )
    async def mark_as_read(
        self, notification_id: str, user_id: str
    ) -> Optional[NotificationEntity]:
        # Returns None if notification doesn't exist
        notification = await self.repository.find_by_id(notification_id)
        if not notification:
            return None

        notification.read_at = datetime.utcnow()
        await self.repository.update(notification)
        return notification
```

**Custom Conditions**: Define your own condition functions:

```python
from platform_infrastructure.caching.cache_wrapper import ConditionFunc
from platform_infrastructure.caching.cache_tags import DocumentCacheTags

def only_if_published(
    result: DocumentEntity,
    args: tuple,
    kwargs: dict,
    key: Optional[str]
) -> bool:
    """Only invalidate cache if document is published."""
    return result.status == "published"

@cache.invalidate_tags(DocumentCacheTags.DOCUMENTS_ALL, condition=only_if_published)
async def update_document(id: str, data: UpdateDTO) -> DocumentEntity:
    return await self.repository.update(id, data)
```

### Distributed Locking

Use Redis as a distributed lock for critical sections:

```python
from platform_infrastructure.caching import cache

class PaymentService:
    async def process_payment(self, order_id: str, amount: Decimal) -> PaymentEntity:
        # Ensure only one process handles this payment
        async with cache.lock(f"payment:lock:{order_id}", timeout=30):
            # Check idempotency
            existing = await self.repository.find_by_order(order_id)
            if existing:
                return existing

            # Process payment
            payment = await self.external_payment_api.charge(amount)
            return await self.repository.save(payment)
```

**Parameters**:
- `timeout`: Lock TTL in seconds (fail-safe if holder crashes)
- Auto-releases when context exits

**Use Cases**:
- Payment processing (avoid double charges)
- File uploads (prevent duplicate processing)
- Critical sections across distributed workers

### Idempotency Keys

Implement idempotent endpoints using cache as temporary state:

```python
from platform_infrastructure.caching.cache_keys import IdempotencyCacheKeys
from platform_infrastructure.caching.cache_ttl import CacheTTL

class DocumentService:
    async def upload_document(
        self,
        request: UploadDocumentDTO,
        file: bytes,
        idempotency_key: Optional[str] = None,
    ) -> DocumentEntity:
        # Check if already processed
        if idempotency_key:
            key = IdempotencyCacheKeys.IDEMPOTENCY_KEY.format(idempotency_key=idempotency_key)
            cached = await cache.get(key)
            if cached:
                return cached  # Return previous result

        # Process upload
        document = await self._process_upload(request, file)

        # Store for idempotency (24 hour window)
        if idempotency_key:
            key = IdempotencyCacheKeys.IDEMPOTENCY_KEY.format(idempotency_key=idempotency_key)
            await cache.set(key, document, ttl=CacheTTL.IDEMPOTENCY_WINDOW)

        return document
```

**API Usage**:
```bash
# Client sends idempotency key in header
POST /api/v1/documents/upload
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

# If network fails and client retries with same key:
# Returns cached result instead of creating duplicate
```

### Bloom Filters

Use Cashews bloom filters for probabilistic existence checks:

```python
from platform_infrastructure.caching import cache

class EmailService:
    async def is_email_used(self, email: str) -> bool:
        """Fast check if email exists (with small false positive rate)."""
        # Check bloom filter first (in-memory, microseconds)
        if not await cache.bloom.is_contains("emails:bloom", email):
            return False  # Definitely not registered

        # Might be registered, verify with DB (only on potential matches)
        return await self.repository.exists_by_email(email)

    async def register_user(self, email: str) -> UserEntity:
        user = await self.repository.create(email)

        # Add to bloom filter
        await cache.bloom.add("emails:bloom", email)
        return user
```

**Benefits**:
- O(1) membership test (vs O(log N) database index)
- Reduces 99%+ database queries for non-existent items
- Useful for uniqueness checks, spam filtering

**Trade-off**: ~1% false positive rate (says "exists" when it doesn't). Always verify positives with DB.

---

## Implementation Guide

### Cache Setup

The cache is initialized during service startup via the global singleton:

```python
# In shared/infrastructure/platform_infrastructure/caching/cache_manager.py
from platform_infrastructure.caching import cache
from platform_infrastructure.config import CacheSettings

def _lazy_init_cache(settings: CacheSettings, service_prefix: str) -> None:
    """Initialize cache with service-specific namespace."""
    cache.setup(
        settings.redis_url,
        secret=settings.secret,
        pickle_type=settings.pickle_type,
        suppress=settings.suppress_errors,
        enable=settings.enabled,
        client_side_prefix=service_prefix,  # e.g., "document:", "admin:"
    )
```

**Service Prefixes**: Each microservice gets its own namespace:
- `document:` for document-service
- `admin:` for admin-service
- `identity:` for identity-service

This prevents key collisions across services.

### Service Integration

**Step 1**: Import cache singleton

```python
from platform_infrastructure.caching import cache
```

**Step 2**: Add caching to orchestration services

```python
from platform_infrastructure.caching.cache_keys import ProjectCacheKeys
from platform_infrastructure.caching.cache_tags import ProjectCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

class ProjectService:
    def __init__(
        self,
        repository: IProjectRepository,
        uow: UnitOfWork,
    ):
        self.repository = repository
        self.uow = uow

    @cache(
        ttl=CacheTTL.PROJECT_METADATA,
        key=ProjectCacheKeys.PROJECT_BY_ID,
        tags=[ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL],
        lock=True
    )
    async def get_project(self, project_id: str) -> ResponseProjectDTO:
        async with self.uow:
            project = await self.repository.find_by_id(project_id, session=self.uow.session)
            if not project:
                raise ProjectNotFoundError(project_id)
            return ProjectMapper.to_dto(project)

    @cache.invalidate_tags(ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL)
    async def update_project(
        self, project_id: str, data: UpdateProjectDTO
    ) -> ResponseProjectDTO:
        async with self.uow:
            project = await self.repository.find_by_id(project_id, session=self.uow.session)
            if not project:
                raise ProjectNotFoundError(project_id)

            # Update entity
            project.name = data.name
            project.description = data.description

            updated = await self.repository.update(project, session=self.uow.session)
            return ProjectMapper.to_dto(updated)
```

**Step 3**: Define reusable cache keys, tags, and TTL constants

```python
# In shared/infrastructure/platform_infrastructure/caching/cache_keys.py
class ProjectCacheKeys:
    """Cache keys for project-related operations."""
    PROJECT_BY_ID = "project:{project_id}"
    PROJECTS_BY_COMPANY = "projects:company:{company_id}"
    PROJECTS_BY_EVENT = "projects:event:{event_id}"

# In shared/infrastructure/platform_infrastructure/caching/cache_tags.py
class ProjectCacheTags:
    """Cache tags for project-related invalidation."""
    PROJECT_BY_ID = "project:{project_id}"
    COMPANY_PROJECTS = "company:{company_id}:projects"
    EVENT_PROJECTS = "event:{event_id}:projects"
    PROJECTS_ALL = "projects:all"

# In shared/infrastructure/platform_infrastructure/caching/cache_ttl.py
class CacheTTL:
    """Standardized TTL values across the application."""
    # User-related
    USER_PROFILE = "1h"
    USER_PERMISSIONS = "30m"

    # Project-related
    PROJECT_METADATA = "1h"
    PROJECT_LIST = "30m"

    # Document-related
    DOCUMENT_METADATA = "30m"
    DOCUMENT_LIST = "15m"

    # System
    CONFIGURATION = "1d"
    IDEMPOTENCY_WINDOW = "24h"

# Usage in service
from platform_infrastructure.caching.cache_keys import ProjectCacheKeys
from platform_infrastructure.caching.cache_tags import ProjectCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

@cache(
    ttl=CacheTTL.PROJECT_METADATA,
    key=ProjectCacheKeys.PROJECT_BY_ID,
    tags=[ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL]
)
async def get_project(self, project_id: str):
    ...
```

### Best Practices

#### 1. **Cache at Orchestration Layer, Not Controller**

❌ **Wrong**: Caching in controllers couples HTTP concerns with caching logic

```python
# controller/api/v1/projects.py
@router.get("/{project_id}")
@cache(ttl="1h", key="project:{project_id}")  # ❌ Wrong layer
async def get_project(project_id: str):
    ...
```

✅ **Correct**: Cache in orchestration services

```python
# orchestration/project/project_service.py
class ProjectService:
    @cache(ttl="1h", key="project:{project_id}")  # ✅ Correct layer
    async def get_project(self, project_id: str):
        ...
```

**Why?**: Orchestration services are reused by multiple consumers (API, CLI, background jobs). Caching here benefits all consumers.

#### 2. **Use Hierarchical Tags**

Structure tags from specific to general:

```python
from platform_infrastructure.caching.cache_keys import DocumentCacheKeys
from platform_infrastructure.caching.cache_tags import DocumentCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

@cache(
    ttl=CacheTTL.DOCUMENT_METADATA,
    key=DocumentCacheKeys.DOCUMENT_BY_ID,
    tags=[
        DocumentCacheTags.DOCUMENT_BY_ID,           # Specific: single document
        DocumentCacheTags.DELIVERABLE_DOCUMENTS,    # Scope: parent container
        DocumentCacheTags.PROJECT_DOCUMENTS,        # Broader: project level
        DocumentCacheTags.DOCUMENTS_ALL             # Global: all documents
    ]
)
```

**Invalidation Strategies**:
- Update document → invalidate `document:{id}` only
- Delete deliverable → invalidate `deliverable:{id}` (cascades to all documents)
- Archive project → invalidate `project:{id}` (cascades to all children)

#### 3. **TTL Guidelines**

| Data Type | Recommended TTL | Reasoning |
|-----------|----------------|-----------|
| User profile | 1-2 hours | Changes infrequently |
| Project metadata | 30-60 minutes | Updated occasionally |
| Document list | 15-30 minutes | Moderate write frequency |
| Real-time stats | 1-5 minutes | Frequently updated |
| Configuration | 1 day | Rarely changes |

**Golden Rule**: Longer TTL for read-heavy, stable data. Shorter TTL for write-heavy, volatile data.

#### 4. **Conditional Caching for Search**

Don't cache filtered/searched queries—low hit rate:

```python
from platform_infrastructure.caching.cache_keys import DocumentCacheKeys
from platform_infrastructure.caching.cache_tags import DocumentCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL
from platform_infrastructure.caching.cache_condition import CacheCondition

@cache(
    ttl=CacheTTL.DOCUMENT_LIST,
    key=DocumentCacheKeys.DOCUMENTS_BY_DELIVERABLE,
    tags=[DocumentCacheTags.DELIVERABLE_DOCUMENTS],
    condition=CacheCondition.for_paginated  # Only cache default list
)
async def list_documents(
    self,
    deliverable_id: str,
    q: Optional[str] = None,          # Search query
    filter_by: Optional[dict] = None,  # Filters
):
    # Only cached when q=None, filter_by=None (default list)
    ...
```

#### 5. **Invalidation Timing**

Invalidate **after** successful database commit:

```python
from platform_infrastructure.caching.cache_tags import ProjectCacheTags

@cache.invalidate_tags(ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL)
async def update_project(self, project_id: str, data: UpdateDTO):
    async with self.uow:
        project = await self.repository.find_by_id(project_id, self.uow.session)
        project.name = data.name
        await self.repository.update(project, self.uow.session)
        # UnitOfWork commits here
    # Cache invalidated after commit (decorator runs after return)
    return ProjectMapper.to_dto(project)
```

**Why?**: If database commit fails, cache isn't invalidated. Maintains consistency.

#### 6. **Error Handling**

Cashews is configured with `suppress=True` by default:

```python
cache.setup(
    settings.redis_url,
    suppress=True,  # Cache failures don't fail requests
)
```

**Behavior**:
- Cache read failure → Falls through to database
- Cache write failure → Logged but request succeeds
- Invalidation failure → Logged as error (cache eventually consistent)

**Result**: Cache is a performance optimization, not a critical dependency. Service degrades gracefully without Redis.

#### 7. **Testing with Cache**

Mock the cache in unit tests:

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.fixture
def mock_cache():
    with patch("platform_infrastructure.caching.cache") as mock:
        mock.get = AsyncMock(return_value=None)  # Simulate cache miss
        mock.set = AsyncMock()
        mock.delete_tags = AsyncMock()
        yield mock

async def test_get_project_caches_result(mock_cache):
    service = ProjectService(repository=mock_repo)

    project = await service.get_project("123")

    # Verify cache was checked
    mock_cache.get.assert_called_once()
```

For integration tests, use real Redis with test data isolation:

```python
@pytest.fixture(autouse=True)
async def clear_cache():
    """Clear cache before each test."""
    await cache.clear()
    yield
    await cache.clear()
```

---

## Performance Considerations

### Cache Key Cardinality

**Problem**: Each unique key consumes Redis memory. Avoid unbounded cardinality:

❌ **High Cardinality** (avoid):
```python
# Creates millions of keys with different timestamps
@cache(ttl="1h", key="report:{user_id}:{timestamp}")
async def generate_report(user_id: str, timestamp: datetime):
    ...
```

✅ **Controlled Cardinality**:
```python
# Creates one key per user (bounded by user count)
@cache(ttl="1h", key="report:{user_id}")
async def generate_report(user_id: str):
    ...
```

### Memory Usage

Monitor Redis memory with typical data sizes:

| Data Type | Avg Size | 10K Cached Items |
|-----------|----------|------------------|
| Simple DTO | 1-5 KB | 10-50 MB |
| List of 100 items | 100-500 KB | 1-5 GB |
| Large document | 1-10 MB | 10-100 GB |

**Guideline**: Keep cached values < 100 KB for optimal performance. For larger data, cache references/IDs instead.

### Tag Set Size

Each tag maintains a Redis Set of key references:

```python
# Tag "projects:all" contains thousands of keys
tags=["project:{id}", "projects:all"]  # "projects:all" grows unbounded
```

**Impact**: Invalidating `projects:all` becomes O(N) where N = total projects. For large N (>10K), consider:
- Shorter TTL instead of invalidation
- More specific tags (e.g., `projects:company:{id}`)
- Versioning strategy (increment counter instead of deleting keys)

### Lock Contention

`lock=True` serializes requests for the same key:

```python
@cache(ttl="1h", key="global:stats", lock=True)
async def get_global_stats():  # Single key for all users
    return await expensive_aggregation()
```

**Problem**: All users wait for the same lock. Consider:
- User-specific keys: `stats:{user_id}`
- Stale-while-revalidate pattern
- Background refresh jobs

---

## Summary

Cashews provides enterprise-grade caching with minimal code:

```python
from platform_infrastructure.caching import cache
from platform_infrastructure.caching.cache_keys import ProjectCacheKeys
from platform_infrastructure.caching.cache_tags import ProjectCacheTags
from platform_infrastructure.caching.cache_ttl import CacheTTL

class ProjectService:
    # ✅ Read: Cache with tags and lock
    @cache(
        ttl=CacheTTL.PROJECT_METADATA,
        key=ProjectCacheKeys.PROJECT_BY_ID,
        tags=[ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL],
        lock=True
    )
    async def get_project(self, id: str):
        return await self.repository.find_by_id(id)

    # ✅ Write: Invalidate tags
    @cache.invalidate_tags(ProjectCacheTags.PROJECT_BY_ID, ProjectCacheTags.PROJECTS_ALL)
    async def update_project(self, id: str, data: UpdateDTO):
        return await self.repository.update(id, data)
```

**Key Takeaways**:
1. **No SCAN**: Tag-based invalidation uses O(Keys per Tag) instead of O(Total Keys)
2. **Type Safety**: Pickle preserves Pydantic models and domain entities
3. **Thundering Herd**: `lock=True` prevents cache stampedes
4. **Graceful Degradation**: Cache failures don't fail requests

This architecture ensures your caching layer scales with your application while maintaining consistency and performance.
