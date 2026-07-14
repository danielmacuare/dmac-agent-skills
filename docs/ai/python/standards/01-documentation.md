# Coding Standards: Documentation

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: Apply these rules when creating or modifying Python functions, modules, classes, or architecture docs.

## 1. Google-Style Docstrings

* All public functions, classes, and methods MUST include a Google-style docstring.
* Docstrings must include explicit sections for `Args:`, `Returns:`, and `Raises:` if applicable.

## 2. Module-Level Documentation

* Every new Python module (`.py` file) must begin with a high-level string explaining its architectural purpose and relationship to the wider system.

## 3. Example

```python
def process_bgp_peer(peer_ip: str) -> bool:
    """Evaluates the readiness of a specific BGP neighbor.

    Args:
        peer_ip: The literal IPv4 address of the target neighbor.

    Returns:
        True if the peer is ready to establish, False otherwise.

    Raises:
        ValueError: If the peer_ip is an invalid IPv4 format.
    """
    return True
```

## 4. Others

- Keep comments up-to-date with code changes
- Include examples in docstrings for complex functions
