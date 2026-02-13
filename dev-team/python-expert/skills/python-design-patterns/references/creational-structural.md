# Creational and Structural Patterns in Python

Full Python implementations of creational and structural design patterns, adapted for Python's dynamic features including Protocols, dataclasses, first-class functions, and dunder methods.

---

## Creational Patterns

### Factory Method

The Factory Method pattern defines an interface for creating objects but lets subclasses decide which class to instantiate.

```python
from abc import ABC, abstractmethod
from typing import Protocol
from dataclasses import dataclass

class Document(Protocol):
    def render(self) -> str: ...

@dataclass(frozen=True)
class HTMLDocument:
    title: str
    body: str

    def render(self) -> str:
        return f"<html><head><title>{self.title}</title></head><body>{self.body}</body></html>"

@dataclass(frozen=True)
class MarkdownDocument:
    title: str
    body: str

    def render(self) -> str:
        return f"# {self.title}\n\n{self.body}"

@dataclass(frozen=True)
class PDFDocument:
    title: str
    body: str

    def render(self) -> str:
        return f"[PDF] {self.title}: {self.body}"

class DocumentFactory(ABC):
    @abstractmethod
    def create(self, title: str, body: str) -> Document: ...

class HTMLFactory(DocumentFactory):
    def create(self, title: str, body: str) -> Document:
        return HTMLDocument(title=title, body=body)

class MarkdownFactory(DocumentFactory):
    def create(self, title: str, body: str) -> Document:
        return MarkdownDocument(title=title, body=body)

# Pythonic alternative: registry dict
_FACTORIES: dict[str, type[Document]] = {
    "html": HTMLDocument,
    "markdown": MarkdownDocument,
    "pdf": PDFDocument,
}

def create_document(format: str, title: str, body: str) -> Document:
    cls = _FACTORIES.get(format)
    if cls is None:
        raise ValueError(f"Unknown format: {format}")
    return cls(title=title, body=body)
```

### Abstract Factory

Groups related factories together for creating families of objects:

```python
from typing import Protocol

class Button(Protocol):
    def click(self) -> str: ...

class TextField(Protocol):
    def input(self, text: str) -> str: ...

class DarkButton:
    def click(self) -> str:
        return "Dark button clicked"

class DarkTextField:
    def input(self, text: str) -> str:
        return f"Dark input: {text}"

class LightButton:
    def click(self) -> str:
        return "Light button clicked"

class LightTextField:
    def input(self, text: str) -> str:
        return f"Light input: {text}"

class UIFactory(Protocol):
    def create_button(self) -> Button: ...
    def create_text_field(self) -> TextField: ...

class DarkThemeFactory:
    def create_button(self) -> Button:
        return DarkButton()

    def create_text_field(self) -> TextField:
        return DarkTextField()

class LightThemeFactory:
    def create_button(self) -> Button:
        return LightButton()

    def create_text_field(self) -> TextField:
        return LightTextField()

def build_form(factory: UIFactory) -> None:
    button = factory.create_button()
    text_field = factory.create_text_field()
    print(text_field.input("Hello"))
    print(button.click())
```

### Builder

Separates complex object construction from its representation:

```python
from dataclasses import dataclass, field
from typing import Self

@dataclass(frozen=True)
class HTTPRequest:
    method: str
    url: str
    headers: dict[str, str]
    body: bytes | None
    timeout: float
    retries: int

class HTTPRequestBuilder:
    def __init__(self, method: str, url: str) -> None:
        self._method = method
        self._url = url
        self._headers: dict[str, str] = {}
        self._body: bytes | None = None
        self._timeout: float = 30.0
        self._retries: int = 0

    def header(self, key: str, value: str) -> Self:
        self._headers[key] = value
        return self

    def body(self, data: bytes) -> Self:
        self._body = data
        return self

    def timeout(self, seconds: float) -> Self:
        self._timeout = seconds
        return self

    def retries(self, count: int) -> Self:
        self._retries = count
        return self

    def build(self) -> HTTPRequest:
        return HTTPRequest(
            method=self._method,
            url=self._url,
            headers=dict(self._headers),
            body=self._body,
            timeout=self._timeout,
            retries=self._retries,
        )

# Usage
request = (
    HTTPRequestBuilder("POST", "https://api.example.com/data")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body(b'{"key": "value"}')
    .timeout(10.0)
    .retries(3)
    .build()
)
```

### Singleton (Module-Level Pattern)

The Pythonic singleton is a module-level instance. For lazy initialization, use `__new__`:

