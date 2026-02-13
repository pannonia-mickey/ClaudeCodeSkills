# BuildKit Features and Advanced Build Techniques

Comprehensive reference covering BuildKit enablement, cache mounts, secret mounts, heredoc syntax, parallel build stages, multi-platform build arguments, and output export options.

---

## Enabling BuildKit

### Environment Variable

```bash
# Enable BuildKit for a single build
DOCKER_BUILDKIT=1 docker build -t myapp .

# Enable BuildKit globally (Linux)
echo '{"features": {"buildkit": true}}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

Docker Desktop has BuildKit enabled by default since Docker 23.0.

### Parser Directive

Add the syntax directive as the first line of the Dockerfile to use the latest BuildKit frontend:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
# ... rest of Dockerfile
```

This ensures consistent behavior regardless of the Docker daemon version. The `1` tag tracks the latest stable 1.x release.

### BuildKit vs Legacy Builder

| Feature | Legacy Builder | BuildKit |
|---------|---------------|----------|
| Cache mounts | No | Yes |
| Secret mounts | No | Yes |
| SSH forwarding | No | Yes |
| Heredoc syntax | No | Yes |
| Parallel stage builds | No | Yes |
| Build progress output | Basic | Rich |
| Garbage collection | No | Yes |
| COPY --link | No | Yes |
| COPY --chmod | No | Yes |

---

## Cache Mounts

Cache mounts persist data across builds without including it in the final image layer. They dramatically speed up package manager operations.

### pip (Python)

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

### apt (Debian/Ubuntu)

```dockerfile
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt/lists \
    apt-get update && apt-get install -y --no-install-recommends gcc libpq-dev
```

Note: When using apt cache mounts, do not add `rm -rf /var/lib/apt/lists/*` since the cache mount handles cleanup.

### npm (Node.js)

```dockerfile
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

### Go Modules

```dockerfile
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /server ./cmd/server
```

### NuGet (.NET)

```dockerfile
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore
```

### Cache Identity

Cache mounts are identified by their `target` path. Multiple Dockerfiles sharing the same target path share the same cache:

```dockerfile
# Unique cache per project
RUN --mount=type=cache,target=/root/.cache/pip,id=myproject-pip \
    pip install -r requirements.txt
```

### Cache Sharing Modes

```dockerfile
# shared (default): concurrent access allowed
RUN --mount=type=cache,target=/root/.cache/pip,sharing=shared \
    pip install -r requirements.txt

# private: exclusive access, one build at a time
RUN --mount=type=cache,target=/root/.cache/pip,sharing=private \
    pip install -r requirements.txt

# locked: wait for exclusive access
RUN --mount=type=cache,target=/root/.cache/pip,sharing=locked \
    pip install -r requirements.txt
```

---

## Secret Mounts

Secret mounts make sensitive data available during build without persisting it in any image layer.

### Basic Usage

```dockerfile
# Dockerfile
RUN --mount=type=secret,id=mysecret \
    cat /run/secrets/mysecret

# Build command
docker build --secret id=mysecret,src=./mysecret.txt .
```

### Private PyPI Repository

```dockerfile
RUN --mount=type=secret,id=pip_conf,target=/root/.pip/pip.conf \
    pip install -r requirements.txt
```

```bash
# pip.conf with private index credentials
docker build --secret id=pip_conf,src=./pip.conf .
```

### npm Private Registry

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

### Environment Variable from Secret

```dockerfile
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    ./configure --api-key="$API_KEY"
```

### SSH Agent Forwarding

Access private Git repositories during build without copying SSH keys:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends git openssh-client \
    && rm -rf /var/lib/apt/lists/*

# Clone private repo using forwarded SSH agent
RUN --mount=type=ssh \
    git clone git@github.com:org/private-repo.git /app/private-repo
```

```bash
# Build with SSH agent forwarding
docker build --ssh default=$SSH_AUTH_SOCK .
```

---

## Bind Mounts in RUN

Bind mounts provide read-only access to files from the build context or other stages without copying them into a layer:

```dockerfile
# Mount a file from the build context
RUN --mount=type=bind,source=config.json,target=/tmp/config.json \
    ./process-config /tmp/config.json

# Mount from another stage
RUN --mount=type=bind,from=builder,source=/app/dist,target=/tmp/dist \
    cp -r /tmp/dist/* /usr/share/nginx/html/
```

This is useful when you need to read a file during build but do not want it in the final image.

