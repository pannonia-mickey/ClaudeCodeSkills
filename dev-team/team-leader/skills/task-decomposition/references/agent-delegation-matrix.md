# Agent Delegation Matrix

## Complete Capability Matrix

### angular-expert
| Capability | Examples |
|-----------|----------|
| Components | Standalone components, signals, control flow (@if, @for), lifecycle |
| Routing | Lazy loading, guards, resolvers, nested routes |
| State | NgRx (signals store), component store, RxJS patterns |
| Forms | Reactive forms, typed forms, validation, dynamic forms |
| Testing | Jest, Cypress, Testing Library, component harness |
| Performance | Change detection (OnPush, signals), lazy loading, SSR |

### cloud-expert
| Capability | Examples |
|-----------|----------|
| AWS Services | Lambda, ECS, DynamoDB, S3, CloudFront, SQS/SNS |
| Infrastructure | CDK, multi-region DR, VPC design, IAM, WAF |
| Serverless | Event-driven patterns, Step Functions, cold start optimization |
| Architecture | Well-Architected, circuit breaker, bulkhead, cost optimization |
| Security | IAM least-privilege, KMS encryption, VPC endpoints, GuardDuty |

### cpp-expert
| Capability | Examples |
|-----------|----------|
| Modern C++ | Concepts, ranges, modules, constexpr/consteval, std::expected |
| Memory Safety | RAII, smart pointers, ownership semantics, rule of zero/five |
| Concurrency | std::jthread, atomics, mutexes, coroutines, memory model |
| Performance | Cache-friendly layout, SIMD, move semantics, benchmarking |
| Templates | Variadic templates, concepts, SFINAE, type traits |
| Build Systems | Modern CMake, Conan 2.x, vcpkg, sanitizers, clang-tidy |
| Security | Memory hardening, stack protectors, ASLR, fuzzing, CFI |
| Design Patterns | CRTP, type erasure, PIMPL, policy-based design |

### django-expert
| Capability | Examples |
|-----------|----------|
| Django Models | Model design, fields, relationships, managers, querysets |
| Django Views | CBVs, FBVs, async views, middleware |
| DRF APIs | Serializers, viewsets, routers, permissions, filtering |
| Migrations | Schema changes, data migrations, squashing |
| Admin | Admin customization, actions, inlines |
| Signals | post_save, pre_delete, custom signals |
| Security | CSRF, authentication, permissions, content security |
| Templates | Django templates, template tags, context processors |
| Django Testing | pytest-django, factory_boy, TestCase, API testing |

### react-expert
| Capability | Examples |
|-----------|----------|
| Components | Functional components, hooks, JSX patterns |
| State | useState, useReducer, Context, Redux, Zustand |
| Performance | React.memo, useMemo, useCallback, code splitting |
| Next.js | SSR, SSG, App Router, Server Components |
| TypeScript | Props interfaces, generics, strict typing |
| Testing | React Testing Library, Jest, Vitest, MSW |
| Styling | CSS Modules, Tailwind, styled-components |
| Architecture | Atomic design, compound components, feature folders |

### fastapi-expert
| Capability | Examples |
|-----------|----------|
| Endpoints | Route handlers, path/query params, request bodies |
| Dependencies | Depends(), dependency injection, scoped deps |
| Pydantic | Model design, validators, serialization, v2 features |
| Async | Async endpoints, background tasks, WebSockets |
| Middleware | Request/response middleware, CORS, error handlers |
| Database | SQLAlchemy 2.0 async, Alembic, sessions |
| OpenAPI | Schema customization, docs, code generation |
| Testing | TestClient, httpx, dependency overrides |

### python-expert
| Capability | Examples |
|-----------|----------|
| Core Language | Type hints, dataclasses, protocols, decorators |
| Design Patterns | Factory, Strategy, Observer, Builder, etc. |
| Async | asyncio, aiohttp, async generators |
| Testing | pytest, unittest.mock, Hypothesis, coverage |
| Packaging | pyproject.toml, Poetry, setuptools, src layout |
| Performance | Profiling, optimization, caching |
| Scripting | CLI tools, automation, data processing |