```python
# Approach 1: Module-level instance (preferred)
# _config.py
class _AppConfig:
    def __init__(self) -> None:
        self._settings: dict[str, object] = {}
        self._loaded = False

    def load(self, path: str) -> None:
        import tomllib
        with open(path, "rb") as f:
            self._settings = tomllib.load(f)
        self._loaded = True

    def get(self, key: str, default: object = None) -> object:
        return self._settings.get(key, default)

app_config = _AppConfig()

# Approach 2: __new__ for lazy singleton (when subclassing is needed)
class DatabasePool:
    _instance: "DatabasePool | None" = None

    def __new__(cls) -> "DatabasePool":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._pool = None
        return cls._instance

    def connect(self, url: str) -> None:
        if self._pool is None:
            self._pool = create_pool(url)
```

---

## Structural Patterns

### Adapter

Converts the interface of one class into another that clients expect:

```python
from typing import Protocol
from dataclasses import dataclass

class PaymentGateway(Protocol):
    def charge(self, amount_cents: int, currency: str) -> str: ...
    def refund(self, transaction_id: str) -> bool: ...

class LegacyPayPal:
    """Third-party class with an incompatible interface."""
    def make_payment(self, amount: float, curr: str) -> dict[str, str]:
        return {"txn_id": "pp_123", "status": "completed"}

    def reverse_payment(self, txn_id: str) -> dict[str, bool]:
        return {"success": True}

class PayPalAdapter:
    """Adapts LegacyPayPal to the PaymentGateway protocol."""
    def __init__(self, paypal: LegacyPayPal) -> None:
        self._paypal = paypal

    def charge(self, amount_cents: int, currency: str) -> str:
        result = self._paypal.make_payment(amount_cents / 100, currency.lower())
        return result["txn_id"]

    def refund(self, transaction_id: str) -> bool:
        result = self._paypal.reverse_payment(transaction_id)
        return result["success"]

def process_payment(gateway: PaymentGateway, amount: int) -> str:
    return gateway.charge(amount, "USD")

# Usage
adapter = PayPalAdapter(LegacyPayPal())
txn_id = process_payment(adapter, 5000)
```

### Decorator (GoF Pattern)

Adds responsibilities to objects dynamically by wrapping them:

```python
from typing import Protocol

class TextProcessor(Protocol):
    def process(self, text: str) -> str: ...

class PlainTextProcessor:
    def process(self, text: str) -> str:
        return text

class UpperCaseDecorator:
    def __init__(self, wrapped: TextProcessor) -> None:
        self._wrapped = wrapped

    def process(self, text: str) -> str:
        return self._wrapped.process(text).upper()

class TrimDecorator:
    def __init__(self, wrapped: TextProcessor) -> None:
        self._wrapped = wrapped

    def process(self, text: str) -> str:
        return self._wrapped.process(text).strip()

class CensorDecorator:
    def __init__(self, wrapped: TextProcessor, words: list[str]) -> None:
        self._wrapped = wrapped
        self._words = words

    def process(self, text: str) -> str:
        result = self._wrapped.process(text)
        for word in self._words:
            result = result.replace(word, "***")
        return result

# Stack decorators
processor = CensorDecorator(
    UpperCaseDecorator(
        TrimDecorator(PlainTextProcessor())
    ),
    words=["BADWORD"],
)
print(processor.process("  hello badword world  "))
# Output: "HELLO *** WORLD"
```

### Facade

Provides a simplified interface to a complex subsystem:

```python
from dataclasses import dataclass

class VideoDecoder:
    def decode(self, data: bytes) -> "VideoFrames":
        print("Decoding video...")
        return VideoFrames(frames=[])

class AudioDecoder:
    def decode(self, data: bytes) -> "AudioSamples":
        print("Decoding audio...")
        return AudioSamples(samples=[])

class SubtitleParser:
    def parse(self, data: bytes) -> list[str]:
        print("Parsing subtitles...")
        return []

class Renderer:
    def render(self, video: "VideoFrames", audio: "AudioSamples", subs: list[str]) -> None:
        print("Rendering media...")

@dataclass
class VideoFrames:
    frames: list[object]

@dataclass
class AudioSamples:
    samples: list[object]

class MediaPlayerFacade:
    """Simplified interface hiding the complexity of multiple subsystems."""
    def __init__(self) -> None:
        self._video_decoder = VideoDecoder()
        self._audio_decoder = AudioDecoder()
        self._subtitle_parser = SubtitleParser()
        self._renderer = Renderer()

    def play(self, video_data: bytes, audio_data: bytes, subtitle_data: bytes | None = None) -> None:
        video = self._video_decoder.decode(video_data)
        audio = self._audio_decoder.decode(audio_data)
        subs = self._subtitle_parser.parse(subtitle_data) if subtitle_data else []
        self._renderer.render(video, audio, subs)

# Client only needs the facade
player = MediaPlayerFacade()
player.play(video_data=b"...", audio_data=b"...")
```

