---
name: Docker Security
description: This skill should be used when the user asks about "Docker security", "container security", "non-root container", "Docker scan", "Trivy", "container hardening", "Docker secrets", or "image vulnerability". It covers container security best practices including non-root execution, minimal base images, vulnerability scanning, secrets management, Linux capabilities, read-only filesystems, supply chain security, and runtime hardening.
---

## Non-Root Containers

### Creating a Dedicated User

Always create a non-root user and switch to it before the application runs:

```dockerfile
# Create a system group and user with no login shell and no home directory
RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser

# For Alpine-based images
RUN addgroup -S appuser && adduser -S -G appuser -h /app -s /sbin/nologin appuser
```

### USER Instruction Placement

Place `USER` after all commands that require root privileges (package installation, directory creation) but before `ENTRYPOINT`/`CMD`:

```dockerfile
FROM python:3.12-slim

# Root operations
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd -r appuser && useradd -r -g appuser -d /app -s /sbin/nologin appuser
WORKDIR /app

# Set ownership during copy
COPY --chown=appuser:appuser . .

# Switch to non-root before running
USER appuser
EXPOSE 8000
ENTRYPOINT ["gunicorn", "config.wsgi:application"]
```

### Numeric UID vs Named User

Use numeric UID when the user may not exist in the container's `/etc/passwd` (e.g., distroless images):

```dockerfile
# Named user (requires user to exist)
USER appuser

# Numeric UID (works even without /etc/passwd)
USER 1001:1001
```

### Complete Non-Root Pattern

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile --prefix=/install -r requirements.txt
COPY . .

FROM python:3.12-slim

# Install runtime dependencies as root
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 tini \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r app --gid=1001 \
    && useradd -r -g app --uid=1001 -d /app -s /sbin/nologin app

WORKDIR /app

# Copy with correct ownership
COPY --from=builder --chown=app:app /install /usr/local
COPY --from=builder --chown=app:app /app /app

# Switch to non-root
USER app

EXPOSE 8000
ENTRYPOINT ["tini", "--"]
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

## Minimal Base Images

### Attack Surface Reduction

Fewer packages in the base image means fewer potential vulnerabilities:

| Base Image | Packages | Typical CVEs | Shell | Package Manager |
|-----------|----------|-------------|-------|-----------------|
| `python:3.12` (full) | ~400 | 50-100+ | Yes | apt |
| `python:3.12-slim` | ~100 | 10-30 | Yes | apt |
| `python:3.12-alpine` | ~30 | 5-15 | Yes | apk |
| `gcr.io/distroless/python3` | ~10 | 0-5 | No | None |
| `cgr.dev/chainguard/python` | ~10 | 0-3 | No | apk (dev variant) |
| `scratch` | 0 | 0 | No | None |

### Distroless Images

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-compile --target=/app/deps -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=builder /app /app
ENV PYTHONPATH=/app/deps
EXPOSE 8000
ENTRYPOINT ["python", "app.py"]
```

### Chainguard Images

Chainguard provides hardened, minimal images with near-zero CVEs:

```dockerfile
FROM cgr.dev/chainguard/python:latest-dev AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-compile --target=/app/deps -r requirements.txt
COPY . .

FROM cgr.dev/chainguard/python:latest
WORKDIR /app
COPY --from=builder /app /app
ENV PYTHONPATH=/app/deps
ENTRYPOINT ["python", "app.py"]
```

---

## Vulnerability Scanning

### Trivy

```bash
# Scan a local image
trivy image myapp:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Fail on critical vulnerabilities (for CI)
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Output as SARIF for GitHub Security
trivy image --format sarif --output trivy-results.sarif myapp:latest

# Scan a Dockerfile
trivy config Dockerfile
```

### GitHub Actions Integration

```yaml
- name: Build Docker image
  run: docker build -t myapp:${{ github.sha }} .

- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif
```

### Grype

```bash
# Scan an image
grype myapp:latest

# Fail on high/critical
grype myapp:latest --fail-on high
```

### Docker Scout

```bash
# Quick scan
docker scout quickview myapp:latest

# Detailed CVE report
docker scout cves myapp:latest

# Compare two image versions
docker scout compare myapp:v2 --to myapp:v1
```

---

## Secrets Management

### Build-Time Secrets (BuildKit)

Never use `ARG` or `ENV` for secrets — they persist in image layers:

```dockerfile
# Bad: visible in docker history
ARG DATABASE_PASSWORD
RUN echo "db_password=${DATABASE_PASSWORD}" > /app/config

# Good: BuildKit secret mount (not stored in any layer)
RUN --mount=type=secret,id=db_password \
    cat /run/secrets/db_password > /app/config && \
    chmod 600 /app/config
```

```bash
docker build --secret id=db_password,src=./secrets/db_password.txt .
```

### Runtime Secrets in Compose

```yaml
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - api_key
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

  db:
    image: postgres:16-alpine
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    file: ./secrets/api_key.txt
```

### Why Not Environment Variables

```bash
# Environment variables are visible in docker inspect
docker inspect mycontainer --format '{{.Config.Env}}'
# Output: [DB_PASSWORD=mysecret ...]

# And in /proc inside the container
cat /proc/1/environ
```

Secret files mounted at `/run/secrets/` are not visible in `docker inspect` or image layers.

---

## Linux Capabilities

### Drop All, Add Selectively

```bash
# Drop ALL capabilities, add only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp
```

### Compose Configuration

```yaml
services:
  app:
    image: myapp:latest
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # bind to ports < 1024
```

### Common Capabilities by Application Type

| Application | Required Capabilities |
|------------|----------------------|
| Web server (port > 1024) | None (drop ALL) |
| Web server (port 80/443) | NET_BIND_SERVICE |
| Health check with ping | NET_RAW |
| Container management | SYS_ADMIN (dangerous) |
| Time sync (NTP) | SYS_TIME |
| Debug with strace | SYS_PTRACE |

### Minimal Web Server

```yaml
services:
  app:
    image: myapp:latest
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    user: "1001:1001"
```

---

## Read-Only Filesystem

### Docker Run

```bash
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp
```

### Compose Configuration

```yaml
services:
  app:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp:size=100M,mode=1777
      - /var/run:size=10M
      - /app/cache:size=50M,uid=1001,gid=1001
    volumes:
      - app-data:/app/data  # writable persistent data
```

### Common Writable Directories

| Directory | Purpose | Mount Type |
|-----------|---------|-----------|
| `/tmp` | Temporary files | tmpfs |
| `/var/run` | PID files, sockets | tmpfs |
| `/var/log` | Application logs | tmpfs or volume |
| `/app/cache` | Application cache | tmpfs |
| `/app/uploads` | User uploads | named volume |
| `/app/data` | Persistent data | named volume |

### Complete Hardened Container

```yaml
services:
  app:
    image: myapp:latest
    read_only: true
    user: "1001:1001"
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    tmpfs:
      - /tmp:size=100M
      - /var/run:size=10M
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
          pids: 100
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')"]
      interval: 30s
      timeout: 5s
      retries: 3
```

---

## References

- **[Container Runtime Hardening](references/container-hardening.md)** — Resource limits, seccomp profiles, AppArmor, security options, logging, user namespace remapping, and Docker socket security.
- **[Container Supply Chain Security](references/supply-chain-security.md)** — Base image provenance, SHA pinning, SBOM generation, build reproducibility, registry security, policy enforcement with Hadolint, and image signing with cosign.
