# Node.js Monorepo Patterns

## pnpm Workspaces

```yaml
# pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

```
my-monorepo/
  apps/
    api/                    # Express/NestJS API
      package.json          # "dependencies": { "@my-org/shared": "workspace:*" }
      src/
    worker/                 # Background job processor
      package.json
      src/
  packages/
    shared/                 # Shared types, utils
      package.json          # "name": "@my-org/shared"
      src/
      dist/
    database/               # Prisma schema + client
      package.json
      prisma/schema.prisma
      src/
    config/                 # Shared configuration
      package.json
      src/
  package.json              # Root — scripts, devDependencies
  pnpm-workspace.yaml
  turbo.json
  tsconfig.base.json
```

```json
// packages/shared/package.json
{
  "name": "@my-org/shared",
  "version": "0.0.0",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch"
  }
}

// apps/api/package.json
{
  "name": "@my-org/api",
  "dependencies": {
    "@my-org/shared": "workspace:*",
    "@my-org/database": "workspace:*"
  }
}
```

## Turborepo Configuration

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "lint": {},
    "dev": {
      "cache": false,
      "persistent": true
    },
    "db:generate": {
      "cache": false
    }
  }
}
```

```json
// Root package.json scripts
{
  "scripts": {
    "build": "turbo run build",
    "dev": "turbo run dev",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "clean": "turbo run clean && rm -rf node_modules"
  }
}
```

```bash
# Turbo commands
turbo run build                    # Build all packages in dependency order
turbo run build --filter=@my-org/api  # Build only api and its dependencies
turbo run test --filter=...[HEAD]  # Test only changed packages
turbo run dev --filter=@my-org/api --filter=@my-org/shared  # Dev specific packages
```

## TypeScript Project References

```json
// tsconfig.base.json (root)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true,
    "skipLibCheck": true
  }
}

// packages/shared/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}

// apps/api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "references": [
    { "path": "../../packages/shared" },
    { "path": "../../packages/database" }
  ],
  "include": ["src"]
}
```

## Docker Builds for Monorepos

```dockerfile
# Dockerfile for apps/api using multi-stage build
FROM node:20-slim AS base
RUN corepack enable

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json ./apps/api/
COPY packages/shared/package.json ./packages/shared/
COPY packages/database/package.json ./packages/database/
RUN pnpm install --frozen-lockfile --prod

FROM base AS build
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/ ./apps/api/
COPY packages/shared/ ./packages/shared/
COPY packages/database/ ./packages/database/
RUN pnpm install --frozen-lockfile
RUN pnpm --filter @my-org/shared build
RUN pnpm --filter @my-org/database build
RUN pnpm --filter @my-org/api build

FROM base AS runner
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/apps/api/dist ./dist
COPY --from=build /app/packages/shared/dist ./packages/shared/dist
USER node
CMD ["node", "dist/main.js"]
```

## CI Optimization

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2 # For turbo change detection
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run lint test build --filter=...[HEAD^1]
        # Only run tasks for packages that changed since last commit
```
