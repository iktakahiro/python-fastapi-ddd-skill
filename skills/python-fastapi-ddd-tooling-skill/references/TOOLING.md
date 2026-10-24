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
  "pyrefly>=<pin>",
  "ruff>=<pin>",
]
```

### Lockfile and security policy

Replace `<pin>` with reviewed versions, then run `uv lock` and commit `uv.lock` alongside `pyproject.toml`. Normal setup and CI use frozen sync; dependency updates explicitly regenerate and review the lockfile.

```toml
[tool.uv]
required-version = ">=0.13.4"
package = false
exclude-newer = "1 week"
# Temporary security exception; remove once the cooldown has passed and relock.
exclude-newer-package = { certifi = "2026-10-24T00:00:00Z" }
constraint-dependencies = ["certifi>=2026.10.19"]

[tool.pyrefly]
project_includes = ["main.py", "yourapp"]
```

`package = false` matches dddpy's application workflow; projects that install their own package should configure a build backend and omit that setting. The certifi constraint enforces the security floor when certifi is transitive without adding a runtime dependency. Advance the temporary exception deliberately if a newer security release is needed.

For the 2026-10-20 emSign CA distrust, use uv `>=0.13.4` ([uv advisory](https://github.com/astral-sh/uv/security/advisories/GHSA-hq85-5cpj-mr62)), pip `>=26.2.2` wherever pip is used ([pip changelog](https://pip.pypa.io/en/stable/news/#v26-2-2)), and certifi `>=2026.10.19` wherever present ([certifi advisory](https://github.com/certifi/python-certifi/security/advisories/GHSA-6q48-6c7w-9gw2)). The certifi advisory lists 2026.10.18 as patched; this template uses the stricter 2026.10.19 floor also vendored by patched pip. Upgrading standalone certifi does not update pip's vendored bundle. Frozen sync preserves an existing lockfile, so review its resolved certifi version too.

### Ruff config (example)

```toml
[tool.ruff]
line-length = 88
target-version = "py313"

[tool.ruff.format]
quote-style = "single"
docstring-code-format = true
```

## 2) Makefile (uv + common tasks)

Frozen sync creates `.venv` and installs the locked dev extra; a separate `uv venv` or editable pip install is unnecessary.

```make
VENV=.venv
PYTHON=$(VENV)/bin/python
PYTEST=$(PYTHON) -m pytest
PYREFLY=$(VENV)/bin/pyrefly
RUFF=$(VENV)/bin/ruff
PYREFLY_FLAGS=--summarize-errors
RUFF_FLAGS=

.PHONY: sync install lint typecheck test format dev

sync:
	uv sync --frozen --extra dev

install: sync

lint: install
	$(RUFF) check . $(RUFF_FLAGS)

typecheck: install
	$(PYREFLY) check $(PYREFLY_FLAGS)

test: typecheck
	$(PYTEST) -vv

format: install
	$(RUFF) format

dev: install
	uv run fastapi dev
```

For optional mise support, pin Python, uv, and ruff in `.mise.toml` and have tasks mirror these targets. dddpy currently pins Python `3.14`, uv `0.13.5`, and ruff `0.17.0`. Keep uv's exact pin consistent with CI and the devcontainer. Its devcontainer copies `/uv` and `/uvx` from `ghcr.io/astral-sh/uv:0.13.5` into Debian bookworm, installs `ca-certificates` and `make`, and runs `make install` after creation. Use a reviewed patched uv image, optionally pinned by digest; avoid unpinned installer scripts. If seeding pip separately, verify pip meets the security floor above.

## 3) GitHub Actions CI (`.github/workflows/test.yaml`)

Run lint, type checking, and tests on push + PR. Replace each `<...-release-sha>` with the full 40-character commit SHA of a reviewed current release before use, and retain its version in a comment. Placeholders demonstrate SHA pinning without freezing stale action versions into the template. Update the exact uv pin deliberately across all environments.

```yaml
name: A workflow to run tests

on: [push, pull_request]

permissions:
  contents: read

env:
  UV_VERSION: "0.13.5"

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.13", "3.14"]
    env:
      UV_PYTHON: ${{ matrix.python-version }}
    steps:
      # Pin each current release to its full commit SHA; add a version comment.
      - uses: actions/checkout@<checkout-release-sha>
        with:
          persist-credentials: false
      - uses: actions/setup-python@<setup-python-release-sha>
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install uv
        uses: astral-sh/setup-uv@<setup-uv-release-sha>
        with:
          version: ${{ env.UV_VERSION }}
          enable-cache: false
      - name: Install dependencies
        run: make install
      - name: Run lint
        run: make lint RUFF_FLAGS=--output-format=github
      - name: Run tests
        run: make test PYREFLY_FLAGS=--output-format=github
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

