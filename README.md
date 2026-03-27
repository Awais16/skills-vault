# Claude AI — Personal Agent & Skills Kit

A portable collection of AI agent customizations, coding skills, and best-practice references designed for Claude Code. Copy these into any software project to get consistent, opinionated, expert-level AI coding assistance.

## What's Inside

```
claude/
├── CLAUDE.md                              # Global coding instructions (always loaded)
├── README.md                              # This file
├── agents/
│   ├── fullstack-developer.agent.md       # Fullstack developer agent (TDD, SOLID, KISS)
│   ├── code-reviewer.agent.md             # Read-only code review agent
│   └── architect.agent.md                 # System design and planning agent
└── skills/
    ├── code-quality/SKILL.md              # Clean code, SOLID, linting, reviews
    ├── typescript/SKILL.md                # Strict TS, generics, utility types, Zod
    ├── react/SKILL.md                     # Components, hooks, Zustand, accessibility
    ├── nextjs/SKILL.md                    # App Router, SSR/SSG, Server Components
    ├── testing/SKILL.md                   # Vitest, Playwright, mocking, test patterns
    ├── api-design/SKILL.md                # REST conventions, validation, auth, pagination
    ├── database/SKILL.md                  # Prisma, Drizzle, schemas, migrations, indexing
    ├── git-workflow/SKILL.md              # Conventional Commits, branching, PRs, hooks
    ├── devops/SKILL.md                    # Docker, GitHub Actions, env vars, deployment
    └── security/SKILL.md                  # OWASP, auth, XSS/CSRF, input validation
```

## Quick Start

### For Claude Code (CLI)

Copy the files into your project's `.claude/` directory:

```bash
# From your project root
cp -r /path/to/claude/CLAUDE.md .
mkdir -p .claude/skills .claude/agents
cp -r /path/to/claude/skills/* .claude/skills/
cp -r /path/to/claude/agents/* .claude/agents/
```

### For Personal (Cross-Project) Setup

Install globally so every project benefits:

```bash
# Skills — available in all projects
cp -r skills/* ~/.claude/skills/

# CLAUDE.md — copy to each project root (project-specific instructions)
cp CLAUDE.md /path/to/your-project/CLAUDE.md
```

### For VS Code Copilot

```bash
# Skills and agents
mkdir -p .github/skills .github/agents
cp -r skills/* .github/skills/
cp -r agents/* .github/agents/

# Global instructions
cp CLAUDE.md .github/copilot-instructions.md
```

## Files Explained

### CLAUDE.md — Global Instructions

Always-on coding conventions that load into every conversation:

- TypeScript strict mode, type conventions
- Naming rules (kebab-case files, `handle` prefixes, boolean `is`/`has` prefixes)
- Default tooling: pnpm, Zustand, Tailwind CSS, Prettier, ESLint
- Error handling, security, and performance guidelines
- Conventional Commits and PR standards

### Fullstack Developer Agent

A senior engineer persona with full tool access (terminal, edit, search, web). It orchestrates all skills and follows your coding conventions automatically.

**Key behaviors:**
- **Existing codebase first** — Discovers project conventions before writing code; asks when conventions are unclear
- **TDD by default** — Red → Green → Refactor for all features and bug fixes
- **SOLID + KISS** — Pragmatic application of SOLID principles; simplest solution that works
- **Small blast radius** — Minimal, focused changes; states what files are affected and why
- Follows project conventions; falls back to your defaults only when user confirms

### Code Reviewer Agent

A **read-only** review agent — no edit or terminal access. Reviews code for correctness, security, performance, readability, and convention adherence.

**Output format:** Structured findings with severity levels (🔴 Critical, 🟡 Warning, 🔵 Suggestion), plus a verdict (Approve / Request Changes / Needs Discussion).

**Tools:** `read`, `search`, `web` only

### Architect Agent

A **planning-only** agent that designs systems, evaluates trade-offs, and creates implementation plans. Does not write code.

