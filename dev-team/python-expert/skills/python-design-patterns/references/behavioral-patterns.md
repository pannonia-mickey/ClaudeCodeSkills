# Behavioral Patterns in Python

Full Python implementations of behavioral design patterns, adapted for Python's first-class functions, generators, Protocols, and match/case syntax.

---

## Strategy Pattern

Defines a family of algorithms, encapsulates each one, and makes them interchangeable. In Python, prefer callables for simple cases and Protocol classes for stateful strategies.

### Callable-Based Strategy

```python
from typing import Callable

type SortKey = Callable[[dict[str, object]], object]

def sort_records(
    records: list[dict[str, object]],
    key_func: SortKey,
    reverse: bool = False,
) -> list[dict[str, object]]:
    return sorted(records, key=key_func, reverse=reverse)

# Strategies as plain functions
def by_name(record: dict[str, object]) -> object:
    return record.get("name", "")

def by_date(record: dict[str, object]) -> object:
    return record.get("created_at", "")

def by_priority(record: dict[str, object]) -> object:
    priority_map = {"high": 0, "medium": 1, "low": 2}
    return priority_map.get(str(record.get("priority", "low")), 99)

# Usage
records = [
    {"name": "Task A", "priority": "low", "created_at": "2024-01-01"},
    {"name": "Task B", "priority": "high", "created_at": "2024-06-15"},
]
sorted_records = sort_records(records, by_priority)
```

### Protocol-Based Strategy (Stateful)

```python
from typing import Protocol
from dataclasses import dataclass

class CompressionStrategy(Protocol):
    def compress(self, data: bytes) -> bytes: ...
    def decompress(self, data: bytes) -> bytes: ...
    @property
    def name(self) -> str: ...

@dataclass
class GzipCompression:
    level: int = 6

    def compress(self, data: bytes) -> bytes:
        import gzip
        return gzip.compress(data, compresslevel=self.level)

    def decompress(self, data: bytes) -> bytes:
        import gzip
        return gzip.decompress(data)

    @property
    def name(self) -> str:
        return f"gzip-{self.level}"

@dataclass
class LZ4Compression:
    block_size: int = 65536

    def compress(self, data: bytes) -> bytes:
        # Placeholder for lz4 compression
        return data

    def decompress(self, data: bytes) -> bytes:
        return data

    @property
    def name(self) -> str:
        return "lz4"

class FileArchiver:
    def __init__(self, strategy: CompressionStrategy) -> None:
        self._strategy = strategy

    def archive(self, files: dict[str, bytes]) -> bytes:
        import json
        manifest = json.dumps(list(files.keys())).encode()
        compressed_files = {
            name: self._strategy.compress(data)
            for name, data in files.items()
        }
        # Simplified archive format
        return manifest + b"\x00" + b"".join(compressed_files.values())

    def set_strategy(self, strategy: CompressionStrategy) -> None:
        self._strategy = strategy
```

---

## Observer Pattern

Defines a one-to-many dependency so that when one object changes state, all dependents are notified.

### Event Bus Implementation

```python
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Callable, Any
from enum import StrEnum, auto
import weakref

class EventType(StrEnum):
    USER_CREATED = auto()
    USER_UPDATED = auto()
    USER_DELETED = auto()
    ORDER_PLACED = auto()
    ORDER_SHIPPED = auto()

type EventHandler = Callable[["Event"], None]

@dataclass(frozen=True)
class Event:
    type: EventType
    data: dict[str, Any]
    timestamp: float = field(default_factory=lambda: __import__("time").time())

class EventBus:
    def __init__(self) -> None:
        self._handlers: dict[EventType, list[EventHandler]] = defaultdict(list)
        self._global_handlers: list[EventHandler] = []

    def on(self, event_type: EventType, handler: EventHandler) -> Callable[[], None]:
        """Subscribe to an event type. Returns an unsubscribe function."""
        self._handlers[event_type].append(handler)

        def unsubscribe() -> None:
            self._handlers[event_type].remove(handler)

        return unsubscribe

    def on_all(self, handler: EventHandler) -> Callable[[], None]:
        """Subscribe to all events."""
        self._global_handlers.append(handler)

        def unsubscribe() -> None:
            self._global_handlers.remove(handler)

        return unsubscribe

    def emit(self, event: Event) -> None:
        """Publish an event to all subscribers."""
        for handler in self._handlers.get(event.type, []):
            handler(event)
        for handler in self._global_handlers:
            handler(event)

# Usage
bus = EventBus()

def log_event(event: Event) -> None:
    print(f"[{event.type}] {event.data}")

def send_welcome_email(event: Event) -> None:
    if event.type == EventType.USER_CREATED:
        print(f"Sending welcome email to {event.data['email']}")

unsubscribe_log = bus.on_all(log_event)
bus.on(EventType.USER_CREATED, send_welcome_email)

bus.emit(Event(
    type=EventType.USER_CREATED,
    data={"name": "Alice", "email": "alice@example.com"},
))
```

