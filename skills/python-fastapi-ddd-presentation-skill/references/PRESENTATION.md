# FastAPI Presentation Patterns (dddpy-based)

This reference expands the patterns used in `dddpy` for the Presentation layer.

## 1) Handlers: route registrar class

Use one “route handler” class per aggregate to keep `main.py` thin and isolate routing concerns.

### Template

```python
from typing import List
from uuid import UUID

from fastapi import Depends, FastAPI, HTTPException, status

from yourapp.domain.todo.exceptions import TodoNotFoundError
from yourapp.domain.todo.value_objects import TodoId, TodoTitle
from yourapp.infrastructure.di.injection import get_create_todo_usecase
from yourapp.presentation.api.todo.schemas import TodoCreateSchema, TodoSchema
from yourapp.usecase.todo import CreateTodoUseCase


class TodoApiRouteHandler:
    def register_routes(self, app: FastAPI) -> None:
        @app.post(
            "/todos",
            response_model=TodoSchema,
            status_code=status.HTTP_201_CREATED,
        )
        def create_todo(
            data: TodoCreateSchema,
            usecase: CreateTodoUseCase = Depends(get_create_todo_usecase),
        ) -> TodoSchema:
            # 1) primitives -> value objects
            try:
                title = TodoTitle(data.title)
            except ValueError as e:
                raise HTTPException(
                    status_code=status.HTTP_400_BAD_REQUEST,
                    detail=str(e),
                ) from e

            # 2) execute use case
            todo = usecase.execute(title=title)

            # 3) entity -> response schema
            return TodoSchema.from_entity(todo)

        @app.get(
            "/todos/{todo_id}",
            response_model=TodoSchema,
            status_code=status.HTTP_200_OK,
            responses={
                status.HTTP_404_NOT_FOUND: {"model": ErrorMessageTodoNotFound},
            },
        )
        def get_todo(
            todo_id: UUID,
            usecase: FindTodoByIdUseCase = Depends(get_find_todo_by_id_usecase),
        ) -> TodoSchema:
            _id = TodoId(todo_id)
            try:
                todo = usecase.execute(_id)
            except TodoNotFoundError as e:
                raise HTTPException(
                    status_code=status.HTTP_404_NOT_FOUND,
                    detail=e.message,
                ) from e
            return TodoSchema.from_entity(todo)
```

### Notes

- Keep the handler “dumb”: do not embed domain rules (e.g., “already completed”) here.
- Convert Value Object `ValueError` to 400.
- Convert Domain exceptions (NotFound, lifecycle errors) to 404/400.
- Prefer `fastapi.status` constants over hard-coded integers.

## 2) Schemas: request/response separation

### Request schema

Use Pydantic constraints as an initial validation layer:

```python
from pydantic import BaseModel, Field


class TodoCreateSchema(BaseModel):
    title: str = Field(min_length=1, max_length=100, examples=["Buy milk"])
    description: str | None = Field(default=None, max_length=1000)
```

Still validate again in Value Objects (the Domain is the source of truth).

### Response schema with `from_entity()`

Make serialization explicit. Convert:

- `UUID` → `str`
- `datetime` → `int` milliseconds (or ISO 8601, but be consistent)
- optional Value Objects → `None` or `""` depending on API contract

```python
from pydantic import BaseModel, Field

from yourapp.domain.todo.entities import Todo


class TodoSchema(BaseModel):
    id: str
    title: str
    description: str
    status: str
    created_at: int
    updated_at: int
    completed_at: int | None

    @staticmethod
    def from_entity(todo: Todo) -> "TodoSchema":
        return TodoSchema(
            id=str(todo.id.value),
            title=todo.title.value,
            description=todo.description.value if todo.description else "",
            status=todo.status.value,
            created_at=int(todo.created_at.timestamp() * 1000),
            updated_at=int(todo.updated_at.timestamp() * 1000),
            completed_at=int(todo.completed_at.timestamp() * 1000)
            if todo.completed_at
            else None,
        )
```

## 3) OpenAPI error documentation with typed payloads

In `dddpy`, error payloads are also Pydantic models so your OpenAPI schema is accurate.

```python
from pydantic import BaseModel, Field
from yourapp.domain.todo.exceptions import TodoNotFoundError


class ErrorMessageTodoNotFound(BaseModel):
    detail: str = Field(examples=[TodoNotFoundError.message])
```

Then reference it from routes:

```python
responses={status.HTTP_404_NOT_FOUND: {"model": ErrorMessageTodoNotFound}}
```

## 4) Common pitfalls

- Returning ORM models (DTOs) directly from handlers (leaks Infrastructure details).
- Putting “business validation” in Pydantic schemas only (Domain should enforce invariants).
- Catching `Exception` everywhere (prefer catching expected domain errors; let unexpected errors surface as 500 with logging/middleware).

