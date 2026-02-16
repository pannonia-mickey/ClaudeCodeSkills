# SOLID Principles at the Architectural Level

## Single Responsibility Principle (SRP) — Per Service

Each service, module, or agent should have one reason to change.

### System-Level SRP

```
BAD: Monolithic service handling users, orders, payments, and notifications
     → Changes to payment logic risk breaking user management

GOOD: Separate bounded contexts
     UserService      → User registration, profiles, authentication
     OrderService     → Order lifecycle, cart management
     PaymentService   → Payment processing, refunds
     NotificationService → Email, SMS, push notifications
```

### Agent-Level SRP

Each specialist agent owns exactly one technology domain:

```
django-expert    → Django/DRF only (not FastAPI, not general Python)
fastapi-expert   → FastAPI only (not Django, not Flask)
python-expert    → General Python (not framework-specific patterns)
```

### Module-Level SRP

Within a service, separate concerns into modules:

```python
# Good: Each module has one responsibility
app/
├── models/          # Data layer only
├── services/        # Business logic only
├── api/             # HTTP interface only
├── repositories/    # Data access only
└── events/          # Event handling only
```

## Open/Closed Principle (OCP) — Extensibility

Systems should be open for extension, closed for modification.

### Plugin Architecture

```python
# Base interface — closed for modification
class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, amount: Decimal, currency: str) -> PaymentResult: ...

# Extensions — open for extension
class StripeProcessor(PaymentProcessor):
    def process(self, amount, currency):
        return self._call_stripe_api(amount, currency)

class PayPalProcessor(PaymentProcessor):
    def process(self, amount, currency):
        return self._call_paypal_api(amount, currency)

# New processors added without modifying existing code
class CryptoProcessor(PaymentProcessor):
    def process(self, amount, currency):
        return self._call_crypto_api(amount, currency)
```

### Event-Driven Extensibility

```python
# Core system publishes events — closed
class OrderService:
    def complete_order(self, order_id: str):
        order = self.repo.complete(order_id)
        self.event_bus.publish(OrderCompleted(order_id=order.id))

# New features subscribe — open for extension
class EmailNotifier:
    @subscribe(OrderCompleted)
    def send_confirmation(self, event: OrderCompleted):
        self.mailer.send_order_confirmation(event.order_id)

class AnalyticsTracker:
    @subscribe(OrderCompleted)
    def track_conversion(self, event: OrderCompleted):
        self.analytics.track("order_completed", event.order_id)
```

### Middleware Pipeline

```python
# FastAPI/Django middleware — add behavior without modifying core
middleware_stack = [
    CORSMiddleware,          # Existing
    AuthenticationMiddleware, # Existing
    RateLimitMiddleware,      # Added later — no core changes
    AuditLogMiddleware,       # Added later — no core changes
]
```

## Liskov Substitution Principle (LSP) — Contract Compatibility

Subtypes must be substitutable for their base types without breaking behavior.

### Service Contract Compatibility

```python
# Base contract
class UserRepository(ABC):
    @abstractmethod
    def get_by_id(self, user_id: str) -> User | None: ...

    @abstractmethod
    def save(self, user: User) -> User: ...

# PostgreSQL implementation — honors the contract
class PostgresUserRepository(UserRepository):
    def get_by_id(self, user_id: str) -> User | None:
        row = self.db.query("SELECT * FROM users WHERE id = %s", user_id)
        return User.from_row(row) if row else None

    def save(self, user: User) -> User:
        self.db.execute("INSERT INTO users ...", user.to_dict())
        return user

# Redis cache implementation — also honors the contract
class CachedUserRepository(UserRepository):
    def __init__(self, inner: UserRepository, cache: Redis):
        self.inner = inner
        self.cache = cache

    def get_by_id(self, user_id: str) -> User | None:
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            return User.from_json(cached)
        return self.inner.get_by_id(user_id)

    def save(self, user: User) -> User:
        result = self.inner.save(user)
        self.cache.set(f"user:{user.id}", user.to_json())
        return result
```

### API Versioning with LSP

```
v1 API contract:
  GET /users/{id} → { "id": "...", "name": "...", "email": "..." }

v2 API contract (backward compatible — LSP compliant):
  GET /users/{id} → { "id": "...", "name": "...", "email": "...", "avatar_url": "..." }
  # v2 adds fields but v1 consumers still work

v2 API contract (BREAKING — LSP violation):
  GET /users/{id} → { "user_id": "...", "full_name": "...", "email": "..." }
  # Renamed fields break existing consumers
```

## Interface Segregation Principle (ISP) — Granular Contracts

No client should depend on interfaces it doesn't use.

### Granular Service Interfaces

```csharp
// BAD: Fat interface forces implementors to handle everything
public interface IUserService
{
    User GetById(string id);
    User Create(CreateUserDto dto);
    User Update(UpdateUserDto dto);
    void Delete(string id);
    void SendVerificationEmail(string userId);
    void ResetPassword(string email);
    List<User> Search(string query);
    UserStats GetStatistics();
}

// GOOD: Segregated interfaces
public interface IUserReader
{
    User GetById(string id);
    List<User> Search(string query);
}

public interface IUserWriter
{
    User Create(CreateUserDto dto);
    User Update(UpdateUserDto dto);
    void Delete(string id);
}

public interface IUserNotifier
{
    void SendVerificationEmail(string userId);
    void ResetPassword(string email);
}
```

### API Endpoint Segregation

```
BAD: One monolithic API serving all clients
  /api/users — returns 50 fields for every consumer

GOOD: Tailored endpoints per consumer need
  /api/users/summary    — Mobile app (minimal fields)
  /api/users/detail     — Admin dashboard (all fields)
  /api/users/export     — Reporting service (specific format)
```

## Dependency Inversion Principle (DIP) — Abstract Boundaries

High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Layer Architecture with DIP

```
Presentation Layer (Controllers/Views)
       │
       ▼ depends on
Application Layer (Services/Use Cases)
       │
       ▼ depends on (abstractions only!)
Domain Layer (Entities, Repository Interfaces)
       ▲
       │ implements
Infrastructure Layer (Database, External APIs, File System)
```

### Dependency Injection Configuration

```csharp
// ASP.NET Core — wire abstractions to implementations
services.AddScoped<IUserRepository, PostgresUserRepository>();
services.AddScoped<IEmailService, SendGridEmailService>();
services.AddScoped<ICacheService, RedisCacheService>();

// Swap implementations without changing business logic
if (environment.IsDevelopment())
{
    services.AddScoped<IEmailService, ConsoleEmailService>();
    services.AddScoped<ICacheService, InMemoryCacheService>();
}
```

### Cross-Service DIP

```
OrderService depends on IPaymentGateway (abstraction)
                         ▲
                         │ implements
             StripeGateway (infrastructure)

OrderService depends on IInventoryChecker (abstraction)
                         ▲
                         │ implements
             InventoryServiceClient (infrastructure)
```

Services communicate through abstract contracts (interfaces, API schemas), not concrete implementations.

## Applying SOLID to Agent Orchestration

| Principle | Architecture Application |
|-----------|------------------------|
| SRP | Each agent owns one technology domain |
| OCP | New agents extend the team without modifying existing ones |
| LSP | All agents follow the same task protocol (receive task → produce output) |
| ISP | Agents only receive tasks within their domain |
| DIP | Agents depend on shared contracts (API schemas), not other agents' internals |
