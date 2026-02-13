# Python Best Practices

Comprehensive reference covering PEP conventions, Pythonic idioms, performance optimization, security considerations, and modern Python 3.12+ features.

---

## PEP 8: Style Guide

### Naming Conventions

| Entity            | Convention          | Example                  |
|-------------------|---------------------|--------------------------|
| Module            | `snake_case`        | `data_loader.py`         |
| Class             | `PascalCase`        | `DataLoader`             |
| Function/Method   | `snake_case`        | `load_data()`            |
| Constant          | `UPPER_SNAKE_CASE`  | `MAX_RETRIES`            |
| Type Variable     | `PascalCase` or `T` | `T`, `KeyType`           |
| Private           | `_leading_underscore`| `_internal_cache`        |
| Name-mangled      | `__double_leading`  | `__secret` (rare, avoid) |

### Import Ordering

```python
# 1. Standard library
import os
import sys
from collections.abc import Sequence
from pathlib import Path

# 2. Third-party packages
import httpx
from pydantic import BaseModel

# 3. Local application imports
from myapp.core import settings
from myapp.models import User
```

Use `isort` with `profile = "black"` for automatic ordering.

### Line Length and Formatting

Use Black with default 88-character line length. Configure in `pyproject.toml`:

```toml
[tool.black]
target-version = ["py312"]

[tool.isort]
profile = "black"

[tool.ruff]
target-version = "py312"
line-length = 88
```

Ruff replaces flake8, isort, and many other linters. Prefer Ruff for new projects.

---

## PEP 484: Type Hints Best Practices

### Always Annotate Public APIs

```python
def fetch_users(
    limit: int = 100,
    offset: int = 0,
    *,
    active_only: bool = True,
) -> list["User"]:
    ...
```

### Use Modern Syntax (3.10+)

```python
# Preferred
def process(items: list[str | int]) -> dict[str, list[int]]: ...

# Avoid in new code
from typing import List, Dict, Union, Optional
def process(items: List[Union[str, int]]) -> Dict[str, List[int]]: ...
```

### TypeGuard for Type Narrowing (3.10+)

```python
from typing import TypeGuard

def is_string_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(item, str) for item in val)

def process(data: list[object]) -> None:
    if is_string_list(data):
        # data is now list[str]
        print(", ".join(data))
```

### TypeAlias and type Statement (3.12+)

```python
# 3.12+ syntax
type Vector = list[float]
type Matrix = list[Vector]
type Callback[T] = Callable[[T], None]

# Pre-3.12
from typing import TypeAlias
Vector: TypeAlias = list[float]
```

### Overload for Multiple Signatures

```python
from typing import overload

@overload
def get(key: str, default: None = None) -> str | None: ...
@overload
def get(key: str, default: str) -> str: ...

def get(key: str, default: str | None = None) -> str | None:
    result = _store.get(key)
    return result if result is not None else default
```

---

## Pythonic Idioms

### EAFP Over LBYL

```python
# EAFP (Easier to Ask Forgiveness than Permission) -- Pythonic
try:
    value = mapping[key]
except KeyError:
    value = default_value

# LBYL (Look Before You Leap) -- less Pythonic
if key in mapping:
    value = mapping[key]
else:
    value = default_value
```

### Comprehensions Over Loops for Transforms

```python
# Good: clear intent
squares = [x ** 2 for x in range(100) if x % 2 == 0]
lookup = {user.id: user for user in users}
unique_names = {name.lower() for name in raw_names}

# Bad: unnecessarily verbose
squares = []
for x in range(100):
    if x % 2 == 0:
        squares.append(x ** 2)
```

Do not write comprehensions longer than ~80 characters or with nested loops beyond one level. Use a generator expression or explicit loop instead.

### Context Managers for Resource Management

```python
# Always use context managers for files, locks, connections
from pathlib import Path

content = Path("data.txt").read_text(encoding="utf-8")

# For custom resources
from contextlib import contextmanager

@contextmanager
def managed_connection(url: str):
    conn = create_connection(url)
    try:
        yield conn
    finally:
        conn.close()
```

### Walrus Operator (:=) for Assignment Expressions

```python
# Process lines while reading
while line := input_stream.readline():
    process(line)

# Filter and capture
if (match := pattern.search(text)) is not None:
    handle(match.group(1))

# List comprehension with expensive computation
results = [
    cleaned
    for raw in data
    if (cleaned := expensive_clean(raw)) is not None
]
```

### Unpacking and Star Expressions

```python
first, *middle, last = scores
head, *_ = values  # Ignore remaining

# Dictionary merging (3.9+)
merged = base_config | override_config | {"debug": True}
```

### Enum for Fixed Sets of Values

```python
from enum import Enum, auto, StrEnum

class Status(StrEnum):
    PENDING = auto()
    ACTIVE = auto()
    ARCHIVED = auto()

# Usage in match/case
match user.status:
    case Status.PENDING:
        send_activation_email(user)
    case Status.ACTIVE:
        pass
    case Status.ARCHIVED:
        raise InactiveUserError(user.id)
```

---

## Performance Tips

### Use Appropriate Data Structures

| Operation            | `list`  | `deque` | `set`   | `dict`  |
|----------------------|---------|---------|---------|---------|
| Append               | O(1)*   | O(1)    | --      | --      |
| Prepend              | O(n)    | O(1)    | --      | --      |
| Lookup by value      | O(n)    | O(n)    | O(1)    | O(1)    |
| Lookup by index      | O(1)    | O(n)    | --      | --      |
| Delete by value      | O(n)    | O(n)    | O(1)    | O(1)    |

