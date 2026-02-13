---
name: Python Mastery
description: This skill should be used when the user asks to write or review "Python class", "Python function", "type hints", "dataclass", "Python decorator", "context manager", or "Python protocol". It covers modern Python 3.10+ practices including the full type system, advanced class design, generator patterns, and async/await concurrency.
---

## Modern Python Type System (3.10+)

### Union Syntax and Optional

Use the pipe operator for union types instead of `typing.Union`:

```python
# Modern (3.10+)
def process(value: int | str | None) -> str:
    match value:
        case int():
            return str(value)
        case str():
            return value
        case None:
            return ""
```

Never write `Optional[X]` in new code. Write `X | None` instead.

### Generics with TypeVar and ParamSpec

Define generic classes and functions with bounded or constrained type variables:

```python
from typing import TypeVar, Protocol

T = TypeVar("T")
T_co = TypeVar("T_co", covariant=True)

class Repository(Protocol[T]):
    def get(self, id: str) -> T | None: ...
    def save(self, entity: T) -> None: ...
    def delete(self, id: str) -> bool: ...
```

For decorators that preserve signatures, use `ParamSpec`:

```python
from typing import ParamSpec, TypeVar, Callable
from functools import wraps

P = ParamSpec("P")
R = TypeVar("R")

def retry(attempts: int = 3) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            for attempt in range(attempts):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if attempt == attempts - 1:
                        raise
            raise RuntimeError("unreachable")
        return wrapper
    return decorator
```

### Protocol Classes (PEP 544)

Prefer `Protocol` over ABC when the interface is small and structural subtyping is desirable:

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Renderable(Protocol):
    def render(self) -> str: ...

class MarkdownDoc:
    def render(self) -> str:
        return "# Hello"

def display(item: Renderable) -> None:
    print(item.render())

# Works without explicit inheritance
display(MarkdownDoc())
```

Use `@runtime_checkable` only when `isinstance` checks are genuinely needed; it adds overhead.

## Dataclasses and attrs

### Dataclass Best Practices

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(frozen=True, slots=True)
class Event:
    name: str
    timestamp: datetime
    tags: list[str] = field(default_factory=list)
    _id: str = field(init=False, repr=False)

    def __post_init__(self) -> None:
        object.__setattr__(self, "_id", f"{self.name}:{self.timestamp.isoformat()}")
```

Always use `frozen=True` for value objects. Add `slots=True` (3.10+) for memory efficiency and faster attribute access. Use `field(default_factory=...)` for mutable defaults.

### When to Use attrs Instead

Reach for attrs when needing validators, converters, or more complex initialization logic:

```python
import attrs

@attrs.define
class Temperature:
    celsius: float = attrs.field(validator=attrs.validators.ge(-273.15))

    @property
    def fahrenheit(self) -> float:
        return self.celsius * 9 / 5 + 32
```

## Generators and Iterators

### Generator Functions

Use generators for lazy evaluation of sequences, especially when working with large datasets:

```python
from collections.abc import Generator

def read_chunks(path: str, size: int = 8192) -> Generator[bytes, None, None]:
    with open(path, "rb") as f:
        while chunk := f.read(size):
            yield chunk

def pipeline(path: str) -> Generator[str, None, int]:
    count = 0
    for chunk in read_chunks(path):
        for line in chunk.decode().splitlines():
            yield line.strip()
            count += 1
    return count
```

### Generator-Based Context Managers

Use `contextlib.contextmanager` to write context managers as generators:

```python
from contextlib import contextmanager
from collections.abc import Generator
import time

@contextmanager
def timer(label: str) -> Generator[None, None, None]:
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{label}: {elapsed:.4f}s")
```

## Context Managers

### Class-Based Context Managers

For context managers that need to hold state across entry and exit:

```python
from types import TracebackType

class DatabaseTransaction:
    def __init__(self, connection: "Connection") -> None:
        self._conn = connection
        self._savepoint: str | None = None

    def __enter__(self) -> "DatabaseTransaction":
        self._savepoint = self._conn.create_savepoint()
        return self

    def __exit__(
        self,
        exc_type: type[BaseException] | None,
        exc_val: BaseException | None,
        exc_tb: TracebackType | None,
    ) -> bool:
        if exc_type is not None:
            self._conn.rollback_to(self._savepoint)
            return False  # re-raise
        self._conn.release_savepoint(self._savepoint)
        return False
```

### Async Context Managers

