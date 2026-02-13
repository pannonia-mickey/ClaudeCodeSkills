# Dockerfile Patterns and Anti-Patterns

Comprehensive reference covering production Dockerfile patterns for multiple languages, ARG/ENV usage, ENTRYPOINT/CMD combinations, COPY techniques, and common anti-patterns.

---

## Production Dockerfile Patterns

### Python / Django

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

WORKDIR /app

# Install system dependencies for building Python packages
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies with cache mount
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile -r requirements.txt

# Copy application and collect static files
COPY . .
RUN python manage.py collectstatic --noinput

# --- Production stage ---
FROM python:3.12-slim

# Install runtime-only system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r django && useradd -r -g django -d /app -s /sbin/nologin django

WORKDIR /app

# Copy installed packages and application
COPY --from=builder /usr/local/lib/python3.12/site-packages/ /usr/local/lib/python3.12/site-packages/
COPY --from=builder /usr/local/bin/ /usr/local/bin/
COPY --from=builder --chown=django:django /app /app

USER django
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"]

ENTRYPOINT ["gunicorn", "config.wsgi:application"]
CMD ["--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "120"]
```

### Node.js

```dockerfile
# syntax=docker/dockerfile:1
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --ignore-scripts

FROM node:20-slim AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
# Remove dev dependencies after build
RUN npm prune --production

FROM node:20-slim AS production
WORKDIR /app
ENV NODE_ENV=production

RUN groupadd -r appuser && useradd -r -g appuser -d /app appuser

COPY --from=builder --chown=appuser:appuser /app/dist ./dist
COPY --from=builder --chown=appuser:appuser /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appuser /app/package.json .

USER appuser
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD ["node", "-e", "require('http').get('http://localhost:3000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"]

ENTRYPOINT ["node", "dist/server.js"]
```

### Go

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS builder
WORKDIR /app

# Download dependencies first (cacheable layer)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Build statically linked binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w -X main.version=${VERSION}" \
    -o /server ./cmd/server

# --- Minimal runtime ---
FROM scratch

# Copy CA certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy timezone data if needed
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Copy the binary
COPY --from=builder /server /server

EXPOSE 8080
ENTRYPOINT ["/server"]
```

### .NET

```dockerfile
# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS builder
WORKDIR /src

# Restore dependencies (cacheable)
COPY *.csproj .
RUN dotnet restore

# Build and publish
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# --- Runtime ---
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app

RUN groupadd -r appuser && useradd -r -g appuser appuser

COPY --from=builder --chown=appuser:appuser /app/publish .

USER appuser
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD ["curl", "-f", "http://localhost:8080/health"] || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

---

## ARG and ENV Patterns

### Build-Time vs Runtime

```dockerfile
# ARG: available only during build, not persisted in image
ARG BUILD_ENV=production
ARG GIT_SHA

# ENV: persisted in the final image, available at runtime
ENV APP_ENV=${BUILD_ENV} \
    GIT_COMMIT=${GIT_SHA}
```

### ARG Before FROM

Use `ARG` before `FROM` to parameterize the base image. Note: ARGs before FROM are not available after FROM unless redeclared:

```dockerfile
ARG PYTHON_VERSION=3.12
FROM python:${PYTHON_VERSION}-slim

# Must redeclare to use after FROM
ARG PYTHON_VERSION
RUN echo "Using Python ${PYTHON_VERSION}"
```

### Default Values and Overrides

```dockerfile
ARG HTTP_PROXY
ARG EXTRA_PACKAGES=""

RUN apt-get update \
    && apt-get install -y --no-install-recommends ${EXTRA_PACKAGES} \
    && rm -rf /var/lib/apt/lists/*
```

Override at build time:

```bash
docker build --build-arg EXTRA_PACKAGES="vim curl" .
```

### Security Warning

ARG values are visible in the image history via `docker history`. Never pass secrets as build arguments:

```bash
# Bad: secret visible in image history
docker build --build-arg DB_PASSWORD=secret .

# Good: use BuildKit secret mounts instead
docker build --secret id=db_password,src=./db_password.txt .
```

---

## ENTRYPOINT and CMD Patterns

### Exec Form vs Shell Form

```dockerfile
# Exec form (preferred): process is PID 1, receives signals directly
ENTRYPOINT ["python", "app.py"]

# Shell form: wraps in /bin/sh -c, process is NOT PID 1
ENTRYPOINT python app.py
# Equivalent to: /bin/sh -c "python app.py"
```

### Combined ENTRYPOINT + CMD

```dockerfile
ENTRYPOINT ["python", "manage.py"]
CMD ["runserver", "0.0.0.0:8000"]

# docker run myapp → python manage.py runserver 0.0.0.0:8000
# docker run myapp migrate → python manage.py migrate
# docker run myapp shell → python manage.py shell
```

### Wrapper Script with exec

Use a wrapper script when you need initialization before the main process:

```bash
#!/bin/sh
# docker-entrypoint.sh

set -e

# Run migrations if requested
if [ "$RUN_MIGRATIONS" = "true" ]; then
    echo "Running migrations..."
    python manage.py migrate --noinput
fi

# Wait for database
if [ -n "$DATABASE_URL" ]; then
    echo "Waiting for database..."
    python -c "
import time, urllib.parse, socket
parsed = urllib.parse.urlparse('$DATABASE_URL')
for i in range(30):
    try:
        socket.create_connection((parsed.hostname, parsed.port or 5432), timeout=1)
        break
    except OSError:
        time.sleep(1)
"
fi

# exec replaces the shell with the main process (becomes PID 1)
exec "$@"
```

```dockerfile
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

### Signal Handling and PID 1

When the main process does not handle SIGTERM (e.g., shell scripts, some Java apps), use `tini` as an init process:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends tini \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["tini", "--"]
CMD ["python", "app.py"]
```

Or use `dumb-init`:

```dockerfile
RUN pip install dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "app.py"]
```

---

## COPY Patterns

### COPY --chown

Set file ownership during copy to avoid a separate `chown` layer:

```dockerfile
# Without --chown: requires extra RUN layer
COPY . /app
RUN chown -R appuser:appuser /app

# With --chown: ownership set in the same layer
COPY --chown=appuser:appuser . /app
```

### COPY --chmod (BuildKit)

Set permissions during copy:

```dockerfile
COPY --chmod=755 docker-entrypoint.sh /usr/local/bin/
```

### COPY --link

Create independent layers that do not depend on previous layers. This improves cache reuse when base image changes:

```dockerfile
# Without --link: layer depends on all previous layers
COPY --from=builder /app/dist /app/dist

# With --link: independent layer, reusable across base image updates
COPY --link --from=builder /app/dist /app/dist
```

### Selective COPY

Copy only what is needed instead of the entire context:

```dockerfile
# Bad: copies everything, including tests, docs, configs
COPY . .

# Good: copy only what the application needs
COPY src/ ./src/
COPY config/ ./config/
COPY pyproject.toml .
```

---

## Anti-Patterns to Avoid

### Running as Root

```dockerfile
# Bad: runs as root by default
FROM python:3.12-slim
COPY . /app
CMD ["python", "app.py"]

# Good: explicit non-root user
FROM python:3.12-slim
RUN groupadd -r app && useradd -r -g app app
COPY --chown=app:app . /app
USER app
CMD ["python", "app.py"]
```

### Using `latest` Tag

```dockerfile
# Bad: non-reproducible, changes without notice
FROM python:latest

# Good: pinned to specific version
FROM python:3.12-slim

# Best: pinned to SHA digest
FROM python:3.12-slim@sha256:abc123...
```

### ADD for Remote URLs

```dockerfile
# Bad: ADD for remote files (no checksum verification, extra layer)
ADD https://example.com/app.tar.gz /app/

# Good: download, verify, extract in one layer
RUN curl -fsSL https://example.com/app.tar.gz -o /tmp/app.tar.gz \
    && echo "expected_sha256  /tmp/app.tar.gz" | sha256sum -c - \
    && tar xzf /tmp/app.tar.gz -C /app/ \
    && rm /tmp/app.tar.gz
```

### Secrets in ENV or ARG

```dockerfile
# Bad: password visible in image layers and docker inspect
ENV DB_PASSWORD=mysecret
ARG API_KEY=sk-12345

# Good: use BuildKit secret mounts
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password | some-setup-command

# Good: pass secrets at runtime
# docker run -e DB_PASSWORD=mysecret myapp
```

### COPY Before Dependencies

```dockerfile
# Bad: any source change invalidates pip install cache
COPY . .
RUN pip install -r requirements.txt

# Good: install deps first, then copy source
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

### Not Cleaning Up in the Same Layer

```dockerfile
# Bad: apt cache persists in a previous layer (adds ~30MB)
RUN apt-get update
RUN apt-get install -y gcc
RUN rm -rf /var/lib/apt/lists/*

# Good: single layer with cleanup
RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc \
    && rm -rf /var/lib/apt/lists/*
```
