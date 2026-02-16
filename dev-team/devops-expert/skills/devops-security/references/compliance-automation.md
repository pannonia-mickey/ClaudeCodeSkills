# Compliance Automation

## CIS Benchmarks

```yaml
# Kubernetes CIS benchmark scan with kube-bench
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
spec:
  template:
    spec:
      hostPID: true
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:latest
          command: ["kube-bench", "run", "--targets", "node,policies"]
          volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
      volumes:
        - name: var-lib-kubelet
          hostPath: { path: /var/lib/kubelet }
        - name: etc-kubernetes
          hostPath: { path: /etc/kubernetes }
      restartPolicy: Never

# CI integration
- name: CIS Docker Benchmark
  run: docker run --net host --pid host -v /var/run/docker.sock:/var/run/docker.sock docker/docker-bench-security
```

## SOC 2 Controls Mapping

```yaml
# Map CI/CD controls to SOC 2 trust service criteria
controls:
  CC6.1_Logical_Access:
    - control: "Branch protection requires PR approval"
      evidence: "GitHub branch protection rules screenshot"
      automated_check: |
        gh api repos/{owner}/{repo}/branches/main/protection | jq '.required_pull_request_reviews'

  CC7.1_System_Monitoring:
    - control: "All deployments are logged and auditable"
      evidence: "GitHub Actions audit log"
      automated_check: |
        gh api orgs/{org}/audit-log --jq '.[].action' | grep -c "workflows"

  CC8.1_Change_Management:
    - control: "All changes pass automated tests before deployment"
      evidence: "Required status checks on main branch"
      automated_check: |
        gh api repos/{owner}/{repo}/branches/main/protection/required_status_checks

  CC6.6_Security_Events:
    - control: "Security scanning runs on every commit"
      evidence: "Semgrep/Snyk workflow runs"
      automated_check: |
        gh run list --workflow=security.yml --limit=30 --json conclusion | jq '[.[] | select(.conclusion=="success")] | length'
```

## Audit Logging

```typescript
// Comprehensive audit trail for compliance
interface AuditEvent {
  timestamp: string;
  actor: { id: string; email: string; ip: string };
  action: string;
  resource: { type: string; id: string };
  changes?: Record<string, { from: unknown; to: unknown }>;
  result: 'success' | 'failure' | 'denied';
  metadata?: Record<string, unknown>;
}

class AuditLogger {
  async log(event: AuditEvent): Promise<void> {
    // Write to immutable audit log (append-only)
    await this.auditStore.append({
      ...event,
      timestamp: new Date().toISOString(),
      // Tamper-evident: include hash of previous entry
      previousHash: await this.getLastHash(),
    });
  }
}

// Usage
await auditLogger.log({
  timestamp: new Date().toISOString(),
  actor: { id: user.id, email: user.email, ip: req.ip },
  action: 'user.role.update',
  resource: { type: 'user', id: targetUser.id },
  changes: { role: { from: 'user', to: 'admin' } },
  result: 'success',
});
```

## Evidence Collection Pipeline

```yaml
# Automated compliance evidence collection
name: Compliance Evidence
on:
  schedule:
    - cron: '0 0 1 * *'  # Monthly

jobs:
  collect-evidence:
    runs-on: ubuntu-latest
    steps:
      - name: Branch protection evidence
        run: |
          gh api repos/${{ github.repository }}/branches/main/protection \
            > evidence/branch-protection.json

      - name: Dependency audit evidence
        run: |
          npm audit --json > evidence/npm-audit.json
          snyk test --json > evidence/snyk-report.json

      - name: Access review evidence
        run: |
          gh api orgs/${{ github.repository_owner }}/members \
            --jq '.[] | {login, role: .role_name}' > evidence/access-review.json

      - name: Security scan evidence
        run: |
          gh run list --workflow=security.yml --limit=30 --json conclusion,createdAt \
            > evidence/security-scans.json

      - name: Upload evidence
        uses: actions/upload-artifact@v4
        with:
          name: compliance-evidence-${{ github.run_id }}
          path: evidence/
          retention-days: 365
```

## Compliance Checklist

```markdown
## Monthly Compliance Review

### Access Control
- [ ] Review and audit team access permissions
- [ ] Verify MFA enforcement for all team members
- [ ] Review and rotate service account credentials
- [ ] Audit third-party app integrations

### Change Management
- [ ] Verify all production changes went through PR review
- [ ] Confirm no direct commits to main/production branches
- [ ] Review emergency change procedures used (if any)
- [ ] Audit deployment logs for unauthorized deployments

### Security
- [ ] Review security scan results (SAST, DAST, dependency)
- [ ] Verify no critical/high vulnerabilities in production
- [ ] Review and action Dependabot/Renovate alerts
- [ ] Confirm secret rotation schedule is maintained

### Monitoring
- [ ] Review incident response times vs SLA
- [ ] Verify alert configurations are current
- [ ] Review and update runbooks
- [ ] Confirm backup and restore tests completed
```
