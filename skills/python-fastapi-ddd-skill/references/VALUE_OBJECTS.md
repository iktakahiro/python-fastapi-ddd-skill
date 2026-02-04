# Value Objects in Python DDD

## Definition

Value Objects are immutable domain concepts identified by their content, not by a unique identifier. Two Value Objects with the same data are considered equal.

## Implementation Patterns

### String Wrapper with Validation

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class TodoTitle:
    value: str

    def __post_init__(self):
        if not self.value:
            raise ValueError("Title is required")
        if len(self.value) > 100:
            raise ValueError("Title must be 100 characters or less")

    def __str__(self) -> str:
        return self.value
```

### Optional String Wrapper

```python
@dataclass(frozen=True)
class TodoDescription:
    value: str

    def __post_init__(self):
        if len(self.value) > 1000:
            raise ValueError("Description must be 1000 characters or less")

    def __str__(self) -> str:
        return self.value
```

### UUID Identifier

```python
from dataclasses import dataclass
from uuid import UUID, uuid4

@dataclass(frozen=True)
class TodoId:
    value: UUID

    @staticmethod
    def generate() -> "TodoId":
        return TodoId(uuid4())

    def __str__(self) -> str:
        return str(self.value)
```

### Enum-Based Status

```python
from enum import Enum

class TodoStatus(Enum):
    NOT_STARTED = "not_started"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"

    def __str__(self) -> str:
        return self.value
```

### Composite Value Object

For multi-field value objects:

```python
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self):
        if self.amount < 0:
            raise ValueError("Amount cannot be negative")
        if len(self.currency) != 3:
            raise ValueError("Currency must be ISO 4217 code")

    def add(self, other: "Money") -> "Money":
        if self.currency != other.currency:
            raise ValueError("Cannot add different currencies")
        return Money(self.amount + other.amount, self.currency)
```

## Key Characteristics

| Property | Implementation |
|----------|---------------|
| **Immutability** | `@dataclass(frozen=True)` |
| **Self-validation** | `__post_init__` raises `ValueError` |
| **Equality by value** | Automatic from `@dataclass` |
| **No identity** | No unique ID field |
| **String representation** | `__str__` returns human-readable form |

## When to Use Value Objects

- Identifiers (user ID, order ID) wrapping UUID
- Constrained strings (email, title, description)
- Enumerations (status, category, role)
- Numeric values with rules (money, percentage, quantity)
- Composite concepts (address, date range, coordinates)

## Guidelines

- Always use `frozen=True` to enforce immutability
- Validate constraints in `__post_init__` and raise `ValueError`
- Wrap primitive types even for simple fields (type safety + validation in one place)
- Value Objects with operations (e.g., `Money.add`) return new instances
- Keep validation focused on the single concept the Value Object represents