### Async Observer

```python
import asyncio
from collections import defaultdict
from typing import Callable, Awaitable

type AsyncEventHandler = Callable[["Event"], Awaitable[None]]

class AsyncEventBus:
    def __init__(self) -> None:
        self._handlers: dict[str, list[AsyncEventHandler]] = defaultdict(list)

    def on(self, event_type: str, handler: AsyncEventHandler) -> None:
        self._handlers[event_type].append(handler)

    async def emit(self, event_type: str, data: dict[str, object]) -> None:
        handlers = self._handlers.get(event_type, [])
        if handlers:
            await asyncio.gather(*(h(Event(type=event_type, data=data)) for h in handlers))
```

---

## Command Pattern

Encapsulates a request as an object, enabling parameterization, queueing, logging, and undo/redo.

```python
from typing import Protocol
from dataclasses import dataclass, field
from collections.abc import Sequence

class Command(Protocol):
    def execute(self) -> None: ...
    def undo(self) -> None: ...
    @property
    def description(self) -> str: ...

@dataclass
class MoveCommand:
    entity: "Entity"
    dx: float
    dy: float

    def execute(self) -> None:
        self.entity.x += self.dx
        self.entity.y += self.dy

    def undo(self) -> None:
        self.entity.x -= self.dx
        self.entity.y -= self.dy

    @property
    def description(self) -> str:
        return f"Move {self.entity.name} by ({self.dx}, {self.dy})"

@dataclass
class ResizeCommand:
    entity: "Entity"
    scale: float
    _original_width: float = field(init=False, default=0.0)
    _original_height: float = field(init=False, default=0.0)

    def execute(self) -> None:
        self._original_width = self.entity.width
        self._original_height = self.entity.height
        self.entity.width *= self.scale
        self.entity.height *= self.scale

    def undo(self) -> None:
        self.entity.width = self._original_width
        self.entity.height = self._original_height

    @property
    def description(self) -> str:
        return f"Resize {self.entity.name} by {self.scale}x"

@dataclass
class MacroCommand:
    """Composite command that executes multiple commands as one."""
    commands: list[Command]
    _description: str = "Macro"

    def execute(self) -> None:
        for cmd in self.commands:
            cmd.execute()

    def undo(self) -> None:
        for cmd in reversed(self.commands):
            cmd.undo()

    @property
    def description(self) -> str:
        return self._description

class CommandHistory:
    def __init__(self, max_size: int = 100) -> None:
        self._undo_stack: list[Command] = []
        self._redo_stack: list[Command] = []
        self._max_size = max_size

    def execute(self, command: Command) -> None:
        command.execute()
        self._undo_stack.append(command)
        if len(self._undo_stack) > self._max_size:
            self._undo_stack.pop(0)
        self._redo_stack.clear()

    def undo(self) -> str | None:
        if not self._undo_stack:
            return None
        command = self._undo_stack.pop()
        command.undo()
        self._redo_stack.append(command)
        return command.description

    def redo(self) -> str | None:
        if not self._redo_stack:
            return None
        command = self._redo_stack.pop()
        command.execute()
        self._undo_stack.append(command)
        return command.description

    @property
    def can_undo(self) -> bool:
        return len(self._undo_stack) > 0

    @property
    def can_redo(self) -> bool:
        return len(self._redo_stack) > 0
```

