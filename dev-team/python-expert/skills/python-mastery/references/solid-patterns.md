# SOLID Patterns in Python

This reference covers each SOLID principle with Python-specific implementations, using ABCs, Protocol classes, decorators, and idiomatic Python patterns.

---

## Single Responsibility Principle (SRP)

Every module, class, and function should have one reason to change. In Python this means organizing code into focused modules and keeping classes single-purpose.

### Anti-Pattern: God Class

```python
# BAD: One class doing everything
class UserManager:
    def create_user(self, name: str, email: str) -> "User": ...
    def send_welcome_email(self, user: "User") -> None: ...
    def generate_report(self, users: list["User"]) -> str: ...
    def export_to_csv(self, users: list["User"]) -> bytes: ...
    def validate_password(self, password: str) -> bool: ...
```

### Corrected: Module Separation

```python
# users/models.py
from dataclasses import dataclass, field
from datetime import datetime

@dataclass(frozen=True, slots=True)
class User:
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.utcnow)


# users/repository.py
from typing import Protocol

class UserRepository(Protocol):
    def save(self, user: User) -> None: ...
    def find_by_email(self, email: str) -> User | None: ...


# users/service.py
class UserService:
    def __init__(self, repo: UserRepository, notifier: "Notifier") -> None:
        self._repo = repo
        self._notifier = notifier

    def create_user(self, name: str, email: str) -> User:
        user = User(name=name, email=email)
        self._repo.save(user)
        self._notifier.send_welcome(user)
        return user


# users/notifications.py
class EmailNotifier:
    def send_welcome(self, user: User) -> None:
        # Focused solely on email delivery
        ...


# users/reports.py
class UserReportGenerator:
    def generate(self, users: list[User]) -> str: ...
    def export_csv(self, users: list[User]) -> bytes: ...
```

Each module has a single axis of change: the repository changes if storage changes, the notifier changes if email delivery changes, and so on.

---

## Open/Closed Principle (OCP)

Classes should be open for extension but closed for modification. In Python, achieve this with ABCs, Protocol classes, the Strategy pattern, and decorators.

### Using ABCs for Extension

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount: float) -> bool: ...

    @abstractmethod
    def refund(self, transaction_id: str) -> bool: ...


class StripeProcessor(PaymentProcessor):
    def process(self, amount: float) -> bool:
        # Stripe-specific implementation
        return True

    def refund(self, transaction_id: str) -> bool:
        return True


class PayPalProcessor(PaymentProcessor):
    def process(self, amount: float) -> bool:
        # PayPal-specific implementation
        return True

    def refund(self, transaction_id: str) -> bool:
        return True
```

Adding a new payment processor requires no changes to existing code.

### Strategy with Protocol

```python
from typing import Protocol

class DiscountStrategy(Protocol):
    def calculate(self, price: float, quantity: int) -> float: ...

def bulk_discount(price: float, quantity: int) -> float:
    if quantity >= 100:
        return price * 0.8
    elif quantity >= 10:
        return price * 0.9
    return price

def seasonal_discount(price: float, quantity: int) -> float:
    return price * 0.85

class PriceCalculator:
    def __init__(self, strategy: DiscountStrategy) -> None:
        self._strategy = strategy

    def total(self, price: float, quantity: int) -> float:
        discounted = self._strategy.calculate(price, quantity)
        return discounted * quantity
```

Because `DiscountStrategy` is a Protocol, any callable with the right signature satisfies it, including plain functions. Extending the system means writing a new function, never modifying existing ones.

### Decorator-Based Extension

```python
from functools import wraps
from typing import Callable

def with_logging(func: Callable[..., object]) -> Callable[..., object]:
    @wraps(func)
    def wrapper(*args: object, **kwargs: object) -> object:
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

def with_retry(attempts: int = 3) -> Callable[[Callable[..., object]], Callable[..., object]]:
    def decorator(func: Callable[..., object]) -> Callable[..., object]:
        @wraps(func)
        def wrapper(*args: object, **kwargs: object) -> object:
            for i in range(attempts):
                try:
                    return func(*args, **kwargs)
                except Exception:
                    if i == attempts - 1:
                        raise
            raise RuntimeError("unreachable")
        return wrapper
    return decorator
```

Decorators extend behavior without modifying the decorated function.

---

## Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering correctness.

### ABC Subclassing Done Right

```python
from abc import ABC, abstractmethod
from collections.abc import Sequence

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

    @abstractmethod
    def perimeter(self) -> float: ...

class Rectangle(Shape):
    def __init__(self, width: float, height: float) -> None:
        self._width = width
        self._height = height

    def area(self) -> float:
        return self._width * self._height

    def perimeter(self) -> float:
        return 2 * (self._width + self._height)

class Square(Shape):
    """Square is NOT a subclass of Rectangle here, avoiding the classic LSP violation."""
    def __init__(self, side: float) -> None:
        self._side = side

    def area(self) -> float:
        return self._side ** 2

    def perimeter(self) -> float:
        return 4 * self._side

