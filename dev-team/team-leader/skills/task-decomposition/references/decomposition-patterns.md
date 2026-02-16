# Decomposition Patterns

## Functional Decomposition

Break a system into its functional components — each function becomes a separate task.

### Example: E-Commerce Platform

```
Functional Units:
├── Authentication (register, login, password reset, OAuth)
├── Product Catalog (CRUD, search, categories, images)
├── Shopping Cart (add, remove, update quantity, persist)
├── Checkout (address, payment method, order summary)
├── Payment Processing (charge, refund, webhook handling)
├── Order Management (status tracking, history, cancellation)
├── Notifications (email confirmation, shipping updates)
└── Admin Dashboard (product management, order management, analytics)
```

### Task Mapping

```
Authentication → django-expert (Django auth + DRF token/JWT)
Product Catalog → django-expert (models + DRF) + react-expert (search UI)
Shopping Cart → react-expert (state) + django-expert (persistence API)
Payment Processing → django-expert or fastapi-expert (webhook handling)
Admin Dashboard → django-expert (Django admin customization)
```

### When to Use
- Greenfield projects with clear functional boundaries
- Teams with function-based ownership
- Traditional waterfall or V-model projects

### Pitfalls
- Functions often have hidden dependencies
- Cross-cutting concerns (auth, logging) touch everything
- Can lead to horizontal silos

---

## Domain Decomposition

Break a system along business domain boundaries (bounded contexts from DDD).

### Example: SaaS Platform

```
Domains:
├── Identity Context
│   ├── User registration and authentication
│   ├── Organization management
│   └── Role-based access control
├── Subscription Context
│   ├── Plan management
│   ├── Billing and invoicing
│   └── Usage tracking
├── Core Product Context
│   ├── Document processing
│   ├── AI-powered analysis
│   └── Collaboration features
└── Analytics Context
    ├── Usage dashboards
    ├── Business metrics
    └── Export and reporting
```

### Task Mapping

Each domain becomes an independent work stream:

```
Identity Context:
  1. User model + auth endpoints (django-expert)
  2. Organization model + RBAC (django-expert)
  3. Login/signup UI (react-expert)

Core Product Context:
  4. Document models + upload API (django-expert or fastapi-expert)
  5. LangChain analysis pipeline (langchain-expert)
  6. Document viewer + editor UI (react-expert)

Subscription Context:
  7. Plan + billing models (django-expert)
  8. Stripe integration (django-expert or fastapi-expert)
  9. Billing dashboard UI (react-expert)
```

### When to Use
- Complex business domains
- Microservices architecture
- Teams organized by domain expertise
- Long-lived products that evolve per domain

---

## Vertical Slicing

Each slice delivers a complete user-facing feature across all layers.

### Example: Project Management Tool

```
Slice 1: Create Project
  Backend: Project model, POST /api/projects/ (django-expert)
  Frontend: "New Project" form + modal (react-expert)
  Test: E2E test for project creation

Slice 2: Task Board
  Backend: Task model, CRUD API, status transitions (django-expert)
  Frontend: Kanban board component, drag-and-drop (react-expert)
  Test: E2E test for task management

Slice 3: AI Task Summary
  Backend: Summary generation endpoint (fastapi-expert)
  AI: LangChain summary chain (langchain-expert)
  Frontend: Summary display widget (react-expert)
  Test: Integration test with mocked LLM

Slice 4: Team Collaboration
  Backend: Comments model, WebSocket notifications (django-expert)
  Frontend: Comment thread, real-time updates (react-expert)
  Test: WebSocket integration test
```

### When to Use
- Agile/Scrum teams wanting shippable increments
- Stakeholders needing frequent demos
- Risk reduction through early end-to-end validation

---

## Horizontal Slicing

Decompose by technical layer — complete one layer before moving to the next.

### Example: API-First Development

```
Layer 1: Data Foundation
  1. Design all entity models (django-expert)
  2. Create database migrations (django-expert)
  3. Set up indexes and constraints (django-expert)
  4. Seed development data (django-expert)

Layer 2: API Layer
  5. Implement all CRUD endpoints (django-expert)
  6. Add authentication middleware (django-expert)
  7. Configure pagination and filtering (django-expert)
  8. Generate OpenAPI documentation (django-expert)

Layer 3: Business Logic
  9. Implement validation rules (django-expert or python-expert)
  10. Add business workflow services (python-expert)
  11. Integrate external APIs (python-expert)

Layer 4: Frontend
  12. Build component library (react-expert)
  13. Implement page layouts (react-expert)
  14. Connect API integration (react-expert)
  15. Add state management (react-expert)
```

### When to Use
- API-first design where backend must be complete before frontend
- Database-heavy projects (migration-first approach)
- Infrastructure or platform work

---

## Hybrid Approach (Recommended)

Combine strategies based on project needs:

```
Phase 1: Foundation (Horizontal)
  - All models and migrations (django-expert)
  - Project scaffolding (react-expert)
  - Shared type definitions (python-expert)

Phase 2: Core Features (Vertical Slices)
  - Slice A: User auth end-to-end
  - Slice B: Main feature end-to-end
  - Slice C: Secondary feature end-to-end

Phase 3: Enhancement (Domain-based)
  - AI domain: RAG pipeline + integration
  - Analytics domain: Dashboards + export
  - Admin domain: Admin panel customization
```

This gives a stable foundation, quick feature delivery, and clean domain separation for enhancements.
