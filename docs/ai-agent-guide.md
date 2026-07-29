# AI Agent Quick Reference

Cheat sheet for AI agents working in this codebase. Read this first, then dive into the specific convention doc when needed.

## Golden Rules

1. **Read before edit.** Never assume file contents.
2. **Follow the layer chain.** Route → Controller → Service → Repository. No shortcuts.
3. **No business logic in controllers.** Controllers parse and respond only.
4. **No req/res in services.** Services are framework-agnostic.
5. **No direct imports of repository instances in services.** Dependencies come via constructor.
6. **No direct DB access from feature code.** Use repository interfaces.
7. **Commit in English.** Conventional commits format: `type: description`.
8. **Document in English.** All docs in US English.
9. **Use Docker for everything.** Run `docker compose up` for development and `docker compose -f docker-compose.yml -f docker-compose.prod.yml` for production. Never skip containers — no bare `npm run dev` in instructions.
10. **Lowercase filenames everywhere.** All files and folders use lowercase with kebab-case. No camelCase, PascalCase, or mixed-case filenames (exceptions: environment variables, CSS custom properties, component function names inside files, and platform-recognized files like `Dockerfile`).

## Architecture Decision Tree

```
New HTTP endpoint?
  → Create route file → add to routes/index.ts (after authMiddleware if protected)
  → Create controller method
  → Create or reuse service class
  → Inject repository via constructor

New data source?
  → Define interface in types/repositories/
  → Implement in repositories/{prisma,fetch,db,xlsx}/
  → Instantiate in repositories/index.ts
  → Inject into services that need it

New UI page?
  → Create directory in web/src/pages/dashboard/ (lowercase kebab-case)
  → Add route in web/src/routes/index.tsx with PageRoute guard
  → Add page key to NAV_ITEMS array
  → Create index.tsx + index.module.css in the page directory

New UI component?
  → Create directory in web/src/components/ (lowercase kebab-case)
  → Create index.tsx + index.module.css + (optional) index.ts
  → Export through components/index.ts
```

## File Placement Rules

| What | Where | Files |
|---|---|---|
| Repository interface | `server/src/types/repositories/` | `{domain}-repository.ts` |
| Repository implementation | `server/src/repositories/{prisma,fetch,db,xlsx,bcrypt}/` | `{source}-{entity}-repository.ts` |
| Repository DI wiring | `server/src/repositories/index.ts` | `index.ts` |
| Adapter interface | `server/src/adapters/` | `{domain}-adapter.ts` |
| Adapter implementation | `server/src/adapters/{nodemailer,...}/` | `{provider}-{domain}-adapter.ts` |
| Service | `server/src/services/{domain}-service/` | `{action}-{domain}-service.ts` |
| Controller | `server/src/controllers/` | `{domain}-controller.ts` |
| Route | `server/src/routes/` | `{domain}-routes.ts` |
| Domain types / entities | `server/src/types/entities/` | `{entity}.ts` |
| Shared middlewares | `server/src/common/` | `{middleware-name}.ts` |
| Unit tests | `server/__tests__/services/{domain}/` | `{service-name}.test.ts` |
| Frontend page | `web/src/pages/dashboard/{page-name}/` | `index.tsx` + `index.module.css` + `index.ts` (optional) |
| Frontend component | `web/src/components/{component-name}/` | `index.tsx` + `index.module.css` + `index.ts` (optional) |
| Frontend context | `web/src/contexts/` | `{name}-context.tsx` |
| Frontend types | `web/src/types/` | `{domain}.ts` |
| Frontend utils | `web/src/utils/` | `{util-name}.ts` |

**Note:** All filesystem names (folders and files) must be lowercase kebab-case.

## Service Template

```typescript
import { logger } from "@/logger";

export class MyNewService {
  constructor(
    private readonly repoA: IRepoA,
    private readonly repoB: IRepoB,
  ) {}

  async execute(input: InputType): Promise<OutputType> {
    try {
      logger.info("MyNewService: starting...");
      const data = await this.repoA.load();
      // business logic here
      logger.info("MyNewService: completed");
      return result;
    } catch (err) {
      logger.error(`MyNewService error: ${err instanceof Error ? err.message : err}`);
      throw err;
    }
  }
}
```

## Controller Template

