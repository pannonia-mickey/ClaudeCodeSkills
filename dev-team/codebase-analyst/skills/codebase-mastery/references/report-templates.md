# Report Templates

## Codebase Overview Report

```markdown
# Codebase Overview: {Project Name}

## Executive Summary
- **Project Type**: [Monolith / Monorepo / Microservices / Library]
- **Primary Language(s)**: [e.g., TypeScript 85%, Python 15%]
- **Framework(s)**: [e.g., Next.js 14, FastAPI]
- **Total Source Files**: [count]
- **Estimated Lines of Code**: [count]
- **Active Contributors (90d)**: [count]
- **Health Score**: [Good / Fair / Needs Attention]

## Technology Stack
| Layer | Technology | Version | Config File |
|-------|-----------|---------|-------------|
| Frontend | React | 18.2 | package.json |
| Backend | FastAPI | 0.104 | pyproject.toml |
| Database | PostgreSQL | 15 | docker-compose.yml |
| Cache | Redis | 7 | docker-compose.yml |
| CI/CD | GitHub Actions | — | .github/workflows/ |

## Project Structure
{tree diagram or directory listing with annotations}

## Entry Points
| Entry Point | Type | File | Purpose |
|------------|------|------|---------|
| `src/main.ts` | Application | src/main.ts:1 | App bootstrap |
| `POST /api/users` | API | src/routes/users.ts:15 | User creation |

## Key Findings
1. [Finding with file:line reference]
2. [Finding with file:line reference]
3. [Finding with file:line reference]

## Recommendations
- **High Priority**: [actionable recommendation]
- **Medium Priority**: [actionable recommendation]
- **Low Priority**: [actionable recommendation]
```

## Architecture Map Report

```markdown
# Architecture Map: {Project Name}

## Architectural Style
**Primary**: [Layered / Hexagonal / Microservices / Event-Driven / Modular Monolith]
**Secondary Patterns**: [CQRS, Event Sourcing, Repository, etc.]

## Layer Diagram
{ASCII or Mermaid diagram}

## Module Inventory
| Module | Directory | Responsibility | Dependencies |
|--------|-----------|---------------|-------------|
| auth | src/auth/ | Authentication & authorization | database, config |
| users | src/users/ | User management CRUD | auth, database |
| orders | src/orders/ | Order processing | users, payments, database |

## Data Flow
### Request Lifecycle: {Example Flow}
1. `src/middleware/auth.ts:12` — Authentication middleware
2. `src/routes/orders.ts:45` — Route handler
3. `src/services/order.service.ts:23` — Business logic
4. `src/repositories/order.repo.ts:18` — Data access
5. `src/models/order.ts:5` — Database model

## Dependency Graph
{Module dependency diagram}

## Integration Points
| Source | Target | Type | Contract |
|--------|--------|------|----------|
| Frontend | Backend API | REST | OpenAPI spec |
| Backend | Database | SQL | Prisma schema |
| Backend | Redis | TCP | Key patterns |

## Architectural Concerns
- [Concern with evidence from codebase]
```

## Pattern Catalog Report

```markdown
# Pattern Catalog: {Project Name}

## Naming Conventions
| Element | Convention | Example | Source |
|---------|-----------|---------|--------|
| Files | kebab-case | `user-service.ts` | src/services/ |
| Classes | PascalCase | `UserService` | src/services/user-service.ts:5 |
| Functions | camelCase | `createUser` | src/services/user-service.ts:12 |
| Constants | UPPER_SNAKE | `MAX_RETRIES` | src/config/constants.ts:3 |
| Database | snake_case | `user_accounts` | prisma/schema.prisma:15 |

## Error Handling Pattern
**Style**: [Custom error classes / Result type / Error codes]
**Example**: {code snippet from codebase with file:line}

## Logging Pattern
**Library**: [winston / pino / structlog / slog]
**Style**: [Structured JSON / Plain text]
**Example**: {code snippet with file:line}

## Configuration Pattern
**Style**: [Environment variables / Config files / Feature flags]
**Location**: {config file paths}

## Design Patterns in Use
| Pattern | Usage | Location |
|---------|-------|----------|
| Repository | Database access abstraction | src/repositories/ |
| Factory | Object creation | src/factories/ |
| Observer | Event handling | src/events/ |
| Strategy | Pluggable algorithms | src/strategies/ |

## Import Organization
**Standard**: {import ordering convention with example}
```

