# Onion Architecture for Python DDD

## Layer Diagram

```
┌─────────────────────────────────────────┐
│         Presentation Layer              │  FastAPI handlers, Pydantic schemas
├─────────────────────────────────────────┤
│           UseCase Layer                 │  Application business rules
├─────────────────────────────────────────┤
│        Infrastructure Layer             │  DB repos, DTOs, DI config
├─────────────────────────────────────────┤
│           Domain Layer                  │  Entities, Value Objects, Repo interfaces
└─────────────────────────────────────────┘
```

**Dependencies flow inward only.** Outer layers depend on inner layers; inner layers know nothing about outer layers.

## Layer Responsibilities

### Domain Layer (Innermost)

The core of the application. Contains pure business logic with **zero external dependencies**.

**Contains:**
- **Entities**: Objects with identity and lifecycle (e.g., `Todo`)
- **Value Objects**: Immutable typed values (e.g., `TodoId`, `TodoTitle`, `TodoStatus`)
- **Repository Interfaces**: Abstract persistence contracts (`TodoRepository(ABC)`)
- **Domain Exceptions**: Business rule violation errors (`TodoNotFoundError`)

**Rules:**
- No imports from FastAPI, SQLAlchemy, Pydantic, or any framework
- No I/O operations (no database, no HTTP, no filesystem)
- All validation logic for business rules

### Infrastructure Layer

Implements domain interfaces and integrates with external systems.

**Contains:**
- **Repository Implementations**: `TodoRepositoryImpl` using SQLAlchemy
- **DTOs**: `TodoDTO` mapping entities to/from database models
- **Database Configuration**: Engine, session, table creation
- **Dependency Injection**: FastAPI `Depends()` wiring

**Rules:**
- Implements domain interfaces (e.g., `TodoRepository`)
- Converts between domain entities and persistence models via DTOs
- Manages session lifecycle (commit/rollback/close)

### UseCase Layer

Orchestrates domain objects to accomplish application actions.

**Contains:**
- **Use Case Interfaces**: Abstract definitions (`CreateTodoUseCase(ABC)`)
- **Use Case Implementations**: Concrete business workflows
- **Factory Functions**: `new_create_todo_usecase()`

**Rules:**
- One use case class per application action
- Single public `execute()` method
- Receives and returns domain objects (not DTOs or schemas)
- Raises domain exceptions for business rule violations

### Presentation Layer (Outermost)

Handles HTTP requests and responses.

**Contains:**
- **Route Handlers**: FastAPI endpoint functions
- **Request Schemas**: Pydantic models for input validation (`TodoCreateSchema`)
- **Response Schemas**: Pydantic models for serialization (`TodoSchema`)
- **Error Messages**: HTTP error response models

**Rules:**
- Converts HTTP input to domain Value Objects
- Calls use case `execute()` methods
- Converts domain entities to response schemas
- Maps domain exceptions to HTTP status codes

## Data Flow

### Request Flow (Create Todo)

```
POST /todos { "title": "Buy milk" }
    │
    ▼
Presentation: TodoCreateSchema validates JSON
    │
    ▼
Presentation: Constructs TodoTitle("Buy milk")
    │
    ▼
UseCase: CreateTodoUseCaseImpl.execute(title)
    │
    ▼
Domain: Todo.create(title) → new Todo entity
    │
    ▼
Infrastructure: TodoRepository.save(todo)
    │
    ▼
Infrastructure: TodoDTO.from_entity(todo) → INSERT
    │
    ▼
Presentation: TodoSchema.from_entity(todo) → JSON response
```

### Dependency Injection Chain

```
FastAPI Request
    ↓
get_session() → SQLAlchemy Session (commit/rollback managed here)
    ↓
get_todo_repository(session) → TodoRepositoryImpl
    ↓
get_create_todo_usecase(repository) → CreateTodoUseCaseImpl
    ↓
Route handler receives usecase, calls execute()
```

## Application Bootstrap

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    create_tables()    # Setup
    yield
    engine.dispose()   # Teardown

app = FastAPI(title="DDD Todo API", lifespan=lifespan)

todo_handler = TodoApiRouteHandler()
todo_handler.register_routes(app)
```

## Adding a New Aggregate

When adding a new domain concept (e.g., `User`):

1. **Domain**: Create `domain/user/entities/`, `value_objects/`, `repositories/`, `exceptions/`
2. **Infrastructure**: Create `infrastructure/sqlite/user/user_dto.py` and `user_repository.py`
3. **UseCase**: Create `usecase/user/create_user_usecase.py`, etc.
4. **Presentation**: Create `presentation/api/user/handlers/`, `schemas/`, `error_messages/`
5. **DI**: Add `get_user_repository()` and `get_*_user_usecase()` functions to `injection.py`
6. **Bootstrap**: Register new route handler in `main.py`

Each aggregate is self-contained within its subdirectory across all layers.
