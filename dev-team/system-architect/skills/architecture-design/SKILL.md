---
name: Architecture Design
description: This skill should be used when the user asks to "design system architecture", "create an architecture", "choose architecture pattern", "design the system", "make an architecture decision", "evaluate architecture options", "recommend a folder structure", "define component boundaries", or needs guidance on system structure, layer separation, SOLID principles, or technology selection.
---

# Architecture Design

## Overview

This skill provides frameworks for designing new system architectures, selecting appropriate patterns, making technology decisions, and documenting architectural choices.

## Architecture Selection Decision Tree

### Step 1: Assess Scale and Team

| Factor | Small (1-3 devs) | Medium (4-10 devs) | Large (10+ devs) |
|--------|------------------|--------------------|--------------------|
| Recommended start | Monolith | Modular Monolith | Microservices |
| Deployment | Single unit | Single unit, modular | Independent services |
| Communication | Function calls | Module interfaces | API/Events |

### Step 2: Identify Key Quality Attributes

Rank these non-functional requirements by importance:
- **Scalability**: Independent scaling of components?
- **Availability**: Uptime requirements (99.9% vs 99.99%)?
- **Performance**: Latency and throughput requirements?
- **Maintainability**: How frequently will components change?
- **Security**: Compliance requirements (HIPAA, PCI-DSS, GDPR)?
- **Deployability**: How often must deployments happen?

### Step 3: Match Pattern to Requirements

| Priority | Recommended Pattern |
|----------|-------------------|
| Scalability + Independent deployment | Microservices |
| Maintainability + Moderate scale | Modular Monolith |
| Performance + Low latency | Monolith or Modular Monolith |
| Event processing + Decoupling | Event-Driven Architecture |
| Read/write asymmetry | CQRS |
| Testability + Clean boundaries | Hexagonal / Clean Architecture |

## Layer Separation Principles

### Standard Layered Architecture

```
┌─────────────────────────┐
│   Presentation Layer    │  Controllers, Views, API endpoints
├─────────────────────────┤
│   Application Layer     │  Use cases, orchestration, DTOs
├─────────────────────────┤
│     Domain Layer        │  Entities, value objects, business rules
├─────────────────────────┤
│  Infrastructure Layer   │  Database, external APIs, file system
└─────────────────────────┘
```

**Key rule**: Dependencies flow downward only. Infrastructure implements domain interfaces.

### Dependency Rules

- Presentation → Application → Domain ← Infrastructure
- Domain layer has ZERO external dependencies
- Infrastructure implements interfaces defined in Domain
- Application orchestrates Domain objects and Infrastructure services

## Architecture Review Checklist

Before finalizing any architecture:

- [ ] **Separation of Concerns**: Each layer/module has a single responsibility
- [ ] **Dependency Direction**: No upward dependencies (Infrastructure does not leak into Domain)
- [ ] **Scalability Path**: Architecture supports horizontal scaling where needed
- [ ] **Testability**: Each component can be tested in isolation
- [ ] **Deployment Strategy**: Clear deployment unit boundaries
- [ ] **Error Handling**: Defined error propagation strategy across boundaries
- [ ] **Security Boundaries**: Authentication/authorization at correct layers
- [ ] **Observability**: Logging, metrics, and tracing strategy defined
- [ ] **Data Consistency**: Transaction boundaries and eventual consistency patterns identified
- [ ] **Evolution Path**: Architecture can evolve without full rewrite

## Technology Selection Criteria

When choosing between technologies, evaluate:

| Criterion | Weight | Questions |
|-----------|--------|-----------|
| Team expertise | High | Does the team know this technology? |
| Ecosystem maturity | High | Are libraries, tools, and docs available? |
| Community support | Medium | Is it actively maintained? |
| Performance fit | Medium | Does it meet NFRs? |
| Hiring market | Medium | Can new developers be found? |
| License | Low | Is the license compatible? |

## Common Architecture Combinations

### Django + React SPA
- Django serves DRF API + static files
- React handles all UI rendering
- JWT/session authentication
- Django Channels for WebSockets (if needed)

### FastAPI Microservices
- FastAPI for each service boundary
- Async communication (Redis Pub/Sub, RabbitMQ)
- Shared Pydantic models package
- API Gateway for routing

### .NET Clean Architecture
- ASP.NET Core Web API
- EF Core for data access
- MediatR for CQRS
- Domain-driven design in core layer

### AI-Augmented Application
- LangChain for AI pipeline
- FastAPI or Django for API layer
- Vector store (Chroma/Pinecone) for RAG
- React for chat UI with streaming

## Additional Resources

### Reference Files

For detailed patterns and decision frameworks, consult:

- **`references/architecture-patterns.md`** — Detailed coverage of Monolith, Microservices, Hexagonal, Clean Architecture, Event-Driven, CQRS, and Serverless patterns
- **`references/architecture-decisions.md`** — ADR templates, trade-off analysis frameworks (ATAM), scalability considerations, and NFR analysis
- **`references/solid-architecture.md`** — SOLID principles applied at the system architecture level with concrete code examples
