---
name: testing
description: "Testing strategies and patterns for TypeScript/React/Next.js. Use when: writing unit tests, integration tests, e2e tests, setting up Vitest/Jest/Playwright, testing React components, testing API routes, mocking dependencies, or establishing testing patterns."
---

# Testing

## When to Use

- Writing unit, integration, or e2e tests
- Setting up test infrastructure (Vitest, Jest, Playwright)
- Testing React components, hooks, and forms
- Testing Next.js API routes and Server Components
- Mocking dependencies, APIs, or databases
- Establishing testing patterns for a project

## Testing Strategy

### What to Test (Priority Order)

1. **Business logic** — Pure functions, calculations, transformations
2. **Integration points** — API handlers, database queries, external services
3. **User interactions** — Forms, navigation, critical user flows
4. **Edge cases** — Null values, empty arrays, boundary conditions, error states
5. **Regression** — Bugs that were fixed (prevent them from coming back)

### What NOT to Test

- Implementation details (internal state, private methods)
- Third-party library internals
- Simple getters/setters with no logic
- Constant values or static configuration
- CSS or visual styling (use visual regression tools instead)

## Vitest (Preferred Test Runner)

### Setup

```bash
pnpm add -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.test.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'src/test/'],
    },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
```

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom/vitest';
```

### Unit Tests

```typescript
// utils/format-currency.ts
export function formatCurrency(amount: number, currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
}

// utils/format-currency.test.ts
import { describe, it, expect } from 'vitest';
import { formatCurrency } from './format-currency';

describe('formatCurrency', () => {
  it('formats USD by default', () => {
    expect(formatCurrency(1234.5)).toBe('$1,234.50');
  });

  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });

  it('handles negative amounts', () => {
    expect(formatCurrency(-50)).toBe('-$50.00');
  });

  it('supports other currencies', () => {
    expect(formatCurrency(1000, 'EUR')).toBe('€1,000.00');
  });
});
```

### React Component Tests

```typescript
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { SearchFilter } from './search-filter';

describe('SearchFilter', () => {
  it('renders with initial query', () => {
    render(<SearchFilter initialQuery="hello" onSearch={vi.fn()} />);
    expect(screen.getByRole('textbox')).toHaveValue('hello');
  });

  it('calls onSearch when form is submitted', async () => {
    const user = userEvent.setup();
    const onSearch = vi.fn();

    render(<SearchFilter initialQuery="" onSearch={onSearch} />);

    await user.type(screen.getByRole('textbox'), 'test query');
    await user.click(screen.getByRole('button', { name: /search/i }));

    expect(onSearch).toHaveBeenCalledWith('test query');
  });

  it('shows error for empty search', async () => {
    const user = userEvent.setup();
    render(<SearchFilter initialQuery="" onSearch={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /search/i }));

    expect(screen.getByRole('alert')).toHaveTextContent('Please enter a search term');
  });
});
```

### Hook Tests

```typescript
import { describe, it, expect } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './use-counter';

describe('useCounter', () => {
  it('starts with initial value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increments', () => {
    const { result } = renderHook(() => useCounter(0));
    act(() => result.current.increment());
    expect(result.current.count).toBe(1);
  });
});
```

## Mocking

### Mock Functions

```typescript
// Mock a module
vi.mock('@/lib/db', () => ({
  db: {
    user: {
      findMany: vi.fn(),
      create: vi.fn(),
    },
  },
}));

// Mock fetch
vi.stubGlobal('fetch', vi.fn());

// Mock implementation for a specific test
it('handles API error', async () => {
  vi.mocked(fetch).mockResolvedValueOnce(
    new Response(JSON.stringify({ error: 'Not found' }), { status: 404 }),
  );

  const result = await fetchUser('invalid-id');
  expect(result.ok).toBe(false);
});
```

### MSW (Mock Service Worker) for API Mocking

```typescript
// src/test/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: '1', name: 'Alice' },
      { id: '2', name: 'Bob' },
    ]);
  }),

  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: '3', ...body }, { status: 201 });
  }),
];

// src/test/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// src/test/setup.ts
import { server } from './server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

## Playwright (E2E Testing)

### Setup

```bash
pnpm add -D @playwright/test
pnpm exec playwright install
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  webServer: {
    command: 'pnpm dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
});
```

### E2E Test Example

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can log in', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('user@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome back')).toBeVisible();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.getByLabel('Email').fill('wrong@example.com');
    await page.getByLabel('Password').fill('wrong');
    await page.getByRole('button', { name: 'Sign in' }).click();

    await expect(page.getByRole('alert')).toContainText('Invalid credentials');
    await expect(page).toHaveURL('/login');
  });
});
```

## Testing Conventions

### File Naming

- Unit/integration: `<module>.test.ts` or `<component>.test.tsx` (co-located)
- E2E: `e2e/<feature>.spec.ts`

### Test Structure (AAA Pattern)

```typescript
it('describes the expected behavior', () => {
  // Arrange — set up preconditions
  const input = { name: 'Alice', age: 30 };

  // Act — perform the action
  const result = formatUser(input);

  // Assert — verify the outcome
  expect(result).toBe('Alice (30)');
});
```

### Test Naming

- Describe what the function/component does, not implementation
- Use natural language: `"shows error when email is invalid"`
- Group with `describe` blocks by function or feature

## Package.json Scripts

```jsonc
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui"
  }
}
```
