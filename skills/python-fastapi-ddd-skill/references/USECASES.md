# Use Cases in Python DDD

## Definition

Use Cases (Application Services) orchestrate domain objects to fulfill a single application action. Each use case has one public `execute` method.

## Structure Pattern

Every use case follows the same three-part structure:

1. **Abstract interface** (defines the contract)
2. **Concrete implementation** (contains logic)
3. **Factory function** (constructs the implementation)

### Create Use Case (Simple)

```python
from abc import ABC, abstractmethod

class CreateTodoUseCase(ABC):
    @abstractmethod
    def execute(self, title: TodoTitle, description: Optional[TodoDescription] = None) -> Todo: ...

class CreateTodoUseCaseImpl(CreateTodoUseCase):
    def __init__(self, todo_repository: TodoRepository):
        self.todo_repository = todo_repository

    def execute(self, title: TodoTitle, description: Optional[TodoDescription] = None) -> Todo:
        todo = Todo.create(title=title, description=description)
        self.todo_repository.save(todo)
        return todo

def new_create_todo_usecase(todo_repository: TodoRepository) -> CreateTodoUseCase:
    return CreateTodoUseCaseImpl(todo_repository)
```

### State Transition Use Case (With Validation)

```python
class StartTodoUseCase(ABC):
    @abstractmethod
    def execute(self, todo_id: TodoId) -> Todo: ...

class StartTodoUseCaseImpl(StartTodoUseCase):
    def __init__(self, todo_repository: TodoRepository):
        self.todo_repository = todo_repository

    def execute(self, todo_id: TodoId) -> Todo:
        todo = self.todo_repository.find_by_id(todo_id)

        if todo is None:
            raise TodoNotFoundError

        if todo.is_completed:
            raise TodoAlreadyCompletedError

        if todo.status == TodoStatus.IN_PROGRESS:
            raise TodoAlreadyStartedError

        todo.start()
        self.todo_repository.save(todo)
        return todo

def new_start_todo_usecase(todo_repository: TodoRepository) -> StartTodoUseCase:
    return StartTodoUseCaseImpl(todo_repository)
```

### Query Use Case

```python
class FindTodoByIdUseCase(ABC):
    @abstractmethod
    def execute(self, todo_id: TodoId) -> Todo: ...

class FindTodoByIdUseCaseImpl(FindTodoByIdUseCase):
    def __init__(self, todo_repository: TodoRepository):
        self.todo_repository = todo_repository

    def execute(self, todo_id: TodoId) -> Todo:
        todo = self.todo_repository.find_by_id(todo_id)
        if todo is None:
            raise TodoNotFoundError
        return todo
```

## Dependency Injection Wiring

Each use case is wired via FastAPI's `Depends()`:

```python
def get_create_todo_usecase(
    todo_repository: TodoRepository = Depends(get_todo_repository),
) -> CreateTodoUseCase:
    return new_create_todo_usecase(todo_repository)

def get_start_todo_usecase(
    todo_repository: TodoRepository = Depends(get_todo_repository),
) -> StartTodoUseCase:
    return new_start_todo_usecase(todo_repository)
```

The dependency chain: `Session → Repository → UseCase → Handler`

## Domain Exceptions

Use cases raise domain-specific exceptions for business rule violations:

```python
class TodoNotFoundError(Exception):
    message = "The Todo you specified does not exist."

    def __str__(self):
        return TodoNotFoundError.message

class TodoAlreadyStartedError(Exception):
    message = "The specified Todo has already been started."

class TodoAlreadyCompletedError(Exception):
    message = "The specified Todo has already been completed."
```

## Presentation Layer Error Handling

The handler catches domain exceptions and maps them to HTTP responses:

```python
@app.patch("/todos/{todo_id}/start")
def start_todo(
    todo_id: UUID,
    usecase: StartTodoUseCase = Depends(get_start_todo_usecase),
):
    _id = TodoId(todo_id)
    try:
        todo = usecase.execute(_id)
    except TodoNotFoundError as e:
        raise HTTPException(status_code=404, detail=e.message) from e
    except TodoAlreadyStartedError as e:
        raise HTTPException(status_code=400, detail=e.message) from e
    except TodoAlreadyCompletedError as e:
        raise HTTPException(status_code=400, detail=e.message) from e
    return TodoSchema.from_entity(todo)
```

## Guidelines

- One use case class per action (Create, Start, Complete, Update, Delete, FindById, FindAll)
- Abstract interface + implementation keeps the UseCase layer testable
- Use cases receive domain Value Objects as parameters, not raw primitives
- Use cases return domain entities, not DTOs or schemas
- Domain exception handling belongs in the use case; HTTP mapping belongs in the handler
- Factory functions (`new_*`) hide implementation details from consumers