### database-expert
| Capability | Examples |
|-----------|----------|
| Schema Design | Normalization, denormalization, indexing strategies, partitioning |
| SQL | PostgreSQL, MySQL, window functions, CTEs, query optimization |
| NoSQL | MongoDB, DynamoDB, Redis, data modeling patterns |
| ORMs | Prisma, Drizzle, TypeORM, SQLAlchemy, EF Core integration |
| Migrations | Schema versioning, zero-downtime migrations, rollback strategies |
| Performance | EXPLAIN analysis, index tuning, connection pooling, caching |

### devops-expert
| Capability | Examples |
|-----------|----------|
| CI/CD | GitHub Actions, GitLab CI, deployment strategies (canary, blue-green) |
| IaC | Terraform, Pulumi, modules, state management, drift detection |
| Monitoring | Prometheus, Grafana, OpenTelemetry, SLI/SLO, alerting |
| Incidents | Runbooks, postmortems, on-call, chaos engineering |
| Security | Supply chain (SBOM, Sigstore), Vault, OPA, compliance automation |

### dotnet-expert
| Capability | Examples |
|-----------|----------|
| C# Language | Records, pattern matching, LINQ, generics |
| ASP.NET Core | Controllers, Minimal APIs, middleware, DI |
| EF Core | DbContext, Fluent API, migrations, queries |
| Testing | xUnit, NUnit, Moq, WebApplicationFactory |
| Architecture | Clean Architecture, CQRS, MediatR, DDD |
| Blazor | Server/WASM, components, state management |
| Performance | Span, async, caching, compiled queries |

### go-expert
| Capability | Examples |
|-----------|----------|
| Language | Interfaces, generics, error handling, project structure |
| Concurrency | Goroutines, channels, errgroup, worker pools, select |
| API Design | net/http 1.22+, Gin, middleware, gRPC, OpenAPI |
| Testing | Table-driven tests, testify, gomock, benchmarks, fuzz testing |
| Security | Input validation, crypto, JWT, secure HTTP, govulncheck |

### langchain-expert
| Capability | Examples |
|-----------|----------|
| LCEL | Pipe chains, RunnablePassthrough, RunnableParallel |
| Chains | Sequential, branching, fallback chains |
| Agents | ReAct, Plan-and-Execute, custom tools |
| RAG | Document loaders, splitters, embeddings, retrievers |
| Vector Stores | Chroma, FAISS, Pinecone, Weaviate |
| Memory | Buffer, summary, conversation history |
| Evaluation | LangSmith, custom evaluators, testing |

### nextjs-expert
| Capability | Examples |
|-----------|----------|
| App Router | File conventions, layouts, loading/error states, parallel routes |
| Data Fetching | Server Components, generateStaticParams, streaming, Server Actions |
| Caching | Request memoization, data cache, full route cache, revalidation |
| Performance | next/image, next/font, bundle analysis, PPR, rendering strategies |
| Security | CSP with nonce, NextAuth.js v5, server-only guards, env safety |

### nodejs-expert
| Capability | Examples |
|-----------|----------|
| Frameworks | Express, NestJS, Fastify, middleware patterns |
| Runtime | Event loop, streams, worker threads, child processes |
| APIs | REST design, GraphQL, WebSocket, rate limiting |
| Architecture | Dependency injection, CQRS, event-driven, microservices |
| Security | Helmet, input validation (Zod), JWT, OWASP patterns |

### docker-expert
| Capability | Examples |
|-----------|----------|
| Dockerfiles | Multi-stage builds, BuildKit, cache mounts, secret mounts |
| Image Optimization | Base image selection (distroless, slim, alpine), .dockerignore |
| Compose | compose.yaml, healthchecks, profiles, volumes, networks |
| Security | Non-root users, read-only fs, seccomp, image scanning (Trivy) |
| CI/CD | Buildx multi-platform, registry auth, GitHub Actions, layer caching |
| Networking | Custom networks, DNS, port mapping, service discovery |

