---
name: typescript
description: "TypeScript best practices, advanced type patterns, and strict typing. Use when: writing TypeScript code, creating type definitions, fixing type errors, designing type-safe APIs, using generics, creating utility types, or migrating from JavaScript to TypeScript."
---

# TypeScript

## When to Use

- Writing new TypeScript code or modules
- Fixing type errors or improving type safety
- Designing type-safe function signatures and APIs
- Creating reusable generic types or utility types
- Migrating JavaScript to TypeScript

## Strict Configuration

Always use strict mode. Recommended `tsconfig.json` base:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "forceConsistentCasingInFileNames": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "moduleResolution": "bundler",
    "module": "ESNext",
    "target": "ES2022",
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true
  }
}
```

## Type Conventions

### Prefer `type` Over `interface`

```typescript
// Default — use type aliases
type User = {
  id: string;
  name: string;
  email: string;
};

// Use interface only when extending or declaration merging is needed
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  findAll(): Promise<T[]>;
  create(data: Omit<T, 'id'>): Promise<T>;
}
```

### Avoid `any` — Use `unknown`

```typescript
// Bad
function parse(input: any): any { ... }

// Good
function parse(input: unknown): Result<ParsedData, ParseError> {
  if (typeof input !== 'string') {
    return { ok: false, error: new ParseError('Expected string') };
  }
  // narrow the type...
}
```

### Prefer Unions Over Enums

```typescript
// Prefer this
type Status = 'idle' | 'loading' | 'success' | 'error';

// Over enum (enums emit runtime code and have quirks)
// Only use enum for numeric values that need reverse mapping
```

### Discriminated Unions for State

```typescript
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function renderState<T>(state: AsyncState<T>) {
  switch (state.status) {
    case 'idle': return null;
    case 'loading': return <Spinner />;
    case 'success': return <Data value={state.data} />;  // TS knows data exists
    case 'error': return <Error message={state.error.message} />;
  }
}
```

## Advanced Patterns

### Branded Types (Prevent Value Mixing)

```typescript
type UserId = string & { readonly __brand: 'UserId' };
type OrderId = string & { readonly __brand: 'OrderId' };

function createUserId(id: string): UserId {
  return id as UserId;
}

function getUser(id: UserId): Promise<User> { ... }

// getUser(orderId) — compile error! Can't pass OrderId as UserId
```

### Const Assertions

```typescript
const ROUTES = {
  home: '/',
  profile: '/profile',
  settings: '/settings',
} as const;

type Route = (typeof ROUTES)[keyof typeof ROUTES];
// Type: '/' | '/profile' | '/settings'
```

### Generic Constraints

```typescript
// Constrain generics to what you actually need
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// Generic with default
type ApiResponse<T = unknown> = {
  data: T;
  status: number;
  timestamp: Date;
};
```

### Type Guards

```typescript
// Custom type guard
function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'email' in value
  );
}

// Assertion function
function assertDefined<T>(value: T | null | undefined, msg?: string): asserts value is T {
  if (value == null) throw new Error(msg ?? 'Value is null or undefined');
}
```

### Utility Types (Quick Reference)

| Type | Purpose | Example |
|------|---------|---------|
| `Partial<T>` | All props optional | `Partial<User>` for updates |
| `Required<T>` | All props required | `Required<Config>` |
| `Pick<T, K>` | Select specific props | `Pick<User, 'id' \| 'name'>` |
| `Omit<T, K>` | Remove specific props | `Omit<User, 'password'>` |
| `Record<K, V>` | Key-value map | `Record<string, number>` |
| `Extract<T, U>` | Extract matching union members | `Extract<Status, 'success' \| 'error'>` |
| `Exclude<T, U>` | Remove matching union members | `Exclude<Status, 'idle'>` |
| `NonNullable<T>` | Remove null/undefined | `NonNullable<string \| null>` |
| `ReturnType<F>` | Get function return type | `ReturnType<typeof fetchUser>` |
| `Awaited<T>` | Unwrap Promise type | `Awaited<Promise<User>>` → `User` |

### Consistent Type Imports

```typescript
// Always use type-only imports for types
import type { User, UserRole } from '@/types/user';
import { createUser } from '@/services/user';
```

## Error Handling Patterns

### Result Type

```typescript
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Usage
async function fetchUser(id: string): Promise<Result<User, ApiError>> {
  try {
    const user = await db.user.findUnique({ where: { id } });
    if (!user) return err(new ApiError('NOT_FOUND', 'User not found'));
    return ok(user);
  } catch (e) {
    return err(new ApiError('INTERNAL', 'Database error'));
  }
}
```

### Exhaustive Switch

```typescript
function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${value}`);
}

function handleStatus(status: Status) {
  switch (status) {
    case 'idle': return;
    case 'loading': return;
    case 'success': return;
    case 'error': return;
    default: assertNever(status); // compile error if a case is missing
  }
}
```

## Zod Integration (Runtime Validation)

```typescript
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'viewer']),
});

// Derive TypeScript type from schema
type User = z.infer<typeof UserSchema>;

// Validate at system boundaries
function parseUser(input: unknown): User {
  return UserSchema.parse(input);
}
```
