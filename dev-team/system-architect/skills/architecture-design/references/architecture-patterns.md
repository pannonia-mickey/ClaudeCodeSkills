# Architecture Patterns Reference

## Monolith

### Description
Single deployable unit containing all application functionality.

### Structure
```
app/
├── users/          # User domain
├── orders/         # Order domain
├── payments/       # Payment domain
├── notifications/  # Notification domain
├── shared/         # Shared utilities
└── main.py         # Single entry point
```

### When to Use
- Small team (1-5 developers)
- Early-stage product with uncertain requirements
- Simple domain with low complexity
- Rapid prototyping and MVP development

### Pros
- Simple deployment and operations
- Easy local development
- No network overhead between components
- Simple transaction management (single database)
- Fast initial development velocity

### Cons
- Scaling requires scaling everything
- Large codebase becomes hard to navigate
- Single point of failure
- Technology lock-in (one stack for everything)
- Deployment risk (all-or-nothing releases)

---

## Modular Monolith

### Description
Single deployment unit with strictly enforced module boundaries and explicit interfaces between modules.

### Structure
```
app/
├── modules/
│   ├── users/
│   │   ├── api.py           # Public interface
│   │   ├── models.py        # Internal models
│   │   ├── services.py      # Internal logic
│   │   └── __init__.py      # Exports only public API
│   ├── orders/
│   │   ├── api.py
│   │   ├── models.py
│   │   └── services.py
│   └── payments/
│       ├── api.py
│       └── services.py
├── shared/
│   └── events.py            # Inter-module events
└── main.py
```

### Key Rule
Modules communicate ONLY through public APIs, never by importing internal classes.

### When to Use
- Medium teams (3-10 developers)
- Domains with clear bounded contexts
- Projects that may evolve to microservices later
- When you want microservice boundaries without operational complexity

### Pros
- Clear module boundaries enable team autonomy
- Can extract modules to microservices later
- Single deployment simplicity
- Database transactions within the monolith
- Faster than microservices (no network hops)

### Cons
- Requires discipline to maintain boundaries
- Still single deployment unit
- Shared database can create coupling

---

## Microservices

### Description
System composed of independently deployable services, each owning its data and communicating via APIs or events.

### Structure
```
services/
├── user-service/          # Own database, own deployment
│   ├── app/
│   ├── Dockerfile
│   └── docker-compose.yml
├── order-service/
│   ├── app/
│   ├── Dockerfile
│   └── docker-compose.yml
├── payment-service/
├── api-gateway/           # Routes requests to services
└── shared-contracts/      # API schemas, event definitions
```

### When to Use
- Large teams (10+ developers) with autonomous squads
- High scalability requirements (independent scaling)
- Polyglot technology needs (different services, different stacks)
- High deployment frequency (multiple deploys per day)

### Pros
- Independent scaling and deployment
- Technology diversity per service
- Fault isolation (one service failure doesn't crash all)
- Team autonomy

### Cons
- Operational complexity (monitoring, tracing, deployment)
- Network latency between services
- Distributed transaction management is hard
- Data consistency challenges (eventual consistency)
- Testing complexity (integration tests span services)

---

## Hexagonal Architecture (Ports & Adapters)

### Description
Application core is isolated from external concerns via ports (interfaces) and adapters (implementations).

### Structure
```
app/
├── core/                  # Business logic — NO external dependencies
│   ├── domain/            # Entities, value objects
│   ├── ports/             # Interfaces (inbound + outbound)
│   │   ├── inbound/       # Use case interfaces
│   │   └── outbound/      # Repository, gateway interfaces
│   └── services/          # Use case implementations
├── adapters/
│   ├── inbound/           # REST controllers, CLI, GraphQL
│   │   ├── rest/
│   │   └── graphql/
│   └── outbound/          # Database, external API clients
│       ├── persistence/
│       └── external_api/
└── config/                # Wiring adapters to ports
```

### When to Use
- Domain-heavy applications
- Systems requiring high testability
- Applications needing multiple interfaces (REST + GraphQL + CLI)
- Long-lived systems where external dependencies will change

### Pros
- Business logic fully isolated and testable
- Easy to swap adapters (change database, add new API format)
- Clear dependency direction
- Framework-agnostic core

### Cons
- More boilerplate (ports + adapters)
- Learning curve for team
- Can be overkill for simple CRUD apps

---

## Clean Architecture

### Description
Concentric layers with dependencies pointing inward. The domain is at the center with zero external dependencies.

### Layers (Inside Out)
```
┌─────────────────────────────────┐
│  Frameworks & Drivers (outer)   │  Web, DB, External APIs
│  ┌───────────────────────────┐  │
│  │  Interface Adapters       │  │  Controllers, Gateways, Presenters
│  │  ┌───────────────────┐   │  │
│  │  │  Application      │   │  │  Use Cases, DTOs
│  │  │  ┌─────────────┐  │   │  │
│  │  │  │  Domain      │  │   │  │  Entities, Value Objects
│  │  │  └─────────────┘  │   │  │
│  │  └───────────────────┘   │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### When to Use
- Enterprise applications with complex business rules
- Systems requiring strict separation of concerns
- .NET / Java projects (strong language support for interfaces)
- Long-term projects where the domain outlives the framework

---

## Event-Driven Architecture

### Description
Components communicate through events rather than direct calls. Producers emit events; consumers react to them.

### Patterns
- **Event Notification**: Simple notification that something happened
- **Event-Carried State Transfer**: Event contains all data needed
- **Event Sourcing**: Store events as source of truth, derive state

### When to Use
- Highly decoupled systems
- Asynchronous processing requirements
- Audit trail requirements (event sourcing)
- Systems with fan-out behavior (one event → many reactions)

---

## CQRS (Command Query Responsibility Segregation)

### Description
Separate the read model (queries) from the write model (commands). Each can have its own data store and optimization strategy.

### Structure
```
Commands (writes)          Queries (reads)
    │                          │
    ▼                          ▼
Command Handlers           Query Handlers
    │                          │
    ▼                          ▼
Write Database             Read Database
(normalized,               (denormalized,
 transactional)             optimized for reads)
```

### When to Use
- Read/write ratio is highly asymmetric (many more reads)
- Complex read models (aggregations, projections)
- Different scaling needs for reads vs writes
- Often paired with Event Sourcing

---

## Serverless

### Description
Application logic deployed as individual functions triggered by events. No server management.

### When to Use
- Sporadic or unpredictable traffic
- Event-triggered processing (file uploads, webhooks)
- Cost optimization for low-traffic applications
- Rapid prototyping without infrastructure concerns

### Cons
- Cold start latency
- Vendor lock-in
- Complex local development
- Function duration limits
- State management challenges
