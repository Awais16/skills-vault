---
name: database
description: "Database schema design, ORMs, migrations, and query optimization. Use when: designing database schemas, writing Prisma/Drizzle models, creating migrations, optimizing queries, handling relationships, seeding data, or troubleshooting N+1 queries."
---

# Database

## When to Use

- Designing or modifying database schemas
- Writing Prisma or Drizzle ORM models
- Creating and running migrations
- Optimizing slow queries or fixing N+1 problems
- Implementing relationships (1:1, 1:N, M:N)
- Setting up database seeding

## ORM Selection

| ORM | Best For |
|-----|----------|
| **Prisma** (default) | Most projects — excellent DX, type-safe, great migrations |
| **Drizzle** | Performance-critical apps, edge runtime, SQL-first approach |

## Prisma

### Setup

```bash
pnpm add @prisma/client
pnpm add -D prisma
pnpm exec prisma init
```

### Schema Design

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  role      Role     @default(USER)
  posts     Post[]
  profile   Profile?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([email])
  @@map("users")
}

model Profile {
  id     String @id @default(cuid())
  bio    String?
  avatar String?
  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("profiles")
}

model Post {
  id          String     @id @default(cuid())
  title       String
  content     String?
  published   Boolean    @default(false)
  authorId    String
  author      User       @relation(fields: [authorId], references: [id], onDelete: Cascade)
  categories  Category[]
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  @@index([authorId])
  @@index([published, createdAt])
  @@map("posts")
}

model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]

  @@map("categories")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### Schema Conventions

- Use `cuid()` or `uuid()` for IDs (not auto-increment for distributed systems)
- Always add `createdAt` and `updatedAt` timestamps
- Add `@@index` on foreign keys and frequently queried fields
- Use `@@map` to control table names (snake_case in DB)
- Set `onDelete: Cascade` for owned relationships
- Use enums for fixed-set values

### Prisma Client Singleton

```typescript
// lib/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query', 'warn', 'error'] : ['error'],
  });

if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = db;
```

### Common Queries

```typescript
// Find with relations
const user = await db.user.findUnique({
  where: { id },
  include: { profile: true, posts: { where: { published: true } } },
});

// Pagination
const users = await db.user.findMany({
  skip: (page - 1) * limit,
  take: limit,
  orderBy: { createdAt: 'desc' },
});

// Transaction
const [user, post] = await db.$transaction([
  db.user.create({ data: userData }),
  db.post.create({ data: postData }),
]);

// Interactive transaction
await db.$transaction(async (tx) => {
  const user = await tx.user.findUnique({ where: { id } });
  if (!user) throw new Error('User not found');
  await tx.post.create({ data: { ...postData, authorId: user.id } });
});

// Upsert
const user = await db.user.upsert({
  where: { email },
  create: { email, name },
  update: { name },
});
```

### Migrations

```bash
# Create migration from schema changes
pnpm exec prisma migrate dev --name add-user-profile

# Apply migrations in production
pnpm exec prisma migrate deploy

# Reset database (dev only)
pnpm exec prisma migrate reset

# Generate client after schema change
pnpm exec prisma generate
```

## Drizzle

### Setup

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit
```

### Schema Definition

```typescript
// db/schema.ts
import { pgTable, text, timestamp, boolean, pgEnum } from 'drizzle-orm/pg-core';

export const roleEnum = pgEnum('role', ['user', 'admin', 'moderator']);

export const users = pgTable('users', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  email: text('email').unique().notNull(),
  name: text('name').notNull(),
  role: roleEnum('role').default('user').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  title: text('title').notNull(),
  content: text('content'),
  published: boolean('published').default(false).notNull(),
  authorId: text('author_id').references(() => users.id, { onDelete: 'cascade' }).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

## Query Optimization

### Avoid N+1 Queries

```typescript
// Bad — N+1: 1 query for users + N queries for posts
const users = await db.user.findMany();
for (const user of users) {
  const posts = await db.post.findMany({ where: { authorId: user.id } });
}

// Good — eager load with include
const users = await db.user.findMany({
  include: { posts: true },
});
```

### Indexing Strategy

| Query Pattern | Index |
|--------------|-------|
| `WHERE email = ?` | Unique index on `email` |
| `WHERE authorId = ?` | Index on `authorId` (foreign key) |
| `WHERE published = true ORDER BY createdAt` | Composite index `(published, createdAt)` |
| `WHERE name LIKE 'A%'` | Index on `name` (prefix match only) |
| Full-text search | Dedicated search index (pg_trgm, tsvector) |

### Select Only Needed Fields

```typescript
// Instead of selecting all columns
const users = await db.user.findMany({
  select: { id: true, name: true, email: true },
  // Don't include: password hash, internal fields, large blobs
});
```

## Seeding

```typescript
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  await prisma.user.createMany({
    data: [
      { email: 'admin@example.com', name: 'Admin', role: 'ADMIN' },
      { email: 'user@example.com', name: 'User', role: 'USER' },
    ],
    skipDuplicates: true,
  });

  console.log('Seed complete');
}

main()
  .catch(console.error)
  .finally(() => prisma.$disconnect());
```

```jsonc
// package.json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```
