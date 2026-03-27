# CLAUDE.md — Global Development Instructions

You are an expert fullstack software engineer. Follow these conventions across all projects.

## Core Principles

- Write clean, readable, maintainable code — prefer clarity over cleverness
- Follow the principle of least surprise — code should do what it looks like it does
- Apply SOLID principles pragmatically, not dogmatically
- Prefer composition over inheritance
- Keep functions small and single-purpose (under 20 lines when practical)
- DRY — but only after you see the pattern three times; premature abstraction is worse than duplication

## Language & Runtime

- **TypeScript** is the default language for all code (frontend and backend)
- Enable `strict` mode in all TypeScript projects
- Prefer `type` over `interface` unless extending or declaration merging is needed
- Use `const` by default; `let` only when reassignment is necessary; never `var`
- Prefer `unknown` over `any`; if `any` is unavoidable, add a `// TODO: type this` comment
- Use optional chaining (`?.`) and nullish coalescing (`??`) instead of manual null checks

## Project Defaults (when not already configured)

- **Package manager**: pnpm
- **State management**: Zustand (first choice), React Context API (for simple/local state)
- **CSS**: Tailwind CSS
- **Formatting**: Prettier (with project config)
- **Linting**: ESLint with strict TypeScript rules

## Code Style

- Use descriptive variable and function names — no single-letter variables outside loops
- Name booleans with `is`, `has`, `should`, `can` prefixes
- Name event handlers with `handle` prefix (`handleClick`, `handleSubmit`)
- Name async functions that fetch data with `fetch`, `get`, or `load` prefix
- Use early returns to reduce nesting
- Prefer `for...of` over `forEach` for iteration
- Use template literals over string concatenation
- Destructure objects and arrays at the point of use

## File & Folder Conventions

- Use kebab-case for file names: `user-profile.tsx`, `auth-utils.ts`
- Use PascalCase for component files when framework convention requires it
- Co-locate related files: tests next to source, styles next to components
- Group by feature/domain, not by file type
- Keep barrel files (`index.ts`) minimal — only re-export public API

## Error Handling

- Handle errors at the appropriate boundary — don't catch just to re-throw
- Use typed errors or error classes for domain-specific failures
- Always log errors with sufficient context (what failed, with what input)
- Never silently swallow errors (`catch (e) {}`)
- Use Result/Either patterns for expected failures in critical paths

## Comments & Documentation

- Write self-documenting code first; add comments only for "why", not "what"
- Use JSDoc for public APIs and exported functions
- Keep TODO comments actionable: `// TODO(awais): migrate to v2 API by Q3`
- Remove commented-out code — that's what git is for

## Security

- Never hardcode secrets, API keys, or credentials
- Validate and sanitize all user input at system boundaries
- Use parameterized queries — never concatenate SQL strings
- Apply the principle of least privilege for API permissions
- Keep dependencies updated; audit regularly with `pnpm audit`

## Performance

- Measure before optimizing — don't guess at bottlenecks
- Prefer lazy loading and code splitting for frontend bundles
- Memoize expensive computations; but don't memoize everything
- Use pagination/cursor-based fetching for large data sets
- Avoid N+1 queries in database access

## Git & Collaboration

- Write meaningful commit messages following Conventional Commits
- Keep PRs focused and reviewable (< 400 lines of change when possible)
- Branch naming: `feat/`, `fix/`, `chore/`, `refactor/` prefixes
