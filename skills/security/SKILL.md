---
name: security
description: "Web application security best practices and OWASP patterns. Use when: implementing authentication, authorization, input validation, sanitization, CSRF/XSS prevention, securing API endpoints, managing secrets, handling file uploads, configuring CORS, or auditing code for security vulnerabilities."
---

# Security

## When to Use

- Implementing authentication or authorization flows
- Validating and sanitizing user input
- Preventing XSS, CSRF, SQL injection, or SSRF
- Securing API endpoints and middleware
- Managing secrets and environment variables
- Configuring CORS, CSP, and security headers
- Auditing code for vulnerabilities
- Handling file uploads safely

## OWASP Top 10 — Quick Reference

| # | Risk | Prevention |
|---|------|-----------|
| 1 | **Broken Access Control** | Deny by default, validate on every request, server-side checks |
| 2 | **Cryptographic Failures** | TLS everywhere, strong hashing (bcrypt/argon2), no secrets in code |
| 3 | **Injection** | Parameterized queries, input validation, output encoding |
| 4 | **Insecure Design** | Threat modeling, least privilege, defense in depth |
| 5 | **Security Misconfiguration** | Secure defaults, remove unused features, security headers |
| 6 | **Vulnerable Components** | Audit deps (`pnpm audit`), keep updated, minimize surface |
| 7 | **Auth Failures** | Strong passwords, MFA, rate limit login, secure session management |
| 8 | **Data Integrity Failures** | Verify signatures, use SRI for CDN scripts, signed deployments |
| 9 | **Logging Failures** | Log auth events, detect anomalies, never log secrets |
| 10 | **SSRF** | Allowlist URLs, validate/sanitize URLs, block internal ranges |

## Input Validation

### Always Validate at System Boundaries

```typescript
import { z } from 'zod';

// Validate ALL user input — never trust the client
const CreateUserSchema = z.object({
  name: z.string().min(1).max(100).trim(),
  email: z.string().email().toLowerCase(),
  age: z.number().int().min(13).max(150),
  bio: z.string().max(500).optional(),
  role: z.enum(['user', 'viewer']),  // Never allow 'admin' from client input
});

// Validate URL parameters
const IdParamSchema = z.object({
  id: z.string().uuid(),  // Enforce format, don't accept arbitrary strings
});

// Validate query params
const SearchSchema = z.object({
  q: z.string().max(200).trim(),
  page: z.coerce.number().int().positive().default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});
```

### Sanitization Rules

- **Trim** whitespace on text inputs
- **Normalize** emails to lowercase
- **Strip** HTML tags from user text (unless rich text is intended)
- **Reject** inputs that exceed expected length/format
- **Never** use user input in `eval()`, `new Function()`, or template literals for SQL/commands

## XSS Prevention

### React's Built-in Protection

React escapes values in JSX by default. These are safe:

```typescript
// Safe — React escapes this automatically
<p>{userInput}</p>
<div title={userInput}>content</div>
```

### Dangerous Patterns to Avoid

```typescript
// DANGEROUS — never do this
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// DANGEROUS — user input in href can execute JS
<a href={userProvidedUrl}>Click</a>  // Could be "javascript:alert(1)"

// Safe — validate URLs
function isSafeUrl(url: string): boolean {
  try {
    const parsed = new URL(url);
    return ['http:', 'https:'].includes(parsed.protocol);
  } catch {
    return false;
  }
}

<a href={isSafeUrl(url) ? url : '#'}>Click</a>
```

### Content Security Policy (CSP)

```typescript
// next.config.ts
const cspHeader = `
  default-src 'self';
  script-src 'self' 'nonce-{NONCE}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' blob: data:;
  font-src 'self';
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
`.replace(/\n/g, '');
```

## CSRF Protection

### Server Actions (Next.js)

Next.js Server Actions have built-in CSRF protection via origin checking. Use them for mutations:

