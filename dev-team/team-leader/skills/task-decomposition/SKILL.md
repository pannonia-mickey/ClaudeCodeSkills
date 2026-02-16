---
name: Task Decomposition
description: This skill should be used when the user asks to "break this task down", "decompose this project", "create subtasks", "split into tasks", "task breakdown", "create a work breakdown structure", or needs to split a complex requirement into agent-assignable subtasks.
---

# Task Decomposition

## Overview

This skill provides strategies for decomposing complex tasks into discrete, agent-assignable subtasks with clear dependencies and execution ordering.

## Decomposition Workflow

### Step 1: Identify Functional Boundaries

Break the requirement into distinct functional areas:
- **Data Layer**: Models, schemas, migrations, database design
- **Business Logic**: Services, use cases, validation rules
- **API Layer**: Endpoints, serializers, authentication
- **UI Layer**: Components, pages, state management
- **AI/ML Layer**: Chains, agents, embeddings, retrieval
- **Infrastructure**: Configuration, deployment, monitoring

### Step 2: Map to Specialist Agents

Assign each functional area to the agent with primary expertise:

| Functional Area | Primary Agent | Backup Agent |
|----------------|---------------|--------------|
| Angular components | angular-expert | typescript-expert |
| ASP.NET Core / .NET | dotnet-expert | — |
| C++ systems code | cpp-expert | rust-expert |
| CI/CD pipelines | devops-expert | docker-expert |
| Cloud infrastructure | cloud-expert | devops-expert |
| Database design/queries | database-expert | — |
| Django models/views | django-expert | python-expert |
| Docker containers | docker-expert | devops-expert |
| E2E/performance testing | testing-expert | — |
| FastAPI endpoints | fastapi-expert | python-expert |
| General Python | python-expert | — |
| Go services | go-expert | — |
| LangChain pipelines | langchain-expert | python-expert |
| Next.js features | nextjs-expert | react-expert |
| Node.js backend | nodejs-expert | typescript-expert |
| React components | react-expert | — |
| Rust systems code | rust-expert | cpp-expert |
| SEO optimization | seo-expert | — |
| Styling (Tailwind) | tailwind-expert | — |
| TypeScript types/tooling | typescript-expert | — |
| UX/UI design | uxui-designer | — |
| Vue components | vue-expert | typescript-expert |
| Architecture decisions | team-leader | — |

### Step 3: Define Dependencies

For each subtask, identify:
- **Blocks**: What tasks cannot start until this one finishes?
- **Blocked by**: What tasks must complete before this one can start?
- **Parallel**: What tasks can run simultaneously?

### Step 4: Order Execution

Apply these ordering rules:
1. Data models before API endpoints
2. API endpoints before frontend integration
3. Core functionality before enhancements
4. Shared contracts before parallel work
5. Unit tests alongside implementation
6. Integration tests after components exist

## Decomposition Strategies

### Vertical Slicing

Decompose by complete user-facing features, each touching all layers:

```
Feature: User Registration
  Task 1: User model + migration (django-expert)
  Task 2: Registration API endpoint (django-expert)
  Task 3: Registration form component (react-expert)
  Task 4: End-to-end test (team-leader coordinates)
```

**Best for**: Agile teams, incremental delivery, demos after each slice.

### Horizontal Slicing

Decompose by technical layer:

```
Layer: Data
  Task 1: All models + migrations (django-expert)
  Task 2: All database indexes (django-expert)

Layer: API
  Task 3: All endpoints (django-expert)
  Task 4: Authentication middleware (django-expert)

Layer: UI
  Task 5: All components (react-expert)
  Task 6: State management (react-expert)
```

**Best for**: Infrastructure work, database-heavy projects, shared foundation.

### Domain Decomposition

Decompose by business domain / bounded context:

```
Domain: Users
  Task 1: User model + API + UI (django-expert, react-expert)

Domain: Orders
  Task 2: Order model + API + UI (django-expert, react-expert)

Domain: AI Search
  Task 3: RAG pipeline + API + Chat UI (langchain-expert, fastapi-expert, react-expert)
```

**Best for**: Microservices, large teams, domain-driven design.

## Subtask Specification Template

```markdown
### Task {id}: {Title}

**Agent**: {agent-name}
**Blocked by**: {task-ids or "none"}
**Blocks**: {task-ids or "none"}
**Priority**: {P0/P1/P2}

**Description**:
{What needs to be done, specific enough for the agent to execute}

**Acceptance Criteria**:
- [ ] {Criterion 1}
- [ ] {Criterion 2}

**Integration Points**:
- {What this task produces that other tasks consume}
```

## Dependency Patterns

### Sequential Chain
```
Task A → Task B → Task C
(Each task depends on the previous)
```

### Fan-Out
```
Task A → Task B
Task A → Task C
Task A → Task D
(B, C, D all depend on A but are independent of each other)
```

### Fan-In
```
Task A ─┐
Task B ─┼→ Task D
Task C ─┘
(D depends on A, B, and C all completing)
```

### Diamond
```
      Task A
     ╱      ╲
Task B      Task C
     ╲      ╱
      Task D
```

## Common Pitfalls

- **Over-decomposition**: Don't create tasks smaller than ~30 minutes of work
- **Missing dependencies**: Always check if Task B needs Task A's output
- **Agent confusion**: One task should map to exactly one agent
- **Implicit contracts**: Always define the data format at integration points
- **Testing gaps**: Include test tasks alongside implementation tasks

## Additional Resources

### Reference Files

For detailed decomposition patterns and agent assignment guidance, consult:

- **`references/decomposition-patterns.md`** — Functional, domain, and vertical slicing patterns with examples
- **`references/agent-delegation-matrix.md`** — Complete matrix of agent capabilities, overlap resolution rules, and handoff protocols
