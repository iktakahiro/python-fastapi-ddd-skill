# Entities in Python DDD

## Definition

Entities are domain objects with a unique identity that persists over time. Two entities are equal when their identifiers match, regardless of other attribute values.

## Implementation Pattern

```python
from datetime import datetime
from typing import Optional

class Todo:
    def __init__(
        self,
        id: TodoId,
        title: TodoTitle,
        description: Optional[TodoDescription] = None,
        status: TodoStatus = TodoStatus.NOT_STARTED,
        created_at: datetime = datetime.now(),
        updated_at: datetime = datetime.now(),
        completed_at: Optional[datetime] = None,
    ):
        self._id = id
        self._title = title
        self._description = description
        self._status = status
        self._created_at = created_at
        self._updated_at = updated_at
        self._completed_at = completed_at

    def __eq__(self, obj: object) -> bool:
        if isinstance(obj, Todo):
            return self.id == obj.id
        return False
```

## Key Characteristics

### Identity-Based Equality

Override `__eq__` to compare by identifier only:

```python
def __eq__(self, obj: object) -> bool:
    if isinstance(obj, Todo):
        return self.id == obj.id
    return False
```

### Encapsulation with Properties

Use private attributes with read-only property accessors:

```python
@property
def id(self) -> TodoId:
    return self._id

@property
def title(self) -> TodoTitle:
    return self._title

@property
def status(self) -> TodoStatus:
    return self._status
```

### State-Changing Methods

Encapsulate state transitions with business rule enforcement:

```python
def start(self) -> None:
    """Mark as in progress."""
    self._status = TodoStatus.IN_PROGRESS
    self._updated_at = datetime.now()

def complete(self) -> None:
    """Mark as completed with validation."""
    if self._status == TodoStatus.COMPLETED:
        raise ValueError("Already completed")
    self._status = TodoStatus.COMPLETED
    self._completed_at = datetime.now()
    self._updated_at = self._completed_at

def update_title(self, new_title: TodoTitle) -> None:
    self._title = new_title
    self._updated_at = datetime.now()
```

### Factory Methods

Use static factory methods for creation with generated identifiers:

```python
@staticmethod
def create(title: TodoTitle, description: Optional[TodoDescription] = None) -> "Todo":
    return Todo(TodoId.generate(), title, description)
```

### Domain Query Methods

Encode business rules as query methods:

```python
@property
def is_completed(self) -> bool:
    return self._status == TodoStatus.COMPLETED

def is_overdue(self, deadline: datetime, current_time: Optional[datetime] = None) -> bool:
    if self.is_completed:
        return False
    return (current_time or datetime.now()) > deadline
```

## Guidelines

- All attributes use Value Objects (not raw primitives)
- State changes go through methods, never direct attribute assignment
- Entity contains no infrastructure concerns (no ORM, no HTTP)
- Factory method (`create`) handles default values (ID generation, initial status)
- Validation of state transitions belongs in the entity or its use case
