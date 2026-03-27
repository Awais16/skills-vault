---
name: api-design
description: "RESTful API and GraphQL design patterns. Use when: designing API endpoints, implementing route handlers, validating request data, handling API errors, structuring API responses, implementing authentication, rate limiting, or versioning APIs."
---

# API Design

## When to Use

- Designing new API endpoints or restructuring existing ones
- Implementing request validation and error handling
- Structuring consistent API responses
- Adding authentication/authorization to APIs
- Implementing pagination, filtering, sorting
- Designing webhook or event-driven APIs

## REST Conventions

### URL Structure

```
GET    /api/users              # List users
GET    /api/users/:id          # Get user by ID
POST   /api/users              # Create user
PATCH  /api/users/:id          # Partial update
PUT    /api/users/:id          # Full replace
DELETE /api/users/:id          # Delete user

# Nested resources
GET    /api/users/:id/orders   # User's orders
POST   /api/users/:id/orders   # Create order for user

# Actions (when CRUD doesn't fit)
POST   /api/users/:id/activate
POST   /api/orders/:id/cancel
```

### Naming Rules

- Use **plural nouns**: `/users`, not `/user`
- Use **kebab-case**: `/order-items`, not `/orderItems`
- Use **nouns** for resources, not verbs: `GET /users`, not `GET /getUsers`
- Nest only 1 level deep: `/users/:id/orders`, not `/users/:id/orders/:oid/items`

### HTTP Status Codes

| Code | Meaning | Use When |
|------|---------|----------|
| 200 | OK | Successful GET, PATCH, PUT |
| 201 | Created | Successful POST that creates a resource |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Validation error, malformed input |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Authenticated but not authorized |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource, version conflict |
| 422 | Unprocessable Entity | Valid syntax but semantic errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server failure |

## Response Format

### Consistent Envelope

```typescript
// Success response
type ApiSuccess<T> = {
  data: T;
  meta?: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
};

// Error response
type ApiError = {
  error: {
    code: string;        // Machine-readable: "VALIDATION_ERROR"
    message: string;     // Human-readable: "Email is required"
    details?: unknown;   // Field-level errors, stack trace (dev only)
  };
};
```

### Examples

```typescript
// GET /api/users?page=1&limit=10
{
  "data": [
    { "id": "1", "name": "Alice", "email": "alice@example.com" },
    { "id": "2", "name": "Bob", "email": "bob@example.com" }
  ],
  "meta": { "page": 1, "limit": 10, "total": 42, "totalPages": 5 }
}

// POST /api/users — 201
{
  "data": { "id": "3", "name": "Charlie", "email": "charlie@example.com" }
}

// POST /api/users — 400
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request body",
    "details": {
      "fieldErrors": {
        "email": ["Invalid email format"],
        "name": ["Name is required"]
      }
    }
  }
}
```

## Validation (Zod)

```typescript
import { z } from 'zod';

// Define schemas for each endpoint
const CreateUserSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  email: z.string().email('Invalid email format'),
  role: z.enum(['admin', 'user', 'viewer']).default('user'),
});

const UpdateUserSchema = CreateUserSchema.partial();

const PaginationSchema = z.object({
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  sort: z.enum(['name', 'email', 'createdAt']).default('createdAt'),
  order: z.enum(['asc', 'desc']).default('desc'),
});

// Reusable validation helper for Next.js Route Handlers
function validateBody<T>(schema: z.ZodSchema<T>, body: unknown): Result<T, ApiError> {
  const parsed = schema.safeParse(body);
  if (!parsed.success) {
    return {
      ok: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid request body',
        details: parsed.error.flatten(),
      },
    };
  }
  return { ok: true, value: parsed.data };
}
```

## Next.js Route Handler Pattern

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  try {
    const params = PaginationSchema.safeParse(
      Object.fromEntries(request.nextUrl.searchParams),
    );

    if (!params.success) {
      return NextResponse.json(
        { error: { code: 'VALIDATION_ERROR', message: 'Invalid parameters', details: params.error.flatten() } },
        { status: 400 },
      );
    }

    const { page, limit, sort, order } = params.data;
    const [users, total] = await Promise.all([
      db.user.findMany({
        skip: (page - 1) * limit,
        take: limit,
        orderBy: { [sort]: order },
      }),
      db.user.count(),
    ]);

    return NextResponse.json({
      data: users,
      meta: { page, limit, total, totalPages: Math.ceil(total / limit) },
    });
  } catch (error) {
    console.error('GET /api/users failed:', error);
    return NextResponse.json(
      { error: { code: 'INTERNAL_ERROR', message: 'Internal server error' } },
      { status: 500 },
    );
  }
}
```

## Authentication Patterns

### JWT Token Verification

```typescript
// lib/auth.ts
import { jwtVerify } from 'jose';

type AuthResult =
  | { authenticated: true; userId: string; role: string }
  | { authenticated: false; error: string };

export async function verifyAuth(request: NextRequest): Promise<AuthResult> {
  const token = request.headers.get('authorization')?.split('Bearer ')[1];

  if (!token) {
    return { authenticated: false, error: 'Missing authorization header' };
  }

  try {
    const { payload } = await jwtVerify(
      token,
      new TextEncoder().encode(process.env.JWT_SECRET),
    );
    return { authenticated: true, userId: payload.sub!, role: payload.role as string };
  } catch {
    return { authenticated: false, error: 'Invalid or expired token' };
  }
}
```

### Protected Route Handler

```typescript
export async function GET(request: NextRequest) {
  const auth = await verifyAuth(request);
  if (!auth.authenticated) {
    return NextResponse.json(
      { error: { code: 'UNAUTHORIZED', message: auth.error } },
      { status: 401 },
    );
  }

  // auth.userId and auth.role are available
  const users = await db.user.findMany();
  return NextResponse.json({ data: users });
}
```

## Pagination Patterns

### Offset-Based (Simple)

```
GET /api/users?page=2&limit=20
```

Good for UI with page numbers. Can be slow for large datasets.

### Cursor-Based (Scalable)

```
GET /api/users?cursor=abc123&limit=20
```

```typescript
type CursorPagination<T> = {
  data: T[];
  nextCursor: string | null;
  hasMore: boolean;
};
```

Good for infinite scroll, real-time feeds. Handles insertions/deletions correctly.

## Rate Limiting

```typescript
// Simple in-memory rate limiter (use Redis in production)
const rateLimitMap = new Map<string, { count: number; resetTime: number }>();

function rateLimit(identifier: string, limit = 60, windowMs = 60_000): boolean {
  const now = Date.now();
  const entry = rateLimitMap.get(identifier);

  if (!entry || now > entry.resetTime) {
    rateLimitMap.set(identifier, { count: 1, resetTime: now + windowMs });
    return true;
  }

  if (entry.count >= limit) return false;

  entry.count++;
  return true;
}
```

## API Design Checklist

- [ ] Consistent URL naming (plural, kebab-case, nouns)
- [ ] Proper HTTP methods and status codes
- [ ] Input validation on all endpoints (Zod)
- [ ] Consistent response envelope (`data` / `error`)
- [ ] Authentication and authorization on protected routes
- [ ] Pagination for list endpoints
- [ ] Error responses include machine-readable `code`
- [ ] No sensitive data leaked in responses (passwords, tokens)
- [ ] Rate limiting on public endpoints
- [ ] Request logging for debugging and monitoring
