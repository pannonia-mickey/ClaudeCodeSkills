---
name: CI/CD Pipelines
description: This skill should be used when the user asks about "GitHub Actions", "GitLab CI", "CI/CD pipeline", "deployment pipeline", "build pipeline", "deployment strategy", "blue/green deployment", "canary deployment", "rolling deployment", "pipeline caching", or "artifact management". It covers pipeline design, deployment strategies, and CI/CD best practices.
---

# CI/CD Pipelines

## GitHub Actions — Multi-Stage Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test -- --shard=${{ matrix.shard }}/4

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v4
        with: { name: build-output }
      - run: ./scripts/deploy.sh staging

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/download-artifact@v4
        with: { name: build-output }
      - run: ./scripts/deploy.sh production
```

## Deployment Strategies

```yaml
# Blue/Green Deployment
# Route traffic from blue (current) to green (new) instantly
# Rollback: switch route back to blue

# Canary Deployment (Kubernetes + Argo Rollouts)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-service
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5        # 5% traffic to canary
        - pause: { duration: 5m }
        - analysis:
            templates:
              - templateName: success-rate
        - setWeight: 25       # 25% traffic
        - pause: { duration: 10m }
        - setWeight: 50       # 50% traffic
        - pause: { duration: 10m }
        - setWeight: 100      # Full rollout
      canaryService: my-service-canary
      stableService: my-service-stable

# Rolling Update (Kubernetes default)
# spec.strategy.rollingUpdate:
#   maxSurge: 25%
#   maxUnavailable: 0    # Zero-downtime
```

## Caching Strategies

```yaml
# GitHub Actions — dependency caching
- uses: actions/cache@v4
  with:
    path: |
      node_modules
      ~/.cache/turbo
    key: ${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-

# Docker layer caching in CI
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## GitLab CI Pipeline

```yaml
# .gitlab-ci.yml
stages: [lint, test, build, deploy]

variables:
  NODE_VERSION: "20"

.node-setup:
  image: node:${NODE_VERSION}
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths: [node_modules/]
  before_script:
    - corepack enable && pnpm install --frozen-lockfile

lint:
  extends: .node-setup
  stage: lint
  script: [pnpm lint, pnpm typecheck]

test:
  extends: .node-setup
  stage: test
  parallel: 4
  script: pnpm test -- --shard=$CI_NODE_INDEX/$CI_NODE_TOTAL
  coverage: '/Statements\s+:\s+(\d+\.?\d*)%/'

deploy:staging:
  stage: deploy
  environment: staging
  script: ./scripts/deploy.sh staging
  only: [main]

deploy:production:
  stage: deploy
  environment: production
  script: ./scripts/deploy.sh production
  when: manual
  only: [main]
```

## References

- [Pipeline Patterns](references/pipeline-patterns.md) — Reusable workflows, composite actions, matrix strategies, conditional jobs, monorepo patterns.
- [Deployment Patterns](references/deployment-patterns.md) — Progressive delivery, feature flags, database migrations in CI, smoke tests, rollback automation.
