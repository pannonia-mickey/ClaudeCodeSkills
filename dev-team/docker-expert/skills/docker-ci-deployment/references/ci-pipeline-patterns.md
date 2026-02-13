# CI/CD Pipeline Patterns for Docker

Comprehensive reference covering GitHub Actions and GitLab CI complete patterns, build optimization strategies, security scanning integration, and deployment patterns.

---

## GitHub Actions Complete Patterns

### Production Workflow with OIDC

Use OIDC for keyless authentication instead of storing long-lived credentials:

```yaml
name: Production Docker Pipeline

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
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  build-and-push:
    needs: lint
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write        # OIDC token for keyless signing
      security-events: write  # Upload SARIF results

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tags: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        if: github.event_name != 'pull_request'
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=
            type=ref,event=branch
            type=raw,value=latest,enable=${{ github.ref == format('refs/heads/{0}', 'main') }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true

      - name: Sign image with cosign
        if: github.event_name != 'pull_request'
        uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
        if: github.event_name != 'pull_request'

  scan:
    needs: build-and-push
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      security-events: write

    steps:
      - name: Run Trivy scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.build-and-push.outputs.image-digest }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH

      - name: Upload scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
```

### Matrix Builds for Multiple Dockerfiles

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - service: api
            dockerfile: services/api/Dockerfile
            context: services/api
          - service: worker
            dockerfile: services/worker/Dockerfile
            context: services/worker
          - service: frontend
            dockerfile: services/frontend/Dockerfile
            context: services/frontend

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          file: ${{ matrix.dockerfile }}
          tags: ghcr.io/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }}
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,scope=${{ matrix.service }},mode=max
```

### Reusable Workflow

```yaml
# .github/workflows/docker-build.yml (reusable)
name: Docker Build (Reusable)

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
      context:
        required: false
        type: string
        default: .
      platforms:
        required: false
        type: string
        default: linux/amd64

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}/${{ inputs.image-name }}

      - uses: docker/build-push-action@v6
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha,scope=${{ inputs.image-name }}
          cache-to: type=gha,scope=${{ inputs.image-name }},mode=max
```

Caller workflow:

```yaml
# .github/workflows/ci.yml
jobs:
  build-api:
    uses: ./.github/workflows/docker-build.yml
    with:
      image-name: api
      context: services/api
      platforms: linux/amd64,linux/arm64

  build-worker:
    uses: ./.github/workflows/docker-build.yml
    with:
      image-name: worker
      context: services/worker
```

---

## GitLab CI Docker Patterns

### Docker-in-Docker

```yaml
# .gitlab-ci.yml
stages:
  - build
  - scan
  - push

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build --cache-from $CI_REGISTRY_IMAGE:latest -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

scan:
  stage: scan
  image:
    name: aquasec/trivy
    entrypoint: [""]
  script:
    - trivy image --exit-code 1 --severity CRITICAL $IMAGE_TAG

push-latest:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  only:
    - main
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
```

### Kaniko (Rootless Builds)

Build without Docker daemon — no privileged mode required:

```yaml
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.0-debug
    entrypoint: [""]
  script:
    - |
      /kaniko/executor \
        --context $CI_PROJECT_DIR \
        --dockerfile $CI_PROJECT_DIR/Dockerfile \
        --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        --cache=true \
        --cache-repo=$CI_REGISTRY_IMAGE/cache