```python
from contextlib import asynccontextmanager
from collections.abc import AsyncGenerator

@asynccontextmanager
async def managed_session(url: str) -> AsyncGenerator["Session", None]:
    session = await create_session(url)
    try:
        yield session
    finally:
        await session.close()
```

## Decorators

### Decorator Patterns by Complexity

**Simple function decorator** (no arguments):

```python
from functools import wraps
from typing import Callable, TypeVar

R = TypeVar("R")

def log_calls(func: Callable[..., R]) -> Callable[..., R]:
    @wraps(func)
    def wrapper(*args: object, **kwargs: object) -> R:
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```

**Decorator with arguments** (decorator factory):

```python
def rate_limit(max_calls: int, period: float) -> Callable[[Callable[P, R]], Callable[P, R]]:
    def decorator(func: Callable[P, R]) -> Callable[P, R]:
        calls: list[float] = []

        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            now = time.time()
            calls[:] = [t for t in calls if now - t < period]
            if len(calls) >= max_calls:
                raise RuntimeError(f"Rate limit exceeded: {max_calls}/{period}s")
            calls.append(now)
            return func(*args, **kwargs)
        return wrapper
    return decorator
```

**Class-based decorator** (for stateful decorators):

```python
class CacheResult:
    def __init__(self, ttl: float = 60.0) -> None:
        self._ttl = ttl
        self._cache: dict[str, tuple[float, object]] = {}

    def __call__(self, func: Callable[P, R]) -> Callable[P, R]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
            key = f"{args}:{kwargs}"
            now = time.time()
            if key in self._cache:
                cached_time, value = self._cache[key]
                if now - cached_time < self._ttl:
                    return value  # type: ignore[return-value]
            result = func(*args, **kwargs)
            self._cache[key] = (now, result)
            return result
        return wrapper
```

## Async/Await Concurrency

### Structured Concurrency with TaskGroup (3.11+)

```python
import asyncio

async def fetch_all(urls: list[str]) -> list[str]:
    results: list[str] = []

    async with asyncio.TaskGroup() as tg:
        async def _fetch(url: str) -> None:
            result = await fetch(url)
            results.append(result)

        for url in urls:
            tg.create_task(_fetch(url))

    return results
```

### Semaphore-Based Concurrency Limits

```python
async def fetch_with_limit(urls: list[str], max_concurrent: int = 10) -> list[str]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async def _limited_fetch(url: str) -> str:
        async with semaphore:
            return await fetch(url)

    async with asyncio.TaskGroup() as tg:
        tasks = [tg.create_task(_limited_fetch(url)) for url in urls]

    return [task.result() for task in tasks]
```

### Async Generators

```python
from collections.abc import AsyncGenerator

async def stream_lines(path: str) -> AsyncGenerator[str, None]:
    async with aiofiles.open(path) as f:
        async for line in f:
            yield line.strip()
```

## Match/Case (Structural Pattern Matching, 3.10+)

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

def describe(shape: object) -> str:
    match shape:
        case Point(x=0, y=0):
            return "origin"
        case Point(x, y) if x == y:
            return f"diagonal at {x}"
        case Point(x, y):
            return f"point({x}, {y})"
        case list() as items if len(items) > 3:
            return f"long list of {len(items)}"
        case _:
            return "unknown"
```

## Descriptor Protocol

Implement custom descriptors for reusable attribute behavior:

```python
from typing import Any, overload

class Validated:
    def __init__(self, min_value: float, max_value: float) -> None:
        self._min = min_value
        self._max = max_value
        self._name = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self._name = name

    @overload
    def __get__(self, obj: None, objtype: type) -> "Validated": ...
    @overload
    def __get__(self, obj: object, objtype: type) -> float: ...

    def __get__(self, obj: object | None, objtype: type) -> "Validated | float":
        if obj is None:
            return self
        return getattr(obj, f"_{self._name}")

    def __set__(self, obj: object, value: float) -> None:
        if not self._min <= value <= self._max:
            raise ValueError(f"{self._name} must be between {self._min} and {self._max}")
        setattr(obj, f"_{self._name}", value)
```

## References

- [SOLID Patterns in Python](references/solid-patterns.md) -- Python-specific implementations of SOLID principles using ABCs, Protocols, and Pythonic idioms.
- [Python Best Practices](references/best-practices.md) -- PEP conventions, Pythonic idioms, performance tips, security, and Python 3.12+ features.
