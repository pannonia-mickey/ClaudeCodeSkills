---
name: Codebase Mastery
description: This skill should be used when the user asks about "codebase analysis", "codebase overview", "project structure", "tech stack detection", "codebase reconnaissance", "entry points", "project type", or "codebase exploration". It covers foundational codebase analysis techniques, systematic exploration methodology, and report generation.
---

# Codebase Analysis Fundamentals

## Systematic Exploration Methodology

Approach every codebase analysis in five phases. Execute phases in order; each builds on the previous.

### Phase 1: Initial Reconnaissance

Identify the project's identity before diving into code.

```bash
# Directory structure overview (top 3 levels)
find . -maxdepth 3 -type d | head -60

# Identify languages by file extension
find . -type f -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.go' \
  -o -name '*.rs' -o -name '*.java' -o -name '*.cs' -o -name '*.cpp' \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn

# Locate config/manifest files
find . -maxdepth 2 -name 'package.json' -o -name 'pyproject.toml' \
  -o -name 'Cargo.toml' -o -name 'go.mod' -o -name '*.csproj' \
  -o -name 'pom.xml' -o -name 'Makefile' -o -name 'Dockerfile'
```

### Phase 2: Framework and Tool Detection

Map the technology stack from configuration files.

| Config File | Indicates |
|------------|-----------|
| `package.json` | Node.js/JavaScript ecosystem |
| `tsconfig.json` | TypeScript project |
| `pyproject.toml` / `setup.py` | Python project |
| `Cargo.toml` | Rust project |
| `go.mod` | Go project |
| `*.csproj` / `*.sln` | .NET project |
| `docker-compose.yml` | Multi-service application |
| `.github/workflows/` | GitHub Actions CI/CD |
| `nx.json` / `turbo.json` | Monorepo tooling |

```bash
# Check for framework indicators
grep -rl "from django" --include="*.py" -l | head -5    # Django
grep -rl "from fastapi" --include="*.py" -l | head -5   # FastAPI
grep -rl "'react'" package.json 2>/dev/null              # React
grep -rl "'next'" package.json 2>/dev/null               # Next.js
grep -rl "gin-gonic\|gorilla/mux" go.mod 2>/dev/null     # Go web
```

### Phase 3: Project Type Classification

Determine the project's structural archetype.

**Monolith** — Single deployable unit:
- One primary `src/` or `app/` directory
- Single build output
- Shared database layer

**Monorepo** — Multiple packages in one repository:
- `packages/` or `apps/` directories
- Workspace configuration (npm workspaces, Turborepo, Nx, Lerna)
- Multiple `package.json` or build manifests

**Microservices** — Independently deployable services:
- `services/` directory with subdirectories
- Multiple Dockerfiles or `docker-compose.yml`
- API gateway or service mesh configuration

**Library/Package** — Reusable code published to a registry:
- `src/` with `index.ts` or `__init__.py` barrel exports
- Build configuration for distribution (rollup, tsc, setuptools)
- Public API surface defined explicitly

### Phase 4: Entry Point Identification

Locate where execution begins and requests enter the system.

```bash
# Common entry points
find . -maxdepth 3 -name 'main.*' -o -name 'index.*' -o -name 'app.*' \
  -o -name 'server.*' -o -name 'manage.py' -o -name 'Program.cs'

# Route/endpoint definitions
grep -rn "app.get\|app.post\|@app.route\|@router\|@Controller\|HandleFunc" \
  --include="*.py" --include="*.ts" --include="*.js" --include="*.go" | head -20

# CLI entry points
grep -rn "if __name__\|func main\|fn main\|static void Main" \
  --include="*.py" --include="*.go" --include="*.rs" --include="*.cs" | head -10
```

### Phase 5: Quick Health Check

Get an immediate sense of project health.

```bash
# Code volume
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' -o -name '*.go' \
  -o -name '*.rs' \) | wc -l

# Test presence
find . -type f -name '*test*' -o -name '*spec*' | wc -l

# Documentation presence
find . -maxdepth 2 -name 'README*' -o -name 'CONTRIBUTING*' -o -name 'docs'

# Git activity (last 30 days)
git log --oneline --since="30 days ago" | wc -l

# Largest files (potential god classes)
find . -type f \( -name '*.py' -o -name '*.ts' -o -name '*.js' \) \
  -exec wc -l {} + | sort -rn | head -10
```

## Report Generation

After completing the five phases, compile findings into a structured report. Use the templates in `references/report-templates.md` for consistent formatting.

Key report sections:
1. **Executive Summary** — Project type, tech stack, health snapshot
2. **Architecture Overview** — Layers, modules, boundaries
3. **Conventions** — Naming, error handling, patterns in use
4. **Dependencies** — External packages, internal module graph
5. **Quality Indicators** — Test ratio, complexity hotspots, documentation coverage

## References

- [Analysis Methodology](references/analysis-methodology.md) — Complete 5-phase analysis workflow with detailed commands and decision trees for each phase.
- [Report Templates](references/report-templates.md) — Structured output templates for Codebase Overview, Architecture Map, Pattern Catalog, Dependency Health, and Quality Assessment reports.