```

---

## Build Optimization in CI

### Selective Builds in Monorepos

Only rebuild services whose files changed:

```yaml
# GitHub Actions
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.changes.outputs.api }}
      frontend: ${{ steps.changes.outputs.frontend }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: changes
        with:
          filters: |
            api:
              - 'services/api/**'
              - 'shared/**'
            frontend:
              - 'services/frontend/**'

  build-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    uses: ./.github/workflows/docker-build.yml
    with:
      image-name: api
      context: services/api

  build-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    uses: ./.github/workflows/docker-build.yml
    with:
      image-name: frontend
      context: services/frontend
```

### Layer Ordering for CI Cache

Structure your Dockerfile so frequently-changing layers are last:

```dockerfile
# Rarely changes → cached in CI
FROM python:3.12-slim
RUN apt-get update && apt-get install -y --no-install-recommends libpq5 && rm -rf /var/lib/apt/lists/*

# Changes when deps change → cached between most builds
COPY requirements.txt .
RUN pip install -r requirements.txt

# Changes every commit → never cached
COPY . .
```

### Build Concurrency

```yaml
# GitHub Actions: cancel previous builds on the same branch
concurrency:
  group: docker-${{ github.ref }}
  cancel-in-progress: true
```

---

## Security Scanning in CI

### Trivy with GitHub Security Integration

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1  # Fail the pipeline

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v3
  if: always()  # Upload even if scan finds vulnerabilities
  with:
    sarif_file: trivy-results.sarif
```

### Grype Alternative

```yaml
- name: Scan with Grype
  uses: anchore/scan-action@v4
  with:
    image: myapp:${{ github.sha }}
    fail-build: true
    severity-cutoff: high
    output-format: sarif

- name: Upload Grype results
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: ${{ steps.scan.outputs.sarif }}
```

### Scan Gate Pattern

Block deployment if vulnerabilities exceed thresholds:

```yaml
scan:
  runs-on: ubuntu-latest
  steps:
    - name: Scan image
      run: |
        trivy image --exit-code 0 --severity CRITICAL --format json \
            myapp:${{ github.sha }} > scan-results.json

        CRITICAL_COUNT=$(jq '[.Results[].Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length' scan-results.json)

        if [ "$CRITICAL_COUNT" -gt 0 ]; then
          echo "::error::Found $CRITICAL_COUNT critical vulnerabilities"
          exit 1
        fi
```

---

## Deployment Patterns

### Rolling Update with Docker Compose

```bash
#!/bin/bash
# deploy.sh — Zero-downtime rolling deployment

set -euo pipefail

# Pull new images
docker compose pull

# Restart services one at a time
docker compose up -d --no-deps --build app

# Wait for health check
echo "Waiting for health check..."
for i in $(seq 1 30); do
    if docker compose exec app curl -sf http://localhost:8000/health/ > /dev/null 2>&1; then
        echo "Health check passed!"
        exit 0
    fi
    sleep 2
done

echo "Health check failed — rolling back"
docker compose rollback app 2>/dev/null || docker compose up -d --no-deps app
exit 1
```

### Blue-Green Deployment

```yaml
# compose.blue-green.yaml
services:
  blue:
    image: myapp:${BLUE_TAG}
    expose: ["8000"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]

  green:
    image: myapp:${GREEN_TAG}
    expose: ["8000"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]

  nginx:
    image: nginx:alpine
    ports: ["80:80"]
    volumes:
      - ./nginx-${ACTIVE_COLOR}.conf:/etc/nginx/conf.d/default.conf:ro
```

### Watchtower (Automated Updates)

Automatically pull and restart containers when new images are available:

```yaml
services:
  watchtower:
    image: containrrr/watchtower
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      WATCHTOWER_CLEANUP: "true"
      WATCHTOWER_POLL_INTERVAL: 300  # Check every 5 minutes
      WATCHTOWER_LABEL_ENABLE: "true"  # Only update labeled containers

  app:
    image: ghcr.io/org/myapp:latest
    labels:
      - "com.centurylinklabs.watchtower.enable=true"
```

### Webhook-Triggered Deployment

```yaml
# GitHub Actions: trigger deployment after successful build
deploy:
  needs: build-and-push
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to production
      run: |
        curl -X POST "${{ secrets.DEPLOY_WEBHOOK_URL }}" \
            -H "Authorization: Bearer ${{ secrets.DEPLOY_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"image": "${{ needs.build-and-push.outputs.image-tags }}", "sha": "${{ github.sha }}"}'
```

### Health Check-Based Rollback

```bash
#!/bin/bash
# deploy-with-rollback.sh

OLD_IMAGE=$(docker inspect --format '{{.Config.Image}}' myapp-web)
NEW_IMAGE=$1

echo "Deploying $NEW_IMAGE (current: $OLD_IMAGE)"

# Deploy new version
docker compose pull app
TAG=$NEW_IMAGE docker compose up -d app

# Health check loop
HEALTHY=false
for i in $(seq 1 30); do
    STATUS=$(docker inspect --format '{{.State.Health.Status}}' myapp-web 2>/dev/null || echo "unknown")
    if [ "$STATUS" = "healthy" ]; then
        HEALTHY=true
        break
    fi
    echo "Waiting for health check... ($i/30, status: $STATUS)"
    sleep 2
done

if [ "$HEALTHY" = "true" ]; then
    echo "Deployment successful!"
    # Remove old image
    docker image prune -f
else
    echo "Deployment failed — rolling back to $OLD_IMAGE"
    TAG=$OLD_IMAGE docker compose up -d app
    exit 1
fi
```
