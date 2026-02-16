# Diagramming Techniques

## ASCII Box Diagrams

### Layer Diagram

```
┌─────────────────────────────────────────────────┐
│                 Presentation Layer                │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  Routes   │  │  Views   │  │  Middleware   │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
├─────────────────────────────────────────────────┤
│                 Application Layer                │
│  ┌──────────────┐  ┌────────────────────────┐   │
│  │  Use Cases    │  │  Application Services  │   │
│  └──────────────┘  └────────────────────────┘   │
├─────────────────────────────────────────────────┤
│                  Domain Layer                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Entities  │  │  Values  │  │  Domain Svc  │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
├─────────────────────────────────────────────────┤
│               Infrastructure Layer               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Database  │  │  Cache   │  │  External API │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
```

### Service Communication Diagram

```
┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client   │──HTTP──▶│  Gateway  │──HTTP──▶│  Auth    │
└──────────┘         └────┬─────┘         └──────────┘
                          │
                    ┌─────┴─────┐
                    │           │
              ┌─────▼────┐ ┌───▼──────┐
              │  Orders  │ │  Users   │
              └────┬─────┘ └────┬─────┘
                   │            │
              ┌────▼────────────▼────┐
              │      PostgreSQL       │
              └──────────────────────┘
```

### Data Flow Diagram

```
Request ──▶ [Auth Middleware] ──▶ [Route Handler] ──▶ [Service]
                                                        │
                                                   [Repository]
                                                        │
                                                   [Database]
                                                        │
                                                   [Repository]
                                                        │
Response ◀── [Serializer] ◀──── [Service] ◀──── [Domain Entity]
```

### ASCII Building Blocks

```
Boxes:      ┌──────────┐    ╔══════════╗
            │  Normal   │    ║  Double   ║
            └──────────┘    ╚══════════╝

Arrows:     ──▶  ◀──  ──▷  ◁──  ───  │  ──┐
                                          │
                                          ▼

Connectors: ┌──┬──┐    ├──┼──┤    └──┴──┘
```

## Mermaid Diagrams

### Component Diagram

```mermaid
graph TD
    subgraph Presentation
        A[Web App] --> B[API Routes]
        C[Mobile App] --> B
    end

    subgraph Application
        B --> D[Auth Service]
        B --> E[Order Service]
        B --> F[User Service]
    end

    subgraph Infrastructure
        D --> G[(PostgreSQL)]
        E --> G
        F --> G
        E --> H[(Redis)]
    end
```

### Sequence Diagram — Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant M as Middleware
    participant R as Router
    participant S as Service
    participant DB as Database

    C->>M: HTTP Request
    M->>M: Validate JWT
    M->>R: Authenticated Request
    R->>S: Call service method
    S->>DB: Query data
    DB-->>S: Result set
    S-->>R: Domain object
    R-->>C: JSON Response
```

### Class/Module Relationship Diagram

```mermaid
classDiagram
    class UserController {
        +createUser(dto)
        +getUser(id)
        +updateUser(id, dto)
    }
    class UserService {
        +create(data)
        +findById(id)
        +update(id, data)
    }
    class UserRepository {
        +save(user)
        +findById(id)
        +update(user)
    }
    class User {
        +id: string
        +email: string
        +name: string
    }

    UserController --> UserService
    UserService --> UserRepository
    UserRepository --> User
```

### Dependency Graph

```mermaid
graph LR
    auth --> database
    auth --> config
    users --> auth
    users --> database
    orders --> users
    orders --> payments
    orders --> database
    payments --> auth
    payments --> database
    notifications --> orders
    notifications --> users
```

### State Diagram

```mermaid
stateDiagram-v2
    [*] --> Pending
    Pending --> Processing: payment_received
    Processing --> Shipped: items_packed
    Processing --> Cancelled: cancel_request
    Shipped --> Delivered: delivery_confirmed
    Shipped --> Returned: return_request
    Delivered --> [*]
    Cancelled --> [*]
    Returned --> Refunded
    Refunded --> [*]
```

### Entity Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ ORDER : places
    ORDER ||--|{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "ordered in"
    USER {
        uuid id PK
        string email UK
        string name
        timestamp created_at
    }
    ORDER {
        uuid id PK
        uuid user_id FK
        string status
        decimal total
    }
```

## Dependency Graph Visualization

### Module Dependency Matrix

```
              auth  users  orders  payments  notifications
auth           -      -      -       -           -
users          ✓      -      -       -           -
orders         -      ✓      -       ✓           -
payments       ✓      -      -       -           -
notifications  -      ✓      ✓       -           -
```

Read as: row depends on column. Example: `orders` depends on `users` and `payments`.

### Generating Dependency Data

```bash
# Extract imports per module and build adjacency list
for dir in src/*/; do
  module=$(basename "$dir")
  deps=$(grep -rh "from '\.\./\|from \"\.\./" "$dir" --include="*.ts" 2>/dev/null \
    | sed "s/.*from ['\"]\.\.\/\([^/]*\).*/\1/" | sort -u | tr '\n' ',')
  echo "$module -> $deps"
done

# Python version
for dir in src/*/; do
  module=$(basename "$dir")
  deps=$(grep -rh "from \.\." "$dir" --include="*.py" 2>/dev/null \
    | sed 's/from \.\.\([a-zA-Z_]*\).*/\1/' | sort -u | tr '\n' ',')
  echo "$module -> $deps"
done
```

## Best Practices

1. **Start simple** — Use ASCII for quick inline diagrams, Mermaid for documentation
2. **Label connections** — Always annotate arrows with the communication type (HTTP, gRPC, events)
3. **Show direction** — Dependency arrows should point from dependent to dependency
4. **Group by concern** — Use subgraphs/boxes to group related components
5. **Include data stores** — Always show databases, caches, and queues
6. **Keep it readable** — Max 15-20 nodes per diagram; split complex systems into multiple views
