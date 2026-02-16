# Deployment Patterns

## Progressive Delivery with Argo Rollouts

```yaml
# Analysis template — automatic rollback on metric degradation
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
    - name: success-rate
      interval: 60s
      successCondition: result[0] >= 0.99
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"2.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))

---
# Rollout with automatic analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: { duration: 2m }
        - analysis:
            templates: [{ templateName: success-rate }]
            args:
              - name: service-name
                value: my-service
        - setWeight: 50
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: success-rate }]
      # Automatic rollback on analysis failure
      abortScaleDownDelaySeconds: 30
```

## Database Migrations in CI

```yaml
# Safe migration pattern — separate from app deployment
jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      # Validate migrations are safe
      - name: Check migration safety
        run: |
          # Reject migrations with DROP TABLE, DROP COLUMN in production
          if grep -i "DROP TABLE\|DROP COLUMN" migrations/*.sql; then
            echo "Destructive migration detected — requires manual review"
            exit 1
          fi

      # Run migrations with timeout
      - name: Apply migrations
        run: npx prisma migrate deploy
        timeout-minutes: 10
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

      # Verify schema state
      - name: Verify migration
        run: npx prisma migrate status
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}

  deploy:
    needs: migrate
    # App deployment only after successful migration
```

## Smoke Tests Post-Deploy

```yaml
# Verify deployment health before routing traffic
deploy:
  steps:
    - name: Deploy
      run: ./scripts/deploy.sh ${{ inputs.environment }}

    - name: Smoke tests
      run: |
        BASE_URL="${{ inputs.base-url }}"

        # Health check
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/health")
        [ "$STATUS" = "200" ] || { echo "Health check failed: $STATUS"; exit 1; }

        # Version check
        VERSION=$(curl -s "$BASE_URL/health" | jq -r '.version')
        [ "$VERSION" = "${{ inputs.version }}" ] || { echo "Version mismatch: $VERSION"; exit 1; }

        # Critical path check
        STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/api/products")
        [ "$STATUS" = "200" ] || { echo "Products API failed: $STATUS"; exit 1; }

    - name: Rollback on failure
      if: failure()
      run: ./scripts/rollback.sh ${{ inputs.environment }}
```

## Feature Flag Deployment

```typescript
// Deploy code behind feature flag — zero-risk deployment
// 1. Deploy code with flag OFF → no user impact
// 2. Enable flag for internal users → internal testing
// 3. Enable flag for 5% of users → canary
// 4. Enable flag for 100% → full rollout
// 5. Remove flag and old code → cleanup

// LaunchDarkly / Unleash / Flagsmith integration
const newFeature = featureFlags.isEnabled('new-checkout', {
  userId: user.id,
  userSegment: user.plan, // target specific segments
});

if (newFeature) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

## Rollback Automation

```yaml
# Automatic rollback on health check failure
deploy:
  steps:
    - name: Record current version
      id: current
      run: echo "version=$(kubectl get deployment my-app -o jsonpath='{.metadata.annotations.version}')" >> $GITHUB_OUTPUT

    - name: Deploy new version
      run: |
        kubectl set image deployment/my-app app=${{ env.IMAGE }}
        kubectl annotate deployment/my-app version=${{ github.sha }} --overwrite

    - name: Wait for rollout
      id: rollout
      run: kubectl rollout status deployment/my-app --timeout=300s
      continue-on-error: true

    - name: Rollback on failure
      if: steps.rollout.outcome == 'failure'
      run: |
        echo "Deployment failed, rolling back to ${{ steps.current.outputs.version }}"
        kubectl rollout undo deployment/my-app
        kubectl rollout status deployment/my-app --timeout=120s
        exit 1  # Fail the workflow
```

## Multi-Region Deployment

```yaml
# Sequential multi-region deployment with validation
jobs:
  deploy-us-east:
    uses: ./.github/workflows/deploy-region.yml
    with:
      region: us-east-1
      weight: 10  # Start with low traffic

  validate-us-east:
    needs: deploy-us-east
    runs-on: ubuntu-latest
    steps:
      - run: ./scripts/validate-region.sh us-east-1

  deploy-eu-west:
    needs: validate-us-east
    uses: ./.github/workflows/deploy-region.yml
    with:
      region: eu-west-1
      weight: 10

  promote-all:
    needs: [validate-us-east, deploy-eu-west]
    runs-on: ubuntu-latest
    environment: production-global
    steps:
      - run: ./scripts/promote-all-regions.sh 100
```
