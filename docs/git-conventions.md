# Git Conventions

## Commit Message Language

**All commit messages must be written in English (US).**

## Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <description>
```

Optional body and footer below the description.

## Types

| Type | When to Use |
|---|---|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `chore` | Maintenance tasks, config changes, dependency updates |
| `docs` | Documentation only |
| `style` | Formatting, semicolons, whitespace (no logic change) |
| `refactor` | Code restructuring without behavior change |
| `test` | Adding or updating tests |
| `ci` | CI/CD pipeline changes |
| `revert` | Reverting a previous commit |

## Scope

Optional scope in parentheses after the type:

```
feat(web): add column freeze/unfreeze with sticky left rendering
fix(server): serialize concurrent uploads per fileKey
chore: remove verbose admin loggers
```

## Rules

1. **Present tense, imperative mood** — "add feature" not "added feature" or "adds feature".
2. **No period at the end** of the subject line.
3. **Lowercase subject** after the type prefix.
4. **Keep subject under 72 characters** when possible.
5. **Body explains why, not what** — the diff shows what changed.

## Examples

```
feat: add status column to pending orders page
fix: remove www subdomain from Traefik rule to resolve ACME certificate error
fix: correct total calculation in report Excel export
feat: add search and export to inventory detail modal
chore: remove verbose admin loggers, document upload mutex in agent guide
fix: serialize concurrent uploads per fileKey to prevent staging table duplication
feat: make batch bar overlay content on mobile in analytics modal
fix(web): improve mobile responsiveness of page header actions
feat(login): replace 2D floating icons with 3D cube background animation
feat(server): add request body validation with Zod to order service
```

## Documentation Language

**All documentation must be written in English (US).**