### Caching

```python
from functools import cache, lru_cache

@cache  # Unbounded cache (3.9+)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

@lru_cache(maxsize=256)  # Bounded cache
def expensive_lookup(key: str) -> dict[str, object]:
    ...
```

### Slots for Memory Efficiency

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Point:
    x: float
    y: float
    z: float

# ~40% less memory than without slots for millions of instances
```

### Generators for Memory-Efficient Pipelines

```python
def process_large_file(path: str) -> int:
    lines = (line.strip() for line in open(path))
    non_empty = (line for line in lines if line)
    parsed = (parse_record(line) for line in non_empty)
    return sum(1 for record in parsed if record.is_valid)
```

### String Joining Over Concatenation

```python
# Good: O(n)
result = "".join(parts)

# Bad: O(n^2) for repeated concatenation
result = ""
for part in parts:
    result += part
```

### `__init_subclass__` for Registration

```python
class PluginBase:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, *, name: str = "", **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if name:
            PluginBase._registry[name] = cls

class JSONPlugin(PluginBase, name="json"):
    ...

class YAMLPlugin(PluginBase, name="yaml"):
    ...
```

---

## Security Best Practices

### Never Use `eval` or `exec` with Untrusted Input

```python
# DANGEROUS
result = eval(user_input)

# SAFE: Use ast.literal_eval for literal structures
import ast
result = ast.literal_eval(user_input)  # Only allows literals
```

### Subprocess Safety

```python
import subprocess

# DANGEROUS: shell=True with user input
subprocess.run(f"grep {user_input} file.txt", shell=True)

# SAFE: Pass arguments as a list
subprocess.run(["grep", user_input, "file.txt"], check=True)
```

### Secrets Management

```python
import secrets
import os

# Generate tokens
token = secrets.token_urlsafe(32)

# Read secrets from environment
db_password = os.environ["DB_PASSWORD"]

# Compare secrets safely (constant-time)
secrets.compare_digest(provided_token, stored_token)
```

### Pickle Risks

Never unpickle data from untrusted sources. Use JSON, MessagePack, or Protocol Buffers instead.

```python
import json

# Safe serialization
data = json.loads(untrusted_input)

# If you must use pickle, sign with HMAC
import hmac
import pickle

def safe_dumps(obj: object, key: bytes) -> bytes:
    payload = pickle.dumps(obj)
    sig = hmac.new(key, payload, "sha256").digest()
    return sig + payload

def safe_loads(data: bytes, key: bytes) -> object:
    sig, payload = data[:32], data[32:]
    expected = hmac.new(key, payload, "sha256").digest()
    if not hmac.compare_digest(sig, expected):
        raise ValueError("Tampered data")
    return pickle.loads(payload)
```

### Dependency Auditing

```bash
# Use pip-audit for vulnerability scanning
pip-audit

# Use safety (now free for open source)
safety check

# Pin dependencies with hashes
pip install --require-hashes -r requirements.txt
```

---

## Python 3.12+ Features

### Type Parameter Syntax (PEP 695)

```python
# 3.12+ generic syntax -- no more TypeVar boilerplate
def first[T](items: list[T]) -> T:
    return items[0]

class Stack[T]:
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

type Point = tuple[float, float]
type Callback[T] = Callable[[T], None]
```

### Improved Error Messages

Python 3.12+ provides more helpful error messages:

```
# NameError now suggests corrections
>>> prnt("hello")
NameError: name 'prnt' is not defined. Did you mean: 'print'?

# ImportError suggestions
>>> from collections import OrderedDic
ImportError: cannot import name 'OrderedDic' from 'collections'. Did you mean: 'OrderedDict'?
```

### Per-Interpreter GIL (PEP 684, 3.12+)

```python
# Subinterpreters can now have independent GILs
# This enables true parallelism for CPU-bound Python code
# Still experimental -- use multiprocessing for production workloads
```

### f-string Improvements (3.12+)

```python
# Nested f-strings with quotes now allowed
name = "world"
message = f"Hello {f'{name!r}'}"

# Multi-line expressions in f-strings
result = f"{
    value
    if condition
    else default
}"
```

### `itertools.batched` (3.12+)

```python
from itertools import batched

# Process items in chunks
for batch in batched(range(100), 10):
    process_batch(batch)  # batch is a tuple of 10 items
```

### `pathlib` Enhancements

```python
from pathlib import Path

# walk() method (3.12+)
for root, dirs, files in Path(".").walk():
    for f in files:
        print(root / f)
```

---

## Configuration and Tooling Summary

### Recommended `pyproject.toml` Tool Stack

| Tool     | Purpose                    | Replaces              |
|----------|----------------------------|-----------------------|
| Ruff     | Linting and formatting     | flake8, isort, black  |
| mypy     | Static type checking       | --                    |
| pytest   | Testing                    | unittest              |
| pip-audit| Vulnerability scanning     | safety                |
| pre-commit| Git hook management       | --                    |

### Minimal `pyproject.toml` Tools Section

```toml
[tool.ruff]
target-version = "py312"
line-length = 88

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "A", "SIM", "TCH"]

[tool.mypy]
python_version = "3.12"
strict = true
warn_return_any = true
warn_unused_configs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra --strict-markers --strict-config"
```