### seo-expert
| Capability | Examples |
|-----------|----------|
| Technical SEO | robots.txt, XML sitemaps, canonical URLs, crawlability, redirects |
| On-Page | Meta titles/descriptions, Open Graph, heading hierarchy, alt text |
| Structured Data | JSON-LD, Schema.org (Product, Article, FAQ, HowTo, BreadcrumbList) |
| Core Web Vitals | LCP, INP, CLS diagnosis and optimization |
| Framework SEO | SSR/SSG/ISR implications, Next.js/Nuxt/Remix/Astro SEO |
| Rich Results | Google Rich Results requirements, testing, validation |

### rust-expert
| Capability | Examples |
|-----------|----------|
| Ownership | Borrowing, lifetimes, lifetime elision, Pin/Unpin |
| Traits/Generics | Trait bounds, impl Trait, associated types, GATs |
| Concurrency | async/await, Tokio, channels, Rayon, Send/Sync |
| Testing | Unit tests, proptest, cargo-fuzz, wiremock, insta snapshots |
| Security | Unsafe guidelines, FFI, crypto (RustCrypto), cargo-audit |

### testing-expert
| Capability | Examples |
|-----------|----------|
| E2E Testing | Playwright, cross-browser, visual regression, accessibility |
| Unit/Integration | Vitest, Jest, mocking strategies, test doubles, fixtures |
| Performance | k6 load testing, Artillery, benchmarks, Lighthouse CI |
| Strategy | Risk-based coverage, mutation testing, contract testing (Pact) |
| Security | SAST (Semgrep), DAST (ZAP), dependency scanning, penetration testing |

### typescript-expert
| Capability | Examples |
|-----------|----------|
| Type System | Generics, conditional types, mapped types, template literals |
| Patterns | Branded types, discriminated unions, type guards, builder pattern |
| Tooling | tsconfig, project references, declaration files, ESLint |
| Migration | JS-to-TS strategies, strict mode adoption, any elimination |
| Runtime | Zod validation, type-safe APIs, ts-pattern matching |

### uxui-designer
| Capability | Examples |
|-----------|----------|
| Design Systems | Design tokens, naming conventions, component APIs, documentation |
| Accessibility | WCAG 2.2 AA, ARIA roles, keyboard navigation, screen readers |
| Visual Design | Typography, color theory, visual hierarchy, spacing (4px grid) |
| Responsive Design | Mobile-first, container queries, clamp(), breakpoint strategies |
| Interaction Design | Micro-interactions, state transitions, animation, loading patterns |
| Component Specs | All states (hover, focus, active, disabled), responsive behavior |
| Form Design | Inline validation, progressive disclosure, error recovery |

### vue-expert
| Capability | Examples |
|-----------|----------|
| Composition API | script setup, ref/reactive, computed, watch, composables |
| Components | Slots, v-model, dynamic components, Teleport, Transitions |
| Router & State | Vue Router 4, Pinia stores, navigation guards, persistence |
| Nuxt 3 | Server routes, middleware, useFetch, hybrid rendering, SEO |
| Testing | Vitest, Vue Test Utils, Playwright, Pinia testing |

## Overlap Resolution Rules

### Rule 1: Framework-Specific Beats General

```
Task: "Write tests for Django views"
  ✅ django-expert (Django-specific test patterns)
  ❌ python-expert (general pytest knowledge)

Task: "Write tests for a pure Python utility"
  ❌ django-expert (overkill, not Django-specific)
  ✅ python-expert (general testing is their domain)
```

### Rule 2: Integration Context Wins

```
Task: "Set up SQLAlchemy with FastAPI"
  ✅ fastapi-expert (knows FastAPI + SQLAlchemy integration)
  ❌ python-expert (knows SQLAlchemy but not FastAPI patterns)

Task: "Set up Pydantic models for LangChain output parsing"
  ✅ langchain-expert (knows LangChain output parser patterns)
  ❌ fastapi-expert (knows Pydantic but not LangChain context)
```