```typescript
import { Request, Response } from "express";

export class MyNewController {
  async handle(req: Request, res: Response) {
    try {
      const id = req.user?.userId;
      if (!id) throw new Error("Missing credentials.");
      const data = await myNewService.execute(req.body);
      return res.json(data);
    } catch (err) {
      return res.status(400).json(String(err).replace("Error: ", ""));
    }
  }
}
```

## Repository Interface Template

```typescript
// For read-only (provider pattern)
import { IProviderRepository } from "../entities/repository";
import { IMyEntity } from "../entities/my-entity";
export type IMyRepository = IProviderRepository<Promise<IMyEntity[]>>;

// For CRUD
import { IDataRepository } from "../entities/repository";
import { IMyEntity } from "../entities/my-entity";
export type IMyRepository = IDataRepository<IMyEntity>;
```

## Logging Rules

- Import: `import { logger } from "@/logger";`
- Services: `info` at entry/exit, `error` in catch, `warn` for guard clauses.
- Repositories: `error` only, return gracefully (empty arrays).
- Controllers: **no logging** — Morgan handles HTTP logging via Winston stream.
- Tests: mock the logger.
- **Log format:** See [`backend-conventions.md` → Logger](./backend-conventions.md#logger) for timestamp format (ISO 8601 with milliseconds), log pattern, and Winston + Morgan integration.

## Test Template

```typescript
jest.mock("@/logger", () => ({
  logger: { info: jest.fn(), warn: jest.fn(), error: jest.fn() },
}));

describe("MyNewService", () => {
  let service: MyNewService;
  let mockRepoA: jest.Mocked<IRepoA>;

  beforeEach(() => {
    mockRepoA = { load: jest.fn() } as any;
    service = new MyNewService(mockRepoA);
  });

  it("should do something", async () => {
    mockRepoA.load.mockResolvedValue([] as any);
    const result = await service.execute(input);
    expect(result).toEqual(expected);
  });
});
```

## Common Pitfalls

- **Prisma generate is mandatory** after any schema change.
- **Route mounting order** — anything before `authMiddleware` is public.
- **`VITE_API_URL`** is build-time only — requires rebuild to change.
- **Staging tables** are truncated on each upload — never read them directly. Use the cache layer instead.
- **`tsconfig.json`**: Pre-existing TypeScript config flags (e.g., `ignoreDeprecations`) exist for compatibility — do not remove without understanding why they were added.
- **Excel export**: Prefer native browser APIs over heavy libraries. If a feature can be built with native APIs (e.g., Excel export via OOXML), justify any third-party dependency.
- **Permission resolution**: Role > Direct UserPermission > `configured: false`.
- **Do not skip Docker.** AI agents commonly omit Docker and output bare `npm run dev` instructions instead. This happens because: (a) Docker adds container wiring, volume mounts, and networking that increases code-generation surface area for the AI; (b) the AI optimizes for the fastest "runs on my machine" path; and (c) training data often shows simple local-dev patterns. **Fight this tendency.** Every project must ship with working `docker-compose.yml`, `Dockerfile` (multi-stage), `entrypoint.sh`, and `nginx.conf`. See [`docker-and-cicd.md`](./docker-and-cicd.md) for the full pattern.
- **No uppercase filenames.** Never create files or folders with PascalCase (`MyComponent.tsx`) or camelCase (`myService.ts`). All filesystem names must be lowercase kebab-case. The only places where non-lowercase is allowed: environment variables (`VITE_API_URL`), CSS custom properties (`--color-primary`), and component function names inside files.

## Key Files

| File | Purpose |
|---|---|
| `server/src/logger.ts` | Winston + Morgan setup, log format, timestamp |
| `server/src/routes/index.ts` | Route mounting order and auth boundary |
| `server/src/repositories/index.ts` | DI container — all repository instances |
| `server/src/adapters/index.ts` | DI container — all adapter instances |
| `server/src/schedule.ts` | Cron logic + fork + snapshot |
| `server/src/env.ts` | Zod env var validation |
| `server/prisma/schema.prisma` | Complete data model |
| `web/src/contexts/auth-context.tsx` | Global auth state |
| `web/src/contexts/filter-context.tsx` | Global filters |
| `web/src/services/api.ts` | Frontend HTTP client |
| `web/src/routes/index.tsx` | Routes, guards, NAV_ITEMS |
| `web/src/utils/export.ts` | Excel export (no library) |
