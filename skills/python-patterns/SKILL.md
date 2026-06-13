---
name: python-patterns
description: Use when writing, reviewing, or refactoring Python code for context-aware idioms, type hints, packaging conventions, logging choices, error handling, concurrency, and maintainability. Not for test-strategy or toolchain setup unless those are secondary to code design.
---

# Python Development Patterns

Use this skill when writing, reviewing, or refactoring Python. Treat these as
strong defaults, not laws: follow the local codebase when it is coherent, and
prefer small, explicit code over clever abstractions.

## Core Principles

- Optimize first for readable control flow, clear names, and simple data shapes.
- Keep import-time behavior boring: define objects, avoid I/O, network calls, and
  global configuration outside the application entry point.
- Make invalid states hard to represent at API boundaries; keep internal code
  direct once inputs are validated.
- Prefer standard-library features and existing project dependencies before
  adding new packages.
- Measure before optimizing. Use performance patterns when data size, latency, or
  memory pressure makes them relevant.

## Type Hints

Annotate public function signatures, dataclasses, protocols, and complex return
types. Avoid noisy annotations for obvious locals unless they clarify intent.

Import collection interfaces from `collections.abc`, not `typing`, when the
runtime ABC exists: `Iterable`, `Iterator`, `Sequence`, `Mapping`,
`MutableMapping`, and `Callable`. Reserve `typing` for type-system constructs
such as `Protocol`, `TypedDict`, `Literal`, `overload`, and `Self`.

Use modern built-in generics and union syntax. On Python 3.12+, prefer PEP
695 type aliases and type parameters:

```python
from collections.abc import Mapping, Sequence

type JSON = (
    dict[str, JSON]
    | list[JSON]
    | str
    | int
    | float
    | bool
    | None
)

def first[T](items: Sequence[T]) -> T | None:
    return items[0] if items else None

def normalize_scores(scores: Mapping[str, float]) -> dict[str, float]:
    total = sum(scores.values())
    if total == 0:
        return {name: 0.0 for name in scores}
    return {name: score / total for name, score in scores.items()}
```

On projects below Python 3.12, use `JSON: TypeAlias = ...`,
`T = TypeVar("T")`, and `def first(items: Sequence[T]) -> T | None`.

Prefer abstract input types when callers can pass many concrete containers:

```python
from collections.abc import Iterable

def send_all(messages: Iterable[str]) -> None:
    for message in messages:
        send(message)
```

Return concrete types when the function constructs and owns the result.

### Protocols

Use protocols for structural interfaces instead of inheritance when behavior is
all that matters.

```python
from collections.abc import Iterable
from typing import Protocol

class Renderable(Protocol):
    def render(self) -> str: ...

def render_all(items: Iterable[Renderable]) -> str:
    return "\n".join(item.render() for item in items)
```

### Decorator Typing

Use parameter syntax for decorators that preserve a wrapped function's
signature.

```python
from collections.abc import Callable
from functools import wraps

def traced[**P, R](func: Callable[P, R]) -> Callable[P, R]:
    @wraps(func)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        logger.debug("calling %s", func.__name__)
        return func(*args, **kwargs)

    return wrapper
```

On projects below Python 3.12, import `ParamSpec` and `TypeVar`, then declare
`P = ParamSpec("P")` and `R = TypeVar("R")` before the decorator.

## Error Handling

- Catch the narrowest exception that lets you recover or add useful context.
- Let unexpected exceptions propagate at low levels; translate them at system
  boundaries into domain errors, HTTP responses, CLI exit codes, or user-facing
  messages.
- Chain exceptions with `raise NewError(...) from exc` when wrapping.
- Avoid `except Exception` except at process, task, or request boundaries where
  logging, cleanup, or isolation is required.
- Do not log and re-raise everywhere. That creates duplicate noisy logs; log at a
  boundary that has request/job/user context.

```python
from pathlib import Path
import json

class ConfigError(Exception):
    """Raised when configuration cannot be loaded."""

def load_config(path: Path) -> Config:
    try:
        raw = path.read_text(encoding="utf-8")
        return Config.from_json(raw)
    except FileNotFoundError as exc:
        raise ConfigError(f"Config file not found: {path}") from exc
    except json.JSONDecodeError as exc:
        raise ConfigError(f"Invalid JSON in config file: {path}") from exc
```

EAFP is useful for operations where pre-checks can race or duplicate work:

```python
def read_optional_text(path: Path) -> str:
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        return ""
```

Use explicit validation when rejecting bad user input or public API arguments:

```python
def set_retries(count: int) -> None:
    if count < 0:
        raise ValueError("count must be non-negative")
```

## Logging

For libraries, use stdlib `logging` and never configure handlers at import time.
Consumers must control output.

```python
import logging

logger = logging.getLogger(__name__)

def import_user(user_id: str) -> None:
    logger.info("importing user %s", user_id)
```

For applications and services you own end-to-end, `loguru` is acceptable when the
project has chosen it. Configure sinks once at startup, not inside library
modules.

```python
import sys
from loguru import logger

def configure_logging() -> None:
    logger.remove()
    logger.add(sys.stderr, level="INFO")
    logger.add("app.log", rotation="10 MB", retention="10 days", serialize=True)
```

Prefer structured context over interpolating everything into the message:

```python
log = logger.bind(request_id=request_id, user_id=user_id)
log.info("request received")
```

## Files, Paths, and Resources

Use `pathlib.Path` for path manipulation and specify encodings for text files.

