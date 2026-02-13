---
name: Docker Mastery
description: This skill should be used when the user asks about "Dockerfile", "Docker build", "multi-stage build", "BuildKit", "Docker layer", "base image", "ENTRYPOINT", or "docker image optimization". It covers core Docker concepts including Dockerfile authoring, multi-stage build patterns, BuildKit features, layer caching strategies, base image selection, and image size optimization.
---

## Dockerfile Fundamentals

### FROM with Pinned Versions

Always pin base image versions. For production, pin to SHA256 digests for full reproducibility:

```dockerfile
# Development: pin to minor version
FROM python:3.12-slim

# Production: pin to SHA256 digest
FROM python:3.12-slim@sha256:a3e58f8c7a3e6c5b...
```

Use `ARG` before `FROM` to parameterize the base image:

```dockerfile
ARG PYTHON_VERSION=3.12
FROM python:${PYTHON_VERSION}-slim AS base
```

### COPY vs ADD

Always prefer `COPY` over `ADD`. Use `ADD` only when extracting local tar archives:

```dockerfile
# Correct: use COPY for files and directories
COPY requirements.txt .
COPY src/ ./src/

# Only valid ADD use: extracting a local archive
ADD rootfs.tar.gz /

# Never: ADD for remote URLs (use curl/wget in RUN instead)
# ADD https://example.com/file.tar.gz /app/
```

### RUN Instruction Consolidation

Combine related commands in a single `RUN` to reduce layers and clean up in the same layer:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        gcc \
        libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```

### WORKDIR, ENV, ARG, EXPOSE

```dockerfile
# Set working directory (creates if not exists)
WORKDIR /app

# Build-time variables (not persisted in final image)
ARG BUILD_ENV=production

# Runtime environment variables (persisted in image)
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Document the port (does not publish it)
EXPOSE 8000
```

### ENTRYPOINT vs CMD

Use exec form (JSON array) for both. Combine them for default arguments:

```dockerfile
# Exec form (preferred) — receives signals directly
ENTRYPOINT ["gunicorn", "config.wsgi:application"]
CMD ["--bind", "0.0.0.0:8000", "--workers", "4"]

# Running: docker run myapp
#   → gunicorn config.wsgi:application --bind 0.0.0.0:8000 --workers 4

# Override CMD: docker run myapp --bind 0.0.0.0:9000 --workers 2
#   → gunicorn config.wsgi:application --bind 0.0.0.0:9000 --workers 2
```

Never use shell form for ENTRYPOINT — it wraps the process in `/bin/sh -c`, preventing signal propagation:

```dockerfile
# Bad: shell form — PID 1 is /bin/sh, not gunicorn
ENTRYPOINT gunicorn config.wsgi:application

# Good: exec form — PID 1 is gunicorn
ENTRYPOINT ["gunicorn", "config.wsgi:application"]
```

### LABEL for OCI Metadata

```dockerfile
LABEL org.opencontainers.image.title="My App" \
      org.opencontainers.image.description="Production web application" \
      org.opencontainers.image.version="1.2.3" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.licenses="MIT"
```

### Complete Production Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

WORKDIR /app

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile -r requirements.txt

COPY . .
RUN python manage.py collectstatic --noinput

FROM python:3.12-slim

RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

WORKDIR /app

COPY --from=builder --chown=appuser:appuser /usr/local/lib/python3.12/site-packages/ /usr/local/lib/python3.12/site-packages/
COPY --from=builder --chown=appuser:appuser /usr/local/bin/gunicorn /usr/local/bin/gunicorn
COPY --from=builder --chown=appuser:appuser /app /app

USER appuser
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"]

ENTRYPOINT ["gunicorn", "config.wsgi:application"]
CMD ["--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "120"]
```

---

## Multi-Stage Builds

### Named Stages

Name every stage for clarity and targeted builds:

```dockerfile
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-slim AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-slim AS test
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run test

FROM node:20-slim AS production
WORKDIR /app
ENV NODE_ENV=production
RUN groupadd -r appuser && useradd -r -g appuser appuser
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package.json .
USER appuser
EXPOSE 3000
ENTRYPOINT ["node", "dist/server.js"]
```

