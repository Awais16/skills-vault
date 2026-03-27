---
name: react
description: "React best practices, component patterns, hooks, and state management. Use when: building React components, managing state with Zustand or Context API, writing custom hooks, optimizing renders, handling forms, implementing accessibility, or structuring component architecture."
---

# React

## When to Use

- Building or refactoring React components
- Managing state (Zustand, Context API)
- Writing custom hooks
- Optimizing component performance
- Handling forms and user input
- Implementing accessible UI patterns

## Component Conventions

### Functional Components Only

```typescript
type UserCardProps = {
  user: User;
  onSelect?: (userId: string) => void;
};

export function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <div
      role="button"
      tabIndex={0}
      onClick={() => onSelect?.(user.id)}
      onKeyDown={(e) => e.key === 'Enter' && onSelect?.(user.id)}
    >
      <h3>{user.name}</h3>
      <p>{user.email}</p>
    </div>
  );
}
```

### Component Organization

```
components/
├── ui/                          # Generic reusable primitives
│   ├── button.tsx
│   ├── input.tsx
│   ├── modal.tsx
│   └── index.ts                 # Public API barrel
├── features/                    # Feature-specific
│   ├── auth/
│   │   ├── login-form.tsx
│   │   ├── use-auth.ts         # Feature hook
│   │   └── auth-provider.tsx
│   └── dashboard/
│       ├── stats-card.tsx
│       └── activity-feed.tsx
└── layouts/
    ├── app-layout.tsx
    └── auth-layout.tsx
```

### Naming Rules

- **Components**: PascalCase — `UserCard`, `SearchFilter`
- **Files**: kebab-case — `user-card.tsx`, `search-filter.tsx`
- **Hooks**: `use` prefix, camelCase — `useAuth`, `useDebounce`
- **Event handlers**: `handle` prefix — `handleSubmit`, `handleClick`
- **Props types**: Component name + `Props` suffix — `UserCardProps`

## State Management

### Decision Tree

```
Is state used by only one component?
  → useState

Is state shared between parent/child?
  → Props (lift state up)

Is state shared between siblings/cousins?
  → Is it simple (1-2 values)?
    → Context API
  → Is it complex (many values, actions, derived state)?
    → Zustand

Is it server/async state?
  → TanStack Query (React Query)
```

### Zustand (Primary State Manager)

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

type AuthState = {
  user: User | null;
  isAuthenticated: boolean;
  login: (user: User) => void;
  logout: () => void;
};

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set) => ({
        user: null,
        isAuthenticated: false,
        login: (user) => set({ user, isAuthenticated: true }, false, 'login'),
        logout: () => set({ user: null, isAuthenticated: false }, false, 'logout'),
      }),
      { name: 'auth-storage' },
    ),
  ),
);

// Usage — select only what you need (prevents unnecessary re-renders)
function UserMenu() {
  const user = useAuthStore((state) => state.user);
  const logout = useAuthStore((state) => state.logout);
  // ...
}
```

### Context API (Simple Shared State)

```typescript
import { createContext, useContext, useState, type ReactNode } from 'react';

type ThemeContextValue = {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
};

const ThemeContext = createContext<ThemeContextValue | null>(null);

export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) throw new Error('useTheme must be used within ThemeProvider');
  return context;
}

export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  const toggleTheme = () => setTheme((prev) => (prev === 'light' ? 'dark' : 'light'));

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

## Hooks

### Built-in Hooks Usage

| Hook | Use When |
|------|----------|
| `useState` | Local state for a single component |
| `useReducer` | Complex state with multiple actions |
| `useEffect` | Side effects: subscriptions, DOM manipulation, external sync |
| `useRef` | DOM references, mutable values that don't trigger re-render |
| `useMemo` | Expensive computations that depend on specific values |
| `useCallback` | Stable function references for memoized children |
| `useId` | Generating unique IDs for accessibility attributes |