### Rule 3: Primary Technology Determines Agent

```
Task: "Create React components that call a Django API"
  Frontend work → react-expert
  Backend work → django-expert
  Split into two tasks, one per agent
```

### Rule 4: .NET — dotnet-expert Covers All

```
Task: "Review my ASP.NET Core controller and fix EF Core N+1"
  ✅ dotnet-expert (covers ASP.NET Core, EF Core, and C# language)

Task: "Implement a C# library with generics and LINQ"
  ✅ dotnet-expert (covers all C#/.NET work)
```

### Rule 5: SEO and UX/UI Delegation

```
Task: "Add JSON-LD structured data to product pages"
  ✅ seo-expert (structured data is core SEO expertise)
  ❌ react-expert (can implement markup but lacks SEO-specific knowledge)

Task: "Design an accessible data table with keyboard navigation"
  ✅ uxui-designer (accessibility and component design expertise)
  ❌ react-expert (can implement but lacks design specification depth)

Task: "Improve Core Web Vitals for the React frontend"
  ✅ seo-expert (Core Web Vitals diagnosis) + react-expert (React-specific fixes)
  Split into diagnostic task (seo-expert) and implementation task (react-expert)
```

### Rule 6: C++ vs Other Languages

```
Task: "Build a high-performance C++ processing library"
  ✅ cpp-expert (C++ language and performance expertise)
  ❌ python-expert (different language domain)

Task: "Write Python bindings for a C++ library using pybind11"
  C++ side → cpp-expert
  Python side → python-expert
  Split into two tasks, one per agent
```

### Rule 7: Next.js vs React

```
Task: "Build a React component with hooks and state"
  ✅ react-expert (pure React, no framework)
  ❌ nextjs-expert (overkill for non-Next.js work)

Task: "Set up Server Components with data fetching in Next.js"
  ✅ nextjs-expert (Next.js-specific RSC patterns)
  ❌ react-expert (knows React but not Next.js specifics)
```

### Rule 8: Node.js vs TypeScript

```
Task: "Build an Express API with middleware"
  ✅ nodejs-expert (runtime and framework expertise)
  ❌ typescript-expert (knows TS but not Node.js patterns)

Task: "Create a shared TypeScript type library"
  ✅ typescript-expert (pure type system work)
  ❌ nodejs-expert (knows TS but not deep type patterns)
```

### Rule 9: Cloud vs DevOps

```
Task: "Design a multi-region DR architecture on AWS"
  ✅ cloud-expert (cloud architecture and AWS services)
  ❌ devops-expert (knows infra but not cloud-specific patterns)

Task: "Set up Terraform modules for AWS resources"
  ✅ devops-expert (IaC expertise) + cloud-expert (AWS resource design)
  Split into architecture (cloud-expert) and IaC implementation (devops-expert)
```

### Rule 10: Testing vs Framework Experts

```
Task: "Write unit tests for a React component"
  ✅ react-expert (framework-specific test patterns)
  ❌ testing-expert (general strategy, not framework-specific)

Task: "Design an E2E test strategy and set up Playwright"
  ✅ testing-expert (cross-cutting test expertise)
  ❌ react-expert (knows testing but not strategy depth)
```

### Rule 11: Database Expert vs ORM Experts

```
Task: "Optimize slow PostgreSQL queries with EXPLAIN"
  ✅ database-expert (database-specific query optimization)
  ❌ django-expert (knows ORM but not raw SQL optimization)

Task: "Design Django models with proper relationships"
  ✅ django-expert (Django ORM-specific patterns)
  ❌ database-expert (knows schema but not Django ORM conventions)
```

### Rule 12: Architecture and Planning to team-leader

```
Task: "Decide between Django and FastAPI for this project"
  ✅ team-leader (architectural decision)
  ❌ django-expert or fastapi-expert (biased toward their framework)
```

## Handoff Protocols