---

## Iterator Pattern

In Python, iterators are built into the language via `__iter__` and `__next__`. Generators are the idiomatic way to implement custom iterators.

### Generator-Based Iterators

```python
from collections.abc import Generator, Iterator
from dataclasses import dataclass

@dataclass
class TreeNode:
    value: int
    left: "TreeNode | None" = None
    right: "TreeNode | None" = None

def inorder(node: TreeNode | None) -> Generator[int, None, None]:
    """In-order traversal using a generator."""
    if node is not None:
        yield from inorder(node.left)
        yield node.value
        yield from inorder(node.right)

def preorder(node: TreeNode | None) -> Generator[int, None, None]:
    if node is not None:
        yield node.value
        yield from preorder(node.left)
        yield from preorder(node.right)

def level_order(root: TreeNode) -> Generator[int, None, None]:
    """Breadth-first traversal."""
    from collections import deque
    queue: deque[TreeNode] = deque([root])
    while queue:
        node = queue.popleft()
        yield node.value
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

# Usage
tree = TreeNode(
    4,
    left=TreeNode(2, TreeNode(1), TreeNode(3)),
    right=TreeNode(6, TreeNode(5), TreeNode(7)),
)

print(list(inorder(tree)))      # [1, 2, 3, 4, 5, 6, 7]
print(list(preorder(tree)))     # [4, 2, 1, 3, 6, 5, 7]
print(list(level_order(tree)))  # [4, 2, 6, 1, 3, 5, 7]
```

### Class-Based Iterator (When State Is Complex)

```python
class PaginatedIterator:
    """Iterates over paginated API results."""
    def __init__(self, client: "APIClient", endpoint: str, page_size: int = 50) -> None:
        self._client = client
        self._endpoint = endpoint
        self._page_size = page_size
        self._current_page = 0
        self._buffer: list[dict[str, object]] = []
        self._exhausted = False

    def __iter__(self) -> "PaginatedIterator":
        return self

    def __next__(self) -> dict[str, object]:
        if not self._buffer:
            if self._exhausted:
                raise StopIteration
            self._fetch_next_page()
        if not self._buffer:
            raise StopIteration
        return self._buffer.pop(0)

    def _fetch_next_page(self) -> None:
        results = self._client.get(
            self._endpoint,
            params={"page": self._current_page, "size": self._page_size},
        )
        if not results:
            self._exhausted = True
        else:
            self._buffer.extend(results)
            self._current_page += 1
```

---

## State Pattern

Allows an object to alter its behavior when its internal state changes. The object appears to change its class.

