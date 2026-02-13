# Compose Patterns and Advanced Configuration

Comprehensive reference covering multi-environment Compose setups, init container patterns, sidecar patterns, scaling strategies, secrets management, and extension fields with YAML anchors.

---

## Multi-Environment Compose

### Override File Pattern

Compose automatically loads `compose.yaml` and `compose.override.yaml` when running `docker compose up`:

```yaml
# compose.yaml — base configuration (production-ready)
services:
  app:
    image: myapp:${TAG:-latest}
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

```yaml
# compose.override.yaml — development overrides (auto-loaded)
services:
  app:
    build:
      context: .
      target: development
    volumes:
      - .:/app:cached
    environment:
      DEBUG: "true"
    ports:
      - "8000:8000"
      - "5678:5678"  # debugger port

  db:
    ports:
      - "5432:5432"  # expose DB for local tools
```

### Environment-Specific Files

```yaml
# compose.prod.yaml — production overrides
services:
  app:
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

  db:
    deploy:
      resources:
        limits:
          memory: 2G
```

```yaml
# compose.test.yaml — test overrides
services:
  app:
    build:
      target: test
    command: pytest --tb=short -q
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/test_db
    depends_on:
      db:
        condition: service_healthy

  db:
    environment:
      POSTGRES_DB: test_db
    tmpfs:
      - /var/lib/postgresql/data  # ephemeral for speed
```

Usage:

```bash
# Development (loads compose.yaml + compose.override.yaml)
docker compose up

# Production (skip override, use prod)
docker compose -f compose.yaml -f compose.prod.yaml up -d

# Testing
docker compose -f compose.yaml -f compose.test.yaml run --rm app
```

---

## Init Container Pattern

Use `depends_on` with `condition: service_completed_successfully` to run one-off setup tasks before the main application starts.

### Database Migrations

```yaml
services:
  migrate:
    build: .
    command: python manage.py migrate --noinput
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy
    restart: "no"

  seed:
    build: .
    command: python manage.py loaddata fixtures/initial_data.json
    depends_on:
      migrate:
        condition: service_completed_successfully
    restart: "no"

  app:
    build: .
    command: gunicorn config.wsgi --bind 0.0.0.0:8000
    depends_on:
      migrate:
        condition: service_completed_successfully
    ports:
      - "8000:8000"

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### Startup Order

1. `db` starts and becomes healthy
2. `migrate` runs and completes successfully
3. `seed` runs (depends on migrate) and completes
4. `app` starts (depends on migrate)

---

## Sidecar Pattern

### Log Collector Sidecar

```yaml
services:
  app:
    build: .
    volumes:
      - app-logs:/var/log/app
    logging:
      driver: json-file
      options:
        max-size: "10m"

  log-collector:
    image: fluent/fluentd:v1.16-1
    volumes:
      - app-logs:/var/log/app:ro
      - ./fluentd.conf:/fluentd/etc/fluentd.conf
    depends_on:
      - app

volumes:
  app-logs:
```

### Reverse Proxy Sidecar

```yaml
services:
  app:
    build: .
    expose:
      - "8000"  # internal only

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certs:/etc/nginx/certs:ro
      - static:/usr/share/nginx/static:ro
    depends_on:
      app:
        condition: service_healthy

volumes:
  certs:
  static:
```

### Metrics Exporter Sidecar

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"

  prometheus-exporter:
    image: prom/statsd-exporter
    ports:
      - "9102:9102"  # metrics endpoint
    command:
      - "--statsd.listen-udp=:9125"
      - "--web.listen-address=:9102"
```

---

## Scaling Services

### Horizontal Scaling with Compose

```bash
docker compose up --scale worker=5
```

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"

  worker:
    build: .
    command: celery -A config worker -l info
    depends_on:
      broker:
        condition: service_healthy
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  broker:
    image: rabbitmq:3-alpine
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_running"]
      interval: 10s
      timeout: 10s
      retries: 5
```

### Load Balancing with Traefik

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  app:
    build: .
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`app.localhost`)"
      - "traefik.http.services.app.loadbalancer.server.port=8000"
    deploy:
      replicas: 3
```

---

## Secrets Management in Compose

### File-Based Secrets

```yaml
services:
  app:
    build: .
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

Inside the container, secrets are mounted at `/run/secrets/<name>`:

```python
# Reading secrets in application code
from pathlib import Path

def get_secret(name: str) -> str:
    secret_file = Path(f"/run/secrets/{name}")
    if secret_file.exists():
        return secret_file.read_text().strip()
    # Fall back to environment variable
    import os
    return os.environ[name.upper()]
```

### External Secrets (Docker Swarm)

```yaml
secrets:
  db_password:
    external: true  # Must exist in Swarm: docker secret create db_password ./file
```

---

## Extension Fields and YAML Anchors

### Reusable Fragments with x-

```yaml
x-common-env: &common-env
  DATABASE_URL: postgres://user:pass@db:5432/myapp
  REDIS_URL: redis://redis:6379/0
  SECRET_KEY: ${SECRET_KEY}

x-common-logging: &common-logging
  logging:
    driver: json-file
    options:
      max-size: "10m"
      max-file: "3"

x-common-deploy: &common-deploy
  deploy:
    resources:
      limits:
        cpus: "1.0"
        memory: 512M
    restart_policy:
      condition: on-failure
      max_attempts: 3

services:
  app:
    build: .
    environment:
      <<: *common-env
      APP_ROLE: web
    <<: *common-logging
    <<: *common-deploy

  worker:
    build: .
    command: celery -A config worker
    environment:
      <<: *common-env
      APP_ROLE: worker
    <<: *common-logging
    <<: *common-deploy

  beat:
    build: .
    command: celery -A config beat
    environment:
      <<: *common-env
      APP_ROLE: beat
    <<: *common-logging
    <<: *common-deploy
```

### Shared Build Configuration

```yaml
x-app-build: &app-build
  build:
    context: .
    dockerfile: Dockerfile
    args:
      PYTHON_VERSION: "3.12"

services:
  app:
    <<: *app-build
    command: gunicorn config.wsgi --bind 0.0.0.0:8000

  worker:
    <<: *app-build
    command: celery -A config worker

  migrate:
    <<: *app-build
    command: python manage.py migrate --noinput
    restart: "no"
```

This avoids duplicating the build configuration across multiple services that share the same image.

---

## Compose CLI Commands

### Common Operations

```bash
# Start all services in detached mode
docker compose up -d

# Start specific services
docker compose up -d app db

# Rebuild images before starting
docker compose up -d --build

# Stop and remove containers, networks
docker compose down

# Stop and remove containers, networks, AND volumes
docker compose down -v

# View logs (follow mode)
docker compose logs -f app

# Execute command in running container
docker compose exec app python manage.py shell

# Run a one-off command in a new container
docker compose run --rm app pytest

# View service status
docker compose ps

# Pull latest images
docker compose pull
```