**What it does:**
- Presents 2-3 options with trade-offs for every significant decision
- Breaks features into vertical slices (< 1 day each)
- Produces ADRs (Architecture Decision Records)
- Flags one-way doors and risks

**Tools:** `read`, `search`, `web`, `todo` only

### Skills Reference

| Skill | What It Covers |
|-------|---------------|
| **code-quality** | Clean code principles, SOLID, linting/Prettier setup, design patterns, code review checklist |
| **typescript** | Strict config, `type` vs `interface`, discriminated unions, branded types, generics, Zod integration |
| **react** | Component patterns, Zustand/Context state management, custom hooks, performance (memo, useCallback), forms, accessibility |
| **nextjs** | App Router structure, Server vs Client Components, data fetching/caching (SSR/SSG/ISR), API routes, Server Actions, middleware, metadata/SEO |
| **testing** | Vitest setup, component/hook testing, MSW mocking, Playwright e2e, AAA pattern, coverage |
| **api-design** | REST conventions, HTTP status codes, response envelopes, Zod validation, JWT auth, pagination (offset + cursor), rate limiting |
| **database** | Prisma/Drizzle schemas, client singleton, common queries, N+1 prevention, indexing strategy, migrations, seeding |
| **git-workflow** | Conventional Commits types, branch naming, PR templates, Husky + lint-staged setup, commitlint, conflict resolution |
| **devops** | Multi-stage Dockerfile, Docker Compose, GitHub Actions CI/CD, env var management with Zod validation, health checks, deployment checklist |
| **security** | OWASP top 10, input validation/sanitization, XSS/CSRF prevention, JWT/cookie auth, RBAC, security headers, rate limiting, file upload safety, dependency auditing |

## Default Preferences

These defaults are used when a project doesn't already have its own conventions:

| Setting | Default |
|---------|---------|
| Package manager | pnpm |
| Language | TypeScript (strict mode) |
| State management | Zustand (complex), Context API (simple) |
| CSS | Tailwind CSS |
| ORM | Prisma (default), Drizzle (performance-critical) |
| Test runner | Vitest (unit/integration), Playwright (e2e) |
| Linting | ESLint with `@typescript-eslint/strict-type-checked` |
| Formatting | Prettier |
| Commit style | Conventional Commits |
| File naming | kebab-case |

> **Note:** The agent always respects existing project conventions first. These defaults only apply when nothing is configured.

## Customization

### Override Defaults Per Project

Add a project-level `CLAUDE.md` to override any global defaults:

```markdown
# CLAUDE.md (project-specific)

## Overrides
- Use npm instead of pnpm
- Use CSS Modules instead of Tailwind
- Use Redux Toolkit for state management
```

### Add Project-Specific Skills

Create skills in your project's `.claude/skills/` directory:

```
.claude/skills/my-custom-skill/
└── SKILL.md
```

### Disable a Skill

Remove it from the skills directory, or set in its frontmatter:

```yaml
disable-model-invocation: true
user-invocable: false
```

## Extending This Kit

### Adding a New Skill

1. Create a folder: `skills/<skill-name>/`
2. Add a `SKILL.md` with required frontmatter:
   ```yaml
   ---
   name: skill-name
   description: "Keyword-rich description. Use when: trigger phrases."
   ---
   ```
3. Include: When to Use, Procedures, Code Examples, Checklists
4. Keep under 500 lines; use `./references/` for overflow

### Adding a New Agent

1. Create: `agents/<agent-name>.agent.md`
2. Add frontmatter with `description`, `tools`, and optionally `model`
3. Define persona, constraints, approach, and output format

### Suggested Future Additions

- **performance** — Core Web Vitals, bundle optimization, caching strategies
- **accessibility** — WCAG compliance, ARIA patterns, screen reader testing
- **monorepo** — Turborepo/Nx setup, workspace management, shared packages
- **mobile** — React Native patterns and conventions

## License

Personal use. Customize freely for your own projects.
