---
name: DevOps Mastery
description: This skill should be used when the user asks about "12-factor app", "GitOps principles", "environment management", "release management", "incident response", "runbooks", "DevOps culture", or "platform engineering". It covers core DevOps principles, environment strategies, release management, and incident response.
---

# DevOps Mastery

## 12-Factor App Principles

```yaml
# Applied to modern containerized applications
I.    Codebase:        One repo per service, tracked in git
II.   Dependencies:    Explicitly declared (package.json, requirements.txt)
III.  Config:          Environment variables, never in code
IV.   Backing Services: Treat databases, caches, queues as attached resources
V.    Build/Release/Run: Strict separation (CI builds, CD releases, runtime runs)
VI.   Processes:       Stateless processes, store state in backing services
VII.  Port Binding:    Export services via port binding (not app server)
VIII. Concurrency:     Scale via process model (horizontal scaling)
IX.   Disposability:   Fast startup, graceful shutdown
X.    Dev/Prod Parity: Keep environments as similar as possible
XI.   Logs:            Treat logs as event streams (stdout, collected externally)
XII.  Admin Processes: Run admin tasks as one-off processes (migrations, scripts)
```

## Environment Strategy

```
┌──────────────────────────────────────────────────────────┐
│ Environment │ Purpose        │ Deploy       │ Access     │
├──────────────────────────────────────────────────────────┤
│ local       │ Development    │ Manual       │ Developer  │
│ dev         │ Integration    │ On push      │ Team       │
│ staging     │ Pre-production │ On PR merge  │ Team + QA  │
│ production  │ Live           │ On release   │ Restricted │
└──────────────────────────────────────────────────────────┘
```

```yaml
# Environment configuration pattern
# config/environments/
# ├── base.yaml          # Shared config
# ├── development.yaml   # Dev overrides
# ├── staging.yaml       # Staging overrides
# └── production.yaml    # Prod overrides

# base.yaml
app:
  name: my-service
  log_level: info
  metrics_enabled: true

# production.yaml (overrides)
app:
  log_level: warn
  replicas: 3
  resources:
    cpu: "500m"
    memory: "512Mi"
```

## Release Management

```yaml
# Semantic Versioning + Conventional Commits
# MAJOR.MINOR.PATCH — 2.1.3
# MAJOR: Breaking changes (feat!: or BREAKING CHANGE:)
# MINOR: New features (feat:)
# PATCH: Bug fixes (fix:)

# Automated release with semantic-release
# .releaserc.yml
branches: ['main']
plugins:
  - '@semantic-release/commit-analyzer'
  - '@semantic-release/release-notes-generator'
  - '@semantic-release/changelog'
  - '@semantic-release/npm'
  - '@semantic-release/github'
  - '@semantic-release/git'
```

## Incident Response

```markdown
## Incident Severity Levels

| Level | Description                    | Response Time | Example                    |
|-------|--------------------------------|---------------|----------------------------|
| SEV1  | Complete service outage        | 15 minutes    | Production down            |
| SEV2  | Major feature degraded         | 30 minutes    | Payments failing           |
| SEV3  | Minor feature impacted         | 4 hours       | Search slow                |
| SEV4  | Cosmetic or low-impact issue   | Next sprint   | UI alignment issue         |

## Incident Workflow
1. Detect → Alert fires or user report
2. Triage → Assign severity, notify on-call
3. Mitigate → Restore service (rollback, scale, failover)
4. Resolve → Fix root cause
5. Postmortem → Blameless review, action items
```

## Graceful Shutdown Pattern

```typescript
// Kubernetes-compatible graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown');

  // 1. Stop accepting new requests
  server.close();

  // 2. Wait for in-flight requests (match terminationGracePeriodSeconds)
  await drainConnections(25_000); // 25s drain, 30s K8s grace

  // 3. Close backing service connections
  await database.disconnect();
  await redis.quit();
  await messageQueue.close();

  // 4. Exit cleanly
  process.exit(0);
});
```

## References

- [DevOps Patterns](references/devops-patterns.md) — Platform engineering, developer experience, internal developer portals, golden paths, self-service infrastructure.
- [Incident Management](references/incident-management.md) — Runbook templates, postmortem format, on-call rotation, escalation policies, chaos engineering.
