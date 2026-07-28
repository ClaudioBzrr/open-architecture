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
  → Create directory in web/src/pages/dashboard/
  → Add route in web/src/routes/index.tsx with PageRoute guard
   → Add page key to NAV_ITEMS array
  → Create page component

New UI component?
  → Create directory in web/src/components/
  → Export through components/index.ts
```

## File Placement Rules

| What | Where |
|---|---|
| Repository interface | `server/src/types/repositories/` |
| Repository implementation | `server/src/repositories/{prisma,fetch,db,xlsx,bcrypt}/` |
| Repository DI wiring | `server/src/repositories/index.ts` |
| Adapter interface | `server/src/adapters/` |
| Adapter implementation | `server/src/adapters/{nodemailer,...}/` |
| Service | `server/src/services/{domain}-service/` |
| Controller | `server/src/controllers/{domain}-controller.ts` |
| Route | `server/src/routes/{domain}-routes.ts` |
| Domain types / entities | `server/src/types/entities/` |
| Shared middlewares | `server/src/common/` |
| Unit tests | `server/__tests__/services/{domain}/` |
| Frontend page | `web/src/pages/dashboard/{page-name}/` |
| Frontend component | `web/src/components/{component-name}/` |
| Frontend context | `web/src/contexts/` |
| Frontend types | `web/src/types/` |
| Frontend utils | `web/src/utils/` |

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
- Controllers: **no logging** — Morgan handles HTTP logging.
- Tests: mock the logger.

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

## Key Files

| File | Purpose |
|---|---|
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
