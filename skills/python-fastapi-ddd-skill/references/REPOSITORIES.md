# Repository Pattern in Python DDD

## Definition

Repositories abstract persistence, providing a collection-like interface for accessing domain entities. The interface is defined in the Domain layer; the implementation lives in the Infrastructure layer.

## Domain Layer: Interface

```python
from abc import ABC, abstractmethod
from typing import List, Optional

class TodoRepository(ABC):
    @abstractmethod
    def save(self, todo: Todo) -> None:
        """Persist new or updated entity."""

    @abstractmethod
    def find_by_id(self, todo_id: TodoId) -> Optional[Todo]:
        """Return entity by ID, or None."""

    @abstractmethod
    def find_all(self) -> List[Todo]:
        """Return all entities."""

    @abstractmethod
    def delete(self, todo_id: TodoId) -> None:
        """Remove entity by ID."""
```

## Infrastructure Layer: Implementation

### SQLAlchemy Repository

```python
from sqlalchemy.exc import NoResultFound
from sqlalchemy.orm.session import Session

class TodoRepositoryImpl(TodoRepository):
    def __init__(self, session: Session):
        self.session = session

    def find_by_id(self, todo_id: TodoId) -> Optional[Todo]:
        try:
            row = self.session.query(TodoDTO).filter_by(id=todo_id.value).one()
        except NoResultFound:
            return None
        return row.to_entity()

    def find_all(self) -> List[Todo]:
        rows = (
            self.session.query(TodoDTO)
            .order_by(desc(TodoDTO.created_at))
            .limit(20)
            .all()
        )
        return [dto.to_entity() for dto in rows]

    def save(self, todo: Todo) -> None:
        todo_dto = TodoDTO.from_entity(todo)
        try:
            existing = self.session.query(TodoDTO).filter_by(id=todo.id.value).one()
        except NoResultFound:
            self.session.add(todo_dto)
        else:
            existing.title = todo_dto.title
            existing.description = todo_dto.description
            existing.status = todo_dto.status
            existing.updated_at = todo_dto.updated_at
            existing.completed_at = todo_dto.completed_at

    def delete(self, todo_id: TodoId) -> None:
        self.session.query(TodoDTO).filter_by(id=todo_id.value).delete()
```

### Factory Function

Expose a factory function, keeping the concrete class as an implementation detail:

```python
def new_todo_repository(session: Session) -> TodoRepository:
    return TodoRepositoryImpl(session)
```

## DTO (Data Transfer Object)

DTOs bridge domain entities and database models with bidirectional conversion:

```python
from sqlalchemy.orm import Mapped, mapped_column

class TodoDTO(Base):
    __tablename__ = "todo"
    id: Mapped[UUID] = mapped_column(primary_key=True, autoincrement=False)
    title: Mapped[str] = mapped_column(String(100), nullable=False)
    description: Mapped[str] = mapped_column(String(1000), nullable=True)
    status: Mapped[str] = mapped_column(index=True, nullable=False)
    created_at: Mapped[int] = mapped_column(index=True, nullable=False)
    updated_at: Mapped[int] = mapped_column(index=True, nullable=False)
    completed_at: Mapped[int] = mapped_column(index=True, nullable=True)

    def to_entity(self) -> Todo:
        return Todo(
            TodoId(self.id),
            TodoTitle(self.title),
            TodoDescription(self.description) if self.description else None,
            TodoStatus(self.status),
            datetime.fromtimestamp(self.created_at / 1000, tz=timezone.utc),
            datetime.fromtimestamp(self.updated_at / 1000, tz=timezone.utc),
            datetime.fromtimestamp(self.completed_at / 1000, tz=timezone.utc)
            if self.completed_at else None,
        )

    @staticmethod
    def from_entity(todo: Todo) -> "TodoDTO":
        return TodoDTO(
            id=todo.id.value,
            title=todo.title.value,
            description=todo.description.value if todo.description else None,
            status=todo.status.value,
            created_at=int(todo.created_at.timestamp() * 1000),
            updated_at=int(todo.updated_at.timestamp() * 1000),
            completed_at=int(todo.completed_at.timestamp() * 1000)
            if todo.completed_at else None,
        )
```

## Dependency Injection with FastAPI

Wire the repository via FastAPI's `Depends()`:

```python
from fastapi import Depends

def get_session() -> Iterator[Session]:
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

def get_todo_repository(session: Session = Depends(get_session)) -> TodoRepository:
    return new_todo_repository(session)
```

## Guidelines

- Domain layer defines interface only (ABC with `@abstractmethod`)
- Infrastructure layer implements using concrete ORM/database
- DTOs handle all conversion logic between layers
- Store timestamps as milliseconds (`int`) in the database for portability
- Use factory functions (`new_*`) to construct repository implementations
- Session lifecycle (commit/rollback/close) is managed in the DI layer, not in the repository
