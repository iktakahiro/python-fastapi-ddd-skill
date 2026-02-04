# Tooling Templates (dddpy-based)

This reference provides copy-friendly templates aligned with `dddpy`.

## 1) `pyproject.toml` essentials

Keep runtime deps minimal and move dev tooling into optional dependencies.

```toml
[project]
name = "yourapp"
version = "0.1.0"
description = "FastAPI + SQLAlchemy DDD app"
dependencies = [
  "fastapi[standard]==<pin>",
  "sqlalchemy==<pin>",
]
requires-python = ">=3.13"

[project.optional-dependencies]
dev = [
  "pytest>=<pin>",
  "mypy>=<pin>",
  "ruff>=<pin>",
]
```

### Ruff config (example)

```toml
[tool.ruff]
line-length = 88
target-version = "py313"

[tool.ruff.format]
quote-style = "single"
docstring-code-format = true
```

## 2) Makefile (uv + venv + common tasks)

`dddpy` uses `.venv` and runs tools from it.

```make
VENV=.venv
PYTEST=$(VENV)/bin/pytest
MYPY=$(VENV)/bin/mypy --ignore-missing-imports
RUFF=$(VENV)/bin/ruff
PACKAGE=yourapp

venv:
	uv venv .venv

install: venv
	uv pip install -e ".[dev]"

test: install
	$(MYPY) main.py ./${PACKAGE}/
	$(PYTEST) -vv

format:
	$(RUFF) format

dev: install
	uv run fastapi dev
```

## 3) GitHub Actions CI (`.github/workflows/test.yaml`)

Run tests on push + PR. Install `uv` in CI and reuse Makefile targets.

```yaml
name: A workflow to run tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: [3.13, 3.14]
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install uv
        run: |
          curl -LsSf https://astral.sh/uv/install.sh | sh
      - name: Install dependencies
        run: |
          make install
      - name: Run tests
        run: |
          make test
```

## 4) App bootstrap (`main.py`) and FastAPI lifespan

Use lifespan to do setup/teardown without leaking DB initialization into route handlers.

```python
import logging
from contextlib import asynccontextmanager
from logging import config

from fastapi import FastAPI

from yourapp.infrastructure.sqlite.database import create_tables, engine
from yourapp.presentation.api.todo.handlers.todo_api_route_handler import (
    TodoApiRouteHandler,
)

config.fileConfig("logging.conf", disable_existing_loggers=False)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    create_tables()
    yield
    engine.dispose()


app = FastAPI(title="Your API", lifespan=lifespan)

TodoApiRouteHandler().register_routes(app)
```

## 5) SQLAlchemy bootstrap (SQLite example)

Keep DB wiring in Infrastructure.

```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

SQLALCHEMY_DATABASE_URL = "sqlite:///./db/sqlite.db"

engine = create_engine(
    SQLALCHEMY_DATABASE_URL,
    connect_args={"check_same_thread": False},
)
SessionLocal = sessionmaker(bind=engine, autoflush=True)

Base = declarative_base()

def create_tables() -> None:
    Base.metadata.create_all(bind=engine)
```

