# Python Coding standards

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: This file applies EXCLUSIVELY to files ending in `.py`.
> If the current task does not involve creating, editing, or refactoring Python files, skip these rules.

## Categories

Read only the file(s) matching your current task:

- `standards/01-documentation.md` — creating/modifying functions, classes, modules, or architecture docs
- `standards/02-data-validation.md` — defining data models, parsing network state, importing config input
- `standards/03-design-principles.md` — architectural refactoring, structural planning, file layout
- `standards/04-type-hinting.md` — every `.py` file creation or modification (non-negotiable)
- `standards/05-tooling.md` — before saving files or running tests
- `standards/06-conventions.md` — HTTP calls, file path handling

## Stack

- Python3.12+
- uv for package management, python versions and and virtual environments
- ruff for linting/formatting (replaces black + flake8)
- pybasedright for type hinting
- Typer for CLI Options

### Project structure

- `src/[package]/` — main package code
- `tests/` — pytest tests (mirrors src/ structure)
- `scripts/` — one-off scripts, not production code
