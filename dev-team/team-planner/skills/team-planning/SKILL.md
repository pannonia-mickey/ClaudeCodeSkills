---
name: Team Planning
description: This skill should be used when the user asks to "coordinate the team", "plan team tasks", "delegate to agents", "orchestrate development", "assign agent tasks", "create execution plan", or needs to understand which specialist agent handles which domain.
---

# Team Planning

## Overview

This skill provides the orchestration framework for coordinating specialist agents across multi-technology projects. It covers task analysis, agent delegation, execution ordering, and cross-agent integration.

## Agent Capability Matrix

| Agent | Primary Domain | Key Capabilities |
|-------|---------------|-----------------|
| django-expert | Django 5.+, DRF | Models, views, serializers, migrations, admin, signals, middleware |
| react-expert | React 18+, TypeScript | Components, hooks, state management, Next.js, testing |
| fastapi-expert | FastAPI, Pydantic v2 | Async endpoints, DI with Depends(), WebSockets, OpenAPI |
| python-expert | Core Python | Design patterns, type hints, packaging, pytest, async/await |
| csharp-expert | C#/.NET 8+ | ASP.NET Core, EF Core, LINQ, DI, Blazor, xUnit |
| langchain-expert | LangChain, LCEL | Chains, agents, RAG, vector stores, LangSmith |
| angularjs-expert | AngularJS 1.x | Components, directives, services, ui-router, security, migration |

## Task Analysis Workflow

### Step 1: Requirement Decomposition

Break the user request into discrete functional units:
- Identify each technology stack involved
- Separate backend, frontend, AI/ML, and infrastructure concerns
- List data models, APIs, and UI components needed

### Step 2: Agent Assignment

Map each functional unit to a specialist agent:
- Match by primary technology (Django → django-expert)
- For overlapping domains, prefer the more specific agent
- General Python work goes to python-expert only when no framework-specific agent applies

### Step 3: Dependency Mapping

Identify task ordering constraints:
- **Data layer first**: Database models before API endpoints
- **API before UI**: Backend endpoints before frontend integration
- **Core before extensions**: Base functionality before AI features
- **Shared contracts early**: Define API schemas before parallel work

### Step 4: Execution Plan

Create a phased plan:
- **Phase 1 — Foundation**: Data models, project scaffolding, shared types
- **Phase 2 — Core Backend**: API endpoints, business logic, authentication
- **Phase 3 — Frontend/Integration**: UI components, API integration, state management
- **Phase 4 — Enhancement**: AI features, performance optimization, testing
- **Phase 5 — Polish**: Security hardening, documentation, deployment

## Overlap Resolution Rules

When a task could belong to multiple agents:

| Scenario | Assign To | Reason |
|----------|-----------|--------|
| Django model testing | django-expert | Framework-specific test patterns |
| General Python utility | python-expert | No framework dependency |
| FastAPI + SQLAlchemy | fastapi-expert | Framework integration knowledge |
| React + TypeScript types | react-expert | Component-specific typing |
| Python async patterns | python-expert | General language feature |
| LangChain + FastAPI integration | langchain-expert | AI-specific orchestration |

## Cross-Agent Integration Patterns

### API Contract Pattern
Define shared API contracts before agents work in parallel:
- Request/response schemas (JSON/Pydantic/TypeScript interfaces)
- Endpoint URLs and HTTP methods
- Authentication headers and token formats
- Error response shapes

### Shared Types Pattern
Create shared type definitions that multiple agents reference:
- Database entity shapes
- API DTO (Data Transfer Object) schemas
- Event payload formats

### Integration Checkpoint Pattern
Schedule verification points where agents' work must align:
- After Phase 1: Verify models are consistent across backends
- After Phase 2: Verify API contracts match frontend expectations
- After Phase 3: End-to-end integration test

## SOLID at the Orchestration Level

- **SRP**: Each agent owns exactly one technology domain
- **OCP**: New agents can be added without modifying existing ones
- **LSP**: All agents follow the same task interface (input → output)
- **ISP**: Agents only receive tasks relevant to their domain
- **DIP**: Agents depend on abstract contracts (API schemas), not concrete implementations

## Quick Reference: Common Project Patterns

### Full-Stack Web App (Django + React)
1. django-expert → Models, DRF APIs, auth
2. react-expert → Components, state, API integration
3. python-expert → Shared utilities, testing infrastructure

### Microservices (FastAPI)
1. python-expert → Shared libraries, base patterns
2. fastapi-expert → Service endpoints, inter-service communication
3. team-planner → Service boundary definitions

### AI-Powered Application
1. langchain-expert → Chains, RAG, agent design
2. fastapi-expert or django-expert → API layer
3. react-expert → Chat UI, streaming display

### .NET Enterprise Application
1. csharp-expert → Domain, services, EF Core, APIs
2. react-expert → Frontend (if SPA)
3. python-expert → Scripting/automation (if needed)

## Additional Resources

### Reference Files

For detailed patterns and techniques, consult:

- **`references/orchestration-patterns.md`** — Delegation strategies, dependency graphs, parallel execution patterns
- **`references/solid-architecture.md`** — SOLID principles applied at the system architecture level with concrete examples
