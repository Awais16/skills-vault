---
name: code-quality
description: "Code quality enforcement and review. Use when: reviewing code, refactoring for clean code, enforcing SOLID principles, setting up linting/formatting, improving readability, fixing code smells, applying design patterns, or conducting code audits."
---

# Code Quality

## When to Use

- Reviewing or auditing existing code for quality issues
- Refactoring messy, complex, or duplicated code
- Setting up or configuring ESLint, Prettier, or other quality tools
- Applying design patterns or SOLID principles
- Improving code readability and maintainability

## Clean Code Principles

### Naming

- Variables/functions: descriptive, intention-revealing names
- Booleans: `is`, `has`, `should`, `can` prefixes — `isLoading`, `hasPermission`
- Functions: verb-first — `getUser`, `handleSubmit`, `validateEmail`, `parseResponse`
- Constants: UPPER_SNAKE_CASE for true constants — `MAX_RETRY_COUNT`, `API_BASE_URL`
- Avoid generic names: `data`, `info`, `item`, `result` — be specific

### Functions

- Single responsibility — one function, one job
- Under 20 lines; extract when longer
- Max 3 parameters; use an options object for more
- Early returns to reduce nesting:
  ```typescript
  // Bad
  function process(user: User) {
    if (user) {
      if (user.isActive) {
        // deep nesting...
      }
    }
  }

  // Good
  function process(user: User) {
    if (!user) return;
    if (!user.isActive) return;
    // flat logic...
  }
  ```
- Pure functions when possible — same input, same output, no side effects

### SOLID Principles (Pragmatic Application)

1. **Single Responsibility**: Each module/class/function does one thing
2. **Open-Closed**: Extend behavior through composition, not modification
3. **Liskov Substitution**: Subtypes must be substitutable for their base types
4. **Interface Segregation**: Prefer many small interfaces over one large one
5. **Dependency Inversion**: Depend on abstractions (types/interfaces), not concretions

### Code Smells to Fix

| Smell | Fix |
|-------|-----|
| Long function (>20 lines) | Extract into smaller functions |
| Deep nesting (>3 levels) | Early returns, extract helpers |
| Primitive obsession | Create value objects or branded types |
| Feature envy | Move logic to the class/module that owns the data |
| Shotgun surgery | Consolidate related changes into one module |
| Duplicated logic (3+ times) | Extract shared utility |
| Boolean parameters | Replace with two separate functions or enum |
| Magic numbers/strings | Extract to named constants |
| God object/module | Split by responsibility |

## Linting & Formatting Setup

### ESLint (Recommended Config)

```jsonc
// .eslintrc.json — use @typescript-eslint for TS projects
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked",
    "prettier"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/consistent-type-imports": "error",
    "no-console": ["warn", { "allow": ["warn", "error"] }]
  }
}
```

### Prettier (Recommended Config)

```jsonc
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

## Design Patterns (When to Apply)

| Pattern | Use When |
|---------|----------|
| Strategy | Multiple algorithms for the same task; swap at runtime |
| Factory | Complex object creation; hide construction logic |
| Observer | Decouple event producers from consumers |
| Builder | Complex objects with many optional parameters |
| Repository | Abstract data access from business logic |
| Adapter | Bridge incompatible interfaces (e.g., third-party APIs) |
| Facade | Simplify a complex subsystem behind a clean API |

## Review Checklist

When reviewing code, check:

- [ ] No `any` types or type assertions without justification
- [ ] Functions are small, focused, and well-named
- [ ] Error handling is present at boundaries
- [ ] No hardcoded secrets or magic values
- [ ] No dead code or commented-out blocks
- [ ] Tests cover the critical path and edge cases
- [ ] Imports are organized and unused imports removed
- [ ] No mutable global state
- [ ] Consistent naming conventions throughout
- [ ] Complex logic has explanatory comments (why, not what)
