# Compose Networking and Storage

Comprehensive reference covering default and custom networking, service discovery, volume patterns, port mapping, DNS configuration, and advanced network settings in Docker Compose.

---

## Default Networking

### Auto-Created Project Network

When you run `docker compose up`, Compose creates a default bridge network named `<project>_default`. All services join this network automatically:

```bash
# Project directory: myapp/
docker compose up
# Creates network: myapp_default
# Services: myapp-app-1, myapp-db-1
```

### Service Discovery via Hostname

Services can reach each other using the service name as hostname:

```yaml
services:
  app:
    build: .
    environment:
      # "db" resolves to the db service container IP
      DATABASE_URL: postgres://user:pass@db:5432/myapp
      # "redis" resolves to the redis service container IP
      REDIS_URL: redis://redis:6379/0

  db:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine
```

DNS resolution is handled by Docker's embedded DNS server (127.0.0.11). Service names resolve to the container's IP on the shared network.

### Container Name Resolution

With `docker compose up --scale worker=3`, DNS for `worker` returns all three container IPs (round-robin):

```python
import socket
# Returns all IPs for scaled service
ips = socket.getaddrinfo("worker", 8000)
```

---

## Custom Networks

### Multiple Networks for Service Isolation

Isolate frontend-accessible services from backend-only services:

```yaml
services:
  nginx:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "80:80"

  app:
    build: .
    networks:
      - frontend    # nginx can reach app
      - backend     # app can reach db

  db:
    image: postgres:16-alpine
    networks:
      - backend     # only app can reach db
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # no external access

volumes:
  pgdata:
```

With this configuration:
- `nginx` can reach `app` (both on `frontend`)
- `app` can reach `db` and `redis` (both on `backend`)
- `nginx` cannot reach `db` or `redis` (not on `backend`)
- `db` and `redis` have no internet access (`internal: true`)

### Network Aliases

Give a service additional DNS names on a specific network:

```yaml
services:
  db-primary:
    image: postgres:16-alpine
    networks:
      backend:
        aliases:
          - database
          - postgres
          - db
```

Now other services on `backend` can reach this service via `db-primary`, `database`, `postgres`, or `db`.

### External Networks

Connect to networks created outside of Compose (for cross-project communication):

```yaml
networks:
  shared:
    external: true
    name: my-shared-network
```

Create the external network first:

```bash
docker network create my-shared-network
```

### Network with Custom Subnet

```yaml
networks:
  backend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
          gateway: 172.28.0.1
```

Assign static IPs to services:

```yaml
services:
  db:
    image: postgres:16-alpine
    networks:
      backend:
        ipv4_address: 172.28.0.10
```

---

## Volume Patterns

### Named Volumes

Named volumes are managed by Docker and persist across container restarts:

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
    driver: local
```

### Bind Mounts

Bind mounts map a host directory into the container. Use for development source code:

```yaml
services:
  app:
    build: .
    volumes:
      # Bind mount with consistency hint
      - ./src:/app/src:cached

      # Read-only bind mount for config
      - ./config/nginx.conf:/etc/nginx/nginx.conf:ro
```

Consistency hints (macOS Docker Desktop):
- `cached` — host is authoritative, container may lag
- `delegated` — container is authoritative, host may lag
- `consistent` — full consistency (default, slowest)

### tmpfs Mounts

In-memory filesystem for temporary data that should not persist:

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /var/run

    # Or with options
    volumes:
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100M
          mode: 1777
```

### Volume for Database Backup

```yaml
services:
  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data

  db-backup:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data:ro
      - ./backups:/backups
    command: >
      sh -c 'pg_dump -U postgres myapp > /backups/backup-$$(date +%Y%m%d-%H%M%S).sql'
    profiles: [backup]

volumes:
  pgdata:
```

Run backup on demand:

```bash
docker compose --profile backup run --rm db-backup
```

### Anonymous Volumes

Anonymous volumes are created without a name and are removed with `docker compose down -v`. Use for ephemeral data only:

```yaml
services:
  test-db:
    image: postgres:16-alpine
    volumes:
      - /var/lib/postgresql/data  # anonymous volume
    profiles: [test]
```

For production, always use named volumes.

### Volume from Another Container

Share a volume defined in one service with another:

```yaml
services:
  app:
    build: .
    volumes:
      - static:/app/staticfiles

  nginx:
    image: nginx:alpine
    volumes:
      - static:/usr/share/nginx/static:ro
    depends_on:
      - app

volumes:
  static:
```

---

## Port Mapping

### Short Syntax

```yaml
ports:
  # HOST:CONTAINER
  - "8000:8000"

  # Bind to specific interface
  - "127.0.0.1:8000:8000"

  # Random host port
  - "8000"

  # UDP protocol
  - "514:514/udp"

  # Port range
  - "8000-8010:8000-8010"
```

### Long Syntax

```yaml
ports:
  - target: 8000        # Container port
    published: 8000     # Host port
    protocol: tcp
    mode: host

  - target: 8443
    published: 443
    protocol: tcp
```

### Expose vs Ports

```yaml
services:
  app:
    expose:
      - "8000"    # Internal only — accessible by other services
    ports:
      - "80:80"   # Published — accessible from host
```

Use `expose` for inter-service communication. Use `ports` to publish to the host.

### Host Network Mode

```yaml
services:
  app:
    network_mode: host
    # No port mapping needed — uses host network directly
```

The container shares the host's network namespace. No network isolation. Useful for performance-critical applications or services that need to bind many ports.

---

## DNS and Service Discovery

### Container DNS Configuration

```yaml
services:
  app:
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - mycompany.local
    dns_opt:
      - ndots:1
```

### Extra Hosts

Add custom hostname-to-IP mappings:

```yaml
services:
  app:
    extra_hosts:
      - "api.local:192.168.1.100"
      - "host.docker.internal:host-gateway"
```

`host-gateway` is a special value that resolves to the host machine IP, enabling container-to-host communication.

### Links (Legacy)

```yaml
services:
  app:
    links:
      - db:database  # creates alias "database" for service "db"
```

Links are legacy. Prefer networks with aliases instead:

```yaml
services:
  app:
    networks:
      - backend
  db:
    networks:
      backend:
        aliases:
          - database
```

---

## Advanced Network Configuration

### IPAM (IP Address Management)

```yaml
networks:
  app-net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 10.5.0.0/16
          ip_range: 10.5.0.0/24
          gateway: 10.5.0.1
          aux_addresses:
            host1: 10.5.0.2
            host2: 10.5.0.3
```

### Network Driver Options

```yaml
networks:
  app-net:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.name: br-myapp
      com.docker.network.bridge.enable_icc: "true"
      com.docker.network.bridge.enable_ip_masquerade: "true"
      com.docker.network.driver.mtu: "1500"
```

### Disable Default Network

```yaml
networks:
  default:
    external: true
    name: my-custom-default
```

### IPv6 Support

```yaml
networks:
  app-net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 172.28.0.0/16
        - subnet: "fd00::/64"
```

---

## Troubleshooting Network Issues

### Inspect Network

```bash
# List networks
docker network ls

# Inspect a specific network
docker network inspect myapp_default

# Find which containers are on a network
docker network inspect myapp_default --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

### DNS Resolution Debugging

```bash
# Test DNS from inside a container
docker compose exec app nslookup db
docker compose exec app getent hosts db

# Check embedded DNS server
docker compose exec app cat /etc/resolv.conf
```

### Connectivity Testing

```bash
# Test connectivity between services
docker compose exec app ping db
docker compose exec app curl -v http://api:8000/health/

# Check listening ports
docker compose exec app netstat -tlnp
```
