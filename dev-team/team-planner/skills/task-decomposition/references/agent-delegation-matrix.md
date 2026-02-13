# Agent Delegation Matrix

## Complete Capability Matrix

### aspnet-mvc-expert
| Capability | Examples |
|-----------|----------|
| Architecture | Clean Architecture, Vertical Slice, CQRS, MediatR, DI |
| Controllers | Routing, model binding, action filters, view components |
| Security | Identity, JWT/Cookie auth, policy-based authorization, CSRF |
| EF Core | DbContext, Fluent API, migrations, query optimization |
| Testing | xUnit, Moq/NSubstitute, WebApplicationFactory, integration tests |
| Razor | Razor Pages, Tag Helpers, view components, templates |
| Minimal APIs | Route handlers, endpoint filters, Results pattern |

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

### csharp-expert
| Capability | Examples |
|-----------|----------|
| C# Language | Records, pattern matching, LINQ, generics |
| ASP.NET Core | Controllers, minimal APIs, middleware, DI |
| EF Core | DbContext, Fluent API, migrations, queries |
| Testing | xUnit, NUnit, Moq, WebApplicationFactory |
| Architecture | Clean Architecture, CQRS, MediatR, DDD |
| Blazor | Server/WASM, components, state management |
| Performance | Span, async, caching, compiled queries |

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

### Rule 4: ASP.NET MVC vs C# General

```
Task: "Review my ASP.NET Core controller and fix EF Core N+1"
  ✅ aspnet-mvc-expert (ASP.NET-specific patterns, EF Core in MVC context)
  ❌ csharp-expert (knows C# but not MVC-specific conventions)

Task: "Implement a C# library with generics and LINQ"
  ❌ aspnet-mvc-expert (overkill, not MVC-specific)
  ✅ csharp-expert (general C# language expertise)
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

### Rule 7: Architecture and Planning to team-planner

```
Task: "Decide between Django and FastAPI for this project"
  ✅ team-planner (architectural decision)
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

1. Database modeling agent produces:
   - Entity relationship diagram / model definitions
   - Field types and constraints
   - Index definitions
   - Migration files

2. API agent consumes:
   - Model imports and querysets
   - Serializer/schema field mapping
   - Relationship traversal patterns

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
