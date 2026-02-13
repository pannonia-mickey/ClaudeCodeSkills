---
name: Docker Compose Design
description: This skill should be used when the user asks about "docker compose", "compose.yaml", "docker-compose.yml", "Docker volume", "Docker network", "Docker healthcheck", "compose profile", or "Docker development workflow". It covers Docker Compose v2 orchestration including service definitions, networking, volumes, healthchecks, environment management, profiles, and development workflows with watch mode.
---

## Compose File Structure

Use `compose.yaml` as the filename (preferred over `docker-compose.yml`). The `version` field is deprecated in Compose v2 and should be omitted:

```yaml
# compose.yaml — no version field needed
services:
  app:
    build: .
    ports:
      - "8000:8000"
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

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

### Top-Level Keys

| Key | Purpose |
|-----|---------|
| `services` | Container definitions (required) |
| `networks` | Custom network definitions |
| `volumes` | Named volume definitions |
| `configs` | Configuration file definitions |
| `secrets` | Secret definitions |

---

## Service Configuration

### Build Configuration

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
      args:
        PYTHON_VERSION: "3.12"
        BUILD_ENV: development
      cache_from:
        - type=registry,ref=ghcr.io/org/app:cache
```

### Ports

```yaml
services:
  app:
    ports:
      # Short syntax: HOST:CONTAINER
      - "8000:8000"
      - "127.0.0.1:8443:443"

      # Long syntax
      - target: 8000
        published: 8000
        protocol: tcp
        mode: host
```

### Environment vs env_file

```yaml
services:
  app:
    # Inline environment variables
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
      REDIS_URL: redis://redis:6379/0
      DEBUG: "false"

    # Or load from files
    env_file:
      - .env
      - .env.local
```

### Restart Policies

```yaml
services:
  app:
    # Options: no, always, unless-stopped, on-failure
    restart: unless-stopped

  worker:
    restart: on-failure
```

### Resource Limits

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 512M
        reservations:
          cpus: "0.5"
          memory: 256M
```

### Full-Featured Service

```yaml
services:
  app:
    build:
      context: .
      target: production
    image: myapp:latest
    container_name: myapp-web
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/myapp
    env_file:
      - .env
    volumes:
      - static:/app/staticfiles
      - media:/app/media
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 5s
      start_period: 15s
      retries: 3
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

---

## Healthchecks

### PostgreSQL

```yaml
db:
  image: postgres:16-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
    interval: 5s
    timeout: 5s
    retries: 5
    start_period: 10s
```

### MySQL

```yaml
db:
  image: mysql:8
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5
```

### Redis

```yaml
redis:
  image: redis:7-alpine
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 5s
    timeout: 3s
    retries: 5
```

### Elasticsearch

```yaml
elasticsearch:
  image: elasticsearch:8.12.0
  healthcheck:
    test: ["CMD-SHELL", "curl -fsSL http://localhost:9200/_cluster/health | grep -q '\"status\":\"green\\|yellow\"'"]
    interval: 10s
    timeout: 10s
    retries: 10
    start_period: 30s
```

### RabbitMQ

```yaml
rabbitmq:
  image: rabbitmq:3-management-alpine
  healthcheck:
    test: ["CMD", "rabbitmq-diagnostics", "check_running"]
    interval: 10s
    timeout: 10s
    retries: 5
```

---

## Depends On with Conditions

### service_healthy

Wait for a service to pass its healthcheck before starting:

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
```

### service_completed_successfully

Wait for a one-off task to finish before starting (init container pattern):

```yaml
services:
  migrate:
    build: .
    command: python manage.py migrate --noinput
    depends_on:
      db:
        condition: service_healthy

  app:
    build: .
    depends_on:
      migrate:
        condition: service_completed_successfully
      db:
        condition: service_healthy
```

### service_started

Start after the service is created (no health guarantee):

```yaml
services:
  worker:
    depends_on:
      broker:
        condition: service_started
```

---

## Profiles

Use profiles to define optional services that only start when explicitly activated:

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"

  db:
    image: postgres:16-alpine

  # Debug tools — only start with --profile debug
  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@local.dev
      PGADMIN_DEFAULT_PASSWORD: admin

  # Monitoring — only start with --profile monitoring
  prometheus:
    image: prom/prometheus
    profiles: [monitoring]
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    profiles: [monitoring]
    ports:
      - "3000:3000"

  # Test services — only start with --profile test
  test-runner:
    build:
      context: .
      target: test
    profiles: [test]
    depends_on:
      db:
        condition: service_healthy
    command: pytest
```

```bash
# Start core services only
docker compose up

# Start with debug tools
docker compose --profile debug up

# Start with monitoring
docker compose --profile monitoring up

# Combine profiles
docker compose --profile debug --profile monitoring up
```

---

## Environment Variable Management

### .env File (Auto-Loaded)

Compose automatically loads a `.env` file from the project directory:

```bash
# .env
POSTGRES_USER=myapp
POSTGRES_PASSWORD=secret
POSTGRES_DB=myapp_db
APP_PORT=8000
```

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}

  app:
    ports:
      - "${APP_PORT:-8000}:8000"
```

### Variable Substitution

```yaml
services:
  app:
    image: myapp:${TAG:-latest}           # Default value
    environment:
      DEBUG: ${DEBUG:?DEBUG must be set}   # Error if not set
      LOG_LEVEL: ${LOG_LEVEL:-info}       # Default to "info"
```

### Multiple env_file

```yaml
services:
  app:
    env_file:
      - .env                 # Base configuration
      - .env.${ENV:-dev}     # Environment-specific overrides
      - .env.local           # Local developer overrides (gitignored)
```

### Precedence Order (Highest to Lowest)

1. `docker compose run -e VAR=value`
2. Shell environment variables
3. `environment` key in compose.yaml
4. `--env-file` flag
5. `env_file` key in compose.yaml
6. `.env` file

---

## Development Workflow with Watch

Compose Watch (`develop.watch`) provides automatic file synchronization and rebuilds during development:

### Python Application

```yaml
services:
  app:
    build:
      context: .
      target: development
    ports:
      - "8000:8000"
    develop:
      watch:
        # Sync source code changes (hot-reload via watchfiles/uvicorn)
        - action: sync
          path: ./src
          target: /app/src

        # Rebuild when dependencies change
        - action: rebuild
          path: ./requirements.txt

        # Sync config and restart the service
        - action: sync+restart
          path: ./config
          target: /app/config
```

### Node.js Application

```yaml
services:
  frontend:
    build:
      context: ./frontend
      target: development
    ports:
      - "3000:3000"
    develop:
      watch:
        - action: sync
          path: ./frontend/src
          target: /app/src

        - action: rebuild
          path: ./frontend/package.json

        - action: rebuild
          path: ./frontend/package-lock.json
```

Start with watch mode:

```bash
docker compose watch
```

### Watch Actions

| Action | When to Use |
|--------|-------------|
| `sync` | Source code changes with hot-reload (e.g., uvicorn --reload, webpack HMR) |
| `rebuild` | Dependency changes (package.json, requirements.txt) that need a new image |
| `sync+restart` | Config file changes that need a process restart but not a rebuild |

---

## References

- **[Compose Patterns and Advanced Configuration](references/compose-patterns.md)** — Multi-environment Compose files, init container pattern, sidecar pattern, scaling, secrets management, and extension fields with YAML anchors.
- **[Compose Networking and Storage](references/compose-networking.md)** — Default and custom networking, service discovery, volume patterns, port mapping, DNS configuration, and advanced network settings.
