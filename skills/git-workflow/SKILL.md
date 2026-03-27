---
name: git-workflow
description: "Git workflow, commit conventions, branching, and PR best practices. Use when: writing commit messages, creating branches, setting up git hooks, reviewing PRs, resolving merge conflicts, managing releases, or configuring git workflows."
---

# Git Workflow

## When to Use

- Writing commit messages
- Creating and managing branches
- Setting up git hooks (Husky, lint-staged)
- Reviewing or creating pull requests
- Resolving merge conflicts
- Managing releases and versioning

## Conventional Commits

### Format

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

### Types

| Type | When to Use |
|------|------------|
| `feat` | New feature for the user |
| `fix` | Bug fix |
| `docs` | Documentation only changes |
| `style` | Formatting, missing semicolons (no code logic change) |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Performance improvement |
| `test` | Adding or fixing tests |
| `chore` | Build process, tooling, dependencies |
| `ci` | CI/CD changes |
| `revert` | Reverting a previous commit |

### Examples

```
feat(auth): add OAuth2 login with Google
fix(api): handle null response from payment gateway
refactor(dashboard): extract chart component from stats page
docs(readme): add deployment instructions
chore(deps): bump next from 14.1 to 14.2
perf(db): add index on users.email for login query

feat(auth)!: change session token format

BREAKING CHANGE: Session tokens now use JWT format.
Existing sessions will be invalidated.
```

### Rules

- Use imperative mood: "add feature" not "added feature" or "adds feature"
- First line under 72 characters
- Scope is optional but recommended for clarity
- Body explains **what** and **why**, not how
- Footer for breaking changes and issue references: `Closes #123`

## Branching Strategy

### Branch Naming

```
feat/user-authentication
fix/payment-null-response
chore/update-dependencies
refactor/extract-chart-component
hotfix/critical-auth-bypass
release/v2.1.0
```

### Flow

```
main (production)
  └── develop (integration)
       ├── feat/feature-a
       ├── feat/feature-b
       └── fix/bug-fix
```

- `main` — always deployable, tagged releases
- `develop` — integration branch (if using gitflow; skip for trunk-based)
- Feature branches — short-lived, merged via PR
- Hotfix branches — branch from `main`, merge back to `main` and `develop`

### Trunk-Based (Preferred for Small Teams)

- Branch directly from `main`
- Keep branches short-lived (< 2 days)
- Use feature flags for incomplete features
- Merge frequently to avoid drift

## Pull Request Guidelines

### PR Title

Follow Conventional Commits format: `feat(auth): add Google OAuth login`

### PR Description Template

```markdown
## What
Brief description of the change.

## Why
Context and motivation — link to issue if applicable.

## How
Technical approach taken.

## Testing
- [ ] Unit tests added/updated
- [ ] Manual testing performed
- [ ] Edge cases considered

## Screenshots
(if UI changes)
```

### PR Best Practices

- Keep PRs small and focused (< 400 lines of change)
- One logical change per PR
- Self-review before requesting review
- Respond to all review comments
- Squash merge for clean history (unless commits are meaningful)
- Delete branch after merge

## Git Hooks Setup (Husky + lint-staged)

```bash
# Setup
pnpm add -D husky lint-staged
pnpm exec husky init
```

```jsonc
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml}": ["prettier --write"]
  }
}
```

```bash
# .husky/pre-commit
pnpm exec lint-staged
```

```bash
# .husky/commit-msg
pnpm exec commitlint --edit $1
```

### Commitlint Config

```javascript
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'ci', 'revert',
    ]],
    'subject-max-length': [2, 'always', 72],
  },
};
```

## Conflict Resolution

1. Pull latest from target branch: `git pull origin main`
2. Rebase your branch: `git rebase main` (preferred) or merge
3. Resolve conflicts in each file — keep both changes when they don't overlap
4. Run tests after resolution to verify nothing broke
5. Force push your branch: `git push --force-with-lease` (safe force push)

## Useful Git Commands

```bash
# Interactive rebase — clean up commits before PR
git rebase -i HEAD~3

# Amend last commit (before push)
git commit --amend --no-edit

# Stash with message
git stash push -m "wip: dashboard refactor"

# Cherry-pick a specific commit
git cherry-pick <commit-hash>

# See what changed in a file
git log --oneline -10 -- path/to/file

# Undo last commit (keep changes staged)
git reset --soft HEAD~1
```
