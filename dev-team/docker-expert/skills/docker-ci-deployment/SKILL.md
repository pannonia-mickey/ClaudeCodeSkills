---
name: Docker CI Deployment
description: This skill should be used when the user asks about "Docker CI/CD", "GitHub Actions Docker", "buildx", "multi-platform build", "Docker registry", "image tagging", "container deployment", or "Docker debugging". It covers CI/CD pipeline integration for Docker including GitHub Actions workflows, multi-platform builds with buildx, registry management, tagging strategies, layer caching in CI, and container debugging techniques.
---

## GitHub Actions Docker Workflow

### Complete Build-Push Workflow

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      security-events: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha,prefix=
            type=ref,event=branch
            type=ref,event=pr

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

### Key Actions

| Action | Purpose |
|--------|---------|
| `docker/setup-qemu-action` | Enables cross-platform emulation for multi-arch builds |
| `docker/setup-buildx-action` | Configures buildx builder with advanced features |
| `docker/login-action` | Authenticates to container registries (GHCR, Docker Hub, ECR, ACR) |
| `docker/metadata-action` | Generates tags and OCI labels from Git metadata |
| `docker/build-push-action` | Builds and optionally pushes Docker images |

### Registry-Specific Login

```yaml
# Docker Hub
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKERHUB_USERNAME }}
    password: ${{ secrets.DOCKERHUB_TOKEN }}

# AWS ECR
- uses: docker/login-action@v3
  with:
    registry: 123456789.dkr.ecr.us-east-1.amazonaws.com
    username: ${{ secrets.AWS_ACCESS_KEY_ID }}
    password: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

# Azure ACR
- uses: docker/login-action@v3
  with:
    registry: myregistry.azurecr.io
    username: ${{ secrets.ACR_USERNAME }}
    password: ${{ secrets.ACR_PASSWORD }}
```

---

## Multi-Platform Builds with buildx

### Local Multi-Platform Build

```bash
# Create a buildx builder
docker buildx create --name multiplatform --use

# Build for multiple platforms
docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    --tag myapp:latest \
    --push .

# Inspect the builder
docker buildx inspect multiplatform
```

### Cross-Compilation Pattern (Go)

Use BuildKit's automatic platform variables for efficient cross-compilation:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder

ARG TARGETOS TARGETARCH

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .

# Cross-compile on the build platform (fast native compilation)
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -o /server ./cmd/server

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /server /server
ENTRYPOINT ["/server"]
```

The `--platform=$BUILDPLATFORM` ensures the build stage runs natively (no emulation), while `TARGETOS`/`TARGETARCH` direct cross-compilation.

### QEMU vs Native Builders

| Method | Speed | Setup | Use Case |
|--------|-------|-------|----------|
| QEMU emulation | Slow (5-20x) | Easy | CI, occasional builds |
| Native remote builders | Fast | Complex | Production pipelines |
| Cross-compilation | Fast | Medium | Go, Rust, C/C++ |

For production pipelines with frequent multi-platform builds, consider remote native builders:

```bash
# Add a remote ARM64 builder
docker buildx create --name multiplatform \
    --node arm64-builder \
    --platform linux/arm64 \
    ssh://user@arm64-host

docker buildx create --name multiplatform --append \
    --node amd64-builder \
    --platform linux/amd64 \
    ssh://user@amd64-host
```

---

## Layer Caching in CI

### GitHub Actions Cache (Recommended)

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`mode=max` exports all intermediate layers (not just the final image layers), maximizing cache hits for multi-stage builds.

### Registry Cache

Store cache layers in a container registry:

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    cache-from: type=registry,ref=ghcr.io/org/myapp:cache
    cache-to: type=registry,ref=ghcr.io/org/myapp:cache,mode=max
```

### Inline Cache

Embed cache metadata in the image itself (simplest but least efficient):

```yaml
- uses: docker/build-push-action@v6
  with:
    context: .
    cache-from: type=registry,ref=ghcr.io/org/myapp:latest
    cache-to: type=inline
```

### Caching Strategy Comparison

| Strategy | Speed | Storage | Setup | Best For |
|----------|-------|---------|-------|----------|
| `type=gha` | Fast | GitHub-managed (10GB limit) | Easiest | GitHub Actions |
| `type=registry` | Medium | Registry (unlimited) | Easy | Cross-workflow sharing |
| `type=inline` | Slow | In-image (no extra) | Simplest | Public images |
| `type=local` | Fast | Local disk | Manual | Self-hosted runners |
| `type=s3` | Medium | S3 bucket | Moderate | AWS-hosted CI |

### Cache Key Strategies

```yaml
# Use a scoped cache key to avoid cache pollution
- uses: docker/build-push-action@v6
  with:
    cache-from: |
      type=gha,scope=${{ github.ref_name }}
      type=gha,scope=main
    cache-to: type=gha,scope=${{ github.ref_name }},mode=max
```

