# python-fastapi-ddd-skill

A collection of Codex skills for designing **FastAPI** backends in Python using **Domain-Driven Design (DDD)** and **Onion Architecture** patterns.

This skill is based on the reference implementation:

- `dddpy`: https://github.com/iktakahiro/dddpy

## Install

This repository contains skills under `skills/`:

- `skills/python-fastapi-ddd-skill/` (architecture + core DDD patterns)
- `skills/python-fastapi-ddd-presentation-skill/` (FastAPI handlers/schemas/error mapping)
- `skills/python-fastapi-ddd-testing-skill/` (pytest patterns for Domain/UseCase layers)
- `skills/python-fastapi-ddd-tooling-skill/` (uv/ruff/mypy/pytest/CI templates)

### Option A: Install via Codex skill installer

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo iktakahiro/python-fastapi-ddd-skill \
  --path skills/python-fastapi-ddd-skill
```

Install the other skills by changing `--path` to one of:

- `skills/python-fastapi-ddd-presentation-skill`
- `skills/python-fastapi-ddd-testing-skill`
- `skills/python-fastapi-ddd-tooling-skill`

Then restart Codex to pick up the new skill.

### Option B: Manual install (copy the folder)

```bash
dest="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$dest"
cp -R skills/python-fastapi-ddd-skill "$dest/"
```

You can also copy any of:

- `skills/python-fastapi-ddd-presentation-skill`
- `skills/python-fastapi-ddd-testing-skill`
- `skills/python-fastapi-ddd-tooling-skill`

Then restart Codex to pick up the new skill.

## Usage

In Codex, you can prompt like:

> Use $python-fastapi-ddd-skill to design a FastAPI + SQLAlchemy service with DDD and Onion Architecture.

Other examples:

> Use $python-fastapi-ddd-presentation-skill to implement FastAPI handlers/schemas that call UseCases and map Domain errors to HTTP responses.

> Use $python-fastapi-ddd-testing-skill to add pytest unit tests for Value Objects and UseCases (mocking repository interfaces).

> Use $python-fastapi-ddd-tooling-skill to set up uv + Makefile + ruff/mypy/pytest + GitHub Actions CI for this project.
