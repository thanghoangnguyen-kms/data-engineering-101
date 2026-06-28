---
id: clean-architecture
title: Clean Architecture
description: Comprehensive guide to the platform's Clean Architecture implementation with layered components and design principles
sidebar_label: Clean Architecture
---

# CLEAN ARCHITECTURE

## Table of Contents

- [Clean Architecture Layer Components](#clean-architecture-layer-components)
  - [1. Controller Layer (API Adapters)](#1-controller-layer-api-adapters)
    - [API Controllers](#api-controllers)
    - [Middleware](#middleware)
    - [Dependencies](#dependencies)
    - [Response Schemas](#response-schemas)
  - [2. Orchestration Layer (Use Case Orchestration)](#2-orchestration-layer-use-case-orchestration)
    - [Orchestration Services](#orchestration-services)
    - [DTO Mappers](#dto-mappers)
    - [Use Cases](#use-cases)
    - [DTOs](#dtos)
  - [3. Domain Layer (Core Business Logic)](#3-domain-layer-core-business-logic)
    - [Business Contexts Structure](#business-contexts-structure)
    - [Entities](#entities)
    - [Types](#types)
    - [Domain Logic](#domain-logic)
    - [Repository Interfaces](#repository-interfaces)
  - [4. Infrastructure Layer (External Concerns)](#4-infrastructure-layer-external-concerns)
    - [Repository Implementations](#repository-implementations)
    - [External Services](#external-services)
    - [Messaging](#messaging)
    - [Configuration & Database](#configuration--database)
    - [Monitoring](#monitoring)
- [Key Architectural Principles](#key-architectural-principles)
  - [Dependency Rule](#dependency-rule)
  - [Business Context Organization](#business-context-organization)
  - [Orchestration Service Pattern](#orchestration-service-pattern)
  - [Multi-Tenant Architecture](#multi-tenant-architecture)
  - [Dependency Injection](#dependency-injection)
  - [Clean Separation](#clean-separation)

## Clean Architecture Layer Components

### **1. Controller Layer** (API Adapters)

#### **API Controllers** (`controller/api/v1/`)

**Definition**: FastAPI route handlers organized by resource that inject orchestration services.

**Example** (`auth.py`):

```python
@router.post(
    "/login",
    response_model=SuccessResponse[ResponseAuthDTO],
    status_code=status.HTTP_200_OK,
    summary="User Login",
)
async def login(
    request: RequestLoginDTO,
    req: Request,
    auth_service: AuthService = Depends(get_auth_service),
) -> SuccessResponse[ResponseAuthDTO]:
    """Authenticate user and return access token."""
    result = await auth_service.login(request)

    return ApiResponse.success(
        data=result,
        correlation_id=str(getattr(req.state, "correlation_id", "")),
        request_id=str(getattr(req.state, "request_id", "")),
    )
```

**Why?**: Provides HTTP API interface while delegating all business orchestration to orchestration services. Controllers remain thin and focused only on HTTP protocol concerns.

#### **Middleware** (`controller/middleware/`)

**Definition**: Cross-cutting concerns applied to HTTP requests/responses.

**Components**:
- `auth_middleware.py`: JWT token validation and user context
- `cors_middleware.py`: Cross-origin resource sharing
- `error_handler_middleware.py`: Exception handling and error responses
- `rate_limit_middleware.py`: Request throttling
- `response_middleware.py`: Response formatting and headers

**Why?**: Cross-cutting concerns should be applied consistently across all endpoints without code duplication.

#### **Dependencies** (`controller/dependencies/`)

**Definition**: FastAPI dependency injection functions.

**Examples**:
- `auth_dependencies.py`: Authentication and authorization utilities
- `service_dependencies.py`: Service DI providers

#### **Response Schemas** (`controller/responses/`)

**Definition**: Standardized API response formats.

**Components**:
- `api_response.py`: Success response wrapper
- `error.py`: Error response formats
- `pagination.py`: Paginated response structure
- `success.py`: Success response utilities

**Why?**: Ensures consistent API responses and error handling across the entire application.

### **2. Orchestration Layer** (Use Case Orchestration)

#### **Orchestration Services** (`orchestration/{context}/`)

**Definition**: Services that handle complete use case execution: DTO conversion, business logic coordination, and repository interaction. Each service is organized by business context.

**Directory Structure**:

```
orchestration/
├── auth/
│   ├── auth_service.py       # Main orchestration service
│   ├── auth_mapper.py        # DTO <-> Entity conversion
│   ├── dto/
│   │   └── auth_dto.py       # Request/Response DTOs
│   └── usecases/
│       └── update_auth_usecase.py  # Complex use case operations
├── user/
│   ├── user_service.py
│   ├── user_mapper.py
│   ├── dto/
│   └── usecases/
└── organization/
    ├── organization_service.py
    ├── organization_mapper.py
    ├── dto/
    └── usecases/
```

**Example** (`{context}_service.py`):

```python
class ResourceService:
    """
    Orchestration service for resource operations.
    Handles complete use case execution: DTO conversion, business logic,
    and repository interaction.
    """

    def __init__(self, resource_repository: ResourceRepository):
        self._resource_repository = resource_repository

    async def create(self, request: RequestCreateResourceDTO) -> ResponseResourceDTO:
        """
        Create resource.
        Orchestrates: DTO conversion → Business logic → Data persistence → Response DTO
        """
        # 1. Convert DTO to domain entity
        entity = ResourceMapper.to_entity(request)

        # 2. Apply business logic and validation
        ResourceLogic.validate_for_creation(entity)

        # 3. Persist via repository
        saved = await self._resource_repository.save(entity)

        # 4. Convert back to response DTO
        return ResourceMapper.to_response_dto(saved)

    async def update(self, resource_id: str, request: RequestUpdateResourceDTO) -> ResponseResourceDTO:
        """Update resource."""
        # Get existing
        existing = await self._resource_repository.find_by_id(resource_id)
        if not existing:
            raise EntityNotFoundError(f"Resource {resource_id} not found")

        # Business logic validation
        ResourceLogic.validate_for_update(existing, request)

        # Apply updates
        updated = ResourceMapper.apply_updates(existing, request)
        result = await self._resource_repository.save(updated)

        return ResourceMapper.to_response_dto(result)

    async def list(
        self, skip: int = 0, limit: Optional[int] = None
    ) -> ResponseResourceListDTO:
        """List resources."""
        records = await self._resource_repository.find_all(skip=skip, limit=limit)
        total = await self._resource_repository.count_all()

        dtos = [ResourceMapper.to_response_dto(record) for record in records]

        return ResponseResourceListDTO(
            data=dtos,
            total=total,
            skip=skip,
            limit=limit,
        )
```

**Why?**: Consolidates the complete use case workflow in one place. Each orchestration service:
- Accepts request DTOs from controllers
- Converts DTOs to domain entities
- Calls domain logic for business rules
- Interacts with repositories for data access
- Returns response DTOs

#### **DTO Mappers** (`orchestration/{context}/{context}_mapper.py`)

**Definition**: Utility classes for bidirectional conversion between domain entities and DTOs. Centralizes all transformation logic.

**Example** (`{context}_mapper.py`):

```python
class ResourceMapper:
    """
    Handles conversion between Resource domain entities and DTOs.
    Centralizes all entity-DTO mapping logic.
    """

    @staticmethod
    def to_entity(request: RequestCreateResourceDTO) -> Resource:
        """Convert create DTO to Resource entity."""
        return Resource(
            name=request.name,
            description=request.description,
            is_active=True,
            created_at=datetime.now(UTC),
            updated_at=datetime.now(UTC),
        )

    @staticmethod
    def to_response_dto(entity: Resource) -> ResponseResourceDTO:
        """Convert Resource entity to response DTO."""
        return ResponseResourceDTO(
            id=str(entity.id),
            name=entity.name,
            description=entity.description,
            is_active=entity.is_active,
            created_at=entity.created_at,
            updated_at=entity.updated_at,
        )

    @staticmethod
    def apply_updates(existing: Resource, request: RequestUpdateResourceDTO) -> Resource:
        """Apply updates from DTO to entity."""
        if request.name is not None:
            existing.name = request.name
        if request.description is not None:
            existing.description = request.description
        if request.is_active is not None:
            existing.is_active = request.is_active

        existing.updated_at = datetime.now(UTC)
        return existing
```

**Why?**: Centralizes entity-DTO conversion logic, making it reusable, testable, and easy to maintain. Prevents conversion logic scattered throughout the codebase.

#### **Use Cases** (`orchestration/{context}/usecases/`)

**Definition**: Optional, for complex multi-step operations that don't fit cleanly into a single orchestration service method.

**Characteristics**:
- Handle complex workflows spanning multiple entities
- Contain business logic for specific use cases
- Called by orchestration services
- Can coordinate with multiple repositories

**Example** (`{context}_usecase.py`):

```python
class BulkImportUseCase:
    """
    Complex use case for bulk importing resources.
    Handles: validation, creation, error handling, notifications.
    """

    def __init__(
        self,
        resource_repository: ResourceRepository,
        audit_repository: AuditRepository,
        notification_service: NotificationService,
    ):
        self._resource_repository = resource_repository
        self._audit_repository = audit_repository
        self._notification_service = notification_service

    async def execute(self, request: BulkImportRequest) -> BulkImportResult:
        """Execute bulk import workflow."""
        results = []
        errors = []

        for item_data in request.items:
            try:
                # Convert and validate
                resource = ResourceMapper.to_entity(item_data)
                ResourceLogic.validate_for_creation(resource)

                # Create resource
                created = await self._resource_repository.save(resource)
                results.append(created)

                # Audit
                await self._audit_repository.log(
                    operation="RESOURCE_CREATED",
                    entity_id=str(created.id),
                    details={"source": "bulk_import"},
                )
            except Exception as e:
                errors.append({"item": item_data, "error": str(e)})

        # Notify
        await self._notification_service.notify_import_complete(
            total=len(request.items),
            created=len(results),
            failed=len(errors),
        )

        return BulkImportResult(
            created=results,
            errors=errors,
            total_processed=len(request.items),
        )
```

**Why?**: Keeps complex workflows organized and reusable. Orchestration services remain clean and focused on simple use cases.

#### **DTOs** (`orchestration/dto/`)

**Definition**: Data Transfer Objects for request/response data transformation.

**Characteristics**:
- Separate DTOs for requests and responses
- Pydantic models with validation
- API-specific schemas
- Include OpenAPI examples

**Example** (`resource_dto.py`):
**Example** (`{context}_dto.py`):

```python
class ResourceDataDTO(BaseModel):
    """DTO for resource fields."""
    name: str
    description: Optional[str] = None
    metadata: Dict[str, Any] = Field(default_factory=dict)

class RequestCreateResourceDTO(BaseModel):
    """DTO for creating a resource."""
    name: str
    description: Optional[str] = None
    metadata: Optional[Dict[str, Any]] = None

class ResponseResourceDTO(BaseModel):
    """DTO for resource response."""
    id: str
    name: str
    description: Optional[str]
    is_active: bool
    created_at: datetime
    updated_at: datetime
    metadata: Dict[str, Any]

class ResponseResourceListDTO(BaseModel):
    """DTO for paginated resource list response."""
    data: List[ResponseResourceDTO]
    total: int
    skip: int
    limit: Optional[int] = None
```

**Why?**: Prevents domain leakage and provides layer separation. DTOs ensure API contracts are explicit and validated.

### **3. Domain Layer** (Core Business Logic)

The domain layer is organized by **business contexts** rather than having a flat structure. Each business context contains its own entities, interfaces, logic, and types.

#### **Business Contexts Structure**

```
domain/
├── audit/           # Audit logging and compliance
├── auth/            # Authentication and authorization
├── organization/    # Organization management
├── role/            # Role and permission management
└── user/            # User management
```

Each business context contains:
- `entities/` - Business entities (SQLAlchemy ORM models)
- `interfaces/` - Repository contracts
- `logic/` - Pure business rules and calculations
- `types/` - Value objects and domain types

**Why?**: Organizing by business contexts promotes domain-driven design, keeps related concepts together, and makes it easier to maintain bounded contexts as the system grows.

#### **Entities** (`domain/{context}/entities/`)

**Definition**: Business entities that represent core domain concepts. These are SQLAlchemy ORM models that inherit from `Base` for PostgreSQL integration.

**Characteristics**:
- Inherit from SQLAlchemy declarative `Base` class
- Use PostgreSQL-specific types (UUID, JSONB, etc.)
- Have properly defined column mappings with `Mapped`
- Support PostgreSQL-specific features (UUIDs, Enums, Indexes)
- Include composite indexes for query performance
- Define relationships with proper backref and cascade settings

**Example** (`{context}_entity.py`):

```python
from enum import Enum
from typing import Optional
from uuid import UUID

from sqlalchemy import Enum as SQLEnum, Index, String
from sqlalchemy.dialects.postgresql import UUID as PostgresUUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from platform_domain.mixin import Base
from platform_common.helper.uuid import UUID7


class ResourceType(str, Enum):
    """Resource type enumeration."""
    TYPE_A = "type_a"
    TYPE_B = "type_b"


class ResourceStatus(str, Enum):
    """Resource status enumeration."""
    DRAFT = "draft"
    ACTIVE = "active"
    ARCHIVED = "archived"


class Resource(Base):
    """Resource domain entity for PostgreSQL with UUIDs."""

    __tablename__ = "resources"

    # Primary Key with UUIDv7
    id: Mapped[UUID] = mapped_column(
        PostgresUUID(as_uuid=True),
        primary_key=True,
        default=UUID7.generate,
        nullable=False,
    )

    # Foreign Keys
    organization_id: Mapped[UUID] = mapped_column(
        PostgresUUID(as_uuid=True), nullable=False
    )

    # Core Fields
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[Optional[str]] = mapped_column(String(1000), nullable=True)

    # Enum fields with proper SQLAlchemy typing
    resource_type: Mapped[ResourceType] = mapped_column(
        SQLEnum(
            ResourceType,
            native_enum=False,
            length=50,
            values_callable=lambda x: [e.value for e in x],
        ),
        nullable=False,
    )

    status: Mapped[ResourceStatus] = mapped_column(
        SQLEnum(
            ResourceStatus,
            native_enum=False,
            length=50,
            values_callable=lambda x: [e.value for e in x],
        ),
        nullable=False,
        default=ResourceStatus.DRAFT,
    )

    is_active: Mapped[bool] = mapped_column(default=True, nullable=False)

    # Audit Fields (inherited or defined locally)
    created_at: Mapped[datetime] = mapped_column(nullable=False)
    updated_at: Mapped[datetime] = mapped_column(nullable=False)

    # Composite Indexes for query performance
    __table_args__ = (
        Index("idx_resource_organization", "organization_id"),
        Index("idx_resource_type", "resource_type"),
        Index("idx_resource_status", "status"),
        Index("idx_resource_organization_type", "organization_id", "resource_type"),
    )

    def __repr__(self) -> str:
        return f"<Resource(id={self.id}, name={self.name}, type={self.resource_type})>"
```

**Why?**: Domain entities are core business objects that need identity tracking, state management, and database persistence. SQLAlchemy models provide PostgreSQL integration with strong typing and relationship management while maintaining clean domain modeling.

#### **Types** (`domain/{context}/types/`)

**Definition**: Immutable value objects that encapsulate domain-specific data types and validation logic.

**Characteristics**:
- Immutable Pydantic BaseModel classes
- Self-validating with custom validators
- No identity (equality based on values)
- Domain-specific validation rules
- Can be shared across entities

**Examples**:
- `Address`: Address information with validation
- `ContactInfo`: Contact details with format validation
- `Money`: Monetary values with currency and precision

**Example** (`types.py`):

```python
class Address(BaseModel):
    """Address value object."""

    street: str
    city: str
    postal_code: str
    country: str

    @field_validator("street", "city", "country")
    @classmethod
    def validate_required_string(cls, v: str) -> str:
        if not v or not v.strip():
            raise ValueError("Field is required")
        return v.strip()

    @field_validator("postal_code")
    @classmethod
    def validate_postal_code(cls, v: str) -> str:
        if not v or len(v.strip()) < 3:
            raise ValueError("Invalid postal code")
        return v.strip()

class ContactInfo(BaseModel):
    """Contact information value object."""

    email: str
    phone: Optional[str] = None

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower().strip()

class Money(BaseModel):
    """Money value object."""

    amount: float
    currency: str = "USD"

    @field_validator("amount")
    @classmethod
    def validate_amount(cls, v: float) -> float:
        if v < 0:
            raise ValueError("Amount cannot be negative")
        return round(v, 2)  # Round to 2 decimal places
```

**Why?**: Domain value objects need consistent validation and normalization everywhere they're used. Value objects ensure data integrity and encapsulate validation logic.

#### **Domain Logic** (`domain/{context}/logic/`)

**Definition**: Stateless services containing complex business rules and domain operations that don't naturally belong to a single entity.

**Characteristics**:
- Static methods for business rule validation
- Coordinate multiple entities and value objects
- Handle complex domain calculations
- Provide reusable business logic

**Example** (`{context}_logic.py`):

```python
class ResourceLogic:
    """
    Pure domain logic for resource operations.
    Handles business rules and calculations.
    """

    # Business Constants
    MAX_UPDATES_PER_DAY = 5
    MIN_AGE_FOR_DELETION_DAYS = 30

    @staticmethod
    def validate_for_creation(resource: Resource) -> None:
        """Validate resource is ready for creation."""
        if not resource.name or not resource.name.strip():
            raise ValidationError("Resource name is required")

    @staticmethod
    def validate_for_update(existing: Resource, request: RequestUpdateResourceDTO) -> None:
        """
        Business rule: validate resource can be updated.

        Args:
            existing: Existing resource entity
            request: Update request

        Raises:
            BusinessError: If business rules prevent update
        """
        # Check update frequency limit
        updates_today = ResourceLogic.count_updates_today(existing)
        if updates_today >= ResourceLogic.MAX_UPDATES_PER_DAY:
            raise BusinessError(
                message=f"Cannot update more than {ResourceLogic.MAX_UPDATES_PER_DAY} times per day",
                error_code="RATE_LIMIT_EXCEEDED",
            )

    @staticmethod
    def validate_for_deletion(resource: Resource, current_time: datetime) -> None:
        """
        Business rule: determine if resource can be deleted.

        Args:
            resource: Resource to check
            current_time: Current timestamp

        Raises:
            BusinessError: If business rules prevent deletion
        """
        record_age_days = (current_time - resource.created_at).days

        if record_age_days < ResourceLogic.MIN_AGE_FOR_DELETION_DAYS:
            raise BusinessError(
                message=f"Can only delete after {ResourceLogic.MIN_AGE_FOR_DELETION_DAYS} days",
                error_code="DELETION_NOT_ALLOWED",
            )

    @staticmethod
    def calculate_priority_score(resource: Resource) -> float:
        """
        Calculate priority score based on business rules.

        Args:
            resource: Resource to score

        Returns:
            Priority score between 0.0 and 1.0
        """
        score = 0.0

        # Recency factor
        age_days = (datetime.now(UTC) - resource.created_at).days
        if age_days < 7:
            score += 0.3
        elif age_days < 30:
            score += 0.2

        # Activity factor
        if resource.is_active:
            score += 0.4

        return min(1.0, score)
```

**Why?**: Complex business operations involve multiple business rules and calculations that span multiple entities. Domain logic services keep business rules centralized and testable.

#### **Repository Interfaces** (`domain/{context}/interfaces/`)

**Definition**: Abstract contracts defining data access operations for each business context.

**Characteristics**:
- Abstract base classes with async methods
- Technology-agnostic data access contracts
- Entity-specific operations
- Multi-tenant aware (organization-scoped operations)

**Example** (`interfaces.py`):

```python
class ResourceRepository(ABC):
    @abstractmethod
    async def find_by_id(self, resource_id: str) -> Optional[Resource]:
        """Find a resource by its unique identifier."""
        pass

    @abstractmethod
    async def find_all(
        self, skip: int = 0, limit: Optional[int] = None
    ) -> List[Resource]:
        """Find all resources."""
        pass

    @abstractmethod
    async def find_active(
        self, skip: int = 0, limit: Optional[int] = None
    ) -> List[Resource]:
        """Find all active resources."""
        pass

    @abstractmethod
    async def save(self, resource: Resource) -> Resource:
        """Save a resource entity."""
        pass

    @abstractmethod
    async def update(self, resource_id: str, updates: Dict[str, Any]) -> bool:
        """Update resource fields."""
        pass

    @abstractmethod
    async def delete(self, resource_id: str) -> bool:
        """Permanently delete a resource."""
        pass

    @abstractmethod
    async def count_all(self) -> int:
        """Count total resources."""
        pass
```

**Why?**: Defines what data access operations are needed without tying to specific databases. Enables dependency inversion and makes the domain layer technology-agnostic.

### **4. Infrastructure Layer** (External Concerns)

#### **Repository Implementations** (`infrastructure/repositories/`)

**Definition**: SQLAlchemy implementations of domain repository interfaces using PostgreSQL.

**Example** (`sqlalchemy_resource_repository.py`):

```python
import logging
from typing import Any, List, Optional
from uuid import UUID

from sqlalchemy import and_, select, update
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker
from domain.resource.entities.resource_entity import Resource, ResourceStatus
from domain.resource.interfaces.resource_repository import ResourceRepository

logger = logging.getLogger(__name__)


class SQLAlchemyResourceRepository(ResourceRepository):
    """
    SQLAlchemy implementation of ResourceRepository.
    Provides data access operations for resource entities with PostgreSQL.
    """

    def __init__(self, session: async_sessionmaker[AsyncSession]) -> None:
        """
        Initialize repository with async session factory.

        Args:
            session: SQLAlchemy async_sessionmaker instance for creating sessions
        """
        self._session_factory = session

    async def find_by_id(self, resource_id: str | UUID) -> Optional[Resource]:
        """
        Find a resource by its unique identifier.

        Args:
            resource_id: Unique resource identifier

        Returns:
            Resource entity if found, None otherwise
        """
        if isinstance(resource_id, str):
            resource_id = UUID(resource_id)

        async with self._session_factory() as session:
            stmt = select(Resource).where(Resource.id == resource_id)
            result = await session.execute(stmt)
            return result.scalar_one_or_none()

    async def find_all(
        self, skip: int = 0, limit: Optional[int] = None
    ) -> List[Resource]:
        """
        Find all resources.

        Args:
            skip: Number of records to skip
            limit: Maximum number of records to return

        Returns:
            List of resource entities
        """
        async with self._session_factory() as session:
            stmt = select(Resource).offset(skip)
            if limit:
                stmt = stmt.limit(limit)
            result = await session.execute(stmt)
            return result.scalars().all()

    async def find_active(
        self, skip: int = 0, limit: Optional[int] = None
    ) -> List[Resource]:
        """
        Find all active resources.

        Args:
            skip: Number of records to skip
            limit: Maximum number of records to return

        Returns:
            List of active resource entities
        """
        async with self._session_factory() as session:
            stmt = (
                select(Resource)
                .where(Resource.status == ResourceStatus.ACTIVE)
                .offset(skip)
            )
            if limit:
                stmt = stmt.limit(limit)
            result = await session.execute(stmt)
            return result.scalars().all()

    async def save(self, resource: Resource) -> Resource:
        """
        Save a resource entity to the database.

        Args:
            resource: The resource entity to save

        Returns:
            The saved resource entity

        Raises:
            Exception: If save operation fails
        """
        try:
            async with self._session_factory() as session:
                session.add(resource)
                await session.flush()
                await session.commit()
                logger.info(f"Resource saved: {resource.id}")
                return resource
        except Exception as e:
            logger.error(f"Failed to save resource: {e}")
            raise

    async def update(self, resource_id: str | UUID, updates: dict[str, Any]) -> bool:
        """
        Update resource fields.

        Args:
            resource_id: ID of the resource to update
            updates: Dictionary of field updates

        Returns:
            True if update was successful, False otherwise
        """
        try:
            if isinstance(resource_id, str):
                resource_id = UUID(resource_id)

            async with self._session_factory() as session:
                stmt = (
                    update(Resource)
                    .where(Resource.id == resource_id)
                    .values(**updates)
                )
                result = await session.execute(stmt)
                await session.commit()

                return result.rowcount > 0
        except Exception as e:
            logger.error(f"Error updating resource {resource_id}: {e}")
            return False

    async def delete(self, resource_id: str | UUID) -> bool:
        """
        Permanently delete a resource.

        Args:
            resource_id: ID of the resource to delete

        Returns:
            True if deletion was successful, False otherwise
        """
        try:
            if isinstance(resource_id, str):
                resource_id = UUID(resource_id)

            async with self._session_factory() as session:
                resource = await self.find_by_id(resource_id)
                if not resource:
                    return False

                await session.delete(resource)
                await session.commit()
                logger.info(f"Resource deleted: {resource_id}")
                return True
        except Exception as e:
            logger.error(f"Error deleting resource {resource_id}: {e}")
            return False

    async def count_all(self) -> int:
        """Count total resources."""
        async with self._session_factory() as session:
            stmt = select(func.count()).select_from(Resource)
            result = await session.execute(stmt)
            return result.scalar() or 0
```

**Why?**: Isolates SQLAlchemy and PostgreSQL-specific code from business logic. Enables testing with different databases and maintains technology-agnostic domain layer through dependency inversion.

#### **External Services** (`infrastructure/external/`)

**Definition**: Integrations with external systems.

**Examples**:
- `AzureADClient`: Azure AD authentication and user management
- Integration clients for third-party services

**Why?**: External integrations change frequently and need isolation from core business logic.

#### **Messaging** (`infrastructure/messaging/`)

**Definition**: Event-driven architecture support.

**Examples**:
- `EventPublisher`: Publish domain events to message queues
- Async communication patterns

**Why?**: Enables event-driven architecture for loose coupling between services and async processing.

#### **Configuration & Database** (`infrastructure/`)

- `config.py`: Orchestration settings management
- `database.py`: MongoDB connection and initialization
- `container.py`: Dependency injection configuration

#### **Monitoring** (`infrastructure/monitoring/`)

- `error_metrics.py`: Error tracking and metrics
- Application observability

**Why?**: Infrastructure concerns should be isolated and easily replaceable. Dependency injection enables clean testing and configuration management.

## Key Architectural Principles

### **Dependency Rule**

- Inner layers (Domain) don't depend on outer layers
- Dependencies point inward only
- Controller → Orchestration Services → Domain Logic → Infrastructure
- Infrastructure implements Domain interfaces (Dependency Inversion)

### **Business Context Organization**

- Domain layer organized by business contexts (user, auth, organization, etc.)
- Each context is self-contained with entities, logic, and interfaces
- Promotes bounded contexts and domain-driven design

### **Orchestration Service Pattern**

- **Orchestration Services**: Single service per context handles complete use case execution
  - Accepts request DTOs from controllers
  - Converts DTOs to domain entities using mappers
  - Calls domain logic for business rules
  - Interacts with repositories for data access
  - Returns response DTOs
- No separate coordinator or domain service layers
- Clear responsibility separation at use case level

### **Multi-Tenant Architecture**

- Organization-scoped data access throughout all layers
- Repository methods include `org_id` parameters
- PostgreSQL indexes optimized for multi-tenant queries

For detailed implementation of our multi-tenant architecture, see the [Multi-Tenancy Guide](multi-tenancy.md).

### **Dependency Injection**

- `dependency-injector` library for IoC container
- All services and repositories injected through container
- Clean separation of concerns and testability

### **Clean Separation**

- **Orchestration Services**: DTO conversion, business logic coordination, data persistence
- **Domain Logic**: Pure business rules and calculations
- **Use Cases**: Complex multi-step operations (optional)
- **Infrastructure**: External concerns (PostgreSQL, Azure AD, messaging)
- **Controller**: REST API adapters and middleware

This architecture provides maintainability, testability, and flexibility for a multi-tenant identity management service with complex business rules and external integrations.
