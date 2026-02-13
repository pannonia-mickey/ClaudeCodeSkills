---
name: Python Design Patterns
description: This skill should be used when the user asks about "Python design pattern", "Python factory", "Python strategy", "Python observer", or "Pythonic pattern". It covers Gang of Four patterns adapted for Python's dynamic nature, identifies when patterns are overkill, and recommends Pythonic alternatives such as first-class functions, context managers, and protocols.
---

## Pattern Selection Guide

Before reaching for a classic design pattern, consider whether Python's built-in features provide a simpler solution.

### When to Use Patterns

Patterns are appropriate when:
- The codebase is large enough that the pattern's overhead pays for itself in organization
- Multiple developers need a shared vocabulary for a recurring structure
- The problem genuinely matches the pattern's intent, not just its shape
- Simpler Pythonic approaches have been considered and found insufficient

### When Patterns Are Overkill

| Classic Pattern     | Pythonic Alternative                      | Use Pattern When...                      |
|---------------------|-------------------------------------------|------------------------------------------|
| Strategy            | Pass a `Callable` or function             | Strategies carry state or config         |
| Observer            | Use `signals` library or callbacks        | Many event types, complex subscription   |
| Singleton           | Module-level instance                     | Need lazy init or subclassing            |
| Factory Method      | `dict` mapping to constructors            | Factory logic is complex or hierarchical |
| Command             | Store callables in a list                 | Commands need undo/redo or serialization |
| Iterator            | Generator function                        | Always prefer generators in Python       |
| Template Method     | Pass callables for variable steps         | Skeleton has complex shared logic        |
| Decorator (GoF)     | Python `@decorator`                       | Need runtime add/remove of behaviors     |
| Builder             | Dataclass with `__post_init__`            | Construction has many optional steps     |
| State               | `match/case` or enum dispatch             | Many states with complex transitions     |

### Decision Framework

Ask these questions in order:

1. **Can a function solve this?** If the pattern just wraps a callable, use a function.
2. **Can a Protocol or ABC solve this?** If the pattern is about polymorphism, define a Protocol.
3. **Can a decorator or context manager solve this?** If the pattern wraps behavior, use Python's built-in wrappers.
4. **Does the pattern add clarity for the team?** If yes, use the pattern. If it adds ceremony without clarity, skip it.

---

## Creational Patterns

### Factory Method / Abstract Factory

Use when object creation logic is complex or varies by context:

```python
from typing import Protocol
from dataclasses import dataclass

class Serializer(Protocol):
    def serialize(self, data: dict[str, object]) -> bytes: ...

@dataclass
class JSONSerializer:
    indent: int = 2

    def serialize(self, data: dict[str, object]) -> bytes:
        import json
        return json.dumps(data, indent=self.indent).encode()

@dataclass
class XMLSerializer:
    root_tag: str = "data"

    def serialize(self, data: dict[str, object]) -> bytes:
        parts = [f"<{self.root_tag}>"]
        for k, v in data.items():
            parts.append(f"  <{k}>{v}</{k}>")
        parts.append(f"</{self.root_tag}>")
        return "\n".join(parts).encode()

# Pythonic factory: dict mapping
_SERIALIZERS: dict[str, type[Serializer]] = {
    "json": JSONSerializer,
    "xml": XMLSerializer,
}

def create_serializer(format: str, **kwargs: object) -> Serializer:
    cls = _SERIALIZERS.get(format)
    if cls is None:
        raise ValueError(f"Unknown format: {format}. Available: {list(_SERIALIZERS)}")
    return cls(**kwargs)
```

### Builder Pattern

For complex object construction with many optional parameters:

```python
from dataclasses import dataclass, field
from typing import Self

@dataclass
class Query:
    table: str
    columns: list[str] = field(default_factory=lambda: ["*"])
    conditions: list[str] = field(default_factory=list)
    order_by: str | None = None
    limit: int | None = None

class QueryBuilder:
    def __init__(self, table: str) -> None:
        self._table = table
        self._columns: list[str] = []
        self._conditions: list[str] = []
        self._order_by: str | None = None
        self._limit: int | None = None

    def select(self, *columns: str) -> Self:
        self._columns.extend(columns)
        return self

    def where(self, condition: str) -> Self:
        self._conditions.append(condition)
        return self

    def order(self, column: str) -> Self:
        self._order_by = column
        return self

    def with_limit(self, n: int) -> Self:
        self._limit = n
        return self

    def build(self) -> Query:
        return Query(
            table=self._table,
            columns=self._columns or ["*"],
            conditions=self._conditions,
            order_by=self._order_by,
            limit=self._limit,
        )

# Usage
query = (
    QueryBuilder("users")
    .select("name", "email")
    .where("active = true")
    .order("name")
    .with_limit(100)
    .build()
)
```

### Singleton (Module-Level)

The Pythonic singleton is simply a module-level instance:

```python
# config.py
class _Config:
    def __init__(self) -> None:
        self._data: dict[str, object] = {}

    def get(self, key: str, default: object = None) -> object:
        return self._data.get(key, default)

    def set(self, key: str, value: object) -> None:
        self._data[key] = value

config = _Config()  # Module-level singleton
```

---

## Structural Patterns

### Adapter

