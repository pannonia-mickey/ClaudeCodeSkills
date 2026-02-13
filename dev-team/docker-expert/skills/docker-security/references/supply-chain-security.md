# Container Supply Chain Security

Comprehensive reference covering base image provenance, SHA pinning, SBOM generation, build reproducibility, registry security, policy enforcement with Hadolint, and image signing with cosign.

---

## Base Image Provenance

### Pinning Images by SHA256 Digest

Tags are mutable — the image behind `python:3.12-slim` can change at any time. Pin to SHA256 digest for reproducible builds:

```dockerfile
# Mutable tag (different image tomorrow)
FROM python:3.12-slim

# Immutable digest (same image forever)
FROM python:3.12-slim@sha256:2b0079146a74e9e4d3e7ea3398d51c8d2f2c2e2a4a5f3d4c3b2a1908070605
```

Find the digest:

```bash
# From Docker Hub
docker manifest inspect python:3.12-slim | jq -r '.config.digest'

# Or pull and inspect
docker pull python:3.12-slim
docker inspect python:3.12-slim --format '{{index .RepoDigests 0}}'
```

### Verified Publisher Images

Prefer images from verified sources:

| Source | Trust Level | Examples |
|--------|------------|---------|
| Docker Official Images | High | `python`, `node`, `postgres`, `nginx` |
| Verified Publisher | High | `bitnami/postgresql`, `grafana/grafana` |
| Chainguard | Very High | `cgr.dev/chainguard/python` |
| Google Distroless | High | `gcr.io/distroless/python3` |
| Community | Varies | Third-party images |

### Automated Base Image Updates

Use Dependabot or Renovate to automatically propose base image updates:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: docker
    directory: "/"
    schedule:
      interval: weekly
    reviewers:
      - "team-devops"
```

```json
// renovate.json
{
  "extends": ["config:recommended"],
  "docker": {
    "enabled": true,
    "pinDigests": true
  }
}
```

---

## SBOM Generation

### Syft

Generate a Software Bill of Materials listing all packages in an image:

```bash
# Generate SBOM in SPDX format
syft myapp:latest -o spdx-json > sbom.spdx.json

# Generate SBOM in CycloneDX format
syft myapp:latest -o cyclonedx-json > sbom.cdx.json

# Generate from a Dockerfile
syft dir:. -o spdx-json > sbom.spdx.json
```

### Docker SBOM Command

```bash
# Built-in Docker CLI (requires Docker Scout)
docker sbom myapp:latest
docker sbom myapp:latest --format spdx-json > sbom.json
```

### Attaching SBOMs to Images

Use ORAS (OCI Registry As Storage) to attach SBOMs alongside images:

```bash
# Attach SBOM to image in registry
oras attach ghcr.io/org/myapp:v1.0.0 \
    --artifact-type application/spdx+json \
    sbom.spdx.json

# Discover attached artifacts
oras discover ghcr.io/org/myapp:v1.0.0
```

### CI Integration

```yaml
# GitHub Actions
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.spdx.json
```

---

## Build Reproducibility

### Deterministic Builds

Ensure the same source code produces the same image:

1. **Pin all dependency versions** — use lock files
2. **Pin base image by digest** — immutable reference
3. **Set SOURCE_DATE_EPOCH** — reproducible timestamps
4. **Disable build randomness** — no random passwords in layers

```dockerfile
# syntax=docker/dockerfile:1
ARG SOURCE_DATE_EPOCH=0

FROM python:3.12-slim@sha256:abc123...

# Use lock file for exact versions
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-compile -r requirements.txt

COPY . .
```

### Lock Files by Ecosystem

| Language | Lock File | Generate Command |
|----------|-----------|-----------------|
| Python (pip) | `requirements.txt` (with ==) | `pip freeze > requirements.txt` |
| Python (Poetry) | `poetry.lock` | `poetry lock` |
| Python (uv) | `uv.lock` | `uv lock` |
| Node.js | `package-lock.json` | `npm install` |
| Go | `go.sum` | `go mod tidy` |
| Rust | `Cargo.lock` | `cargo generate-lockfile` |
| .NET | `packages.lock.json` | `dotnet restore --locked-mode` |

### SOURCE_DATE_EPOCH

Set a fixed timestamp for reproducible file modification times:

```dockerfile
ARG SOURCE_DATE_EPOCH
ENV SOURCE_DATE_EPOCH=${SOURCE_DATE_EPOCH:-0}
```

```bash
# Build with fixed timestamp
docker build --build-arg SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) .
```

---

## Registry Security

### Private Registry Authentication

```bash
# Docker Hub
docker login -u username

# GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# AWS ECR
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

# Azure Container Registry
az acr login --name myregistry

# Google Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev
```

### Image Immutability

Prevent tag overwriting to ensure deployed images cannot be silently replaced:

**AWS ECR:**
```bash
aws ecr put-image-tag-mutability \
    --repository-name myapp \
    --image-tag-mutability IMMUTABLE
```

**Harbor:**
```bash
# Enable tag immutability via Harbor UI or API
curl -X PUT "https://harbor.example.com/api/v2.0/projects/myproject" \
    -H "Content-Type: application/json" \
    -d '{"metadata": {"enable_content_trust": "true"}}'
```

### Garbage Collection

Remove untagged and unreferenced images to save storage:

**Docker Registry:**
```bash
# Run garbage collection on self-hosted registry
docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

**ECR Lifecycle Policy:**
```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPatternList": ["v*"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## Policy Enforcement

### Hadolint

Lint Dockerfiles against best practices:

```bash
# Lint a Dockerfile
hadolint Dockerfile

# Ignore specific rules
hadolint --ignore DL3008 --ignore DL3009 Dockerfile

# Use a config file
hadolint --config .hadolint.yaml Dockerfile
```

Configuration:

```yaml
# .hadolint.yaml
ignored:
  - DL3008  # Pin versions in apt-get install
  - DL3013  # Pin versions in pip install

override:
  error:
    - DL3000  # Use absolute WORKDIR
    - DL3001  # For some bash commands
  warning:
    - DL3042  # Avoid cache dir in pip install
  info:
    - DL3032  # yum clean all

trustedRegistries:
  - docker.io
  - gcr.io
  - ghcr.io
```

### GitHub Actions Integration

```yaml
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: warning
```

### Dockle

Audit container images against CIS Docker Benchmark:

```bash
# Scan an image
dockle myapp:latest

# Fail on warnings
dockle --exit-code 1 --exit-level warn myapp:latest
```

Dockle checks:
- Running as non-root user
- HEALTHCHECK presence
- No sensitive files (credentials, keys)
- Use of COPY over ADD
- Content trust enabled

---

## Image Signing with Cosign

### Signing Images

```bash
# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign --key cosign.key ghcr.io/org/myapp:v1.0.0

# Verify signature
cosign verify --key cosign.pub ghcr.io/org/myapp:v1.0.0
```

### Keyless Signing (Sigstore)

Sign without managing keys — uses OIDC identity from CI providers:

```bash
# Keyless signing (interactive — opens browser for OIDC)
cosign sign ghcr.io/org/myapp:v1.0.0

# Verify keyless signature
cosign verify \
    --certificate-identity=user@example.com \
    --certificate-oidc-issuer=https://accounts.google.com \
    ghcr.io/org/myapp:v1.0.0
```

### GitHub Actions Integration

```yaml
- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Sign image with GitHub OIDC
  run: cosign sign --yes ghcr.io/${{ github.repository }}:${{ github.sha }}
  env:
    COSIGN_EXPERIMENTAL: 1

- name: Verify signature
  run: |
    cosign verify \
        --certificate-identity=https://github.com/${{ github.repository }}/.github/workflows/build.yml@refs/heads/main \
        --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
        ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Admission Control

Enforce that only signed images can be deployed:

```yaml
# Kubernetes policy (Kyverno)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-signature
      match:
        resources:
          kinds:
            - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/org/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/org/*"
                    issuer: "https://token.actions.githubusercontent.com"
```

---

## Dependency Scanning

### Multi-Layer Scanning Strategy

Scan both OS-level packages and application dependencies:

```bash
# OS packages (Trivy)
trivy image --vuln-type os myapp:latest

# Application dependencies (Trivy)
trivy image --vuln-type library myapp:latest

# Both (default)
trivy image myapp:latest
```

### Scanning Application Lock Files

```bash
# Scan without building an image
trivy fs --security-checks vuln .

# Scan specific lock files
trivy fs --scanners vuln requirements.txt
trivy fs --scanners vuln package-lock.json
```

### Continuous Monitoring

Set up scheduled scans to catch newly disclosed vulnerabilities:

```yaml
# GitHub Actions scheduled scan
name: Scheduled Security Scan
on:
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6 AM

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - name: Scan latest production image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/org/myapp:latest
          format: sarif
          output: trivy-results.sarif

      - name: Upload results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```