### Backend → Frontend Handoff

1. Backend agent produces:
   - API endpoint documentation (OpenAPI or manual)
   - Request/response examples
   - Authentication requirements
   - Error response format

2. Frontend agent consumes:
   - API endpoint URLs and methods
   - TypeScript interfaces from response schemas
   - Auth token handling requirements

### AI Pipeline → API Handoff

1. LangChain agent produces:
   - Chain/agent input/output schemas
   - Streaming behavior specification
   - Error types and retry behavior
   - Token/cost estimates

2. API agent (Django/FastAPI) consumes:
   - Input validation requirements
   - Response streaming setup
   - Error handling and user-facing messages

### SEO → Frontend Handoff

1. SEO agent produces:
   - Meta tag requirements (titles, descriptions, Open Graph, Twitter Cards)
   - Structured data schemas (JSON-LD templates)
   - Core Web Vitals findings and required fixes
   - Heading hierarchy requirements
   - Image optimization requirements (alt text, srcset, lazy loading)

2. Frontend agent (React) consumes:
   - Meta tag implementation in page components
   - JSON-LD script injection
   - Performance fixes (code splitting, image optimization, font loading)
   - Semantic HTML structure changes

### UX/UI → Frontend Handoff

1. UX/UI Designer produces:
   - Design tokens (colors, typography, spacing, breakpoints)
   - Component specifications with all states
   - Accessibility requirements (ARIA, keyboard, screen reader)
   - Responsive behavior specifications
   - Interaction and animation specifications

2. Frontend agent (React) consumes:
   - Token values as CSS custom properties or theme config
   - Component implementation with all specified states
   - ARIA attributes and keyboard handlers
   - Responsive breakpoints and layout changes

### Docker → Backend Handoff

1. Docker agent produces:
   - Dockerfile with correct runtime, ports, and entrypoint
   - Compose service definitions with healthchecks
   - Environment variable contracts (required env vars)
   - Volume mount points for persistent data

2. Backend agent (Django/FastAPI/ASP.NET) consumes:
   - Runtime configuration matching container environment
   - Health check endpoint implementation
   - Environment variable reading (settings/config)
   - Static file and media volume paths

### Database → API Handoff

1. database-expert produces:
   - Entity relationship diagram / model definitions
   - Field types and constraints
   - Index definitions
   - Migration files

2. API agent (Django/FastAPI/Node.js/Go) consumes:
   - Model imports and querysets
   - Serializer/schema field mapping
   - Relationship traversal patterns

### DevOps → All Teams Handoff

1. devops-expert produces:
   - CI/CD pipeline configuration
   - Deployment strategy (canary, blue-green)
   - Monitoring dashboards and alerts
   - Environment configuration

2. All agents consume:
   - Build and test commands for their stack
   - Environment variable contracts
   - Deployment triggers and hooks

### Cloud → DevOps Handoff

1. cloud-expert produces:
   - Architecture diagrams and service selections
   - IAM policies and security groups
   - Cost estimates and optimization recommendations

2. devops-expert consumes:
   - Infrastructure requirements for IaC implementation
   - Monitoring endpoints and health check specifications
   - Deployment targets and scaling policies

### Testing → All Teams Handoff

1. testing-expert produces:
   - Test strategy and coverage targets
   - E2E test scenarios and acceptance criteria
   - Performance budgets and load test configurations

2. Framework agents consume:
   - Unit test requirements for their components
   - Integration test contracts
   - Performance targets to meet

## Integration Checkpoint Template

```markdown
## Checkpoint: {Name}

**After Phase**: {phase number}
**Agents Involved**: {list}

### Verify:
- [ ] API contract matches frontend expectations
- [ ] Shared types are consistent across services
- [ ] Authentication flow works end-to-end
- [ ] Error handling is consistent
- [ ] Data flows correctly between components

### Artifacts to Review:
- [ ] OpenAPI schema
- [ ] TypeScript interface definitions
- [ ] Database migration files
- [ ] Integration test results
```
