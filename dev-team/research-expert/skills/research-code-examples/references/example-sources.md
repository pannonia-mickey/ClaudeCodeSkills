# Example Sources Reference

Curated guide to finding working code examples by ecosystem, with specific URLs, search patterns, and documentation site conventions.

---

## Ecosystem Source Directories

### JavaScript / Node.js / npm

**Step 1 — Package registry page:**
```
https://npmjs.com/package/{package-name}
```
Contains: README with installation + basic usage, weekly downloads, version history, repository link.

**Step 2 — GitHub repository examples:**
```
https://github.com/{owner}/{repo}/tree/main/examples
https://github.com/{owner}/{repo}/tree/main/example
https://github.com/{owner}/{repo}/tree/main/demo
```
Most mature packages include a runnable `examples/` directory.

**Step 3 — Repository test files:**
```
https://github.com/{owner}/{repo}/tree/main/test
https://github.com/{owner}/{repo}/tree/main/__tests__
https://github.com/{owner}/{repo}/tree/main/spec
```
Test files are guaranteed to work — they run in CI. Search for the specific method or feature.

**Step 4 — CodeSandbox / StackBlitz templates:**
```
Search: "codesandbox {library} starter" OR "stackblitz {library} template"
```
Interactive environments with working code.

**Major framework example repos:**
| Framework | Official Examples Repo |
|-----------|----------------------|
| Next.js | github.com/vercel/next.js/tree/canary/examples |
| Express | github.com/expressjs/express/tree/master/examples |
| NestJS | github.com/nestjs/nest/tree/master/sample |
| Fastify | github.com/fastify/fastify/tree/main/examples |

---

### Python / PyPI

**Step 1 — PyPI project page:**
```
https://pypi.org/project/{package-name}
```
Contains: description (usually README), homepage link, documentation link.

**Step 2 — ReadTheDocs (most Python packages):**
```
https://{package-name}.readthedocs.io/en/latest/
https://{package-name}.readthedocs.io/en/stable/
```
Quickstart and API reference with examples.

**Step 3 — GitHub examples:**
```
https://github.com/{owner}/{repo}/tree/main/examples
https://github.com/{owner}/{repo}/tree/main/docs/examples
```

**Step 4 — Jupyter notebook examples:**
```
Search: "{library} tutorial site:github.com filetype:ipynb"
```
Many scientific Python packages publish example notebooks.

**Key framework documentation patterns:**
| Framework | Docs URL Pattern |
|-----------|-----------------|
| FastAPI | fastapi.tiangolo.com/tutorial/ |
| Django | docs.djangoproject.com/en/{version}/topics/ |
| SQLAlchemy | docs.sqlalchemy.org/en/{version}/ |
| Pydantic | docs.pydantic.dev/latest/ |
| Celery | docs.celeryq.dev/en/stable/ |

---

### Rust / Crates.io

**Step 1 — docs.rs (auto-generated from source docs):**
```
https://docs.rs/{crate-name}/latest/{crate-name}/
```
Contains: all public types/functions with inline code examples from doc comments.

**Step 2 — Crates.io page:**
```
https://crates.io/crates/{crate-name}
```
Links to repository, documentation, and README.

**Step 3 — The Rust Book / Cookbook:**
```
https://doc.rust-lang.org/book/
https://rust-lang-nursery.github.io/rust-cookbook/
```
Idiomatic patterns for common tasks.

**Cargo test patterns:**
```
https://github.com/{owner}/{repo}/blob/main/tests/
```
Integration tests often show real usage patterns.

---

### Go / pkg.go.dev

**Step 1 — pkg.go.dev (official package registry + docs):**
```
https://pkg.go.dev/{module-path}
```
Contains: README, all exported types with examples, source links.

**Step 2 — Example tests in source:**
```
https://github.com/{owner}/{repo}/blob/main/example_test.go
```
Go convention: `Example` functions in `*_test.go` files are runnable documentation.

---

### REST APIs / HTTP APIs

**Step 1 — Official API reference:**
```
docs.{provider}.com/api/
{provider}.com/developers/
```

**Step 2 — Official Postman collections:**
```
Search: "{api-name} postman collection site:postman.com"
```
Many APIs publish ready-to-import Postman collections.

**Step 3 — Official SDKs:**
```
github.com/{provider}/{language}-sdk
```
SDK examples/ directory shows real usage.

**Step 4 — OpenAPI / Swagger specs:**
```
Search: "{api-name} openapi.yaml site:github.com"
{api-url}/openapi.json
```
Use Swagger UI locally to explore and generate example requests.

---

## GitHub Code Search Patterns

GitHub code search finds real-world usage in production codebases.

### Search URL

```
https://github.com/search?type=code&q={query}
```

### Effective Query Patterns

```bash
# Find usage of a specific function
"functionName(" language:javascript
"ClassName.method(" language:python

# Find configuration patterns
"new Redis({" language:typescript
"DATABASES = {" language:python filename:settings.py

# Find import patterns
"from {library} import" language:python
"require('{library}')" language:javascript

# Find files that set up a specific feature
"@RateLimit" language:typescript filename:*.controller.ts

# Limit to high-quality repos (stars filter requires GitHub Advanced Search)
# Alternative: add "stars:>100" to regular search
```

### Filtering Search Results for Quality

After getting results:

1. **Filter by language** — use the language sidebar
2. **Look at repo context** — click through to the file's repository; check stars and last commit date
3. **Prefer test files** — `filename:*.test.ts`, `filename:*_test.py` — guaranteed to be working code
4. **Prefer config files for setup patterns** — `filename:docker-compose.yml`, `filename:*.config.js`

---

## Stack Overflow Search Operators

```bash
# Basic
[{tag}] {question keywords}

# Tagged searches (most reliable)
[express] [rate-limiting]
[python] [asyncio] gather exceptions

# Sort by votes
https://stackoverflow.com/search?q={query}&tab=votes

# Date filter
https://stackoverflow.com/search?q={query}&fromdate={unix-timestamp}

# For known good answers
is:answer score:20 {keywords} [{tag}]
```

### Evaluating SO Answers

Quality signals in order of importance:
1. Accepted (green checkmark) — asker confirmed it worked
2. Score ≥ 20 — community endorsement
3. No comments saying "this is outdated" or "doesn't work in v{current}"
4. Code compiles/runs mentally without obvious errors
5. Answerer has 1,000+ rep in the relevant tag

---

## Documentation Site Navigation Patterns

### Finding the Right Version's Docs

Most documentation sites follow one of these patterns:

```
# Versioned path
docs.example.com/v2/       ← major version
docs.example.com/2.5/      ← minor version
docs.example.com/en/4.2/   ← language + version (Django, Sphinx)

# Version selector
docs.example.com/stable    ← latest stable
docs.example.com/latest    ← latest (may be pre-release)

# GitHub-based docs (check version in URL or branch)
github.com/{owner}/{repo}/blob/v2.0/docs/
```

### Finding API Reference vs Guide

| URL Pattern | Content Type |
|-------------|-------------|
| `/api/`, `/reference/`, `/api-reference/` | API reference (function signatures) |
| `/guide/`, `/tutorial/`, `/getting-started/` | Tutorial with examples |
| `/cookbook/`, `/recipes/`, `/examples/` | Concrete task-focused examples |
| `/changelog/`, `/releases/`, `/migration/` | Version changes |

### Search Within Docs

Most documentation sites support:
```
site:{docs-url} {feature-name}
```

Example:
```
site:docs.djangoproject.com/en/4.2 class-based views permissions
```