```python
from pathlib import Path

def load_template(path: Path) -> str:
    return path.read_text(encoding="utf-8")

def write_report(path: Path, content: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(content, encoding="utf-8")
```

Use context managers for resources with lifetimes:

```python
from contextlib import contextmanager
from time import perf_counter
from collections.abc import Iterator

@contextmanager
def timer(name: str) -> Iterator[None]:
    start = perf_counter()
    try:
        yield
    finally:
        logger.info("%s took %.3fs", name, perf_counter() - start)
```

## Comprehensions and Iteration

Use comprehensions for simple mapping and filtering. Expand into a loop when the
conditions, side effects, error handling, or naming become non-trivial.

```python
names = [user.name for user in users if user.is_active]

def active_even_doubled(items: Iterable[int]) -> list[int]:
    result: list[int] = []
    for item in items:
        if item <= 0:
            continue
        if item % 2 != 0:
            continue
        result.append(item * 2)
    return result
```

Use generator expressions for streaming into consumers:

```python
total = sum(item.cost for item in invoice.items)
```

Use generator functions when the caller should process data lazily:

```python
from collections.abc import Iterator

def iter_lines(path: Path) -> Iterator[str]:
    with path.open(encoding="utf-8") as file:
        for line in file:
            yield line.rstrip("\n")
```

## Data Modeling

Use dataclasses for plain data with light behavior. Consider `frozen=True` for
value objects and `slots=True` when many instances make memory relevant.

```python
from dataclasses import dataclass, field
from datetime import UTC, datetime

@dataclass(frozen=True, slots=True)
class UserId:
    value: str

    def __post_init__(self) -> None:
        if not self.value:
            raise ValueError("user id must not be empty")

@dataclass
class User:
    id: UserId
    email: str
    created_at: datetime = field(default_factory=lambda: datetime.now(UTC))
```

Use `timezone.utc` instead of `UTC` only when supporting Python earlier than
3.11.

Use `NamedTuple` or a frozen dataclass for small immutable records. Use
validation libraries already present in the project, such as Pydantic or attrs,
when parsing untrusted input, enforcing schemas, or serializing external data.

## Package Organization

Prefer `src/` layout for reusable packages to avoid accidentally importing from
the repository root. A flat layout is fine for small applications and scripts.

```text
myproject/
├── pyproject.toml
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── api.py
│       └── models.py
└── tests/
    ├── conftest.py
    └── test_api.py
```

Import order should be stdlib, third-party, then local imports. Let ruff/isort
apply the exact formatting when the project uses them.

```python
from pathlib import Path

import requests

from mypackage.models import User
```

Keep `__init__.py` light. Re-export stable public APIs deliberately; avoid heavy
imports, I/O, and framework setup there.

```python
"""Public API for mypackage."""

from mypackage.models import User

__all__ = ["User"]
```

## Concurrency

Choose the concurrency model based on the bottleneck and surrounding framework:

- Threads: blocking I/O with sync libraries.
- Async: high-concurrency I/O when the stack is already async.
- Processes: CPU-bound work that can be pickled and split into independent jobs.

Add timeouts and bounds. Unbounded concurrency is a production bug.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from urllib.request import urlopen

def fetch_url(url: str, timeout: float = 5.0) -> bytes:
    with urlopen(url, timeout=timeout) as response:
        return response.read()

def fetch_all(urls: list[str]) -> dict[str, bytes]:
    results: dict[str, bytes] = {}
    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = {executor.submit(fetch_url, url): url for url in urls}
        for future in as_completed(futures):
            url = futures[future]
            results[url] = future.result()
    return results
```

For `ProcessPoolExecutor`, keep worker functions top-level and use the
`if __name__ == "__main__"` guard in scripts, especially on macOS and Windows.

For async code, preserve cancellation and use timeouts from the async library or
`asyncio.timeout` where supported.

## Performance and Memory

- Prefer clear code until profiling shows a real bottleneck.
- Use generators for large or streaming data, but return lists when callers need
  indexing, repeated iteration, or a stable snapshot.
- Use `str.join()` or `io.StringIO` for large string assembly.
- Use `dataclass(slots=True)` or `__slots__` only when many instances make the
  memory tradeoff worthwhile.
- Avoid quadratic loops over large collections; choose dictionaries, sets, or
  sorted data structures when lookup patterns demand them.

```python
result = "".join(str(item) for item in items)

seen = {user.id for user in users}
missing = [user_id for user_id in requested_ids if user_id not in seen]
```

## Correct Patterns for Common Pitfalls

The examples below are the preferred forms, not examples to avoid.

```python
# Mutable default argument
def append_to(item: str, items: list[str] | None = None) -> list[str]:
    if items is None:
        items = []
    items.append(item)
    return items

# Prefer isinstance to exact type checks when subclasses are acceptable
if isinstance(value, list):
    process(value)

# Use identity checks for None
if value is None:
    process()

# Prefer explicit imports
from os.path import exists, join

# Catch the exception you can handle
try:
    risky_operation()
except SpecificError as exc:
    logger.warning("operation failed: %s", exc)
```

## Review Checklist

- Does the code match local conventions and avoid surprising import-time work?
- Are public APIs typed, with concrete returns and flexible input types where
  useful?
- Are exceptions specific, contextual, and logged at the right boundary?
- Is logging configured only by the application entry point?
- Are files opened with context managers and explicit text encodings?
- Are comprehensions, decorators, dataclasses, and concurrency used only where
  they make the code clearer or materially better?
- Has performance-sensitive code been measured or justified by data size?
