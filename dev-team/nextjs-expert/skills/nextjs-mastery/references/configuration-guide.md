# Configuration Guide

## Environment Variables

```bash
# .env.local (never committed — local development)
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
NEXTAUTH_SECRET=your-secret-key

# .env (committed — defaults)
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:3000/api

# Rules:
# NEXT_PUBLIC_ prefix → available in browser (bundled into client JS)
# No prefix → server-only (never exposed to client)
# .env.local > .env.development > .env

# Access in code:
# Server: process.env.DATABASE_URL
# Client: process.env.NEXT_PUBLIC_APP_URL
```

```typescript
// lib/env.ts — type-safe env with Zod
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(32),
  NEXTAUTH_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  NEXT_PUBLIC_APP_URL: z.string().url(),
});

export const env = envSchema.parse(process.env);
// Throws at build time if env vars are missing/invalid
```

## next.config.js Deep Dive

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Output mode
  output: 'standalone',  // For Docker deployment
  // output: 'export',   // Static site (no server features)

  // Turbopack (dev only — faster HMR)
  // next dev --turbopack

  // Strict mode (double-renders in dev to catch bugs)
  reactStrictMode: true,

  // Image optimization
  images: {
    formats: ['image/avif', 'image/webp'],
    remotePatterns: [
      { protocol: 'https', hostname: '**.example.com' },
    ],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256],
  },

  // Experimental features
  experimental: {
    typedRoutes: true,        // Type-safe Link href
    ppr: true,                // Partial Prerendering
    serverActions: {
      bodySizeLimit: '2mb',
    },
  },

  // Webpack customization
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.fallback = { fs: false, net: false, tls: false };
    }
    return config;
  },

  // Redirects
  async redirects() {
    return [
      { source: '/blog/:slug', destination: '/posts/:slug', permanent: true },
    ];
  },

  // Rewrites (proxy)
  async rewrites() {
    return [
      { source: '/api/v1/:path*', destination: 'https://api.example.com/:path*' },
    ];
  },

  // Security headers
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-DNS-Prefetch-Control', value: 'on' },
          { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
          { key: 'X-Frame-Options', value: 'SAMEORIGIN' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
        ],
      },
    ];
  },

  // Logging
  logging: {
    fetches: { fullUrl: true }, // Log fetch URLs in dev
  },
};

module.exports = nextConfig;
```

## TypeScript Configuration

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./src/*"],
      "@/components/*": ["./src/components/*"],
      "@/lib/*": ["./src/lib/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## Output Modes

```
standalone (default for deployment):
  - Self-contained Node.js server
  - Includes only necessary files
  - Ideal for Docker: COPY .next/standalone ./
  - Supports all Next.js features

export (static):
  - Pure static HTML/CSS/JS
  - No server required
  - Deploy to any static host (S3, Cloudflare Pages)
  - NO: SSR, ISR, middleware, Route Handlers, Server Actions
  - YES: SSG, client-side rendering

Docker with standalone:
  FROM node:20-slim AS runner
  WORKDIR /app
  COPY --from=builder /app/.next/standalone ./
  COPY --from=builder /app/.next/static ./.next/static
  COPY --from=builder /app/public ./public
  ENV PORT=3000
  EXPOSE 3000
  CMD ["node", "server.js"]
```
