# Pipeline Patterns

## Reusable Workflows (GitHub Actions)

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: true
        type: string
    secrets:
      DEPLOY_TOKEN:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ inputs.version }}
      - run: ./scripts/deploy.sh ${{ inputs.environment }}
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}

# Caller workflow
# .github/workflows/release.yml
jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: ${{ github.sha }}
    secrets:
      DEPLOY_TOKEN: ${{ secrets.STAGING_TOKEN }}

  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      version: ${{ github.sha }}
    secrets:
      DEPLOY_TOKEN: ${{ secrets.PROD_TOKEN }}
```

## Composite Actions

```yaml
# .github/actions/setup-node-project/action.yml
name: Setup Node Project
description: Install Node.js, pnpm, and project dependencies

inputs:
  node-version:
    description: Node.js version
    default: '20'

runs:
  using: composite
  steps:
    - uses: pnpm/action-setup@v4
      with: { version: 9 }
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: pnpm
    - run: pnpm install --frozen-lockfile
      shell: bash

# Usage in workflows:
# - uses: ./.github/actions/setup-node-project
#   with: { node-version: '20' }
```

## Monorepo Pipeline with Change Detection

```yaml
# Detect changed packages and test only affected
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      api: ${{ steps.filter.outputs.api }}
      web: ${{ steps.filter.outputs.web }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            api: ['packages/api/**', 'packages/shared/**']
            web: ['packages/web/**', 'packages/shared/**']
            shared: ['packages/shared/**']

  test-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm --filter api test

  test-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pnpm --filter web test
```

## Matrix Strategies

```yaml
# Cross-platform, multi-version testing
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18  # Skip old Node on Windows
        include:
          - os: ubuntu-latest
            node: 22
            coverage: true  # Only collect coverage once
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: ${{ matrix.node }} }
      - run: npm ci && npm test
      - if: matrix.coverage
        uses: codecov/codecov-action@v4
```

## Conditional Job Execution

```yaml
# Skip CI for docs-only changes
jobs:
  check-skip:
    runs-on: ubuntu-latest
    outputs:
      should-skip: ${{ steps.skip.outputs.should_skip }}
    steps:
      - uses: fkirc/skip-duplicate-actions@v5
        id: skip
        with:
          paths_ignore: '["docs/**", "*.md", "LICENSE"]'

  test:
    needs: check-skip
    if: needs.check-skip.outputs.should-skip != 'true'
    runs-on: ubuntu-latest
    steps:
      - run: npm test

# Environment-specific jobs
  deploy:
    if: |
      github.ref == 'refs/heads/main' &&
      github.event_name == 'push' &&
      !contains(github.event.head_commit.message, '[skip deploy]')
```

## Pipeline Notifications

```yaml
# Slack notification on failure
- name: Notify Slack
  if: failure()
  uses: slackapi/slack-github-action@v1.27.0
  with:
    payload: |
      {
        "text": "Pipeline failed: ${{ github.workflow }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*Pipeline Failed* :x:\n*Workflow*: ${{ github.workflow }}\n*Branch*: ${{ github.ref_name }}\n*Commit*: ${{ github.event.head_commit.message }}\n*Author*: ${{ github.actor }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```
