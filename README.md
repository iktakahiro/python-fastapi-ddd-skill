# python-fastapi-ddd-skill

A Codex skill for designing **FastAPI** backends in Python using **Domain-Driven Design (DDD)** and **Onion Architecture** patterns.

This skill is based on the reference implementation:

- `dddpy`: https://github.com/iktakahiro/dddpy

## Install

This repository contains the skill at `skills/python-fastapi-ddd-skill/`.

### Option A: Install via Codex skill installer

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo iktakahiro/python-fastapi-ddd-skill \
  --path skills/python-fastapi-ddd-skill
```

Then restart Codex to pick up the new skill.

### Option B: Manual install (copy the folder)

```bash
dest="${CODEX_HOME:-$HOME/.codex}/skills"
mkdir -p "$dest"
cp -R skills/python-fastapi-ddd-skill "$dest/"
```

Then restart Codex to pick up the new skill.

## Usage

In Codex, you can prompt like:

> Use $python-fastapi-ddd-skill to design a FastAPI + SQLAlchemy service with DDD and Onion Architecture.