```python
from typing import Protocol
from dataclasses import dataclass

class OrderState(Protocol):
    def pay(self, order: "Order") -> None: ...
    def ship(self, order: "Order") -> None: ...
    def deliver(self, order: "Order") -> None: ...
    def cancel(self, order: "Order") -> None: ...
    @property
    def name(self) -> str: ...

class PendingState:
    @property
    def name(self) -> str:
        return "pending"

    def pay(self, order: "Order") -> None:
        print("Payment processed.")
        order.state = PaidState()

    def ship(self, order: "Order") -> None:
        raise InvalidTransitionError("Cannot ship unpaid order")

    def deliver(self, order: "Order") -> None:
        raise InvalidTransitionError("Cannot deliver unpaid order")

    def cancel(self, order: "Order") -> None:
        print("Order cancelled.")
        order.state = CancelledState()

class PaidState:
    @property
    def name(self) -> str:
        return "paid"

    def pay(self, order: "Order") -> None:
        raise InvalidTransitionError("Already paid")

    def ship(self, order: "Order") -> None:
        print("Order shipped.")
        order.state = ShippedState()

    def deliver(self, order: "Order") -> None:
        raise InvalidTransitionError("Must ship before delivering")

    def cancel(self, order: "Order") -> None:
        print("Refund issued. Order cancelled.")
        order.state = CancelledState()

class ShippedState:
    @property
    def name(self) -> str:
        return "shipped"

    def pay(self, order: "Order") -> None:
        raise InvalidTransitionError("Already paid")

    def ship(self, order: "Order") -> None:
        raise InvalidTransitionError("Already shipped")

    def deliver(self, order: "Order") -> None:
        print("Order delivered.")
        order.state = DeliveredState()

    def cancel(self, order: "Order") -> None:
        raise InvalidTransitionError("Cannot cancel shipped order")

class DeliveredState:
    @property
    def name(self) -> str:
        return "delivered"

    def pay(self, order: "Order") -> None:
        raise InvalidTransitionError("Order complete")

    def ship(self, order: "Order") -> None:
        raise InvalidTransitionError("Order complete")

    def deliver(self, order: "Order") -> None:
        raise InvalidTransitionError("Already delivered")

    def cancel(self, order: "Order") -> None:
        raise InvalidTransitionError("Cannot cancel delivered order")

class CancelledState:
    @property
    def name(self) -> str:
        return "cancelled"

    def pay(self, order: "Order") -> None:
        raise InvalidTransitionError("Order is cancelled")

    def ship(self, order: "Order") -> None:
        raise InvalidTransitionError("Order is cancelled")

    def deliver(self, order: "Order") -> None:
        raise InvalidTransitionError("Order is cancelled")

    def cancel(self, order: "Order") -> None:
        raise InvalidTransitionError("Already cancelled")

class InvalidTransitionError(Exception):
    pass

@dataclass
class Order:
    id: str
    state: OrderState

    def pay(self) -> None:
        self.state.pay(self)

    def ship(self) -> None:
        self.state.ship(self)

    def deliver(self) -> None:
        self.state.deliver(self)

    def cancel(self) -> None:
        self.state.cancel(self)

# Usage
order = Order(id="ORD-001", state=PendingState())
order.pay()       # "Payment processed." -> state becomes PaidState
order.ship()      # "Order shipped." -> state becomes ShippedState
order.deliver()   # "Order delivered." -> state becomes DeliveredState
```

---

## Chain of Responsibility

Passes a request along a chain of handlers. Each handler decides to process the request or pass it to the next handler.

```python
from typing import Protocol
from dataclasses import dataclass

@dataclass
class SupportTicket:
    severity: str  # "low", "medium", "high", "critical"
    category: str
    description: str
    resolved: bool = False
    resolution: str = ""

class TicketHandler(Protocol):
    def handle(self, ticket: SupportTicket) -> bool: ...

class SupportChain:
    def __init__(self) -> None:
        self._handlers: list[TicketHandler] = []

    def add_handler(self, handler: TicketHandler) -> "SupportChain":
        self._handlers.append(handler)
        return self

    def handle(self, ticket: SupportTicket) -> bool:
        for handler in self._handlers:
            if handler.handle(ticket):
                return True
        return False

class FAQHandler:
    _FAQ: dict[str, str] = {
        "password_reset": "Visit /reset-password to reset your password.",
        "billing": "Contact billing@example.com for billing inquiries.",
    }

    def handle(self, ticket: SupportTicket) -> bool:
        if ticket.severity == "low" and ticket.category in self._FAQ:
            ticket.resolved = True
            ticket.resolution = self._FAQ[ticket.category]
            return True
        return False

class TechSupportHandler:
    def handle(self, ticket: SupportTicket) -> bool:
        if ticket.severity in ("low", "medium") and ticket.category == "technical":
            ticket.resolved = True
            ticket.resolution = "Assigned to tech support team."
            return True
        return False

class EscalationHandler:
    def handle(self, ticket: SupportTicket) -> bool:
        if ticket.severity in ("high", "critical"):
            ticket.resolved = True
            ticket.resolution = f"Escalated to senior engineering ({ticket.severity})."
            return True
        return False

class FallbackHandler:
    def handle(self, ticket: SupportTicket) -> bool:
        ticket.resolved = True
        ticket.resolution = "Ticket queued for manual review."
        return True

# Build the chain
chain = (
    SupportChain()
    .add_handler(FAQHandler())
    .add_handler(TechSupportHandler())
    .add_handler(EscalationHandler())
    .add_handler(FallbackHandler())
)

ticket = SupportTicket(severity="critical", category="outage", description="Site is down")
chain.handle(ticket)
print(ticket.resolution)  # "Escalated to senior engineering (critical)."
```