def total_area(shapes: Sequence[Shape]) -> float:
    return sum(s.area() for s in shapes)
```

The classic Rectangle/Square inheritance problem is avoided by making both direct subtypes of `Shape`.

### Protocol Classes (PEP 544)

```python
from typing import Protocol

class Closeable(Protocol):
    def close(self) -> None: ...

class FileWrapper:
    def close(self) -> None:
        print("File closed")

class ConnectionWrapper:
    def close(self) -> None:
        print("Connection closed")

def cleanup(resources: list[Closeable]) -> None:
    for r in resources:
        r.close()

# Both satisfy Closeable without inheriting from it
cleanup([FileWrapper(), ConnectionWrapper()])
```

Structural subtyping via Protocol ensures LSP compliance because any object matching the protocol shape is substitutable.

---

## Interface Segregation Principle (ISP)

No client should be forced to depend on methods it does not use. In Python, achieve this with small Protocol classes and targeted mixins.

### Small Protocol Interfaces

```python
from typing import Protocol

class Readable(Protocol):
    def read(self, size: int = -1) -> bytes: ...

class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

class Seekable(Protocol):
    def seek(self, offset: int, whence: int = 0) -> int: ...

class ReadWriteStream(Readable, Writable, Protocol):
    """Composed only where both are needed."""
    ...

def copy_stream(source: Readable, dest: Writable) -> int:
    total = 0
    while chunk := source.read(8192):
        total += dest.write(chunk)
    return total
```

Functions accept only the interface slices they actually need.

### Mixin Classes

```python
class JSONSerializableMixin:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)

class XMLSerializableMixin:
    def to_xml(self) -> str:
        parts = [f"<{k}>{v}</{k}>" for k, v in self.__dict__.items()]
        return f"<root>{''.join(parts)}</root>"

class APIResponse(JSONSerializableMixin):
    def __init__(self, status: int, data: dict[str, object]) -> None:
        self.status = status
        self.data = data

# APIResponse only has JSON serialization, not forced to carry XML
```

---

## Dependency Inversion Principle (DIP)

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### ABC Injection

```python
from abc import ABC, abstractmethod

class Logger(ABC):
    @abstractmethod
    def log(self, message: str) -> None: ...

class ConsoleLogger(Logger):
    def log(self, message: str) -> None:
        print(f"[LOG] {message}")

class FileLogger(Logger):
    def __init__(self, path: str) -> None:
        self._path = path

    def log(self, message: str) -> None:
        with open(self._path, "a") as f:
            f.write(f"{message}\n")

class OrderService:
    def __init__(self, logger: Logger) -> None:
        self._logger = logger

    def place_order(self, item: str) -> None:
        self._logger.log(f"Order placed: {item}")
```

### Protocol Typing

```python
from typing import Protocol

class CacheBackend(Protocol):
    def get(self, key: str) -> object | None: ...
    def set(self, key: str, value: object, ttl: int = 300) -> None: ...
    def delete(self, key: str) -> bool: ...

class RedisCacheBackend:
    def get(self, key: str) -> object | None: ...
    def set(self, key: str, value: object, ttl: int = 300) -> None: ...
    def delete(self, key: str) -> bool: ...

class InMemoryCacheBackend:
    def __init__(self) -> None:
        self._store: dict[str, object] = {}

    def get(self, key: str) -> object | None:
        return self._store.get(key)

    def set(self, key: str, value: object, ttl: int = 300) -> None:
        self._store[key] = value

    def delete(self, key: str) -> bool:
        return self._store.pop(key, None) is not None

class ProductService:
    def __init__(self, cache: CacheBackend) -> None:
        self._cache = cache
```

### Simple DI Container

```python
from typing import Any, TypeVar, Type

T = TypeVar("T")

class Container:
    def __init__(self) -> None:
        self._registry: dict[type, Any] = {}

    def register(self, interface: type, implementation: Any) -> None:
        self._registry[interface] = implementation

    def resolve(self, interface: Type[T]) -> T:
        impl = self._registry.get(interface)
        if impl is None:
            raise KeyError(f"No registration for {interface}")
        if callable(impl) and isinstance(impl, type):
            return impl()
        return impl

# Usage
container = Container()
container.register(Logger, ConsoleLogger)
container.register(CacheBackend, InMemoryCacheBackend)

logger = container.resolve(Logger)
cache = container.resolve(CacheBackend)
```

For production applications, consider established DI libraries like `dependency-injector` or `inject`, but the pattern above illustrates the principle clearly.

---

## Summary Table

| Principle | Python Mechanism               | Key Benefit                        |
|-----------|--------------------------------|------------------------------------|
| SRP       | Module separation, small classes | Each unit has one reason to change |
| OCP       | ABC, Protocol, decorators       | Extend without modifying           |
| LSP       | ABC contracts, Protocol typing  | Safe substitutability              |
| ISP       | Small Protocols, mixins         | No forced dependencies             |
| DIP       | ABC/Protocol injection, DI      | Decoupled, testable architecture   |
