# Coding Standards: Type Hinting

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: Apply to every single Python file creation or modification. This rule group is non-negotiable.

## 1. Modern Typings (Python 3.12+)

* Use built-in generics directly (`list`, `dict`, `set`) instead of importing capitalized variants from the legacy `typing` module.
* Use the native pipe operator (`|`) for union and optional evaluations. Never import `Union` or `Optional`.

## 2. Explicit Signatures

* Every function or method signature must explicitly type all inputs and declare a return type (e.g., use `-> None` if nothing is returned).
* Use `Any` only as an absolute last resort. If a variable can hold multiple defined shapes, use an explicit type union.
