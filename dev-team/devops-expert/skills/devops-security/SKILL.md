---
name: DevSecOps
description: This skill should be used when the user asks about "DevSecOps", "secret management", "Vault", "SAST in pipeline", "DAST in pipeline", "dependency scanning in CI", "image scanning", "policy-as-code", "OPA", "supply chain security", or "security pipeline". It covers integrating security into CI/CD pipelines, secret management, and policy enforcement.
---

# DevSecOps

## Security Pipeline Integration

```yaml
# .github/workflows/security.yml — shift-left security
name: Security Pipeline
on: [push, pull_request]

jobs:
  # Stage 1: Static analysis
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: semgrep/semgrep-action@v1
        with:
          config: p/owasp-top-ten p/javascript p/typescript

  # Stage 2: Dependency scanning
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm audit --audit-level=high
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # Stage 3: Secret detection
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2

  # Stage 4: Container scanning
  image-scan:
    needs: [sast, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t app:${{ github.sha }} .
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: 1
```

## HashiCorp Vault Integration

```typescript
// Dynamic secret retrieval
import Vault from 'node-vault';

const vault = Vault({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN, // Or use K8s auth
});

// Read static secret
const { data } = await vault.read('secret/data/my-service/config');
const dbPassword = data.data.db_password;

// Dynamic database credentials (auto-rotating)
const creds = await vault.read('database/creds/my-service-role');
const dbUrl = `postgresql://${creds.data.username}:${creds.data.password}@db:5432/app`;
// Credentials auto-expire after TTL

// Kubernetes auth method
await vault.kubernetesLogin({
  role: 'my-service',
  jwt: fs.readFileSync('/var/run/secrets/kubernetes.io/serviceaccount/token', 'utf8'),
});
```

```yaml
# Vault Agent Injector — transparent secret injection into K8s pods
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-service"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/my-service/config"
        vault.hashicorp.com/agent-inject-template-config: |
          {{- with secret "secret/data/my-service/config" -}}
          DATABASE_URL={{ .Data.data.db_url }}
          API_KEY={{ .Data.data.api_key }}
          {{- end }}
```

## Policy-as-Code with OPA

```rego
# policy/kubernetes.rego
package kubernetes.admission

# Deny containers running as root
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.securityContext.runAsNonRoot
  msg := sprintf("Container '%s' must set runAsNonRoot: true", [container.name])
}

# Require resource limits
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  not container.resources.limits.memory
  msg := sprintf("Container '%s' must set memory limits", [container.name])
}

# Deny privileged containers
deny[msg] {
  input.request.kind.kind == "Pod"
  container := input.request.object.spec.containers[_]
  container.securityContext.privileged
  msg := sprintf("Container '%s' must not be privileged", [container.name])
}
```

## SOPS for Encrypted Secrets in Git

```yaml
# .sops.yaml — encryption rules
creation_rules:
  - path_regex: environments/production/.*
    kms: arn:aws:kms:us-east-1:123456:key/abc-def
  - path_regex: environments/staging/.*
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Encrypt: sops -e secrets.yaml > secrets.enc.yaml
# Decrypt: sops -d secrets.enc.yaml
# Edit:    sops secrets.enc.yaml (decrypts in editor, re-encrypts on save)

# CI usage
- name: Decrypt secrets
  run: |
    sops -d environments/production/secrets.enc.yaml > secrets.yaml
    source secrets.yaml
```

## References

- [Supply Chain Security](references/supply-chain.md) — SBOM generation, image signing (cosign/Sigstore), provenance attestation, dependency pinning, Renovate/Dependabot.
- [Compliance Automation](references/compliance-automation.md) — CIS benchmarks, SOC 2 controls, audit logging, evidence collection, automated compliance checks.
