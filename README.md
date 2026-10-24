# python-fastapi-ddd-skill

A collection of [Agent Skills](https://agentskills.io/) for designing **FastAPI** backends in Python using **Domain-Driven Design (DDD)** and **Onion Architecture** patterns.

This skill is based on the reference implementation:

- `dddpy`: https://github.com/iktakahiro/dddpy

## Skills

| Skill | Description |
| ----- | ----------- |
| `python-fastapi-ddd-skill` | Architecture + core DDD patterns (Entities, Value Objects, Repositories, UseCases) |
| `python-fastapi-ddd-presentation-skill` | FastAPI handlers, Pydantic schemas, error mapping |
| `python-fastapi-ddd-testing-skill` | pytest patterns for Domain / UseCase layers |
| `python-fastapi-ddd-tooling-skill` | uv / ruff / Pyrefly / pytest / CI templates |

## Install

### Claude Code (recommended)

Register this repository as a plugin marketplace, then install:

```bash
# 1. Register the marketplace
/plugin marketplace add iktakahiro/python-fastapi-ddd-skill

# 2. Install all skills at once
/plugin install python-fastapi-ddd@python-fastapi-ddd-skills
```

Or browse and install interactively:

```bash
/plugin
# → "Browse and install plugins"
# → "python-fastapi-ddd-skills"
# → "python-fastapi-ddd"
# → "Install now"
```

### OpenAI Codex

#### Via skill installer

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

#### Manual install (copy the folder)

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

Once installed, mention the skill in your prompt:

> Design a FastAPI + SQLAlchemy service with DDD and Onion Architecture.

> Implement FastAPI handlers/schemas that call UseCases and map Domain errors to HTTP responses.

> Add pytest unit tests for Value Objects and UseCases (mocking repository interfaces).

> Set up uv + Makefile + ruff/Pyrefly/pytest + GitHub Actions CI for this project.

## License

Apache-2.0