### Proxy

Controls access to another object, adding behavior like lazy loading, access control, or caching:

```python
from typing import Protocol

class ImageLoader(Protocol):
    def load(self) -> bytes: ...
    @property
    def dimensions(self) -> tuple[int, int]: ...

class RealImage:
    def __init__(self, path: str) -> None:
        self._path = path
        self._data = self._read_file()
        self._width = 1920
        self._height = 1080

    def _read_file(self) -> bytes:
        print(f"Loading image from disk: {self._path}")
        with open(self._path, "rb") as f:
            return f.read()

    def load(self) -> bytes:
        return self._data

    @property
    def dimensions(self) -> tuple[int, int]:
        return (self._width, self._height)

class LazyImageProxy:
    """Defers expensive loading until actually needed."""
    def __init__(self, path: str) -> None:
        self._path = path
        self._real_image: RealImage | None = None

    def _ensure_loaded(self) -> RealImage:
        if self._real_image is None:
            self._real_image = RealImage(self._path)
        return self._real_image

    def load(self) -> bytes:
        return self._ensure_loaded().load()

    @property
    def dimensions(self) -> tuple[int, int]:
        return self._ensure_loaded().dimensions

class AccessControlProxy:
    """Adds permission checks before delegating."""
    def __init__(self, image: ImageLoader, allowed_users: set[str]) -> None:
        self._image = image
        self._allowed = allowed_users

    def load(self, user: str) -> bytes:
        if user not in self._allowed:
            raise PermissionError(f"User {user} not allowed")
        return self._image.load()

    @property
    def dimensions(self) -> tuple[int, int]:
        return self._image.dimensions
```

### Composite

Composes objects into tree structures to represent part-whole hierarchies:

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field

class FileSystemNode(ABC):
    @abstractmethod
    def size(self) -> int: ...

    @abstractmethod
    def display(self, indent: int = 0) -> str: ...

@dataclass
class File(FileSystemNode):
    name: str
    _size: int

    def size(self) -> int:
        return self._size

    def display(self, indent: int = 0) -> str:
        return f"{'  ' * indent}{self.name} ({self._size} bytes)"

@dataclass
class Directory(FileSystemNode):
    name: str
    children: list[FileSystemNode] = field(default_factory=list)

    def add(self, node: FileSystemNode) -> None:
        self.children.append(node)

    def remove(self, node: FileSystemNode) -> None:
        self.children.remove(node)

    def size(self) -> int:
        return sum(child.size() for child in self.children)

    def display(self, indent: int = 0) -> str:
        lines = [f"{'  ' * indent}{self.name}/"]
        for child in self.children:
            lines.append(child.display(indent + 1))
        return "\n".join(lines)

# Usage
root = Directory("project")
src = Directory("src")
src.add(File("main.py", 1500))
src.add(File("utils.py", 800))
root.add(src)
root.add(File("README.md", 200))

print(root.display())
# project/
#   src/
#     main.py (1500 bytes)
#     utils.py (800 bytes)
#   README.md (200 bytes)

print(f"Total size: {root.size()} bytes")
# Total size: 2500 bytes
```

---

## Pattern Comparison Table

| Pattern          | Intent                        | Python Mechanism         | Complexity |
|------------------|-------------------------------|--------------------------|:----------:|
| Factory Method   | Delegate instantiation        | Dict registry, Protocol  | Low        |
| Abstract Factory | Families of related objects   | Protocol, classes        | Medium     |
| Builder          | Step-by-step construction     | Fluent API, dataclass    | Medium     |
| Singleton        | Single instance               | Module-level variable    | Low        |
| Adapter          | Interface translation         | Wrapper class, Protocol  | Low        |
| Decorator (GoF)  | Dynamic behavior addition     | Wrapper or `@decorator`  | Medium     |
| Facade           | Simplified subsystem access   | Wrapper class            | Low        |
| Proxy            | Controlled access             | Wrapper with lazy init   | Medium     |
| Composite        | Tree structures               | ABC with children list   | Medium     |