### Custom Hook Patterns

```typescript
// useDebounce — debounce rapidly changing values
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// useLocalStorage — persist state to localStorage
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    window.localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
}
```

## Performance

### Avoid Unnecessary Re-renders

```typescript
// 1. Memoize expensive components
const ExpensiveList = memo(function ExpensiveList({ items }: { items: Item[] }) {
  return items.map((item) => <ExpensiveItem key={item.id} item={item} />);
});

// 2. Stable references for callbacks passed to children
function Parent() {
  const [count, setCount] = useState(0);

  const handleIncrement = useCallback(() => {
    setCount((prev) => prev + 1);
  }, []);

  return <ExpensiveChild onIncrement={handleIncrement} />;
}

// 3. Derive state instead of syncing with useEffect
// Bad
const [filteredItems, setFilteredItems] = useState(items);
useEffect(() => {
  setFilteredItems(items.filter((i) => i.active));
}, [items]);

// Good — compute during render
const filteredItems = useMemo(
  () => items.filter((i) => i.active),
  [items],
);
```

### Rules of Memoization

- **Don't memoize everything** — default rendering is fast; profile first
- **DO memoize** when: component receives same props but parent re-renders frequently, or computation is genuinely expensive (>1ms)
- **DON'T memoize** when: props change on every render anyway, or component is simple/cheap

## Forms

### Controlled Form Pattern

```typescript
'use client';

import { useState, type FormEvent } from 'react';

type FormData = {
  name: string;
  email: string;
};

export function ContactForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const [formData, setFormData] = useState<FormData>({ name: '', email: '' });
  const [errors, setErrors] = useState<Partial<FormData>>({});

  function handleSubmit(e: FormEvent) {
    e.preventDefault();
    const newErrors: Partial<FormData> = {};

    if (!formData.name.trim()) newErrors.name = 'Name is required';
    if (!formData.email.includes('@')) newErrors.email = 'Invalid email';

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }

    onSubmit(formData);
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="name">Name</label>
      <input
        id="name"
        value={formData.name}
        onChange={(e) => setFormData((prev) => ({ ...prev, name: e.target.value }))}
        aria-invalid={!!errors.name}
        aria-describedby={errors.name ? 'name-error' : undefined}
      />
      {errors.name && <span id="name-error" role="alert">{errors.name}</span>}

      <label htmlFor="email">Email</label>
      <input
        id="email"
        type="email"
        value={formData.email}
        onChange={(e) => setFormData((prev) => ({ ...prev, email: e.target.value }))}
        aria-invalid={!!errors.email}
        aria-describedby={errors.email ? 'email-error' : undefined}
      />
      {errors.email && <span id="email-error" role="alert">{errors.email}</span>}

      <button type="submit">Submit</button>
    </form>
  );
}
```

## Accessibility Essentials

- Use semantic HTML: `<button>`, `<nav>`, `<main>`, `<section>`, `<article>`
- All `<img>` elements need `alt` text
- All form inputs need associated `<label>` elements
- Use `aria-` attributes when native semantics aren't sufficient
- Ensure keyboard navigability: `tabIndex`, `onKeyDown` for custom interactive elements
- Use `role="alert"` for dynamic error messages
- Use `useId()` for generating unique IDs for `htmlFor`/`aria-describedby`
- Test with screen reader and keyboard-only navigation

## Anti-patterns to Avoid

| Anti-pattern | Do Instead |
|-------------|------------|
| `useEffect` for derived state | Compute during render or `useMemo` |
| `useEffect` to sync props to state | Use the prop directly, or use a key to reset |
| Prop drilling through 5+ levels | Context API or Zustand |
| Huge monolithic components | Extract into smaller, focused components |
| Index as key in dynamic lists | Use stable unique IDs |
| Direct DOM manipulation | Use refs or React state |
| Fetching in `useEffect` (client) | TanStack Query or Server Components |
