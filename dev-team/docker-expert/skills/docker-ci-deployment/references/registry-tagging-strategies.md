# Container Registry and Tagging Strategies

Comprehensive reference covering registry types and configuration, tagging conventions, lifecycle management, multi-architecture manifests, and image promotion workflows.

---

## Registry Types and Configuration

### Docker Hub

```bash
# Login
docker login -u username

# Push
docker tag myapp:latest docker.io/username/myapp:v1.0.0
docker push docker.io/username/myapp:v1.0.0
```

Free tier: 1 private repository, unlimited public. Rate limits for anonymous pulls (100/6h).

### GitHub Container Registry (GHCR)

```bash
# Login with personal access token
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Push (image visibility follows repo settings)
docker tag myapp:latest ghcr.io/org/myapp:v1.0.0
docker push ghcr.io/org/myapp:v1.0.0
```

Advantages: integrated with GitHub Actions (no extra secrets for GITHUB_TOKEN), free for public images, included storage with GitHub plans.

### AWS Elastic Container Registry (ECR)

```bash
# Login (token valid for 12 hours)
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp --image-tag-mutability IMMUTABLE

# Push
docker tag myapp:latest 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0
```

Features: lifecycle policies, image scanning, cross-region replication, IAM-based access control.

### Azure Container Registry (ACR)

```bash
# Login
az acr login --name myregistry

# Push
docker tag myapp:latest myregistry.azurecr.io/myapp:v1.0.0
docker push myregistry.azurecr.io/myapp:v1.0.0

# Build in the cloud (no local Docker needed)
az acr build --registry myregistry --image myapp:v1.0.0 .
```

### Google Artifact Registry (GAR)

```bash
# Configure Docker to use gcloud credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Push
docker tag myapp:latest us-central1-docker.pkg.dev/my-project/my-repo/myapp:v1.0.0
docker push us-central1-docker.pkg.dev/my-project/my-repo/myapp:v1.0.0
```

### Self-Hosted: Harbor

```bash
# Login
docker login harbor.example.com

# Push
docker tag myapp:latest harbor.example.com/myproject/myapp:v1.0.0
docker push harbor.example.com/myproject/myapp:v1.0.0
```

Features: vulnerability scanning, content trust, replication, RBAC, audit logs, tag retention policies.

---

## Tagging Conventions

### Semantic Versioning with Cascading Tags

When releasing version `v1.2.3`, create multiple tags:

```bash
# All these tags point to the same image digest
docker tag myapp:latest myapp:1.2.3   # Exact version (immutable)
docker tag myapp:latest myapp:1.2     # Minor version (updated on patches)
docker tag myapp:latest myapp:1       # Major version (updated on minor/patch)
docker tag myapp:latest myapp:latest  # Most recent release
```

This allows consumers to choose their update granularity:
- `myapp:1.2.3` — pinned, no automatic updates
- `myapp:1.2` — automatic patch updates
- `myapp:1` — automatic minor + patch updates
- `myapp:latest` — always the newest

### docker/metadata-action Full Configuration

```yaml
- uses: docker/metadata-action@v5
  id: meta
  with:
    images: |
      ghcr.io/${{ github.repository }}
    tags: |
      # Semantic versioning
      type=semver,pattern={{version}}
      type=semver,pattern={{major}}.{{minor}}
      type=semver,pattern={{major}}

      # Git SHA (short, immutable identifier)
      type=sha,prefix=,format=short

      # Branch name for non-tag pushes
      type=ref,event=branch

      # PR number
      type=ref,event=pr,prefix=pr-

      # Latest on default branch
      type=raw,value=latest,enable=${{ github.ref == format('refs/heads/{0}', 'main') }}

      # Date-based tag
      type=raw,value={{date 'YYYYMMDD'}},enable=${{ github.ref == format('refs/heads/{0}', 'main') }}

    labels: |
      org.opencontainers.image.title=My Application
      org.opencontainers.image.description=Production web service
      org.opencontainers.image.vendor=My Company
```

### When to Use `latest`

**Do use `latest`**:
- For development and testing convenience
- As a pointer to the most recent stable release
- In tutorials and getting-started guides

**Do NOT use `latest`**:
- In production deployments (non-reproducible)
- In Dockerfiles (FROM myapp:latest)
- In CI/CD pipelines
- When immutability matters

### Environment-Based Tags

```bash
myapp:dev        # Development environment
myapp:staging    # Staging environment
myapp:production # Production environment
```

These are mutable and updated by CI/CD as images are promoted through environments.

---

## Image Lifecycle Management

### ECR Lifecycle Policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire untagged images after 3 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 3
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 5 dev images",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["dev-*"],
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 3,
      "description": "Keep only last 20 SHA-tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["sha-*"],
        "countType": "imageCountMoreThan",
        "countNumber": 20
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 10,
      "description": "Keep all semver-tagged images for 1 year",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["[0-9]*"],
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 365
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

Apply:

```bash
aws ecr put-lifecycle-policy \
    --repository-name myapp \
    --lifecycle-policy-text file://lifecycle-policy.json
```

### GHCR Cleanup with GitHub Actions