```python
from typing import Protocol

class ModernLogger(Protocol):
    def log(self, level: str, message: str) -> None: ...

class LegacyLogger:
    def write_log(self, msg: str, severity: int) -> None:
        print(f"[{severity}] {msg}")

class LegacyLoggerAdapter:
    _LEVEL_MAP = {"DEBUG": 0, "INFO": 1, "WARNING": 2, "ERROR": 3}

    def __init__(self, legacy: LegacyLogger) -> None:
        self._legacy = legacy

    def log(self, level: str, message: str) -> None:
        severity = self._LEVEL_MAP.get(level, 1)
        self._legacy.write_log(message, severity)
```

### Decorator (GoF) via Python Decorators

```python
from typing import Protocol

class DataSource(Protocol):
    def read(self) -> str: ...
    def write(self, data: str) -> None: ...

class FileDataSource:
    def __init__(self, path: str) -> None:
        self._path = path

    def read(self) -> str:
        with open(self._path) as f:
            return f.read()

    def write(self, data: str) -> None:
        with open(self._path, "w") as f:
            f.write(data)

class EncryptedDataSource:
    """GoF Decorator wrapping a DataSource."""
    def __init__(self, source: DataSource) -> None:
        self._source = source

    def read(self) -> str:
        return decrypt(self._source.read())

    def write(self, data: str) -> None:
        self._source.write(encrypt(data))

class CompressedDataSource:
    def __init__(self, source: DataSource) -> None:
        self._source = source

    def read(self) -> str:
        return decompress(self._source.read())

    def write(self, data: str) -> None:
        self._source.write(compress(data))

# Stack decorators
source = CompressedDataSource(EncryptedDataSource(FileDataSource("data.bin")))
```

---

## Behavioral Patterns

### Strategy

For simple cases, use callables. For complex cases with configuration, use Protocol:

```python
from typing import Protocol, Callable

# Simple: callable strategy
def apply_discount(price: float, strategy: Callable[[float], float]) -> float:
    return strategy(price)

total = apply_discount(100.0, lambda p: p * 0.9)  # 10% off

# Complex: Protocol strategy with state
class ShippingStrategy(Protocol):
    def calculate(self, weight: float, distance: float) -> float: ...

class StandardShipping:
    rate_per_kg: float = 5.0

    def calculate(self, weight: float, distance: float) -> float:
        return weight * self.rate_per_kg + distance * 0.01

class ExpressShipping:
    rate_per_kg: float = 15.0
    express_surcharge: float = 10.0

    def calculate(self, weight: float, distance: float) -> float:
        return weight * self.rate_per_kg + distance * 0.02 + self.express_surcharge
```

### Observer

```python
from collections import defaultdict
from typing import Callable
from dataclasses import dataclass, field

type EventHandler = Callable[[dict[str, object]], None]

@dataclass
class EventBus:
    _handlers: dict[str, list[EventHandler]] = field(
        default_factory=lambda: defaultdict(list)
    )

    def subscribe(self, event: str, handler: EventHandler) -> None:
        self._handlers[event].append(handler)

    def unsubscribe(self, event: str, handler: EventHandler) -> None:
        self._handlers[event].remove(handler)

    def publish(self, event: str, data: dict[str, object] | None = None) -> None:
        for handler in self._handlers.get(event, []):
            handler(data or {})

# Usage
bus = EventBus()
bus.subscribe("user.created", lambda data: print(f"Welcome {data['name']}"))
bus.subscribe("user.created", lambda data: send_email(data["email"]))
bus.publish("user.created", {"name": "Alice", "email": "alice@test.com"})
```

### Command

```python
from typing import Protocol
from dataclasses import dataclass

class Command(Protocol):
    def execute(self) -> None: ...
    def undo(self) -> None: ...

@dataclass
class InsertTextCommand:
    document: "Document"
    position: int
    text: str

    def execute(self) -> None:
        self.document.insert(self.position, self.text)

    def undo(self) -> None:
        self.document.delete(self.position, len(self.text))

class CommandHistory:
    def __init__(self) -> None:
        self._history: list[Command] = []
        self._redo_stack: list[Command] = []

    def execute(self, command: Command) -> None:
        command.execute()
        self._history.append(command)
        self._redo_stack.clear()

    def undo(self) -> None:
        if self._history:
            command = self._history.pop()
            command.undo()
            self._redo_stack.append(command)

    def redo(self) -> None:
        if self._redo_stack:
            command = self._redo_stack.pop()
            command.execute()
            self._history.append(command)
```

---

## Combining Patterns

Real systems often combine multiple patterns. Here is a service layer combining Factory, Strategy, and Observer:

```python
class NotificationService:
    def __init__(
        self,
        channel_factory: Callable[[str], "NotificationChannel"],
        event_bus: EventBus,
    ) -> None:
        self._factory = channel_factory
        self._bus = event_bus
        self._bus.subscribe("order.completed", self._on_order_completed)

    def _on_order_completed(self, data: dict[str, object]) -> None:
        channel_name = str(data.get("preferred_channel", "email"))
        channel = self._factory(channel_name)
        channel.send(
            to=str(data["customer_email"]),
            message=f"Order {data['order_id']} confirmed!",
        )
```

Keep the combination minimal. If a class needs more than two or three patterns, reconsider the design.

---

## References

- [Creational and Structural Patterns](references/creational-structural.md) -- Full Python implementations of Factory, Builder, Singleton, Adapter, Decorator, Facade, Proxy, and Composite patterns.
- [Behavioral Patterns](references/behavioral-patterns.md) -- Full Python implementations of Strategy, Observer, Command, Iterator, State, Chain of Responsibility, and Template Method patterns.
