# Container Runtime Hardening

Comprehensive reference covering resource limits, seccomp profiles, AppArmor, security options, logging and auditing, user namespace remapping, and Docker socket security.

---

## Resource Limits

### Memory Limits

```bash
# Hard memory limit (container killed if exceeded)
docker run --memory=512m myapp

# Memory + swap limit (total ceiling)
docker run --memory=512m --memory-swap=1g myapp

# Disable swap entirely
docker run --memory=512m --memory-swap=512m myapp
```

### CPU Limits

```bash
# Limit to 1.5 CPUs
docker run --cpus=1.5 myapp

# CPU shares (relative weight, default 1024)
docker run --cpu-shares=512 myapp

# Pin to specific CPU cores
docker run --cpuset-cpus="0,1" myapp
```

### PID Limits

Prevent fork bombs by limiting the number of processes:

```bash
docker run --pids-limit=100 myapp
```

### Ulimits

```bash
# Set file descriptor limit
docker run --ulimit nofile=1024:2048 myapp

# Set max processes
docker run --ulimit nproc=64:128 myapp
```

### Compose Resource Configuration

```yaml
services:
  app:
    deploy:
      resources:
        limits:
          cpus: "2.0"
          memory: 1G
          pids: 200
        reservations:
          cpus: "0.5"
          memory: 256M
    ulimits:
      nofile:
        soft: 1024
        hard: 2048
      nproc:
        soft: 64
        hard: 128
```

### OOM (Out of Memory) Behavior

```yaml
services:
  app:
    # Disable OOM killer (container pauses instead of being killed)
    oom_kill_disable: true

    # Set OOM score adjustment (-1000 to 1000, lower = less likely to be killed)
    oom_score_adj: -500
```

---

## Seccomp Profiles

### Default Profile

Docker applies a default seccomp profile that blocks ~44 dangerous syscalls (e.g., `reboot`, `mount`, `kexec_load`). Verify it is active:

```bash
docker inspect --format '{{.HostConfig.SecurityOpt}}' mycontainer
# Should not show "seccomp=unconfined"
```

### Custom Seccomp Profile

