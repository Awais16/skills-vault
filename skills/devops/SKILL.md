---
name: devops
description: "Docker, CI/CD, deployment, and infrastructure patterns. Use when: writing Dockerfiles, configuring CI/CD pipelines (GitHub Actions), setting up environment variables, deploying to Vercel/AWS, managing multi-stage builds, or configuring monitoring and logging."
---

# DevOps

## When to Use

- Writing or optimizing Dockerfiles
- Setting up CI/CD pipelines (GitHub Actions)
- Configuring deployment (Vercel, AWS, Docker)
- Managing environment variables securely
- Setting up monitoring, logging, health checks
- Configuring preview/staging/production environments

## Docker

### Multi-Stage Dockerfile (Next.js)

```dockerfile
# Stage 1: Dependencies
FROM node:20-alpine AS deps
RUN corepack enable
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

# Stage 2: Build
FROM node:20-alpine AS builder
RUN corepack enable
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN pnpm build

# Stage 3: Production
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
CMD ["node", "server.js"]
```

### Docker Compose (Development)

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: deps
    command: pnpm dev
    volumes:
      - .:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### .dockerignore

```
node_modules
.next
.git
.env*.local
Dockerfile
docker-compose.yml
README.md
.github
```

## GitHub Actions CI/CD

### CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:run
      - run: pnpm test:coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  e2e:
    runs-on: ubuntu-latest
    needs: lint-and-typecheck
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps
      - run: pnpm build
      - run: pnpm test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  build:
    runs-on: ubuntu-latest
    needs: [test, e2e]
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

### Deploy Pipeline (Vercel)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ steps.deploy.outputs.url }}
    steps:
      - uses: actions/checkout@v4
      - uses: amondnet/vercel-action@v25
        id: deploy
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
```

## Environment Variables

### Structure

```bash
# .env.example — committed to git (template only)
DATABASE_URL=postgresql://user:password@localhost:5432/myapp
NEXT_PUBLIC_APP_URL=http://localhost:3000
JWT_SECRET=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

# .env.local — gitignored, local development values
# .env.development — development defaults
# .env.production — production defaults (non-secret only)
```

### Validation with Zod

```typescript
// lib/env.ts
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  STRIPE_SECRET_KEY: z.string().optional(),
});

export const env = envSchema.parse(process.env);
```

### Rules

- **Never** commit `.env.local` or files with real secrets
- **Always** provide `.env.example` with placeholder values
- Use `NEXT_PUBLIC_` prefix only for values safe to expose to the browser
- Validate all env vars at startup — fail fast if missing
- Use different values per environment (dev/staging/prod)

## Package.json Scripts

```jsonc
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint . --ext .ts,.tsx",
    "lint:fix": "eslint . --ext .ts,.tsx --fix",
    "typecheck": "tsc --noEmit",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "db:migrate": "prisma migrate dev",
    "db:push": "prisma db push",
    "db:seed": "prisma db seed",
    "db:studio": "prisma studio",
    "docker:up": "docker compose up -d",
    "docker:down": "docker compose down"
  }
}
```

## Health Check Endpoint

```typescript
// app/api/health/route.ts
import { NextResponse } from 'next/server';
import { db } from '@/lib/db';

export async function GET() {
  try {
    await db.$queryRaw`SELECT 1`;
    return NextResponse.json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version ?? 'unknown',
    });
  } catch {
    return NextResponse.json(
      { status: 'unhealthy', timestamp: new Date().toISOString() },
      { status: 503 },
    );
  }
}
```

## Deployment Checklist

- [ ] All env vars set in production environment
- [ ] Database migrations applied
- [ ] Build succeeds with `pnpm build`
- [ ] Health check endpoint responds
- [ ] Error tracking configured (Sentry or similar)
- [ ] Logging in structured format (JSON)
- [ ] SSL/TLS configured
- [ ] CORS configured for allowed origins
- [ ] Rate limiting on public APIs
- [ ] Secrets rotated and not in git history
