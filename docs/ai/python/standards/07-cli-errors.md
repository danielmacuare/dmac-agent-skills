# Coding Standards: CLI Error Handling & Exception Routing

> [META-INSTRUCTION FOR AI AGENTS]
> Scope: Apply this standard whenever creating or modifying CLI entry points,
> Typer/Click commands, or user-facing terminal interfaces.

## 1. Core Principle

No raw Python traceback ever reaches the end-user terminal by default — for **any** exception, not just the ones you anticipated.

- **For the User:** Display a clean, one-line, context-rich error message.
- **For the Developer (Debug Mode):** Expose the full traceback stack only when a global `--debug` flag is explicitly provided.
- **For the Logs:** Write the complete traceback stack (including chained causes) to a persistent log file.

## 2. The Boundary Pattern

1. Exceptions in core business logic MUST use explicit chaining (`raise B from A`) to preserve debugging diagnostics.
2. **One** try/except boundary exists, and it wraps `app()` in the console-script entry point — not inside individual commands. A per-command boundary has to be repeated in every command, and any command that forgets it leaks a traceback.
3. The boundary catches two classes of failure and forks on the `--debug` flag:
   - `DomainError` — an expected, explainable failure. Exit code **1**.
   - `Exception` — an unanticipated bug. Exit code **2**. Still formatted, still logged, never a raw trace.

Commands themselves stay clean: they raise domain exceptions and let them propagate to the boundary.

## 3. Implementation Blueprint

```python
import sys
from pathlib import Path
from typing import Annotated

import typer
from rich.console import Console

from myapp.errors import DomainError  # Project-wide base exception
from myapp.logging import logger      # Pre-configured file logger

app = typer.Typer(add_completion=False)
err_console = Console(stderr=True)


class CLIState:
    """Global state for one CLI execution session."""

    def __init__(self) -> None:
        self.debug: bool = False


state = CLIState()


@app.callback()
def main_callback(
    debug: Annotated[
        bool,
        typer.Option("--debug", help="Show full traceback on failure."),
    ] = False,
) -> None:
    """Set global CLI configuration."""
    state.debug = debug


@app.command()
def process_data(
    source: Annotated[Path, typer.Argument(help="Path to the source data file.")],
) -> None:
    """Execute business logic. Raises domain errors; does not catch them."""
    if not source.exists():
        raise DomainError(f"Source file not found: {source}")
    # Standard execution steps here.


def main() -> None:
    """Console-script entry point. The single error boundary for the whole CLI."""
    try:
        app()
    except DomainError as error:
        _report(error, exit_code=1)
    except Exception as error:  # noqa: BLE001 — deliberate catch-all at the boundary
        _report(error, exit_code=2, unexpected=True)


def _report(error: Exception, exit_code: int, unexpected: bool = False) -> None:
    """Log the full trace, show the user a clean message, and exit."""
    logger.exception("Unhandled exception at the CLI boundary.")

    if state.debug:
        err_console.print_exception(show_locals=True)
    elif unexpected:
        err_console.print(f"[bold red]Internal error:[/] {error!r}")
        err_console.print("This is a bug. Re-run with --debug for the full trace.")
    else:
        err_console.print(f"[bold red]Error:[/] {error}")
        err_console.print("Re-run with --debug for the full trace.")

    raise SystemExit(exit_code)
```

## 4. Non-Negotiable Agent Constraints

- **Errors go to stderr, never stdout.** Never use `print()` for errors. Use `rich.Console(stderr=True)` so stdout stays pure for Unix piping. This follows `06-conventions.md`, which mandates `rich` for all terminal output.

- **Preserve `__cause__`.** Never break exception chaining. Source modules must raise with `raise CustomError(...) from original_error`.

- **Exit code precision.** `1` = an expected domain failure the user can act on. `2` = an unanticipated bug. Inside a Typer *command*, abort with `raise typer.Exit(code=...)` — never `sys.exit()`, which disrupts test-harness runners. At the `main()` boundary you are outside Typer's runner, so `raise SystemExit(...)` is correct there.

- **Never catch to silence.** A bare `except` that swallows an error without logging and re-raising is forbidden. The catch-all in `main()` is the one sanctioned broad catch in the codebase.
