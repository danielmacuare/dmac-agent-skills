# Coding Standards: Tooling & Execution

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: Apply prior to saving files or executing test runs.

## 1. Linting & Formatting (Ruff)

* Adhere strictly to the settings verified by `ruff`. Avoid trailing whitespaces, unreferenced imports, and redundant variable definitions.
* Let the language server handle automatic formatting via Ruff to conform with modern PEP 8 configurations.

## 2. Static Analysis & Testing

* Ensure code completely clears `basedpyright` strict typing thresholds before finalizing execution steps.
* Every feature change or implementation must be backed by a deterministic integration or unit test suite using `pytest`.