This pattern uses branch-specific cache with fallback to main branch cache.

---

## Tagging Strategies

### docker/metadata-action Configuration

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: |
      ghcr.io/${{ github.repository }}
      docker.io/${{ github.repository_owner }}/myapp
    tags: |
      # Semantic versioning from git tags
      type=semver,pattern={{version}}        # v1.2.3 → 1.2.3
      type=semver,pattern={{major}}.{{minor}} # v1.2.3 → 1.2
      type=semver,pattern={{major}}           # v1.2.3 → 1

      # Git SHA for traceability
      type=sha,prefix=                       # abc1234

      # Branch name
      type=ref,event=branch                  # main, develop

      # PR number
      type=ref,event=pr                      # pr-42

      # Latest tag on default branch
      type=raw,value=latest,enable=${{ github.ref == format('refs/heads/{0}', 'main') }}
```

### Tagging Decision Matrix

| Trigger | Tags Generated | Push? |
|---------|---------------|-------|
| Push to `main` | `main`, `latest`, `<sha>` | Yes |
| Push to feature branch | `feature-name`, `<sha>` | Yes |
| Git tag `v1.2.3` | `1.2.3`, `1.2`, `1`, `latest`, `<sha>` | Yes |
| Pull request | `pr-42` | No (build only) |

### Immutable vs Mutable Tags

- **Immutable**: Git SHA, semantic versions (`1.2.3`) — never overwritten
- **Mutable**: `latest`, branch names (`main`, `develop`) — updated on each push

Production deployments should always reference immutable tags:

```bash
# Bad: mutable tag, may change
docker pull myapp:latest

# Good: immutable reference
docker pull myapp:1.2.3
docker pull myapp@sha256:abc123...
```

---

## Container Debugging

### Log Inspection

```bash
# Follow logs in real-time
docker logs -f mycontainer

# Show logs since a specific time
docker logs --since 5m mycontainer

# Show last N lines
docker logs --tail 100 mycontainer

# Show timestamps
docker logs -t mycontainer

# Compose: logs for a specific service
docker compose logs -f app
```

### Interactive Shell

```bash
# Execute shell in running container
docker exec -it mycontainer /bin/bash
docker exec -it mycontainer /bin/sh  # Alpine/distroless-debug

# Run as root (even if USER is non-root)
docker exec -u 0 -it mycontainer /bin/bash

# Execute a specific command
docker exec mycontainer python manage.py shell
```

### Container Inspection

```bash
# Full container configuration
docker inspect mycontainer

# Specific fields
docker inspect --format '{{.State.Status}}' mycontainer
docker inspect --format '{{.NetworkSettings.IPAddress}}' mycontainer
docker inspect --format '{{json .Config.Env}}' mycontainer | jq
docker inspect --format '{{.HostConfig.Memory}}' mycontainer
```

### Resource Monitoring

```bash
# Real-time resource usage
docker stats mycontainer

# One-shot stats (for scripting)
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.PIDs}}"
```

### Filesystem Debugging

```bash
# Show filesystem changes since container started
docker diff mycontainer
# A = Added, C = Changed, D = Deleted

# Copy files from container to host
docker cp mycontainer:/app/logs/error.log ./error.log

# Copy files from host to container
docker cp ./config.json mycontainer:/app/config.json
```

### Debugging Distroless Containers

Distroless images have no shell. Use debug variants or ephemeral containers:

```bash
# Use the debug variant (has busybox shell)
FROM gcr.io/distroless/python3-debian12:debug
# Then: docker exec -it mycontainer /busybox/sh

# Kubernetes: ephemeral debug container
kubectl debug -it mycontainer --image=busybox --target=mycontainer
```

### Common Debugging Checklist

```bash
#!/bin/bash
# docker-debug.sh — Quick diagnostic script
CONTAINER=$1

echo "=== Container Status ==="
docker inspect --format '{{.State.Status}} (exit code: {{.State.ExitCode}})' $CONTAINER

echo "=== Recent Logs ==="
docker logs --tail 20 $CONTAINER

echo "=== Resource Usage ==="
docker stats --no-stream $CONTAINER

echo "=== Network ==="
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $CONTAINER

echo "=== Ports ==="
docker port $CONTAINER

echo "=== Filesystem Changes ==="
docker diff $CONTAINER | head -20

echo "=== Health Status ==="
docker inspect --format '{{.State.Health.Status}}' $CONTAINER 2>/dev/null || echo "No healthcheck"
```

---

## References

- **[CI/CD Pipeline Patterns for Docker](references/ci-pipeline-patterns.md)** — Complete GitHub Actions and GitLab CI patterns, build optimization, security scanning in CI, and deployment strategies including blue-green and rolling updates.
- **[Container Registry and Tagging Strategies](references/registry-tagging-strategies.md)** — Registry types and configuration (GHCR, ECR, ACR, GAR, Harbor), tagging conventions, lifecycle management, multi-architecture manifests, and image promotion workflows.
