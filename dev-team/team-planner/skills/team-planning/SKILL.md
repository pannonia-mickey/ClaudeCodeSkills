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
| angular-expert | Angular 18+ | Components, signals, RxJS, routing, NgRx, testing |
| cloud-expert | AWS, Serverless | Lambda, CDK, multi-region DR, cost optimization, IAM |
| cpp-expert | C++ (C++11–C++23) | Systems programming, memory safety, concurrency, CMake |
| database-expert | SQL, NoSQL, ORMs | Schema design, query optimization, migrations, Redis, MongoDB |
| devops-expert | CI/CD, IaC, Monitoring | GitHub Actions, Terraform, Prometheus, Grafana, incident management |
| django-expert | Django 5.+, DRF | Models, views, serializers, migrations, admin, signals, middleware |
| docker-expert | Docker, Compose | Dockerfiles, multi-stage builds, Compose, container security |
| dotnet-expert | C#/.NET 8+, ASP.NET | ASP.NET Core, EF Core, LINQ, DI, Blazor, xUnit, Minimal APIs |
| fastapi-expert | FastAPI, Pydantic v2 | Async endpoints, DI with Depends(), WebSockets, OpenAPI |
| go-expert | Go 1.22+ | net/http, Gin, gRPC, goroutines, channels, CLIs (cobra) |
| langchain-expert | LangChain, LCEL | Chains, agents, RAG, vector stores, LangSmith |
| nextjs-expert | Next.js 14+ | App Router, RSC, Server Actions, ISR, caching, middleware |
| nodejs-expert | Node.js 20+ | Express, NestJS, event loop, streams, worker threads |
| python-expert | Core Python | Design patterns, type hints, packaging, pytest, async/await |
| react-expert | React 18+, TypeScript | Components, hooks, state management, testing |
| rust-expert | Rust | Ownership, traits, async/Tokio, FFI, WebAssembly |
| seo-expert | Technical SEO | Meta tags, structured data, Core Web Vitals, crawlability |
| tailwind-expert | Tailwind CSS | Utility-first styling, responsive design, custom themes |
| testing-expert | QA, E2E, Performance | Test strategy, Playwright, k6, contract testing, security scanning |
| typescript-expert | TypeScript 5+ | Type system, generics, tooling, migration, project config |
| uxui-designer | UX/UI Design | Design systems, accessibility, responsive layouts, component specs |
| vue-expert | Vue 3, Nuxt 3 | Composition API, Pinia, Vue Router, SSR, component design |

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
| Next.js page with React components | nextjs-expert | Next.js-specific patterns (RSC, routing) |
| Node.js + TypeScript backend | nodejs-expert | Runtime-specific knowledge |
| TypeScript type utility library | typescript-expert | Pure type system work |
| Vue + Nuxt SSR | vue-expert | Framework-specific SSR patterns |
| Angular + TypeScript types | angular-expert | Angular-specific type patterns |
| CI/CD pipeline setup | devops-expert | Pipeline and deployment expertise |
| Docker + CI integration | docker-expert (containers) + devops-expert (CI) | Split by concern |
| AWS infrastructure with Terraform | cloud-expert (architecture) + devops-expert (IaC) | Split by concern |
| Database schema for Django | django-expert | Framework ORM integration |
| Database schema (standalone) | database-expert | Pure database design |
| E2E test strategy | testing-expert | Cross-cutting test expertise |
| Go gRPC service | go-expert | Go-specific patterns |
| Rust + C FFI | rust-expert (Rust side) + cpp-expert (C side) | Split by language |
| .NET + EF Core migrations | dotnet-expert | Framework-specific ORM |

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
1. dotnet-expert → Domain, services, EF Core, APIs
2. react-expert → Frontend (if SPA)
3. database-expert → Schema design, query optimization

### Full-Stack Node.js App
1. nodejs-expert → Express/NestJS APIs, middleware
2. typescript-expert → Shared types, strict config
3. react-expert → Frontend components, state
4. database-expert → Data layer, migrations

### Cloud-Native Microservices
1. go-expert → Service implementation, gRPC
2. docker-expert → Containerization
3. cloud-expert → AWS infrastructure, serverless
4. devops-expert → CI/CD pipelines, monitoring

### Modern Angular App
1. angular-expert → Components, signals, routing
2. typescript-expert → Shared types, tooling
3. tailwind-expert → Styling, design system

### Vue/Nuxt Full-Stack
1. vue-expert → Components, Pinia, Nuxt server routes
2. typescript-expert → Type safety, shared types
3. database-expert → Data layer

### End-to-End Testing Pipeline
1. testing-expert → Strategy, Playwright tests, k6 load tests
2. devops-expert → CI integration, test environments

### Rust Systems Project
1. rust-expert → Core implementation, async/Tokio
2. devops-expert → CI/CD, cross-compilation
3. testing-expert → Fuzzing strategy, benchmarks

## Additional Resources

### Reference Files

For detailed patterns and techniques, consult:

- **`references/orchestration-patterns.md`** — Delegation strategies, dependency graphs, parallel execution patterns
- **`references/solid-architecture.md`** — SOLID principles applied at the system architecture level with concrete examples