```typescript
'use server';

export async function updateProfile(formData: FormData) {
  // CSRF is handled automatically by Next.js
  const session = await getSession();
  if (!session) throw new Error('Unauthorized');

  // Process the form...
}
```

### API Routes — CSRF Token Pattern

```typescript
// For API routes called by custom JavaScript (not Server Actions)
import { headers } from 'next/headers';

export async function POST(request: NextRequest) {
  const origin = request.headers.get('origin');
  const host = request.headers.get('host');

  // Verify the request is from your own domain
  if (!origin || !origin.includes(host ?? '')) {
    return NextResponse.json({ error: 'CSRF check failed' }, { status: 403 });
  }

  // Process request...
}
```

## Authentication

### Password Hashing

```typescript
import bcrypt from 'bcryptjs';

const SALT_ROUNDS = 12;

async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

### Password Requirements

```typescript
const PasswordSchema = z
  .string()
  .min(8, 'Password must be at least 8 characters')
  .max(128, 'Password must be at most 128 characters')
  .regex(/[a-z]/, 'Must contain a lowercase letter')
  .regex(/[A-Z]/, 'Must contain an uppercase letter')
  .regex(/[0-9]/, 'Must contain a number');
```

### JWT Best Practices

```typescript
import { SignJWT, jwtVerify } from 'jose';

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET);

async function createToken(userId: string, role: string): Promise<string> {
  return new SignJWT({ sub: userId, role })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('1h')        // Short-lived access tokens
    .setIssuer('your-app')
    .setAudience('your-app')
    .sign(SECRET);
}

async function verifyToken(token: string) {
  const { payload } = await jwtVerify(token, SECRET, {
    issuer: 'your-app',
    audience: 'your-app',
  });
  return payload;
}
```

**JWT Rules:**
- Keep access tokens short-lived (15min–1h)
- Use refresh tokens (stored HttpOnly cookie) for session extension
- Never store JWTs in `localStorage` — use HttpOnly cookies
- Always validate `iss`, `aud`, and `exp` claims
- Use `HS256` for single-service, `RS256`/`ES256` for multi-service

### Secure Cookie Configuration

```typescript
import { cookies } from 'next/headers';

async function setSessionCookie(token: string) {
  const cookieStore = await cookies();
  cookieStore.set('session', token, {
    httpOnly: true,      // Not accessible via JavaScript
    secure: true,        // HTTPS only
    sameSite: 'lax',     // CSRF protection
    maxAge: 60 * 60,     // 1 hour
    path: '/',
  });
}
```

## Authorization

### Role-Based Access Control (RBAC)

```typescript
type Role = 'user' | 'moderator' | 'admin';

const PERMISSIONS = {
  'post:read': ['user', 'moderator', 'admin'],
  'post:create': ['user', 'moderator', 'admin'],
  'post:edit': ['moderator', 'admin'],
  'post:delete': ['admin'],
  'user:manage': ['admin'],
} as const satisfies Record<string, readonly Role[]>;

type Permission = keyof typeof PERMISSIONS;

function hasPermission(userRole: Role, permission: Permission): boolean {
  return PERMISSIONS[permission].includes(userRole);
}

// Middleware pattern
function requirePermission(permission: Permission) {
  return async function (request: NextRequest) {
    const auth = await verifyAuth(request);
    if (!auth.authenticated) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
    }
    if (!hasPermission(auth.role as Role, permission)) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 });
    }
  };
}
```

### Never Trust Client-Side Authorization

```typescript
// BAD — checking role on the client only
if (user.role === 'admin') {
  await deleteUser(targetId); // API has no auth check!
}

// GOOD — always enforce on the server
export async function DELETE(request: NextRequest, { params }: { params: Promise<{ id: string }> }) {
  const auth = await verifyAuth(request);
  if (!auth.authenticated) return unauthorized();
  if (auth.role !== 'admin') return forbidden();

  const { id } = await params;
  await db.user.delete({ where: { id } });
  return NextResponse.json(null, { status: 204 });
}
```

## SQL Injection Prevention

```typescript
// ALWAYS use parameterized queries

