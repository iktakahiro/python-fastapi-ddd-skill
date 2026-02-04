# Testing Patterns (dddpy-based)

This reference uses the testing style shown in `dddpy/tests/`.

## 1) Value Object tests

Value Objects should validate in `__post_init__`. Test both valid and invalid inputs.

```python
import pytest

def test_todo_title_accepts_valid_value():
    title = TodoTitle("Hello")
    assert str(title) == "Hello"

def test_todo_title_rejects_empty():
    with pytest.raises(ValueError, match="Title is required"):
        TodoTitle("")

def test_todo_title_rejects_too_long():
    with pytest.raises(ValueError, match="100 characters"):
        TodoTitle("a" * 101)
```

## 2) Entity tests (state transitions + invariants)

Test methods like `start()`, `complete()`, `update_*()` and derived properties like `is_completed`.

```python
import pytest
from dddpy.domain.todo.entities import Todo
from dddpy.domain.todo.value_objects import TodoTitle, TodoStatus

def test_start_sets_in_progress():
    todo = Todo.create(TodoTitle("Test"))
    todo.start()
    assert todo.status == TodoStatus.IN_PROGRESS

def test_complete_twice_raises():
    todo = Todo.create(TodoTitle("Test"))
    todo.complete()
    with pytest.raises(ValueError, match="Already completed"):
        todo.complete()
```

### Time-based logic

If your Domain has time rules (deadlines, TTL, etc.), prefer passing explicit timestamps
into methods so tests are deterministic.

## 3) UseCase tests (mock repository)

### Arrange: repository mock

```python
from unittest.mock import Mock
import pytest

@pytest.fixture
def todo_repository_mock():
    return Mock(spec=TodoRepository)
```

### Example: happy path

```python
def test_find_todos_returns_repo_results(todo_repository_mock):
    todo_repository_mock.find_all.return_value = [Todo.create(TodoTitle("A"))]

    usecase = FindTodosUseCaseImpl(todo_repository_mock)
    result = usecase.execute()

    assert len(result) == 1
    todo_repository_mock.find_all.assert_called_once()
```

### Example: not found -> domain exception

```python
def test_find_by_id_raises_when_missing(todo_repository_mock):
    todo_repository_mock.find_by_id.return_value = None
    usecase = FindTodoByIdUseCaseImpl(todo_repository_mock)

    with pytest.raises(TodoNotFoundError):
        usecase.execute(TodoId.generate())
```

### Example: lifecycle validation

Use cases should enforce cross-entity rules and raise domain-specific errors.

```python
def test_complete_requires_started(todo_repository_mock):
    todo = Todo.create(TodoTitle("A"))
    todo_repository_mock.find_by_id.return_value = todo
    usecase = CompleteTodoUseCaseImpl(todo_repository_mock)

    with pytest.raises(TodoNotStartedError):
        usecase.execute(todo.id)
```

## 4) What not to test in unit tests

- SQLAlchemy mappings and DB I/O (belongs to infrastructure integration tests)
- FastAPI routing / dependency injection (belongs to API tests)

