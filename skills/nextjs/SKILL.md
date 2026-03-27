---
name: nextjs
description: "Next.js App Router best practices and patterns. Use when: building Next.js applications, creating pages/layouts/routes, implementing Server Components, setting up API routes, configuring middleware, handling SSR/SSG/ISR, managing metadata/SEO, or optimizing Next.js performance."
---

# Next.js (App Router)

## When to Use

- Building or scaffolding Next.js applications
- Creating pages, layouts, loading/error states
- Implementing Server vs Client Components
- Setting up API Route Handlers
- Configuring middleware, redirects, rewrites
- Managing metadata and SEO
- Optimizing performance (caching, streaming, ISR)

## Project Structure

```
src/
├── app/                      # App Router
│   ├── layout.tsx            # Root layout
│   ├── page.tsx              # Home page
│   ├── loading.tsx           # Global loading UI
│   ├── error.tsx             # Global error boundary
│   ├── not-found.tsx         # 404 page
│   ├── (auth)/               # Route group (no URL segment)
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── dashboard/
│   │   ├── layout.tsx        # Dashboard-specific layout
│   │   ├── page.tsx
│   │   └── settings/page.tsx
│   └── api/                  # Route Handlers
│       └── users/route.ts
├── components/
│   ├── ui/                   # Generic, reusable UI components
│   └── features/             # Feature-specific components
├── lib/                      # Shared utilities, configs
├── types/                    # TypeScript type definitions
└── styles/                   # Global styles
```

## Server vs Client Components

### Default: Server Components

All components are Server Components by default. Keep them server-side unless they need interactivity.

```typescript
// app/users/page.tsx — Server Component (default)
import { db } from '@/lib/db';

export default async function UsersPage() {
  const users = await db.user.findMany();

  return (
    <ul>
      {users.map((user) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Client Components — Only When Needed

Add `'use client'` only for:
- Event handlers (`onClick`, `onChange`, etc.)
- React hooks (`useState`, `useEffect`, `useRef`, etc.)
- Browser-only APIs (`window`, `localStorage`, etc.)
- Third-party libraries that require client context

```typescript
'use client';

import { useState } from 'react';

export function SearchFilter({ initialQuery }: { initialQuery: string }) {
  const [query, setQuery] = useState(initialQuery);

  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

### Composition Pattern — Push Client Boundary Down

```typescript
// app/dashboard/page.tsx — Server Component
import { db } from '@/lib/db';
import { DashboardChart } from '@/components/features/dashboard-chart';

export default async function DashboardPage() {
  const stats = await db.stats.getMonthly();

  // Pass server data as props to client component
  return (
    <div>
      <h1>Dashboard</h1>
      <DashboardChart data={stats} />   {/* Client boundary is here, not the page */}
    </div>
  );
}
```

## Data Fetching

### Server Component Fetching (Preferred)

```typescript
// Fetch directly in Server Components — no useEffect, no loading states
export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id);

  if (!product) notFound();

  return <ProductDetail product={product} />;
}
```

### Caching Strategies

```typescript
// Default: cached (equivalent to SSG)
const data = await fetch('https://api.example.com/data');

// Revalidate every 60 seconds (ISR)
const data = await fetch('https://api.example.com/data', {
  next: { revalidate: 60 },
});

// No cache (SSR — fresh on every request)
const data = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});
```

### Route Segment Config

```typescript
// Per-page caching configuration
export const dynamic = 'force-dynamic';       // SSR
export const dynamic = 'force-static';        // SSG
export const revalidate = 3600;               // ISR (seconds)
export const runtime = 'edge';                // Edge runtime
```

## API Route Handlers

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';

const CreateUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = Number(searchParams.get('page') ?? '1');
  const limit = Number(searchParams.get('limit') ?? '10');

  const users = await db.user.findMany({
    skip: (page - 1) * limit,
    take: limit,
  });

  return NextResponse.json({ data: users, page, limit });
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  const parsed = CreateUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Validation failed', details: parsed.error.flatten() },
      { status: 400 },
    );
  }

  const user = await db.user.create({ data: parsed.data });
  return NextResponse.json({ data: user }, { status: 201 });
}
```

## Server Actions

```typescript
// app/actions/user.ts
'use server';

import { revalidatePath } from 'next/cache';
import { z } from 'zod';

const UpdateProfileSchema = z.object({
  name: z.string().min(1).max(100),
  bio: z.string().max(500).optional(),
});

export async function updateProfile(formData: FormData) {
  const parsed = UpdateProfileSchema.safeParse({
    name: formData.get('name'),
    bio: formData.get('bio'),
  });

  if (!parsed.success) {
    return { error: 'Invalid input' };
  }

  await db.user.update({
    where: { id: getCurrentUserId() },
    data: parsed.data,
  });

  revalidatePath('/profile');
  return { success: true };
}
```

## Metadata & SEO

```typescript
// Static metadata
export const metadata: Metadata = {
  title: 'Dashboard',
  description: 'Your personal dashboard',
};

// Dynamic metadata
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const product = await getProduct(id);
  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      images: [product.imageUrl],
    },
  };
}
```

## Error Handling

```typescript
// app/dashboard/error.tsx — Error boundary (must be client component)
'use client';

export default function DashboardError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}

// app/dashboard/loading.tsx — Streaming/Suspense loading
export default function DashboardLoading() {
  return <DashboardSkeleton />;
}
```

## Middleware

```typescript
// middleware.ts (root of project)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('session')?.value;

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/:path*'],
};
```

## Performance Checklist

- [ ] Use Server Components by default — only `'use client'` when needed
- [ ] Push client boundaries down — don't wrap entire pages in `'use client'`
- [ ] Use `next/image` for all images (automatic optimization)
- [ ] Use `next/font` for fonts (self-hosted, no layout shift)
- [ ] Use `next/link` for navigation (prefetching)
- [ ] Implement `loading.tsx` for Suspense streaming
- [ ] Set appropriate `revalidate` values for data freshness vs performance
- [ ] Use `generateStaticParams` for static generation of dynamic routes
- [ ] Implement parallel routes and intercepting routes where appropriate
- [ ] Use Route Groups `(folder)` to organize without affecting URL structure
