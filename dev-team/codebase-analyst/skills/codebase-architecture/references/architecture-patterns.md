# Architecture Patterns — Detection Guide

## Layered / N-Tier Architecture

### Characteristics
- Clear horizontal separation: presentation, business, data access
- Each layer only calls the layer directly below
- Common in traditional web applications

### Detection Heuristics

```bash
# Look for layered directory structure
find . -maxdepth 2 -type d | grep -iE 'controller|service|repository|model|view'

# Verify import direction (top-down only)
# Controllers should import services, NOT the reverse
grep -rn "from.*controller" --include="*.py" | grep -v "test\|controller/" | head -10

# Check for strict layer boundaries
# If services import controllers, layers are violated
grep -rn "import.*controller\|from.*controller" --include="*.py" \
  | grep "service/" && echo "LAYER VIOLATION: service imports controller"
```

### Typical Structure
```
src/
├── controllers/    # HTTP handlers (presentation)
├── services/       # Business logic
├── repositories/   # Data access
├── models/         # Data structures
└── middleware/      # Cross-cutting concerns
```

## Clean Architecture / Hexagonal / Ports & Adapters

### Characteristics
- Domain/core at the center, independent of frameworks
- Dependencies point inward (infrastructure depends on domain, not reverse)
- Explicit ports (interfaces) and adapters (implementations)
- Use cases / interactors as application orchestrators

### Detection Heuristics

```bash
# Look for domain-centric structure
find . -maxdepth 2 -type d | grep -iE 'domain|core|entity|use.?case|port|adapter|interactor'

# Check for interface/port definitions
grep -rn "interface.*Port\|interface.*Repository\|Protocol\|ABC" \
  --include="*.ts" --include="*.py" | head -20

# Verify domain has zero framework imports
grep -rn "import.*express\|import.*django\|import.*fastapi\|import.*gin" \
  $(find . -path '*/domain/*' -name '*.py' -o -path '*/domain/*' -name '*.ts') 2>/dev/null
# Should return nothing if clean architecture is followed
```

### Typical Structure
```
src/
├── domain/           # Entities, value objects (no framework deps)
│   ├── entities/
│   └── value-objects/
├── application/      # Use cases, ports (interfaces)
│   ├── use-cases/
│   └── ports/
├── infrastructure/   # Adapters (implementations)
│   ├── database/
│   ├── http/
│   └── messaging/
└── presentation/     # Controllers, CLI, UI
```

## Microservices

### Characteristics
- Multiple independently deployable services
- Each service owns its data store
- Inter-service communication via HTTP/gRPC/messaging
- Service discovery and API gateway

### Detection Heuristics

```bash
# Multiple Dockerfiles or services in compose
find . -name 'Dockerfile*' | wc -l
grep -c "services:" docker-compose*.yml 2>/dev/null

# Multiple package manifests (one per service)
find . -name 'package.json' -not -path '*/node_modules/*' | wc -l
find . -name 'go.mod' | wc -l

# Service directory pattern
find . -maxdepth 2 -type d -name 'services' -o -name 'apps' -o -name 'microservices'
ls services/ 2>/dev/null || ls apps/ 2>/dev/null

# Inter-service communication
grep -rn "http://\|https://\|grpc\.\|nats\.\|kafka\." \
  --include="*.ts" --include="*.py" --include="*.go" | grep -v test | head -20

# API gateway configuration
find . -name 'nginx.conf' -o -name 'kong.yml' -o -name 'traefik.*' | head -5
```

### Typical Structure
```
project/
├── services/
│   ├── user-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   ├── order-service/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── src/
│   └── payment-service/
├── gateway/
├── shared/              # Shared contracts/types
└── docker-compose.yml
```

## Event-Driven Architecture

### Characteristics
- Components communicate via events, not direct calls
- Event bus / message broker as central infrastructure
- Eventual consistency between components
- Event sourcing optionally stores state as event log

### Detection Heuristics

```bash
# Event definitions
grep -rn "class.*Event\|interface.*Event\|type.*Event" \
  --include="*.ts" --include="*.py" --include="*.go" | head -20

# Event bus / emitter usage
grep -rn "EventEmitter\|EventBus\|publish\|subscribe\|dispatch\|emit" \
  --include="*.ts" --include="*.py" | head -20

# Message broker configuration
grep -rn "kafka\|rabbitmq\|redis.*stream\|aws.*sqs\|nats" \
  --include="*.ts" --include="*.py" --include="*.yml" --include="*.yaml" | head -10

# Event handler directories
find . -maxdepth 3 -type d | grep -iE 'event|handler|listener|subscriber|consumer'
```

## Monorepo Patterns

### Characteristics
- Multiple packages/apps in a single repository
- Shared tooling and configuration
- Workspace manager for dependency resolution
- Build caching and task orchestration

### Detection Heuristics

```bash
# Workspace configuration
grep -l "workspaces" package.json 2>/dev/null
find . -maxdepth 1 -name 'nx.json' -o -name 'turbo.json' -o -name 'lerna.json' \
  -o -name 'pnpm-workspace.yaml' -o -name 'rush.json'

# Multiple packages
ls packages/ 2>/dev/null | wc -l
ls apps/ 2>/dev/null | wc -l

# Shared packages
find . -maxdepth 3 -path '*/packages/shared/*' -o -path '*/packages/common/*' \
  -o -path '*/packages/utils/*' 2>/dev/null | head -10

# Internal package references
grep -rn "@myorg/\|workspace:" --include="package.json" | head -10
```

### Typical Structure
```
monorepo/
├── apps/
│   ├── web/           # Frontend application
│   ├── api/           # Backend API
│   └── admin/         # Admin dashboard
├── packages/
│   ├── ui/            # Shared UI components
│   ├── config/        # Shared configuration
│   └── types/         # Shared TypeScript types
├── turbo.json
└── package.json       # Root workspace config
```

## MVC / MVVM / MVP Variations

### Detection Heuristics

```bash
# MVC indicators
find . -maxdepth 3 -type d | grep -iE 'model|view|controller'

# MVVM indicators (common in Angular, WPF)
find . -maxdepth 3 -type d | grep -iE 'model|view|viewmodel'
grep -rn "ViewModel\|@observable\|@computed" --include="*.ts" | head -10

# MVP indicators
find . -maxdepth 3 -type d | grep -iE 'model|view|presenter'
```

## CQRS (Command Query Responsibility Segregation)

### Detection Heuristics

```bash
# Separate command and query paths
find . -maxdepth 3 -type d | grep -iE 'command|query|read.?model|write.?model'

# Command/Query handler patterns
grep -rn "CommandHandler\|QueryHandler\|class.*Command\b\|class.*Query\b" \
  --include="*.ts" --include="*.py" | head -20

# Separate read/write database connections
grep -rn "readDatabase\|writeDatabase\|replica\|primary" \
  --include="*.ts" --include="*.py" --include="*.yml" | head -10
```
