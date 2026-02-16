# Supply Chain Security

## SBOM Generation

```yaml
# Generate Software Bill of Materials
# Syft — SBOM generator
- name: Generate SBOM
  run: |
    syft packages app:${{ github.sha }} -o spdx-json > sbom.spdx.json
    # Or CycloneDX format
    syft packages app:${{ github.sha }} -o cyclonedx-json > sbom.cdx.json

# Attach SBOM to container image
- run: cosign attach sbom --sbom sbom.spdx.json app:${{ github.sha }}

# Scan SBOM for vulnerabilities
- run: grype sbom:./sbom.spdx.json --fail-on high
```

## Image Signing with Cosign/Sigstore

```yaml
# Sign container images — keyless signing with OIDC
- name: Install cosign
  uses: sigstore/cosign-installer@v3

- name: Sign image
  run: cosign sign --yes app:${{ github.sha }}
  env:
    COSIGN_EXPERIMENTAL: "1"  # Keyless signing with Fulcio

# Verify signature before deployment
- name: Verify image
  run: |
    cosign verify \
      --certificate-identity=https://github.com/myorg/myrepo/.github/workflows/build.yml@refs/heads/main \
      --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
      app:${{ github.sha }}

# Kubernetes admission policy — only allow signed images
# Kyverno policy
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-cosign
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences: ["registry.example.com/*"]
          attestors:
            - entries:
                - keyless:
                    issuer: https://token.actions.githubusercontent.com
                    subject: https://github.com/myorg/*
```

## Provenance Attestation (SLSA)

```yaml
# SLSA Level 3 provenance with GitHub Actions
- name: Generate SLSA provenance
  uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
  with:
    image: app
    digest: ${{ steps.build.outputs.digest }}

# Verify provenance
- run: |
    slsa-verifier verify-image app@sha256:abc123 \
      --source-uri github.com/myorg/myrepo \
      --source-tag v1.0.0
```

## Dependency Pinning

```dockerfile
# Dockerfile — pin everything
# Pin base image by digest
FROM node:20.11.0-slim@sha256:abc123def456...

# Pin package manager
RUN corepack enable && corepack prepare pnpm@9.1.0 --activate

# Lock file ensures reproducible installs
COPY pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
```

```json
// package.json — use exact versions
{
  "dependencies": {
    "express": "4.21.0",
    "prisma": "5.20.0"
  },
  "overrides": {
    "lodash": "4.17.21"
  }
}
```

## Renovate / Dependabot

```json
// renovate.json — automated dependency updates
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended", "security:openssl-sha256"],
  "schedule": ["before 6am on Monday"],
  "labels": ["dependencies"],
  "vulnerabilityAlerts": {
    "enabled": true,
    "labels": ["security"]
  },
  "packageRules": [
    {
      "groupName": "minor and patch",
      "matchUpdateTypes": ["minor", "patch"],
      "automerge": true,
      "automergeType": "pr"
    },
    {
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "reviewers": ["team:platform"]
    },
    {
      "matchPackagePatterns": ["eslint", "prettier", "typescript"],
      "groupName": "dev tooling",
      "automerge": true
    }
  ]
}
```

## Pre-Commit Security Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.86.0
    hooks:
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_checkov

  - repo: https://github.com/hadolint/hadolint
    rev: v2.12.0
    hooks:
      - id: hadolint
```