## Dependency Health Report

```markdown
# Dependency Health Report: {Project Name}

## Summary
- **Total Dependencies**: [count]
- **Production**: [count]
- **Development**: [count]
- **Outdated**: [count] ({percentage}%)
- **Vulnerable**: [count]
- **Deprecated**: [count]

## Vulnerability Inventory
| Package | Severity | CVE | Fix Available | Current | Fixed In |
|---------|----------|-----|---------------|---------|----------|
| lodash | High | CVE-2021-23337 | Yes | 4.17.20 | 4.17.21 |

## Outdated Packages
| Package | Current | Latest | Semver Gap | Risk |
|---------|---------|--------|-----------|------|
| react | 17.0.2 | 18.2.0 | Major | High |
| axios | 0.27.2 | 1.6.0 | Major | Medium |

## Dependency Graph Metrics
- **Direct dependencies**: [count]
- **Transitive dependencies**: [count]
- **Max dependency depth**: [number]
- **Duplicate packages**: [list if any]

## License Compliance
| License | Count | Packages | Status |
|---------|-------|----------|--------|
| MIT | 145 | ... | Approved |
| Apache-2.0 | 23 | ... | Approved |
| GPL-3.0 | 2 | pkg-a, pkg-b | Review Required |

## Recommendations
1. **Critical**: [fix vulnerabilities]
2. **High**: [upgrade majors with breaking changes]
3. **Medium**: [update outdated minors]
```

## Quality Assessment Report

```markdown
# Quality Assessment: {Project Name}

## Health Dashboard
| Metric | Value | Rating |
|--------|-------|--------|
| Test File Ratio | 35% | Fair |
| Avg File Size | 180 LOC | Good |
| Max File Size | 1,200 LOC | Needs Attention |
| Documentation | README + inline | Fair |
| Commit Frequency (30d) | 47 commits | Active |

## Complexity Hotspots
| File | LOC | Functions | Churn (90d) | Risk |
|------|-----|-----------|-------------|------|
| src/services/order.ts | 850 | 32 | 28 changes | High |
| src/utils/helpers.ts | 620 | 45 | 15 changes | High |

## Technical Debt Inventory
| ID | Category | Description | Severity | File |
|----|----------|-------------|----------|------|
| TD-1 | Design | God class in order.ts | High | src/services/order.ts |
| TD-2 | Test | No tests for payment module | High | src/payments/ |
| TD-3 | Code | Duplicated validation logic | Medium | src/validators/ |

## Test Coverage Gaps
| Module | Source Files | Test Files | Gap |
|--------|-------------|------------|-----|
| auth | 5 | 4 | 1 missing |
| payments | 8 | 0 | 8 missing |
| users | 6 | 5 | 1 missing |

## Remediation Roadmap
### Immediate (This Sprint)
- [Action item with file reference]

### Short-Term (This Quarter)
- [Action item with file reference]

### Long-Term (Next Quarter)
- [Action item with file reference]
```

## Quick Analysis Summary

For rapid assessments when a full report is not needed:

```markdown
## Quick Analysis: {Project Name}

**Stack**: {language} + {framework} | **Type**: {monolith/monorepo/etc.}
**Size**: ~{LOC}k LOC across {file count} files | **Tests**: {ratio}%
**Activity**: {commits/30d} commits/month | **Contributors**: {count}

**Architecture**: {one-line description}
**Key Pattern**: {dominant design pattern}
**Top Concern**: {most important finding}
**Recommendation**: {single most impactful action}
```