---

## Template Method

Defines the skeleton of an algorithm in a base class, letting subclasses override specific steps without changing the structure.

### ABC-Based Template Method

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from pathlib import Path

class DataPipeline(ABC):
    """Template method pattern: the skeleton is in `run()`."""

    def run(self, source: str, destination: str) -> int:
        raw_data = self.extract(source)
        validated = self.validate(raw_data)
        transformed = self.transform(validated)
        count = self.load(transformed, destination)
        self.notify(count)
        return count

    @abstractmethod
    def extract(self, source: str) -> list[dict[str, object]]:
        """Extract raw data from the source."""
        ...

    def validate(self, data: list[dict[str, object]]) -> list[dict[str, object]]:
        """Optional validation step. Override to add custom validation."""
        return [record for record in data if record]

    @abstractmethod
    def transform(self, data: list[dict[str, object]]) -> list[dict[str, object]]:
        """Transform data into the target format."""
        ...

    @abstractmethod
    def load(self, data: list[dict[str, object]], destination: str) -> int:
        """Load transformed data into the destination."""
        ...

    def notify(self, count: int) -> None:
        """Optional notification hook. Override to customize."""
        print(f"Pipeline complete: {count} records processed.")

class CSVToJSONPipeline(DataPipeline):
    def extract(self, source: str) -> list[dict[str, object]]:
        import csv
        with open(source) as f:
            return list(csv.DictReader(f))

    def transform(self, data: list[dict[str, object]]) -> list[dict[str, object]]:
        return [
            {k.lower().replace(" ", "_"): v for k, v in record.items()}
            for record in data
        ]

    def load(self, data: list[dict[str, object]], destination: str) -> int:
        import json
        with open(destination, "w") as f:
            json.dump(data, f, indent=2)
        return len(data)

class APIToDBPipeline(DataPipeline):
    def extract(self, source: str) -> list[dict[str, object]]:
        import httpx
        response = httpx.get(source)
        return response.json()["results"]

    def validate(self, data: list[dict[str, object]]) -> list[dict[str, object]]:
        return [r for r in data if "id" in r and "name" in r]

    def transform(self, data: list[dict[str, object]]) -> list[dict[str, object]]:
        return [{"external_id": r["id"], "display_name": r["name"]} for r in data]

    def load(self, data: list[dict[str, object]], destination: str) -> int:
        # Placeholder for database insertion
        print(f"Inserting {len(data)} records into {destination}")
        return len(data)

    def notify(self, count: int) -> None:
        print(f"API sync complete: {count} records. Sending Slack notification...")
```

### Callable-Based Template (Pythonic Alternative)

For simpler cases, pass the variable steps as callables instead of using inheritance:

```python
from typing import Callable

type Extractor = Callable[[str], list[dict[str, object]]]
type Transformer = Callable[[list[dict[str, object]]], list[dict[str, object]]]
type Loader = Callable[[list[dict[str, object]], str], int]

def run_pipeline(
    source: str,
    destination: str,
    extract: Extractor,
    transform: Transformer,
    load: Loader,
) -> int:
    raw = extract(source)
    transformed = transform(raw)
    return load(transformed, destination)

# Usage with plain functions
count = run_pipeline(
    source="data.csv",
    destination="output.json",
    extract=extract_csv,
    transform=normalize_columns,
    load=write_json,
)
```

---

## Pattern Comparison Table

| Pattern                 | Intent                              | Python Mechanism              | Complexity |
|-------------------------|-------------------------------------|-------------------------------|:----------:|
| Strategy                | Interchangeable algorithms          | Callable or Protocol          | Low        |
| Observer                | Event notification                  | EventBus, callbacks           | Medium     |
| Command                 | Encapsulate requests, undo/redo     | Protocol + dataclass          | Medium     |
| Iterator                | Sequential access                   | Generator, `__iter__`         | Low        |
| State                   | State-dependent behavior            | Protocol + state classes      | High       |
| Chain of Responsibility | Pass request along handler chain    | List of Protocol handlers     | Medium     |
| Template Method         | Algorithm skeleton with hooks       | ABC or callables              | Medium     |
