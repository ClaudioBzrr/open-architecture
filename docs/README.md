# Open Architecture — Conventions Reference

AI-agent-oriented coding reference for projects (Express + React + PostgreSQL).

## Table of Contents

| Document | Scope |
|---|---|
| [architecture.md](./architecture.md) | Layer pattern, data flow, DI |
| [backend-conventions.md](./backend-conventions.md) | Server folder layout, services, controllers, repositories |
| [frontend-conventions.md](./frontend-conventions.md) | React components, contexts, routes, API client |
| [database-conventions.md](./database-conventions.md) | Prisma schema rules |
| [testing-conventions.md](./testing-conventions.md) | Jest setup, mocking strategy |
| [git-conventions.md](./git-conventions.md) | Commit message format, branching |
| [docker-and-cicd.md](./docker-and-cicd.md) | Docker multi-stage, Compose layers, Nginx, GitHub Actions CI/CD |
| [ai-agent-guide.md](./ai-agent-guide.md) | Quick-reference cheat sheet |

## How to Use This Reference

1. **Before modifying any file** — read the relevant convention doc for that layer.
2. **When adding a new feature** — follow the full stack path: route → controller → service → repository interface → implementation.
3. **When writing commits** — follow `git-conventions.md`.
4. **When unsure about a pattern** — check `ai-agent-guide.md` for the decision tree.