---

## Heredoc Syntax

BuildKit supports heredoc syntax for multi-line RUN commands and inline file creation.

### Multi-Line RUN

```dockerfile
RUN <<EOF
apt-get update
apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev
rm -rf /var/lib/apt/lists/*
EOF
```

### Inline File Creation

```dockerfile
COPY <<EOF /etc/nginx/conf.d/default.conf
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://app:8000;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }

    location /static/ {
        alias /app/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
EOF
```

### Python Script Inline

```dockerfile
COPY <<'EOF' /app/healthcheck.py
import urllib.request
import sys

try:
    response = urllib.request.urlopen("http://localhost:8000/health/")
    sys.exit(0 if response.status == 200 else 1)
except Exception:
    sys.exit(1)
EOF
```

Use `<<'EOF'` (quoted) to prevent variable expansion. Use `<<EOF` (unquoted) when you want shell variable substitution.

---

## Parallel Build Stages

BuildKit automatically builds independent stages in parallel. Design your Dockerfile to maximize parallelism:

```dockerfile
# These three stages build in parallel
FROM node:20-slim AS frontend-builder
WORKDIR /frontend
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

FROM python:3.12-slim AS backend-builder
WORKDIR /backend
COPY backend/requirements.txt .
RUN pip install --no-compile -r requirements.txt
COPY backend/ .
RUN python manage.py collectstatic --noinput

FROM golang:1.22-alpine AS api-gateway
WORKDIR /gateway
COPY gateway/go.mod gateway/go.sum ./
RUN go mod download
COPY gateway/ .
RUN CGO_ENABLED=0 go build -o /gateway-bin ./cmd/gateway

# Final stage: combines all outputs
FROM nginx:alpine
COPY --from=frontend-builder /frontend/dist /usr/share/nginx/html
COPY --from=backend-builder /backend/staticfiles /usr/share/nginx/static
COPY --from=api-gateway /gateway-bin /usr/local/bin/gateway
```

BuildKit detects that `frontend-builder`, `backend-builder`, and `api-gateway` are independent and builds them simultaneously.

---

## Multi-Platform Build Arguments

BuildKit provides automatic build arguments for cross-compilation:

| Variable | Example | Description |
|----------|---------|-------------|
| `TARGETPLATFORM` | `linux/arm64` | Full platform spec |
| `TARGETOS` | `linux` | Target operating system |
| `TARGETARCH` | `arm64` | Target architecture |
| `TARGETVARIANT` | `v8` | Architecture variant |
| `BUILDPLATFORM` | `linux/amd64` | Build machine platform |
| `BUILDOS` | `linux` | Build machine OS |
| `BUILDARCH` | `amd64` | Build machine architecture |

### Conditional Logic Per Platform

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder
ARG TARGETOS TARGETARCH

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -o /server ./cmd/server

FROM scratch
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

### Platform-Specific Package Installation

```dockerfile
ARG TARGETARCH

RUN case ${TARGETARCH} in \
        amd64) ARCH="x86_64" ;; \
        arm64) ARCH="aarch64" ;; \
        *) echo "Unsupported arch: ${TARGETARCH}" && exit 1 ;; \
    esac \
    && curl -fsSL "https://example.com/binary-${ARCH}.tar.gz" | tar xz -C /usr/local/bin/
```

---

## Output and Export

### Local Directory Output

Export build results to a local directory instead of creating an image:

```bash
docker build --output type=local,dest=./out .
```

### Tar Archive Output

```bash
docker build --output type=tar,dest=image.tar .
```

### OCI Image Format

```bash
docker build --output type=oci,dest=image-oci.tar .
```

### Extracting Specific Stage Output

```bash
# Export only the test results
docker build --target=test --output type=local,dest=./test-results .
```

This is useful in CI pipelines to extract build artifacts, test results, or compiled binaries without creating a full container image.

---

## BuildKit Cache Management

### Inspect Cache Usage

```bash
docker builder du
```

### Prune Build Cache

```bash
# Remove all unused cache
docker builder prune

# Remove all cache (including in-use)
docker builder prune --all

# Remove cache older than 24 hours
docker builder prune --filter until=24h
```

### Cache Size Limits

Configure max cache size in the daemon or buildkitd config:

```toml
# /etc/buildkit/buildkitd.toml
[worker.oci]
  maxParallelism = 4
  gcKeepStorage = 10000  # MB
```
