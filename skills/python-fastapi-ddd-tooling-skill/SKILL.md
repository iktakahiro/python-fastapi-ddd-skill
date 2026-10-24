---
name: python-fastapi-ddd-tooling-skill
description: Guides project tooling for a Python FastAPI + SQLAlchemy DDD/Onion Architecture codebase: uv-based environment setup, Makefile workflows, ruff formatting/linting, Pyrefly typing, pytest, and CI (GitHub Actions), based on the dddpy reference. Use when bootstrapping a repo or tightening developer experience and quality gates.
license: Apache-2.0
metadata:
  author: Takahiro Ikeuchi
  version: "1.0.0"
---

# Tooling & DX for FastAPI DDD Projects (uv / ruff / Pyrefly / CI)

This skill is about **developer experience and quality automation** for a DDD FastAPI project, following the conventions in `dddpy`.

## Goals

- One-command local setup (`make install`)
- Fast feedback loop (`make test`, `make format`, `make dev`)
- Reproducible CI (GitHub Actions + Python matrix)

## Recommended stack (dddpy-style)

- Python `>=3.13`
- `uv>=0.13.4` for lockfile-based dependency sync + running tools
- `ruff` for formatting/linting
- `Pyrefly` for type checking
- `pytest` for tests

## Makefile workflow (core targets)

Keep a small Makefile that shells out to tools inside `.venv/`:

- `sync`: run `uv sync --frozen --extra dev` (creates `.venv`)
- `install`: alias for `sync`
- `lint`: run ruff checks
- `typecheck`: run Pyrefly
- `test`: run typecheck + pytest
- `format`: run ruff formatter
- `dev`: run `fastapi dev` via uv

## CI workflow

Run `make install`, `make lint`, and `make test` on push/PR with a Python version matrix (e.g., 3.13, 3.14). Pin actions to full commit SHAs, use `permissions: contents: read` and checkout with `persist-credentials: false`, and install an exact patched uv version via `astral-sh/setup-uv`. Set `UV_PYTHON` from the matrix.

## Dependency safety

Commit `uv.lock`; regenerate it deliberately with `uv lock` when dependencies change. Require uv `>=0.13.4` in `[tool.uv]` and set `exclude-newer = "1 week"`. For the 2026-10-20 emSign CA distrust, also require pip `>=26.2.2` wherever pip is used and certifi `>=2026.10.19` wherever present. Security updates may need a temporary, package-specific cooldown exception; review and commit the resulting lockfile. Keep exact uv pins consistent across CI, `.mise.toml`, and devcontainer images. See the reference for advisory links and examples.

## App bootstrap patterns

- Use FastAPI `lifespan` to create tables on startup and dispose the engine on shutdown.
- Keep SQLAlchemy engine/session setup in `infrastructure/sqlite/database.py` (or equivalent).
- Configure logging once at startup (`logging.config.fileConfig(...)`).

For concrete file templates (Makefile, workflow YAML, bootstrap snippets), read `references/TOOLING.md`.