Create a restrictive profile that only allows needed syscalls:

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_AARCH64"
  ],
  "syscalls": [
    {
      "names": [
        "accept4", "access", "arch_prctl", "bind", "brk",
        "clock_gettime", "clone", "close", "connect",
        "epoll_create1", "epoll_ctl", "epoll_wait",
        "eventfd2", "execve", "exit", "exit_group",
        "fchmod", "fchown", "fcntl", "fstat", "futex",
        "getcwd", "getdents64", "getegid", "geteuid",
        "getgid", "getpeername", "getpid", "getppid",
        "getrandom", "getsockname", "getsockopt", "getuid",
        "ioctl", "listen", "lseek", "madvise", "memfd_create",
        "mmap", "mprotect", "munmap", "nanosleep",
        "newfstatat", "openat", "pipe2", "poll", "pread64",
        "pwrite64", "read", "readlink", "recvfrom", "recvmsg",
        "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
        "sched_getaffinity", "sched_yield", "sendmsg", "sendto",
        "set_robust_list", "set_tid_address", "setgid",
        "setsockopt", "setuid", "shutdown", "sigaltstack",
        "socket", "stat", "statfs", "tgkill", "uname",
        "unlink", "wait4", "write", "writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

Apply the profile:

```bash
docker run --security-opt seccomp=./custom-seccomp.json myapp
```

```yaml
# Compose
services:
  app:
    security_opt:
      - seccomp:./custom-seccomp.json
```

### Generating a Profile

Use `strace` or `oci-seccomp-bpf-hook` to record which syscalls your application uses, then build a profile from that:

```bash
# Record syscalls during a test run
docker run --security-opt seccomp=unconfined \
    --cap-add SYS_PTRACE \
    strace -ff -o /tmp/trace myapp

# Analyze unique syscalls
cat /tmp/trace.* | awk '{print $1}' | sort -u
```

---

## AppArmor Profiles

### Default Profile

Docker applies the `docker-default` AppArmor profile automatically:

```bash
# Verify AppArmor is active
docker inspect --format '{{.AppArmorProfile}}' mycontainer
# Output: docker-default
```

### Custom AppArmor Profile

```
# /etc/apparmor.d/docker-myapp
#include <tunables/global>

profile docker-myapp flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>

  # Allow network access
  network inet tcp,
  network inet udp,
  network inet6 tcp,

  # Allow reading application files
  /app/** r,
  /app/run.py ix,

  # Allow writing to specific directories only
  /tmp/** rw,
  /var/run/** rw,
  /app/logs/** rw,

  # Deny access to sensitive paths
  deny /etc/shadow r,
  deny /etc/passwd w,
  deny /proc/*/mem rw,
  deny /sys/** w,
}
```

Load and apply:

```bash
sudo apparmor_parser -r /etc/apparmor.d/docker-myapp
docker run --security-opt apparmor=docker-myapp myapp
```

---

## Security Options

### no-new-privileges

Prevent processes from gaining additional privileges via `setuid` or `setgid` binaries:

```bash
docker run --security-opt no-new-privileges:true myapp
```

This is one of the most important security flags. It ensures that even if a vulnerability allows code execution, the attacker cannot escalate to root via SUID binaries.

```yaml
# Compose
services:
  app:
    security_opt:
      - no-new-privileges:true
```

### Combined Security Options

```yaml
services:
  app:
    image: myapp:latest
    user: "1001:1001"
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
      - seccomp:./seccomp-profile.json
    tmpfs:
      - /tmp:noexec,nosuid,size=100M
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
          pids: 100
```

---

## Logging and Auditing

### Container Logging Drivers

```yaml
services:
  app:
    logging:
      driver: json-file
      options:
        max-size: "10m"   # Max log file size
        max-file: "3"     # Number of rotated files
        tag: "{{.Name}}"  # Tag for log identification
```

Available drivers:

| Driver | Use Case |
|--------|----------|
| `json-file` | Default, local development |
| `syslog` | Central syslog server |
| `journald` | systemd journal |
| `fluentd` | Fluentd log collector |
| `awslogs` | AWS CloudWatch |
| `gcplogs` | Google Cloud Logging |
| `splunk` | Splunk HTTP Event Collector |

### Log Rotation

Always configure log rotation to prevent disk exhaustion:

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  }
}
```

### Docker Events Monitoring

Monitor container lifecycle events for auditing:

```bash
# Watch all events
docker events

# Filter by container events
docker events --filter type=container

# Filter by specific actions
docker events --filter event=start --filter event=stop --filter event=die

# JSON output for parsing
docker events --format '{{json .}}'
```

### Audit Logging Script

```bash
#!/bin/bash
# Container audit logger
docker events --format '{{json .}}' | while read event; do
  timestamp=$(echo "$event" | jq -r '.time')
  action=$(echo "$event" | jq -r '.Action')
  actor=$(echo "$event" | jq -r '.Actor.Attributes.name')
  echo "$(date -d @$timestamp '+%Y-%m-%d %H:%M:%S') | $action | $actor" >> /var/log/docker-audit.log
done
```

---

## User Namespace Remapping

### Configuration

Map container root (UID 0) to an unprivileged host user:

```json
// /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

Docker creates a `dockremap` user and assigns a subordinate UID/GID range:

```bash
# Check subordinate ranges
cat /etc/subuid
# dockremap:100000:65536

cat /etc/subgid
# dockremap:100000:65536
```

With this configuration, container root (UID 0) maps to host UID 100000 — an unprivileged user.

### Custom User Mapping

```json
{
  "userns-remap": "myuser:mygroup"
}
```

### Limitations

- Volumes may have permission issues (host UID ≠ container UID)
- Some images require real root and may break
- Cannot be used with `--privileged`
- Network port binding below 1024 requires additional configuration

---

## Docker Socket Security

### Risks of Mounting the Docker Socket

Mounting `/var/run/docker.sock` into a container gives that container full control over the Docker daemon — equivalent to root access on the host:

```yaml
# Dangerous: full Docker control inside container
services:
  ci-runner:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

An attacker who compromises this container can:
- Start privileged containers
- Mount host filesystems
- Execute commands on the host
- Access all other containers

### Read-Only Socket Mount

```yaml
services:
  monitor:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Note: Read-only mount on the socket file still allows API calls. The `:ro` flag only prevents deleting/replacing the socket file itself.

### Docker Socket Proxy

Use a TCP proxy that filters Docker API calls:

```yaml
services:
  docker-proxy:
    image: tecnativa/docker-socket-proxy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      CONTAINERS: 1      # Allow container listing
      SERVICES: 0        # Deny service management
      TASKS: 0           # Deny task management
      NETWORKS: 0        # Deny network management
      VOLUMES: 0         # Deny volume management
      POST: 0            # Deny all write operations

  ci-runner:
    environment:
      DOCKER_HOST: tcp://docker-proxy:2375
    depends_on:
      - docker-proxy
```

### Alternatives to Socket Mounting

| Alternative | Use Case | Security |
|------------|----------|----------|
| Kaniko | Building images in CI | No Docker daemon needed |
| Buildah | Building OCI images | Rootless, daemonless |
| Podman | Running containers | Rootless, daemonless |
| Docker-in-Docker (dind) | CI/CD runners | Isolated Docker daemon |
| Sysbox | Nested containers | Enhanced isolation |

### Docker-in-Docker (DinD)

```yaml
services:
  dind:
    image: docker:dind
    privileged: true
    environment:
      DOCKER_TLS_CERTDIR: /certs
    volumes:
      - dind-certs:/certs

  ci-runner:
    image: docker:cli
    environment:
      DOCKER_HOST: tcp://dind:2376
      DOCKER_TLS_VERIFY: 1
      DOCKER_CERT_PATH: /certs/client
    volumes:
      - dind-certs:/certs/client:ro
    depends_on:
      - dind

volumes:
  dind-certs:
```