// Prisma — safe by default (parameterized)
const user = await db.user.findFirst({
  where: { email: userInput },  // Prisma parameterizes this
});

// Raw queries — ALWAYS use tagged templates
const users = await db.$queryRaw`
  SELECT * FROM users WHERE email = ${userInput}
`;  // Prisma parameterizes the template literal

// NEVER concatenate user input into SQL
// BAD: await db.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${userInput}'`);
```

## Security Headers

```typescript
// next.config.ts
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '0' },  // Rely on CSP instead
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
];

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }];
  },
};
```

## Rate Limiting

```typescript
// Simple per-IP rate limiter for login endpoints
import { NextRequest, NextResponse } from 'next/server';

const loginAttempts = new Map<string, { count: number; resetAt: number }>();
const MAX_ATTEMPTS = 5;
const WINDOW_MS = 15 * 60 * 1000; // 15 minutes

function checkLoginRateLimit(ip: string): boolean {
  const now = Date.now();
  const entry = loginAttempts.get(ip);

  if (!entry || now > entry.resetAt) {
    loginAttempts.set(ip, { count: 1, resetAt: now + WINDOW_MS });
    return true;
  }

  if (entry.count >= MAX_ATTEMPTS) return false;
  entry.count++;
  return true;
}

// In your login route handler
export async function POST(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') ?? 'unknown';

  if (!checkLoginRateLimit(ip)) {
    return NextResponse.json(
      { error: { code: 'RATE_LIMITED', message: 'Too many login attempts. Try again later.' } },
      { status: 429, headers: { 'Retry-After': '900' } },
    );
  }

  // Process login...
}
```

## File Upload Safety

```typescript
const ALLOWED_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp', 'application/pdf']);
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateUpload(file: File): Result<File, string> {
  if (!ALLOWED_TYPES.has(file.type)) {
    return { ok: false, error: `File type ${file.type} not allowed` };
  }
  if (file.size > MAX_SIZE) {
    return { ok: false, error: `File exceeds ${MAX_SIZE / 1024 / 1024}MB limit` };
  }
  // Never trust the file extension — validate by content type / magic bytes
  return { ok: true, value: file };
}
```

## Secrets Management

- **Never** commit secrets to git — use `.env.local` (gitignored)
- **Always** provide `.env.example` with placeholder values
- **Validate** all env vars at startup with Zod
- **Rotate** secrets regularly and after any suspected compromise
- **Use** platform-native secret stores in production (Vercel env vars, AWS Secrets Manager)
- **Scope** `NEXT_PUBLIC_` prefix only to values safe for browser exposure

## Dependency Security

```bash
# Audit dependencies regularly
pnpm audit

# Fix known vulnerabilities
pnpm audit --fix

# Check for outdated packages
pnpm outdated

# Use lockfile to ensure reproducible installs
pnpm install --frozen-lockfile
```

## Security Audit Checklist

- [ ] All user input validated with Zod at system boundaries
- [ ] No `dangerouslySetInnerHTML` with user content
- [ ] URLs validated before use in `href`, `src`, `fetch`, or redirects
- [ ] Passwords hashed with bcrypt (cost 12+) or argon2
- [ ] JWTs have short expiration, stored in HttpOnly cookies
- [ ] RBAC enforced server-side on every protected endpoint
- [ ] SQL uses parameterized queries (Prisma/Drizzle by default)
- [ ] Security headers configured (CSP, HSTS, X-Frame-Options)
- [ ] Rate limiting on login, signup, and password reset endpoints
- [ ] File uploads validated by type and size
- [ ] No secrets in source code or git history
- [ ] Dependencies audited with `pnpm audit`
- [ ] CORS configured to allow only expected origins
- [ ] Error messages don't leak internal details to clients
- [ ] Logging captures auth events without exposing secrets
