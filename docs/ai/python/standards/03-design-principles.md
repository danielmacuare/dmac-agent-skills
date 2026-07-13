# Coding Standards: Design Principles

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: Apply during architectural refactoring, structural planning, and file layout creations.

## 1. KISS & DRY Alignment

* Prioritize readability over hyper-clever abstractions. Code should be straightforward to trace line by line.
* Avoid code duplication by isolating operational components into small, single-purpose utilities.

## 2. Network Isolation (Modular Architecture)

* Isolate structural definitions (e.g., Pydantic models) from dynamic runners (e.g., Nornir, Scrapli tasks) to prevent circular imports.
* Do not write execution logic directly inside package constructors (`__init__.py`). Keep front-door files limited to metadata, package exports, and API hoisting.