Build specific stages with `--target`:

```bash
docker build --target=test -t myapp:test .
docker build --target=production -t myapp:latest .
```

### Go Application with Scratch

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

---

## Layer Optimization

### Layer Ordering Principle

Order instructions from least-changing to most-changing. Dependencies change less often than source code:

```dockerfile
# Bad: COPY . . invalidates cache for every source change
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

# Good: dependencies cached separately from source
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Cleaning Up in the Same Layer

Cleanup in a separate `RUN` does not reduce image size — the files still exist in the previous layer:

```dockerfile
# Bad: cleanup in separate layer (files still in layer 1)
RUN apt-get update && apt-get install -y gcc
RUN rm -rf /var/lib/apt/lists/*

# Good: cleanup in same layer
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc \
    && rm -rf /var/lib/apt/lists/*
```

### Alpine Package Cleanup

```dockerfile
RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev
```

The `--no-cache` flag avoids storing the package index, eliminating the need for `rm -rf /var/cache/apk/*`.

---

## Base Image Selection

### Comparison Table

| Base Image | Size | Security | Debugging | Compatibility |
|-----------|------|----------|-----------|---------------|
| `python:3.12` (full) | ~900MB | Low — many packages | Easy — full tooling | High — glibc |
| `python:3.12-slim` | ~150MB | Medium | Good — apt available | High — glibc |
| `python:3.12-alpine` | ~50MB | Medium-High | Limited — no bash/glibc | Low — musl libc |
| `gcr.io/distroless/python3` | ~50MB | High — no shell | Hard — no shell access | Medium |
| `scratch` | 0MB | Highest — nothing | None — empty | Static binaries only |
| `cgr.dev/chainguard/python` | ~40MB | Very High — hardened | Limited | High — glibc |

### Alpine musl vs Debian glibc

Alpine uses musl libc instead of glibc. This causes issues with:
- Python packages with C extensions (numpy, pandas, cryptography)
- Pre-compiled wheels from PyPI (may need to compile from source)
- DNS resolution behavior differences
- Locale/encoding edge cases

**Recommendation**: Use `slim` for Python/Node.js workloads. Use `alpine` for Go or simple shell-based images.

---

## .dockerignore

### Comprehensive Example

```dockerignore
# Version control
.git
.gitignore

# Docker files (don't send to build context)
Dockerfile*
docker-compose*.yml
compose*.yaml
.dockerignore

# Dependencies (installed in container)
node_modules
__pycache__
*.pyc
.venv
venv

# IDE and editor files
.vscode
.idea
*.swp
*.swo

# Documentation
*.md
LICENSE
docs/

# Environment and secrets
.env
.env.*
*.pem
*.key

# Build artifacts
dist/
build/
*.egg-info

# Testing
.coverage
htmlcov/
.pytest_cache
.tox
```

Build context size directly impacts build speed. A large `.git` directory or `node_modules` folder sent to the daemon slows every build. Always verify with:

```bash
docker build --no-cache --progress=plain . 2>&1 | head -5
# Look for "transferring context" size
```

---

## HEALTHCHECK

### Web Application

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["curl", "-f", "http://localhost:8000/health/"] || exit 1
```

For images without curl, use Python or wget:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"]
```

### Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `--interval` | 30s | Time between health checks |
| `--timeout` | 30s | Max time for a single check |
| `--start-period` | 0s | Grace period for container startup |
| `--retries` | 3 | Consecutive failures before unhealthy |

Set `--start-period` generously for applications with slow startup (database migrations, model loading).

---

## References

- **[Dockerfile Patterns and Anti-Patterns](references/dockerfile-patterns.md)** — Production Dockerfile patterns for Python, Node.js, Go, and .NET. Covers ARG/ENV patterns, ENTRYPOINT/CMD combinations, COPY techniques, and common anti-patterns to avoid.
- **[BuildKit Features and Advanced Build Techniques](references/buildkit-features.md)** — BuildKit cache mounts, secret mounts, heredoc syntax, parallel build stages, multi-platform build arguments, and output export options.