```yaml
name: Cleanup old container images

on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  cleanup:
    runs-on: ubuntu-latest
    permissions:
      packages: write

    steps:
      - uses: actions/delete-package-versions@v5
        with:
          package-name: myapp
          package-type: container
          min-versions-to-keep: 10
          delete-only-untagged-versions: true
```

### Harbor Tag Retention

Configure retention rules in Harbor UI or API:

```bash
# Keep the 10 most recent tagged images
# Delete images not pulled in the last 30 days
# Always keep images matching 'v*' pattern
```

---

## Multi-Architecture Manifests

### Creating a Manifest List

```bash
# Build platform-specific images
docker build --platform linux/amd64 -t myapp:v1.0.0-amd64 .
docker build --platform linux/arm64 -t myapp:v1.0.0-arm64 .

# Push individual images
docker push myapp:v1.0.0-amd64
docker push myapp:v1.0.0-arm64

# Create and push manifest list
docker manifest create myapp:v1.0.0 \
    myapp:v1.0.0-amd64 \
    myapp:v1.0.0-arm64

docker manifest push myapp:v1.0.0
```

### Automatic with buildx

```bash
# buildx creates manifests automatically
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --tag myapp:v1.0.0 \
    --push .
```

### Inspecting Manifests

```bash
# Show manifest details
docker manifest inspect myapp:v1.0.0

# Output includes platform information
{
  "manifests": [
    {
      "digest": "sha256:abc...",
      "platform": { "architecture": "amd64", "os": "linux" }
    },
    {
      "digest": "sha256:def...",
      "platform": { "architecture": "arm64", "os": "linux" }
    }
  ]
}

# Check which platform image will be pulled
docker manifest inspect --verbose myapp:v1.0.0
```

### Verifying Platform Support

```bash
# Check what platform an image supports
docker buildx imagetools inspect myapp:v1.0.0
```

---

## Image Promotion Workflow

### Dev → Staging → Production

```
┌─────────┐     ┌──────────┐     ┌─────────────┐
│   Dev   │────→│  Staging │────→│  Production │
│ :sha-abc│     │ :staging │     │ :v1.2.3     │
└─────────┘     └──────────┘     └─────────────┘
     │               │                  │
  CI build      QA approval       Release tag
  Auto-push     Manual gate       Manual gate
```

### Promoting with crane

`crane` is a tool for managing container images without Docker daemon:

```bash
# Install crane
go install github.com/google/go-containerregistry/cmd/crane@latest

# Promote from dev to staging (re-tag without pulling/pushing full image)
crane tag ghcr.io/org/myapp:sha-abc123 staging

# Promote from staging to production
crane tag ghcr.io/org/myapp:staging v1.2.3
crane tag ghcr.io/org/myapp:staging 1.2
crane tag ghcr.io/org/myapp:staging 1
crane tag ghcr.io/org/myapp:staging latest
```

### Promoting with skopeo

`skopeo` copies images between registries without a daemon:

```bash
# Copy between registries
skopeo copy \
    docker://ghcr.io/org/myapp:staging \
    docker://prod-registry.example.com/myapp:v1.2.3

# Copy within same registry (re-tag)
skopeo copy \
    docker://ghcr.io/org/myapp:sha-abc123 \
    docker://ghcr.io/org/myapp:staging
```

### CI/CD Promotion Pipeline

```yaml
name: Promote Image

on:
  workflow_dispatch:
    inputs:
      source-tag:
        description: 'Source image tag to promote'
        required: true
      target-env:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  promote:
    runs-on: ubuntu-latest
    permissions:
      packages: write

    steps:
      - name: Install crane
        uses: imjasonh/setup-crane@v0.3

      - name: Login to registry
        run: echo ${{ secrets.GITHUB_TOKEN }} | crane auth login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Verify source image exists
        run: crane manifest ghcr.io/${{ github.repository }}:${{ inputs.source-tag }}

      - name: Scan before promotion
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ inputs.source-tag }}
          exit-code: 1
          severity: CRITICAL

      - name: Promote image
        run: |
          crane tag ghcr.io/${{ github.repository }}:${{ inputs.source-tag }} ${{ inputs.target-env }}
          echo "Promoted ${{ inputs.source-tag }} → ${{ inputs.target-env }}"

      - name: Create deployment record
        run: |
          echo "## Promotion Record" >> $GITHUB_STEP_SUMMARY
          echo "- **Source:** \`${{ inputs.source-tag }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- **Target:** \`${{ inputs.target-env }}\`" >> $GITHUB_STEP_SUMMARY
          echo "- **Time:** \`$(date -u)\`" >> $GITHUB_STEP_SUMMARY
          echo "- **Actor:** \`${{ github.actor }}\`" >> $GITHUB_STEP_SUMMARY
```

### Promotion Gates

Enforce quality checks before promotion:

| Gate | Dev → Staging | Staging → Production |
|------|---------------|---------------------|
| Build succeeds | Required | N/A |
| Unit tests pass | Required | N/A |
| Vulnerability scan | Warning only | Required (no criticals) |
| Integration tests | Optional | Required |
| Load tests | Not required | Recommended |
| Manual approval | Not required | Required |
| SBOM generated | Required | Required |
| Image signed | Required | Required + verified |
