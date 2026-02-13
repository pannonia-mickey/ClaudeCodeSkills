# Orchestration Patterns

## Delegation Strategies

### Capability-Based Delegation

Assign tasks based on the agent's declared expertise:

```
Task: "Create a Django REST API for user management"
Analysis:
  - Technology: Django, DRF
  - Domain: Backend API
  - Delegate to: django-expert
  - Reason: Direct match on Django + DRF capability
```

### Domain-Based Delegation

When a task spans multiple technologies, delegate to the agent owning the primary domain:

```
Task: "Build a FastAPI service that uses SQLAlchemy and Alembic"
Analysis:
  - Primary: FastAPI (framework)
  - Secondary: SQLAlchemy (database layer)
  - Delegate to: fastapi-expert
  - Reason: FastAPI-specific integration patterns for SQLAlchemy
```

### Cascade Delegation

For tasks requiring sequential handoffs between agents:

```
Task: "Build a full-stack feature with Django backend and React frontend"
Cascade:
  1. django-expert → Design models, create API endpoints, define response schemas
  2. react-expert → Consume API, build UI components, implement state management
  Integration point: API contract (OpenAPI schema from DRF)
```

## Dependency Graph Patterns

### Linear Dependencies

```
[Models] → [API Endpoints] → [Frontend Components] → [Integration Tests]
  django      django/fastapi      react                  all agents
```

### Parallel with Convergence

```
[Django Models] ──→ [DRF API] ──────────────┐
                                             ├─→ [Integration]
[React Scaffolding] → [React Components] ───┘
```

Both backend and frontend can start independently, converging at integration.

### Diamond Dependencies

```
         ┌→ [Auth Service (FastAPI)] ─┐
[Shared  │                            ├→ [API Gateway]
 Types] ─┤                            │
         └→ [User Service (Django)] ──┘
```

Shared types must complete before parallel services can start.

## Parallel Execution Strategies

### Independent Parallel

When tasks have no dependencies, execute simultaneously:

```
Phase 2 (Parallel):
  - django-expert: Implement User model and API
  - react-expert: Build UI shell and routing
  - langchain-expert: Design RAG pipeline
```

### Phased Parallel

Group tasks into phases; tasks within a phase run in parallel, phases run sequentially:

```
Phase 1 (Sequential prerequisite):
  - python-expert: Create shared utilities and base classes

Phase 2 (Parallel):
  - django-expert: Backend API using shared utilities
  - fastapi-expert: Microservice using shared utilities

Phase 3 (Parallel, depends on Phase 2):
  - react-expert: Frontend consuming both APIs
  - langchain-expert: AI service consuming backend data
```

### Pipeline Parallel

Each agent works on the next item while another finishes the previous:

```
Feature 1: django-expert (building) → react-expert (waiting)
Feature 2: django-expert (waiting)  → react-expert (building previous)
```

## Handoff Protocols

### API Contract Handoff

When backend agents hand off to frontend agents:

1. Backend agent produces an OpenAPI/JSON schema
2. Schema defines: endpoints, request/response shapes, auth requirements
3. Frontend agent consumes the schema to build API integration
4. Both agents validate against the shared schema

### Data Model Handoff

When database design flows to API design:

1. Database agent defines entity relationships and field types
2. API agent creates serializers/schemas matching the data models
3. Validation: serializer fields map 1:1 to model fields (or explicitly transform)

### Integration Test Handoff

When individual components are ready for integration:

1. Each agent provides unit-tested components
2. Orchestrator defines integration test scenarios
3. Tests verify cross-agent boundaries (API calls, data flow, error propagation)

## Conflict Resolution

### Technology Overlap

When two agents could handle a task:

| Conflict | Resolution Rule |
|----------|----------------|
| Python testing for Django app | django-expert (framework-specific) |
| Python testing for general library | python-expert (no framework) |
| Pydantic models in FastAPI | fastapi-expert (integration context) |
| Pydantic models standalone | python-expert (general usage) |
| TypeScript types for React | react-expert (component context) |
| Async Python in FastAPI | fastapi-expert (framework patterns) |
| Async Python general | python-expert (language patterns) |

### Scope Creep Resolution

When an agent's task expands beyond their domain:

1. Agent completes work within their domain boundary
2. Agent documents the handoff point and requirements for the next agent
3. Orchestrator assigns the overflow to the appropriate specialist
4. Never have one agent work outside their declared expertise

## Estimation Heuristics

### Task Complexity Categories

| Complexity | Description | Typical Subtasks |
|-----------|-------------|-----------------|
| Simple | Single agent, single concern | 1-3 subtasks |
| Moderate | Single agent, multiple concerns | 3-7 subtasks |
| Complex | Multiple agents, dependencies | 7-15 subtasks |
| Large | Full project, all phases | 15+ subtasks |

### Dependency Impact

- Each cross-agent dependency adds coordination overhead
- Minimize dependencies by defining clear contracts early
- Prefer parallel-friendly architectures (shared nothing, contract-based)
